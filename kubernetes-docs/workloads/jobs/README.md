# Jobs & CronJobs — Batch Workloads

---

## Job vs CronJob vs Deployment

| | Job | CronJob | Deployment |
|---|---|---|---|
| Purpose | Run a task to completion | Run a Job on a schedule | Keep N replicas running forever |
| Restart on success | No | No | Yes |
| Restart on failure | Yes (configurable) | Yes | Yes |
| Scheduling | Once (manual trigger) | Cron expression | Continuous |
| Use case | DB migration, data export | Nightly backups, reports | Web servers, APIs |

---

## Job

A Job creates one or more Pods and ensures a specified number complete successfully.

### Minimal Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
spec:
  template:
    spec:
      restartPolicy: OnFailure    # required: OnFailure or Never (not Always)
      containers:
        - name: migration
          image: my-app:1.0
          command: ["./migrate", "--up"]
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db_url
```

### Job Spec Fields

```yaml
spec:
  completions: 1           # total successful completions needed (default: 1)
  parallelism: 1           # Pods running simultaneously (default: 1)
  backoffLimit: 4          # retries before marking Job as failed (default: 6)
  activeDeadlineSeconds: 600   # kill Job if not done within 10 min
  ttlSecondsAfterFinished: 300 # auto-delete Job+Pods 5 min after completion
```

### restartPolicy Options

| Value | Behavior |
|---|---|
| `OnFailure` | Restart the same Pod on failure — simpler, but Pod history is lost |
| `Never` | Create a new Pod on failure — logs preserved per attempt; use with low `backoffLimit` |

> Use `Never` when you need logs from failed attempts. Use `OnFailure` for simple retries.

---

## Parallel Jobs

### Fixed Completion Count

Run N independent tasks total, M at a time.

```yaml
spec:
  completions: 10     # run 10 tasks total
  parallelism: 3      # 3 at a time
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: my-worker:1.0
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

Each Pod receives a unique `JOB_COMPLETION_INDEX` (0-based) — use this to partition work.

### Work Queue (indexed)

```yaml
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed   # each Pod gets a unique index
```

### Unstructured Parallelism (work queue via external queue)

```yaml
spec:
  parallelism: 5       # run 5 workers
  # No completions set — Job succeeds when at least 1 Pod succeeds
  # Workers pull from SQS/Redis and exit when queue is empty
```

---

## CronJob

A CronJob creates a Job on a recurring schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
  namespace: production
spec:
  schedule: "0 2 * * *"         # 2:00 AM UTC every day
  timeZone: "America/New_York"  # k8s 1.27+ (otherwise use UTC offset)
  concurrencyPolicy: Forbid     # don't start new Job if previous is still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 300  # skip if missed start by more than 5 min

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600   # kill if running more than 1 hour
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: backup-sa
          containers:
            - name: backup
              image: my-backup:1.0
              command: ["./backup.sh"]
              env:
                - name: S3_BUCKET
                  value: "my-backups"
              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
```

### Cron Expression Reference

```
┌──── minute       (0–59)
│  ┌─── hour       (0–23)
│  │  ┌── day of month  (1–31)
│  │  │  ┌─ month       (1–12)
│  │  │  │  ┌ day of week  (0–6, Sun=0)
│  │  │  │  │
*  *  *  *  *

Examples:
"0 2 * * *"       → 2:00 AM every day
"*/15 * * * *"    → every 15 minutes
"0 9 * * 1-5"     → 9:00 AM Monday–Friday
"0 0 1 * *"       → midnight on the 1st of each month
"0 */6 * * *"     → every 6 hours
```

### concurrencyPolicy

| Value | Behavior |
|---|---|
| `Allow` | Multiple Jobs can run simultaneously (default) |
| `Forbid` | Skip new Job if previous is still running |
| `Replace` | Cancel running Job and start a new one |

> Use `Forbid` for jobs that must not overlap (backups, reports). Use `Replace` for jobs that should always represent the latest run.

---

## Job Patterns

### Database Migration (one-shot, pre-deploy)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate-v1-3-0
  namespace: production
  annotations:
    helm.sh/hook: pre-upgrade
    helm.sh/hook-weight: "-1"
    helm.sh/hook-delete-policy: hook-succeeded
spec:
  backoffLimit: 0               # fail fast — don't retry migrations
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: my-app:1.3.0
          command: ["./migrate", "--up", "--validate"]
```

### Data Export to S3

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: export-report-2024-01
  namespace: production
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400   # keep for 24h
  template:
    spec:
      restartPolicy: OnFailure
      serviceAccountName: s3-export-sa    # IRSA with S3 write permissions
      containers:
        - name: exporter
          image: my-reporter:1.0
          command: ["python", "export.py", "--month=2024-01"]
          env:
            - name: S3_BUCKET
              value: "reports-bucket"
            - name: AWS_REGION
              value: "us-east-1"
```

---

## Commands

```bash
# Run a Job manually
kubectl create job manual-backup --from=cronjob/nightly-backup -n production

# List Jobs and their status
kubectl get jobs -n production

# Watch Job completion
kubectl get jobs -n production -w

# View Job Pods and logs
kubectl get pods -n production -l job-name=nightly-backup
kubectl logs -l job-name=nightly-backup -n production

# Describe a failed Job
kubectl describe job db-migration -n production

# List CronJobs
kubectl get cronjobs -n production

# Suspend a CronJob (pause scheduling)
kubectl patch cronjob nightly-backup -p '{"spec":{"suspend":true}}' -n production

# Resume a CronJob
kubectl patch cronjob nightly-backup -p '{"spec":{"suspend":false}}' -n production

# Delete completed Jobs (cleanup)
kubectl delete jobs -n production --field-selector status.successful=1
```
