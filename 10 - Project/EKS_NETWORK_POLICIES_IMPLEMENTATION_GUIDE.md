---
tags:
  - My_CV_Project
  - EKS
  - AWS
  - Kubernetes
---
# EKS Network Policies - Implementation Guide

**Security Best Practice:** Zero-trust networking for Kubernetes workloads

This guide provides step-by-step instructions to implement Kubernetes Network Policies for your EKS chatbot application. Network policies enforce **least-privilege networking** by controlling pod-to-pod communication at Layer 3/4, complementing your existing IAM security.

---

## 📋 Table of Contents

1. [What are Network Policies](#what-are-network-policies)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Prerequisites Check](#prerequisites-check)
4. [Network Policy Design](#network-policy-design)
5. [Implementation Steps](#implementation-steps)
6. [Testing and Validation](#testing-and-validation)
7. [Troubleshooting](#troubleshooting)
8. [CV Impact](#cv-impact)

---

## 🎯 What are Network Policies

### **Purpose:**
Network Policies act as a **firewall for Kubernetes pods**, controlling:
- Which pods can communicate with each other
- Which external endpoints pods can access
- Which ports and protocols are allowed

### **Security Model:**
- **Default behavior**: All traffic allowed (insecure)
- **With policies**: Explicit allow rules (zero-trust)
- **Scope**: Namespace-level or cluster-wide

### **Real-world Analogy:**
Like AWS Security Groups but for pod-to-pod communication inside the cluster.

### **Why You Need This:**

✅ **Compliance**: Required for PCI-DSS, HIPAA, SOC2  
✅ **Security**: Limits blast radius if a pod is compromised  
✅ **Best Practice**: Defense in depth (IAM + Network segmentation)  
✅ **CV Value**: Shows advanced Kubernetes security knowledge  

---

## 🏗️ Current Architecture Analysis

### **Application Components (platform-ai-chatbot):**

```
Internet
   ↓
[Application Load Balancer] (managed by AWS LB Controller)
   ↓
[Frontend Pods] (Streamlit on port 8501)
   ↓
[Backend Pods] (FastAPI on port 8000)
   ↓ ↓ ↓
[RDS MySQL] [AWS Bedrock] [AWS Secrets Manager]
(external)   (external)    (external)
```

### **Pod Labels (Existing):**
- **Backend**: `app: chatbot-backend`
- **Frontend**: `app: chatbot-frontend`
- **Namespace**: `default` (confirmed from service account)

### **Communication Requirements:**

| Source | Destination | Port | Protocol | Required? |
|--------|-------------|------|----------|-----------|
| Frontend | Backend | 8000 | TCP | ✅ Yes |
| Backend | RDS MySQL | 3306 | TCP | ✅ Yes |
| Backend | AWS Bedrock | 443 | TCP | ✅ Yes |
| Backend | AWS Secrets Manager | 443 | TCP | ✅ Yes (init container) |
| All Pods | CoreDNS (kube-system) | 53 | UDP/TCP | ✅ Yes |
| All Pods | Kubernetes API | 443 | TCP | ✅ Yes |
| ALB Controller | Frontend/Backend | Any | TCP | ✅ Yes |
| Frontend | Frontend | - | - | ❌ No |
| Backend | Backend | - | - | ❌ No |
| Internet | Backend | 8000 | TCP | ❌ No (only via Frontend) |

### **Security Requirements:**
1. **Default deny all** ingress/egress traffic
2. **Allow** frontend → backend communication
3. **Allow** backend → RDS, Bedrock, Secrets Manager
4. **Allow** all pods → DNS resolution (CoreDNS)
5. **Allow** all pods → Kubernetes API
6. **Allow** ALB Controller → Frontend pods
7. **Deny** direct internet access to backend
8. **Deny** pod-to-pod communication within same tier

---

## ✅ Prerequisites Check

### ✓ Already Configured:
1. **EKS Cluster** - Running with VPC CNI ✅
2. **Pod Labels** - `app: chatbot-backend` and `app: chatbot-frontend` ✅
3. **ClusterIP Services** - For internal communication ✅
4. **Namespace** - Using `default` namespace ✅

### ⚠️ CNI Plugin Check:

**EKS uses Amazon VPC CNI by default**, which **supports Network Policies** starting from:
- **EKS 1.25+** with VPC CNI plugin
- Your cluster is **EKS 1.34** ✅ (confirmed from infra repo)

**No additional CNI installation needed!** Amazon VPC CNI natively supports Network Policies in your EKS version.

> **Note**: Older guides mention Calico, but **it's no longer required** for EKS 1.25+. VPC CNI now has built-in support.

### 📊 What We'll Create:

```
platform-ai-chatbot/k8s/templates/
├── network-policy-default-deny.yaml      [NEW]
├── network-policy-backend.yaml           [NEW]
├── network-policy-frontend.yaml          [NEW]
├── network-policy-kube-system-dns.yaml   [NEW]
└── network-policy-alb-controller.yaml    [NEW]

platform-ai-chatbot/docs/
└── network-policies-guide.md             [NEW]
```

---

## 🎨 Network Policy Design

### **Strategy: Default Deny + Explicit Allow**

1. **Baseline**: Deny all ingress and egress
2. **Layer 1**: Allow DNS (all pods need name resolution)
3. **Layer 2**: Allow Kubernetes API access
4. **Layer 3**: Allow application-specific traffic
5. **Layer 4**: Allow AWS service endpoints

### **Policy Diagram:**

```
┌─────────────────────────────────────────────────────────┐
│  Default Deny All (Baseline)                            │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│  Allow: All Pods → CoreDNS (port 53)                    │
│  Allow: All Pods → Kubernetes API (port 443)            │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│  Frontend:                                               │
│    Ingress: ALB Controller → Frontend (port 8501)       │
│    Egress: Frontend → Backend (port 8000)               │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│  Backend:                                                │
│    Ingress: Frontend → Backend (port 8000)              │
│    Egress: Backend → RDS (port 3306)                    │
│    Egress: Backend → AWS Services (port 443)            │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Implementation Steps

### Step 1: Default Deny All Policy

**Create file:** `platform-ai-chatbot/k8s/templates/network-policy-default-deny.yaml`

```yaml
# ========================================
# Default Deny All Network Policy
# ========================================
# Baseline security posture: deny all ingress and egress traffic
# Other policies will explicitly allow required traffic
# IMPORTANT: This must be applied first before other policies

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
  namespace: default
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
    - Ingress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-egress
  namespace: default
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
    - Egress
```

---

### Step 2: DNS and Kubernetes API Access

**Create file:** `platform-ai-chatbot/k8s/templates/network-policy-kube-system.yaml`

```yaml
# ========================================
# Allow DNS and Kubernetes API Access
# ========================================
# All pods need DNS resolution (CoreDNS in kube-system)
# All pods need Kubernetes API access for service discovery

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-k8s-api
  namespace: default
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
    - Egress
  egress:
    # ========================================
    # Allow DNS (CoreDNS in kube-system)
    # ========================================
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # ========================================
    # Allow Kubernetes API Server
    # ========================================
    # Required for service discovery and pod identity
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: default
      ports:
        - protocol: TCP
          port: 443
    
    # ========================================
    # Allow Kubernetes API via Service CIDR
    # ========================================
    # EKS API endpoint (typically 10.100.0.1 or 172.20.0.1)
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 443
```

---

### Step 3: Backend Network Policy

**Create file:** `platform-ai-chatbot/k8s/templates/network-policy-backend.yaml`

```yaml
# ========================================
# Backend Network Policy
# ========================================
# Controls traffic to/from chatbot backend pods
# Allows: Frontend → Backend, Backend → RDS/AWS Services

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chatbot-backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: chatbot-backend
  
  policyTypes:
    - Ingress
    - Egress
  
  # ========================================
  # Ingress Rules (Traffic TO backend)
  # ========================================
  ingress:
    # Allow traffic from frontend pods
    - from:
        - podSelector:
            matchLabels:
              app: chatbot-frontend
      ports:
        - protocol: TCP
          port: 8000  # FastAPI port
    
    # Allow health checks from ALB Controller
    # ALB Controller pods in kube-system namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: aws-load-balancer-controller
      ports:
        - protocol: TCP
          port: 8000
  
  # ========================================
  # Egress Rules (Traffic FROM backend)
  # ========================================
  egress:
    # ========================================
    # Allow DNS (inherits from allow-dns-and-k8s-api policy)
    # ========================================
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # ========================================
    # Allow Kubernetes API
    # ========================================
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 443
    
    # ========================================
    # Allow RDS MySQL
    # ========================================
    # RDS is outside cluster, allow by CIDR (VPC CIDR)
    # Adjust CIDR based on your VPC configuration
    - to:
        - ipBlock:
            cidr: 10.0.0.0/16  # Your VPC CIDR (adjust if different)
      ports:
        - protocol: TCP
          port: 3306  # MySQL port
    
    # ========================================
    # Allow AWS Services (Bedrock, Secrets Manager)
    # ========================================
    # AWS service endpoints use HTTPS (port 443)
    # Allow all external HTTPS for AWS API calls
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # Block metadata service (security)
      ports:
        - protocol: TCP
          port: 443  # HTTPS for AWS Bedrock & Secrets Manager
```

---

### Step 4: Frontend Network Policy

**Create file:** `platform-ai-chatbot/k8s/templates/network-policy-frontend.yaml`

```yaml
# ========================================
# Frontend Network Policy
# ========================================
# Controls traffic to/from chatbot frontend pods
# Allows: ALB → Frontend, Frontend → Backend

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chatbot-frontend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: chatbot-frontend
  
  policyTypes:
    - Ingress
    - Egress
  
  # ========================================
  # Ingress Rules (Traffic TO frontend)
  # ========================================
  ingress:
    # Allow traffic from Application Load Balancer
    # ALB uses target-type: ip, connects directly to pods
    # Allow from anywhere since ALB is internet-facing
    - from:
        - ipBlock:
            cidr: 0.0.0.0/0  # ALB can come from any IP
      ports:
        - protocol: TCP
          port: 8501  # Streamlit port
    
    # Also allow from ALB Controller for health checks
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: aws-load-balancer-controller
      ports:
        - protocol: TCP
          port: 8501
  
  # ========================================
  # Egress Rules (Traffic FROM frontend)
  # ========================================
  egress:
    # ========================================
    # Allow DNS
    # ========================================
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # ========================================
    # Allow Kubernetes API
    # ========================================
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 443
    
    # ========================================
    # Allow Backend Service
    # ========================================
    - to:
        - podSelector:
            matchLabels:
              app: chatbot-backend
      ports:
        - protocol: TCP
          port: 8000  # Backend API port
```

---

### Step 5: ALB Controller Policy

**Create file:** `platform-ai-chatbot/k8s/templates/network-policy-alb-controller.yaml`

```yaml
# ========================================
# AWS Load Balancer Controller Network Policy
# ========================================
# Allows ALB Controller to discover and health-check application pods
# Applied to kube-system namespace where controller runs

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-alb-controller-egress
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: aws-load-balancer-controller
  
  policyTypes:
    - Egress
  
  egress:
    # ========================================
    # Allow DNS
    # ========================================
    - to:
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # ========================================
    # Allow Kubernetes API
    # ========================================
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443
    
    # ========================================
    # Allow access to application pods (all namespaces)
    # ========================================
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 8000  # Backend
        - protocol: TCP
          port: 8501  # Frontend
    
    # ========================================
    # Allow AWS API calls (EC2, ELB, etc.)
    # ========================================
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

---

### Step 6: Update Helm Chart (Optional but Recommended)

**Modify:** `platform-ai-chatbot/k8s/values.yaml`

Add network policy configuration section:

```yaml
# ========================================
# Network Policies Configuration
# ========================================
networkPolicies:
  enabled: true
  
  # VPC CIDR for RDS access (adjust per environment)
  vpcCidr: "10.0.0.0/16"
```

**Modify:** `platform-ai-chatbot/k8s/values-dev.yaml`

```yaml
# Override VPC CIDR for dev environment
networkPolicies:
  enabled: true
  vpcCidr: "10.0.0.0/16"  # Dev VPC CIDR
```

**Modify:** `platform-ai-chatbot/k8s/values-prod.yaml`

```yaml
# Override VPC CIDR for prod environment
networkPolicies:
  enabled: true
  vpcCidr: "10.0.0.0/16"  # Prod VPC CIDR (same or different)
```

**Optional: Make CIDR templated in network-policy-backend.yaml**

Replace hardcoded CIDR:
```yaml
    - to:
        - ipBlock:
            cidr: {{ .Values.networkPolicies.vpcCidr }}
```

---

### Step 7: Documentation

**Create file:** `platform-ai-chatbot/docs/network-policies-guide.md`

```markdown
# Network Policies Guide

## Overview

This project implements **Kubernetes Network Policies** for zero-trust networking. All pod-to-pod communication is denied by default, with explicit allow rules for required traffic only.

## Network Architecture

### Communication Flow

```
Internet → ALB → Frontend Pods → Backend Pods → RDS/Bedrock/Secrets Manager
                                              ↓
                                          CoreDNS (DNS Resolution)
```

### Policies Applied

| Policy Name | Purpose | Scope |
|-------------|---------|-------|
| `default-deny-all-ingress` | Deny all incoming traffic | All pods |
| `default-deny-all-egress` | Deny all outgoing traffic | All pods |
| `allow-dns-and-k8s-api` | Allow DNS and K8s API | All pods |
| `chatbot-backend-policy` | Allow backend traffic | Backend pods |
| `chatbot-frontend-policy` | Allow frontend traffic | Frontend pods |
| `allow-alb-controller-egress` | Allow ALB controller | kube-system |

## Allowed Traffic

### Frontend Pods
**Ingress:**
- Internet (via ALB) → Port 8501

**Egress:**
- Backend pods → Port 8000
- CoreDNS → Port 53
- Kubernetes API → Port 443

### Backend Pods
**Ingress:**
- Frontend pods → Port 8000
- ALB Controller (health checks) → Port 8000

**Egress:**
- RDS MySQL → Port 3306
- AWS Bedrock/Secrets Manager → Port 443
- CoreDNS → Port 53
- Kubernetes API → Port 443

## Verification

### Check Network Policies

```bash
# List all network policies
kubectl get networkpolicies -n default

# Describe specific policy
kubectl describe networkpolicy chatbot-backend-policy -n default
```

### Test Connectivity

```bash
# Test from frontend to backend (should work)
kubectl exec -it deployment/chatbot-frontend-deployment -- curl http://chatbot-backend-service:8000/health

# Test from backend to frontend (should fail - not allowed)
kubectl exec -it deployment/chatbot-backend-deployment -- curl http://chatbot-frontend-service:8501
```

## Troubleshooting

### Pods Can't Communicate

**Check policy is applied:**
```bash
kubectl get networkpolicy -n default
```

**Check pod labels match policy selectors:**
```bash
kubectl get pods --show-labels -n default
```

**Check DNS resolution:**
```bash
kubectl exec -it <pod-name> -- nslookup chatbot-backend-service
```

### RDS Connection Failed

**Verify VPC CIDR in policy matches your actual VPC:**
```bash
# Check VPC CIDR from AWS
aws ec2 describe-vpcs --query "Vpcs[].CidrBlock"
```

Update `network-policy-backend.yaml` if CIDR is different.

### AWS API Calls Failing

**Check egress to 0.0.0.0/0:443 is allowed:**
```bash
kubectl describe networkpolicy chatbot-backend-policy -n default
```

Should show egress rule allowing port 443 to 0.0.0.0/0.

## Security Considerations

### ✅ Implemented
- Default deny all traffic
- Explicit allow for required communication only
- DNS and K8s API access for all pods
- AWS service endpoints allowed via HTTPS only
- Metadata service (169.254.169.254) blocked

### ⚠️ Important Notes
- Network policies enforce at Layer 3/4 (IP/port level)
- Application-level security (IAM, TLS) still required
- Policies are additive (multiple policies = union of rules)
- No egress from frontend to internet (except backend)

## Monitoring

### CloudWatch Logs
Network policy denials may not appear in CloudWatch by default. Consider:
- Enable VPC Flow Logs for pod ENIs
- Use AWS Network Firewall for advanced logging
- Monitor application logs for connection failures

### Metrics
Monitor these metrics for policy impact:
- Connection timeout errors in application logs
- Pod restart counts (may indicate connectivity issues)
- Service mesh metrics if implemented

## Maintenance

### Adding New Services
When adding new services:
1. Define required communication paths
2. Update relevant network policies
3. Test connectivity before production deployment
4. Document in this guide

### Environment-Specific Policies
- Dev: May have more permissive policies for debugging
- Prod: Strict policies enforced
- Use Helm values to customize per environment

## References

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Amazon VPC CNI Network Policies](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html)
- [Network Policy Best Practices](https://kubernetes.io/docs/concepts/services-networking/network-policies/#best-practices)
```

---

## 🧪 Testing and Validation

### Step 1: Check VPC CNI Version

Verify your VPC CNI supports network policies:

```bash
# Check VPC CNI version (should be 1.12.0+ for network policies)
kubectl describe daemonset aws-node -n kube-system | grep Image

# Expected: amazon-k8s-cni:v1.12.0 or higher
```

If version is too old, update:
```bash
# Update VPC CNI addon
aws eks update-addon \
  --cluster-name platform-dev \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1
```

### Step 2: Apply Policies in Order

**IMPORTANT**: Apply in this specific order to avoid breaking connectivity:

```bash
cd platform-ai-chatbot

# 1. Apply DNS and K8s API access FIRST
kubectl apply -f k8s/templates/network-policy-kube-system.yaml

# 2. Apply application policies
kubectl apply -f k8s/templates/network-policy-backend.yaml
kubectl apply -f k8s/templates/network-policy-frontend.yaml
kubectl apply -f k8s/templates/network-policy-alb-controller.yaml

# 3. Apply default deny LAST (after allow rules are in place)
kubectl apply -f k8s/templates/network-policy-default-deny.yaml
```

### Step 3: Verify Policies

```bash
# List all network policies
kubectl get networkpolicies -n default

# Expected output:
# NAME                          POD-SELECTOR           AGE
# default-deny-all-ingress      <none>                 1m
# default-deny-all-egress       <none>                 1m
# allow-dns-and-k8s-api         <none>                 2m
# chatbot-backend-policy        app=chatbot-backend    2m
# chatbot-frontend-policy       app=chatbot-frontend   2m

# Describe backend policy
kubectl describe networkpolicy chatbot-backend-policy -n default
```

### Step 4: Test Allowed Traffic

```bash
# Test 1: Frontend can reach backend (SHOULD WORK)
kubectl exec -it deployment/chatbot-frontend-deployment -- \
  curl -s http://chatbot-backend-service:8000/health

# Expected: {"status": "healthy"}

# Test 2: Frontend can access ALB (SHOULD WORK)
curl https://chatbot.mostafa-medhat.online

# Expected: Streamlit UI loads successfully

# Test 3: Backend can reach RDS (SHOULD WORK)
# Check backend logs for successful DB connection
kubectl logs deployment/chatbot-backend-deployment | grep -i "database\|mysql"

# Expected: No connection errors
```

### Step 5: Test Denied Traffic

```bash
# Test 1: Backend cannot reach frontend (SHOULD FAIL)
kubectl exec -it deployment/chatbot-backend-deployment -- \
  curl -m 5 http://chatbot-frontend-service:8501

# Expected: Timeout after 5 seconds

# Test 2: Frontend cannot reach frontend peer (SHOULD FAIL)
FRONTEND_POD=$(kubectl get pods -l app=chatbot-frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $FRONTEND_POD -- \
  curl -m 5 http://<another-frontend-pod-ip>:8501

# Expected: Timeout (pods in same tier can't communicate)
```

### Step 6: Monitor Application

```bash
# Check pod status (should remain healthy)
kubectl get pods -n default

# Check backend logs for errors
kubectl logs deployment/chatbot-backend-deployment --tail=50

# Check frontend logs for errors
kubectl logs deployment/chatbot-frontend-deployment --tail=50

# Monitor for restart loops (indicates policy issues)
kubectl get pods -w -n default
```

### Step 7: Test End-to-End

1. **Open application**: https://chatbot.mostafa-medhat.online
2. **Send test message**: "Hello, how are you?"
3. **Verify response**: Should get AI response from Bedrock
4. **Check database**: Conversation should be stored in MySQL

If all tests pass, network policies are working correctly! ✅

---

## 🔧 Troubleshooting

### Issue: Pods Can't Start After Applying Policies

**Cause**: Default deny applied before allow rules

**Solution**:
```bash
# Remove default deny temporarily
kubectl delete networkpolicy default-deny-all-ingress -n default
kubectl delete networkpolicy default-deny-all-egress -n default

# Reapply in correct order (see Step 2)
```

### Issue: Backend Can't Connect to RDS

**Cause**: VPC CIDR mismatch in policy

**Solution**:
```bash
# Check your actual VPC CIDR
aws ec2 describe-vpcs --query "Vpcs[].CidrBlock"

# Update network-policy-backend.yaml with correct CIDR
# Example: If VPC CIDR is 172.31.0.0/16, update:
cidr: 172.31.0.0/16
```

### Issue: Init Container Can't Fetch Secrets

**Cause**: AWS Secrets Manager egress blocked

**Solution**: Verify egress to 0.0.0.0/0:443 is allowed in backend policy

### Issue: DNS Resolution Failing

**Cause**: DNS policy not applied or incorrect selector

**Solution**:
```bash
# Verify CoreDNS label
kubectl get pods -n kube-system -l k8s-app=kube-dns --show-labels

# Ensure policy matches the label
kubectl describe networkpolicy allow-dns-and-k8s-api -n default
```

### Issue: ALB Health Checks Failing

**Cause**: ALB Controller can't reach pods

**Solution**:
```bash
# Check ALB Controller label
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --show-labels

# Verify ingress rules allow traffic from ALB Controller
kubectl describe networkpolicy chatbot-frontend-policy -n default
```

### Issue: Policies Not Enforced

**Cause**: VPC CNI version too old

**Solution**:
```bash
# Update VPC CNI addon
aws eks update-addon \
  --cluster-name platform-dev \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1

# Wait for rollout
kubectl rollout status daemonset/aws-node -n kube-system
```

---

## 💰 Cost Impact

**Good news**: Network Policies have **ZERO additional cost**!

- ✅ No additional AWS charges
- ✅ No extra infrastructure required
- ✅ Built into Amazon VPC CNI plugin
- ✅ No performance impact (enforced at pod ENI level)

---

## 📝 Summary of Changes

### New Files Created:

```
platform-ai-chatbot/k8s/templates/
├── network-policy-default-deny.yaml          (~30 lines)
├── network-policy-kube-system.yaml           (~50 lines)
├── network-policy-backend.yaml               (~120 lines)
├── network-policy-frontend.yaml              (~80 lines)
└── network-policy-alb-controller.yaml        (~60 lines)

platform-ai-chatbot/docs/
└── network-policies-guide.md                 (~250 lines)
```

### Modified Files (Optional):

```
platform-ai-chatbot/k8s/
├── values.yaml          [OPTIONAL] - Add networkPolicies section
├── values-dev.yaml      [OPTIONAL] - Override VPC CIDR
└── values-prod.yaml     [OPTIONAL] - Override VPC CIDR
```

### Total Lines of Code:
- **YAML**: ~340 lines
- **Documentation**: ~250 lines
- **Total**: ~590 lines

---

## ✅ CV Impact

### **What to Highlight:**

**Skills Demonstrated:**
- ✅ Kubernetes Network Policies (zero-trust networking)
- ✅ Layer 3/4 security controls
- ✅ Defense-in-depth security strategy
- ✅ Compliance-ready architecture (PCI-DSS, HIPAA, SOC2)
- ✅ Least-privilege networking principles

**Talking Points for Interviews:**

1. **Security mindset**: "I implemented network policies to enforce least-privilege communication between pods, following zero-trust principles"

2. **Layered security**: "My architecture has multiple security layers: IAM for AWS service access, Pod Identity for workload authentication, and Network Policies for pod-to-pod traffic control"

3. **Compliance**: "Network policies are essential for regulatory compliance in industries like finance and healthcare, ensuring workload isolation and minimizing blast radius"

4. **Production readiness**: "I designed the policies with a default-deny approach, explicitly allowing only required communication paths documented in the network architecture"

### **Resume Bullet Points:**

- "Implemented Kubernetes Network Policies for **zero-trust pod networking**, reducing attack surface with default-deny ingress/egress rules"
- "Designed **multi-layer security architecture** combining IAM, Pod Identity, and Network Policies for defense-in-depth"
- "Configured **compliance-ready network segmentation** following least-privilege principles for production EKS workloads"

---

## 🎯 Next Steps

1. **Review this guide** - Understand policy design and rationale
2. **Check VPC CNI version** - Ensure network policy support
3. **Create network policy files** - Follow Step 1-6
4. **Test in dev first** - Apply policies and validate
5. **Test connectivity** - Run validation tests
6. **Monitor application** - Ensure no disruption
7. **Apply to production** - After successful dev testing
8. **Update README** - Add network policies to features
9. **Update CV** - Add security skills

---

## 📚 References

- [Amazon VPC CNI Network Policies](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)
- [EKS Best Practices - Network Security](https://aws.github.io/aws-eks-best-practices/security/docs/network/)

---

## 🔗 Relation to AWS Backup

**Combined Security & Operations Excellence:**

| Feature | Category | CV Impact |
|---------|----------|-----------|
| **AWS Backup for EKS** | Disaster Recovery / Operations | ⭐⭐⭐⭐⭐ |
| **Network Policies** | Security / Compliance | ⭐⭐⭐⭐ |
| **Together** | Production-Ready Platform | ⭐⭐⭐⭐⭐ |

Implementing both demonstrates:
- **Operational excellence**: Automated backups for business continuity
- **Security excellence**: Zero-trust networking for workload isolation
- **Compliance readiness**: Meets regulatory requirements (SOC2, PCI-DSS)
- **Platform engineering mindset**: Building production-grade, enterprise-ready infrastructure

---

**Document Version:** 1.0  
**Last Updated:** December 2, 2025  
**Author:** Infrastructure & Security Team  
**Status:** Ready for Implementation ✅
