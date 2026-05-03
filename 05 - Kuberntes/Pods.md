---
tags:
  - Kubernetes
---
A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network and storage.

---

## Main idea

- a Pod is not a container — it is a wrapper around one or more containers
- containers in the same Pod share: network namespace (same IP), localhost, and volumes
- Pods are ephemeral — they are created and destroyed, not moved
- you rarely create Pods directly; you use Deployments or other controllers

---

## Basic Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

---

## Pod lifecycle phases

| Phase | Meaning |
|---|---|
| `Pending` | Pod accepted but containers not yet running (scheduling or image pull) |
| `Running` | At least one container is running |
| `Succeeded` | All containers exited with code 0 (Jobs) |
| `Failed` | All containers exited, at least one with non-zero code |
| `Unknown` | Node unreachable, status cannot be determined |

---

## restartPolicy

Controls what happens when a container exits.

| Policy | Behavior |
|---|---|
| `Always` | Always restart (default for Deployments) |
| `OnFailure` | Restart only on non-zero exit (Jobs) |
| `Never` | Never restart |

```yaml
spec:
  restartPolicy: Always
```

---

## Multi-container patterns

### Sidecar

A helper container that runs alongside the main container.

Examples:
- log shipper (Fluentd) next to the app container
- Envoy proxy next to a microservice
- secret sync container

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
    - name: log-shipper
      image: fluentd:v1.16
```

### Init containers

Run to completion before the main container starts.

Examples:
- wait for a database to be ready
- download config files
- run database migrations

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
    - name: app
      image: my-app:1.0
```

---

## Environment variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
```

---

## Common Pod debugging commands

```bash
kubectl get pods
kubectl get pods -o wide                  # show node assignment
kubectl describe pod <pod-name>           # events, state, conditions
kubectl logs <pod-name>                   # container logs
kubectl logs <pod-name> -c <container>    # specific container
kubectl logs <pod-name> --previous        # logs from crashed container
kubectl exec -it <pod-name> -- bash       # shell into container
kubectl exec -it <pod-name> -c <c> -- sh  # shell into specific container
```

---

## Common mistakes

- creating Pods directly instead of using Deployments (orphan Pods are not rescheduled)
- not setting resource requests and limits
- not using init containers to handle startup dependencies
- forgetting that containers in the same Pod share localhost
- using `latest` image tag (unpredictable behavior)
- not checking events with `kubectl describe` when a Pod is stuck

---

## Good practices

- always set resource requests and limits
- always set liveness and readiness probes
- use specific image tags, never `latest` in production
- use init containers for startup ordering
- use Deployments instead of raw Pods
- keep Pods single-purpose — one main process per container

---

## Must memorize

```bash
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <name> --previous
kubectl exec -it <name> -- bash
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
```

---

## Key ideas

- Pods are ephemeral and should be treated as disposable.
- Containers in the same Pod share network and storage.
- Use Deployments to manage Pods, not raw Pod manifests.
- Init containers solve startup ordering problems cleanly.
- Sidecar containers extend the main container without modifying it.
- Always set resource requests, limits, and probes.
