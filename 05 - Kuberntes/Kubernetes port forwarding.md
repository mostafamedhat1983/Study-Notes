# Kubernetes Port Forwarding

`kubectl port-forward` creates a tunnel from your **local machine** to a Pod, Service, or Deployment inside the cluster. It is intended for debugging and development — not production traffic.

---

## Syntax

```bash
# Forward to a Pod
kubectl port-forward pod/<pod-name> <local-port>:<pod-port>

# Forward to a Service (picks a backing Pod)
kubectl port-forward svc/<service-name> <local-port>:<service-port>

# Forward to a Deployment
kubectl port-forward deployment/<name> <local-port>:<container-port>

# Bind to all interfaces (not just localhost)
kubectl port-forward pod/<pod-name> --address 0.0.0.0 8080:80
```

---

## Examples

```bash
# Access a web app running on port 8080 inside the pod
kubectl port-forward pod/web-abc123 3000:8080
# Now visit http://localhost:3000

# Access a PostgreSQL database on port 5432
kubectl port-forward svc/postgres 5432:5432 -n production
# Connect: psql -h localhost -U myuser -d mydb

# Access Prometheus UI
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring
```

---

## How It Works

1. `kubectl` opens a connection to the API server
2. The API server proxies the connection to the kubelet on the node
3. The kubelet forwards traffic to the target container port
4. Traffic flows: `localhost:<local-port>` → API server → kubelet → container

This means it goes through the API server — it is **not a direct TCP tunnel** and has overhead.

---

## Limitations

- **Debugging only** — not suitable for production traffic or load testing
- Connection drops if the Pod restarts — you must re-run the command
- High-latency path (through API server)
- Cannot forward UDP by default (TCP only)
- Only one session at a time per port-forward process

---

## Run in Background

```bash
# Run in background
kubectl port-forward svc/myapp 8080:80 &

# Kill it later
kill %1
# or find the process
lsof -i :8080
kill <PID>
```

---

## Alternatives

| Method | Use Case |
|---|---|
| `kubectl port-forward` | Quick local debugging |
| `kubectl proxy` | Access Kubernetes API and services via HTTP proxy |
| Ingress | Production external access |
| LoadBalancer Service | Production external access |
| NodePort Service | Direct node access (dev/staging) |
| VPN / Bastion | Secure remote cluster access |
