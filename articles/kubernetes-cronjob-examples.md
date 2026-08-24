# Kubernetes CronJob Examples & Reference

Related: [Cron vs CronJob in Kubernetes](articles/kubernetes-cron-vs-cronjob.md)

## Cron Schedule Syntax

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–6, Sunday=0)
│ │ │ │ │
* * * * *
```

### Common Schedule Examples

| Schedule         | Meaning                        |
|------------------|--------------------------------|
| `* * * * *`      | Every minute                   |
| `*/5 * * * *`    | Every 5 minutes                |
| `*/30 * * * *`   | Every 30 minutes               |
| `0 * * * *`      | Every hour (at minute 0)       |
| `0 */6 * * *`    | Every 6 hours                  |
| `0 2 * * *`      | Daily at 2:00 AM               |
| `0 9,17 * * *`   | Twice daily at 9 AM and 5 PM   |
| `0 0 * * 0`      | Weekly on Sunday at midnight   |
| `0 0 1 * *`      | First day of every month       |
| `0 0 1 1 *`      | Once a year (Jan 1 at midnight)|

## CronJob Spec Fields

### `spec` (CronJob level)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `schedule` | string | **required** | Cron expression defining when to run |
| `timeZone` | string | UTC | IANA timezone (e.g. `"America/New_York"`) — requires K8s 1.27+ |
| `concurrencyPolicy` | string | `Allow` | How to handle concurrent executions: `Allow`, `Forbid`, `Replace` |
| `suspend` | bool | `false` | If `true`, all subsequent executions are suspended |
| `successfulJobsHistoryLimit` | int | `3` | Number of successful finished jobs to keep |
| `failedJobsHistoryLimit` | int | `1` | Number of failed finished jobs to keep |
| `startingDeadlineSeconds` | int | unlimited | Deadline (seconds) for starting the job if it missed its scheduled time |

### `concurrencyPolicy` options

| Value | Behavior |
|-------|----------|
| `Allow` | Multiple jobs can run simultaneously (default) |
| `Forbid` | Skip the new run if the previous one is still active |
| `Replace` | Cancel the currently running job and start a new one |

### `jobTemplate.spec` (Job level)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `backoffLimit` | int | `6` | Number of retries before marking the job as failed |
| `activeDeadlineSeconds` | int | unlimited | Maximum time (seconds) the job is allowed to run |
| `ttlSecondsAfterFinished` | int | none | Auto-delete the job and its pods after this many seconds |
| `completions` | int | `1` | Number of pods that need to successfully complete |
| `parallelism` | int | `1` | Number of pods to run in parallel |

### `jobTemplate.spec.template.spec` (Pod level)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `restartPolicy` | string | **required** | Must be `OnFailure` or `Never` (cannot be `Always` for Jobs) |
| `nodeSelector` | map | none | Schedule the pod on specific nodes |
| `tolerations` | list | none | Allow scheduling on tainted nodes |
| `serviceAccountName` | string | `default` | Service account for RBAC permissions |
| `volumes` | list | none | Volumes to mount into containers |

### Container-level fields

| Field | Type | Description |
|-------|------|-------------|
| `image` | string | Container image to use |
| `command` | list | Entrypoint override |
| `args` | list | Arguments to the entrypoint |
| `env` | list | Environment variables |
| `envFrom` | list | Import env vars from ConfigMaps/Secrets |
| `resources.requests` | map | Minimum CPU/memory guaranteed |
| `resources.limits` | map | Maximum CPU/memory allowed |
| `volumeMounts` | list | Where to mount volumes inside the container |

## Examples

### 1. Basic "Hello World" — every minute

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-world
  namespace: default
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ["/bin/sh", "-c", "echo Hello from CronJob at $(date)"]
          restartPolicy: OnFailure
```

### 2. Database Backup — daily at 2 AM

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: default
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      activeDeadlineSeconds: 600
      backoffLimit: 2
      template:
        spec:
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backups/dump-$(date +%F).sql
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_NAME
              value: "myapp"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backups
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 500m
                memory: 512Mi
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
```

### 3. Cache Cleanup — every 6 hours

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cache-cleanup
  namespace: default
spec:
  schedule: "0 */6 * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 120
      template:
        spec:
          containers:
          - name: cleanup
            image: redis:7-alpine
            command:
            - /bin/sh
            - -c
            - redis-cli -h redis-service FLUSHDB
            resources:
              requests:
                cpu: 50m
                memory: 64Mi
              limits:
                cpu: 100m
                memory: 128Mi
          restartPolicy: OnFailure
```

### 4. HTTP Health Check — every 30 minutes

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-report
  namespace: monitoring
spec:
  schedule: "*/30 * * * *"
  concurrencyPolicy: Replace
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          containers:
          - name: healthcheck
            image: curlimages/curl:8.5.0
            command:
            - /bin/sh
            - -c
            - |
              STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://my-app-service/health)
              echo "Health check returned: $STATUS"
              if [ "$STATUS" != "200" ]; then
                echo "UNHEALTHY - expected 200, got $STATUS"
                exit 1
              fi
              echo "OK"
            resources:
              requests:
                cpu: 25m
                memory: 32Mi
              limits:
                cpu: 50m
                memory: 64Mi
          restartPolicy: OnFailure
```

### 5. Log Rotation — weekly on Sunday at midnight

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-rotation
  namespace: default
spec:
  schedule: "0 0 * * 0"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: log-cleaner
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - |
              echo "Removing logs older than 30 days..."
              find /logs -type f -mtime +30 -delete
              echo "Done. Remaining files:"
              ls -lh /logs
            volumeMounts:
            - name: log-volume
              mountPath: /logs
            resources:
              requests:
                cpu: 50m
                memory: 64Mi
              limits:
                cpu: 100m
                memory: 128Mi
          restartPolicy: OnFailure
          volumes:
          - name: log-volume
            persistentVolumeClaim:
              claimName: app-logs-pvc
```

## Full Annotated Template

A complete CronJob manifest with all common fields annotated:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
  namespace: default
  labels:
    app: my-app
spec:
  # --- CronJob-level settings ---
  schedule: "0 2 * * *"                  # When to run (cron syntax)
  timeZone: "Europe/Berlin"              # Optional: IANA timezone (K8s 1.27+)
  concurrencyPolicy: Forbid              # Allow | Forbid | Replace
  suspend: false                         # Set to true to pause all runs
  startingDeadlineSeconds: 200           # How late a job can start after its schedule
  successfulJobsHistoryLimit: 3          # Keep last N successful job objects
  failedJobsHistoryLimit: 1              # Keep last N failed job objects

  jobTemplate:
    spec:
      # --- Job-level settings ---
      backoffLimit: 2                    # Retries before marking job as failed
      activeDeadlineSeconds: 300         # Kill the job if it runs longer than this
      ttlSecondsAfterFinished: 60        # Auto-delete job+pods after completion
      completions: 1                     # Number of successful completions needed
      parallelism: 1                     # Number of pods running in parallel

      template:
        metadata:
          labels:
            app: my-app
        spec:
          # --- Pod-level settings ---
          restartPolicy: OnFailure       # OnFailure or Never (not Always)
          serviceAccountName: my-sa      # For RBAC permissions
          nodeSelector:
            workload: batch              # Schedule on specific nodes
          tolerations:
          - key: "dedicated"
            operator: "Equal"
            value: "batch"
            effect: "NoSchedule"

          containers:
          - name: worker
            image: my-registry/my-image:1.0.0
            command: ["/bin/sh", "-c"]
            args: ["echo Running task && do-work"]

            env:
            - name: ENV_VAR
              value: "hardcoded-value"
            - name: SECRET_VAR
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: api-key
            - name: CONFIG_VAR
              valueFrom:
                configMapKeyRef:
                  name: my-config
                  key: setting

            resources:
              requests:
                cpu: 100m
                memory: 128Mi
              limits:
                cpu: 500m
                memory: 512Mi

            volumeMounts:
            - name: data
              mountPath: /data

          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: my-pvc
```

## Troubleshooting

### CronJob never fires

```bash
# Check if the CronJob is suspended
kubectl get cronjob my-cronjob -o jsonpath='{.spec.suspend}'

# Check events for scheduling issues
kubectl describe cronjob my-cronjob | grep -A 10 Events

# Verify the schedule is valid (common mistake: wrong field order)
kubectl get cronjob my-cronjob -o jsonpath='{.spec.schedule}'
```

> If `startingDeadlineSeconds` is set and the controller was down longer than that window, missed schedules are permanently skipped. If more than 100 schedules are missed, the CronJob stops trying entirely and logs: `Cannot determine if job needs to be started. Too many missed start times.`

### Job pods stuck in CrashLoopBackOff

```bash
# Check pod logs for the failing job
kubectl logs -l job-name=my-cronjob-28456123

# Check pod events
kubectl describe pod <pod-name> | grep -A 20 Events

# Check if it's an OOMKill
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

Common causes:
- **Exit code 1** — Script error. Check logs.
- **Exit code 137** — OOMKilled. Increase `resources.limits.memory`.
- **Exit code 126/127** — Command not found or not executable. Check `command` and `image`.

### Jobs pile up (not being cleaned)

```bash
# Check how many jobs exist for a cronjob
kubectl get jobs --selector=job-name -l cronjob-name=my-cronjob | wc -l

# Force cleanup — delete completed jobs older than 1 hour
kubectl delete jobs --field-selector status.successful=1
```

Fix: Set `ttlSecondsAfterFinished` on the job spec, or lower `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`.

### Job runs but does nothing

- Verify `command` is correct — YAML list syntax is easy to get wrong
- Check if the container image has the tools you expect (`sh`, `curl`, `pg_dump`, etc.)
- Test manually: `kubectl create job --from=cronjob/my-cronjob test-run` then check logs

## Useful kubectl Commands

```bash
# List all cronjobs
kubectl get cronjobs

# Describe a specific cronjob
kubectl describe cronjob my-cronjob

# View jobs created by a cronjob
kubectl get jobs --selector=job-name=my-cronjob

# Manually trigger a cronjob (create a job from it)
kubectl create job --from=cronjob/my-cronjob manual-run-001

# Suspend a cronjob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":true}}'

# Resume a cronjob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":false}}'

# Delete a cronjob (also deletes child jobs and pods)
kubectl delete cronjob my-cronjob

# View logs from the most recent job pod
kubectl logs job/my-cronjob-28456123
```

## Delete Multiple Failed Jobs from a Namespace

### Delete All Failed Jobs

```bash
kubectl delete jobs --field-selector status.successful=0 -n <namespace>
```

### List First, Then Delete (Safer)

```bash
# List failed jobs
kubectl get jobs --field-selector status.successful=0 -n <namespace>

# Delete them
kubectl delete jobs --field-selector status.successful=0 -n <namespace>
```

### Using jsonpath to Target Specifically Failed Jobs

```bash
kubectl get jobs -n <namespace> -o jsonpath='{.items[?(@.status.failed>=1)].metadata.name}' | xargs kubectl delete job -n <namespace>
```

### Delete Failed Jobs Using jq (More Precise)

```bash
kubectl get jobs -n <namespace> -o json | jq -r '.items[] | select(.status.conditions[]? | select(.type=="Failed" and .status=="True")) | .metadata.name' | xargs kubectl delete job -n <namespace>
```

### Delete Jobs from a Specific Day

Delete failed jobs created on a specific date:

```bash
kubectl get jobs -n <namespace> -o json | jq -r '.items[] | select(.metadata.creationTimestamp | startswith("2025-06-15")) | select(.status.conditions[]? | select(.type=="Failed" and .status=="True")) | .metadata.name' | xargs kubectl delete job -n <namespace>
```

Delete all jobs (regardless of status) from a specific date:

```bash
kubectl get jobs -n <namespace> -o json | jq -r '.items[] | select(.metadata.creationTimestamp | startswith("2025-06-15")) | .metadata.name' | xargs kubectl delete job -n <namespace>
```

List jobs from a specific date before deleting:

```bash
kubectl get jobs -n <namespace> -o json | jq -r '.items[] | select(.metadata.creationTimestamp | startswith("2025-06-15")) | "\(.metadata.name) \(.metadata.creationTimestamp)"'
```

> Replace `2025-06-15` with the target date (format: `YYYY-MM-DD`). The `creationTimestamp` is stored as ISO 8601, so `startswith` works cleanly for date filtering.

### Prevent Accumulation with TTL

Set `ttlSecondsAfterFinished` on your Job spec to auto-clean completed/failed jobs:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  ttlSecondsAfterFinished: 3600  # auto-delete 1 hour after completion
  template:
    spec:
      containers:
      - name: my-container
        image: my-image
      restartPolicy: Never
```

The `ttlSecondsAfterFinished` field works for both successful and failed jobs. The TTL controller will clean them up automatically.
