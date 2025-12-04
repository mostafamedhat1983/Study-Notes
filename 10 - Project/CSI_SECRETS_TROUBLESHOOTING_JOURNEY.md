---
tags:
  - My_CV_Project
  - AWS
  - EKS
  - Kubernetes
---
# AWS Secrets Store CSI Driver Troubleshooting Journey

## Overview
This document chronicles the complete journey of attempting to implement AWS Secrets Store CSI Driver for managing database credentials in a Kubernetes chatbot application, the issues encountered, and the decision to pivot to an init container approach.

---

## Current Branch Status
- **Infrastructure Repo**: `fix/remove-csi-driver-use-init-container`
- **Chatbot Repo**: `fix/remove-csi-driver-use-init-container`
- **Main Branch**: Both repos have all CSI-related commits pushed to main
- **Feature Branch**: Has uncommitted changes that need decision

---

## ⚠️ DECISION NEEDED: Uncommitted Changes

### Current Situation
You have **uncommitted changes** on the `fix/remove-csi-driver-use-init-container` branch in **both repos**:

**Infrastructure Repo (my-project-Infra)**:
- ✅ **GOOD**: Removed CSI driver, added Secrets Manager policy to backend role
- 📊 **Impact**: ~60 lines net reduction, cleaner code
- 💡 **Recommendation**: **COMMIT & PUSH** - These changes are valid and needed

**Chatbot Repo (platform-ai-chatbot)**:
- ✅ **GOOD**: Deleted `chatbot-backend-secret-provider-class.yaml` (not needed anymore)
- ❌ **BAD**: `chatbot-backend-deployment.yaml` is in a **broken state**
  - Has CSI volume mount added
  - Still references deleted SecretProviderClass
  - Would fail to deploy with "SecretProviderClass not found" error
- 💡 **Recommendation**: Either:
  1. **Complete the init container implementation** (see Phase 1 below), THEN commit everything
  2. **Discard changes** (`git restore .`) and start fresh with init container

### Recommendation: Complete & Commit (Option 1)

**Phase 1: Chatbot Repo - Complete Init Container**
```bash
# 1. Complete the deployment changes (see "What Needs to Happen Next" section)
# 2. Remove CSI volume configuration
# 3. Add init container
# 4. Test locally if possible
# 5. Commit all changes:
git add .
git commit -m "Replace CSI driver with init container for secrets management"
git push origin fix/remove-csi-driver-use-init-container
```

**Phase 2: Infrastructure Repo - Commit CSI Removal**
```bash
git add terraform/modules/eks/main.tf
git commit -m "Remove Secrets Store CSI Driver and add direct Secrets Manager access to backend"
git push origin fix/remove-csi-driver-use-init-container
```

**Phase 3: Merge to Main** (only after testing)
```bash
# Test in dev environment first!
# Then merge feature branches to main
```

### Alternative: Discard & Start Fresh (Option 2)

If you prefer to abandon current uncommitted changes:

**Infrastructure Repo**:
```bash
cd my-project-Infra
git restore terraform/modules/eks/main.tf
# This reverts to having CSI driver still configured
# You'd need to remove it properly later
```

**Chatbot Repo**:
```bash
cd platform-ai-chatbot
git restore k8s/templates/chatbot-backend-deployment.yaml
git restore k8s/templates/chatbot-backend-secret-provider-class.yaml  # Undelete it
# This reverts to having CSI driver configured
```

---

## Timeline of Changes

### Phase 1: Initial CSI Driver Implementation (Nov 24, 2025)

#### Infrastructure Changes (my-project-Infra)

**Branch**: `feature/secrets-store-csi-driver`

**Commit**: `00a431e` - Add Secrets Store CSI Drive
- **Files Modified**: 4 files, 105 insertions
- **Changes**:
  - `terraform/modules/eks/main.tf`: Added CSI driver addon, IAM policy, role, and Pod Identity association
  - `terraform/modules/eks/variables.tf`: Added namespace variable for CSI driver
  - `terraform/dev/main.tf`: Added namespace parameter
  - `terraform/prod/main.tf`: Added namespace parameter

**What Was Added**:
```hcl
# EKS Addon
resource "aws_eks_addon" "secrets_store_csi_driver" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "aws-secrets-store-csi-driver-provider"
}

# IAM Policy (Environment-scoped)
resource "aws_iam_policy" "secrets_store_csi" {
  # Permissions: secretsmanager:GetSecretValue, DescribeSecret
  # Resource pattern: *-{environment}-* (dev/prod isolation)
}

# IAM Role via Module
module "secrets_store_csi_role" {
  source = "../role"
  name   = "${var.cluster_name}-secrets-store-csi"
  service = "pods.eks.amazonaws.com"
}

# Pod Identity Association
resource "aws_eks_pod_identity_association" "secrets_store_csi" {
  namespace       = "kube-system"
  service_account = "secrets-store-csi-driver-provider-aws"
  role_arn        = module.secrets_store_csi_role.role_arn
}
```

**Commit**: `498cfc3` - Fixing Secrets Store CSI Driver addon name
- Fixed addon name typo/correction

**Commit**: `869f7e7` - Edit README to include AWS Secrets Store CSI Driver details
- Added documentation about CSI driver usage and benefits

---

### Phase 2: Kubernetes Integration (Nov 28, 2025)

#### Chatbot Application Changes (platform-ai-chatbot)

**Commit**: `f077a76` - Add service account and secret provider class for AWS integration

**Files Created**:
1. **`k8s/templates/chatbot-backend-service-account.yaml`**
   - Service account for backend pods
   - Links to IAM role via Pod Identity
   
2. **`k8s/templates/chatbot-backend-secret-provider-class.yaml`**
   - SecretProviderClass resource
   - Maps AWS Secrets Manager to Kubernetes Secret
   - JMESPath mappings for: DB_USER, DB_PASSWORD, DB_NAME, DB_HOST, DB_PORT

**Files Modified**:
1. **`k8s/templates/chatbot-backend-deployment.yaml`**
   - Added `serviceAccountName: chatbot-backend-service-account`
   - Added CSI volume mount: `/mnt/secrets-store`
   - Added CSI volume definition with SecretProviderClass reference
   - Changed `envFrom` to use `chatbot-backend-db-credentials` secret

2. **`k8s/values.yaml`, `k8s/values-dev.yaml`, `k8s/values-prod.yaml`**
   - Added RDS endpoint placeholder values

---

**Commit**: `54034fa` - Refactor deployment configuration

**Files Deleted**:
- `k8s/templates/mysql-external-name-service.yaml` (ExternalName service for RDS)

**Files Modified**:
- **`k8s/templates/chatbot-backend-deployment.yaml`**
  - Removed DB_HOST and DB_PORT from environment variables
  - Rationale: These would come from Secrets Manager via CSI driver
  
- **Values files**: Removed RDS endpoint placeholders (no longer needed)

---

### Phase 3: Chatbot Backend IAM Setup (Nov 28, 2025)

**Branch**: `fix/remove-csi-driver-use-init-container` (branched from main)

**Commit**: `2946c6c` - Add chatbot backend IAM resources and namespace variable

**Files Modified**:
- `terraform/modules/eks/main.tf`: Added chatbot backend IAM policy, role, and Pod Identity
- `terraform/modules/eks/variables.tf`: Added `chatbot_namespace` variable
- `terraform/dev/main.tf`: Added `chatbot_namespace = "default"`
- `terraform/prod/main.tf`: Added `chatbot_namespace = "default"`

**What Was Added**:
```hcl
# Chatbot Backend Bedrock IAM Policy
resource "aws_iam_policy" "chatbot_backend_bedrock" {
  # Permissions: bedrock:InvokeModel, InvokeModelWithResponseStream
  # Resource: deepseek.v3-v1:0 model
}

# Chatbot Backend IAM Role (Pod Identity)
module "chatbot_backend_role" {
  source = "../role"
  name   = "${var.cluster_name}-chatbot-backend"
  policy_arns = [aws_iam_policy.chatbot_backend_bedrock.arn]
}

# Pod Identity Association
resource "aws_eks_pod_identity_association" "chatbot_backend" {
  namespace       = var.chatbot_namespace
  service_account = "chatbot-backend-service-account"
  role_arn        = module.chatbot_backend_role.role_arn
}
```

**Also Attempted** (Later deleted):
- Added SSM Parameter Store for RDS endpoints in both dev/prod `main.tf`
- These were removed in commit `8f9b6ed` (see below)

---

**Commit**: `8f9b6ed` - Remove RDS endpoint parameter from both dev and prod

**Rationale**: RDS endpoint will be retrieved directly from Secrets Manager JSON, not stored separately in SSM Parameter Store.

**Files Modified**:
- `terraform/dev/main.tf`: Removed `aws_ssm_parameter.rds_endpoint`
- `terraform/prod/main.tf`: Removed `aws_ssm_parameter.rds_endpoint`

---

### Phase 4: Troubleshooting & Pivot Decision (Nov 29, 2025)

#### Issues Encountered

**Problem**: CSI Driver not working as expected
- Secrets not being mounted/created properly
- Pod identity authentication issues
- CSI driver logs showing errors (specifics lost in session crash)

#### Decision: Remove CSI Driver Approach

**Current Branch**: `fix/remove-csi-driver-use-init-container`

**Infrastructure Changes (my-project-Infra) - UNCOMMITTED**:

**File Modified**: `terraform/modules/eks/main.tf`
- **Status**: ⚠️ Uncommitted changes on feature branch

**What Was Removed**:
```hcl
# Deleted (lines 186-273 in previous version):
- aws_eks_addon.secrets_store_csi_driver
- aws_iam_policy.secrets_store_csi
- module.secrets_store_csi_role
- aws_eks_pod_identity_association.secrets_store_csi
```

**What Was Added**:
```hcl
# New: Chatbot Backend Secrets Manager Policy
resource "aws_iam_policy" "chatbot_backend_secrets" {
  name = "${var.cluster_name}-chatbot-backend-secrets"
  # Permissions: secretsmanager:GetSecretValue, DescribeSecret
  # Resource: arn:aws:secretsmanager:us-east-2:*:secret:platform-db-*
}

# Updated: Chatbot Backend Role (now has TWO policies)
module "chatbot_backend_role" {
  policy_arns = [
    aws_iam_policy.chatbot_backend_bedrock.arn,
    aws_iam_policy.chatbot_backend_secrets.arn  # NEW - added in uncommitted changes
  ]
}
```

**Summary of Changes**:
- Removed ~90 lines (entire CSI driver setup)
- Added ~30 lines (Secrets Manager policy)
- Net: Simplified by ~60 lines of code

**Chatbot Application Changes (platform-ai-chatbot) - UNCOMMITTED**:

**File Deleted**: 
- `k8s/templates/chatbot-backend-secret-provider-class.yaml` (59 lines removed)
- **Status**: ⚠️ Uncommitted deletion on feature branch

**File Modified**: `k8s/templates/chatbot-backend-deployment.yaml`
- **Status**: ⚠️ Uncommitted changes on feature branch - **INCOMPLETE**

**What Was Changed** (so far):
```yaml
# Added volumeMounts section (lines 128-131):
volumeMounts:
- name: secrets-store
  mountPath: "/mnt/secrets-store"
  readOnly: true
```

**Current Problem** (needs fixing):
```yaml
# Lines 138-145 still reference CSI driver (MUST BE REMOVED):
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "chatbot-backend-secrets"  # This resource was deleted!
```

**The Issue**: 
- The deployment references `secretProviderClass: "chatbot-backend-secrets"` 
- But this SecretProviderClass file was deleted
- The volume mount was added but never completed with init container approach
- This would cause deployment to fail with "SecretProviderClass not found" error

---

## What Needs to Happen Next

### 1. Complete Infrastructure Cleanup
- [ ] Commit changes to `terraform/modules/eks/main.tf`
- [ ] Test Terraform plan/apply to ensure CSI driver removed
- [ ] Verify chatbot backend IAM role has Secrets Manager permissions

### 2. Implement Init Container Approach

**Option A: Init Container Creates Kubernetes Secret**
```yaml
initContainers:
- name: fetch-secrets
  image: amazon/aws-cli:latest
  command:
    - /bin/sh
    - -c
    - |
      # Fetch secret from AWS Secrets Manager
      SECRET_JSON=$(aws secretsmanager get-secret-value \
        --secret-id platform-db-${ENVIRONMENT}-credentials \
        --region us-east-2 \
        --query SecretString --output text)
      
      # Parse JSON and create K8s Secret
      DB_USER=$(echo $SECRET_JSON | jq -r .username)
      DB_PASSWORD=$(echo $SECRET_JSON | jq -r .password)
      DB_NAME=$(echo $SECRET_JSON | jq -r .dbname)
      DB_HOST=$(echo $SECRET_JSON | jq -r .host)
      DB_PORT=$(echo $SECRET_JSON | jq -r .port)
      
      # Create secret using kubectl or write to shared volume
      kubectl create secret generic chatbot-backend-db-credentials \
        --from-literal=DB_USER=$DB_USER \
        --from-literal=DB_PASSWORD=$DB_PASSWORD \
        --from-literal=DB_NAME=$DB_NAME \
        --from-literal=DB_HOST=$DB_HOST \
        --from-literal=DB_PORT=$DB_PORT \
        --dry-run=client -o yaml | kubectl apply -f -
```

**Option B: Init Container Writes to Shared Volume**
```yaml
initContainers:
- name: fetch-secrets
  image: amazon/aws-cli:latest
  volumeMounts:
  - name: secrets
    mountPath: /secrets
  command:
    - /bin/sh
    - -c
    - |
      aws secretsmanager get-secret-value \
        --secret-id platform-db-${ENVIRONMENT}-credentials \
        --region us-east-2 \
        --query SecretString --output text > /secrets/db-credentials.json

containers:
- name: backend-container
  volumeMounts:
  - name: secrets
    mountPath: /app/secrets
    readOnly: true
  # App reads from /app/secrets/db-credentials.json

volumes:
- name: secrets
  emptyDir:
    medium: Memory  # Store in memory, not disk
```

### 3. Update Deployment Configuration
- [ ] Remove CSI volume mounts from deployment
- [ ] Remove CSI volumes section
- [ ] Add init container (choose Option A or B)
- [ ] Add necessary RBAC if using Option A (ServiceAccount needs secret create/update permissions)
- [ ] Update backend application code if using Option B (read from file instead of env vars)

### 4. Testing Checklist
- [ ] Deploy to dev environment
- [ ] Verify init container successfully fetches secrets
- [ ] Verify backend pod starts successfully
- [ ] Verify backend can connect to RDS
- [ ] Check logs for any authentication errors
- [ ] Test application functionality (chat queries)

---

## Key Learnings

### Why CSI Driver Failed
1. **Complexity**: Multiple moving parts (addon, service account, SecretProviderClass, volume mounts)
2. **Pod Identity Issues**: Authentication between CSI driver and AWS Secrets Manager
3. **Debugging Difficulty**: CSI driver runs in kube-system namespace, logs harder to access
4. **Addon Dependencies**: Requires Pod Identity Agent addon working correctly

### Why Init Container is Better
1. **Simplicity**: Single init container with AWS CLI
2. **Direct Authentication**: Uses Pod Identity directly, no intermediary CSI driver
3. **Easier Debugging**: Init container logs visible in pod describe/logs
4. **Less Infrastructure**: No EKS addon required
5. **More Control**: Full control over secret fetching logic and error handling

### Architecture Comparison

**CSI Driver Flow**:
```
AWS Secrets Manager 
  ↓ (CSI Driver with Pod Identity)
SecretProviderClass 
  ↓ (Creates K8s Secret)
Pod Environment Variables
```

**Init Container Flow (Option A)**:
```
AWS Secrets Manager 
  ↓ (Init container with Pod Identity + AWS CLI)
Kubernetes Secret 
  ↓ (envFrom)
Pod Environment Variables
```

**Init Container Flow (Option B)**:
```
AWS Secrets Manager 
  ↓ (Init container with Pod Identity + AWS CLI)
Shared Memory Volume 
  ↓ (File read)
Application Code
```

---

## Files Summary

### Committed Changes (Already Pushed to main)
**Infrastructure (my-project-Infra)**:
- ✅ `terraform/modules/eks/main.tf` - CSI driver added (commit 00a431e), Backend IAM added (2946c6c)
- ✅ `terraform/modules/eks/variables.tf` - Added `chatbot_namespace` variable (2946c6c)
- ✅ `terraform/dev/main.tf` - Added then removed RDS endpoint parameter (2946c6c → 8f9b6ed)
- ✅ `terraform/prod/main.tf` - Added then removed RDS endpoint parameter (2946c6c → 8f9b6ed)

**Chatbot Application (platform-ai-chatbot)**:
- ✅ `k8s/templates/chatbot-backend-service-account.yaml` - Created (f077a76)
- ✅ `k8s/templates/chatbot-backend-secret-provider-class.yaml` - Created then committed (f077a76)
- ✅ `k8s/templates/chatbot-backend-deployment.yaml` - Modified for CSI driver (f077a76, 54034fa)
- ✅ `k8s/templates/mysql-external-name-service.yaml` - Deleted (54034fa)
- ✅ `k8s/values.yaml`, `k8s/values-dev.yaml`, `k8s/values-prod.yaml` - Modified (f077a76, 54034fa)

### Uncommitted Changes on Feature Branch (⚠️ DECISION NEEDED)

**Infrastructure (my-project-Infra)** - Branch: `fix/remove-csi-driver-use-init-container`
1. ⚠️ `terraform/modules/eks/main.tf` 
   - **Changes**: Removed CSI driver resources (~90 lines), Added Secrets Manager policy to backend role
   - **Status**: Valid and needed for init container approach
   - **Recommendation**: ✅ **COMMIT & PUSH** these changes

**Chatbot Application (platform-ai-chatbot)** - Branch: `fix/remove-csi-driver-use-init-container`
1. ⚠️ `k8s/templates/chatbot-backend-secret-provider-class.yaml`
   - **Changes**: File deleted
   - **Status**: Correct - not needed for init container approach
   - **Recommendation**: ✅ **COMMIT deletion**

2. ⚠️ `k8s/templates/chatbot-backend-deployment.yaml`
   - **Changes**: Added volumeMounts, CSI volume still present
   - **Status**: ❌ **INCOMPLETE** - broken state, needs init container implementation
   - **Recommendation**: ❌ **DO NOT COMMIT** - Complete init container first OR discard changes

### Files Created (Net Total: 1)
1. ✅ `k8s/templates/chatbot-backend-service-account.yaml` (Committed, still needed)
2. ❌ `k8s/templates/chatbot-backend-secret-provider-class.yaml` (Created then deleted)

### Files Deleted (Net Total: 2)
1. ✅ `k8s/templates/mysql-external-name-service.yaml` (Committed deletion)
2. ⚠️ `k8s/templates/chatbot-backend-secret-provider-class.yaml` (Uncommitted deletion)

### Files Modified (Total: 8 unique files)
**Infrastructure** (4 files):
1. `terraform/modules/eks/main.tf` - Multiple changes over 4 commits + uncommitted
2. `terraform/modules/eks/variables.tf` - Added namespace variable (committed)
3. `terraform/dev/main.tf` - Added/removed parameters (committed)
4. `terraform/prod/main.tf` - Added/removed parameters (committed)

**Chatbot Application** (4 files):
5. `k8s/templates/chatbot-backend-deployment.yaml` - Multiple changes (2 committed, 1 uncommitted incomplete)
6. `k8s/values.yaml` - RDS endpoint changes (committed)
7. `k8s/values-dev.yaml` - RDS endpoint changes (committed)  
8. `k8s/values-prod.yaml` - RDS endpoint changes (committed)

---

## Resources & Documentation

### AWS Documentation
- [AWS Secrets Store CSI Driver](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

### Kubernetes Documentation
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

### Best Practices
- Store secrets in memory volumes, not on disk
- Use least-privilege IAM policies (environment-scoped)
- Implement secret rotation strategy
- Monitor secret access via CloudTrail
- Use different secrets per environment

---

## Next Steps Decision Point

**Recommended Approach**: Init Container Option A (Creates Kubernetes Secret)

**Reasoning**:
1. Backend already expects environment variables (no code changes)
2. Existing Helm values structure works as-is
3. Standard Kubernetes pattern
4. Secret rotation can be handled by restarting pods
5. Consistent with current architecture

**Implementation Priority**:
1. High: Complete cleanup of CSI driver remnants
2. High: Implement init container
3. Medium: Add RBAC for secret creation
4. Medium: Test in dev environment
5. Low: Document for team
6. Low: Consider external-secrets operator for future

---

## Commit Messages to Use

### Infrastructure Repo
```
Remove AWS Secrets Store CSI Driver and grant direct Secrets Manager access to chatbot backend

- Remove Secrets Store CSI Driver EKS addon
- Remove CSI driver IAM policy, role, and Pod Identity association
- Add Secrets Manager IAM policy to chatbot backend role
- Simplify secrets management by using init container approach

This change reduces complexity and improves debugging capability
by eliminating the CSI driver intermediary layer.
```

### Chatbot Repo
```
Replace CSI driver with init container for secret management

- Remove SecretProviderClass resource
- Remove CSI volume mounts and volumes from deployment
- Add init container to fetch secrets from AWS Secrets Manager
- Create Kubernetes Secret from fetched values
- Add RBAC permissions for secret creation

This approach provides better visibility, simpler debugging,
and more direct control over secret retrieval.
```

---

**Document Created**: November 29, 2025  
**Last Updated**: November 29, 2025  
**Author**: Learning Session Notes  
**Status**: In Progress - Init Container Implementation Pending
