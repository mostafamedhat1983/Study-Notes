# kubectl

`kubectl` is the CLI tool for interacting with a Kubernetes cluster via the API server.

---

## Context and Config

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set default namespace for current context
kubectl config set-context --current --namespace=production
```

---

## Get / Describe

```bash
# List resources
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods -A                        # all namespaces
kubectl get pods -o wide                   # with node and IP
kubectl get pods -o yaml                   # full YAML output
kubectl get pods --show-labels
kubectl get pods -l app=web                # filter by label

# Watch live
kubectl get pods -w

# Describe (events, conditions, details)
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe deployment <name>
```

---

## Logs

```bash
# Get logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>     # multi-container pod
kubectl logs <pod-name> --previous         # crashed container logs
kubectl logs <pod-name> -f                 # follow/stream logs
kubectl logs <pod-name> --tail=100         # last 100 lines
kubectl logs -l app=web                    # logs from all matching pods
```

---

## Exec and Debug

```bash
# Shell into a running container
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/bash

# Run a one-off command
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- cat /etc/config/app.conf

# Debug with ephemeral container (Kubernetes 1.23+)
kubectl debug -it <pod-name> --image=busybox --target=<container>

# Run a temporary debug pod
kubectl run debug --image=busybox -it --rm --restart=Never -- sh
```

---

## Apply / Create / Delete

```bash
# Apply manifest (create or update)
kubectl apply -f manifest.yaml
kubectl apply -f ./k8s/                    # apply all files in dir

# Create (fails if exists)
kubectl create -f manifest.yaml

# Delete
kubectl delete -f manifest.yaml
kubectl delete pod <pod-name>
kubectl delete pod <pod-name> --force --grace-period=0   # force delete
```

---

## Deployments and Rollouts

```bash
# Rollout status
kubectl rollout status deployment/<name>

# Rollout history
kubectl rollout history deployment/<name>

# Rollback to previous version
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=2

# Restart all pods in a deployment (rolling)
kubectl rollout restart deployment/<name>

# Scale
kubectl scale deployment <name> --replicas=5
```

---

## Port Forwarding

```bash
# Forward local port to pod port
kubectl port-forward pod/<pod-name> 8080:80

# Forward to service
kubectl port-forward svc/<service-name> 8080:80

# Forward to deployment
kubectl port-forward deployment/<name> 8080:80
```

---

## Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods
kubectl top pods -n <namespace>
kubectl top pods --sort-by=memory
```

---

## Events and Troubleshooting

```bash
# Get events (sorted by time)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Get events for a specific pod
kubectl describe pod <pod-name> | grep -A20 Events

# Check why a pod is pending
kubectl describe pod <pod-name> | grep -A5 'Events\|Conditions'
```

---

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging

# Run command in a namespace
kubectl get pods -n kube-system
```

---

## Useful Shortcuts

| Short | Full |
|---|---|
| `po` | `pods` |
| `svc` | `services` |
| `deploy` | `deployments` |
| `ns` | `namespaces` |
| `cm` | `configmaps` |
| `pvc` | `persistentvolumeclaims` |
| `sa` | `serviceaccounts` |
| `ing` | `ingresses` |
| `no` | `nodes` |

```bash
# Examples using shortcuts
kubectl get po -A
kubectl get svc -n production
kubectl describe deploy web
```
