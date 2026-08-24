# Fix Cluster Autoscaler on Hetzner Cloud — Empty nodeConfigs

## Problem

Cluster Autoscaler crashes immediately with:

```
F0207 13:14:01.555991       1 hetzner_cloud_provider.go:198] No cluster config present provider: <nil>
```

## Root Cause

The `HCLOUD_CLUSTER_CONFIG` environment variable has empty `nodeConfigs`:

```json
{"imagesForArch":{"arm64":"ubuntu-24.04","amd64":"ubuntu-24.04"},"nodeConfigs":{}}
```

The autoscaler can't manage any node pools because none are defined in the config.

## Solution

### 1. Check the Logs

```bash
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

### 2. Inspect the Current Config

```bash
kubectl get deployment cluster-autoscaler -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="HCLOUD_CLUSTER_CONFIG")].value}' | base64 -d
```

Or if the value is stored as a raw string (not base64):

```bash
kubectl get deployment cluster-autoscaler -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="HCLOUD_CLUSTER_CONFIG")].value}'
```

### 3. Create Proper Config with Node Pool Definition

Create `cluster-autoscaler-config.json`:

```json
{
  "imagesForArch": {
    "arm64": "ubuntu-24.04",
    "amd64": "ubuntu-24.04"
  },
  "nodeConfigs": {
    "workers": {
      "serverType": "cx23",
      "location": "hel1",
      "minSize": 1,
      "maxSize": 5
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `workers` | Node pool name (must match your cluster.yaml pool name) |
| `serverType` | Hetzner server type (cx11, cx21, cx23, cx31, cx41, etc.) |
| `location` | Hetzner datacenter (fsn1, nbg1, hel1, ash, hil) |
| `minSize` | Minimum nodes in the pool |
| `maxSize` | Maximum nodes the autoscaler can scale to |

### 4. Encode the Config

```bash
base64 -i cluster-autoscaler-config.json
```

### 5. Update the Deployment

```bash
ENCODED_CONFIG=$(base64 -i cluster-autoscaler-config.json | tr -d '\n')

kubectl set env deployment/cluster-autoscaler -n kube-system \
  HCLOUD_CLUSTER_CONFIG="$ENCODED_CONFIG"
```

### 6. Verify the Fix

```bash
# Check pod is running
kubectl get pods -n kube-system -l app=cluster-autoscaler

# Check logs for successful startup
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=20
```

Expected output:

```
NAME                                  READY   STATUS    RESTARTS   AGE
cluster-autoscaler-59d5867b67-cpk7g   1/1     Running   0          10s
```

## Multiple Node Pools

If you have multiple pools defined in your cluster:

```json
{
  "imagesForArch": {
    "arm64": "ubuntu-24.04",
    "amd64": "ubuntu-24.04"
  },
  "nodeConfigs": {
    "workers": {
      "serverType": "cx23",
      "location": "hel1",
      "minSize": 1,
      "maxSize": 5
    },
    "compute": {
      "serverType": "cx41",
      "location": "hel1",
      "minSize": 0,
      "maxSize": 3
    }
  }
}
```

## Common Hetzner Server Types

| Type | vCPUs | RAM | Disk | Use Case |
|------|-------|-----|------|----------|
| cx11 | 1 | 2 GB | 20 GB | Minimal workloads |
| cx21 | 2 | 4 GB | 40 GB | Small apps |
| cx23 | 2 | 4 GB | 40 GB | General purpose (newer gen) |
| cx31 | 2 | 8 GB | 80 GB | Medium workloads |
| cx41 | 4 | 16 GB | 160 GB | Compute-heavy |
| cx51 | 8 | 32 GB | 240 GB | Large workloads |

## Notes

- The node pool name in `nodeConfigs` must match the pool defined in your `cluster.yaml`
- Adjust `minSize`/`maxSize` based on your workload requirements
- Changes to the config require a pod restart (happens automatically with `kubectl set env`)
- If using base64 in a Secret instead of an env var, decode/re-encode the Secret data field
