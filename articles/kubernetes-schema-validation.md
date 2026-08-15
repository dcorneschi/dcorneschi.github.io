# Kubernetes Schema Validation: Catching Misconfigurations Before They Hit Your Cluster

## Why Validate Kubernetes YAML Files?

How do you ensure the stability of your Kubernetes clusters? How do you know that your manifests are syntactically valid, free of invalid data types, or not missing mandatory fields?

Most often, we only become aware of these misconfigurations at the worst time — when trying to deploy the new manifests. Specialized tools and a "shift-left" approach make it possible to verify a Kubernetes schema before they're applied to a cluster.

Running schema validation tests is important, and the sooner the better.

---

## Available Tools

### kubectl --dry-run

Verifying the state of Kubernetes manifests may seem like a trivial task, because the Kubernetes CLI (kubectl) has the ability to verify resources before they're applied to a cluster. You can verify the schema by using the `--dry-run` flag (`--dry-run=client/server`) when specifying the `kubectl create` or `kubectl apply` commands, which will perform the validation without applying Kubernetes resources to the cluster.

However, this approach is more complex than it appears. A running Kubernetes cluster is required to obtain the schema for the set of resources being validated. When incorporating manifest verification into a CI process, you must also manage connectivity and credentials to perform the validation. This becomes even more challenging when dealing with multiple microservices in several environments (prod, dev, etc.).

**Client mode** (`--dry-run=client`):
- Runs locally — does not send the request to the API server
- Only builds and prints the object
- Misses many obvious misconfigurations
- No authentication or authorization checks are performed
- Server-side behaviors like defaulting or admission control don't apply
- Almost useless for comprehensive validation
- Useful for **generating YAML templates** from imperative commands:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment webapp --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl expose deploy webapp --port=80 --dry-run=client -o yaml > svc.yaml
```

> **Note:** Some client-side commands (like `kubectl expose`) still contact the API server to look up existing resources, even in client mode. If the referenced resource doesn't exist, the command will fail.

**Server mode** (`--dry-run=server`):
- Sends the request to the API server but does not persist to etcd
- Catches all misconfigurations
- Slower execution (especially with many files)
- Requires cluster connectivity and credentials
- Returns the full object as it would be stored, including all server-applied defaults (imagePullPolicy, terminationGracePeriodSeconds, serviceAccount, tolerations, etc.)

```bash
# See the full defaulted object
kubectl apply -f deployment.yaml --dry-run=server -o yaml

# Verbose output showing the actual API request
kubectl apply -f deployment.yaml --dry-run=server -v=8
```

**None** (`--dry-run=none`):
- Same as not using the flag — the request is made and persisted if it succeeds

#### What Server-Side Dry Run Actually Validates

Unlike client-side dry run that only builds and prints the object locally, server-side dry run sends your manifest through the entire admission chain without persisting it to etcd. This catches:

- **Schema validation** — field types, required fields
- **Admission controllers** — both built-in and custom
- **Validating webhooks** — custom validation logic
- **Mutating webhooks** — see what mutations would happen
- **RBAC permissions** — whether you can create the resource
- **Resource quotas** — whether namespace has capacity
- **Pod Security Admission** — security standard violations

### yamllint

Yamllint is a linter for YAML files. It has **no knowledge of Kubernetes** — it doesn't know what a Pod or Deployment is. Its job is purely to answer: "Is this valid, well-formatted YAML?"

**What it validates:**
- YAML syntax errors (bad indentation, invalid characters, broken quoting)
- Duplicate keys (which YAML parsers silently override — taking only the last value)
- Style and formatting (consistent indentation, line length, trailing spaces, comment spacing)

**What it does NOT validate:**
- Whether `kind: Deployment` is a valid Kubernetes resource
- Whether `spec.containers` has the correct structure
- Whether field types are correct (string vs integer)
- Whether required Kubernetes fields are present

**Installation:**

```bash
pip install yamllint
```

**Usage:**

```bash
yamllint deployment.yaml
yamllint manifests/
```

**Recommended config for Kubernetes manifests (`~/.yamllint`):**

Line length and document-start (`---`) rules don't apply well to Kubernetes manifests, so disable them:

```yaml
extends: default

rules:
  line-length: disable
  indentation:
    spaces: 2
  document-start: disable
  trailing-spaces: enable
```

yamllint is useful as the **first step** in your validation pipeline because if YAML syntax is broken, kubeconform will also fail — but with a less helpful error message. yamllint gives clearer feedback at the syntax level.

### kubeconform

Kubeconform is a command-line tool developed to validate Kubernetes manifests without the requirement of having a running Kubernetes environment. Verification is performed against pre-generated JSON schemas that are created from the OpenAPI specifications (swagger.json) for each particular Kubernetes version.

```bash
kubeconform --strict misconfigs/*.yaml
```

**Key features:**
- Relies on `yannh/kubernetes-json-schema` (actively maintained)
- Supports the latest Kubernetes schema versions
- Supports Custom Resource Definitions (CRDs)
- Fastest execution speed among all tools tested

**Validating CRDs with custom schemas:**

By default, kubeconform skips resources it doesn't have a schema for (like custom CRDs). Use `--schema-location` to point it to your CRD schemas:

```bash
kubeconform --schema-location default \
  --schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  manifests/*.yaml
```

### kubectl --validate=strict

Since Kubernetes v1.27, kubectl supports `--validate=strict` which rejects unknown fields in manifests without performing a full dry-run. This is a lighter alternative when you just want strict field checking without the overhead of the admission chain:

```bash
kubectl apply -f deployment.yaml --validate=strict --dry-run=client
```

This catches typos in field names (e.g. `replicas` misspelled as `replica`) without needing cluster-side validation.

### pluto

Pluto is a tool specifically for detecting deprecated and removed Kubernetes API versions in your manifests. It's useful when preparing for cluster upgrades — for example, finding all manifests still using `extensions/v1beta1` or `policy/v1beta1`.

```bash
# Scan manifests for deprecated APIs
pluto detect-files -d manifests/

# Check against a target Kubernetes version
pluto detect-files -d manifests/ --target-versions k8s=v1.29.0

# Scan a running cluster
pluto detect-helm -o wide
```

Pluto doesn't validate schema — it only checks API version deprecations. Use it alongside kubeconform for full coverage.

---

## kubectl dry-run: Client vs Server

| Feature | --dry-run=client | --dry-run=server |
|---|---|---|
| Requires cluster connection | No (local only) | Yes |
| Sends request to API server | No | Yes |
| AuthN/AuthZ checks | No | Yes |
| API deprecation | Caught | Caught |
| Invalid kind value | Didn't catch | Caught |
| Invalid label value | Didn't catch | Caught |
| Invalid protocol type | Didn't catch | Caught |
| Invalid spec key | Caught | Caught |
| Missing image | Didn't catch | Caught |
| Wrong K8s indentation | Caught | Caught |
| RBAC permissions | Didn't catch | Caught |
| Resource quota exceeded | Didn't catch | Caught |
| Admission webhook rejection | Didn't catch | Caught |
| Pod Security Admission violations | Didn't catch | Caught |
| Shows server-applied defaults | No | Yes |
| Speed (100 files) | Fast | ~60 seconds |
| CRD support | Yes | Yes |

**Conclusion:** Client mode is almost useless — it misses most obvious misconfigurations and still requires a cluster connection. Server mode catches everything but is significantly slower, especially with many files.

### Output Comparison Example

The key difference in output — server mode shows all defaulted fields that would be applied:

**`--dry-run=client` output** (minimal, your YAML as-is):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx-container
        ports:
        - containerPort: 80
```

**`--dry-run=server` output** (full object with all server-applied defaults):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: "2025-01-15T14:35:44Z"
  generation: 1
  name: nginx-deployment
  namespace: default
  uid: 95669a87-3895-4fd0-977e-e48915290cc2
spec:
  progressDeadlineSeconds: 600          # defaulted
  replicas: 3
  revisionHistoryLimit: 10              # defaulted
  selector:
    matchLabels:
      app: nginx
  strategy:                             # defaulted
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always         # defaulted
        name: nginx-container
        ports:
        - containerPort: 80
          protocol: TCP                 # defaulted
        resources: {}                   # defaulted
        terminationMessagePath: /dev/termination-log     # defaulted
        terminationMessagePolicy: File                   # defaulted
      dnsPolicy: ClusterFirst           # defaulted
      restartPolicy: Always             # defaulted
      schedulerName: default-scheduler  # defaulted
      securityContext: {}               # defaulted
      terminationGracePeriodSeconds: 30 # defaulted
```

Server mode reveals what the cluster would actually create — useful for understanding defaults, debugging unexpected behavior, and seeing mutations from admission webhooks.

### Understanding Server-Applied Defaults: RollingUpdate Strategy

One of the most common defaults server mode reveals is the Deployment strategy:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
  type: RollingUpdate
```

These are applied by Kubernetes when you don't declare a strategy explicitly.

**`maxSurge: 25%`** — Maximum pods that can be created ABOVE the desired count during an update.

**`maxUnavailable: 25%`** — Maximum pods that can be UNAVAILABLE during an update.

Percentages are rounded up. Example with 6 replicas (25% of 6 = 1.5 → rounds to 2):

```
Update flow (6 replicas, maxSurge=2, maxUnavailable=2):

1. Start: 6 old pods running
2. Create 2 new pods → 8 total (maxSurge allows 6+2)
3. New pods ready → terminate 2 old pods → 6 total
4. Create 2 more new pods → 8 total
5. Ready → terminate 2 old → 6 total
6. Create 2 final new pods → 8 total
7. Ready → terminate last 2 old → 6 new pods running

Minimum available at any time: 4 pods (6 - maxUnavailable 2)
Maximum total at any time: 8 pods (6 + maxSurge 2)
```

Key takeaway: `--dry-run=client` won't show these defaults, `--dry-run=server` will — this is one of the primary reasons to use server mode.

---

## Benchmark Speed Test

Using `hyperfine` to benchmark execution time:

**Against 7 files with misconfigurations:**
1. **kubeconform** — fastest
2. **kubectl --dry-run=client** — fast
3. **kubectl --dry-run=server** — slowest

**Against 100 valid Kubernetes files:**
- kubeconform and kubectl `--dry-run=client` provide fast results
- kubectl `--dry-run=server` is significantly slower (~60 seconds for 100 files)

---

## Kubernetes Versions Support

Kubeconform accepts the Kubernetes schema version as a flag and supports the latest Kubernetes versions thanks to its actively maintained schema repository (`yannh/kubernetes-json-schema`).

The variety of Kubernetes schemas support is especially important if you want to migrate to a new Kubernetes version. With kubeconform you can set the version and start evaluating which configurations must be changed to support the cluster upgrade.

---

## CRD Support

- **kubectl dry-run** — supports CRDs
- **kubeconform** — supports CRDs

---

## Server-Side Dry Run: Additional Use Cases

### Seeing Mutations from Webhooks

Running `kubectl apply -f pod.yaml --dry-run=server -o yaml` captures the output after mutating webhooks have run. You can compare the original manifest against the mutated result to see exactly what labels, annotations, resource requests, or sidecars a webhook would inject.

### Testing RBAC Permissions

Dry run reveals RBAC issues before they block a real deployment. Using `--as=<user>` with `--dry-run=server` lets you verify whether a specific user or service account has permission to create or update a resource, without actually creating it.

### Validating Helm Charts

You can combine Helm templating with server-side dry run:

```bash
helm template myapp ./chart | kubectl apply --dry-run=server -f -
```

This renders the chart locally, then validates the resulting manifests against the live API server — catching both templating errors and admission issues.

### Combining Dry Run with Diff

Use `kubectl diff -f deployment.yaml` alongside dry run to first validate that a manifest is accepted, then see exactly what fields would change. This is useful for reviewing updates before applying them.

### Dry Run with Patches

You can dry-run patch operations (strategic merge or JSON patch) to verify they would succeed and see the resulting object, without modifying the live resource.

```bash
kubectl patch deployment webapp --dry-run=server --type strategic \
  --patch '{"spec": {"replicas": 10}}'
```

### Limitations of Server-Side Dry Run

Dry run doesn't catch everything:

- Resource dependencies that aren't created yet
- External system integrations
- Eventual consistency issues
- Timing-dependent problems
- Issues that only appear under load

Always test in a staging environment for complete validation.

---

## Recommended Strategies

### Shift-Left Approach

When possible, the best setup is to run `kubectl --dry-run=server` on every code change. However, you probably can't do that because you can't allow every developer or CI machine to have a connection to your cluster. The second-best effort is to run **kubeconform**.

### Pair with Policy Enforcement

Because kubeconform doesn't cover all common misconfigurations, it's recommended to run it with a **policy enforcement tool** on every code change to fill the coverage gap. The combination of `kubeconform + conftest` provides good coverage.

### During CD

During the CD step, it shouldn't be a problem to have a connection with your cluster, so you should always run `kubectl --dry-run=server` before deploying new code changes.

### Minikube Workaround

Another option for using kubectl dry-run in server mode without a connection to your production Kubernetes environment is to run `minikube + kubectl --dry-run=server`. The downside is that the minikube cluster must be set up like production (same volumes, namespace, etc.) or you'll encounter errors when validating your manifests.

---

## Quick Start: Validating with kubeconform

### Installation

```bash
# macOS
brew install kubeconform

# Linux
go install github.com/yannh/kubeconform/cmd/kubeconform@latest
```

### Basic Usage

```bash
# Validate a single file
kubeconform deployment.yaml

# Validate with strict mode (reject additional properties)
kubeconform --strict deployment.yaml

# Validate against a specific Kubernetes version
kubeconform -kubernetes-version 1.27.0 deployment.yaml

# Validate all YAML files in a directory
kubeconform --strict manifests/*.yaml

# Use with summary output
kubeconform --summary manifests/
```

### CI Integration Example

```yaml
# GitHub Actions example
name: Validate K8s Manifests
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install kubeconform
        run: |
          curl -sSL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          sudo mv kubeconform /usr/local/bin/
      - name: Validate manifests
        run: kubeconform --strict --summary k8s/
```

---

## Conclusion

For most teams, the recommended validation pipeline is:

1. **yamllint** → "Is this valid YAML?" (syntax, formatting, duplicate keys)
2. **kubeconform** (+ policy enforcement tool) → "Is this valid Kubernetes?" (schema, field types, required fields)
3. **kubectl --dry-run=server** (during CD) → "Will the API server accept this?" (webhooks, RBAC, quotas, admission controllers)

Each layer catches a different class of errors. Running them in order gives you the clearest feedback at the right level, without requiring every developer to have direct cluster access.
