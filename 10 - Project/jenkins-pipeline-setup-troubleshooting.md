---
tags:
  - My_CV_Project
  - Jenkins
---
# Jenkins Pipeline Setup and Troubleshooting Guide

Complete record of Jenkins pipeline implementation with all issues encountered and solutions applied.

---

## Overview

Implemented 4 Jenkins pipelines for automated deployment to EKS:
1. **Setup Pipeline** - Configures kubectl context (one-time)
2. **ALB Controller Pipeline** - Installs AWS Load Balancer Controller (one-time)
3. **Monitoring Stack Pipeline** - Deploys Metrics Server, Prometheus, Grafana (one-time)
4. **Application Pipeline** - Builds and deploys chatbot application (automated)

---

## Architecture Decisions

### Global Environment Variable Approach
**Decision:** Use Jenkins global `TARGET_ENVIRONMENT` variable instead of pipeline parameters

**Reason:** Simplifies workflow - set environment once, all pipelines use it automatically

**Configuration:**
- Jenkins → Manage Jenkins → System → Global properties → Environment variables
- Add: `TARGET_ENVIRONMENT` = `dev` or `prod`

### Kubernetes Agent Pods
**Decision:** Run pipelines as pods on EKS (not on Jenkins EC2)

**Reason:** Isolation, scalability, and resource efficiency

**Requirement:** Setup pipeline must run first to configure kubectl context

---

## Issues Encountered and Solutions

### Issue 1: Kubernetes Agent Pod Connection Timeout

**Problem:** Pods created but couldn't connect to Jenkins
```
Connection timed out
http://10.0.3.132:8080/login is not ready
```

**Root Cause:** Jenkins security group didn't allow traffic from EKS pods

**Investigation:**
1. Checked EKS nodes use cluster security group (`sg-0197d3df1c0fceaa7`), not the node security group we created
2. Jenkins security group only allowed port 50000 from itself

**Solution:** Updated Terraform to allow traffic from EKS cluster security group
```hcl
# terraform/modules/network/main.tf
resource "aws_security_group_rule" "jenkins_web" {
  from_port                = 8080
  source_security_group_id = var.eks_cluster_security_group_id  # EKS cluster SG
  security_group_id        = aws_security_group.jenkins.id
}

resource "aws_security_group_rule" "jenkins_agent" {
  from_port                = 50000
  source_security_group_id = var.eks_cluster_security_group_id
  security_group_id        = aws_security_group.jenkins.id
}
```

**Removed:** Unused `eks_node` security group that wasn't being attached to nodes

**Applied:** `terraform apply` in `terraform/dev`

---

### Issue 2: RBAC Permissions Error

**Problem:** Helm couldn't list secrets in kube-system namespace
```
secrets is forbidden: User "system:serviceaccount:default:default" cannot list resource "secrets"
```

**Root Cause:** Pods used default service account with no permissions

**Solution:** Use `jenkins-sa` service account with cluster-admin role
```yaml
# Jenkinsfile-alb-controller
spec:
  serviceAccountName: jenkins-sa
```

**Note:** `jenkins-sa` was already created during post-deployment setup with proper RBAC

---

### Issue 3: kubectl Not Found in Container

**Problem:** Verify stage failed - kubectl command not found
```
kubectl: not found
```

**Root Cause:** `alpine/helm` image only has Helm, not kubectl

**Solution:** Use `alpine/k8s:1.30.7` image which includes both kubectl and Helm
```yaml
containers:
  - name: helm
    image: alpine/k8s:1.30.7
```

---

## Jenkins Configuration

### Global Properties
- **TARGET_ENVIRONMENT:** `dev` or `prod`
- **Jenkins URL:** `http://10.0.3.132:8080` (Jenkins private IP)
- **Jenkins tunnel:** (empty - using WebSocket)

### Kubernetes Cloud Configuration
- **Kubernetes URL:** `https://<eks-endpoint>`
- **Namespace:** default
- **Credentials:** jenkins-k8s-token (service account token)
- **WebSocket:** ☑️ Enabled
- **Disable HTTPS certificate check:** ☑️ Enabled

---

## Pipeline Files

### Jenkinsfile-setup
- **Purpose:** Configure kubectl context on Jenkins EC2
- **Agent:** `any` (runs on Jenkins EC2, not Kubernetes pod)
- **When to run:** Once per environment or when switching environments
- **What it does:** Runs `aws eks update-kubeconfig --name platform-{env}`

### Jenkinsfile-alb-controller
- **Purpose:** Install AWS Load Balancer Controller
- **Agent:** Kubernetes pod with `alpine/k8s:1.30.7` image
- **Service Account:** jenkins-sa
- **When to run:** Once per environment
- **What it does:** Helm install ALB controller, verify deployment

### Jenkinsfile-monitoring
- **Purpose:** Deploy monitoring stack
- **Agent:** Kubernetes pod with kubectl and helm containers
- **Service Account:** jenkins-sa
- **When to run:** Once per environment
- **What it does:** Deploy Metrics Server, Prometheus, Grafana with ingress

### Jenkinsfile (Application)
- **Purpose:** Build and deploy chatbot application
- **Agent:** Kubernetes pod with docker and helm containers
- **When to run:** Automatically on Git commits
- **What it does:** Build images, push to ECR, deploy with Helm

---

## Deployment Workflow

### Initial Setup (One-Time)
1. Set `TARGET_ENVIRONMENT` = `dev` in Jenkins global properties
2. Run **Setup Pipeline** → configures kubectl for dev cluster
3. Run **ALB Controller Pipeline** → installs ALB controller
4. Run **Monitoring Stack Pipeline** → deploys monitoring
5. Run **Application Pipeline** → deploys chatbot

### Switching Environments
1. Change `TARGET_ENVIRONMENT` to `prod` in Jenkins
2. Run **Setup Pipeline** → reconfigures kubectl for prod cluster
3. Run **ALB Controller Pipeline** → installs ALB controller in prod
4. Run **Monitoring Stack Pipeline** → deploys monitoring in prod
5. Run **Application Pipeline** → deploys chatbot to prod

### Daily Operations
- Application pipeline runs automatically on Git commits
- Uses current `TARGET_ENVIRONMENT` value
- No manual intervention needed

---

## Troubleshooting Commands

### Check Pod Status
```bash
kubectl get pods -n default | grep <pipeline-name>
kubectl describe pod <pod-name>
```

### Check Pod Logs
```bash
kubectl logs <pod-name> -c helm
kubectl logs <pod-name> -c jnlp  # Jenkins agent logs
```

### Check Security Groups
```bash
# Get Jenkins security group
aws ec2 describe-instances --instance-ids <jenkins-id> \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId'

# Check rules
aws ec2 describe-security-groups --group-ids <sg-id> \
  --query 'SecurityGroups[0].IpPermissions[?FromPort==`8080`]'
```

### Check EKS Node Security Group
```bash
# Get node IP
kubectl get nodes -o wide

# Get node security groups
aws ec2 describe-instances --filters "Name=private-ip-address,Values=<node-ip>" \
  --query 'Reservations[0].Instances[0].SecurityGroups[*].[GroupId,GroupName]'
```

### Test Connectivity from Pod
```bash
kubectl exec -it <pod-name> -c helm -- sh
curl -v http://10.0.3.132:8080
```

---

## Key Learnings

1. **EKS creates its own cluster security group** - Don't create a separate node security group, use the one EKS provides
2. **Jenkins needs port 8080 open** - Not just 50000 for agents, also 8080 for WebSocket connection
3. **Service accounts are critical** - Default service account has no permissions, must use jenkins-sa
4. **Image selection matters** - Use images with all required tools (alpine/k8s has kubectl + helm)
5. **Setup pipeline is mandatory** - Must run before other pipelines to configure kubectl context
6. **Global variables simplify workflow** - Better than parameters for environment selection

---

## Current Status

✅ Jenkins security group configured with EKS cluster SG  
✅ Setup pipeline working (configures kubectl)  
✅ ALB controller pipeline working (installs controller)  
✅ Monitoring pipeline ready (not yet tested)  
✅ Application pipeline ready (not yet tested)  
✅ Global TARGET_ENVIRONMENT variable configured  
✅ All pipelines use Kubernetes agents with proper RBAC  

---

## Next Steps

1. Test monitoring stack pipeline
2. Test application pipeline
3. Configure Route 53 DNS for ALB endpoints
4. Set up GitHub webhooks for automatic application deployments
5. Document DNS setup process
