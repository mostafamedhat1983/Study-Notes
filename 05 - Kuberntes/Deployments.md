---
tags:
  - Kubernetes
---
A Deployment manages a set of identical Pods and ensures the desired number of replicas is always running.

---

## Main idea

- a Deployment is the standard way to run stateless applications in Kubernetes
- it creates and manages a ReplicaSet, which in turn manages Pods
- you describe the desired state; the Deployment controller makes it happen
- it handles rolling updates and rollbacks automatically

---

## Deployment hierarchy

```
Deployment
  └── ReplicaSet
        └── Pod
        └── Pod
        └── Pod
```

- you manage the Deployment
- the Deployment manages ReplicaSets
- ReplicaSets manage Pods
- you almost never touch ReplicaSets directly

---

## Basic Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

---

## Update strategies

### RollingUpdate (default)

Gradually replaces old Pods with new ones.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # max pods that can be unavailable during update
      maxSurge: 1         # max extra pods that can be created during update
```

- zero downtime when configured correctly
- both old and new versions run temporarily at the same time
- requires readiness probes to work reliably

### Recreate

Kills all old Pods first, then creates new ones.

```yaml
spec:
  strategy:
    type: Recreate
```

- causes downtime during the update
- useful when old and new versions cannot run at the same time

---

## Common Deployment commands

```bash
# Apply or create
kubectl apply -f deployment.yaml

# Check status
kubectl get deployments
kubectl rollout status deployment/my-app

# Update image
kubectl set image deployment/my-app my-app=my-app:2.0

# Rollback
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# View rollout history
kubectl rollout history deployment/my-app

# Scale
kubectl scale deployment/my-app --replicas=5

# Pause / Resume rollout
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app
```

---

## How rolling updates work step by step

1. you apply a new image or config change
2. Kubernetes creates a new ReplicaSet for the new version
3. new Pods are created in the new ReplicaSet
4. once new Pods pass readiness checks, old Pods are terminated
5. this continues until all old Pods are replaced
6. the old ReplicaSet stays with 0 replicas (used for rollback)

---

## Common mistakes

- not setting `selector.matchLabels` to match `template.metadata.labels`
- not setting resource requests and limits
- not setting readiness probes (rolling update may route traffic to unready pods)
- using `Recreate` strategy in production when rolling update is available
- not checking `kubectl rollout status` after updating
- updating Pods directly instead of the Deployment (changes get overwritten)

---

## Good practices

- always set resource requests and limits
- always set readiness and liveness probes
- use `RollingUpdate` with `maxUnavailable: 0` for zero-downtime deploys
- pin image tags — never use `latest` in production
- use `kubectl rollout history` to track changes
- use `minReadySeconds` to slow down rollouts for safety
- annotate changes with `--record` or via CI/CD comments for traceability

---

## Must memorize

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl set image deployment/<name> <container>=<image>:<tag>
kubectl scale deployment/<name> --replicas=<n>
kubectl rollout history deployment/<name>
```

---

## Key ideas

- Deployments are the standard way to run stateless apps in Kubernetes.
- A Deployment manages ReplicaSets; a ReplicaSet manages Pods.
- Rolling updates allow zero-downtime deployments when readiness probes are configured.
- Rollback is fast because old ReplicaSets are kept.
- Always set resource limits and readiness probes for safe rolling updates.
