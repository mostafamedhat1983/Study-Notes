# AWS EKS Authentication: aws-auth ConfigMap vs Access Entry API

---

## Executive Summary

AWS EKS provides two methods for granting IAM users/roles access to Kubernetes clusters:

1. **aws-auth ConfigMap** (Legacy, Deprecated)
2. **EKS Access Entry API** (Modern, Recommended since 2023)

**Key Difference:** aws-auth is a Kubernetes ConfigMap that lives in the cluster, while Access Entry API is managed through AWS APIs with individual resources for each IAM principal.

---

## Table of Contents

1. [The Old Way: aws-auth ConfigMap](#the-old-way-aws-auth-configmap)
2. [The New Way: EKS Access Entry API](#the-new-way-eks-access-entry-api)
3. [Side-by-Side Comparison](#side-by-side-comparison)
4. [The Race Condition You Encountered](#the-race-condition-you-encountered)
5. [Migration Path](#migration-path)
6. [Why AWS Created Access Entry API](#why-aws-created-access-entry-api)

---

## The Old Way: aws-auth ConfigMap

### What Is It?

The `aws-auth` ConfigMap is a **Kubernetes resource** (not an AWS resource) that lives in the `kube-system` namespace. It maps IAM principals (users/roles) to Kubernetes RBAC groups.

### How It Works

**1. Structure:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/JenkinsRole
      username: jenkins-user
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters
```

**2. Authentication Flow:**
```
IAM User/Role → AWS STS → Kubernetes API Server → Check aws-auth ConfigMap → Grant access based on groups
```

### Managing with Terraform

**Option A: Kubernetes Provider (Complex)**
```hcl
resource "kubernetes_config_map_v1_data" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = yamlencode([
      {
        rolearn  = aws_iam_role.jenkins.arn
        username = "jenkins-user"
        groups   = ["system:masters"]
      },
      {
        rolearn  = aws_iam_role.nodes.arn
        username = "system:node:{{EC2PrivateDNSName}}"
        groups   = ["system:bootstrappers", "system:nodes"]
      }
    ])
    
    mapUsers = yamlencode([
      {
        userarn  = "arn:aws:iam::123456789012:user/admin"
        username = "admin"
        groups   = ["system:masters"]
      }
    ])
  }

  force = true

  depends_on = [aws_eks_cluster.main]
}
```

**Option B: Manual kubectl (Traditional)**
```bash
# Edit manually
kubectl edit configmap aws-auth -n kube-system

# Or apply from file
kubectl apply -f aws-auth.yaml
```

### Problems with aws-auth ConfigMap

#### ❌ **Problem 1: Monolithic Resource**

All access mappings in ONE ConfigMap:
- Team A adds node groups
- Team B adds developer access
- Team C adds Jenkins access

**Result:** Merge conflicts, overwrites, coordination nightmares

**Example:**
```hcl
# Module A: manages nodes
resource "kubernetes_config_map_v1_data" "aws_auth" {
  data = {
    mapRoles = yamlencode([node_role])
  }
}

# Module B: manages Jenkins (CONFLICT!)
resource "kubernetes_config_map_v1_data" "aws_auth" {
  data = {
    mapRoles = yamlencode([jenkins_role])
  }
}
```

These are duplicate Terraform resource addresses and cannot be declared as two independent resources in one configuration. A single owner must combine the mappings; otherwise one team's update can overwrite another team's data.

#### ❌ **Problem 2: Race Conditions**

**Scenario:** In clusters using ConfigMap authentication, EKS managed node groups may add node-role mappings to `aws-auth`. A Terraform configuration that also owns or replaces the ConfigMap can conflict with those mappings if ownership and ordering are not designed carefully.

**What happens:**
```
Time 0: EKS cluster created
Time 1: A managed node group may add its node-role mapping to aws-auth (when using ConfigMap authentication)
Time 2: Your Terraform tries to create aws-auth ConfigMap
Time 3: Terraform's create/update may conflict with the existing ConfigMap or replace mappings if the configuration does not preserve them
```

**Correct approach:** Use one clearly defined owner for the ConfigMap and merge all required mappings. `depends_on` controls ordering but does not merge data or prevent overwrites. Provider `force` controls field ownership conflicts; it is not a substitute for preserving node-role mappings. Prefer EKS Access Entries where supported.

#### ❌ **Problem 3: Manual Editing Risk**

```bash
kubectl edit configmap aws-auth -n kube-system
```

Opens vim/nano with YAML. One wrong space = broken cluster access.

**Real-world horror:**
- Delete wrong line → lose access to cluster
- Bad YAML indentation → ConfigMap broken
- No validation until you save
- No dry-run or rollback

#### ❌ **Problem 4: Not Version Controlled Naturally**

- Lives in Kubernetes, not Git (unless you export it)
- No audit trail of who changed what
- Hard to review changes before applying
- Difficult to rollback

#### ❌ **Problem 5: Deprecated by AWS**

AWS documents `aws-auth` as deprecated and recommends access entries for supported clusters. Deprecation does not mean the ConfigMap has already been removed; clusters using ConfigMap authentication may continue to depend on it.

---

## The New Way: EKS Access Entry API

### What Is It?

**EKS Access Entry API** is an **AWS API** (not Kubernetes API) that creates individual resources for each IAM principal's cluster access.

Released: **2023**  
Status: **Recommended** by AWS for all new clusters

### How It Works

**1. Individual Resources:**

Each IAM user/role gets its own Access Entry:

```hcl
resource "aws_eks_access_entry" "jenkins" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "jenkins_admin" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }
}
```

**2. Authentication Flow:**
```
IAM User/Role → AWS STS → EKS Access Entry API → Kubernetes API Server → Grant access based on policy
```

### Managing with Terraform

**Simple and Clean:**

```hcl
# Grant Jenkins full admin access
resource "aws_eks_access_entry" "jenkins" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
}

resource "aws_eks_access_policy_association" "jenkins_admin" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }
}

# Grant developer read-only access to specific namespace
resource "aws_eks_access_entry" "developer" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::123456789012:user/developer"
}

resource "aws_eks_access_policy_association" "developer_readonly" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::123456789012:user/developer"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["dev"]
  }
}
```

### Available AWS-Managed Policies

```
arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy
```

### Using AWS CLI

```bash
# Create access entry
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/JenkinsRole

# Associate policy
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/JenkinsRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

### Advantages of Access Entry API

#### ✅ **Advantage 1: Granular Resources**

Each IAM principal = separate Terraform resource

```hcl
# Module A: manages nodes (handled by EKS automatically)

# Module B: manages Jenkins
resource "aws_eks_access_entry" "jenkins" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
}

# Module C: manages developers
resource "aws_eks_access_entry" "dev_team" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::123456789012:role/DevTeam"
}
```

**No conflicts!** Each module manages its own access entries independently.

#### ✅ **Advantage 2: No Race Conditions**

AWS manages the lifecycle. No timing issues with cluster creation.

```hcl
resource "aws_eks_cluster" "main" {
  name = "my-cluster"
  # ...
}

# This works cleanly - no depends_on needed
resource "aws_eks_access_entry" "jenkins" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.jenkins.arn
}
```

#### ✅ **Advantage 3: API-Driven, No Manual Editing**

All changes through AWS API:
- Validation before applying
- Can't break cluster access with bad YAML
- Programmatic and repeatable

#### ✅ **Advantage 4: Proper Version Control**

Lives in Terraform code:
- Git history shows all changes
- PR reviews before applying
- Easy rollback (git revert)
- Audit trail built-in

#### ✅ **Advantage 5: CloudTrail Logging**

All access changes automatically logged:
```json
{
  "eventName": "CreateAccessEntry",
  "userIdentity": {
    "principalId": "AIDAI...",
    "arn": "arn:aws:iam::123456789012:user/admin"
  },
  "requestParameters": {
    "clusterName": "my-cluster",
    "principalArn": "arn:aws:iam::123456789012:role/JenkinsRole"
  }
}
```

**Compliance-friendly:** Who added Jenkins access? When? Why? All tracked.

#### ✅ **Advantage 6: Namespace-Scoped Access**

```hcl
resource "aws_eks_access_policy_association" "dev_readonly" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = "arn:aws:iam::123456789012:user/developer"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["dev", "staging"]  # Only these namespaces
  }
}
```

**This is also possible with aws-auth:** map the IAM principal to a Kubernetes group and bind that group with namespace-scoped RBAC. EKS access policies provide a separate AWS-managed authorization option.

---

## Side-by-Side Comparison

| Aspect | aws-auth ConfigMap | Access Entry API |
|--------|-------------------|------------------|
| **Introduced** | 2018 (EKS launch) | 2023 |
| **Status** | Deprecated | Recommended |
| **Resource Type** | Kubernetes ConfigMap | AWS API resource |
| **Managed By** | Kubernetes API | AWS API |
| **Mapping management** | Multiple mappings stored in one ConfigMap | Individual access-entry resources per IAM principal |

| **Terraform Approach** | One ConfigMap resource/owner containing all required mappings | One access-entry resource per IAM principal |
| **Team Collaboration** | ❌ Conflicts, overwrites | ✅ Independent modules |
| **Race Conditions** | ⚠️ Possible when node-group mappings and Terraform both manage the ConfigMap | ⚠️ Possible if automation competes with an automatically created or existing access entry |
| **Manual Editing** | ⚠️ kubectl edit (risky) | ✅ AWS API only |
| **Version Control** | ⚠️ Export required | ✅ Native (Terraform) |
| **Audit Trail** | Kubernetes audit logs can record ConfigMap changes when audit logging is enabled | CloudTrail records EKS access-entry API changes |
| **Validation** | ❌ Apply first, fail later | ✅ API validates |
| **Rollback** | ⚠️ Manual kubectl revert | ✅ Git revert |
| **Namespace Scope** | Supported through mapped Kubernetes RBAC groups | Supported through Kubernetes RBAC or scoped EKS access policies |
| **Learning Curve** | Moderate (YAML, K8s) | Low (AWS API) |
| **Terraform Complexity** | High (data merging) | Low (individual resources) |

---

## The Race Condition You Encountered

### The Problem (Commit: 838e2c4)

**Date:** November 3, 2025  
**Commit Message:** "Fix Terraform count dependency issue in RDS security group rule"

### What Happened

You had this code in `terraform/modules/network/main.tf`:

```hcl
resource "aws_security_group_rule" "rds_from_eks" {
  count                    = var.eks_cluster_security_group_id != "" ? 1 : 0  # PROBLEM
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = var.eks_cluster_security_group_id
  description              = "MySQL from EKS cluster"
}
```

### The Race Condition

**Step-by-step breakdown:**

1. **Terraform Plan Phase:**
   ```
   Network module evaluates: eks_cluster_security_group_id = ""
   Condition: "" != "" ? 1 : 0 → Result: 0
   Terraform decides: Don't create this resource
   ```

2. **Terraform Apply Phase:**
   ```
   EKS cluster gets created
   EKS cluster security group ID becomes available
   BUT: Network module already decided count = 0 during plan
   Security group rule is NOT created
   ```

3. **Result:**
   ```
   RDS security group exists
   EKS cluster exists
   BUT: No security group rule connecting them
   RDS is unreachable from EKS pods
   ```

### Why It's a Race Condition

**"Race"** = Terraform modules evaluating in order, but EKS cluster SG ID not available when network module needs it.

**Timing:**
```
Time 0: Network module plan (eks_sg_id = "")     → count = 0
Time 1: EKS module creates cluster
Time 2: EKS cluster SG ID available (sg-0197d3...)
Time 3: Network module already skipped rule creation
Time 4: RDS unreachable ❌
```

### The Fix (Commit 838e2c4)

**Removed the conditional count:**

```hcl
resource "aws_security_group_rule" "rds_from_eks" {
  # count removed - always create this rule
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = var.eks_cluster_security_group_id
  description              = "MySQL from EKS cluster"
}
```

**Why this works:**

Terraform now creates the rule every time. The `var.eks_cluster_security_group_id` is passed as an output from EKS module, so Terraform automatically handles the dependency:

```hcl
# In EKS module
output "cluster_security_group_id" {
  value = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
}

# In root module
module "network" {
  # ...
  eks_cluster_security_group_id = module.eks.cluster_security_group_id
}
```

**Terraform dependency graph:**
```
module.eks → creates cluster → outputs SG ID → module.network uses SG ID → creates rule
```

No race condition because Terraform waits for the output before creating dependent resources.

### Related Issue: EKS Node Security Group

**Earlier Problem (Commit 4be5436, 860a0dc):**

You initially created a custom `eks_node` security group in the network module:

```hcl
resource "aws_security_group" "eks_node" {
  name   = "${var.environment}-eks-node"
  vpc_id = aws_vpc.main.id
  # ...
}
```

**Problem:** EKS creates its own cluster security group automatically. Your custom SG was never attached to nodes.

**Result:** 
- RDS allowed traffic from your custom SG
- EKS nodes used EKS-managed SG
- RDS was unreachable

**Fix:** Use EKS cluster's auto-created security group:

```hcl
# Output from EKS module
output "cluster_security_group_id" {
  value       = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
  description = "EKS cluster security group ID (auto-created by EKS)"
}
```

**Lesson:** Don't fight EKS defaults. Use the security group EKS creates for nodes.

---

## Migration Path

### If You're Currently Using aws-auth ConfigMap

**Step 1: Create Access Entries for New Principals**

```hcl
# New IAM roles/users: use Access Entry API
resource "aws_eks_access_entry" "new_developer" {
  cluster_name      = aws_eks_cluster.main.name
  principal_arn     = aws_iam_role.new_developer.arn
  kubernetes_groups = ["developers"]
}

# A matching RoleBinding/ClusterRoleBinding for the `developers` group
# must exist in Kubernetes, or associate an EKS access policy instead.
resource "aws_eks_access_policy_association" "new_developer_view" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.new_developer.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["development"]
  }
}
```

**Step 2: Migrate Existing Entries Gradually**

```bash
# List current aws-auth mappings
kubectl get configmap aws-auth -n kube-system -o yaml

# For each IAM principal, create Access Entry
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn <arn>

# Verify access still works
# Remove from aws-auth ConfigMap
```

**Step 3: Remove the legacy ConfigMap only after the authentication migration is complete**

Before deleting `aws-auth`, verify all human, automation, node, and Fargate role mappings have an equivalent supported access entry or are otherwise no longer required. Confirm the cluster authentication mode and node-group behavior. If the cluster still uses ConfigMap authentication, deleting `aws-auth` can break node or user access. Prefer changing the cluster to API authentication mode only after validating all required principals and access paths. Do not delete the ConfigMap as a routine cleanup step.

**Note:** Managed node-group access behavior depends on the cluster authentication mode and how the node group was created. In clusters using the EKS API authentication mode (or API_AND_CONFIG_MAP), EKS can create the required node access entry for managed node groups. In ConfigMap-only mode, node role mappings use `aws-auth`. Do not infer access-entry creation from the Kubernetes version alone.

---

## Why AWS Created Access Entry API

### The Problems AWS Wanted to Solve

1. **Customer Confusion**
   - "Why is IAM managed through Kubernetes?"
   - "Why do I need kubectl to manage AWS access?"

2. **Operational Complexity**
   - Merge conflicts in multi-team environments
   - Race conditions during cluster creation
   - Manual YAML editing errors

3. **Lack of Auditing**
   - No CloudTrail logs for access changes
   - Hard to track who modified what

4. **Not "Infrastructure as Code" Friendly**
   - Difficult to manage with Terraform/CloudFormation
   - Required Kubernetes provider AND AWS provider

5. **Namespace-Scoped Access Was Hard**
   - aws-auth maps principals to Kubernetes groups; those groups can be authorized with namespace-scoped Kubernetes RBAC
   - Needed manual Kubernetes RBAC for namespace limits

### AWS's Solution: Access Entry API

Make cluster access management:
- **AWS-native** (AWS API, not Kubernetes API)
- **Auditable** (CloudTrail logging)
- **Granular** (individual resources)
- **Namespace-aware** (built-in scope control)
- **IaC-friendly** (clean Terraform resources)

---

## Conclusion

### aws-auth ConfigMap (Old Way)

**Use When:**
- Legacy clusters (pre-2023)
- Already heavily invested in aws-auth
- No immediate need to migrate

**Avoid For:**
- New clusters
- Multi-team environments
- Compliance-heavy workloads

### Access Entry API (New Way)

**Use When:**
- New EKS clusters (2023+)
- Multi-team/multi-module infrastructure
- Need namespace-scoped access
- Want CloudTrail audit logs
- Using Terraform/IaC

**Recommended:** All new clusters should use Access Entry API.

---

## Quick Reference Commands

### aws-auth ConfigMap

```bash
# View current mappings
kubectl get configmap aws-auth -n kube-system -o yaml

# Edit manually
kubectl edit configmap aws-auth -n kube-system

# Apply from file
kubectl apply -f aws-auth.yaml
```

### Access Entry API

```bash
# Create access entry
aws eks create-access-entry \
  --cluster-name CLUSTER \
  --principal-arn IAM_ARN

# List access entries
aws eks list-access-entries --cluster-name CLUSTER

# Associate policy
aws eks associate-access-policy \
  --cluster-name CLUSTER \
  --principal-arn IAM_ARN \
  --policy-arn POLICY_ARN \
  --access-scope type=cluster

# Delete access entry
aws eks delete-access-entry \
  --cluster-name CLUSTER \
  --principal-arn IAM_ARN
```

---

## Additional Resources

- [AWS EKS Access Entry API Documentation](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Terraform aws_eks_access_entry Resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_access_entry)
- [AWS Blog: Introducing EKS Access Entry API](https://aws.amazon.com/blogs/containers/a-deep-dive-into-simplified-amazon-eks-access-management-controls/)
- [aws-auth ConfigMap Documentation (Legacy)](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)

---

**Created:** December 12, 2025  
**Last Updated:** December 12, 2025  
**Status:** Complete explanation with race condition analysis
