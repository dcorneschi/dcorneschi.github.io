<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

# Kubernetes emptyDir Volumes

## The Basics

```yaml
volumes:
- name: log-volume
  emptyDir: {}
```

An `emptyDir` volume is a temporary, empty directory created when a Pod is scheduled to a node. It exists for the lifetime of the Pod and is deleted when the Pod is removed.

## Why emptyDir?

It's the lightest-weight volume type. Ideal for transient data that only needs to exist while the Pod is running — like logs being shipped by a sidecar, scratch space for processing, or shared config between containers.

| Volume Type | Why Not (for transient data) |
|---|---|
| `hostPath` | Ties to a specific node, security concerns |
| `PersistentVolumeClaim` | Overkill for throwaway data, costs storage |
| `configMap` / `secret` | Read-only, not applicable for runtime data |

## Properties

- Automatically created when the Pod is scheduled.
- Shared across all containers in the Pod.
- Cleaned up when the Pod is deleted (no orphaned data).
- Fast — lives on the node's local disk by default.
- Survives container restarts (but not Pod deletion).

## Sharing Files Between Containers

Containers in a Pod have isolated filesystems by default. They share the network (same `localhost`, same IP) but not the disk. There's no implicit shared directory — you need a volume.

The volume name is what links containers, not the mount path. Each container can mount the same volume at a different path:

```yaml
containers:
- name: app
  volumeMounts:
  - name: log-volume
    mountPath: /app/logs        # app writes here

- name: log-agent
  volumeMounts:
  - name: log-volume
    mountPath: /fluentd/input   # fluentd reads from here

volumes:
- name: log-volume
  emptyDir: {}
```

## How Different Paths Share the Same Data

The `emptyDir` volume is a folder on the node, outside of both containers. Each container mounts that same folder to a different location inside its own filesystem:

```
Node disk:  /var/lib/kubelet/.../log-volume/
                    │
        ┌───────────┴───────────┐
        │                       │
   app container          log-agent container
   sees it at:            sees it at:
   /app/logs              /fluentd/input
```

When the app writes `/app/logs/output.log`, it's actually writing to the node-level `log-volume/output.log`. The log-agent sees that same file as `/fluentd/input/output.log`.

The `mountPath` is just an alias — where the volume appears inside that container's filesystem. The underlying data is the same.

## Actual Path on the Node

The `emptyDir` lives on the node at:

```
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>/
```

For example:

```
/var/lib/kubelet/pods/a1b2c3d4-e5f6-7890-abcd-ef1234567890/volumes/kubernetes.io~empty-dir/log-volume/
```

You don't control this path — kubelet manages it automatically. It creates it when the Pod starts and deletes it when the Pod is removed.

### Finding the Pod UID

```bash
# most direct way
kubectl get pod <pod-name> -o jsonpath='{.metadata.uid}'

# or from the yaml output, under metadata.uid
kubectl get pod <pod-name> -o yaml
```

## Memory-Backed emptyDir

By default, `emptyDir` uses the node's disk. You can switch to RAM by setting `medium: "Memory"`:

```yaml
volumes:
- name: log-volume
  emptyDir:
    medium: "Memory"
```

This creates a `tmpfs` mount — faster, but the data counts against the container's memory limit.

## What Happens Without a Volume?

If a `volumeMounts` entry references a volume name that doesn't exist in the `volumes:` list, the Pod will fail. It won't get past `ContainerCreating` — `kubectl describe pod` will show the mismatch.

The volume name in `volumeMounts.name` must match a `volumes.name` entry in the same Pod spec. No match, no start.

## The Two Steps

1. **Define the volume** at the Pod level under `volumes:` — give it a name and a type.
2. **Mount it in each container** under `volumeMounts:` — reference the same volume name and pick whatever `mountPath` you want.

That's it. The volume name is the glue.
