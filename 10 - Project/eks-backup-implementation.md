---
tags:
  - My_CV_Project
  - Kubernetes
  - AWS
  - EKS
---
# AWS Backup for EKS Implementation Guide

Complete guide to implement AWS Backup for Amazon EKS with full cluster backup and restore capabilities using the new December 2024 features.

---

## Overview

**What:** Automated backup of entire EKS cluster (Kubernetes resources + persistent volumes)  
**Why:** Disaster recovery, compliance, point-in-time restore  
**New Feature:** AWS Backup for EKS (December 2024) - backs up all Kubernetes resources, not just EBS volumes

**Difficulty:** Easy (30 minutes)  
**Cost:** ~$5-30/month depending on retention and cluster size

---

## What Gets Backed Up

### Included in Backup
✅ **All Kubernetes resources** - Deployments, Services, ConfigMaps, Secrets, Ingress  
✅ **Persistent volumes** - EBS snapshots with application-consistent backups  
✅ **RBAC** - Roles, RoleBindings, ClusterRoles, ServiceAccounts  
✅ **Custom resources** - CRDs and their instances  
✅ **Namespace metadata** - Labels, annotations, resource quotas  
✅ **Network policies** - If implemented  
✅ **Pod Identity associations** - IAM role mappings

### Not Included
❌ **Running pod state** - In-memory data (restart required after restore)  
❌ **External resources** - RDS, S3, external services (backup separately)  
❌ **Cluster configuration** - Node groups, addons (recreated from Terraform)

---

## Phase 1: Add Terraform Resources

### Step 1.1: Add Backup IAM Role

Add to `terraform/modules/eks/main.tf`:

```hcl
# ========================================
# AWS Backup for EKS
# ========================================
# Automated backup of EKS cluster including all Kubernetes resources and persistent volumes
# Uses AWS Backup service (December 2024 feature) for complete cluster backup

# IAM role for AWS Backup service
resource "aws_iam_role" "eks_backup" {
  name        = "${var.cluster_name}-eks-backup-role"
  description = "IAM role for AWS Backup to backup EKS cluster"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "backup.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
  
  tags = {
    Name        = "${var.cluster_name}-eks-backup-role"
    Environment = var.environment
  }
}

# AWS managed policy for backup operations
resource "aws_iam_role_policy_attachment" "eks_backup_policy" {
  role       = aws_iam_role.eks_backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"
}

# AWS managed policy for restore operations
resource "aws_iam_role_policy_attachment" "eks_backup_restore_policy" {
  role       = aws_iam_role.eks_backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores"
}

# Additional permissions for EKS-specific operations
resource "aws_iam_role_policy" "eks_backup_additional" {
  name = "${var.cluster_name}-eks-backup-additional"
  role = aws_iam_role.eks_backup.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EKSDescribe"
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListClusters"
        ]
        Resource = "*"
      },
      {
        Sid    = "EBSSnapshot"
        Effect = "Allow"
        Action = [
          "ec2:CreateSnapshot",
          "ec2:DeleteSnapshot",
          "ec2:DescribeSnapshots",
          "ec2:DescribeVolumes",
          "ec2:CreateTags"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Step 1.2: Add Backup Vault

```hcl
# Backup vault for storing recovery points
resource "aws_backup_vault" "eks" {
  name        = "${var.cluster_name}-backup-vault"
  kms_key_arn = aws_kms_key.this.arn  # Use existing EKS KMS key for encryption
  
  tags = {
    Name        = "${var.cluster_name}-backup-vault"
    Environment = var.environment
  }
}

# Vault lock configuration (optional - prevents deletion)
resource "aws_backup_vault_lock_configuration" "eks" {
  count               = var.environment == "prod" ? 1 : 0
  backup_vault_name   = aws_backup_vault.eks.name
  min_retention_days  = 7
}
```

### Step 1.3: Add Backup Plan

```hcl
# Backup plan with daily schedule
resource "aws_backup_plan" "eks" {
  name = "${var.cluster_name}-backup-plan"
  
  # Daily backup rule
  rule {
    rule_name         = "daily-eks-backup"
    target_vault_name = aws_backup_vault.eks.name
    schedule          = "cron(0 2 * * ? *)"  # 2 AM UTC daily
    start_window      = 60                    # Start within 1 hour of scheduled time
    completion_window = 180                   # Complete within 3 hours
    
    lifecycle {
      delete_after = var.environment == "prod" ? 30 : 7  # Retention: 30 days prod, 7 days dev
    }
    
    recovery_point_tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Cluster     = var.cluster_name
    }
  }
  
  # Weekly backup rule (prod only)
  dynamic "rule" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      rule_name         = "weekly-eks-backup"
      target_vault_name = aws_backup_vault.eks.name
      schedule          = "cron(0 3 ? * SUN *)"  # 3 AM UTC every Sunday
      start_window      = 60
      completion_window = 180
      
      lifecycle {
        delete_after = 90  # Keep weekly backups for 90 days
      }
      
      recovery_point_tags = {
        Environment = var.environment
        ManagedBy   = "terraform"
        Cluster     = var.cluster_name
        Type        = "weekly"
      }
    }
  }
  
  tags = {
    Name        = "${var.cluster_name}-backup-plan"
    Environment = var.environment
  }
}
```

### Step 1.4: Add Backup Selection

```hcl
# Backup selection - which resources to backup
resource "aws_backup_selection" "eks" {
  name         = "${var.cluster_name}-backup-selection"
  plan_id      = aws_backup_plan.eks.id
  iam_role_arn = aws_iam_role.eks_backup.arn
  
  # Backup the entire EKS cluster
  resources = [
    aws_eks_cluster.this.arn
  ]
  
  # Optional: Only backup resources with specific tag
  # Uncomment to enable selective backup
  # selection_tag {
  #   type  = "STRINGEQUALS"
  #   key   = "backup"
  #   value = "true"
  # }
}
```

### Step 1.5: Add Variables (Optional)

Add to `terraform/modules/eks/variables.tf`:

```hcl
variable "backup_enabled" {
  description = "Enable AWS Backup for EKS cluster"
  type        = bool
  default     = true
}

variable "backup_retention_days" {
  description = "Number of days to retain daily backups"
  type        = number
  default     = 7
}

variable "backup_schedule" {
  description = "Cron expression for backup schedule (UTC)"
  type        = string
  default     = "cron(0 2 * * ? *)"  # 2 AM UTC daily
}
```

### Step 1.6: Add Outputs

Add to `terraform/modules/eks/outputs.tf`:

```hcl
output "backup_vault_name" {
  description = "Name of the backup vault"
  value       = aws_backup_vault.eks.name
}

output "backup_plan_id" {
  description = "ID of the backup plan"
  value       = aws_backup_plan.eks.id
}

output "backup_role_arn" {
  description = "ARN of the backup IAM role"
  value       = aws_iam_role.eks_backup.arn
}
```

---

## Phase 2: Deploy Backup Infrastructure

### Step 2.1: Apply Terraform Changes

```bash
# Development environment
cd terraform/dev
terraform plan  # Review changes
terraform apply

# Production environment
cd terraform/prod
terraform plan
terraform apply
```

### Step 2.2: Verify Backup Resources Created

```bash
# List backup plans
aws backup list-backup-plans --region us-east-2

# Expected output:
# {
#   "BackupPlansList": [
#     {
#       "BackupPlanName": "platform-dev-backup-plan",
#       "BackupPlanId": "...",
#       "CreationDate": "..."
#     }
#   ]
# }

# Describe backup plan
aws backup get-backup-plan --backup-plan-id <plan-id> --region us-east-2

# List backup selections
aws backup list-backup-selections --backup-plan-id <plan-id> --region us-east-2

# Verify backup vault
aws backup describe-backup-vault --backup-vault-name platform-dev-backup-vault --region us-east-2
```

---

## Phase 3: Test Backup

### Step 3.1: Trigger Manual Backup (Optional)

```bash
# Get cluster ARN
CLUSTER_ARN=$(aws eks describe-cluster --name platform-dev --region us-east-2 --query 'cluster.arn' --output text)

# Get backup role ARN
BACKUP_ROLE_ARN=$(aws iam get-role --role-name platform-dev-eks-backup-role --query 'Role.Arn' --output text)

# Trigger manual backup
aws backup start-backup-job \
  --backup-vault-name platform-dev-backup-vault \
  --resource-arn $CLUSTER_ARN \
  --iam-role-arn $BACKUP_ROLE_ARN \
  --region us-east-2

# Note the BackupJobId from output
```

### Step 3.2: Monitor Backup Progress

```bash
# Check backup job status
aws backup describe-backup-job --backup-job-id <job-id> --region us-east-2

# Expected states: CREATED → RUNNING → COMPLETED
# Backup typically takes 15-30 minutes depending on cluster size

# List all backup jobs
aws backup list-backup-jobs \
  --by-resource-arn $CLUSTER_ARN \
  --region us-east-2
```

### Step 3.3: Verify Backup Completed

```bash
# List recovery points (backups)
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2

# Expected output shows recovery point with Status: COMPLETED

# Get recovery point details
aws backup describe-recovery-point \
  --backup-vault-name platform-dev-backup-vault \
  --recovery-point-arn <recovery-point-arn> \
  --region us-east-2
```

---

## Phase 4: Restore Testing (Non-Production)

### Step 4.1: List Available Backups

```bash
# List all recovery points for cluster
aws backup list-recovery-points-by-resource \
  --resource-arn $CLUSTER_ARN \
  --region us-east-2 \
  --query 'RecoveryPoints[*].[RecoveryPointArn,CreationDate,Status]' \
  --output table
```

### Step 4.2: Restore to New Cluster (Test)

```bash
# Restore cluster with new name
aws backup start-restore-job \
  --recovery-point-arn <recovery-point-arn> \
  --iam-role-arn $BACKUP_ROLE_ARN \
  --metadata '{
    "ClusterName": "platform-dev-restored",
    "RoleArn": "arn:aws:iam::586794447516:role/EKS-cluster-role-dev",
    "SubnetIds": "subnet-xxx,subnet-yyy",
    "SecurityGroupIds": "sg-xxx"
  }' \
  --region us-east-2

# Note the RestoreJobId from output
```

### Step 4.3: Monitor Restore Progress

```bash
# Check restore job status
aws backup describe-restore-job --restore-job-id <job-id> --region us-east-2

# Restore typically takes 20-40 minutes

# Verify restored cluster
aws eks describe-cluster --name platform-dev-restored --region us-east-2
```

### Step 4.4: Verify Restored Resources

```bash
# Configure kubectl for restored cluster
aws eks update-kubeconfig --name platform-dev-restored --region us-east-2

# Check all resources restored
kubectl get all --all-namespaces

# Check persistent volumes
kubectl get pv

# Check secrets
kubectl get secrets --all-namespaces

# Verify application pods
kubectl get pods -n default
```

### Step 4.5: Cleanup Test Restore

```bash
# Delete restored cluster (test only)
aws eks delete-cluster --name platform-dev-restored --region us-east-2

# Delete associated resources via AWS Console or CLI
```

---

## Phase 5: Selective Backup (Optional)

### Enable Namespace-Level Backup

Tag specific namespaces to backup:

```bash
# Tag namespace for backup
kubectl label namespace default backup=true
kubectl label namespace monitoring backup=true

# Exclude namespace from backup
kubectl label namespace kube-system backup=false
```

Update Terraform backup selection:

```hcl
resource "aws_backup_selection" "eks" {
  # ... existing config ...
  
  # Only backup namespaces with backup=true label
  selection_tag {
    type  = "STRINGEQUALS"
    key   = "backup"
    value = "true"
  }
}
```

---

## Monitoring & Alerts

### CloudWatch Alarms for Backup Failures

Add to `terraform/modules/eks/main.tf`:

```hcl
resource "aws_cloudwatch_metric_alarm" "backup_failed" {
  alarm_name          = "${var.cluster_name}-backup-failed"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "NumberOfBackupJobsFailed"
  namespace           = "AWS/Backup"
  period              = 86400  # 24 hours
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alert when EKS backup job fails"
  treat_missing_data  = "notBreaching"
  
  dimensions = {
    BackupVaultName = aws_backup_vault.eks.name
  }
  
  # Add SNS topic for notifications (optional)
  # alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### Check Backup Status

```bash
# List recent backup jobs
aws backup list-backup-jobs \
  --by-backup-vault-name platform-dev-backup-vault \
  --max-results 10 \
  --region us-east-2

# Check for failed backups
aws backup list-backup-jobs \
  --by-state FAILED \
  --region us-east-2
```

---

## Cost Analysis

### Development Environment
- **Backup storage:** ~$0.05/GB/month
- **EBS snapshots:** ~$0.05/GB/month
- **Estimated cluster size:** 50GB (OS + apps + PVs)
- **Daily backups (7 days retention):** 50GB × 7 × $0.05 = ~$17.50/month
- **Total:** ~$5-10/month (incremental snapshots reduce cost)

### Production Environment
- **Daily backups (30 days retention):** 50GB × 30 × $0.05 = ~$75/month
- **Weekly backups (90 days retention):** 50GB × 13 × $0.05 = ~$32.50/month
- **Total:** ~$20-30/month (incremental snapshots reduce cost)

**Note:** Costs are estimates. Actual costs depend on:
- Cluster size and data volume
- Change rate (incremental backups)
- Retention period
- Cross-region copy (if enabled)

---

## Best Practices

### 1. Test Restores Regularly

```bash
# Monthly restore test (automated via Jenkins)
# 1. Trigger restore to test cluster
# 2. Verify all resources present
# 3. Run smoke tests
# 4. Delete test cluster
```

### 2. Retention Strategy

**Development:**
- Daily backups: 7 days
- No weekly backups (cost optimization)

**Production:**
- Daily backups: 30 days
- Weekly backups: 90 days
- Monthly backups: 1 year (optional)

### 3. Backup Validation

```bash
# Verify backup completed successfully
aws backup describe-recovery-point \
  --backup-vault-name platform-dev-backup-vault \
  --recovery-point-arn <arn> \
  --region us-east-2 \
  --query 'Status'

# Expected: COMPLETED
```

### 4. Cross-Region Backup (Disaster Recovery)

```hcl
# Add copy rule to backup plan
rule {
  rule_name = "daily-eks-backup"
  # ... existing config ...
  
  copy_action {
    destination_vault_arn = "arn:aws:backup:us-west-2:586794447516:backup-vault:platform-dev-backup-vault-dr"
    
    lifecycle {
      delete_after = 30
    }
  }
}
```

---

## Troubleshooting

### Issue 1: Backup Job Fails - "Access Denied"

**Symptom:** Backup job fails with IAM permission error

**Cause:** Missing IAM permissions

**Solution:**
```bash
# Verify IAM role has required policies
aws iam list-attached-role-policies --role-name platform-dev-eks-backup-role

# Check inline policies
aws iam list-role-policies --role-name platform-dev-eks-backup-role

# Re-apply Terraform if policies missing
cd terraform/dev && terraform apply
```

### Issue 2: Backup Takes Too Long

**Symptom:** Backup exceeds completion window (3 hours)

**Cause:** Large cluster or many persistent volumes

**Solution:**
```hcl
# Increase completion window
rule {
  completion_window = 360  # 6 hours
}
```

### Issue 3: Restore Fails - "Subnet Not Found"

**Symptom:** Restore job fails with subnet error

**Cause:** Subnets deleted or incorrect subnet IDs in metadata

**Solution:**
```bash
# Get current subnet IDs from Terraform
cd terraform/dev
terraform output

# Use correct subnet IDs in restore metadata
```

### Issue 4: High Backup Costs

**Symptom:** Backup costs higher than expected

**Cause:** Full backups instead of incremental

**Solution:**
```bash
# Check backup sizes
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2 \
  --query 'RecoveryPoints[*].[CreationDate,BackupSizeInBytes]'

# Reduce retention or enable lifecycle policies
```

---

## Disaster Recovery Procedures

### Scenario 1: Cluster Corruption

```bash
# 1. Identify last known good backup
aws backup list-recovery-points-by-resource \
  --resource-arn $CLUSTER_ARN \
  --region us-east-2

# 2. Restore to new cluster
aws backup start-restore-job \
  --recovery-point-arn <arn> \
  --iam-role-arn $BACKUP_ROLE_ARN \
  --metadata '{"ClusterName":"platform-dev-restored"}' \
  --region us-east-2

# 3. Update DNS to point to new cluster ALB
# 4. Verify application functionality
# 5. Delete corrupted cluster
```

### Scenario 2: Accidental Resource Deletion

```bash
# 1. Find backup before deletion
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2

# 2. Restore specific resources (not full cluster)
# Note: Requires manual extraction from backup
# Use kubectl apply with backed-up manifests
```

### Scenario 3: Region Failure

```bash
# 1. Restore from cross-region backup
aws backup start-restore-job \
  --recovery-point-arn <cross-region-arn> \
  --iam-role-arn $BACKUP_ROLE_ARN \
  --region us-west-2

# 2. Update Route 53 to point to new region
# 3. Verify application functionality
```

---

## Compliance & Audit

### Backup Compliance Report

```bash
# List all backups with retention
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2 \
  --query 'RecoveryPoints[*].[RecoveryPointArn,CreationDate,Status,Lifecycle.DeleteAt]' \
  --output table

# Export to CSV for audit
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2 \
  --output json > backup-audit-$(date +%Y%m%d).json
```

### Backup Vault Access Logs

```bash
# Enable CloudTrail logging for backup operations
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::Backup::BackupVault \
  --region us-east-2
```

---

## Cleanup (If Disabling Backups)

```bash
# 1. Delete all recovery points
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name platform-dev-backup-vault \
  --region us-east-2 \
  --query 'RecoveryPoints[*].RecoveryPointArn' \
  --output text | xargs -I {} aws backup delete-recovery-point \
  --backup-vault-name platform-dev-backup-vault \
  --recovery-point-arn {} \
  --region us-east-2

# 2. Remove Terraform resources
# Comment out or delete backup resources in terraform/modules/eks/main.tf

# 3. Apply Terraform
cd terraform/dev && terraform apply
```

---

## References

- [AWS Backup for Amazon EKS](https://docs.aws.amazon.com/aws-backup/latest/devguide/backing-up-eks.html)
- [AWS Backup Pricing](https://aws.amazon.com/backup/pricing/)
- [EKS Disaster Recovery Best Practices](https://docs.aws.amazon.com/eks/latest/userguide/disaster-recovery-resiliency.html)

---

**Last Updated:** 2025-01-XX
