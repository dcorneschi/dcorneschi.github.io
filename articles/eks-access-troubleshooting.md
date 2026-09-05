# Troubleshooting EKS Access: The 401 That Isn't Kubernetes, and Its Cousins

A `kubectl` command fails against an EKS cluster. The instinct is to start poking at
Kubernetes — RBAC, the `aws-auth` ConfigMap, access entries, the endpoint. That instinct is
usually **wrong**, because on EKS the Kubernetes API server is the *last* link in a long
chain that mostly lives in **AWS land**:

```
kubeconfig → exec plugin (aws eks get-token) → AWS CLI → credential chain → STS
          → presigned URL → bearer token → EKS API server → (authn) → (authz/RBAC)
```

Most access failures happen somewhere in that chain **before** Kubernetes ever gets to make
an authorization decision. This article walks the canonical example — a `kubectl` 401 caused
by a two-year-old credentials file (from Pradhyuman Pandey's write-up, linked in Sources) —
and then generalizes it into a triage method and a catalog of the other EKS access problems
you'll hit.

---

## 1. The anchor case: a 401 that wasn't Kubernetes

The symptom, from a bastion that "worked until last week":

```
error: You must be logged in to the server
       (the server has asked for the client to provide credentials)
```

The cluster was healthy, the kubeconfig looked right, and nothing had "changed."

### Read the error exactly

The wording is precise: *the server has asked for the client to provide credentials* means
the API server received a request, looked at the **bearer token**, and **rejected the token
itself**. That is an **authentication** failure.

This distinction is the whole game:

- **401 / "provide credentials"** → authentication → the *token* is bad → look at **token
  generation** (AWS side).
- **403 / `Forbidden`** → authorization → the token was fine, but **RBAC / access entries**
  don't permit the action → look at **Kubernetes/EKS access mapping**.

Chasing RBAC for a 401 is looking in the wrong layer.

### Who mints the token? The exec plugin

On EKS, the kubeconfig user is almost always the `aws eks get-token` exec plugin:

```yaml
users:
- name: arn:aws:eks:<region>:<account-id>:cluster/<cluster-name>
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args: [--region, <region>, eks, get-token, --cluster-name, <cluster-name>]
```

`kubectl` shells out to `aws eks get-token`, which **signs a presigned STS URL** using
whatever credentials the AWS CLI can find, and hands that to the API server as the bearer
token. So a 401 can originate in any of four layers: kubectl, the exec plugin, the **AWS
CLI / credential chain**, or the EKS auth mapping.

### Check the cheapest layer first

```bash
$ aws sts get-caller-identity
An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation:
  The security token included in the request is invalid.
```

That single command short-circuits everything: if STS itself rejects the credentials,
nothing built on top of it (the presigned URL, the bearer token) can possibly work.
**Kubernetes wasn't broken — AWS auth was.**

### The root cause: a dead key shadowing a working role

```bash
$ aws configure list
      access_key   ****  shared-credentials-file
      secret_key   ****  shared-credentials-file
      region       <r>   imds

$ ls -la ~/.aws/credentials
-rw------- 1 ubuntu ubuntu 116 Sep  7 2023 credentials     # ~2.5 years old
```

A static IAM-user access key had been sitting in `~/.aws/credentials` for years. The key
was rotated or the user deprovisioned upstream, but the bastion never found out — **cached
sessions and long-running processes don't re-auth**, so the break only surfaced when
something forced a fresh STS call (like `aws eks get-token`).

The bastion also had an **instance profile** attached and already mapped into the cluster's
access entries — authorized and waiting. But the **credential chain reads
`~/.aws/credentials` before it falls back to IMDS**, so the dead static key silently
shadowed a perfectly good instance role.

### The reversible probe and the fix

Test the hypothesis without risk — rename the file, wrap it in a `trap` that restores it on
exit (see the companion doc `aws-credentials-disable-with-trap.md`):

```bash
mv ~/.aws/credentials ~/.aws/credentials.disabled.$$
trap "mv ~/.aws/credentials.disabled.$$ ~/.aws/credentials" EXIT
aws sts get-caller-identity     # now returns the assumed-role identity via IMDS
kubectl get ns                  # works
```

The permanent fix was the same move, minus the trap — **rename, don't delete**, to keep an
audit trail:

```bash
mv ~/.aws/credentials ~/.aws/credentials.dead-key-2026
```

No kubeconfig edits, no IAM changes, no RBAC work. The cluster side was correct all along.

---

## 2. Decode the AWS error codes — they're not interchangeable

For any AWS-backed credentials failure, the exact error tells you the fix:

| Error | What it means | Fix |
|---|---|---|
| `InvalidClientTokenId` | The access key ID doesn't exist in IAM (deleted, disabled, never valid). | Get valid creds / fall back to a role. |
| `SignatureDoesNotMatch` | The key ID exists but the **secret** is wrong. | Correct the secret key. |
| `ExpiredToken` / `ExpiredTokenException` | A temporary session token timed out. | Refresh the session (re-login/assume). |
| `AccessDenied` | Creds are valid, but IAM policy forbids the action. | Fix the IAM policy (this is authZ, not authN). |
| `UnrecognizedClientException` | Similar to InvalidClientTokenId at the STS/service edge. | Check the key and region. |

Read them slowly — `InvalidClientTokenId`, `SignatureDoesNotMatch`, and `ExpiredToken` are
three different problems with three different fixes.

---

## 3. The pocket triage recipe (kubectl 401 on EKS)

Walk from the cheapest layer to the most expensive. **Stop at the first failure.**

```bash
# 1. Is AWS itself happy?  (If this fails, it's NOT a Kubernetes problem.)
aws sts get-caller-identity

# 2. Where is the CLI getting credentials + region from?
aws configure list

# 3. Can the kubeconfig's exec plugin actually mint a token?
kubectl config view --minify --raw          # find the exec command
aws --region <r> eks get-token --cluster-name <c>

# 4. ONLY if 1–3 succeed, look at EKS auth mapping:
aws eks describe-cluster --name <c> --query 'cluster.accessConfig'
aws eks list-access-entries --cluster-name <c>
```

If step 1 fails, everything you do before fixing it is wasted effort.

---

## 4. The wider family of EKS access problems

The 401-from-a-dead-key is one species. Here are the others you'll meet, grouped by which
layer they live in.

### A. Credential-chain problems (authentication, AWS side)

- **Dead/rotated static key shadowing a role** — the anchor case. Symptom: 401,
  `InvalidClientTokenId` from `sts get-caller-identity`.
- **Expired SSO / assumed-role session** — Symptom: 401, `ExpiredToken`. Fix:
  `aws sso login` or re-assume the role. Common on laptops mid-afternoon.
- **Env vars overriding everything** — stale `AWS_ACCESS_KEY_ID` / `AWS_SESSION_TOKEN`
  exported in the shell sit **above** the file and IMDS in the chain. `aws configure list`
  shows the source as `env`. Fix: `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY
  AWS_SESSION_TOKEN`.
- **Wrong profile** — kubeconfig or shell targeting the wrong `AWS_PROFILE`, so you
  authenticate as an identity that isn't mapped. Symptom: often a **403**, since the token
  is valid but the identity isn't authorized.
- **IMDS unavailable / hop limit** — on EC2, if IMDSv2 is enforced and the metadata hop
  limit is 1, containers/pods can't reach IMDS to get the instance-profile creds. Symptom:
  no fallback identity; 401 or timeouts.

### B. Exec-plugin / kubeconfig problems

- **`aws` CLI not on PATH (or wrong version)** — the exec plugin can't run. Symptom:
  `kubectl` errors like `exec: executable aws not found`, or an old CLI that doesn't support
  `eks get-token`.
- **`apiVersion` mismatch** — old kubeconfigs pin
  `client.authentication.k8s.io/v1alpha1`, which newer clients reject. Fix: regenerate with
  `aws eks update-kubeconfig`.
- **Stale kubeconfig / wrong cluster or region** — `update-kubeconfig` was run against a
  different account/region, or the cluster was recreated (new CA/endpoint). Symptom: TLS or
  auth errors. Fix: re-run `aws eks update-kubeconfig --name <c> --region <r>`.

### C. EKS authorization mapping (authZ — the 403 family)

EKS has **two** ways to map IAM identities to Kubernetes permissions, and confusion between
them is a top cause of 403s:

- **Access entries (the modern API)** vs the **`aws-auth` ConfigMap (legacy)**. The
  cluster's **authentication mode** (`API`, `API_AND_CONFIG_MAP`, or `CONFIG_MAP`) decides
  which is authoritative:
  ```bash
  aws eks describe-cluster --name <c> --query 'cluster.accessConfig.authenticationMode'
  aws eks list-access-entries --cluster-name <c>
  kubectl -n kube-system get configmap aws-auth -o yaml   # legacy path
  ```
  If the mode is `API`-only, edits to `aws-auth` do nothing — a classic "I added the role and
  it still 403s" trap.
- **Identity not mapped** — the IAM role/user simply isn't in access entries or `aws-auth`.
  Symptom: 403 `Forbidden` (authentication succeeded).
- **Role ARN path/case mismatch** — the mapped ARN doesn't exactly match the assumed-role
  ARN (e.g. an IAM path, or the STS `assumed-role` form vs the role ARN). Symptom: 403 even
  though "the role is mapped."
- **Cluster-creator confusion** — the IAM principal that *created* the cluster has implicit
  admin, but teammates using other identities don't until explicitly granted. Symptom:
  "works for me, 403 for everyone else."

### D. Network / endpoint reachability (not auth at all)

- **Private-only endpoint** — cluster endpoint access is `Private`, and you're calling from
  outside the VPC/connected network. Symptom: connection timeout / no route, not a 401.
  Check `aws eks describe-cluster --query 'cluster.resourcesVpcConfig'`.
- **Public endpoint CIDR allowlist** — `publicAccessCidrs` doesn't include your source IP.
  Symptom: timeout from the wrong network.
- **DNS resolution for private endpoint** — the VPC lacks the DNS settings for the
  EKS-managed private hosted zone. Symptom: can't resolve the endpoint. (See the companion
  doc on EKS control-plane architecture.)

---

## 5. A layer map for reading symptoms

| Symptom | Most likely layer | First command |
|---|---|---|
| `401` / "provide credentials" | Token generation (AWS creds/STS) | `aws sts get-caller-identity` |
| `403 Forbidden` | EKS authZ (access entries / aws-auth) | `aws eks list-access-entries --cluster-name <c>` |
| `exec: executable aws not found` | Exec plugin / PATH | `which aws && aws --version` |
| `v1alpha1` / auth apiVersion error | Stale kubeconfig | `aws eks update-kubeconfig ...` |
| Connection timeout / no route | Network / endpoint access | `aws eks describe-cluster --query cluster.resourcesVpcConfig` |
| Works for creator, 403 for others | Missing access mapping | add an access entry |
| Intermittent after hours | Expired SSO/session | `aws sso login` |

---

## Summary

- A `kubectl` **401** on EKS ("the server has asked for the client to provide credentials")
  is almost always an **authentication** failure in the AWS part of the chain — a bad
  **bearer token** — not RBAC. **403 `Forbidden`** is the authorization/RBAC case.
- The token comes from `aws eks get-token`, which signs an STS presigned URL with whatever
  the **AWS credential chain** resolves. So **`aws sts get-caller-identity` is the cheapest,
  first check** — if it fails, stop looking at Kubernetes.
- Read AWS error codes literally: `InvalidClientTokenId`, `SignatureDoesNotMatch`,
  `ExpiredToken`, and `AccessDenied` are four different problems.
- The classic root cause is a **stale static key in `~/.aws/credentials` shadowing a working
  instance profile**, because the file outranks IMDS in the chain. Prove it with a
  reversible `mv` + `trap` probe; fix it by **renaming (not deleting)** the dead file.
- Beyond that, EKS access problems cluster into: **credential-chain** (env vars, wrong
  profile, expired SSO, IMDS), **exec-plugin/kubeconfig** (PATH, apiVersion, stale config),
  **authZ mapping** (access entries vs `aws-auth`, authentication mode, ARN mismatch), and
  **network/endpoint** (private endpoint, CIDR allowlist, DNS). Match the **symptom to the
  layer** and walk cheapest-first.

---

### Sources

- [Configure credential precedence / provider chain (AWS SDKs and Tools reference)](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)
- [Grant IAM users access to Kubernetes with access entries (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Cluster authentication mode and the aws-auth ConfigMap (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html)
- [Cluster API server endpoint access (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
