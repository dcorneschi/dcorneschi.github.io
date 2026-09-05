# Kubectl: Time Required for a Pod to Reach Running/Ready State

## Quick Check — When Did the Pod Become Ready?

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastTransitionTime}'
```

## Time Delta — Seconds from Creation to Ready

### Using jq (cleanest, cross-platform)

```bash
kubectl get pod <pod-name> -o json | jq -r '(((.status.conditions[] | select(.type=="Ready")).lastTransitionTime | fromdateiso8601) - (.metadata.creationTimestamp | fromdateiso8601)) | "\(.)s to reach Ready"'
```

This uses `jq`'s built-in `fromdateiso8601` to do the date math directly — no shell date parsing needed.

### Using jsonpath + awk (macOS)

```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.creationTimestamp} {.status.conditions[?(@.type=="Ready")].lastTransitionTime}' | awk '{cmd="date -j -f %Y-%m-%dT%H:%M:%SZ "$1" +%s"; cmd | getline t1; close(cmd); cmd="date -j -f %Y-%m-%dT%H:%M:%SZ "$2" +%s"; cmd | getline t2; close(cmd); print t2-t1 "s"}'
```

## All Pods in a Namespace — Time to Ready

```bash
kubectl get pods -o json | jq -r '
  .items[]
  | select(.status.conditions[]? | select(.type=="Ready" and .status=="True"))
  | .metadata.name + ": " + (
      ((.status.conditions[] | select(.type=="Ready")).lastTransitionTime | fromdateiso8601)
      - (.metadata.creationTimestamp | fromdateiso8601)
    | tostring) + "s"'
```

## Notes

- The **Ready** condition means the pod is eligible to receive traffic through Services — all its containers are ready (each has passed its readiness probe, or has no probe) and any configured readiness gates are satisfied. "Ready" gates Service endpoint membership; it does not by itself prove the pod is actively serving traffic.
- A container with no `readinessProbe` is considered ready as soon as it is running, so `Ready` can flip `True` without any probe having run.
- **Readiness gates** (`spec.readinessGates`) can delay the Ready condition: even with all containers ready, the pod-level `Ready` stays `False` until every gate condition is `True`. Since the delta measured here ends at the Ready condition's `lastTransitionTime`, gates directly affect the number you get.
- If you care about when containers actually started (vs. ready to serve), check `.status.containerStatuses[].state.running.startedAt` instead.
- For pods that never became ready, the condition won't exist or `.status` will be `"False"`.
