---
tags:
  - Kubernetes
---
DaemonSets ensure one Pod runs on every node. Jobs run tasks to completion.

---

## DaemonSet

### Main idea

- runs exactly one Pod on every node in the cluster
- when a new node joins, a Pod is automatically created on it
- when a node is removed, the Pod is garbage collected
- you cannot scale a DaemonSet manually — it scales with the cluster

### DaemonSet use cases

- log collection agents (Fluentd, Filebeat)
- monitoring agents (Datadog, Node Exporter)
- network plugins (Calico, Cilium, Weave)
- storage drivers (Ceph, Longhorn)
- security agents

### DaemonSet manifest

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

### DaemonSet commands

```bash
kubectl get daemonsets
kubectl get daemonsets -n kube-system    # view system DaemonSets
kubectl describe daemonset <name>
```

---

## Job

### Main idea

- runs one or more Pods to completion
- tracks successful completions
- retries on failure up to `backoffLimit`
- once all completions succeed, the Job is done
- Pods are kept after completion (for log inspection) but not restarted

### Job manifest

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: my-app:1.0
          command: ["python", "manage.py", "migrate"]
```

### Job parameters

| Parameter | Meaning |
|---|---|
| `completions` | total successful runs required |
| `parallelism` | how many Pods run at the same time |
| `backoffLimit` | retries before Job is marked failed |
| `activeDeadlineSeconds` | max time before Job is terminated |

---

## CronJob

### Main idea

- creates Jobs on a schedule (like Linux cron)
- uses standard cron syntax
- manages Job history automatically

### CronJob manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"     # every day at 2am
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["/bin/backup.sh"]
```

### CronJob schedule examples

```
"0 * * * *"      every hour
"0 2 * * *"      every day at 2am
"0 2 * * 0"      every Sunday at 2am
"*/5 * * * *"    every 5 minutes
"0 0 1 * *"      first day of every month
```

### CronJob commands

```bash
kubectl get cronjobs
kubectl get jobs
kubectl describe cronjob <name>
# manually trigger a CronJob
kubectl create job --from=cronjob/<name> manual-run
```

---

## Common mistakes

- using a Deployment for one-time tasks instead of a Job
- not setting `backoffLimit` (Job retries forever by default)
- not setting `activeDeadlineSeconds` (stuck Jobs run forever)
- not setting `restartPolicy: OnFailure` or `Never` in Job Pods (required — `Always` is not allowed)
- not setting `successfulJobsHistoryLimit` (old Jobs accumulate)
- CronJob schedule in wrong timezone (Kubernetes uses UTC by default)

---

## Good practices

- use Jobs for database migrations, batch processing, one-time tasks
- use CronJobs for scheduled backups, cleanup tasks, reports
- set `activeDeadlineSeconds` to prevent runaway jobs
- set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` to keep history clean
- use DaemonSets for any per-node agent — never manually place Pods on nodes

---

## Must memorize

```
DaemonSet  → one Pod per node automatically
Job        → run to completion, retry on failure
CronJob    → Job on a schedule (cron syntax)
```

```bash
kubectl get daemonsets
kubectl get jobs
kubectl get cronjobs
kubectl create job --from=cronjob/<name> <job-name>
```

---

## Key ideas

- DaemonSets run one Pod per node automatically — used for cluster-wide agents.
- Jobs run tasks to completion with retry logic.
- CronJobs schedule Jobs using standard cron syntax.
- Always set restartPolicy, backoffLimit, and activeDeadlineSeconds for Jobs.
- DaemonSets are not scaled manually — they follow the cluster node count.
