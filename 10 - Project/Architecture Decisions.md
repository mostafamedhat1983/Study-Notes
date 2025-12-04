# Architecture Decisions - Detailed Explanation

Complete breakdown of all technical approaches used in both repositories, including requirements, alternatives, and decision rationale.

---

## Table of Contents

1. [EKS Authentication & Authorization](#1-eks-authentication--authorization)
2. [Persistent Storage](#2-persistent-storage)
3. [Secrets Management](#3-secrets-management)
4. [Networking Architecture](#4-networking-architecture)
5. [Infrastructure as Code](#5-infrastructure-as-code)
6. [CI/CD Infrastructure](#6-cicd-infrastructure)
7. [Database Configuration](#7-database-configuration)
8. [Container Security](#8-container-security)
9. [High Availability](#9-high-availability)
10. [Application Architecture](#10-application-architecture)
11. [Monitoring & Observability](#11-monitoring--observability)
12. [Image Management](#12-image-management)
13. [Cost Optimization](#13-cost-optimization)

---

## 1. EKS Authentication & Authorization

### 1.1 EKS Pod Identity vs OIDC/IRSA

**What We Used:** EKS Pod Identity

**Requirements:**
- EKS cluster version 1.24+
- Pod Identity addon installed (`eks-pod-identity-agent`)
- IAM role with trust policy for `pods.eks.amazonaws.com`
- Pod Identity association linking role to service account
- Service account in Kubernetes manifest

**Setup Example:**
```terraform
# 1. Create IAM role
module "chatbot_backend_role" {
  source  = "../role"
  name    = "platform-dev-chatbot-backend"
  service = "pods.eks.amazonaws.com"  # Pod Identity trust policy
  policy_arns = [
    aws_iam_policy.chatbot_backend_bedrock.arn
  ]
}

# 2. Install Pod Identity addon
resource "aws_eks_addon" "pod_identity_agent" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "eks-pod-identity-agent"
}

# 3. Create association
resource "aws_eks_pod_identity_association" "chatbot_backend" {
  cluster_name    = aws_eks_cluster.this.name
  namespace       = "default"
  service_account = "chatbot-backend-service-account"
  role_arn        = module.chatbot_backend_role.role_arn
}

# 4. Reference in Kubernetes
# k8s/templates/chatbot-backend-deployment.yaml
spec:
  serviceAccountName: chatbot-backend-service-account  # Links to Pod Identity
```

**Alternative 1: OIDC + IRSA (IAM Roles for Service Accounts)**

**Requirements:**
- OIDC provider created for EKS cluster
- IAM role with trust policy for OIDC provider
- Service account annotation with role ARN
- More complex trust policy with conditions

**Setup:**
```terraform
# 1. Create OIDC provider
data "tls_certificate" "eks" {
  url = aws_eks_cluster.this.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.this.identity[0].oidc[0].issuer
}

# 2. Create IAM role with complex trust policy
resource "aws_iam_role" "chatbot_backend" {
  name = "platform-dev-chatbot-backend"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:chatbot-backend-service-account"
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# 3. Annotate service account in Kubernetes
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chatbot-backend-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/platform-dev-chatbot-backend
```

**Alternative 2: Static AWS Credentials (Anti-Pattern)**

**Requirements:**
- Access keys (access key ID + secret access key)
- Kubernetes Secret storing credentials
- Environment variables or volume mounts

**Setup:**
```yaml
# Store credentials in Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
type: Opaque
data:
  access-key-id: <base64-encoded>
  secret-access-key: <base64-encoded>

# Reference in pod
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: access-key-id
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: secret-access-key
```

**Alternative 3: EC2 Instance Profiles**

Only works if pods run on EC2 nodes with instance profile, but:
- All pods on that node share same permissions (no isolation)
- Can't scope permissions per application
- Violates least privilege principle

**Why We Chose Pod Identity:**

| Feature | Pod Identity | OIDC/IRSA | Static Keys | Instance Profile |
|---------|-------------|-----------|-------------|------------------|
| **Complexity** | ✅ Simple | ❌ Complex | ✅ Simple | ✅ Simple |
| **Security** | ✅ Temporary credentials | ✅ Temporary credentials | ❌ Long-lived | ⚠️ Shared |
| **Per-pod permissions** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Setup steps** | ✅ 3 steps | ❌ 5+ steps | ⚠️ Manual rotation | ✅ 2 steps |
| **Credential rotation** | ✅ Automatic | ✅ Automatic | ❌ Manual | ✅ Automatic |
| **AWS recommendation** | ✅ Latest (2023) | ⚠️ Legacy | ❌ Not recommended | ❌ Not recommended |
| **Debugging** | ✅ Clear errors | ⚠️ OIDC errors cryptic | ✅ Clear | ⚠️ Unclear |

**Decision:** Pod Identity is AWS's modern approach (released 2023), simpler than OIDC, and more secure than static keys.

---

### 1.2 EKS Access Entry API vs aws-auth ConfigMap

**What We Used:** EKS Access Entry API

**Requirements:**
- EKS cluster with `authentication_mode = "API"` or `"API_AND_CONFIG_MAP"`
- IAM principal (user/role) ARN
- Access policy association (optional, for fine-grained control)

**Setup:**
```terraform
# 1. Enable API mode in cluster
resource "aws_eks_cluster" "this" {
  name = "platform-dev"
  
  access_config {
    authentication_mode = "API"  # Modern approach
  }
}

# 2. Create access entry for Jenkins
resource "aws_eks_access_entry" "jenkins" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = "arn:aws:iam::123456789012:role/platform-dev-jenkins"
  type          = "STANDARD"
}

# 3. Associate policy (cluster admin)
resource "aws_eks_access_policy_association" "jenkins" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = "arn:aws:iam::123456789012:role/platform-dev-jenkins"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  
  access_scope {
    type = "cluster"  # Can also scope to specific namespaces
  }
}
```

**Alternative: aws-auth ConfigMap (Deprecated)**

**Requirements:**
- Manual ConfigMap editing in `kube-system` namespace
- YAML formatting must be perfect (common source of errors)
- No Terraform state tracking
- Difficult to audit changes

**Setup:**
```bash
# 1. Manually edit ConfigMap
kubectl edit configmap aws-auth -n kube-system

# 2. Add entry in mapRoles section
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/platform-dev-jenkins
      username: jenkins
      groups:
        - system:masters
```

**Comparison:**

| Feature | Access Entry API | aws-auth ConfigMap |
|---------|-----------------|-------------------|
| **AWS recommendation** | ✅ Current (2023+) | ⚠️ Deprecated |
| **Terraform managed** | ✅ Yes | ❌ No (manual kubectl) |
| **Version control** | ✅ In Terraform state | ❌ Manual tracking |
| **Error prone** | ✅ Validated by API | ❌ YAML formatting errors common |
| **Rollback** | ✅ Terraform state | ❌ Manual |
| **Audit trail** | ✅ CloudTrail | ⚠️ Limited |
| **Fine-grained control** | ✅ Namespace-scoped policies | ⚠️ All or nothing |

**Why We Chose Access Entry API:**
- AWS's modern approach (aws-auth ConfigMap will be removed eventually)
- Fully managed in Terraform (no manual kubectl commands)
- Better error handling and validation
- Supports fine-grained namespace-scoped permissions
- CloudTrail audit trail for access changes

---

## 2. Persistent Storage

### 2.1 EBS CSI Driver with Pod Identity

**What We Used:** EBS CSI Driver with Pod Identity authentication

**Requirements:**
- Pod Identity addon installed
- IAM role with `AmazonEBSCSIDriverPolicy` managed policy
- Pod Identity association for `ebs-csi-controller-sa` service account
- EBS CSI Driver addon installed

**Setup:**
```terraform
# 1. Create IAM role for EBS CSI Driver
module "ebs_csi_driver_role" {
  source = "../role"
  
  name    = "platform-dev-ebs-csi-driver"
  service = "pods.eks.amazonaws.com"  # Pod Identity trust
  
  policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  ]
}

# 2. Create Pod Identity association
resource "aws_eks_pod_identity_association" "ebs_csi_driver" {
  cluster_name    = aws_eks_cluster.this.name
  namespace       = "kube-system"
  service_account = "ebs-csi-controller-sa"  # Created by addon
  role_arn        = module.ebs_csi_driver_role.role_arn
  
  depends_on = [aws_eks_addon.pod_identity_agent]
}

# 3. Install EBS CSI Driver addon
resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "aws-ebs-csi-driver"
  
  depends_on = [aws_eks_pod_identity_association.ebs_csi_driver]
}

# 4. Use in Kubernetes with PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-workspace-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3  # EBS CSI Driver storage class
  resources:
    requests:
      storage: 20Gi
```

**Alternative 1: EBS CSI Driver with OIDC/IRSA**

**Requirements:**
- OIDC provider configured
- IAM role with OIDC trust policy
- Service account annotation

**Setup:**
```terraform
# Complex OIDC trust policy
resource "aws_iam_role" "ebs_csi_driver" {
  name = "platform-dev-ebs-csi-driver"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }]
  })
}

# Annotate service account
kubectl annotate serviceaccount ebs-csi-controller-sa \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/platform-dev-ebs-csi-driver
```

**Alternative 2: EFS (Elastic File System)**

**Use Case:** Shared storage across multiple pods/nodes

**Setup:**
```terraform
# Create EFS file system
resource "aws_efs_file_system" "this" {
  encrypted = true
}

# EFS mount targets in each AZ
resource "aws_efs_mount_target" "this" {
  for_each = var.private_subnets
  
  file_system_id  = aws_efs_file_system.this.id
  subnet_id       = aws_subnet.private[each.key].id
  security_groups = [aws_security_group.efs.id]
}

# EFS CSI Driver
resource "aws_eks_addon" "efs_csi_driver" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "aws-efs-csi-driver"
}

# Use in Kubernetes
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods can read/write
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
```

**Alternative 3: Instance Store (Ephemeral)**

**Use Case:** Temporary data that doesn't need persistence

- Free (included with instance)
- Fastest performance
- Data lost on pod restart
- No CSI driver needed

**Alternative 4: No Persistent Storage**

Some workloads don't need persistent storage (stateless applications).

**Comparison:**

| Feature | EBS CSI | EFS | Instance Store | No Storage |
|---------|---------|-----|----------------|-----------|
| **Performance** | ✅ Good (SSD) | ⚠️ Lower | ✅ Fastest | N/A |
| **Cost** | ⚠️ $0.08/GB/mo | ❌ $0.30/GB/mo | ✅ Free | ✅ Free |
| **Persistence** | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| **Access mode** | ⚠️ Single pod | ✅ Multi-pod | ⚠️ Single pod | N/A |
| **Use case** | Databases, Jenkins | Shared files | Caches, temp | Stateless apps |
| **Backup** | ✅ EBS snapshots | ✅ EFS backup | ❌ None | N/A |

**Why We Chose EBS CSI with Pod Identity:**
- Jenkins workspace needs persistent storage (build artifacts, workspace state)
- Single pod access sufficient (ReadWriteOnce)
- Cost-effective ($0.08/GB/month vs EFS $0.30/GB/month)
- Pod Identity simpler than OIDC setup
- Better performance than EFS for single-pod workloads

---

## 3. Secrets Management

### 3.1 Init Containers + Secrets Manager vs CSI Driver

**What We Used:** Init Containers fetching from AWS Secrets Manager

**Requirements:**
- AWS Secrets Manager secret created
- IAM policy allowing `secretsmanager:GetSecretValue`
- Pod Identity association for service account
- Init container with AWS CLI and kubectl
- Main container reads from Kubernetes Secret

**Setup:**
```terraform
# 1. Create secret in Secrets Manager (manual, outside Terraform)
aws secretsmanager create-secret \
  --name platform-db-dev-credentials \
  --secret-string '{"username":"admin","password":"secret","dbname":"platformdb"}'

# 2. IAM policy for secrets access
resource "aws_iam_policy" "chatbot_backend_secrets" {
  name = "platform-dev-chatbot-backend-secrets"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = "arn:aws:secretsmanager:us-east-2:*:secret:platform-db-*"
    }]
  })
}

# 3. Attach to Pod Identity role
module "chatbot_backend_role" {
  source = "../role"
  name   = "platform-dev-chatbot-backend"
  service = "pods.eks.amazonaws.com"
  policy_arns = [
    aws_iam_policy.chatbot_backend_secrets.arn
  ]
}

# 4. Init container in deployment
initContainers:
- name: fetch-secrets
  image: amazon/aws-cli:latest
  command:
    - /bin/sh
    - -c
    - |
      # Install kubectl
      curl -LO "https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl"
      chmod +x kubectl && mv kubectl /usr/local/bin/
      
      # Fetch secret from Secrets Manager
      SECRET_JSON=$(aws secretsmanager get-secret-value \
        --secret-id platform-db-dev-credentials \
        --region us-east-2 \
        --query SecretString \
        --output text)
      
      # Parse JSON
      DB_USER=$(echo "$SECRET_JSON" | python3 -c "import sys, json; print(json.load(sys.stdin)['username'])")
      DB_PASSWORD=$(echo "$SECRET_JSON" | python3 -c "import sys, json; print(json.load(sys.stdin)['password'])")
      
      # Create Kubernetes Secret
      kubectl create secret generic chatbot-backend-db-credentials \
        --from-literal=DB_USER="$DB_USER" \
        --from-literal=DB_PASSWORD="$DB_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -

# 5. Main container reads from K8s Secret
envFrom:
- secretRef:
    name: chatbot-backend-db-credentials
```

**Alternative 1: AWS Secrets Store CSI Driver**

**Requirements:**
- Secrets Store CSI Driver installed
- AWS Provider for Secrets Store CSI Driver
- SecretProviderClass custom resource
- Volume mount in pod
- OIDC or Pod Identity for authentication

**Setup:**
```terraform
# 1. Install Secrets Store CSI Driver
resource "helm_release" "secrets_store_csi_driver" {
  name       = "secrets-store-csi-driver"
  repository = "https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts"
  chart      = "secrets-store-csi-driver"
  namespace  = "kube-system"
  
  set {
    name  = "syncSecret.enabled"
    value = "true"
  }
}

# 2. Install AWS Provider
resource "helm_release" "secrets_store_csi_driver_provider_aws" {
  name       = "secrets-store-csi-driver-provider-aws"
  repository = "https://aws.github.io/secrets-store-csi-driver-provider-aws"
  chart      = "secrets-store-csi-driver-provider-aws"
  namespace  = "kube-system"
}

# 3. Create SecretProviderClass
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "platform-db-dev-credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: DB_USER
          - path: password
            objectAlias: DB_PASSWORD

# 4. Mount as volume
spec:
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "aws-secrets"
  
  containers:
  - name: backend
    volumeMounts:
      - name: secrets-store
        mountPath: "/mnt/secrets"
        readOnly: true
    env:
      - name: DB_USER
        valueFrom:
          secretKeyRef:
            name: chatbot-backend-db-credentials
            key: DB_USER
```

**Alternative 2: Kubernetes Secrets (Manual)**

**Setup:**
```bash
# Create secret manually
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=secret

# Reference in pod
envFrom:
- secretRef:
    name: db-credentials
```

**Cons:** 
- Secrets in base64 only (not encrypted at rest by default)
- No rotation mechanism
- Manual management

**Alternative 3: Environment Variables in Deployment**

**Anti-pattern:** Never do this

```yaml
env:
  - name: DB_PASSWORD
    value: "my-secret-password"  # ❌ Visible in Git, kubectl describe
```

**Alternative 4: External Secrets Operator**

**Modern GitOps approach:**

```yaml
# Install External Secrets Operator
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# Create SecretStore pointing to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

# Create ExternalSecret that syncs automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h  # Auto-refresh
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: chatbot-backend-db-credentials
  data:
  - secretKey: DB_USER
    remoteRef:
      key: platform-db-dev-credentials
      property: username
  - secretKey: DB_PASSWORD
    remoteRef:
      key: platform-db-dev-credentials
      property: password
```

**Comparison:**

| Feature | Init Container | CSI Driver | Manual K8s Secret | External Secrets |
|---------|---------------|-----------|-------------------|-----------------|
| **Debugging** | ✅ Logs in app pod | ❌ Logs in kube-system | N/A | ⚠️ Separate operator |
| **Complexity** | ✅ Simple | ❌ Complex (2 Helm charts) | ✅ Simple | ⚠️ Medium |
| **Auto-rotation** | ❌ Manual pod restart | ✅ Yes | ❌ No | ✅ Yes |
| **Secret updates** | ⚠️ Pod restart needed | ✅ Automatic | ❌ Manual | ✅ Automatic |
| **Infrastructure code** | ✅ ~60 lines | ❌ ~150 lines | ✅ Minimal | ⚠️ ~100 lines |
| **AWS integration** | ✅ Direct API | ✅ Via CSI | ❌ None | ✅ Via operator |
| **Error visibility** | ✅ Clear AWS errors | ⚠️ CSI layer errors | N/A | ⚠️ Operator errors |
| **Multi-secret support** | ⚠️ One script per secret | ✅ Multiple in one class | N/A | ✅ Multiple sources |

**Why We Chose Init Containers:**
- **Better debugging:** Logs visible in application pod (`kubectl logs <pod> -c fetch-secrets`)
- **Simpler architecture:** No additional Helm charts or operators
- **Direct error visibility:** AWS API errors shown immediately
- **Sufficient for use case:** DB credentials rarely rotate (manual pod restart acceptable)
- **Less infrastructure code:** ~60 lines vs ~150 for CSI driver
- **Full control:** Complete visibility into secret retrieval logic

**When to use alternatives:**
- **CSI Driver:** Frequent secret rotation, multiple secrets per pod
- **External Secrets Operator:** GitOps workflow, multiple secret sources (Vault, GCP, Azure)
- **Manual K8s Secrets:** Dev/testing only

---

## 4. Networking Architecture

### 4.1 NAT Gateway Strategy

**What We Used:** 
- Dev: 1 NAT Gateway (cost-optimized)
- Prod: 2 NAT Gateways (high availability)

**Requirements:**
- Internet Gateway in VPC
- Elastic IP for each NAT Gateway
- NAT Gateway in public subnet
- Route table entries in private subnets

**Setup:**
```terraform
# Dynamic NAT Gateway count based on environment
locals {
  nat_subnets = var.nat_gateway_count == 1 ? {
    # Dev: Use only first public subnet
    for k, v in var.public_subnets : k => v if k == keys(var.public_subnets)[0]
  } : {
    # Prod: Use all public subnets (one NAT per AZ)
    for k, v in var.public_subnets : k => v
  }
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  for_each = local.nat_subnets
  domain   = "vpc"
}

# NAT Gateways
resource "aws_nat_gateway" "this" {
  for_each      = local.nat_subnets
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.public[each.key].id
}

# Private route tables
resource "aws_route_table" "private" {
  for_each = var.private_subnets
  vpc_id   = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    # Dev: All traffic through single NAT
    # Prod: Traffic through AZ-specific NAT
    nat_gateway_id = var.nat_gateway_count == 1 ? 
      aws_nat_gateway.this[local.first_nat_key].id : 
      aws_nat_gateway.this[each.value.availability_zone].id
  }
}

# Usage in main.tf
# Dev
nat_gateway_count = 1

# Prod
nat_gateway_count = 2
```

**Alternative 1: VPC Endpoints (No NAT Gateway)**

**Use Case:** Cost optimization for AWS service access

**Setup:**
```terraform
# ECR VPC Endpoints (3 needed for full ECR access)
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.us-east-2.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [for s in aws_subnet.private : s.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.us-east-2.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [for s in aws_subnet.private : s.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.this.id
  service_name      = "com.amazonaws.us-east-2.s3"
  vpc_endpoint_type = "Gateway"  # Free for S3
  route_table_ids   = [for rt in aws_route_table.private : rt.id]
}

# Secrets Manager endpoint
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.us-east-2.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [for s in aws_subnet.private : s.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

**Cost Comparison:**

| Approach | Cost (us-east-2) | Pros | Cons |
|----------|-----------------|------|------|
| **1 NAT Gateway** | $32/mo + data | ✅ Simple, all traffic | ❌ Single point of failure |
| **2 NAT Gateways** | $64/mo + data | ✅ High availability | ⚠️ Higher cost |
| **VPC Endpoints** | ~$7-10/mo per endpoint | ✅ Lower data transfer cost | ❌ Complex, per-service setup |
| **NAT + Endpoints** | $32/mo + $20-30/mo | ✅ Best of both | ⚠️ Most expensive |

**Data transfer costs:**
- NAT Gateway: $0.045/GB processed
- VPC Endpoint: $0.01/GB processed
- Typical workload: ~100GB/month = $4.50 via NAT vs $1 via endpoints

**Alternative 2: NAT Instance (EC2)**

**Cost:** ~$5-10/month for t3.micro

**Setup:**
```terraform
# Launch NAT instance
resource "aws_instance" "nat" {
  ami           = "ami-nat-instance"  # AWS NAT AMI
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public["us-east-2a"].id
  
  source_dest_check = false  # Required for NAT
  
  tags = {
    Name = "nat-instance"
  }
}

# Route through NAT instance
resource "aws_route" "private_nat_instance" {
  route_table_id         = aws_route_table.private["us-east-2a"].id
  destination_cidr_block = "0.0.0.0/0"
  instance_id            = aws_instance.nat.id
}
```

**Cons:**
- Manual management (patching, updates)
- Single point of failure
- Lower throughput than NAT Gateway
- Not recommended by AWS

**Alternative 3: No Outbound Internet (Fully Private)**

**Use Case:** Maximum security, air-gapped

**Requirements:**
- VPC endpoints for ALL AWS services
- Private Docker registry (not ECR)
- Internal package mirrors
- Complex setup and maintenance

**Why We Chose This Approach:**

**Dev (1 NAT Gateway):**
- **Cost-optimized:** $32/month vs $64 for 2 NATs
- **Acceptable risk:** Dev downtime OK if NAT fails
- **Simple:** Single route configuration

**Prod (2 NAT Gateways):**
- **High availability:** One NAT per AZ
- **No downtime:** If us-east-2a NAT fails, us-east-2b continues
- **AWS recommendation:** Multi-AZ for production

**Why NOT VPC Endpoints (for now):**
- Workload has low data transfer (~50-100GB/month)
- Cost savings minimal (~$3-4/month)
- NAT Gateway simpler (one resource vs 5+ endpoints)
- Can add endpoints later if data transfer costs increase

---

### 4.2 Public vs Private Subnets

**What We Used:** Public + Private subnet architecture

**Architecture:**
```
VPC (10.0.0.0/16)
├── Public Subnets (2 subnets)
│   ├── us-east-2a: 10.0.0.0/24
│   └── us-east-2b: 10.0.1.0/24
│   └── Contains: NAT Gateways, (future: Load Balancers)
│
└── Private Subnets (6 subnets)
    ├── Jenkins Subnets (2)
    │   ├── us-east-2a: 10.0.10.0/24
    │   └── us-east-2b: 10.0.11.0/24
    │
    ├── EKS Subnets (2)
    │   ├── us-east-2a: 10.0.20.0/24
    │   └── us-east-2b: 10.0.21.0/24
    │
    └── RDS Subnets (2)
        ├── us-east-2a: 10.0.30.0/24
        └── us-east-2b: 10.0.31.0/24
```

**Setup:**
```terraform
# Public subnet configuration
resource "aws_subnet" "public" {
  for_each = var.public_subnets
  
  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = true  # Auto-assign public IPs
  
  tags = {
    "kubernetes.io/role/elb" = "1"  # For public load balancers
  }
}

# Public route table (direct to Internet Gateway)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
}

# Private subnet configuration
resource "aws_subnet" "private" {
  for_each = var.private_subnets
  
  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = false  # No public IPs
  
  tags = {
    "kubernetes.io/role/internal-elb" = "1"  # For internal load balancers
  }
}

# Private route table (via NAT Gateway)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this["us-east-2a"].id
  }
}
```

**Alternative 1: All Public Subnets**

```terraform
resource "aws_subnet" "public_only" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = "10.0.0.0/24"
  map_public_ip_on_launch = true
}

# Direct internet access
resource "aws_route_table" "public_only" {
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
}
```

**Pros:**
- No NAT Gateway cost
- Direct internet access (faster)
- Simpler routing

**Cons:**
- ❌ Security risk (all resources publicly accessible by default)
- ❌ No defense in depth
- ❌ Violates AWS best practices
- ❌ Compliance issues (PCI-DSS, HIPAA)

**Alternative 2: All Private Subnets (No Internet)**

Requires VPC endpoints for everything.

**Pros:**
- Maximum security
- Air-gapped architecture

**Cons:**
- Complex setup
- Higher cost (many VPC endpoints)
- Difficult package management

**Alternative 3: Single Subnet (Flat Network)**

**Anti-pattern:** Don't do this

**Cons:**
- No network segmentation
- Can't isolate database from internet
- No defense in depth

**Why We Chose Public + Private:**

**Public Subnets (NAT Gateways only):**
- ✅ Minimal attack surface
- ✅ Future-ready for load balancers
- ✅ Controlled internet egress point

**Private Subnets (Jenkins, EKS, RDS):**
- ✅ No direct internet exposure
- ✅ Defense in depth
- ✅ AWS best practice
- ✅ Compliance-friendly

**Subnet Separation (Jenkins, EKS, RDS):**
- ✅ Network segmentation
- ✅ Future-ready for Network ACLs
- ✅ Easier security group management
- ✅ Clear resource organization

---

## 5. Infrastructure as Code

### 5.1 S3 Native State Locking vs DynamoDB

**What We Used:** S3 native state locking (2024 feature)

**Requirements:**
- Terraform >= 1.7.0
- S3 bucket with versioning enabled
- `use_lockfile = true` in backend config

**Setup:**
```terraform
# terraform/dev/backend.tf
terraform {
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "dev/terraform.tfstate"
    region = "us-east-2"
    
    # S3 native locking (Terraform 1.7+)
    use_lockfile = true
    
    # Optional: Encryption
    encrypt = true
  }
}

# S3 bucket configuration
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-terraform-state-bucket"
}

# Enable versioning (required)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Optional: Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**How it works:**
1. Terraform creates `.terraform.lock.s3` file in S3 bucket
2. File contains lock ID and timestamp
3. Other Terraform processes check for lock file
4. If lock exists, wait or fail
5. Lock released when operation completes

**Alternative 1: DynamoDB Locking (Legacy)**

**Requirements:**
- S3 bucket for state
- DynamoDB table for locks
- Additional IAM permissions

**Setup:**
```terraform
# Create DynamoDB table for locks
resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-state-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Backend configuration
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-east-2"
    
    # DynamoDB locking (legacy)
    dynamodb_table = "terraform-state-locks"
    
    encrypt = true
  }
}

# Required IAM permissions
{
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "dynamodb:PutItem",
      "dynamodb:GetItem",
      "dynamodb:DeleteItem"
    ],
    "Resource": "arn:aws:dynamodb:*:*:table/terraform-state-locks"
  }]
}
```

**Alternative 2: Terraform Cloud**

**Features:**
- Remote execution
- State storage included
- Built-in locking
- Team collaboration
- Policy as Code

**Setup:**
```terraform
terraform {
  cloud {
    organization = "your-org"
    
    workspaces {
      name = "dev-environment"
    }
  }
}
```

**Cost:** Free for up to 5 users, then $20/user/month

**Alternative 3: No Locking (Dangerous)**

Only for single-user, non-critical environments.

```terraform
terraform {
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "dev/terraform.tfstate"
    region = "us-east-2"
    # No locking configured
  }
}
```

**Risk:** Concurrent applies can corrupt state.

**Comparison:**

| Feature | S3 Native Lock | DynamoDB Lock | Terraform Cloud | No Locking |
|---------|---------------|---------------|-----------------|-----------|
| **Cost** | ✅ Free | ⚠️ ~$0.50/mo | ❌ $20/user/mo | ✅ Free |
| **Setup complexity** | ✅ 1 resource | ⚠️ 2 resources | ✅ Simple | ✅ None |
| **IAM permissions** | ✅ S3 only | ⚠️ S3 + DynamoDB | N/A | ✅ S3 only |
| **Lock reliability** | ✅ Reliable | ✅ Reliable | ✅ Reliable | ❌ None |
| **Terraform version** | ⚠️ 1.7+ required | ✅ All versions | ✅ All versions | N/A |
| **Troubleshooting** | ✅ Simple (S3 file) | ⚠️ Check DynamoDB | ⚠️ UI only | N/A |

**Why We Chose S3 Native Locking:**
- **Cost:** Free vs $0.50/month for DynamoDB
- **Simplicity:** One less resource to manage
- **Modern:** Latest Terraform feature (2024)
- **Fewer IAM permissions:** S3 only, no DynamoDB access needed
- **Easier debugging:** Lock file visible in S3 console
- **Sufficient:** Same reliability as DynamoDB for our use case

**When to use alternatives:**
- **DynamoDB:** Terraform < 1.7, need proven legacy approach
- **Terraform Cloud:** Team collaboration, need CI/CD integration, policy enforcement
- **No locking:** Personal dev only (never for shared environments)

---

### 5.2 Modular Terraform Architecture

**What We Used:** Custom reusable modules

**Structure:**
```
terraform/
├── dev/                    # Development environment
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── provider.tf
│   └── backend.tf
├── prod/                   # Production environment
│   └── (same structure)
└── modules/                # Reusable modules
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    ├── eks/
    ├── rds/
    ├── ec2/
    └── role/               # Flexible IAM role module
```

**Module Usage:**
```terraform
# terraform/dev/main.tf

# Network module
module "network" {
  source = "../modules/network"
  
  vpc_name              = "platform-dev-vpc"
  vpc_cidr_block        = "10.0.0.0/16"
  nat_gateway_count     = 1  # Dev-specific
  environment           = "dev"
  cluster_name          = "platform-dev"
  # ... other variables
}

# EKS module
module "eks" {
  source = "../modules/eks"
  
  cluster_name         = "platform-dev"
  cluster_version      = "1.34"
  node_desired_size    = 2  # Dev-specific
  node_min_size        = 1
  node_max_size        = 3
  instance_types       = ["t3.small"]  # Dev-specific
  # ... dependencies from network module
  subnet_ids           = module.network.eks_subnet_ids
}

# Reusable IAM role module (Pod Identity)
module "chatbot_backend_role" {
  source = "../modules/role"
  
  name    = "platform-dev-chatbot-backend"
  service = "pods.eks.amazonaws.com"  # Pod Identity
  
  policy_arns = [
    aws_iam_policy.chatbot_backend_bedrock.arn
  ]
}

# Same role module for different use case (EC2)
module "jenkins_role" {
  source = "../modules/role"
  
  name    = "platform-dev-jenkins"
  service = "ec2.amazonaws.com"  # EC2 instance profile
  
  policy_arns = [
    "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  ]
}
```

**Flexible Role Module:**
```terraform
# modules/role/main.tf
resource "aws_iam_role" "this" {
  name = var.name

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = var.service  # Can be pods.eks.amazonaws.com, ec2.amazonaws.com, etc.
      }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each = toset(var.policy_arns)
  
  role       = aws_iam_role.this.name
  policy_arn = each.value
}

# modules/role/variables.tf
variable "name" {
  description = "IAM role name"
  type        = string
}

variable "service" {
  description = "AWS service for trust policy (pods.eks.amazonaws.com, ec2.amazonaws.com, etc.)"
  type        = string
}

variable "policy_arns" {
  description = "List of IAM policy ARNs to attach"
  type        = list(string)
  default     = []
}
```

**Alternative 1: Terraform Registry Modules**

**Example:**
```terraform
# Using official AWS VPC module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "platform-dev-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-2a", "us-east-2b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

**Pros:**
- ✅ Well-tested
- ✅ Community-maintained
- ✅ Best practices built-in
- ✅ Quick to implement

**Cons:**
- ❌ Less control over implementation
- ❌ May include unnecessary features
- ❌ Harder to customize
- ❌ Version updates can break things
- ❌ Learning curve to understand module internals

**Alternative 2: Monolithic Configuration (No Modules)**

**All in one file:**
```terraform
# Everything in main.tf
resource "aws_vpc" "dev" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "dev_public_2a" {
  vpc_id     = aws_vpc.dev.id
  cidr_block = "10.0.0.0/24"
}

# ... 50+ more resources in same file
```

**Cons:**
- ❌ Code duplication between dev/prod
- ❌ Hard to maintain
- ❌ No reusability
- ❌ Difficult to test individual components

**Alternative 3: Workspaces (Same code, different state)**

```terraform
# Single codebase for all environments
resource "aws_eks_cluster" "this" {
  name = "platform-${terraform.workspace}"
  
  # Different values based on workspace
  version = terraform.workspace == "prod" ? "1.34" : "1.33"
}
```

**Cons:**
- ❌ Hard to have truly different configurations
- ❌ Risk of accidental prod changes
- ❌ Less explicit than separate directories

**Why We Chose Custom Modules:**

**Reusability:**
- ✅ Same network module for dev and prod
- ✅ Flexible role module for EC2, EKS, Pod Identity
- ✅ DRY principle (Don't Repeat Yourself)

**Control:**
- ✅ Full control over implementation
- ✅ Customize to exact requirements
- ✅ No unnecessary features

**Learning:**
- ✅ Understand AWS resources deeply
- ✅ Not just "using someone else's code"
- ✅ Better troubleshooting

**Maintainability:**
- ✅ Environment-specific values in dev/prod directories
- ✅ Shared logic in modules
- ✅ Clear dependencies between modules

**Environment Differences Made Easy:**
```terraform
# Dev: Cost-optimized
nat_gateway_count = 1
instance_types    = ["t3.small"]
node_desired_size = 2

# Prod: High availability
nat_gateway_count = 2
instance_types    = ["t3.medium"]
node_desired_size = 3
```

---

## 6. CI/CD Infrastructure

### 6.1 Packer + Ansible vs User Data

**What We Used:** Packer + Ansible for immutable AMI builds

**Requirements:**
- Packer installed locally
- Ansible playbook for configuration
- AWS credentials for AMI creation
- Source AMI (Amazon Linux 2023)

**Setup:**
```hcl
# packer/jenkins.pkr.hcl
packer {
  required_plugins {
    amazon = {
      source  = "github.com/hashicorp/amazon"
      version = "~> 1"
    }
    ansible = {
      source  = "github.com/hashicorp/ansible"
      version = "~> 1"
    }
  }
}

source "amazon-ebs" "jenkins" {
  ami_name      = "jenkins-master-{{timestamp}}"
  instance_type = "t3.medium"
  region        = "us-east-2"
  source_ami_filter {
    filters = {
      name                = "al2023-ami-*-x86_64"
      virtualization-type = "hvm"
    }
    owners      = ["amazon"]
    most_recent = true
  }
  
  ssh_username = "ec2-user"
  
  tags = {
    Name        = "Jenkins Master AMI"
    BuildTime   = "{{timestamp}}"
    Environment = "production"
  }
}

build {
  sources = ["source.amazon-ebs.jenkins"]

  # Provision with Ansible
  provisioner "ansible" {
    playbook_file = "./ansible/jenkins-playbook.yml"
  }
  
  # Vulnerability scanning
  provisioner "shell" {
    inline = [
      "curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.67.2",
      "trivy rootfs --severity CRITICAL --exit-code 1 /"
    ]
  }

  # Save AMI ID to manifest
  post-processor "manifest" {
    output     = "manifest.json"
    strip_path = true
  }
}
```

**Ansible Playbook:**
```yaml
# packer/ansible/jenkins-playbook.yml
---
- name: Configure Jenkins Master AMI
  hosts: all
  become: true
  tasks:
    # System updates
    - name: Update all system packages
      ansible.builtin.dnf:
        name: "*"
        state: latest
        update_cache: true
    
    # Install packages
    - name: Install Docker, Java 21, Git
      ansible.builtin.dnf:
        name:
          - docker
          - java-21-amazon-corretto
          - git
        state: present
    
    # Jenkins setup
    - name: Import Jenkins GPG key
      ansible.builtin.rpm_key:
        state: present
        key: https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    
    - name: Add Jenkins repository
      ansible.builtin.yum_repository:
        name: jenkins
        description: Jenkins stable repository
        baseurl: https://pkg.jenkins.io/redhat-stable/
        gpgcheck: true
        gpgkey: https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    
    - name: Install Jenkins
      ansible.builtin.dnf:
        name: jenkins
        state: present
    
    # Add jenkins user to docker group
    - name: Add jenkins user to docker group
      ansible.builtin.user:
        name: jenkins
        groups: docker
        append: true
    
    # Start services
    - name: Start and enable services
      ansible.builtin.systemd:
        name: "{{ item }}"
        state: started
        enabled: true
      loop:
        - docker
        - jenkins
    
    # Install kubectl
    - name: Add Kubernetes repository
      ansible.builtin.yum_repository:
        name: kubernetes
        description: Kubernetes
        baseurl: https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
        gpgcheck: true
        gpgkey: https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
        exclude: kubelet kubeadm
    
    - name: Install kubectl
      ansible.builtin.dnf:
        name: kubectl
        state: present
    
    # Install Helm
    - name: Download Helm installation script
      ansible.builtin.get_url:
        url: https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
        dest: /tmp/get-helm-3.sh
        mode: '0755'
    
    - name: Install Helm
      ansible.builtin.command: /tmp/get-helm-3.sh
      args:
        creates: /usr/local/bin/helm
    
    # Cleanup
    - name: Clean package manager cache
      ansible.builtin.command: dnf clean all
      changed_when: false
```

**Build Process:**
```bash
cd packer
packer init jenkins.pkr.hcl
packer build jenkins.pkr.hcl

# Output:
# ==> amazon-ebs.jenkins: AMI: ami-0a1b2c3d4e5f6g7h8
# Saved to manifest.json
```

**Use in Terraform:**
```terraform
# Read AMI ID from Packer manifest
data "local_file" "packer_manifest" {
  filename = "${path.module}/../../packer/manifest.json"
}

locals {
  packer_ami = jsondecode(data.local_file.packer_manifest.content).builds[0].artifact_id
  ami_id     = split(":", local.packer_ami)[1]
}

# Launch EC2 with Packer-built AMI
resource "aws_instance" "jenkins" {
  ami           = local.ami_id
  instance_type = "t3.medium"
  # ... other configuration
}
```

**Alternative 1: User Data Scripts**

**Setup:**
```terraform
resource "aws_instance" "jenkins" {
  ami           = "ami-0c55b159cbfafe1f0"  # Base Amazon Linux 2023
  instance_type = "t3.medium"
  
  user_data = <<-EOF
    #!/bin/bash
    # Install Docker
    dnf install -y docker
    systemctl enable docker
    systemctl start docker
    
    # Install Java
    dnf install -y java-21-amazon-corretto
    
    # Install Jenkins
    wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    dnf install -y jenkins
    systemctl enable jenkins
    systemctl start jenkins
    
    # Add jenkins to docker group
    usermod -aG docker jenkins
    
    # Install kubectl
    cat <<REPO > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
    REPO
    dnf install -y kubectl
    
    # Install Helm
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  EOF
}
```

**Alternative 2: AWS Systems Manager (SSM) Documents**

```terraform
resource "aws_ssm_document" "jenkins_setup" {
  name          = "JenkinsSetup"
  document_type = "Command"

  content = jsonencode({
    schemaVersion = "2.2"
    description   = "Install Jenkins"
    mainSteps = [{
      action = "aws:runShellScript"
      name   = "installJenkins"
      inputs = {
        runCommand = [
          "dnf install -y docker java-21-amazon-corretto jenkins",
          "systemctl enable docker jenkins",
          "systemctl start docker jenkins"
        ]
      }
    }]
  })
}

# Run document on instance launch
resource "aws_ssm_association" "jenkins_setup" {
  name = aws_ssm_document.jenkins_setup.name
  
  targets {
    key    = "tag:Name"
    values = ["jenkins-master"]
  }
}
```

**Alternative 3: Configuration Management (Chef, Puppet)**

Similar to Ansible but different tooling.

**Alternative 4: Container-Based Jenkins (ECS/EKS)**

Run Jenkins itself as a container.

**Comparison:**

| Feature | Packer+Ansible | User Data | SSM Documents | Container |
|---------|---------------|-----------|---------------|-----------|
| **Build time** | ⚠️ ~10-15 min | ✅ Instant | ✅ Instant | ✅ Instant |
| **Boot time** | ✅ ~1-2 min | ❌ ~5-10 min | ❌ ~5-10 min | ✅ ~1-2 min |
| **Consistency** | ✅ Identical every time | ❌ Can fail mid-script | ⚠️ Network-dependent | ✅ Identical |
| **Testability** | ✅ Test before deploy | ❌ Test in production | ❌ Test in production | ✅ Test locally |
| **Rollback** | ✅ Use previous AMI | ❌ Hard | ⚠️ Rerun document | ✅ Previous image |
| **Debugging** | ✅ Clear Ansible output | ❌ Check /var/log | ⚠️ SSM logs | ✅ Container logs |
| **Security** | ✅ Scan before deploy | ❌ No scanning | ❌ No scanning | ✅ Scan images |
| **Version control** | ✅ Playbook in Git | ⚠️ Script in Terraform | ⚠️ JSON in Terraform | ✅ Dockerfile in Git |
| **Drift** | ✅ Immutable | ❌ Can drift | ❌ Can drift | ✅ Immutable |

**Why We Chose Packer + Ansible:**

**Immutability:**
- ✅ AMI is immutable (can't be changed after build)
- ✅ No configuration drift over time
- ✅ Same AMI in dev and prod

**Speed:**
- ✅ Instance boots ready-to-use (~2 minutes)
- ✅ No waiting for user data to run
- ✅ Faster autoscaling (if needed)

**Reliability:**
- ✅ Test AMI before production
- ✅ Packer build fails if any step fails
- ✅ Trivy scans for vulnerabilities before AMI creation

**Rollback:**
- ✅ Keep previous AMI versions
- ✅ Instant rollback by changing AMI ID in Terraform

**Debugging:**
- ✅ Clear Ansible task output
- ✅ Can SSH into test instance during build
- ✅ Manifest file shows exactly what was built

**Security:**
- ✅ Vulnerability scanning with Trivy
- ✅ Fail build if CRITICAL vulnerabilities found
- ✅ No secrets in user data (visible in console)

**When to use alternatives:**
- **User Data:** Quick prototypes, one-off instances
- **SSM Documents:** Fleet management, periodic updates
- **Containers:** Jenkins agents (we do this), highly dynamic workloads

---

### 6.2 SSM Session Manager vs SSH/Bastion

**What We Used:** AWS Systems Manager Session Manager

**Requirements:**
- SSM Agent installed on EC2 (pre-installed on Amazon Linux 2023)
- IAM instance profile with `AmazonSSMManagedInstanceCore` policy
- AWS CLI with Session Manager plugin on local machine
- No security group ingress rules needed

**Setup:**
```terraform
# 1. IAM role for Jenkins with SSM permissions
resource "aws_iam_role" "jenkins" {
  name = "platform-dev-jenkins"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}

# 2. Attach SSM managed policy
resource "aws_iam_role_policy_attachment" "jenkins_ssm" {
  role       = aws_iam_role.jenkins.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# 3. Instance profile
resource "aws_iam_instance_profile" "jenkins" {
  name = "platform-dev-jenkins-profile"
  role = aws_iam_role.jenkins.name
}

# 4. Launch instance (no SSH key needed!)
resource "aws_instance" "jenkins" {
  ami                  = local.ami_id
  instance_type        = "t3.medium"
  iam_instance_profile = aws_iam_instance_profile.jenkins.name
  subnet_id            = module.network.jenkins_subnet_ids[0]
  
  # Security group with NO SSH ingress
  vpc_security_group_ids = [module.network.jenkins_sg_id]
  
  # No key pair needed!
  # key_name = null
}

# Security group - no SSH port 22
resource "aws_security_group" "jenkins" {
  name   = "jenkins-sg"
  vpc_id = aws_vpc.this.id

  # Only outbound traffic (no SSH ingress)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Connect to Instance:**
```bash
# Install Session Manager plugin (one-time)
# macOS
brew install --cask session-manager-plugin

# Linux
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb

# Connect via AWS CLI
aws ssm start-session --target i-0123456789abcdef0 --region us-east-2

# Port forwarding (access Jenkins UI)
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=8080,localPortNumber=8080"

# Then access: http://localhost:8080

# Copy files (SCP-like)
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=22,localPortNumber=2222"

scp -P 2222 file.txt localhost:/tmp/
```

**Alternative 1: SSH with Key Pair**

**Setup:**
```terraform
# Create key pair
resource "aws_key_pair" "jenkins" {
  key_name   = "jenkins-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

# Security group with SSH access
resource "aws_security_group_rule" "jenkins_ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]  # ⚠️ Open to internet
  security_group_id = aws_security_group.jenkins.id
}

# Launch instance with key
resource "aws_instance" "jenkins" {
  ami           = local.ami_id
  instance_type = "t3.medium"
  key_name      = aws_key_pair.jenkins.key_name
  
  # Instance needs public IP for SSH access
  associate_public_ip_address = true
}
```

**Connect:**
```bash
ssh -i ~/.ssh/id_rsa ec2-user@<public-ip>
```

**Cons:**
- ❌ Requires managing SSH keys
- ❌ Security group must allow port 22
- ❌ Often open to internet (0.0.0.0/0)
- ❌ Key compromise = full access
- ❌ No audit trail of commands run

**Alternative 2: Bastion Host**

**Architecture:**
```
Local Machine
    ↓ SSH
Bastion Host (public subnet)
    ↓ SSH
Jenkins (private subnet)
```

**Setup:**
```terraform
# Bastion in public subnet
resource "aws_instance" "bastion" {
  ami                         = "ami-xxx"
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public["us-east-2a"].id
  associate_public_ip_address = true
  key_name                    = aws_key_pair.bastion.key_name
  
  vpc_security_group_ids = [aws_security_group.bastion.id]
}

# Bastion security group
resource "aws_security_group" "bastion" {
  name = "bastion-sg"
  
  # Allow SSH from your IP only
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/32"]  # Your IP
  }
}

# Jenkins security group
resource "aws_security_group_rule" "jenkins_from_bastion" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.jenkins.id
}
```

**Connect:**
```bash
# SSH to bastion first
ssh -i ~/.ssh/bastion-key.pem ec2-user@<bastion-public-ip>

# Then SSH to Jenkins
ssh -i ~/.ssh/jenkins-key.pem ec2-user@<jenkins-private-ip>

# Or use SSH tunneling
ssh -i ~/.ssh/bastion-key.pem -L 8080:<jenkins-private-ip>:8080 ec2-user@<bastion-public-ip>
```

**Cons:**
- ❌ Additional instance to manage and pay for (~$7/month)
- ❌ Bastion is publicly accessible
- ❌ Still requires SSH keys
- ❌ Two-hop connection (slower)
- ❌ Bastion is single point of failure

**Alternative 3: VPN**

Connect your network to VPC via VPN, then SSH directly.

**Cost:** ~$72/month for AWS Client VPN

**Alternative 4: EC2 Instance Connect**

AWS-managed SSH without keys (uses temporary keys).

```bash
aws ec2-instance-connect send-ssh-public-key \
  --instance-id i-xxx \
  --instance-os-user ec2-user \
  --ssh-public-key file://~/.ssh/id_rsa.pub

ssh ec2-user@<public-ip>
```

**Cons:**
- Still requires public IP
- Still requires security group rule for port 22
- Keys only valid for 60 seconds

**Comparison:**

| Feature | SSM Session Manager | SSH + Key | Bastion | VPN |
|---------|-------------------|-----------|---------|-----|
| **Cost** | ✅ Free | ✅ Free | ⚠️ $7/mo | ❌ $72/mo |
| **Security** | ✅ No open ports | ❌ Port 22 open | ⚠️ Bastion exposed | ✅ Private |
| **Key management** | ✅ No keys | ❌ Manage keys | ❌ Manage keys | ⚠️ VPN certs |
| **Public IP needed** | ✅ No | ❌ Yes | ❌ Yes (bastion) | ✅ No |
| **Audit trail** | ✅ CloudTrail logs | ❌ Limited | ⚠️ Bastion logs | ⚠️ VPN logs |
| **Port forwarding** | ✅ Built-in | ⚠️ SSH tunnel | ⚠️ SSH tunnel | ✅ Direct |
| **Session recording** | ✅ Optional | ❌ No | ⚠️ Manual | ❌ No |
| **MFA support** | ✅ Yes | ❌ No | ⚠️ Bastion only | ✅ Yes |
| **Network isolation** | ✅ No ingress rules | ❌ SSH ingress | ❌ SSH ingress | ✅ Full isolation |

**Why We Chose SSM Session Manager:**

**Security:**
- ✅ No SSH ports open (no security group ingress rules)
- ✅ No SSH keys to manage or rotate
- ✅ No bastion host to maintain
- ✅ Works with private instances (no public IP needed)
- ✅ IAM-based access control

**Auditability:**
- ✅ All sessions logged in CloudTrail
- ✅ See who accessed what instance and when
- ✅ Optional session recording (S3 + CloudWatch Logs)

**Cost:**
- ✅ Free (no additional charges)
- ✅ No bastion host costs
- ✅ No VPN costs

**Convenience:**
- ✅ Port forwarding built-in
- ✅ No key files to distribute
- ✅ Works through AWS CLI
- ✅ Multi-region support

**Compliance:**
- ✅ Meets SOC, PCI-DSS requirements
- ✅ Zero-trust architecture
- ✅ Centralized access management

---

## 7. Database Configuration

### 7.1 RDS MySQL Multi-AZ vs Single-AZ

**What We Used:**
- Dev: Single-AZ (cost-optimized)
- Prod: Multi-AZ (high availability)

**Multi-AZ Setup (Production):**
```terraform
resource "aws_db_instance" "this" {
  identifier = "platform-prod"
  
  engine         = "mysql"
  engine_version = "8.0.39"
  instance_class = "db.t3.small"
  
  allocated_storage     = 50
  max_allocated_storage = 100  # Auto-scaling enabled
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  # Multi-AZ for high availability
  multi_az = true  # Creates standby in different AZ
  
  # Network configuration
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [var.rds_sg_id]
  publicly_accessible    = false
  
  # Backups
  backup_retention_period = 7
  backup_window          = "03:00-04:00"  # UTC
  maintenance_window     = "Mon:04:00-Mon:05:00"
  
  # Monitoring
  enabled_cloudwatch_logs_exports = ["error", "general", "slowquery"]
  performance_insights_enabled    = true
  monitoring_interval             = 60  # Enhanced monitoring
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
  
  # Deletion protection
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "platform-prod-final-snapshot"
  
  # Parameter and option groups
  parameter_group_name = aws_db_parameter_group.this.name
  
  tags = {
    Environment = "prod"
  }
}
```

**How Multi-AZ Works:**
1. Primary instance in us-east-2a
2. Synchronous replication to standby in us-east-2b
3. Automatic failover if primary fails (~1-2 minutes)
4. Single endpoint (DNS updated automatically)
5. Zero application code changes needed

**Single-AZ Setup (Development):**
```terraform
resource "aws_db_instance" "this" {
  identifier = "platform-dev"
  
  engine         = "mysql"
  engine_version = "8.0.39"
  instance_class = "db.t3.micro"  # Smaller instance
  
  allocated_storage     = 20  # Less storage
  max_allocated_storage = 50
  storage_encrypted     = true
  
  # Single-AZ (cost-optimized)
  multi_az = false
  
  # Shorter backup retention
  backup_retention_period = 3
  
  # No enhanced monitoring
  monitoring_interval = 0
  
  # No deletion protection (easier to destroy)
  deletion_protection       = false
  skip_final_snapshot      = true
  
  tags = {
    Environment = "dev"
  }
}
```

**Alternative 1: Read Replicas**

**Use Case:** Scale read traffic, not high availability

**Setup:**
```terraform
# Primary instance
resource "aws_db_instance" "primary" {
  identifier     = "platform-prod-primary"
  engine         = "mysql"
  instance_class = "db.t3.small"
  
  backup_retention_period = 7  # Required for read replicas
}

# Read replica (can be in same or different AZ/region)
resource "aws_db_instance" "read_replica" {
  identifier     = "platform-prod-replica"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class = "db.t3.small"
  
  # Can be in different AZ
  availability_zone = "us-east-2b"
}
```

**Application code changes needed:**
```python
# Write to primary
write_connection = connect(primary_endpoint)
write_connection.execute("INSERT INTO users ...")

# Read from replica
read_connection = connect(replica_endpoint)
read_connection.execute("SELECT * FROM users ...")
```

**Cons:**
- ❌ Application must handle two endpoints
- ❌ Replication lag (eventual consistency)
- ❌ Not automatic failover

**Alternative 2: Aurora MySQL**

**AWS's managed MySQL-compatible database:**

```terraform
resource "aws_rds_cluster" "this" {
  cluster_identifier = "platform-prod-aurora"
  engine            = "aurora-mysql"
  engine_version    = "8.0.mysql_aurora.3.04.0"
  database_name     = "platformdb"
  master_username   = "admin"
  
  # Aurora is always multi-AZ
  availability_zones = ["us-east-2a", "us-east-2b", "us-east-2c"]
  
  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"
}

# Instances (one writer, multiple readers)
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "platform-prod-aurora-writer"
  cluster_identifier = aws_rds_cluster.this.id
  instance_class     = "db.t3.small"
  engine             = aws_rds_cluster.this.engine
}

resource "aws_rds_cluster_instance" "reader" {
  count              = 2  # Two read replicas
  identifier         = "platform-prod-aurora-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.this.id
  instance_class     = "db.t3.small"
  engine             = aws_rds_cluster.this.engine
}
```

**Pros:**
- ✅ Faster failover (~30 seconds vs 1-2 minutes)
- ✅ Up to 15 read replicas
- ✅ Storage auto-scales (up to 128TB)
- ✅ Better performance (5x RDS MySQL)

**Cons:**
- ❌ More expensive (~1.5x RDS MySQL)
- ❌ Minimum of 2 instances (writer + reader)
- ⚠️ More complex for simple use cases

**Alternative 3: MySQL on EC2**

**Self-managed MySQL:**

```terraform
resource "aws_instance" "mysql" {
  ami           = "ami-xxx"
  instance_type = "t3.small"
  
  user_data = <<-EOF
    #!/bin/bash
    dnf install -y mysql-server
    systemctl enable mysqld
    systemctl start mysqld
  EOF
}
```

**Cons:**
- ❌ Manual backups
- ❌ Manual patching
- ❌ Manual failover
- ❌ No automatic scaling
- ❌ More operational overhead

**Comparison:**

| Feature | RDS Multi-AZ | RDS Single-AZ | Read Replicas | Aurora | EC2 MySQL |
|---------|-------------|---------------|---------------|--------|-----------|
| **Cost (prod)** | ⚠️ $50/mo | ✅ $25/mo | ⚠️ $50+/mo | ❌ $75/mo | ✅ $15/mo |
| **High availability** | ✅ Auto-failover | ❌ Manual | ❌ Manual | ✅ Auto-failover | ❌ Manual |
| **Failover time** | ⚠️ 1-2 min | N/A | N/A | ✅ ~30 sec | ❌ Hours |
| **Read scaling** | ❌ No | ❌ No | ✅ Yes | ✅ Yes (15 replicas) | ⚠️ Manual |
| **Backups** | ✅ Automated | ✅ Automated | ✅ Automated | ✅ Automated | ❌ Manual |
| **Maintenance** | ✅ Managed | ✅ Managed | ✅ Managed | ✅ Managed | ❌ Self-managed |
| **SSL/TLS** | ✅ Built-in | ✅ Built-in | ✅ Built-in | ✅ Built-in | ⚠️ Manual config |

**Why We Chose This Approach:**

**Development (Single-AZ):**
- ✅ Cost-optimized: $12-15/month vs $25/month
- ✅ Sufficient for testing
- ✅ Acceptable downtime for dev environment
- ✅ Easy to destroy and recreate

**Production (Multi-AZ):**
- ✅ High availability: Automatic failover
- ✅ No application changes needed
- ✅ Data durability: Synchronous replication
- ✅ Maintenance without downtime
- ✅ Cost-effective: ~$50/month vs $75+ for Aurora

**Why NOT Aurora:**
- Workload doesn't need 15 read replicas
- Traffic doesn't justify higher cost
- RDS MySQL sufficient for current scale
- Can migrate to Aurora later if needed

---

### 7.2 SSL/TLS for RDS Connections

**What We Used:** SSL/TLS encryption for MySQL connections

**Requirements:**
- RDS instance with SSL enabled (default)
- Application configured to use SSL
- RDS CA certificate (optional, for verification)

**RDS Configuration:**
```terraform
resource "aws_db_instance" "this" {
  identifier = "platform-dev"
  engine     = "mysql"
  
  # SSL is enabled by default, but can enforce
  parameter_group_name = aws_db_parameter_group.this.name
}

# Parameter group enforcing SSL
resource "aws_db_parameter_group" "this" {
  name   = "platform-dev-mysql"
  family = "mysql8.0"

  # Require SSL for all connections
  parameter {
    name  = "require_secure_transport"
    value = "1"
  }
}
```

**Application Configuration (Python):**
```python
# backend/main.py
import mysql.connector
import ssl

# SSL configuration
ssl_config = {
    'ssl_ca': '/path/to/rds-ca-cert.pem',  # Optional
    'ssl_disabled': False
}

# Create connection with SSL
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=os.getenv('DB_PORT'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME'),
    ssl_ca=ssl_config['ssl_ca'],
    ssl_verify_cert=True  # Verify server certificate
)

# Or using connection string
connection_string = (
    f"mysql://{user}:{password}@{host}:{port}/{database}"
    f"?ssl_ca={ssl_ca}&ssl_verify_cert=true"
)
```

**Download RDS CA Certificate:**
```bash
# Download AWS RDS CA bundle
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

# Include in Docker image
# Dockerfile
COPY global-bundle.pem /usr/local/share/ca-certificates/rds-ca-cert.pem
```

**Alternative 1: No SSL (Anti-Pattern)**

```python
connection = mysql.connector.connect(
    host=host,
    port=port,
    user=user,
    password=password,
    database=database,
    ssl_disabled=True  # ❌ Insecure
)
```

**Cons:**
- ❌ Credentials sent in plaintext
- ❌ Data visible to network sniffers
- ❌ Man-in-the-middle attacks possible
- ❌ Compliance violations

**Alternative 2: IAM Database Authentication**

**Use AWS IAM instead of passwords:**

```terraform
resource "aws_db_instance" "this" {
  identifier = "platform-dev"
  engine     = "mysql"
  
  # Enable IAM authentication
  iam_database_authentication_enabled = true
}

# IAM policy for database access
resource "aws_iam_policy" "rds_connect" {
  name = "platform-dev-rds-connect"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = "rds-db:connect"
      Resource = "arn:aws:rds-db:us-east-2:123456789012:dbuser:${aws_db_instance.this.resource_id}/app_user"
    }]
  })
}
```

**Application Code:**
```python
import boto3
import mysql.connector

# Generate auth token (valid for 15 minutes)
client = boto3.client('rds')
token = client.generate_db_auth_token(
    DBHostname=db_host,
    Port=db_port,
    DBUsername='app_user',
    Region='us-east-2'
)

# Connect using token instead of password
connection = mysql.connector.connect(
    host=db_host,
    port=db_port,
    user='app_user',
    password=token,  # Token, not password
    database=db_name,
    ssl_ca='/path/to/rds-ca-cert.pem'
)
```

**Pros:**
- ✅ No database passwords
- ✅ IAM-based access control
- ✅ Tokens rotate automatically (15-min lifetime)
- ✅ Audit via CloudTrail

**Cons:**
- ⚠️ Requires code changes
- ⚠️ Token generation adds latency
- ⚠️ More complex connection pooling

**Comparison:**

| Feature | SSL/TLS + Password | IAM Auth | No SSL |
|---------|-------------------|----------|--------|
| **Encryption** | ✅ Yes | ✅ Yes | ❌ No |
| **Password needed** | ⚠️ Yes | ✅ No | ⚠️ Yes |
| **Complexity** | ✅ Simple | ⚠️ Medium | ✅ Simple |
| **Performance** | ✅ Fast | ⚠️ Token generation overhead | ✅ Fast |
| **Security** | ✅ Good | ✅ Excellent | ❌ Poor |
| **Compliance** | ✅ Yes | ✅ Yes | ❌ No |

**Why We Chose SSL/TLS with Password:**
- ✅ Encryption for all traffic
- ✅ Simple to implement
- ✅ No performance overhead
- ✅ Works with connection pooling
- ⚠️ Can migrate to IAM auth later if needed

**When to use IAM auth:**
- Zero-trust requirements
- Eliminate passwords entirely
- Short-lived connections (not pooled)
- High security compliance needs

---

## 8. Container Security

### 8.1 Non-Root Containers with UID/GID 1000

**What We Used:** Run containers as non-root user with explicit UID/GID

**Requirements:**
- Create user in Dockerfile with specific UID/GID
- Set ownership of application files
- Configure securityContext in Kubernetes

**Setup:**

**Dockerfile (Backend):**
```dockerfile
FROM python:3.11-slim

# Create non-root user with explicit UID/GID for security
RUN groupadd -r -g 1000 appuser && useradd -r -u 1000 -g 1000 appuser

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .

# Change ownership to appuser
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Kubernetes Deployment:**
```yaml
spec:
  containers:
  - name: backend-container
    image: backend:v1.0.0
    
    securityContext:
      runAsUser: 1000                    # Must match Dockerfile UID
      runAsGroup: 1000                   # Must match Dockerfile GID
      runAsNonRoot: true                 # Enforce non-root
      allowPrivilegeEscalation: false    # Prevent privilege escalation
      capabilities:
        drop:
          - ALL                          # Drop all Linux capabilities
      readOnlyRootFilesystem: false      # App needs to write (logs, temp)
```

**Why UID/GID 1000:**
- Standard practice across industry
- Avoids conflicts with system users (UID < 1000)
- Consistent across dev/prod
- Matches typical Linux user setup

**Alternative 1: Root User (Anti-Pattern)**

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY main.py .

# No USER directive - runs as root (UID 0)
CMD ["python", "main.py"]
```

**Cons:**
- ❌ Container has root privileges
- ❌ Container breakout = root on host
- ❌ Can modify system files
- ❌ Violates least privilege principle
- ❌ Security scanner warnings

**Alternative 2: Random High UID**

**Dockerfile:**
```dockerfile
RUN useradd -r -u 65534 nobody
USER nobody
```

**Kubernetes:**
```yaml
securityContext:
  runAsUser: 65534
```

**Cons:**
- ⚠️ Less predictable
- ⚠️ 65534 often reserved for "nobody"
- ⚠️ Harder to debug file permission issues

**Alternative 3: Kubernetes runAsUser Only**

**No USER in Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
# No user creation
```

**Kubernetes enforces:**
```yaml
securityContext:
  runAsUser: 1000
  fsGroup: 1000  # File system group
```

**Cons:**
- ⚠️ UID 1000 may not exist in container
- ⚠️ Username shows as numeric ID
- ⚠️ No home directory
- ⚠️ Some apps fail without proper user

**Alternative 4: Distroless Images**

**Google's distroless base images:**
```dockerfile
FROM gcr.io/distroless/python3-debian11

# Already runs as non-root (UID 65532)
COPY --chown=nonroot:nonroot . /app
USER nonroot

CMD ["python", "main.py"]
```

**Pros:**
- ✅ Minimal image (no shell, package manager)
- ✅ Smaller attack surface
- ✅ Non-root by default

**Cons:**
- ⚠️ Harder to debug (no shell)
- ⚠️ Can't install packages in running container
- ⚠️ More complex builds

**Comparison:**

| Approach | Security | Debuggability | Compatibility | Image Size |
|----------|----------|---------------|---------------|------------|
| **UID 1000** | ✅ Good | ✅ Easy | ✅ High | ⚠️ Standard |
| **Root** | ❌ Poor | ✅ Easy | ✅ High | ⚠️ Standard |
| **Random UID** | ✅ Good | ⚠️ Medium | ⚠️ Medium | ⚠️ Standard |
| **K8s Only** | ✅ Good | ❌ Harder | ⚠️ App-dependent | ⚠️ Standard |
| **Distroless** | ✅ Excellent | ❌ No shell | ⚠️ App-dependent | ✅ Minimal |

**Why We Chose UID/GID 1000:**
- ✅ Industry standard practice
- ✅ Easy to debug (can exec into container)
- ✅ Works with all applications
- ✅ File permissions predictable
- ✅ Security scanners pass
- ✅ Can still install tools for debugging

---

### 8.2 Dropping Linux Capabilities

**What We Used:** Drop ALL capabilities

**Requirements:**
- Kubernetes 1.19+
- securityContext.capabilities configuration

**Setup:**
```yaml
# k8s/templates/chatbot-backend-deployment.yaml
spec:
  containers:
  - name: backend-container
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL  # Drop all 38 Linux capabilities
```

**What are Linux Capabilities?**

Linux divides root privileges into 38 distinct capabilities:
- `CAP_NET_BIND_SERVICE` - Bind to ports < 1024
- `CAP_NET_RAW` - Use RAW and PACKET sockets
- `CAP_SYS_ADMIN` - Perform system administration tasks
- `CAP_SYS_TIME` - Set system clock
- `CAP_CHOWN` - Change file ownership
- ... 33 more

**Default Docker capabilities:**
```
CAP_CHOWN
CAP_DAC_OVERRIDE
CAP_FOWNER
CAP_FSETID
CAP_KILL
CAP_NET_BIND_SERVICE
CAP_SETGID
CAP_SETUID
CAP_SYS_CHROOT
CAP_AUDIT_WRITE
CAP_NET_RAW
```

**Alternative 1: Keep Default Capabilities**

```yaml
securityContext:
  runAsUser: 1000
  # No capabilities configuration
```

**Cons:**
- ⚠️ Container has 14 capabilities by default
- ⚠️ More attack surface
- ⚠️ Can still do privileged operations

**Alternative 2: Drop Specific Capabilities**

```yaml
securityContext:
  capabilities:
    drop:
      - NET_RAW          # No raw sockets
      - SYS_CHROOT       # No chroot
      - DAC_OVERRIDE     # No permission override
    add:
      - NET_BIND_SERVICE # Need to bind to port 80
```

**Use Case:** Application needs specific capability

**Alternative 3: Add Capabilities (Usually Bad)**

```yaml
securityContext:
  capabilities:
    add:
      - SYS_ADMIN  # ⚠️ Dangerous - almost root
```

**When needed:**
- Ping requires `CAP_NET_RAW`
- tcpdump requires `CAP_NET_RAW` + `CAP_NET_ADMIN`
- iptables requires `CAP_NET_ADMIN`

**Our Application Needs:**

**Backend (FastAPI):**
- Binds to port 8000 (> 1024) ✅ No capability needed
- Reads/writes application files ✅ No capability needed
- Makes HTTP requests ✅ No capability needed
- Connects to MySQL ✅ No capability needed

**Frontend (Streamlit):**
- Binds to port 8501 (> 1024) ✅ No capability needed
- Serves static files ✅ No capability needed
- Makes HTTP requests to backend ✅ No capability needed

**Result:** Drop ALL capabilities safely

**Comparison:**

| Approach | Security | Compatibility | Use Case |
|----------|----------|---------------|----------|
| **Drop ALL** | ✅ Maximum | ✅ Most apps | Standard web apps |
| **Default** | ⚠️ Medium | ✅ All apps | Legacy apps |
| **Selective drop** | ✅ Good | ✅ Most apps | Need specific capabilities |
| **Add capabilities** | ❌ Risky | ✅ All apps | System tools |

**Why We Chose Drop ALL:**
- ✅ Application doesn't need any capabilities
- ✅ Binds to non-privileged ports (>1024)
- ✅ Maximum security posture
- ✅ Defense in depth
- ✅ Best practice for web applications

---

### 8.3 Read-Only Root Filesystem

**What We Did NOT Use:** Read-only root filesystem (yet)

**Current Setup:**
```yaml
securityContext:
  readOnlyRootFilesystem: false  # Application needs to write
```

**Why Not Read-Only:**
- Application writes logs to disk
- Temporary files in /tmp
- Python creates .pyc files
- Would require volume mounts for writable paths

**If We Wanted Read-Only:**
```yaml
spec:
  containers:
  - name: backend-container
    securityContext:
      readOnlyRootFilesystem: true  # Make root filesystem read-only
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp              # Writable temp directory
    - name: cache
      mountPath: /app/.cache       # Writable cache directory
  
  volumes:
  - name: tmp
    emptyDir: {}                   # Temporary storage (lost on restart)
  - name: cache
    emptyDir: {}
```

**Pros of Read-Only:**
- ✅ Prevents container modifications
- ✅ Immutability enforced
- ✅ Malware can't persist

**Cons:**
- ⚠️ Requires code changes
- ⚠️ More complex configuration
- ⚠️ Some apps break

**Future Enhancement:** Consider for next iteration

---

## 9. High Availability

### 9.1 Pod Disruption Budgets (PDB)

**What We Used:** PDB to ensure minimum pod availability

**Requirements:**
- Kubernetes 1.21+
- Multiple replicas
- PDB resource definition

**Setup:**
```yaml
# k8s/templates/chatbot-backend-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chatbot-backend-pdb
spec:
  minAvailable: 1  # Keep at least 1 pod running during disruptions
  selector:
    matchLabels:
      app: chatbot-backend
```

**What PDB Protects Against:**
- Node drains (for maintenance)
- Node upgrades
- Cluster autoscaler scale-downs
- Voluntary pod evictions

**Dev Environment:**
```yaml
# 2 replicas, minAvailable: 1
# During disruption: 1 pod stays up, 1 can be evicted
```

**Prod Environment:**
```yaml
# 3 replicas, minAvailable: 2
# During disruption: 2 pods stay up, 1 can be evicted
```

**Alternative 1: No PDB**

**Risk:**
```bash
# Without PDB, kubectl drain can evict all pods simultaneously
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Result: All pods on node-1 evicted at once
# If all replicas were on node-1: DOWNTIME
```

**Alternative 2: maxUnavailable Instead**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chatbot-backend-pdb
spec:
  maxUnavailable: 1  # Allow at most 1 pod to be down
  selector:
    matchLabels:
      app: chatbot-backend
```

**Comparison:**

| Config | 3 Replicas | During Disruption | Best For |
|--------|-----------|-------------------|----------|
| **minAvailable: 1** | 3 pods | 1 pod must stay up, 2 can be evicted | Cost-optimized |
| **minAvailable: 2** | 3 pods | 2 pods must stay up, 1 can be evicted | High availability |
| **maxUnavailable: 1** | 3 pods | At most 1 pod down, 2 must stay up | Same as minAvailable: 2 |
| **No PDB** | 3 pods | All can be evicted at once | ❌ Not recommended |

**Why We Chose minAvailable:**
- ✅ Explicit guarantee of available pods
- ✅ Clear intent (always have X pods running)
- ✅ Works well with autoscaling
- ✅ Protects against node failures during maintenance

**Real-World Scenario:**
```bash
# Node maintenance triggered
kubectl drain node-1 --ignore-daemonsets

# With PDB:
# 1. Kubernetes checks PDB
# 2. Evicts 1 pod at a time
# 3. Waits for replacement pod to be Ready
# 4. Only then evicts next pod
# Result: Zero downtime

# Without PDB:
# 1. All pods evicted immediately
# 2. Brief downtime until pods reschedule
```

---

### 9.2 Pod Anti-Affinity

**What We Used:** Preferred (soft) anti-affinity

**Requirements:**
- Multiple nodes
- Multiple replicas
- PodAntiAffinity configuration

**Setup:**
```yaml
# k8s/templates/chatbot-backend-deployment.yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: chatbot-backend
              topologyKey: kubernetes.io/hostname  # Spread across nodes
```

**How It Works:**
```
Cluster:
Node-1: [backend-pod-1, backend-pod-2]  # Before anti-affinity
Node-2: []

With Anti-Affinity:
Node-1: [backend-pod-1]
Node-2: [backend-pod-2]  # Spread across nodes
```

**Alternative 1: Required (Hard) Anti-Affinity**

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: chatbot-backend
    topologyKey: kubernetes.io/hostname
```

**Behavior:**
- If 3 replicas but only 2 nodes, 3rd pod stays Pending
- **Strict** requirement

**Cons:**
- ❌ Can prevent scheduling if not enough nodes
- ❌ Pods may stay Pending forever

**Alternative 2: Zone Anti-Affinity**

**Spread across availability zones:**
```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: chatbot-backend
      topologyKey: topology.kubernetes.io/zone  # Spread across AZs
```

**Result:**
```
us-east-2a: [backend-pod-1]
us-east-2b: [backend-pod-2]
us-east-2a: [backend-pod-3]  # Cycles back if needed
```

**Alternative 3: Pod Topology Spread Constraints (Modern)**

**Kubernetes 1.19+ feature:**
```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                    # Max difference between zones
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway  # Soft constraint
    labelSelector:
      matchLabels:
        app: chatbot-backend
```

**More flexible than anti-affinity:**
- Control skew (difference in pod count)
- Better for large clusters
- Newer, recommended approach

**Comparison:**

| Feature | Preferred Anti-Affinity | Required Anti-Affinity | Topology Spread |
|---------|------------------------|----------------------|-----------------|
| **Flexibility** | ✅ Soft (best effort) | ❌ Hard (strict) | ✅ Configurable |
| **Risk of Pending** | ✅ Low | ❌ High | ⚠️ Depends on config |
| **Multi-zone** | ✅ Works | ✅ Works | ✅ Better control |
| **Kubernetes version** | ✅ All versions | ✅ All versions | ⚠️ 1.19+ |

**Why We Chose Preferred Anti-Affinity:**
- ✅ Best effort spreading (won't block scheduling)
- ✅ Works with limited nodes (2 nodes in dev)
- ✅ Simple configuration
- ✅ Good enough for our scale
- ⚠️ Can migrate to Topology Spread later

**Real-World Benefit:**
```
Scenario: Node failure

Without Anti-Affinity:
Node-1: [pod-1, pod-2, pod-3]  ❌ Node fails = all pods down
Node-2: []

With Anti-Affinity:
Node-1: [pod-1, pod-2]
Node-2: [pod-3]  ✅ Node-1 fails = pod-3 still serving traffic
```

---

### 9.3 Rolling Updates

**What We Used:** Rolling update with maxUnavailable=1, maxSurge=1

**Requirements:**
- Multiple replicas
- Update strategy configuration
- Health checks (readiness probes)

**Setup:**
```yaml
# k8s/templates/chatbot-backend-deployment.yaml
spec:
  replicas: 2  # Dev
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # At most 1 pod down during update
      maxSurge: 1        # At most 1 extra pod during update
  
  template:
    spec:
      containers:
      - name: backend-container
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**How Rolling Update Works:**

**Initial State:**
```
Node-1: [v1.0-pod-1]
Node-2: [v1.0-pod-2]
```

**Update to v1.1:**
```
Step 1: Create new pod (surge)
Node-1: [v1.0-pod-1]
Node-2: [v1.0-pod-2, v1.1-pod-3 (Starting)]

Step 2: Wait for v1.1-pod-3 to be Ready

Step 3: Terminate one v1.0 pod
Node-1: [v1.0-pod-1]
Node-2: [v1.1-pod-3]  # v1.0-pod-2 terminated

Step 4: Create another new pod
Node-1: [v1.0-pod-1, v1.1-pod-4 (Starting)]
Node-2: [v1.1-pod-3]

Step 5: Wait for v1.1-pod-4 to be Ready

Step 6: Terminate last v1.0 pod
Node-1: [v1.1-pod-4]
Node-2: [v1.1-pod-3]  # Update complete
```

**Result:** Zero downtime (always 2 pods serving traffic)

**Alternative 1: Recreate Strategy**

```yaml
strategy:
  type: Recreate  # Kill all old pods, then create new
```

**Process:**
```
Step 1: Terminate all v1.0 pods
  [Empty cluster]  ❌ DOWNTIME

Step 2: Create all v1.1 pods
  [v1.1-pod-1, v1.1-pod-2]  ✅ Back online
```

**Use Case:** Stateful apps that can't have multiple versions running

**Cons:**
- ❌ Downtime during update
- ❌ Not suitable for production web apps

**Alternative 2: Blue-Green Deployment**

**Deploy entire new stack, switch traffic:**
```yaml
# Blue deployment (v1.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-blue
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: backend
        version: v1.0

# Green deployment (v1.1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-green
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: backend
        version: v1.1

# Service points to blue initially
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
    version: v1.0  # Switch to v1.1 to cut over
```

**Process:**
1. Deploy green (v1.1) alongside blue (v1.0)
2. Test green in isolation
3. Switch service selector to green
4. Monitor for issues
5. Delete blue if successful, or rollback

**Pros:**
- ✅ Instant rollback (change selector back)
- ✅ Test new version before switching
- ✅ Zero downtime

**Cons:**
- ❌ 2x resources during deployment
- ❌ More complex
- ❌ Requires service mesh or manual selector change

**Alternative 3: Canary Deployment**

**Gradual rollout to small percentage:**
```yaml
# Main deployment (95% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-stable
spec:
  replicas: 19
  template:
    metadata:
      labels:
        app: backend
        track: stable

# Canary deployment (5% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-canary
spec:
  replicas: 1  # Only 1 pod = ~5% of traffic
  template:
    metadata:
      labels:
        app: backend
        track: canary

# Service load balances across both
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend  # Matches both stable and canary
```

**Process:**
1. Deploy canary (1 pod)
2. 5% of users get new version
3. Monitor metrics, errors
4. If good: scale canary to 100%, delete stable
5. If bad: delete canary

**Pros:**
- ✅ Limit blast radius (only 5% affected)
- ✅ Gradual confidence building
- ✅ Can test in production with real traffic

**Cons:**
- ⚠️ More complex
- ⚠️ Requires good monitoring
- ⚠️ Manual scaling steps

**Comparison:**

| Strategy | Downtime | Rollback Speed | Resource Usage | Complexity |
|----------|----------|----------------|----------------|------------|
| **Rolling Update** | ✅ Zero | ⚠️ Slow (reverse rolling) | ✅ Minimal | ✅ Simple |
| **Recreate** | ❌ Yes | ✅ Fast (redeploy) | ✅ Minimal | ✅ Simple |
| **Blue-Green** | ✅ Zero | ✅ Instant | ❌ 2x during deploy | ⚠️ Medium |
| **Canary** | ✅ Zero | ✅ Fast (delete canary) | ⚠️ Extra pods | ❌ Complex |

**Why We Chose Rolling Update:**
- ✅ Zero downtime (always 2 pods serving)
- ✅ Kubernetes-native (no extra tooling)
- ✅ Minimal resource usage
- ✅ Automatic rollback detection (rollout undo)
- ✅ Works with PDB and anti-affinity
- ✅ Good for our scale and maturity

**When to use alternatives:**
- **Recreate:** Stateful apps, dev environments
- **Blue-Green:** Need instant rollback, have spare capacity
- **Canary:** High-traffic production, mature DevOps

---

### 9.4 Health Checks

**What We Used:** Liveness and Readiness probes

**Requirements:**
- Application health endpoint
- HTTP server in container
- Proper HTTP status codes

**Setup:**
```yaml
# k8s/templates/chatbot-backend-deployment.yaml
spec:
  containers:
  - name: backend-container
    
    # Liveness Probe: Restart unhealthy pods
    livenessProbe:
      httpGet:
        path: /health
        port: 8000
      initialDelaySeconds: 30  # Wait for app to start
      periodSeconds: 10        # Check every 10 seconds
      failureThreshold: 3      # Restart after 3 failures
    
    # Readiness Probe: Control traffic routing
    readinessProbe:
      httpGet:
        path: /health
        port: 8000
      initialDelaySeconds: 5   # Faster than liveness
      periodSeconds: 5         # Check more frequently
      failureThreshold: 3      # Remove from service after 3 failures
```

**Application Health Endpoint (FastAPI):**
```python
# backend/main.py
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/health")
async def health_check():
    """
    Health check endpoint for Kubernetes probes.
    Returns 200 if healthy, 503 if not.
    """
    try:
        # Check database connectivity
        db_connection = await test_db_connection()
        
        # Check AWS Bedrock access
        bedrock_available = await test_bedrock_connection()
        
        if not db_connection or not bedrock_available:
            raise HTTPException(status_code=503, detail="Service unhealthy")
        
        return {"status": "healthy"}
    
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))
```

**Liveness vs Readiness:**

| Probe | Purpose | Failure Action | Use Case |
|-------|---------|----------------|----------|
| **Liveness** | Is app alive? | Restart container | Deadlock, memory leak, crash |
| **Readiness** | Can app serve traffic? | Remove from service | Startup, dependencies down, overloaded |

**Example Scenario:**

**Liveness Failure:**
```
App deadlocked, not responding
→ Liveness probe fails 3 times
→ Kubernetes restarts container
→ Fresh container starts
```

**Readiness Failure:**
```
App starting up, not ready yet
→ Readiness probe fails
→ Pod not added to service endpoints
→ No traffic sent to pod
→ Once ready, probe succeeds
→ Pod added to service, receives traffic
```

**Alternative 1: TCP Socket Probe**

**For apps without HTTP endpoint:**
```yaml
livenessProbe:
  tcpSocket:
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Checks:** Can connect to port (doesn't verify app health)

**Alternative 2: Exec Probe**

**Run command in container:**
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

**App writes file when healthy:**
```python
# App code
if is_healthy():
    with open('/tmp/healthy', 'w') as f:
        f.write('ok')
else:
    os.remove('/tmp/healthy')
```

**Alternative 3: No Probes**

**Kubernetes behavior without probes:**
- Liveness: Never restarts (even if crashed)
- Readiness: Assumes always ready (sends traffic immediately)

**Cons:**
- ❌ Pods stuck in bad state
- ❌ Traffic sent to unhealthy pods
- ❌ Manual intervention needed

**Alternative 4: Startup Probe (Kubernetes 1.16+)**

**For slow-starting applications:**
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 60  # 60 * 5 = 300 seconds (5 minutes) to start

livenessProbe:
  httpGet:
    path: /health
    port: 8000
  periodSeconds: 10
  # No initialDelaySeconds (startup probe protects)
```

**Benefit:** Liveness probe starts only after startup succeeds

**Comparison:**

| Probe Type | Best For | Overhead | Accuracy |
|------------|----------|----------|----------|
| **HTTP** | Web apps, REST APIs | ⚠️ Medium | ✅ High (checks actual logic) |
| **TCP** | TCP servers, databases | ✅ Low | ⚠️ Low (only checks port) |
| **Exec** | Any app | ❌ High (exec per check) | ✅ High (custom logic) |
| **Startup** | Slow-starting apps | ✅ Low | ✅ High |
| **None** | ❌ Never | ✅ None | ❌ None |

**Why We Chose HTTP Probes:**
- ✅ FastAPI has `/health` endpoint
- ✅ Verifies application logic (not just port open)
- ✅ Can check dependencies (DB, Bedrock)
- ✅ Standard practice for web apps
- ✅ Good balance of accuracy and overhead

**Tuning Recommendations:**

**Fast-Starting Apps (our case):**
```yaml
readinessProbe:
  initialDelaySeconds: 5   # App ready quickly
  periodSeconds: 5         # Check frequently

livenessProbe:
  initialDelaySeconds: 30  # Give app time to start
  periodSeconds: 10        # Check less frequently
```

**Slow-Starting Apps:**
```yaml
startupProbe:
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes to start

readinessProbe:
  initialDelaySeconds: 0  # Startup probe protects
  periodSeconds: 5

livenessProbe:
  initialDelaySeconds: 0  # Startup probe protects
  periodSeconds: 10
```

---

## 10. Application Architecture

### 10.1 AWS Bedrock vs Self-Hosted LLM

**What We Used:** AWS Bedrock with DeepSeek V3.1

**Requirements:**
- AWS account with Bedrock access
- Model access request (one-time)
- IAM permissions for `bedrock:InvokeModel`
- Boto3 SDK

**Setup:**
```python
# backend/main.py
import boto3
import json

# Initialize Bedrock client
bedrock_runtime = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-2'
)

# Invoke model
def chat_with_ai(user_message: str) -> str:
    response = bedrock_runtime.invoke_model(
        modelId='deepseek.v3-v1:0',  # DeepSeek V3.1 model
        contentType='application/json',
        accept='application/json',
        body=json.dumps({
            "messages": [
                {"role": "user", "content": user_message}
            ],
            "max_tokens": 2048,
            "temperature": 0.7
        })
    )
    
    result = json.loads(response['body'].read())
    return result['content'][0]['text']
```

**IAM Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "arn:aws:bedrock:us-east-2::foundation-model/deepseek.v3-v1:0"
  }]
}
```

**Cost:** $0.27 per 1M tokens (pay-per-use)

**Alternative 1: Self-Hosted LLM (GPU Instance)**

**Infrastructure:**
```terraform
# EC2 with GPU
resource "aws_instance" "llm_server" {
  ami           = "ami-gpu-optimized"
  instance_type = "g5.xlarge"  # 1x NVIDIA A10G GPU
  
  user_data = <<-EOF
    #!/bin/bash
    # Install CUDA, PyTorch, etc.
    # Download model weights (7-70GB)
    # Start inference server
  EOF
}
```

**Cost:**
- **g5.xlarge:** ~$1.00/hour = $720/month (24/7)
- **g5.2xlarge:** ~$1.50/hour = $1,080/month (more capacity)
- Plus: Storage for model weights (~100GB)

**Setup:**
```python
# Use local inference server
import requests

response = requests.post(
    'http://llm-server:8000/v1/completions',
    json={
        'prompt': user_message,
        'max_tokens': 2048
    }
)
```

**Pros:**
- ✅ Full control over model
- ✅ Predictable latency
- ✅ No API rate limits
- ✅ Can fine-tune model

**Cons:**
- ❌ High fixed cost ($720+/month even with no usage)
- ❌ GPU management complexity
- ❌ Model updates manual
- ❌ Scaling requires multiple instances
- ❌ Long cold start (load model to GPU)

**Alternative 2: SageMaker Endpoint**

**AWS managed inference:**
```terraform
resource "aws_sagemaker_model" "llm" {
  name               = "deepseek-model"
  execution_role_arn = aws_iam_role.sagemaker.arn
  
  primary_container {
    image          = "763104351884.dkr.ecr.us-east-2.amazonaws.com/pytorch-inference:2.0.0-gpu-py310"
    model_data_url = "s3://my-bucket/model.tar.gz"
  }
}

resource "aws_sagemaker_endpoint" "llm" {
  name                 = "deepseek-endpoint"
  endpoint_config_name = aws_sagemaker_endpoint_configuration.llm.name
}
```

**Cost:**
- **ml.g5.xlarge:** ~$1.15/hour = $828/month
- Slightly more than raw EC2 but managed

**Pros:**
- ✅ Managed infrastructure
- ✅ Auto-scaling
- ✅ Built-in monitoring

**Cons:**
- ❌ Still high fixed cost
- ❌ More expensive than EC2
- ❌ Complex setup

**Alternative 3: OpenAI API**

**Hosted LLM service:**
```python
import openai

openai.api_key = os.getenv('OPENAI_API_KEY')

response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[
        {"role": "user", "content": user_message}
    ]
)
```

**Cost:**
- **GPT-4:** $30 per 1M input tokens, $60 per 1M output tokens
- **GPT-3.5:** $0.50 per 1M input tokens, $1.50 per 1M output tokens

**Pros:**
- ✅ State-of-the-art models
- ✅ Simple API
- ✅ Pay-per-use

**Cons:**
- ❌ More expensive than Bedrock
- ❌ Data sent to OpenAI (privacy concerns)
- ❌ Rate limits
- ❌ Vendor lock-in

**Comparison:**

| Option | Cost (1M tokens) | Fixed Cost/Month | Latency | Control | Privacy |
|--------|-----------------|------------------|---------|---------|---------|
| **Bedrock** | $0.27 | ✅ $0 | ✅ Fast | ⚠️ Medium | ✅ AWS-only |
| **Self-Hosted** | ✅ $0 | ❌ $720+ | ✅ Fast | ✅ Full | ✅ Full |
| **SageMaker** | ✅ $0 | ❌ $828+ | ✅ Fast | ✅ High | ✅ AWS-only |
| **OpenAI GPT-4** | $45 avg | ✅ $0 | ⚠️ Variable | ❌ None | ❌ External |
| **OpenAI GPT-3.5** | $1.00 | ✅ $0 | ✅ Fast | ❌ None | ❌ External |

**Cost Example (10M tokens/month):**
- **Bedrock:** $2.70
- **Self-Hosted:** $720 (fixed)
- **SageMaker:** $828 (fixed)
- **OpenAI GPT-4:** $450
- **OpenAI GPT-3.5:** $10

**Why We Chose Bedrock:**
- ✅ **Pay-per-use:** $0 when not using
- ✅ **Cost-effective:** $0.27/1M tokens (DeepSeek)
- ✅ **Serverless:** No infrastructure to manage
- ✅ **Scalable:** Handles burst traffic
- ✅ **AWS integration:** Works with Pod Identity
- ✅ **Privacy:** Data stays in AWS
- ✅ **No GPU management:** AWS handles infrastructure

**When to use alternatives:**
- **Self-Hosted:** >1B tokens/month, need custom model
- **SageMaker:** Need auto-scaling, custom inference logic
- **OpenAI:** Need GPT-4 quality, low volume

---

### 10.2 FastAPI vs Flask vs Django

**What We Used:** FastAPI for backend

**Requirements:**
- Python 3.7+
- ASGI server (Uvicorn)
- Pydantic for data validation

**Setup:**
```python
# backend/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import uvicorn

app = FastAPI()

# Request model with validation
class ChatRequest(BaseModel):
    message: str
    session_id: str

# Response model
class ChatResponse(BaseModel):
    response: str
    conversation_id: int

@app.post("/chat", response_model=ChatResponse)
async def chat_endpoint(request: ChatRequest):
    """
    Chat with AI endpoint.
    Validates input with Pydantic, returns structured response.
    """
    if not request.message:
        raise HTTPException(status_code=400, detail="Message cannot be empty")
    
    # Call AI model
    ai_response = await chat_with_ai(request.message)
    
    # Store in database
    conversation_id = await store_conversation(
        session_id=request.session_id,
        user_message=request.message,
        bot_response=ai_response
    )
    
    return ChatResponse(
        response=ai_response,
        conversation_id=conversation_id
    )

@app.get("/health")
async def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Alternative 1: Flask**

**Synchronous micro-framework:**
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/chat', methods=['POST'])
def chat_endpoint():
    data = request.get_json()
    
    # Manual validation
    if 'message' not in data:
        return jsonify({'error': 'Message required'}), 400
    
    # Synchronous call (blocks thread)
    ai_response = chat_with_ai(data['message'])
    
    return jsonify({'response': ai_response})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

**Alternative 2: Django + Django REST Framework**

**Full-featured framework:**
```python
# myapp/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import serializers

class ChatSerializer(serializers.Serializer):
    message = serializers.CharField()
    session_id = serializers.CharField()

class ChatView(APIView):
    def post(self, request):
        serializer = ChatSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=400)
        
        ai_response = chat_with_ai(serializer.validated_data['message'])
        return Response({'response': ai_response})
```

**Comparison:**

| Feature | FastAPI | Flask | Django |
|---------|---------|-------|--------|
| **Performance** | ✅ Async, fast | ⚠️ Sync, slower | ⚠️ Sync, slower |
| **Auto validation** | ✅ Pydantic | ❌ Manual | ⚠️ DRF serializers |
| **Auto docs** | ✅ Swagger/OpenAPI | ❌ None | ⚠️ DRF browsable API |
| **Type hints** | ✅ Native | ❌ Not used | ⚠️ Limited |
| **Learning curve** | ✅ Easy | ✅ Easy | ❌ Steep |
| **Batteries included** | ⚠️ Minimal | ⚠️ Minimal | ✅ ORM, admin, auth |
| **Modern Python** | ✅ Async/await | ❌ Sync only | ⚠️ Async support added |

**Why We Chose FastAPI:**
- ✅ **Async support:** Non-blocking AI and DB calls
- ✅ **Automatic validation:** Pydantic models
- ✅ **Auto documentation:** Swagger UI at `/docs`
- ✅ **Type safety:** Type hints enforced
- ✅ **Modern:** Uses latest Python features
- ✅ **Fast:** ASGI vs WSGI
- ✅ **Easy to learn:** Similar to Flask but better

**Performance Example:**
```python
# FastAPI: Multiple concurrent requests
async def handle_requests():
    tasks = [
        chat_with_ai("Question 1"),
        chat_with_ai("Question 2"),
        chat_with_ai("Question 3")
    ]
    responses = await asyncio.gather(*tasks)  # Parallel execution

# Flask: Sequential
def handle_requests():
    responses = [
        chat_with_ai("Question 1"),  # Waits
        chat_with_ai("Question 2"),  # Waits
        chat_with_ai("Question 3")   # Waits
    ]
```

**When to use alternatives:**
- **Flask:** Simple APIs, team familiar with Flask, no async needed
- **Django:** Need ORM, admin panel, authentication, full CMS

---

## 11. Monitoring & Observability (Planned)

### 11.1 Prometheus + Grafana vs CloudWatch

**What We're Planning:** Prometheus + Grafana stack

**Requirements:**
- Helm chart: kube-prometheus-stack
- Persistent storage for Prometheus (EBS)
- Ingress or port forwarding for access

**Setup:**
```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (includes Prometheus + Grafana + Alertmanager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

**What It Includes:**
- **Prometheus:** Metrics collection and storage
- **Grafana:** Visualization and dashboards
- **Alertmanager:** Alert routing and notifications
- **Node Exporter:** Node-level metrics
- **kube-state-metrics:** Kubernetes object metrics

**Alternative 1: CloudWatch Container Insights**

**AWS-native monitoring:**
```terraform
# Enable Container Insights
resource "aws_eks_cluster" "this" {
  name = "platform-dev"
  
  # CloudWatch logging enabled
  enabled_cluster_log_types = ["api", "audit", "authenticator"]
}

# Install CloudWatch agent (DaemonSet)
resource "helm_release" "cloudwatch_agent" {
  name       = "aws-cloudwatch-metrics"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-cloudwatch-metrics"
  namespace  = "amazon-cloudwatch"
}
```

**Cost:** ~$10-30/month for logs + metrics

**Alternative 2: Datadog**

**SaaS monitoring platform:**
```bash
# Install Datadog agent
helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=$DD_API_KEY \
  --set datadog.site=datadoghq.com
```

**Cost:** $15/host/month + $0.10/million custom metrics

**Alternative 3: New Relic**

**APM + Infrastructure monitoring:**

**Cost:** $99/user/month

**Comparison:**

| Feature | Prometheus + Grafana | CloudWatch | Datadog | New Relic |
|---------|---------------------|------------|---------|-----------|
| **Cost** | ✅ Free (self-hosted) | ⚠️ $10-30/mo | ❌ $180/mo | ❌ $1,188/year |
| **Setup** | ⚠️ Medium | ✅ Easy | ✅ Easy | ✅ Easy |
| **Customization** | ✅ Full | ⚠️ Limited | ✅ High | ✅ High |
| **Query language** | ✅ PromQL | ⚠️ CloudWatch Insights | ✅ DQL | ✅ NRQL |
| **Alerting** | ✅ Alertmanager | ✅ CloudWatch Alarms | ✅ Built-in | ✅ Built-in |
| **Dashboards** | ✅ Grafana | ⚠️ Basic | ✅ Advanced | ✅ Advanced |
| **Retention** | ✅ Configurable | ⚠️ 15 months max | ✅ 15 months+ | ✅ Custom |

**Why We're Choosing Prometheus + Grafana:**
- ✅ **Cost:** Free (only storage costs)
- ✅ **Kubernetes-native:** Designed for Kubernetes
- ✅ **PromQL:** Powerful query language
- ✅ **Community:** Large ecosystem of exporters
- ✅ **No vendor lock-in:** Open source
- ✅ **Custom dashboards:** Full control

**When to use alternatives:**
- **CloudWatch:** Already using AWS, simple needs, low volume
- **Datadog:** Need APM, distributed tracing, team collaboration
- **New Relic:** Enterprise support, compliance requirements

---

### 11.2 Horizontal Pod Autoscaler (HPA)

**What We're Planning:** CPU-based HPA

**Requirements:**
- Metrics Server installed (default in EKS)
- Resource requests configured
- HPA resource definition

**Setup:**
```yaml
# k8s/hpa/backend-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chatbot-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chatbot-backend-deployment
  
  minReplicas: 2   # Never scale below 2
  maxReplicas: 10  # Never scale above 10
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 50              # Scale down max 50% of pods at once
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
      - type: Percent
        value: 100             # Scale up max 100% (double) at once
        periodSeconds: 15
```

**How It Works:**
```
Current: 2 pods @ 40% CPU → No scaling
Traffic increases → CPU hits 75% → Scale to 3 pods
More traffic → CPU hits 80% → Scale to 4 pods
Traffic decreases → CPU drops to 30% → Wait 5 minutes → Scale down to 3 pods
```

**Alternative 1: Memory-Based HPA**

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80  # Scale when memory > 80%
```

**Alternative 2: Custom Metrics (Request Count)**

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"  # Scale when > 1000 req/sec per pod
```

**Requires:** Prometheus adapter or custom metrics API

**Alternative 3: External Metrics (SQS Queue Length)**

```yaml
metrics:
- type: External
  external:
    metric:
      name: sqs_queue_length
    target:
      type: AverageValue
      averageValue: "30"  # Scale when > 30 messages per pod
```

**Alternative 4: Vertical Pod Autoscaler (VPA)**

**Adjust resource requests/limits instead of replicas:**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chatbot-backend-deployment
  updatePolicy:
    updateMode: "Auto"  # Automatically restart pods with new resources
```

**VPA adjusts:**
```yaml
# Before
resources:
  requests:
    cpu: 100m
    memory: 128Mi

# After (VPA detected need for more resources)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

**Comparison:**

| Type | Scales | Best For | Complexity |
|------|--------|----------|------------|
| **HPA - CPU** | Replicas | CPU-bound apps | ✅ Simple |
| **HPA - Memory** | Replicas | Memory-bound apps | ✅ Simple |
| **HPA - Custom** | Replicas | Request-based apps | ⚠️ Medium |
| **VPA** | Resources | Right-sizing | ⚠️ Medium |
| **Cluster Autoscaler** | Nodes | Node capacity | ⚠️ Medium |

**Why We're Choosing CPU-Based HPA:**
- ✅ **Simple:** Works out of box with Metrics Server
- ✅ **Effective:** AI inference is CPU-bound
- ✅ **Predictable:** CPU correlates with load
- ✅ **Quick response:** Scales up in 15 seconds
- ✅ **Stable:** 5-minute stabilization prevents flapping

**Future Enhancement:** Add custom metrics (requests/sec) after Prometheus

---

## 12. Image Management

### 12.1 ECR vs Docker Hub

**What We Used:** Amazon ECR (Elastic Container Registry)

**Requirements:**
- AWS account
- IAM permissions for ECR
- ECR repository created

**Setup:**
```terraform
# Create ECR repository
resource "aws_ecr_repository" "platform_app" {
  name = "platform-app"
  
  image_scanning_configuration {
    scan_on_push = true  # Automatic vulnerability scanning
  }
  
  image_tag_mutability = "MUTABLE"  # Allow tag overwriting (dev)
  
  encryption_configuration {
    encryption_type = "AES256"  # Encrypt images at rest
  }
}

# Lifecycle policy (cleanup old images)
resource "aws_ecr_lifecycle_policy" "platform_app" {
  repository = aws_ecr_repository.platform_app.name
  
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 30 images"
      selection = {
        tagStatus     = "any"
        countType     = "imageCountMoreThan"
        countNumber   = 30
      }
      action = {
        type = "expire"
      }
    }]
  })
}
```

**Authentication:**
```bash
# Get ECR login token
aws ecr get-login-password --region us-east-2 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-2.amazonaws.com

# Build and push
docker build -t 123456789012.dkr.ecr.us-east-2.amazonaws.com/platform-app:backend-v1.0.0 .
docker push 123456789012.dkr.ecr.us-east-2.amazonaws.com/platform-app:backend-v1.0.0
```

**EKS pulls from ECR automatically (no ImagePullSecrets needed):**
```yaml
spec:
  containers:
  - name: backend
    image: 123456789012.dkr.ecr.us-east-2.amazonaws.com/platform-app:backend-v1.0.0
    # No imagePullSecrets required - EKS node IAM role has ECR access
```

**Alternative 1: Docker Hub**

**Public registry:**
```bash
# Push to Docker Hub
docker login -u myusername
docker tag backend:v1.0.0 myusername/platform-backend:v1.0.0
docker push myusername/platform-backend:v1.0.0
```

**Free tier:** 
- Public repos: Unlimited
- Private repos: 1 free
- Pull rate limit: 200 pulls / 6 hours (anonymous), 5000 / day (authenticated)

**Paid:** $5/month for unlimited private repos

**Kubernetes:**
```yaml
# Need ImagePullSecret for private repos
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>

---
spec:
  imagePullSecrets:
  - name: dockerhub-secret
  containers:
  - name: backend
    image: myusername/platform-backend:v1.0.0
```

**Alternative 2: GitHub Container Registry**

**Free with GitHub:**
```bash
# Login
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push
docker tag backend:v1.0.0 ghcr.io/username/platform-backend:v1.0.0
docker push ghcr.io/username/platform-backend:v1.0.0
```

**Free tier:** Unlimited public repos, 500MB free for private

**Alternative 3: Self-Hosted Registry**

**Harbor, GitLab Registry, etc.:**
```bash
# Run registry
docker run -d -p 5000:5000 --name registry registry:2

# Push
docker tag backend:v1.0.0 localhost:5000/backend:v1.0.0
docker push localhost:5000/backend:v1.0.0
```

**Cons:**
- ❌ Must manage registry infrastructure
- ❌ Backup and HA needed
- ❌ Security scanning not included

**Comparison:**

| Registry | Cost | Rate Limits | Scanning | AWS Integration |
|----------|------|-------------|----------|----------------|
| **ECR** | ✅ $0.10/GB/mo | ✅ None | ✅ Built-in | ✅ Native |
| **Docker Hub** | ⚠️ $5/mo or limits | ❌ 200-5000/day | ⚠️ Paid only | ❌ ImagePullSecret |
| **GitHub** | ✅ Free (500MB) | ✅ Generous | ⚠️ Basic | ❌ ImagePullSecret |
| **Self-Hosted** | ⚠️ Infrastructure | ✅ None | ❌ Manual | ❌ ImagePullSecret |

**Why We Chose ECR:**
- ✅ **No rate limits:** Unlimited pulls
- ✅ **AWS integration:** EKS pulls without credentials
- ✅ **Security scanning:** Automatic vulnerability detection
- ✅ **Encryption:** Images encrypted at rest
- ✅ **Lifecycle policies:** Auto-cleanup old images
- ✅ **IAM-based:** No registry credentials to manage
- ✅ **Low cost:** ~$1-2/month for our usage

**Cost breakdown:**
- Storage: $0.10/GB/month (~5GB = $0.50)
- Data transfer: Free within same region
- Total: ~$0.50-2/month

---

### 12.2 Image Tagging Strategy

**What We're Planning:** Semantic Version + Git SHA + Build Number

**Format:** `v1.0.0-a3f2c1b-142`

**Jenkins Pipeline:**
```groovy
pipeline {
    agent any
    
    environment {
        ECR_REPO = "123456789012.dkr.ecr.us-east-2.amazonaws.com/platform-app"
        SEMANTIC_VERSION = "v1.0.0"  // Updated manually for major releases
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    // Get Git commit SHA
                    def GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    
                    // Build tag: v1.0.0-a3f2c1b-142
                    def IMAGE_TAG = "${SEMANTIC_VERSION}-${GIT_COMMIT_SHORT}-${BUILD_NUMBER}"
                    
                    env.BACKEND_IMAGE = "${ECR_REPO}:backend-${IMAGE_TAG}"
                    env.FRONTEND_IMAGE = "${ECR_REPO}:frontend-${IMAGE_TAG}"
                    
                    // Build images
                    sh "docker build -t ${BACKEND_IMAGE} ./backend"
                    sh "docker build -t ${FRONTEND_IMAGE} ./frontend"
                }
            }
        }
        
        stage('Push') {
            steps {
                sh """
                    aws ecr get-login-password --region us-east-2 | \
                      docker login --username AWS --password-stdin ${ECR_REPO}
                    
                    docker push ${BACKEND_IMAGE}
                    docker push ${FRONTEND_IMAGE}
                """
            }
        }
        
        stage('Deploy') {
            steps {
                sh """
                    helm upgrade --install chatbot ./k8s \
                      -f k8s/values-dev.yaml \
                      --set backend.image=${BACKEND_IMAGE} \
                      --set frontend.image=${FRONTEND_IMAGE}
                """
            }
        }
    }
}
```

**Alternative 1: latest Tag**

```bash
docker build -t backend:latest .
docker push myrepo/backend:latest
```

**Cons:**
- ❌ No version tracking
- ❌ Can't rollback (which was "latest"?)
- ❌ Cache issues (Kubernetes may not pull new "latest")
- ❌ No traceability to code
- ❌ **Never use in production**

**Alternative 2: Git SHA Only**

```bash
docker build -t backend:a3f2c1b .
```

**Pros:**
- ✅ Traceable to exact code
- ✅ Unique per commit

**Cons:**
- ⚠️ No human-readable version
- ⚠️ Hard to track releases

**Alternative 3: Build Number Only**

```bash
docker build -t backend:142 .
```

**Cons:**
- ⚠️ No Git traceability (need Jenkins to look up)
- ⚠️ Build numbers reset if Jenkins replaced

**Alternative 4: Date-Based**

```bash
docker build -t backend:2025-12-03-1430 .
```

**Cons:**
- ⚠️ No Git traceability
- ⚠️ No semantic meaning

**Comparison:**

| Strategy | Traceability | Readability | Rollback | Production-Ready |
|----------|-------------|-------------|----------|-----------------|
| **Semver+SHA+Build** | ✅ Excellent | ✅ Good | ✅ Easy | ✅ Yes |
| **latest** | ❌ None | ✅ Simple | ❌ Impossible | ❌ No |
| **Git SHA** | ✅ Perfect | ⚠️ Technical | ✅ Yes | ✅ Yes |
| **Build Number** | ⚠️ Medium | ✅ Good | ✅ Yes | ⚠️ OK |
| **Date** | ❌ Poor | ✅ Good | ⚠️ Hard | ⚠️ OK |

**Why We're Choosing Semver + SHA + Build:**
- ✅ **Semantic version:** Human-readable (v1.0.0 = stable)
- ✅ **Git SHA:** Exact code traceability
- ✅ **Build number:** Unique identifier
- ✅ **Rollback:** Easy to identify previous versions
- ✅ **Audit:** Know exactly what's deployed
- ✅ **GitOps-friendly:** Can track in Git

**Example Versioning:**
```
Development:
- backend-v1.0.0-a3f2c1b-142
- backend-v1.0.0-b4e3d2c-143
- backend-v1.0.0-c5f4e3d-144

Major Release:
- backend-v1.1.0-d6g5f4e-145  ← New features

Bug Fix:
- backend-v1.1.1-e7h6g5f-146  ← Patch

Breaking Change:
- backend-v2.0.0-f8i7h6g-147  ← Major version bump
```

---

## 13. Cost Optimization

### 13.1 Environment-Specific Sizing

**What We Used:** Different instance types and configurations per environment

**Dev Environment Strategy:**
```terraform
# Development - Cost-Optimized
module "eks" {
  instance_types    = ["t3.small"]  # $0.0208/hour = $15/month per node
  node_desired_size = 2
  node_min_size     = 1
  node_max_size     = 3
  disk_size         = 20  # GB
}

module "rds" {
  instance_class          = "db.t3.micro"  # $0.017/hour = $12/month
  allocated_storage       = 20             # GB
  multi_az                = false          # Single-AZ
  backup_retention_period = 3              # Days
  deletion_protection     = false          # Easy to destroy
}

module "network" {
  nat_gateway_count = 1  # $0.045/hour = $32/month
}

# Total Dev: ~$177/month
```

**Prod Environment Strategy:**
```terraform
# Production - High Availability
module "eks" {
  instance_types    = ["t3.medium"]  # $0.0416/hour = $30/month per node
  node_desired_size = 3              # More capacity
  node_min_size     = 2
  node_max_size     = 5
  disk_size         = 30  # GB
}

module "rds" {
  instance_class          = "db.t3.small"   # $0.034/hour = $25/month
  allocated_storage       = 50              # GB
  multi_az                = true            # 2x cost for standby
  backup_retention_period = 7               # Days
  deletion_protection     = true
}

module "network" {
  nat_gateway_count = 2  # $0.090/hour = $64/month (2 NATs)
}

# Total Prod: ~$294/month
```

**Alternative 1: Same Sizing for All Environments**

```terraform
# Both dev and prod use same config
instance_types = ["t3.medium"]
multi_az       = true
nat_gateway_count = 2

# Dev cost: $294/month (same as prod)
# Waste: $117/month on dev
```

**Cons:**
- ❌ Overpaying for dev
- ❌ Dev doesn't need HA
- ❌ Unused capacity

**Alternative 2: Spot Instances (Dev Only)**

```terraform
# Use spot instances for dev
resource "aws_eks_node_group" "spot" {
  capacity_type = "SPOT"  # Up to 90% cheaper
  instance_types = ["t3.small", "t3.medium"]
}
```

**Pros:**
- ✅ 70-90% cost savings
- ✅ Good for dev/test

**Cons:**
- ❌ Can be interrupted (2-minute warning)
- ❌ Not suitable for production
- ⚠️ Requires workload tolerance

**Alternative 3: Fargate (Serverless)**

```terraform
# No EC2 nodes, pay per pod
resource "aws_eks_fargate_profile" "apps" {
  cluster_name = aws_eks_cluster.this.name
  
  selector {
    namespace = "default"
  }
}
```

**Cost:** $0.04048/vCPU/hour + $0.004445/GB/hour

**Pros:**
- ✅ No node management
- ✅ Auto-scaling
- ✅ Pay per pod

**Cons:**
- ❌ More expensive than EC2 at scale
- ❌ Slower cold starts
- ❌ Limited to specific workloads

**Comparison (100 hours/month pod runtime):**

| Instance Type | vCPU | Memory | Cost/Hour | Monthly Cost | Use Case |
|--------------|------|--------|-----------|--------------|----------|
| **t3.micro** | 2 | 1GB | $0.0104 | $7.50 | Dev RDS |
| **t3.small** | 2 | 2GB | $0.0208 | $15 | Dev EKS |
| **t3.medium** | 2 | 4GB | $0.0416 | $30 | Prod EKS |
| **Fargate (2vCPU, 4GB)** | 2 | 4GB | $0.099 | $71 | Serverless |
| **Spot t3.small** | 2 | 2GB | ~$0.006 | ~$4.50 | Dev tolerant |

**Why We Chose Environment-Specific Sizing:**
- ✅ **Cost-effective:** Save $117/month on dev
- ✅ **Right-sized:** Each environment gets what it needs
- ✅ **Acceptable tradeoffs:** Dev can tolerate Single-AZ
- ✅ **Production safety:** Prod has HA, backups, capacity

**Monthly Cost Breakdown:**

**Development ($177/month):**
- EKS: 2x t3.small nodes = $30
- RDS: db.t3.micro Single-AZ = $12
- NAT Gateway: 1x = $32
- EBS: 40GB = $4
- ELB: $18
- Data Transfer: ~$10
- Misc (CloudWatch, etc.): ~$5
- **Buffer: ~$66**

**Production ($294/month):**
- EKS: 3x t3.medium nodes = $90
- RDS: db.t3.small Multi-AZ = $50
- NAT Gateway: 2x = $64
- EBS: 90GB = $9
- ELB: $18
- Data Transfer: ~$15
- Misc: ~$10
- **Buffer: ~$38**

---

### 13.2 Resource Requests and Limits

**What We Used:** Conservative requests, higher limits

**Dev Configuration:**
```yaml
# k8s/values-dev.yaml
backend:
  resources:
    requests:
      memory: "128Mi"  # Guaranteed minimum
      cpu: "100m"      # 0.1 CPU core
    limits:
      memory: "256Mi"  # Can burst up to 256Mi
      cpu: "200m"      # Can burst to 0.2 CPU

frontend:
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"
```

**Prod Configuration:**
```yaml
# k8s/values-prod.yaml
backend:
  resources:
    requests:
      memory: "256Mi"  # Higher baseline
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "500m"

frontend:
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

**Node Capacity Planning:**

**Dev (t3.small: 2 vCPU, 2GB RAM):**
```
Available: ~1.7 vCPU, ~1.4GB RAM (after system pods)

Backend (2 replicas):
  Requests: 2 × 100m = 200m CPU, 2 × 128Mi = 256Mi RAM

Frontend (2 replicas):
  Requests: 2 × 100m = 200m CPU, 2 × 128Mi = 256Mi RAM

System pods: ~300m CPU, ~500Mi RAM

Total: ~700m CPU, ~1012Mi RAM
Utilization: ~41% CPU, ~70% RAM
✅ Fits on 2 nodes with room for bursting
```

**Alternative 1: No Limits (Dangerous)**

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  # No limits - can use entire node
```

**Cons:**
- ❌ Pod can consume all node resources
- ❌ Noisy neighbor problems
- ❌ Node instability
- ❌ OOMKiller may kill random pods

**Alternative 2: Limits = Requests (Guaranteed QoS)**

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "256Mi"  # Same as request
    cpu: "200m"
```

**Pros:**
- ✅ Guaranteed resources (QoS: Guaranteed)
- ✅ Predictable performance
- ✅ Last to be evicted under pressure

**Cons:**
- ❌ Wastes resources (can't burst)
- ❌ More expensive (need more nodes)

**Alternative 3: No Requests or Limits (Anti-Pattern)**

```yaml
resources: {}  # Nothing specified
```

**Cons:**
- ❌ Scheduler doesn't know pod requirements
- ❌ Can overcommit nodes
- ❌ No QoS guarantees
- ❌ **Never do this**

**QoS Classes:**

| Config | QoS Class | Eviction Priority | Best For |
|--------|-----------|------------------|----------|
| **Limits > Requests** | Burstable | Medium | Most apps (our choice) |
| **Limits = Requests** | Guaranteed | Low (protected) | Critical apps |
| **No Requests** | BestEfffort | High (first evicted) | Batch jobs |

**Why We Chose Burstable QoS:**
- ✅ **Efficient:** Pods can burst when needed
- ✅ **Cost-effective:** Don't overprovision
- ✅ **Realistic:** Most apps have variable load
- ✅ **Node utilization:** Better bin packing

**How We Determined Values:**

1. **Run load tests:**
```bash
# Stress test application
ab -n 10000 -c 100 http://backend:8000/chat
```

2. **Monitor resource usage:**
```bash
kubectl top pods
```

3. **Set requests = typical usage**
4. **Set limits = 2x requests** (burst capacity)

**Example metrics:**
```
Backend steady state: 80m CPU, 100Mi RAM
Backend under load: 150m CPU, 180Mi RAM

Requests: 100m CPU, 128Mi RAM (slightly above steady)
Limits: 200m CPU, 256Mi RAM (above peak)
```

---

## Summary: Key Takeaways

**Modern AWS & Kubernetes Choices:**
1. **EKS Pod Identity** over OIDC - Simpler setup, better debugging
2. **Access Entry API** over aws-auth - Modern, Terraform-managed
3. **EBS CSI with Pod Identity** - Persistent storage without OIDC complexity
4. **Init containers for secrets** - Better visibility than CSI driver
5. **S3 native locking** - Free vs DynamoDB cost
6. **Packer + Ansible** - Immutable AMIs over user data
7. **SSM Session Manager** - No SSH keys or bastion hosts
8. **Non-root containers + Drop ALL capabilities** - Maximum security

**Cost-Effective Decisions:**
9. **NAT Gateway: 1 for dev, 2 for prod** - Balance cost and HA
10. **Multi-AZ RDS only in prod** - Dev doesn't need HA
11. **t3 instances** - Right-sized for workload
12. **AWS Bedrock** - Pay-per-use vs $720/month GPU
13. **ECR** - No rate limits, free within AWS

**High Availability:**
14. **PDB** - Prevents zero-pod scenarios during maintenance
15. **Pod Anti-Affinity** - Spread replicas across nodes
16. **Rolling Updates** - Zero downtime deployments
17. **Health Probes** - Automatic restart and traffic control

**Observability (Planned):**
18. **Prometheus + Grafana** - Free, powerful, Kubernetes-native
19. **HPA** - Auto-scale based on CPU/memory
20. **Custom metrics** - Request-based scaling (future)

**Every decision documented with alternatives and rationale for interview prep!** 🚀

