# HPA ScalingLimited (TooManyReplicas)

Troubleshooting an HPA that has hit its `maxReplicas` ceiling and can no longer
meet its CPU target.

## What the Alert Means

The HPA condition `ScalingLimited` is `True` with reason `TooManyReplicas`. The autoscaler has calculated it needs more replicas than `maxReplicas` allows, so it's stuck at the ceiling and can't meet the CPU target.

## Diagnosis

```bash
# Check HPA status
kubectl describe hpa <hpa-name> -n <namespace>

# Look for:
# - ScalingLimited condition = True
# - Current CPU % close to or above target
# - current replicas == max replicas
```

### Key Fields to Check

| Field | What to Look For |
|-------|-----------------|
| `ScalingLimited` | `True` + reason `TooManyReplicas` |
| CPU current/target | Current near or above target % |
| Deployment pods | `current == max replicas` |
| Events | Repeated `SuccessfulRescale` to max |

## Risk

When the HPA is capped:

- Further load increases push CPU higher with no scaling headroom
- Increased latency and potential 5xx errors
- Possible OOMKills if memory follows CPU pressure

## Resolution Options

### 1. Increase maxReplicas (quickest fix)

```bash
kubectl patch hpa <hpa-name> -n <namespace> -p '{"spec":{"maxReplicas":25}}'
```

Ensure the cluster has enough node capacity and available IPs in the subnet.

### 2. Raise CPU target utilization (e.g., 60% → 70%)

```bash
kubectl patch hpa <hpa-name> -n <namespace> \
  -p '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":70}}}]}}'
```

This lets each pod absorb more load before triggering scale-up.

### 3. Increase CPU requests per pod

If pods can handle more work, increase the resource request so fewer replicas are needed. Update the deployment spec and roll out.

### 4. Optimize the application

Profile CPU usage to find hotspots. Reduce unnecessary computation or add caching.

## Datadog Monitor Details

The alert likely monitors one of:

```text
kubernetes_state.hpa.condition{condition:ScalingLimited, status:true}
```

or checks that current replicas have hit max:

```text
kubernetes_state.hpa.max_replicas - kubernetes_state.hpa.current_replicas == 0
```

## Auto-Resolution

Once the HPA is no longer capped (e.g., after increasing `maxReplicas` or reducing load), the `ScalingLimited` condition flips to `False` and the Datadog alert auto-resolves.

## Example Incident (hsr-api)

```text
Namespace:    hsr-api-prd
HPA:          hsr-api
CPU:          57% (2294m) / 60% target
Replicas:     20 current / 20 max
Condition:    ScalingLimited = True (TooManyReplicas)
```

Resolution: Increased maxReplicas from 20 → 25 to provide scaling headroom.

## Skills Practiced

- Reading HPA conditions and interpreting `ScalingLimited` / `TooManyReplicas`
- Diagnosing a capped autoscaler with `kubectl describe hpa`
- Patching `maxReplicas` and CPU target utilization live
- Correlating a Datadog HPA monitor with cluster state and auto-resolution
