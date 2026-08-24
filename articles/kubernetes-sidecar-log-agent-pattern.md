# Sidecar Log Agent Pattern

## The Snippet

```yaml
      containers:
      - name: app
        image: myapp:1.0
        volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

      # Sidecar container
      - name: log-agent
        image: fluentd:v1.16
        volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

      volumes:
      - name: log-volume
        emptyDir: {}
```

## What It Does

This adds a sidecar container (`log-agent` running Fluentd) alongside the main application container in the same Pod. Both containers share a volume called `log-volume` backed by an `emptyDir`.

- The app container writes logs to `/var/log/app`.
- The `log-agent` sidecar reads from that same path via the shared volume.
- Fluentd then forwards/processes those logs to a central logging system.

## Why `emptyDir`?

`emptyDir` is used because the logs are transient — they only need to exist long enough for Fluentd to read and forward them. There's no need to persist them beyond the Pod's lifetime.

| Volume Type | Why Not |
|---|---|
| `hostPath` | Ties to a specific node, security concerns |
| `PersistentVolumeClaim` | Overkill for throwaway log data |
| `configMap` / `secret` | Read-only, not applicable |

`emptyDir` is:

- Automatically created when the Pod is scheduled.
- Shared across all containers in the Pod.
- Cleaned up when the Pod is deleted (no orphaned data).
- Fast — lives on the node's local disk (or memory if `medium: "Memory"` is set).

## How It Works

1. **Define the volume** at the Pod level under `volumes:` — give it a name and a type (e.g., `emptyDir`).
2. **Mount it in each container** under `volumeMounts:` — reference the same volume name and choose whatever `mountPath` you want.

The `mountPath` can be different in each container. The volume name is what links them, not the path.

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

Both containers see the same data — they're just accessing it from different paths inside their own filesystem. The volume name (`log-volume`) is the glue; the mount paths are just where each container wants to "see" that shared directory.

## How Different Paths Share the Same Data

Containers in a Pod have isolated filesystems by default. They share the network (same `localhost`, same IP) but not the disk. You need a volume to share files — there's no implicit shared directory.

The `emptyDir` volume is a folder that lives on the node, outside of both containers. Each container mounts that same folder to a different location inside its own filesystem:

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

The `mountPath` is just an alias — it's where the volume appears inside that specific container's filesystem. The underlying data is the same.

### Actual Path on the Node

The `emptyDir` volume lives on the node at:

```
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>/
```

For example:

```
/var/lib/kubelet/pods/a1b2c3d4-e5f6-7890-abcd-ef1234567890/volumes/kubernetes.io~empty-dir/log-volume/
```

You don't control or choose this path — kubelet manages it automatically. It creates it when the Pod starts and deletes it when the Pod is removed. You never reference this node path in your configs; the `mountPath` in each container is the only path you work with.

If you set `medium: "Memory"` on the `emptyDir`, it uses a `tmpfs` mount instead of disk — data lives in RAM (faster, but counts against the container's memory limit).

## Init Containers vs Sidecars

Init containers run before any regular containers start. They're sequential — each must complete before the next begins, and all must finish before the main containers launch.

Common init container use cases: waiting for a dependency, downloading config, running migrations, setting file permissions.

```yaml
initContainers:
- name: init-config
  image: busybox
  command: ['sh', '-c', 'wget -O /config/app.conf http://config-server/config']
  volumeMounts:
  - name: config-volume
    mountPath: /config

containers:
- name: app
  image: myapp:1.0
  volumeMounts:
  - name: config-volume
    mountPath: /config
```

| | Init Containers | Sidecar Containers |
|---|---|---|
| Spec location | `initContainers:` | `containers:` (same as main app) |
| When they run | Before main containers start | Alongside main containers |
| Lifecycle | Run to completion, then exit | Run for the entire Pod lifetime |
| Order | Sequential (one after another) | All start together in parallel |

There's no special `sidecar:` field in the spec. A sidecar is just a regular container that plays a supporting role — Kubernetes doesn't distinguish between "main" and "sidecar."

## Native Sidecars (Kubernetes 1.28+)

Since Kubernetes 1.28, you can set `restartPolicy: Always` on an init container to make it a native sidecar:

```yaml
spec:
  initContainers:
  - name: log-agent
    image: fluentd:v1.16
    restartPolicy: Always    # this makes it a native sidecar
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app

  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app

  volumes:
  - name: log-volume
    emptyDir: {}
```

This gives you startup and shutdown ordering guarantees:

- Starts during the init phase but doesn't block the next containers.
- Keeps running for the entire Pod lifetime.
- Guaranteed to start before the main containers.
- Guaranteed to be the last to shut down when the Pod terminates.

| Approach | Where | Startup order | Shutdown order |
|---|---|---|---|
| Old-style sidecar | `containers:` | No guarantee | No guarantee |
| Native sidecar (1.28+) | `initContainers:` + `restartPolicy: Always` | Starts before main | Stops after main |

If you're on 1.28+, the native sidecar approach is preferred for log agents, service meshes, and proxies.

## Tip

Avoid `fluentd:latest` in production. Pin to a specific tag (e.g., `fluentd:v1.16`) to prevent unexpected breaking changes on image pulls.
