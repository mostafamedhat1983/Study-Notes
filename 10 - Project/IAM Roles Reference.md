---
tags:
  - AWS
---
# IAM Roles Reference - terraform-aws-eks Project

  

## Overview

  

This document provides a complete reference of all IAM roles, policies, and permissions used in the `terraform-aws-eks` infrastructure project. Understanding these roles is critical for security, troubleshooting, and interview preparation.

  

---

  

## Role Summary Table

  

| Role Name | Used By | Authentication Method | Managed In | Status |

|-----------|---------|----------------------|------------|--------|

| **Jenkins EC2 Role** | Jenkins EC2 instance | EC2 Instance Profile | `dev/main.tf`, `prod/main.tf` | ✅ Active |

| **EKS Cluster Role** | EKS Control Plane | Service-linked role | `dev/main.tf`, `prod/main.tf` | ✅ Active |

| **EKS Node Role** | Worker Nodes (EC2) | EC2 Instance Profile | `dev/main.tf`, `prod/main.tf` | ✅ Active |

| **EBS CSI Driver Role** | EBS CSI Controller pods | Pod Identity | `modules/eks/main.tf` | ✅ Active |

| **Chatbot Backend Role** | Chatbot backend pods | Pod Identity | `modules/eks/main.tf` | ✅ Active |

| **AWS Load Balancer Controller Role** | LB Controller pods | Pod Identity | `modules/eks/main.tf` | ❌ To be added |

  

---

  

## Role Details

  

### 1. Jenkins EC2 Role

  

**Name:** `ec2-ssm-ecr-role-{env}` (e.g., `ec2-ssm-ecr-role-dev`)

  

**Used By:**

- Jenkins Controller EC2 instance

  

**Authentication:**

- EC2 Instance Profile (attached at instance launch)

- No static credentials needed

  

**Permissions (Managed Policies):**

1. **AmazonSSMManagedInstanceCore** (AWS Managed)

   - Enables AWS Systems Manager Session Manager

   - Allows secure shell access without SSH keys

   - Actions: `ssm:UpdateInstanceInformation`, `ssmmessages:*`, `ec2messages:*`

  

2. **AmazonEC2ContainerRegistryPowerUser** (AWS Managed)

   - Push/pull Docker images to/from ECR

   - Actions: `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, etc.

  

**Custom Policies:**

  

#### jenkins-eks-access-{env}

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Action": ["eks:DescribeCluster"],

    "Resource": "arn:aws:eks:us-east-2:*:cluster/platform-{env}"

  }]

}

```

**Why needed:** Jenkins needs to get cluster endpoint and CA certificate for kubectl configuration.

  

#### jenkins-bedrock-access-{env}

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Action": ["bedrock:InvokeModel"],

    "Resource": "arn:aws:bedrock:us-east-2::foundation-model/deepseek.v3-v1:0"

  }]

}

```

**Why needed:** Jenkins may test chatbot functionality during CI/CD pipelines.

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "ec2.amazonaws.com"

    },

    "Action": "sts:AssumeRole"

  }]

}

```

  

**Security Notes:**

- ✅ Scoped to specific EKS cluster ARN (not wildcard)

- ✅ Scoped to specific Bedrock model

- ✅ No S3 or database access

- ✅ SSM Session Manager eliminates SSH key management

  

---

  

### 2. EKS Cluster Role

  

**Name:** `EKS-cluster-role-{env}` (e.g., `EKS-cluster-role-dev`)

  

**Used By:**

- EKS Control Plane (Kubernetes masters)

  

**Authentication:**

- Service-linked role (AWS manages authentication)

  

**Permissions (Managed Policies):**

  

1. **AmazonEKSClusterPolicy** (AWS Managed)

   - Create/manage AWS resources for EKS cluster

   - Actions:

     - `ec2:CreateNetworkInterface`, `ec2:DescribeSubnets`

     - `elasticloadbalancing:*` (for LoadBalancer services)

     - `iam:CreateServiceLinkedRole`

     - `logs:CreateLogGroup`, `logs:PutLogEvents`

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "eks.amazonaws.com"

    },

    "Action": "sts:AssumeRole"

  }]

}

```

  

**What the Control Plane Does:**

- Runs Kubernetes API server, scheduler, controller manager

- Provisions ENIs for pod networking

- Creates security groups

- Integrates with CloudWatch for logging

  

**Security Notes:**

- ✅ Managed by AWS (no custom policies)

- ✅ Limited to EKS service operations

- ✅ Cannot access application data or pods directly

  

---

  

### 3. EKS Node Role

  

**Name:** `EKS-node-role-{env}` (e.g., `EKS-node-role-dev`)

  

**Used By:**

- Worker nodes (EC2 instances in node group)

  

**Authentication:**

- EC2 Instance Profile (attached to node group)

  

**Permissions (Managed Policies):**

  

1. **AmazonEKSWorkerNodePolicy** (AWS Managed)

   - Join EKS cluster

   - Update node status

   - Actions: `ec2:DescribeInstances`, `eks:DescribeCluster`

  

2. **AmazonEC2ContainerRegistryReadOnly** (AWS Managed)

   - Pull container images from ECR

   - Actions: `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`

  

3. **AmazonEKS_CNI_Policy** (AWS Managed)

   - Manage pod networking (ENIs, IP addresses)

   - Actions: `ec2:AssignPrivateIpAddresses`, `ec2:AttachNetworkInterface`

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "ec2.amazonaws.com"

    },

    "Action": "sts:AssumeRole"

  }]

}

```

  

**What Nodes Do:**

- Run kubelet (Kubernetes node agent)

- Pull container images from ECR

- Assign IP addresses to pods

- Report node/pod status to control plane

  

**Security Notes:**

- ✅ Read-only ECR access (can't push images)

- ✅ Limited EC2 operations (only networking)

- ✅ No access to other AWS services

  

---

  

### 4. EBS CSI Driver Role

  

**Name:** `{cluster_name}-ebs-csi-driver` (e.g., `platform-dev-ebs-csi-driver`)

  

**Used By:**

- EBS CSI Controller pods in `kube-system` namespace

  

**Authentication:**

- **Pod Identity** (modern, simplified approach)

- Service Account: `ebs-csi-controller-sa`

  

**Permissions (Managed Policies):**

  

1. **AmazonEBSCSIDriverPolicy** (AWS Managed)

   - Create/attach/delete EBS volumes

   - Actions:

     - `ec2:CreateVolume`, `ec2:DeleteVolume`

     - `ec2:AttachVolume`, `ec2:DetachVolume`

     - `ec2:CreateSnapshot`, `ec2:DeleteSnapshot`

     - `ec2:DescribeVolumes`, `ec2:DescribeSnapshots`

     - `ec2:CreateTags`

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "pods.eks.amazonaws.com"

    },

    "Action": ["sts:AssumeRole", "sts:TagSession"]

  }]

}

```

  

**Pod Identity Association:**

```hcl

resource "aws_eks_pod_identity_association" "ebs_csi_driver" {

  cluster_name    = aws_eks_cluster.this.name

  namespace       = "kube-system"

  service_account = "ebs-csi-controller-sa"

  role_arn        = module.ebs_csi_driver_role.role_arn

}

```

  

**What EBS CSI Driver Does:**

- Provisions EBS volumes when PVC is created

- Attaches volumes to nodes where pods are scheduled

- Deletes volumes when PVC is deleted (if reclaim policy is Delete)

  

**Use Cases:**

- Jenkins agent workspace persistence

- Database data persistence (if running DB in K8s)

- Stateful application storage

  

**Security Notes:**

- ✅ Limited to EBS operations only

- ✅ Cannot access other AWS services

- ✅ Automatic credential rotation via Pod Identity

  

---

  

### 5. Chatbot Backend Role

  

**Name:** `{cluster_name}-chatbot-backend` (e.g., `platform-dev-chatbot-backend`)

  

**Used By:**

- Chatbot backend pods in `default` namespace

  

**Authentication:**

- **Pod Identity**

- Service Account: `chatbot-backend-service-account`

  

**Custom Policies:**

  

#### Chatbot Backend Bedrock Policy

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Sid": "InvokeBedrock",

    "Effect": "Allow",

    "Action": [

      "bedrock:InvokeModel",

      "bedrock:InvokeModelWithResponseStream"

    ],

    "Resource": "arn:aws:bedrock:us-east-2::foundation-model/deepseek.v3-v1:0"

  }]

}

```

  

#### Chatbot Backend Secrets Policy

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Sid": "ReadSecrets",

    "Effect": "Allow",

    "Action": [

      "secretsmanager:GetSecretValue",

      "secretsmanager:DescribeSecret"

    ],

    "Resource": "arn:aws:secretsmanager:us-east-2:*:secret:platform-db-*"

  }]

}

```

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "pods.eks.amazonaws.com"

    },

    "Action": ["sts:AssumeRole", "sts:TagSession"]

  }]

}

```

  

**Pod Identity Association:**

```hcl

resource "aws_eks_pod_identity_association" "chatbot_backend" {

  cluster_name    = aws_eks_cluster.this.name

  namespace       = var.chatbot_namespace  # default

  service_account = "chatbot-backend-service-account"

  role_arn        = module.chatbot_backend_role.role_arn

}

```

  

**What Backend Pods Do:**

- Call AWS Bedrock DeepSeek V3 model for AI responses

- Fetch database credentials from Secrets Manager (via init container)

- Store/retrieve conversation history from RDS MySQL

  

**Security Notes:**

- ✅ Scoped to specific Bedrock model only

- ✅ Scoped to secrets matching `platform-db-*` pattern

- ✅ Cannot create/delete secrets

- ✅ No access to other AWS services (S3, EC2, etc.)

  

---

  

### 6. AWS Load Balancer Controller Role (To Be Added)

  

**Name:** `{cluster_name}-aws-load-balancer-controller` (e.g., `platform-dev-aws-load-balancer-controller`)

  

**Used By:**

- AWS Load Balancer Controller pods in `kube-system` namespace

  

**Authentication:**

- **Pod Identity**

- Service Account: `aws-load-balancer-controller`

  

**Permissions (Custom Policy from AWS):**

  

Official policy from: https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

  

**Key Permissions:**

- `elasticloadbalancing:*` - Create/manage ALB, NLB, Target Groups

- `ec2:DescribeSubnets` - Find subnets for ALB placement

- `ec2:DescribeSecurityGroups` - Manage ALB security groups

- `ec2:CreateTags`, `ec2:DeleteTags` - Tag AWS resources

- `iam:CreateServiceLinkedRole` - Create ALB service-linked role

- `waf:*`, `wafv2:*` - Integrate with AWS WAF (optional)

- `shield:*` - Integrate with AWS Shield (optional)

  

**Trust Relationship:**

```json

{

  "Version": "2012-10-17",

  "Statement": [{

    "Effect": "Allow",

    "Principal": {

      "Service": "pods.eks.amazonaws.com"

    },

    "Action": ["sts:AssumeRole", "sts:TagSession"]

  }]

}

```

  

**Pod Identity Association (To Be Added):**

```hcl

resource "aws_eks_pod_identity_association" "aws_load_balancer_controller" {

  cluster_name    = aws_eks_cluster.this.name

  namespace       = "kube-system"

  service_account = "aws-load-balancer-controller"

  role_arn        = module.aws_load_balancer_controller_role.role_arn

}

```

  

**What Load Balancer Controller Does:**

- Watches for Ingress resources in Kubernetes

- Provisions AWS Application Load Balancers (ALB)

- Creates Target Groups and registers pod IPs

- Configures ALB listeners (HTTP, HTTPS)

- Attaches ACM certificates to ALB

- Deletes ALB when Ingress is deleted

  

**Security Notes:**

- ✅ Scoped to load balancing operations only

- ✅ Cannot launch EC2 instances

- ✅ Cannot access application data

- ⚠️ Has broad `elasticloadbalancing:*` permissions (required by AWS)

  

---

  

## Pod Identity vs IRSA Comparison

  

### Old Way: IRSA (IAM Roles for Service Accounts)

  

**Setup:**

1. Create OIDC provider for EKS cluster

2. Create IAM role with OIDC trust policy

3. Annotate Kubernetes service account

4. Inject AWS credentials via webhook

  

**Complexity:**

- OIDC provider management

- Complex trust policies with StringEquals conditions

- Service account annotations

- Webhook configuration

  

---

  

### New Way: Pod Identity (Used in This Project)

  

**Setup:**

1. Create IAM role with `pods.eks.amazonaws.com` trust

2. Create Pod Identity association (cluster + namespace + SA + role)

3. Done!

  

**Benefits:**

- ✅ Simpler trust relationship

- ✅ No OIDC provider needed

- ✅ Centralized association management

- ✅ Works across multiple clusters

- ✅ Modern approach (introduced 2023)

  

**Example:**

```hcl

# Old way (IRSA) - ~20 lines of code

resource "aws_iam_openid_connect_provider" "eks" { ... }

resource "aws_iam_role" "this" {

  assume_role_policy = jsonencode({

    Statement = [{

      Principal = {

        Federated = aws_iam_openid_connect_provider.eks.arn

      }

      Condition = {

        StringEquals = {

          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"

        }

      }

    }]

  })

}

  

# New way (Pod Identity) - 3 lines of code

resource "aws_eks_pod_identity_association" "this" {

  cluster_name    = "platform-dev"

  namespace       = "kube-system"

  service_account = "aws-load-balancer-controller"

  role_arn        = aws_iam_role.this.arn

}

```

  

---

  

## Role Module Reusability

  

All roles use the same reusable module: `terraform/modules/role`

  

**Module supports two use cases:**

  

### 1. EC2 Instance Role (Default)

```hcl

module "jenkins_role" {

  source = "../modules/role"

  name   = "jenkins-role"

  policy_arns = ["arn:aws:iam::aws:policy/..."]

}

```

**Trust:** `ec2.amazonaws.com`  

**Output:** IAM role + instance profile

  

---

  

### 2. Pod Identity Role

```hcl

module "ebs_csi_driver_role" {

  source  = "../modules/role"

  name    = "ebs-csi-driver-role"

  service = "pods.eks.amazonaws.com"  # Switch to Pod Identity

  policy_arns = ["arn:aws:iam::aws:policy/..."]

}

```

**Trust:** `pods.eks.amazonaws.com`  

**Output:** IAM role only (no instance profile)

  

---

  

## Security Best Practices

  

### 1. Least Privilege

- ✅ Each role has only required permissions

- ✅ Resource-level restrictions where possible (ARN-specific)

- ✅ No `*` wildcards in Resource fields (except where required by AWS)

  

### 2. Scoped Policies

- ✅ Jenkins: Scoped to specific EKS cluster ARN

- ✅ Chatbot Backend: Scoped to specific Bedrock model

- ✅ Secrets access: Scoped to `platform-db-*` pattern

  

### 3. No Static Credentials

- ✅ EC2 roles: Instance profile

- ✅ Pod roles: Pod Identity (temporary credentials)

- ✅ Automatic credential rotation

  

### 4. Separation of Concerns

- ✅ Jenkins role: CI/CD operations

- ✅ Node role: Kubernetes infrastructure

- ✅ Backend role: Application operations

- ✅ CSI driver role: Storage operations

  

### 5. Audit Trail

- ✅ All role assumptions logged in CloudTrail

- ✅ Pod Identity associations visible in EKS console

- ✅ Tags for resource organization

  

---

  

## Troubleshooting

  

### Permission Denied Errors

  

**Symptom:** Pod logs show `AccessDenied` or `UnauthorizedOperation`

  

**Debug steps:**

1. Check Pod Identity association exists:

   ```bash

   aws eks list-pod-identity-associations --cluster-name platform-dev

   ```

  

2. Verify service account matches:

   ```bash

   kubectl get pod <pod-name> -o yaml | grep serviceAccountName

   ```

  

3. Check IAM role has required policies:

   ```bash

   aws iam list-attached-role-policies --role-name platform-dev-chatbot-backend

   ```

  

4. Test IAM policy simulator:

   - AWS Console → IAM → Policy Simulator

   - Select role and test specific actions

  

---

  

### Pod Identity Not Working

  

**Common causes:**

- Pod Identity Agent not installed (`eks-pod-identity-agent` addon)

- Service account name mismatch

- Wrong namespace in association

- Role ARN incorrect

  

**Verify:**

```bash

# Check Pod Identity Agent

kubectl get pods -n kube-system | grep pod-identity-agent

  

# Check associations

aws eks describe-pod-identity-association \

  --cluster-name platform-dev \

  --association-id <id>

```

  

---

  

## Cost Impact

  

**IAM Roles:** **FREE**

- No charge for creating IAM roles

- No charge for policy attachments

- No charge for Pod Identity associations

  

**What you pay for:**

- AWS services accessed by roles (Bedrock API calls, EBS volumes, etc.)

- Not the authentication mechanism itself

  

---

  

## Interview Talking Points

  

### Question: "Explain the IAM roles in your EKS infrastructure"

  

**Strong Answer:**

> "We use 6 IAM roles following least-privilege principles:

>

> 1. **Jenkins role** for CI/CD with SSM access and ECR push/pull

> 2. **EKS cluster role** for control plane operations

> 3. **Node role** for worker nodes to join cluster and pull images

> 4. **EBS CSI driver role** via Pod Identity for volume management

> 5. **Chatbot backend role** for Bedrock API and Secrets Manager access

> 6. **Load Balancer controller role** for ALB provisioning

>

> We use **Pod Identity** (not IRSA) for Kubernetes workloads - it's simpler than OIDC and eliminates static credentials. All roles are scoped to specific resources using ARN-level restrictions."

  

---

  

### Question: "How do pods authenticate to AWS?"

  

**Strong Answer:**

> "We use **EKS Pod Identity**, which is AWS's modern authentication mechanism. Here's how it works:

>

> 1. We create an IAM role that trusts `pods.eks.amazonaws.com`

> 2. We create a Pod Identity association linking cluster + namespace + service account + IAM role

> 3. When a pod starts with that service account, the Pod Identity Agent (running on nodes) provides temporary credentials

> 4. The pod uses AWS SDK to call services - credentials auto-rotate

>

> This is simpler than IRSA because it doesn't require OIDC providers or complex trust policies. It's also more secure than instance roles because permissions are scoped per-pod, not per-node."

  

---

  

## Summary Checklist

  

**Existing Roles (Already Deployed):**

- [x] Jenkins EC2 role with SSM, ECR, EKS access

- [x] EKS cluster role for control plane

- [x] EKS node role for worker nodes

- [x] EBS CSI driver role with Pod Identity

- [x] Chatbot backend role with Pod Identity

  

**To Be Added:**

- [ ] AWS Load Balancer Controller role with Pod Identity

- [ ] Load Balancer Controller IAM policy (from AWS official repo)

- [ ] Pod Identity association for controller

  

---

  

**Document Version:** 1.0  

**Last Updated:** 2025-12-02  

**Project:** terraform-aws-eks  

**Repository:** https://github.com/mostafamedhat1983/terraform-aws-eks