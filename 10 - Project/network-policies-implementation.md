---
tags:
  - My_CV_Project
---
# Network Policies Implementation Guide

Complete guide to add Kubernetes Network Policies for pod-to-pod traffic control in the chatbot application.

---

## Overview

**What:** Kubernetes firewall rules controlling pod-to-pod communication  
**Why:** Defense-in-depth security, limit blast radius, compliance requirements  
**Difficulty:** Easy (15-30 minutes)  
**Prerequisites:** None - EKS VPC CNI supports network policies by default

---

## Architecture

### Current State (No Network Policies)
- All pods can communicate with all pods
- No traffic restrictions within cluster
- Relies on security groups for external traffic only

### Target State (With Network Policies)
- Frontend → Backend only (port 8000)
- Backend → RDS only (port 3306)
- Backend → AWS APIs only (port 443 for Bedrock/Secrets Manager)
- All pods → DNS only (port 53)
- Block all other traffic

---

## Implementation Steps

### Step 1: Create Network Policy Templates

Create three new files in `platform-ai-chatbot/k8s/templates/`:

#### File 1: `chatbot-backend-network-policy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chatbot-backend-policy
  labels:
    app: chatbot-backend
spec:
  podSelector:
    matchLabels:
      app: chatbot-backend
  policyTypes:
  - Ingress
  - Egress
  
  # Ingress: Allow traffic FROM frontend only
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: chatbot-frontend
    ports:
    - protocol: TCP
      port: 8000
  
  # Egress: Allow traffic TO specific destinations
  egress:
  # Allow RDS database access
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 3306
  
  # Allow AWS API calls (Bedrock, Secrets Manager)
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
  
  # Allow DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
{{- end }}
```

#### File 2: `chatbot-frontend-network-policy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chatbot-frontend-policy
  labels:
    app: chatbot-frontend
spec:
  podSelector:
    matchLabels:
      app: chatbot-frontend
  policyTypes:
  - Ingress
  - Egress
  
  # Ingress: Allow traffic FROM ALB (all sources for now)
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 8501
  
  # Egress: Allow traffic TO backend and DNS only
  egress:
  # Allow backend API calls
  - to:
    - podSelector:
        matchLabels:
          app: chatbot-backend
    ports:
    - protocol: TCP
      port: 8000
  
  # Allow DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
{{- end }}
```

#### File 3: `default-deny-network-policy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
{{- end }}
```

---

### Step 2: Update Helm Values

#### `k8s/values.yaml` (base configuration)

```yaml
# Add at the end of file
networkPolicy:
  enabled: false  # Disabled by default for dev
```

#### `k8s/values-dev.yaml` (development)

```yaml
# Add at the end of file
networkPolicy:
  enabled: false  # Keep disabled in dev for easier debugging
```

#### `k8s/values-prod.yaml` (production)

```yaml
# Add at the end of file
networkPolicy:
  enabled: true  # Enable in production for security
```

---

### Step 3: Deploy Network Policies

#### Option A: Via Jenkins Pipeline

```bash
# Commit and push changes
git add k8s/templates/*network-policy.yaml
git add k8s/values*.yaml
git commit -m "Add network policies for pod-to-pod traffic control"
git push

# Jenkins will automatically deploy to current TARGET_ENVIRONMENT
```

#### Option B: Manual Deployment

```bash
# Development (network policies disabled)
helm upgrade --install chatbot ./k8s -f k8s/values-dev.yaml

# Production (network policies enabled)
helm upgrade --install chatbot ./k8s -f k8s/values-prod.yaml
```

---

### Step 4: Verify Network Policies

#### Check Network Policies Created

```bash
# List all network policies
kubectl get networkpolicy

# Expected output:
# NAME                      POD-SELECTOR           AGE
# chatbot-backend-policy    app=chatbot-backend    1m
# chatbot-frontend-policy   app=chatbot-frontend   1m
# default-deny-all          <none>                 1m

# Describe backend policy
kubectl describe networkpolicy chatbot-backend-policy

# Describe frontend policy
kubectl describe networkpolicy chatbot-frontend-policy
```

#### Test Allowed Traffic (Should Work)

```bash
# Test frontend → backend (ALLOWED)
kubectl exec -it <frontend-pod> -- wget -O- http://chatbot-backend:8000/health

# Expected: {"status":"healthy",...}

# Test backend → RDS (ALLOWED - check logs)
kubectl logs -l app=chatbot-backend -c chatbot-backend | grep "Database pool created"

# Expected: "Database pool created successfully"

# Test backend → AWS Bedrock (ALLOWED - send message via frontend)
# Open https://chatbot.mostafa-medhat.online and send a message
# Expected: AI response received
```

#### Test Blocked Traffic (Should Fail)

```bash
# Test external pod → backend (BLOCKED)
kubectl run test-pod --rm -it --image=busybox -- wget -O- http://chatbot-backend:8000/health

# Expected: Connection timeout or refused

# Test frontend → RDS directly (BLOCKED)
kubectl exec -it <frontend-pod> -- nc -zv <rds-endpoint> 3306

# Expected: Connection timeout
```

---

## Troubleshooting

### Issue 1: Pods Can't Communicate After Deployment

**Symptom:** Frontend can't reach backend, 500 errors

**Cause:** Network policy too restrictive or incorrect labels

**Solution:**
```bash
# Check pod labels match network policy selectors
kubectl get pods --show-labels | grep chatbot

# Verify labels match:
# - app=chatbot-backend
# - app=chatbot-frontend

# Temporarily disable network policies
helm upgrade --install chatbot ./k8s -f k8s/values-prod.yaml --set networkPolicy.enabled=false
```

### Issue 2: Backend Can't Access RDS

**Symptom:** Backend logs show database connection errors

**Cause:** Egress rule blocking RDS port 3306

**Solution:**
```bash
# Check RDS endpoint and port
kubectl get secret <db-secret> -o jsonpath='{.data.host}' | base64 -d

# Verify egress rule allows port 3306
kubectl describe networkpolicy chatbot-backend-policy | grep -A 5 Egress

# Test connectivity from backend pod
kubectl exec -it <backend-pod> -- nc -zv <rds-endpoint> 3306
```

### Issue 3: Backend Can't Call AWS Bedrock

**Symptom:** AI responses fail, AWS API errors in logs

**Cause:** Egress rule blocking HTTPS (port 443)

**Solution:**
```bash
# Verify egress allows port 443
kubectl describe networkpolicy chatbot-backend-policy | grep -A 5 "port: 443"

# Test AWS API connectivity
kubectl exec -it <backend-pod> -- curl -I https://bedrock-runtime.us-east-2.amazonaws.com
```

### Issue 4: DNS Resolution Fails

**Symptom:** Pods can't resolve service names or external domains

**Cause:** DNS egress rule missing or incorrect

**Solution:**
```bash
# Check kube-dns is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Verify DNS egress rule exists
kubectl describe networkpolicy chatbot-backend-policy | grep -A 5 "port: 53"

# Test DNS from pod
kubectl exec -it <backend-pod> -- nslookup chatbot-backend
```

---

## Rollback

If network policies cause issues:

```bash
# Disable network policies
helm upgrade --install chatbot ./k8s -f k8s/values-prod.yaml --set networkPolicy.enabled=false

# Or delete network policies manually
kubectl delete networkpolicy chatbot-backend-policy
kubectl delete networkpolicy chatbot-frontend-policy
kubectl delete networkpolicy default-deny-all
```

---

## Best Practices

### Start Permissive, Tighten Gradually

1. **Phase 1:** Deploy without default-deny (allow all, add specific rules)
2. **Phase 2:** Test application functionality thoroughly
3. **Phase 3:** Add default-deny policy
4. **Phase 4:** Monitor logs for blocked traffic
5. **Phase 5:** Adjust rules based on legitimate traffic patterns

### Monitor Network Policy Impact

```bash
# Check pod logs for connection errors
kubectl logs -l app=chatbot-backend --tail=100 | grep -i "connection\|timeout\|refused"

# Monitor application metrics
kubectl top pods

# Check ALB target health
aws elbv2 describe-target-health --target-group-arn <arn> --region us-east-2
```

### Development vs Production

- **Dev:** Keep network policies disabled for easier debugging
- **Prod:** Enable network policies for defense-in-depth security
- **Staging:** Enable network policies to catch issues before prod

---

## Security Benefits

### Defense in Depth
- Limits lateral movement if pod is compromised
- Reduces blast radius of security incidents
- Complements security groups (network layer) with pod-level controls (application layer)

### Compliance
- Demonstrates network segmentation for compliance frameworks (PCI-DSS, SOC 2, ISO 27001)
- Provides audit trail of allowed/blocked traffic patterns
- Enforces principle of least privilege at network level

### Incident Response
- Easier to isolate compromised pods
- Clear visibility into expected traffic patterns
- Faster detection of anomalous network behavior

---

## Cost Impact

**Zero additional cost** - Network policies are native Kubernetes resources with no AWS charges.

---

## Future Enhancements

### Fine-Grained ALB Ingress Control

Replace `namespaceSelector: {}` with specific ALB subnet CIDRs:

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.1.0/24  # Public subnet 2a
  - ipBlock:
      cidr: 10.0.2.0/24  # Public subnet 2b
```

### Monitoring Stack Integration

Add network policies for Prometheus scraping:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
    podSelector:
      matchLabels:
        app: prometheus
  ports:
  - protocol: TCP
    port: 8000  # Metrics endpoint
```

### Egress to Specific AWS Endpoints

Replace broad HTTPS egress with specific AWS service endpoints:

```yaml
egress:
- to:
  - ipBlock:
      cidr: 52.94.0.0/16  # AWS Bedrock IP range
  ports:
  - protocol: TCP
    port: 443
```

---

## References

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [AWS VPC CNI Network Policy Support](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html)
- [Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)

---

**Last Updated:** 2025-01-XX
