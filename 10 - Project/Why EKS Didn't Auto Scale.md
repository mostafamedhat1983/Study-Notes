---
tags:
  - My_CV_Project
  - AWS
  - EKS
  - Kubernetes
---
## Why EKS Didn't Auto-Scale:

**The Problem:**  
You hit the **pod limit per node** (11 pods for t3.small), not CPU/memory limits. Cluster Autoscaler only scales based on **resource requests** (CPU/memory), not pod count.

**What's Actually Happening:**

```c
Node has capacity for: 11 pods, 2 vCPU, 2GB RAM
Current usage: 11 pods, 0.5 vCPU (25%), 1GB RAM (50%)
                ↓
Cluster Autoscaler sees: CPU/memory available ✅
Pod scheduling fails: No pod slots available ❌
                ↓
Autoscaler does NOT scale (thinks resources are available)
```

Copy

**The Root Cause:**  
AWS limits pods per node based on **ENI (Elastic Network Interface) capacity**, not resources. Each pod needs an IP address, and t3.small supports only 3 ENIs × 4 IPs = 11 pods max (1 IP reserved for node).

## Why Autoscaler Didn't Trigger:

**Cluster Autoscaler scales when:**

- ✅ Pods are **Pending** due to insufficient CPU/memory
    
- ❌ NOT when pods are **Unschedulable** due to pod limit
    

**Your case:**

```c
Pod status: Unschedulable (Too many pods)
Reason: 0/3 nodes available: 3 Too many pods
Autoscaler: Sees CPU/memory available, does nothing
```

Copy

## How to Enable Auto-Scaling for Pod Limits:

**Option 1: Increase max_size (Already Done)**

```hcl
node_max_size = 6  # Allows manual scaling up to 6 nodes
```

Copyhcl

But Cluster Autoscaler still won't trigger automatically for pod limits.

**Option 2: Install Cluster Autoscaler (Recommended)**

Even with Cluster Autoscaler, it won't help with pod limits unless you configure it properly:

```bash
# Install Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=platform-dev \
  --set awsRegion=us-east-2 \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false
```

Copybash

**But:** Cluster Autoscaler still primarily looks at resource requests, not pod count.

## The Lesson:

**Pod limits are hard limits** - Cluster Autoscaler doesn't automatically scale for them. You must:

1. Monitor pod usage manually
    
2. Set 
    
    ```plaintext
    node_max_size
    ```
    
     high enough for growth
    
3. Use larger instance types (more pods per node)
    
4. Or manually scale when approaching limits
    

**Best practice:** Always leave 20-30% pod capacity headroom to avoid hitting limits unexpectedly.

**Your fix (4 nodes) is the right approach** - manual scaling when you know you need more capacity.