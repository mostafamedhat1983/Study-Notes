---
tags:
  - Kubernetes
---
DaemonSets manage node-local workloads; Jobs manage finite tasks that run to completion; CronJobs create Jobs on a schedule.

## DaemonSet

A DaemonSet ensures that a Pod runs on each node that matches its scheduling rules. It does **not** necessarily mean every node in the cluster: node selectors, affinity, taints/tolerations, and other scheduling constraints can limit eligible nodes.

- When an eligible node is added, the DaemonSet controller creates a Pod for it.
- When a node is removed, the Pod associated with that node is garbage-collected.
- You normally change DaemonSet coverage through scheduling configuration and the cluster's eligible nodes, rather than setting a replica count.
- Common uses include log collectors, monitoring agents, network components, and node-level storage/security agents.

Example:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: example/agent:1.0
```

Commands:

```bash
kubectl get daemonsets -A
kubectl describe daemonset <name> -n <namespace>
```

## Job

A Job creates Pods and tracks successful completion of a finite task.

- `completions`: number of successful completions required.
- `parallelism`: maximum number of Pods running concurrently.
- `backoffLimit`: number of retries before the Job is considered failed. If omitted, the default is 6; it does not retry forever.
- `activeDeadlineSeconds`: optional time limit for the Job. When exceeded, the Job and its running Pods are terminated.
- Job Pods may remain after completion so their logs can be inspected; cleanup can be managed with TTL or other retention settings.

Example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: my-app:1.0
          command: ["python", "manage.py", "migrate"]
```

For Job Pods, `restartPolicy` must be `Never` or `OnFailure`; `Always` is not allowed. Retry behavior depends on this setting and the Job controller's backoff policy.

## CronJob

A CronJob creates Jobs according to a schedule. It uses cron syntax and supports controls such as concurrency policy, starting deadline, and Job history limits.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["/bin/backup.sh"]
```

The schedule's timezone should be explicit when timing matters. Use `.spec.timeZone` where supported; if it is omitted, the controller manager's local timezone is used. Do not assume it is always UTC.

Examples:

```text
0 * * * *      every hour
0 2 * * *      daily at 02:00 in the configured timezone
0 2 * * 0      Sundays at 02:00
*/5 * * * *    every five minutes
0 0 1 * *      first day of each month
```

Commands:

```bash
kubectl get cronjobs
kubectl get jobs
kubectl describe cronjob <name>
kubectl create job --from=cronjob/<name> manual-run
```

## Common mistakes

- Using a Deployment for a finite batch task when a Job is the appropriate controller.
- Assuming a Job retries forever: the default `backoffLimit` is 6.
- Omitting a sensible deadline for tasks that must not run indefinitely.
- Using `restartPolicy: Always` in a Job Pod template.
- Assuming a DaemonSet always runs on every node, regardless of taints or scheduling rules.
- Assuming CronJobs use UTC when no timezone is specified.

## Key ideas

- DaemonSet: one Pod on each eligible node.
- Job: run a finite task and track successful completions.
- CronJob: create Jobs on a schedule.
- Job retries are bounded by `backoffLimit`; deadlines and retention should be chosen for the workload.
