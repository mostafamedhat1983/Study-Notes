---
tags:
  - Kubernetes
---
# PDB, HPA, and ReplicaSet Interaction

## Overview

Pod Disruption Budget (PDB), Horizontal Pod Autoscaler (HPA), and ReplicaSet work together without conflicts. Each has a distinct role in managing pod availability and scaling.

## Component Roles

### ReplicaSet (managed by Deployment)
- **Role:** Maintains desired number of pods
- **Example:** "Keep 2 pods running"
- **Managed by:** Deployment controller

### HPA (Horizontal Pod Autoscaler)
- **Role:** Dynamically changes desired replica count based on metrics
- **Example:** "Scale from 2 to 5 pods when CPU > 75%"
- **Controls:** ReplicaSet's desired replica count

### PDB (Pod Disruption Budget)
- **Role:** Protects pods during voluntary disruptions (node drain, cluster updates, manual evictions)
- **Example:** "Keep at least 1 pod available during disruptions"
- **Does NOT control:** Replica count (only prevents disruptions)

## How They Work Together

```
┌─────────────────────────────────────────────────────────────┐
│ HPA monitors CPU/Memory                                      │
│ Adjusts ReplicaSet desired replicas (2 → 5)                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ ReplicaSet maintains pod count                               │
│ Creates/deletes pods to match desired replicas              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ PDB protects during disruptions                              │
│ Ensures minAvailable pods remain during node maintenance    │
└─────────────────────────────────────────────────────────────┘
```

## Example Scenario

**Configuration:**
- Deployment: 2 replicas
- HPA: minReplicas=2, maxReplicas=5, targetCPU=75%
- PDB: minAvailable=1

**Timeline:**

### 1. Normal Operation
- **State:** 2 pods running
- **HPA:** Monitoring CPU (currently 50%)
- **PDB:** Inactive (no disruptions)

### 2. High CPU Load
- **CPU:** Increases to 80%
- **HPA:** Scales to 4 pods
- **ReplicaSet:** Creates 2 new pods
- **PDB:** Still inactive

### 3. Node Maintenance (Drain)
- **Admin:** Drains node for maintenance
- **Kubernetes:** Tries to evict pods
- **PDB:** Blocks eviction if it would violate minAvailable=1
- **Result:** Pods evicted one at a time, ensuring 1 always available

### 4. After Maintenance
- **HPA:** Adjusts replicas based on current CPU
- **ReplicaSet:** Maintains desired count
- **PDB:** Returns to monitoring

## Configuration Rules

### Safe Configuration
**Rule:** `minAvailable` should be **less than** `minReplicas`

**Why:** Allows Kubernetes to perform disruptions while maintaining availability.

**Example (Dev):**
```yaml
replicaCount: 2
hpa:
  minReplicas: 2
  maxReplicas: 5
pdb:
  minAvailable: 1  # ✅ 1 < 2 (Safe)
```

**Example (Prod):**
```yaml
replicaCount: 3
hpa:
  minReplicas: 3
  maxReplicas: 7
pdb:
  minAvailable: 2  # ✅ 2 < 3 (Safe)
```

### Unsafe Configuration
```yaml
replicaCount: 2
hpa:
  minReplicas: 2
pdb:
  minAvailable: 2  # ❌ 2 = 2 (Blocks all disruptions)
```

**Problem:** Cannot drain nodes because evicting any pod violates PDB.

## Conflict Scenarios (None in Our Config)

### ✅ No Conflict: HPA and PDB
- HPA changes replica count
- PDB only prevents disruptions
- They operate on different dimensions

### ✅ No Conflict: ReplicaSet and PDB
- ReplicaSet creates/deletes pods
- PDB only blocks voluntary evictions
- Involuntary terminations (crashes) not affected by PDB

### ✅ No Conflict: Deployment replicas and HPA
- When HPA is active, it overrides deployment's replica count
- As long as deployment replicas = HPA minReplicas, no conflict

## Our Configuration Analysis

### Development Environment
```yaml
replicaCount: 2
backend:
  hpa:
    minReplicas: 2  # Matches replicaCount ✅
    maxReplicas: 5
    targetCPU: 75
  pdb:
    minAvailable: 1  # Less than minReplicas ✅
```

**Analysis:**
- ✅ Deployment starts with 2 pods
- ✅ HPA takes control with minReplicas=2 (no conflict)
- ✅ PDB allows disruption of 1 pod (2 - 1 = 1 available)
- ✅ Can scale up to 5 pods under load

### Production Environment
```yaml
replicaCount: 3
backend:
  hpa:
    minReplicas: 3  # Matches replicaCount ✅
    maxReplicas: 7
    targetCPU: 70
  pdb:
    minAvailable: 2  # Less than minReplicas ✅
```

**Analysis:**
- ✅ Deployment starts with 3 pods
- ✅ HPA takes control with minReplicas=3 (no conflict)
- ✅ PDB allows disruption of 1 pod (3 - 1 = 2 available)
- ✅ Can scale up to 7 pods under load

## Best Practices

1. **Set minAvailable < minReplicas**
   - Allows node maintenance without blocking
   - Maintains high availability

2. **Match deployment replicas to HPA minReplicas**
   - Prevents initial scale-down
   - Smooth transition to HPA control

3. **Use percentage-based PDB for large deployments**
   ```yaml
   pdb:
     minAvailable: 50%  # Instead of fixed number
   ```

4. **Test disruptions**
   ```bash
   # Simulate node drain
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   
   # Verify PDB is respected
   kubectl get pdb
   ```

## Troubleshooting

### PDB Blocking Disruptions
```bash
# Check PDB status
kubectl get pdb chatbot-backend-pdb -o yaml

# Check if disruptions are allowed
kubectl describe pdb chatbot-backend-pdb
```

**Look for:**
```
Allowed disruptions: 1
Current: 2
Desired: 2
```

### HPA Not Scaling
```bash
# Check HPA status
kubectl get hpa chatbot-backend-hpa

# Check metrics
kubectl top pods -l app=chatbot-backend
```

## Summary

**PDB, HPA, and ReplicaSet are complementary:**
- **HPA:** Controls how many pods (dynamic scaling)
- **ReplicaSet:** Maintains pod count (reconciliation)
- **PDB:** Protects availability during disruptions (safety net)

**No conflicts exist when:**
- minAvailable < minReplicas
- deployment replicas = HPA minReplicas

**Our configuration follows all best practices.**
