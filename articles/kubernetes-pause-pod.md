# How to Pause a Pod in Kubernetes

There is no native "pause" concept for a Kubernetes pod, but here are the common approaches depending on what you're trying to achieve.

## 1. Scale the Deployment to 0 (stop the pod entirely)

```bash
kubectl scale deployment <deployment-name> -n <namespace> --replicas=0
```

Resume later:

```bash
kubectl scale deployment <deployment-name> -n <namespace> --replicas=1
```

## 2. Suspend a CronJob

If it's a CronJob, suspend it so no new Jobs are created:

```bash
kubectl patch cronjob <cronjob-name> -n <namespace> -p '{"spec":{"suspend":true}}'
```

Resume:

```bash
kubectl patch cronjob <cronjob-name> -n <namespace> -p '{"spec":{"suspend":false}}'
```

## 3. Freeze a Running Container (container runtime level)

Pause the process inside a container (like `docker pause`) at the node level:

```bash
# SSH into the node, then:
crictl pause <container-id>
```

Resume:

```bash
crictl unpause <container-id>
```

> **Note:** This freezes the process (SIGSTOP) without killing the pod. Kubernetes is not aware of this state, so liveness probes may fail and restart the pod.

## Summary

| Goal | Method |
|------|--------|
| Stop workload temporarily | Scale to 0 |
| Prevent scheduled runs | Suspend CronJob |
| Freeze process in-place (advanced) | `crictl pause` on node |

Scaling to 0 is the most common and safest "pause" in Kubernetes.
