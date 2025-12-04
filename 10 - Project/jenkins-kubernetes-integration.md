---
tags:
  - My_CV_Project
  - Jenkins
  - Kubernetes
---
# Jenkins Kubernetes Integration Setup

## Overview

This guide explains how to configure Jenkins to use EKS as a Kubernetes cloud for dynamic agent provisioning. Jenkins will create ephemeral pods in EKS to run pipeline jobs instead of using static EC2 instances.

## Why This Integration is Needed

### The Problem
- **Static Jenkins agents** are expensive and underutilized
- **Manual scaling** doesn't handle variable workloads efficiently
- **Resource waste** when agents sit idle

### The Solution
- **Dynamic pod creation** - Jenkins spawns agents on-demand
- **Cost optimization** - Pay only for active build time
- **Scalability** - Unlimited parallel builds (within cluster limits)
- **Isolation** - Each build gets fresh environment

### Architecture Benefits
```
Jenkins Controller (EC2) → EKS API → Agent Pods (on-demand)
```
- ✅ **Controller**: Persistent, manages jobs and UI
- ✅ **Agents**: Ephemeral, created per build, auto-deleted
- ✅ **Storage**: EBS volumes for workspace persistence

## Authentication Challenge

### The Core Issue
Jenkins and kubectl use **different authentication methods**:

- **kubectl**: Uses `~/.kube/config` + AWS CLI profile
- **Jenkins Plugin**: Needs explicit Kubernetes credentials

Even though both run on the same EC2 instance with EKS access, Jenkins plugin requires separate authentication configuration.

### Why AWS IAM Alone Isn't Enough
1. **Jenkins EC2 has EKS access** via IAM role and Access Entry
2. **kubectl works** because it uses kubeconfig + AWS credentials
3. **Jenkins plugin fails** because it doesn't inherit EC2 instance credentials automatically
4. **Solution**: Create Kubernetes service account with cluster admin permissions

## Setup Options

### Option 1: Automated Setup (Not Recommended)

**Why automation is problematic:**
- ❌ Jenkins startup timing is unpredictable (2-3 minutes)
- ❌ Jenkins CLI requires authentication (admin user/token)
- ❌ Initial admin password is random and must be retrieved first
- ❌ Terraform can't reliably wait for Jenkins readiness
- ❌ Debugging automation failures wastes more time than manual setup
- ❌ Jenkins configuration not tracked in Terraform state

**If you still want to automate service account creation only:**

```hcl
# In main.tf - Creates service account only (not Jenkins credentials)
resource "null_resource" "jenkins_k8s_serviceaccount" {
  provisioner "local-exec" {
    command = <<-EOT
      aws eks update-kubeconfig --name ${var.cluster_name} --region us-east-2
      
      kubectl create serviceaccount jenkins-sa -n default --dry-run=client -o yaml | kubectl apply -f -
      
      kubectl create clusterrolebinding jenkins-admin \
        --clusterrole=cluster-admin \
        --serviceaccount=default:jenkins-sa \
        --dry-run=client -o yaml | kubectl apply -f -
      
      echo "Service account created. Generate token manually:"
      echo "kubectl create token jenkins-sa --duration=8760h -n default"
    EOT
  }
  
  depends_on = [module.eks]
}
```

**Then complete manually:**
1. Generate token: `kubectl create token jenkins-sa --duration=8760h -n default`
2. Add to Jenkins UI (see Option 2 for detailed steps)

### Option 2: Manual Setup (Recommended)

**Why manual is better:**
- Jenkins must be fully started and configured
- One-time setup (5 minutes per environment)
- Easy to verify each step
- No timing issues or race conditions
- Better for learning and troubleshooting

**Steps:**

```bash
# 1. Connect to Jenkins EC2 via SSM
aws ssm start-session --target <jenkins-instance-id> --region us-east-2

# 2. Configure kubectl (if not already done)
aws eks update-kubeconfig --name platform-dev --region us-east-2

# 3. Create Kubernetes service account
kubectl create serviceaccount jenkins-sa -n default

# 4. Create cluster role binding
kubectl create clusterrolebinding jenkins-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:jenkins-sa

# 5. Generate token (valid for 1 year)
kubectl create token jenkins-sa --duration=8760h -n default

# Copy the token output - you'll need it in the next step
```

**6. Add token to Jenkins UI:**
- Navigate to: **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)**
- Click **Add Credentials**
- **Kind:** Secret text
- **Scope:** Global
- **Secret:** Paste the token from step 5
- **ID:** `jenkins-k8s-token` (exact match required)
- **Description:** Kubernetes Service Account Token
- Click **Create**

## Jenkins Configuration

### 1. Install Kubernetes Plugin
- **Manage Jenkins** → **Plugins** → **Available**
- Search for "Kubernetes"
- Install and restart Jenkins

### 2. Configure Kubernetes Cloud
- **Manage Jenkins** → **Configure Clouds** → **Add Kubernetes**

**Required Settings:**
- **Name:** eks-cluster
- **Kubernetes URL:** `https://<your-eks-endpoint>`
- **Kubernetes Namespace:** default
- **Credentials:** jenkins-k8s-token
- **☑️ Disable https certificate check** (required for self-signed EKS certs)
- **☑️ WebSocket** (enabled by default - required for agent communication)

**Optional (use defaults):**
- Connection Timeout: 5 seconds
- Read Timeout: 15 seconds
- Max connections: 32

**Pod Template (optional - can also define in Jenkinsfile):**
- Scroll down to **Pod Templates** section
- Click **Add Pod Template** → **Pod Template details**
- **Name:** jenkins-agent
- **Namespace:** default
- **Labels:** jenkins-agent (used in pipeline: `agent { label 'jenkins-agent' }`)
- Add containers as needed (or skip and define in Jenkinsfile YAML)

### 3. Test Connection
- Click **Test Connection**
- Should show: "Connected to Kubernetes"

## Pod Template Configuration

### Basic Agent Template
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "200m"
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: workspace
      mountPath: /workspace
  volumes:
  - name: workspace
    persistentVolumeClaim:
      claimName: jenkins-workspace-pvc
      storageClassName: ebs-storage-class
```

### Storage Configuration
**Uses existing EBS storage class:**
- **StorageClass**: `ebs-storage-class` (already configured)
- **Volume Type**: GP3 with encryption
- **Expansion**: Enabled for growing workspaces

## Pipeline Usage

### Example Jenkinsfile
```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep']
    args: ['infinity']
"""
        }
    }
    
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}
```

## Troubleshooting

### Common Issues

**1. "Unauthorized" Error**
- **Cause**: Service account not created or token expired
- **Fix**: Recreate service account and token

**2. "Certificate Unknown" Error**
- **Cause**: SSL certificate validation
- **Fix**: Enable "Disable https certificate check"

**3. "Connection Timeout"**
- **Cause**: Network connectivity or wrong endpoint
- **Fix**: Verify EKS endpoint and security groups

**4. Pods Not Starting**
- **Cause**: Resource constraints or image pull issues
- **Fix**: Check node capacity and image availability

### Debug Commands
```bash
# Check service account
kubectl get serviceaccount jenkins-sa

# Check cluster role binding
kubectl get clusterrolebinding jenkins-admin

# Test token
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:jenkins-sa

# Check pod logs
kubectl logs <jenkins-agent-pod>
```

## Security Considerations

### Service Account Permissions
- **Current**: Cluster admin (full access)
- **Production**: Consider least privilege RBAC
- **Namespace isolation**: Restrict to specific namespaces

### Token Management
- **Duration**: 1 year (8760 hours)
- **Rotation**: Manual renewal required
- **Storage**: Encrypted in Jenkins credentials

### Network Security
- **Private EKS**: API endpoint not public
- **SSM Access**: No SSH keys or bastion hosts
- **Security Groups**: Restricted access between components

## Benefits Achieved

### Cost Optimization
- **No idle agents** - Pods created only when needed
- **Resource efficiency** - Right-sized containers per job
- **Shared infrastructure** - Multiple builds on same nodes

### Scalability
- **Parallel builds** - Limited only by cluster capacity
- **Auto-scaling** - EKS nodes scale based on demand
- **Isolation** - Each build gets clean environment

### Operational Excellence
- **Immutable agents** - Fresh environment per build
- **Persistent storage** - Workspaces survive pod restarts
- **Centralized logging** - All logs in Kubernetes

## Next Steps

1. **Create first pipeline** using Kubernetes agents
2. **Configure pod templates** for different job types
3. **Set up monitoring** for agent usage and costs
4. **Implement RBAC** for production security
5. **Add resource quotas** to prevent resource exhaustion

---

**Note**: This integration transforms Jenkins from static agent architecture to dynamic, cloud-native CI/CD platform, enabling cost-effective and scalable build automation.