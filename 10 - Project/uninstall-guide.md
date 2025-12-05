---
tags:
  - My_CV_Project
---
# Complete Uninstall Guide

Complete guide to delete all platform resources in correct order to avoid dependency issues and orphaned resources.

---

## ⚠️ Important Notes

- **Order matters:** Follow steps sequentially to avoid dependency errors
- **Backup data:** Export any important data from RDS before destroying
- **Cost awareness:** Some resources (NAT Gateway, ALB) continue charging until deleted
- **State files:** Keep Terraform state files until all resources confirmed deleted

---

## 🗑️ Deletion Order

### Phase 1: Application Layer (Kubernetes)

Delete application and supporting services running in EKS cluster.

#### 1.1 Delete Chatbot Application
```bash
# Connect to Jenkins EC2 via SSM
aws ssm start-session --target <jenkins-instance-id> --region us-east-2

# Delete Helm release
helm uninstall chatbot -n default

# Verify deletion
kubectl get pods -n default
kubectl get ingress -n default
```

**What gets deleted:**
- Backend/frontend deployments
- Services
- Ingress (triggers ALB deletion)
- Service accounts
- RBAC roles
- Pod Disruption Budgets

#### 1.2 Delete Monitoring Stack
```bash
# Delete Grafana ingress first
kubectl delete ingress grafana-ingress -n monitoring

# Wait for ALB deletion (2-3 minutes)
aws elbv2 describe-load-balancers --region us-east-2 | grep grafana

# Delete Prometheus stack
helm uninstall prometheus -n monitoring

# Delete namespace
kubectl delete namespace monitoring
```

**What gets deleted:**
- Prometheus
- Grafana
- Alertmanager
- Node exporters
- Grafana ALB

#### 1.3 Delete Metrics Server
```bash
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 1.4 Delete AWS Load Balancer Controller
```bash
# Delete controller
helm uninstall aws-load-balancer-controller -n kube-system

# Verify no ALBs remain
aws elbv2 describe-load-balancers --region us-east-2
```

**Important:** If ALBs still exist, delete manually:
```bash
# List ALBs
aws elbv2 describe-load-balancers --region us-east-2 --query 'LoadBalancers[*].[LoadBalancerArn,LoadBalancerName]' --output table

# Delete ALB
aws elbv2 delete-load-balancer --load-balancer-arn <arn> --region us-east-2

# Delete target groups
aws elbv2 describe-target-groups --region us-east-2 --query 'TargetGroups[*].[TargetGroupArn,TargetGroupName]' --output table
aws elbv2 delete-target-group --target-group-arn <arn> --region us-east-2
```

#### 1.5 Delete Jenkins Service Account
```bash
# Delete RBAC
kubectl delete clusterrolebinding jenkins-admin

# Delete service account
kubectl delete serviceaccount jenkins-sa -n default
```

---

### Phase 2: Infrastructure Layer (Terraform)

Delete AWS infrastructure (VPC, EKS, RDS, EC2, networking).

#### 2.1 Verify Kubernetes Resources Deleted
```bash
# Check for any remaining ingresses (would block VPC deletion)
kubectl get ingress --all-namespaces

# Check for any LoadBalancer services
kubectl get svc --all-namespaces -o wide | grep LoadBalancer

# Check for any PVCs (would block EBS volume deletion)
kubectl get pvc --all-namespaces
```

#### 2.2 Run Terraform Destroy
```bash
cd terraform/dev  # or terraform/prod

# Preview what will be deleted
terraform plan -destroy

# Destroy infrastructure
terraform destroy

# Confirm when prompted: yes
```

**What gets deleted:**
- EKS cluster and node groups
- EC2 instances (Jenkins)
- RDS database
- VPC, subnets, route tables
- NAT Gateways, Internet Gateway
- Security groups
- IAM roles and policies
- KMS keys (after 7-day waiting period)
- EBS volumes

**Timing:** ~15-20 minutes

#### 2.3 Verify Deletion
```bash
# Check EKS clusters
aws eks list-clusters --region us-east-2

# Check EC2 instances
aws ec2 describe-instances --region us-east-2 --filters "Name=tag:Name,Values=*platform*" --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output table

# Check RDS instances
aws rds describe-db-instances --region us-east-2 --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus]' --output table

# Check VPCs
aws ec2 describe-vpcs --region us-east-2 --filters "Name=tag:Name,Values=*platform*" --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value|[0]]' --output table

# Check NAT Gateways
aws ec2 describe-nat-gateways --region us-east-2 --filter "Name=state,Values=available" --query 'NatGateways[*].[NatGatewayId,State]' --output table
```

---

### Phase 3: Manual Cleanup (if needed)

Resources that may require manual deletion if Terraform fails.

#### 3.1 Delete Orphaned Network Interfaces
```bash
# List ENIs
aws ec2 describe-network-interfaces --region us-east-2 --filters "Name=description,Values=*EKS*" --query 'NetworkInterfaces[*].[NetworkInterfaceId,Status,Description]' --output table

# Delete ENI (if status = available)
aws ec2 delete-network-interface --network-interface-id <eni-id> --region us-east-2
```

#### 3.2 Delete Security Groups
```bash
# List security groups
aws ec2 describe-security-groups --region us-east-2 --filters "Name=tag:Name,Values=*platform*" --query 'SecurityGroups[*].[GroupId,GroupName]' --output table

# Delete security group
aws ec2 delete-security-group --group-id <sg-id> --region us-east-2
```

#### 3.3 Delete EBS Volumes
```bash
# List volumes
aws ec2 describe-volumes --region us-east-2 --filters "Name=tag:Name,Values=*platform*" --query 'Volumes[*].[VolumeId,State,Size]' --output table

# Delete volume
aws ec2 delete-volume --volume-id <vol-id> --region us-east-2
```

#### 3.4 Delete CloudWatch Log Groups
```bash
# List log groups
aws logs describe-log-groups --region us-east-2 --log-group-name-prefix /aws/eks/platform

# Delete log group
aws logs delete-log-group --log-group-name <name> --region us-east-2
```

---

### Phase 4: Persistent Resources (Optional)

Resources intentionally kept for redeployment or cost reasons.

#### 4.1 RDS Secrets (Keep for Redeployment)
```bash
# List secrets
aws secretsmanager list-secrets --region us-east-2 --filters Key=name,Values=platform-db

# Delete secret (if needed)
aws secretsmanager delete-secret --secret-id platform-db-dev-credentials --region us-east-2 --force-delete-without-recovery
```

#### 4.2 ECR Repository (Keep for Images)
```bash
# List images
aws ecr list-images --repository-name platform-app --region us-east-2

# Delete specific images
aws ecr batch-delete-image --repository-name platform-app --image-ids imageTag=<tag> --region us-east-2

# Delete entire repository (if needed)
aws ecr delete-repository --repository-name platform-app --region us-east-2 --force
```

#### 4.3 S3 State Bucket (Keep for Terraform State)
```bash
# List objects
aws s3 ls s3://your-terraform-state-bucket --region us-east-2

# Delete bucket (if needed - WARNING: loses Terraform state)
aws s3 rb s3://your-terraform-state-bucket --region us-east-2 --force
```

#### 4.4 Packer AMIs (Keep for Fast Redeployment)
```bash
# List AMIs
aws ec2 describe-images --owners self --region us-east-2 --filters "Name=name,Values=jenkins-*" --query 'Images[*].[ImageId,Name,CreationDate]' --output table

# Deregister AMI
aws ec2 deregister-image --image-id <ami-id> --region us-east-2

# Delete associated snapshot
aws ec2 describe-snapshots --owner-ids self --region us-east-2 --filters "Name=description,Values=*jenkins*"
aws ec2 delete-snapshot --snapshot-id <snap-id> --region us-east-2
```

---

## 🔍 Verification Checklist

After deletion, verify no resources remain:

```bash
# EKS
aws eks list-clusters --region us-east-2

# EC2
aws ec2 describe-instances --region us-east-2 --filters "Name=instance-state-name,Values=running,stopped" --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]' --output table

# RDS
aws rds describe-db-instances --region us-east-2

# VPC
aws ec2 describe-vpcs --region us-east-2 --filters "Name=isDefault,Values=false"

# NAT Gateways
aws ec2 describe-nat-gateways --region us-east-2 --filter "Name=state,Values=available,pending"

# ALBs
aws elbv2 describe-load-balancers --region us-east-2

# EBS Volumes
aws ec2 describe-volumes --region us-east-2 --filters "Name=status,Values=available"

# Security Groups (excluding default)
aws ec2 describe-security-groups --region us-east-2 --filters "Name=group-name,Values=*platform*"
```

---

## 💰 Cost Impact

**Immediate savings after deletion:**
- NAT Gateway: ~$35-70/month
- EKS cluster: ~$73/month
- EC2 instances: ~$30/month
- RDS: ~$15-30/month
- ALB: ~$16/month
- EBS volumes: ~$2-6/month

**Total:** ~$177-294/month (dev/prod)

---

## 🔄 Redeployment

To redeploy from scratch:

1. **Keep these resources:**
   - S3 state bucket
   - RDS secrets in Secrets Manager
   - ECR repository
   - Packer AMIs (optional - can rebuild)

2. **Run deployment:**
   ```bash
   # Build AMI (if deleted)
   cd packer && packer build jenkins.pkr.hcl
   
   # Deploy infrastructure
   cd terraform/dev && terraform apply
   
   # Configure Jenkins and run pipelines
   ```

3. **Timing:** ~30-40 minutes for full redeployment

---

## ⚠️ Common Issues

### Issue 1: VPC Deletion Fails
**Error:** `DependencyViolation: The vpc has dependencies and cannot be deleted`

**Cause:** Orphaned ENIs, security groups, or load balancers

**Solution:**
```bash
# Find and delete ENIs
aws ec2 describe-network-interfaces --region us-east-2 --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 delete-network-interface --network-interface-id <eni-id>

# Delete load balancers
aws elbv2 describe-load-balancers --region us-east-2
aws elbv2 delete-load-balancer --load-balancer-arn <arn>
```

### Issue 2: Security Group Deletion Fails
**Error:** `DependencyViolation: resource has a dependent object`

**Cause:** Security group referenced by another security group

**Solution:**
```bash
# Remove all ingress/egress rules first
aws ec2 revoke-security-group-ingress --group-id <sg-id> --ip-permissions <permissions>
aws ec2 revoke-security-group-egress --group-id <sg-id> --ip-permissions <permissions>

# Then delete
aws ec2 delete-security-group --group-id <sg-id>
```

### Issue 3: EKS Cluster Deletion Hangs
**Cause:** Pods with finalizers or PVCs not deleted

**Solution:**
```bash
# Force delete stuck pods
kubectl delete pod <pod-name> --grace-period=0 --force

# Remove finalizers from stuck resources
kubectl patch pvc <pvc-name> -p '{"metadata":{"finalizers":null}}'
```

---

## 📝 Notes

- **Terraform state:** Keep state files until all resources verified deleted
- **Secrets rotation:** If redeploying, consider rotating RDS passwords
- **DNS records:** Update or delete Route 53/Namecheap records pointing to deleted ALBs
- **Cost monitoring:** Check AWS Cost Explorer 24 hours after deletion to verify no charges

---

**Last Updated:** 2025-01-XX
