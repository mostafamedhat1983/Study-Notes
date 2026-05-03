# Testing Database Connectivity from a Pod

When a service can't reach a database inside the cluster, you need to test connectivity **from within the cluster network** — not from your local machine. These are the standard techniques.

---

## Method 1: Temporary Pod with psql / mysql client

### PostgreSQL
```bash
# Spin up a temporary pod with psql
kubectl run pg-test \
  --image=postgres:15 \
  --restart=Never \
  --rm -it \
  -n <namespace> \
  -- psql -h <db-service-name> -U <username> -d <dbname>

# Example
kubectl run pg-test \
  --image=postgres:15 \
  --restart=Never \
  --rm -it \
  -n production \
  -- psql -h postgres-svc -U appuser -d appdb
```

### MySQL / MariaDB
```bash
kubectl run mysql-test \
  --image=mysql:8 \
  --restart=Never \
  --rm -it \
  -n production \
  -- mysql -h mysql-svc -u appuser -p appdb
```

---

## Method 2: Exec into Existing Pod

If your app Pod already has a DB client or curl:

```bash
# Shell into the app pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Inside the pod: test TCP connectivity
nc -zv <db-service-name> 5432

# Test DNS resolution
nslookup <db-service-name>
nslookup <db-service-name>.<namespace>.svc.cluster.local

# Test with psql if available
psql -h <db-service-name> -U <user> -d <db>
```

---

## Method 3: netcat / busybox TCP Test

When no DB client is available:

```bash
# Test if the port is reachable (TCP only)
kubectl run nc-test \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -n production \
  -- nc -zv postgres-svc 5432

# Output: postgres-svc (10.0.1.45:5432) open   ← success
# Output: nc: connect to postgres-svc port 5432 failed   ← failure
```

---

## Method 4: DNS Resolution Check

Before testing the port, verify DNS resolves correctly:

```bash
kubectl run dns-test \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -n production \
  -- nslookup postgres-svc

# Full FQDN format:
# <service>.<namespace>.svc.cluster.local
kubectl run dns-test \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- nslookup postgres-svc.production.svc.cluster.local
```

---

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| `nslookup` fails | DNS not working | Check `coredns` pods in `kube-system` |
| DNS resolves but port fails | NetworkPolicy blocking | Check NetworkPolicy in the DB namespace |
| Connection refused | DB not listening / wrong port | Check DB pod logs and service spec |
| Timeout | Security group / firewall (EKS) | Check AWS Security Group rules |
| Auth error | Wrong credentials | Verify Secret values: `kubectl get secret <name> -o yaml` |
| SSL error | TLS required by DB | Add `sslmode=require` or `sslmode=disable` to connection string |

---

## Decode a Secret to Check Credentials

```bash
# View secret
kubectl get secret <secret-name> -n production -o yaml

# Decode base64 value
kubectl get secret <secret-name> -n production \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## Check the Service and Endpoints

```bash
# Verify service exists and has the right port
kubectl get svc postgres-svc -n production

# Verify service has backing endpoints (pods are ready)
kubectl get endpoints postgres-svc -n production
# Empty endpoints = no ready pods matching the service selector
```
