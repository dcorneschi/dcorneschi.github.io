# Cron vs CronJob in Kubernetes

Related: [Kubernetes CronJob Examples & Reference](articles/kubernetes-cronjob-examples.md)

In Kubernetes there's only one resource: **CronJob** (`batch/v1`). There's no separate "Cron" object.

## CronJob

- The Kubernetes resource that runs **Jobs on a schedule** (cron syntax).
- It creates a new **Job** object each time the schedule fires.
- API kind: `CronJob`, group: `batch/v1`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command: ["/bin/sh", "-c", "echo cleanup"]
          restartPolicy: OnFailure
```

## Job

- A one-shot workload that runs to completion.
- Created manually or automatically by a CronJob.
- API kind: `Job`, group: `batch/v1`

## The Relationship

```
CronJob (scheduled) → creates → Job (one-time) → creates → Pod(s)
```

## Key Differences from Linux cron

| Aspect | Linux cron | Kubernetes CronJob |
|--------|-----------|-------------------|
| Runs on | A single host | Cluster-wide (scheduled by kube-scheduler) |
| Failure handling | Silent unless you set up mail/logging | Built-in: `backoffLimit`, `activeDeadlineSeconds`, status conditions |
| Concurrency | Up to you | `concurrencyPolicy`: Allow, Forbid, Replace |
| History | Only logs | `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` keeps old Job objects |
| Missed schedules | Runs next time | `startingDeadlineSeconds` controls what happens if schedule is missed |

## Common Confusion

- **"cron"** — people use this informally to mean "scheduled task in Kubernetes," but the actual resource is always `CronJob`.
- **"Job"** — the child resource that a CronJob spawns. You can also create Jobs independently for one-off tasks.

So if someone says "create a cron," they mean create a `CronJob`. There's no `kind: Cron` in Kubernetes.
