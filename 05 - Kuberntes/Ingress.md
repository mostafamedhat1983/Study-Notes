---
tags:
  - Kubernetes
---
Ingress exposes HTTP and HTTPS routes from outside the cluster to Services inside the cluster.

---

## Main idea

- a Service of type LoadBalancer creates one cloud load balancer per Service (expensive)
- Ingress uses one load balancer and routes traffic to many Services based on rules
- an Ingress resource defines the routing rules
- an Ingress controller (e.g. ingress-nginx, AWS ALB Ingress) implements the rules

---

## Ingress vs Service LoadBalancer

| | LoadBalancer Service | Ingress |
|---|---|---|
| Load balancer per service | yes (one per service) | no (one for all) |
| HTTP path routing | no | yes |
| Host-based routing | no | yes |
| TLS termination | manual | built-in |
| Cost | expensive at scale | cost-effective |

---

## Ingress controller

The Ingress resource alone does nothing — you need an Ingress controller installed.

Common controllers:
- `ingress-nginx` (most common self-managed)
- `AWS Load Balancer Controller` (ALB)
- `Traefik`
- `HAProxy Ingress`
- `GKE Ingress` (Google Cloud)

```bash
# Install ingress-nginx with Helm
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

---

## Basic Ingress manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

---

## Path-based routing

Route different paths to different Services:

```yaml
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

## Host-based routing

Route different hostnames to different Services:

```yaml
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

---

## TLS configuration

```yaml
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls    # Secret containing tls.crt and tls.key
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

Use cert-manager to automate TLS certificate provisioning with Let's Encrypt.

---

## pathType options

| pathType | Behavior |
|---|---|
| `Prefix` | matches path prefix (most common) |
| `Exact` | exact path match only |
| `ImplementationSpecific` | controller-specific behavior |

---

## Common Ingress commands

```bash
kubectl get ingress
kubectl get ingress -n production
kubectl describe ingress <name>
kubectl get ingress <name> -o yaml
```

---

## Common mistakes

- creating an Ingress without installing an Ingress controller (nothing happens)
- not specifying `ingressClassName` (may use wrong controller)
- not creating TLS secrets before referencing them in Ingress
- using `Exact` pathType when `Prefix` is needed
- forgetting that Ingress is namespace-scoped (must be in the same namespace as the Service)
- DNS not pointing to the Ingress controller's external IP

---

## Good practices

- use one Ingress controller per cluster (or per environment)
- use cert-manager to automate TLS certificate management
- use annotations to configure controller-specific behavior
- use `Prefix` pathType for most routes
- always specify `ingressClassName` explicitly
- monitor Ingress controller metrics (requests, errors, latency)

---

## Must memorize

```
Ingress resource  → defines routing rules
Ingress controller → implements the rules (must be installed separately)
```

```bash
kubectl get ingress
kubectl describe ingress <name>
```

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

---

## Key ideas

- Ingress routes HTTP/HTTPS traffic to multiple Services using one load balancer.
- An Ingress controller must be installed for the Ingress resource to work.
- Ingress supports host-based and path-based routing.
- TLS is configured directly in the Ingress spec using Kubernetes Secrets.
- Use cert-manager to automate TLS certificate issuance and renewal.
