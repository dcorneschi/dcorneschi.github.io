# Kubernetes Job and CronJob Cheatsheet

## CronJob Commands

```sh
# List all cronjobs
kubectl get cronjobs

# Describe a cronjob (schedule, history, events)
kubectl describe cronjob <name>

# Create a cronjob from a YAML file
kubectl apply -f cronjob.yaml

# Delete a cronjob (also deletes its jobs and pods)
kubectl delete cronjob <name>

# Suspend a cronjob (stops scheduling new jobs)
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'

# Resume a suspended cronjob
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

# Manually trigger a job from a cronjob
kubectl create job <job-name> --from=cronjob/<cronjob-name>
```

## Job Commands

```sh
# List all jobs
kubectl get jobs

# Describe a job
kubectl describe job <name>

# Get logs from a job's pod
kubectl logs job/<name>

# Delete a specific job
kubectl delete job <name>

# Delete all completed jobs
kubectl delete jobs --field-selector status.successful=1

# Delete all failed jobs
kubectl delete jobs --field-selector status.successful=0

# Watch jobs in real time
kubectl get jobs --watch

# Delete all completed (Succeeded) pods manually
kubectl delete pods --field-selector=status.phase==Succeeded
```

## Cron Schedule Syntax

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *
```

| Schedule | Meaning |
|----------|---------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Every day at midnight |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 9 1 * *` | 1st of every month at 9am |

## Edit a CronJob Schedule (Live Cluster)

```sh
# Patch (one-liner, takes effect immediately)
kubectl patch cronjob <name> -p '{"spec":{"schedule":"0 */2 * * *"}}'

# Edit interactively (opens in default editor)
kubectl edit cronjob <name>
```

No need to delete or recreate — changes apply instantly.

## Key CronJob Spec Fields

| Field | Description |
|-------|-------------|
| `schedule` | Cron expression for when to run |
| `concurrencyPolicy` | `Allow` (default), `Forbid`, or `Replace` |
| `suspend` | `true` to pause scheduling |
| `successfulJobsHistoryLimit` | Number of successful jobs to keep (default: 3) |
| `failedJobsHistoryLimit` | Number of failed jobs to keep (default: 1) |
| `startingDeadlineSeconds` | Max seconds late a job can start |

## Key Job Spec Fields

| Field | Description |
|-------|-------------|
| `backoffLimit` | Number of retries before marking as failed (default: 6) |
| `activeDeadlineSeconds` | Max runtime before job is killed |
| `completions` | Number of pods that must succeed (default: 1) |
| `parallelism` | Number of pods running in parallel (default: 1) |
| `ttlSecondsAfterFinished` | Auto-delete job after N seconds |

## Concurrency Policies

- **Allow** — Multiple jobs can run at the same time (default)
- **Forbid** — Skip the new run if the previous one is still active
- **Replace** — Kill the current job and start a new one

## Why CrashLoopBackOff Pods Pile Up

`failedJobsHistoryLimit` only controls how many **finished** failed Job objects are kept in history. It does NOT limit pods from a **currently active** job that's still retrying.

### What Happens

1. CronJob fires → creates a Job
2. Pod fails → `restartPolicy: OnFailure` → kubelet restarts the container in the same pod
3. Container keeps crashing → exponential backoff → `CrashLoopBackOff`
4. Job hasn't hit `backoffLimit` yet, so it's still **active** (not "failed")
5. CronJob fires again → creates another Job (if `concurrencyPolicy: Allow`)
6. Multiple active Jobs with pods in CrashLoopBackOff

### The Fix

```yaml
spec:
  concurrencyPolicy: Forbid          # Don't start new jobs while one is active
  jobTemplate:
    spec:
      backoffLimit: 2                 # Fail the job after 2 retries
      activeDeadlineSeconds: 60       # Kill the job if it runs too long
      template:
        spec:
          restartPolicy: Never        # Don't restart in-place, create a new pod instead
```

### Key Distinction

| Setting | Controls |
|---------|----------|
| `failedJobsHistoryLimit` | How many *finished* failed Jobs to keep in history |
| `backoffLimit` | How many times the Job retries before being marked as failed |
| `restartPolicy: OnFailure` | Kubelet restarts the container in the same pod (causes CrashLoopBackOff) |
| `restartPolicy: Never` | Job controller creates a new pod for each retry (cleaner, no CrashLoopBackOff) |
| `concurrencyPolicy: Forbid` | Prevents stacking multiple active jobs |

## Troubleshooting

```sh
# See why a cronjob isn't firing
kubectl describe cronjob <name> | grep -A5 "Events"

# Check if a job is stuck
kubectl get pods --selector=job-name=<job-name>

# See pod status for a failed job
kubectl describe pod <pod-name>

# Force delete a stuck job
kubectl delete job <name> --grace-period=0 --force
```

## Tips

- Use `ttlSecondsAfterFinished` to auto-cleanup old jobs without relying on history limits
- Set `activeDeadlineSeconds` to prevent runaway jobs
- Use `concurrencyPolicy: Forbid` for jobs that shouldn't overlap
- The minimum CronJob interval is 1 minute (`* * * * *`)
- Use `kubectl create job --from=cronjob/<name>` to test without waiting for the schedule
