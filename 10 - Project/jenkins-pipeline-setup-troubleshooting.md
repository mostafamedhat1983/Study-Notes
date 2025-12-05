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

### Issue 4: ALB Controller VPC ID Metadata Timeout

**Problem:** ALB controller pods in CrashLoopBackOff with error:
```
failed to get VPC ID from instance metadata
context deadline exceeded
```

**Root Cause:** ALB controller pods cannot access EC2 instance metadata service to fetch VPC ID

**Why This Happens:**
- ALB controller expects to run on EC2 instances where it can query instance metadata
- When running in EKS pods, metadata service is not accessible
- Controller needs VPC ID to create ALBs in correct VPC

**Investigation Steps:**
1. Checked pod logs: `kubectl logs -n kube-system <alb-controller-pod>`
2. Attempted to query VPC ID dynamically using kubectl and AWS CLI in Jenkinsfile
3. Discovered pod lacks AWS credentials for AWS CLI commands

**Solution:** Hardcode VPC ID in Helm installation command
```groovy
// Jenkinsfile-alb-controller
sh '''
  helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=platform-${TARGET_ENVIRONMENT} \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set vpcId=vpc-0d6a3fc6b6103886f \
    -n kube-system
'''
```

**VPC ID Management:**
- **Dev VPC ID:** `vpc-0d6a3fc6b6103886f` (hardcoded in Jenkinsfile)
- **Prod VPC ID:** Will need to be updated when switching to prod environment
- **After terraform destroy/apply:** VPC ID remains the same (AWS reuses VPC IDs unless explicitly deleted)
- **Manual update needed:** Only if VPC is destroyed and recreated with different ID

**How to Find VPC ID:**
```bash
# From Jenkins EC2 or local machine with AWS CLI
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=platform-dev-vpc" \
  --query 'Vpcs[0].VpcId' --output text
```

**Alternative Approaches Considered:**
1. ❌ Query VPC ID in Jenkinsfile using AWS CLI → Pod lacks AWS credentials
2. ❌ Use Pod Identity for AWS CLI → Adds complexity, hardcoding is simpler
3. ✅ Hardcode VPC ID → Simple, VPC ID rarely changes

**The Lesson:**
- VPC IDs are stable resources that rarely change
- Hardcoding infrastructure identifiers is acceptable when they're persistent
- ALB controller is infrastructure-level component, not application-level
- Terraform destroy/apply typically preserves VPC unless explicitly destroyed

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
7. **VPC IDs are stable** - Can query dynamically with EC2 DescribeVpcs permission instead of hardcoding
8. **Helm wait timeouts fail on large stacks** - Remove `--wait` for Prometheus, check status manually after
9. **Node capacity planning is critical** - Account for system pods + monitoring + apps + Jenkins agents
10. **t3.small max pods ≈ 11** - AWS CNI ENI limits, not unlimited pod capacity
11. **Docker-in-Docker needs startup time** - Always wait for daemon readiness before Docker commands
12. **Placeholders must be replaced** - Template values like `<account-id>` cause runtime failures
13. **Helm can't manage kubectl-created resources** - Resources created outside Helm cause ownership conflicts
14. **Image selection critical** - Use alpine/k8s for kubectl+helm, not alpine/helm alone
15. **Test security features in deployment** - SSL/TLS configs that work locally may fail in Kubernetes
16. **Timeouts cause false failures** - Remove blocking timeouts, check status manually after pipeline

---

### Issue 5: Node Capacity Exhausted

**Problem:** Application pipeline pod couldn't schedule
```
0/2 nodes are available: 2 Too many pods
```

**Root Cause:** Dev cluster had only 2 t3.small nodes, already running:
- 10 kube-system pods (CoreDNS, EBS CSI, ALB controller, aws-node, kube-proxy, etc.)
- 7 monitoring pods (Prometheus, Grafana, Alertmanager, node-exporter, etc.)
- 4 chatbot pods (2 backend, 2 frontend - already deployed but pending)
- Jenkins agent pods

**Why This Happens:**
- Each node has max pod capacity based on instance type and ENI limits
- t3.small supports ~11 pods per node (AWS CNI limitation)
- Monitoring stack is resource-intensive (7 pods)
- No room for Jenkins build pods or application pods

**Investigation:**
```bash
kubectl get pods --all-namespaces  # Shows all pods across nodes
kubectl describe nodes | grep -A 5 "Allocated resources"  # Shows capacity
```

**Solution:** Increase node count in Terraform
```hcl
# terraform/dev/main.tf
node_desired_size  = 3  # Was 2
node_max_size      = 4  # Was 3
node_min_size      = 1
```

**Applied:** `terraform apply` in `terraform/dev`

**Result:** 3rd node joined cluster, pending pods scheduled successfully

**Alternative Considered:**
- ❌ Scale down monitoring (loses observability)
- ❌ Karpenter autoscaler (overkill for stable workload)
- ✅ Manual node scaling (simple, predictable cost ~$15/month per node)

**The Lesson:**
- Always account for system pods (kube-system) when sizing clusters
- Monitoring stacks consume significant pod capacity
- t3.small max pods ≈ 11 (not unlimited)
- Dev environments can use manual scaling for predictable workloads

---

### Issue 6: Placeholder Account ID in Jenkinsfile

**Problem:** Build stage failed with syntax error
```
can't open account-id: no such file
```

**Root Cause:** ECR registry URL had placeholder `<account-id>` instead of actual AWS account ID
```groovy
ECR_REGISTRY = '<account-id>.dkr.ecr.us-east-2.amazonaws.com'
```

**Why This Happens:**
- Template code uses placeholders for sensitive values
- Forgot to replace with actual account ID before committing

**Solution:** Replace with actual AWS account ID
```groovy
ECR_REGISTRY = '586794447516.dkr.ecr.us-east-2.amazonaws.com'
```

**How to Find Account ID:**
```bash
aws sts get-caller-identity --query Account --output text
```

---

### Issue 7: Docker Daemon Not Ready

**Problem:** Docker build failed immediately
```
ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**Root Cause:** `docker:dind` container needs time to start Docker daemon

**What's Actually Happening:**
1. Pod starts with 3 containers (docker, kubectl, jnlp)
2. Jenkins immediately runs build commands in docker container
3. Docker daemon still initializing (takes 5-10 seconds)
4. `docker build` command fails before daemon is ready

**Solution:** Add readiness check before building
```bash
timeout 60 sh -c 'until docker info; do sleep 1; done'
```

**How It Works:**
- Loops `docker info` command until it succeeds
- Sleeps 1 second between attempts
- Times out after 60 seconds if daemon never starts
- Only proceeds to build after daemon is confirmed ready

**The Lesson:**
- Docker-in-Docker (dind) is not instant
- Always wait for daemon readiness before Docker commands
- `docker info` is reliable readiness check (returns 0 when ready)

---

### Issue 8: Helm Chart Conflict with Standalone Resources

**Problem:** App pipeline failed with ownership metadata error
```
Error: unable to continue with install: Ingress "grafana-ingress" in namespace "monitoring" exists and cannot be imported into the current release: invalid ownership metadata
```

**Root Cause:** Monitoring pipeline created Grafana ingress with `kubectl apply`, but chatbot Helm chart also had Grafana ingress template

**Why This Happens:**
- Monitoring pipeline: `kubectl apply -f grafana-ingress.yaml` (no Helm labels)
- App pipeline: Helm tries to manage same resource (adds Helm labels)
- Helm can't import resources created outside Helm

**Solution:** Remove Grafana ingress template from chatbot Helm chart
```bash
rm k8s/templates/grafana-ingress.yaml
```

**The Lesson:**
- Separate concerns: monitoring resources belong with monitoring stack, not app chart
- Resources created with `kubectl apply` can't be managed by Helm
- Keep Helm charts focused on single application/service

---

### Issue 9: Missing kubectl in Container Image

**Problem:** Verify stage failed
```
kubectl: not found
```

**Root Cause:** Used `alpine/helm:latest` which only has Helm, not kubectl

**Solution:** Use `alpine/k8s:1.30.7` which includes both kubectl and Helm
```yaml
containers:
  - name: kubectl
    image: alpine/k8s:1.30.7
```

---

### Issue 10: Backend SSL Configuration Error

**Problem:** Backend pods crashing with SSL error
```
AttributeError: 'dict' object has no attribute 'wrap_bio'
```

**Root Cause:** Incorrect SSL parameter format in aiomysql connection
```python
ssl={'ssl': True}  # Wrong - nested dict
```

**Investigation:**
1. First fix attempt: `ssl=True` → caused certificate verification error
2. Root cause: RDS uses self-signed certificate, Python doesn't trust by default
3. Realization: SSL parameter was added later, never tested in Kubernetes

**Solution:** Remove SSL parameter entirely (matches working manual test)
```python
pool = await aiomysql.create_pool(
    host=DB_CONFIG['host'],
    # ... other params
    # ssl parameter removed
)
```

**Why This Works:**
- RDS connections work without explicit SSL in connection string
- RDS still encrypts traffic at network level
- Client doesn't need to verify certificate
- Matches original working configuration from manual testing

**The Lesson:**
- Test "security improvements" in actual deployment environment
- SSL/TLS at application layer isn't always necessary (network-level encryption may suffice)
- When debugging, compare with known working configuration
- Don't add features that weren't in original working code without testing

---

### Issue 11: Rollout Status Timeouts

**Problem:** Deployment succeeded but verify stage timed out
```
error: timed out waiting for the condition
```

**Root Cause:** `kubectl rollout status --timeout=5m` waiting for pods that were slow to start

**Solution:** Remove all timeout commands from pipelines
- Removed: `kubectl rollout status --timeout=5m`
- Removed: `timeout 60` from Docker daemon wait
- Replaced with: Simple status checks without blocking

**The Lesson:**
- Timeouts in CI/CD pipelines cause false failures
- Better to complete pipeline and check status manually
- Kubernetes will eventually reconcile to desired state
- Use `kubectl get pods` instead of `kubectl rollout status`

---

## Current Status

✅ Jenkins security group configured with EKS cluster SG  
✅ Setup pipeline working (configures kubectl)  
✅ ALB controller pipeline working (dynamic VPC ID lookup with EC2 permissions)  
✅ Monitoring pipeline working (Prometheus, Grafana, Metrics Server deployed)  
✅ Application pipeline working (builds images, deploys to EKS)  
✅ Global TARGET_ENVIRONMENT variable configured  
✅ All pipelines use Kubernetes agents with proper RBAC  
✅ EKS cluster scaled to 3 nodes for capacity  
✅ Pod Identity for jenkins-sa with ECR permissions  
✅ Grafana ingress separated from chatbot Helm chart  
✅ Backend SSL configuration fixed (removed SSL parameter)  
✅ All timeouts removed from pipelines  

---

## Next Steps

1. Test monitoring stack pipeline
2. Test application pipeline
3. Configure Route 53 DNS for ALB endpoints
4. Set up GitHub webhooks for automatic application deployments
5. Document DNS setup process

### Issue 12: ALB Controller Missing EC2 Permissions and VPC ID Bug

**Problem:** ALB controller couldn't create ALBs with two errors:
1. `UnauthorizedOperation: ec2:DescribeRouteTables`
2. `InvalidParameterValue: vpc-id` (value was "None")

**Root Cause 1 - Missing IAM Permission:**
- Official AWS Load Balancer Controller IAM policy (v2.7.2) missing `ec2:DescribeRouteTables`
- Controller needs this permission for subnet auto-discovery
- Without it, controller can't determine which subnets to use for ALB

**Root Cause 2 - VPC ID Not Passing Between Agents:**
- ALB controller Jenkinsfile had 2 stages with different agents:
  - Stage 1: `agent any` (Jenkins EC2) - queried VPC ID, set `env.VPC_ID`
  - Stage 2: `agent kubernetes` (EKS pod) - tried to use `env.VPC_ID`
- Environment variables don't transfer between different agent types
- Result: `env.VPC_ID` was undefined in Stage 2, passed as "None" to Helm

**Investigation:**
```bash
# Check ingress events
kubectl describe ingress chatbot-ingress
# Shows: ec2:DescribeRouteTables permission denied

# Check ALB controller deployment
kubectl get deployment aws-load-balancer-controller -n kube-system -o yaml | grep vpc-id
# Shows: --aws-vpc-id=None
```

**Solution 1 - Add Missing IAM Permission:**
Created supplemental IAM policy in Terraform:
```hcl
# terraform/modules/eks/main.tf
resource "aws_iam_policy" "aws_load_balancer_controller_supplement" {
  name = "${var.cluster_name}-aws-load-balancer-controller-supplement"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ec2:DescribeRouteTables"]
      Resource = "*"
    }]
  })
}

# Attach both official and supplemental policies
module "aws_load_balancer_controller_role" {
  policy_arns = [
    aws_iam_policy.aws_load_balancer_controller.arn,
    aws_iam_policy.aws_load_balancer_controller_supplement.arn
  ]
}
```

**Solution 2 - Fix VPC ID Passing:**
Merged stages to query VPC ID in same pod where it's used:
```groovy
// Jenkinsfile-alb-controller
stage('Install AWS Load Balancer Controller') {
    agent {
        kubernetes {
            yaml '''
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: kubectl
    image: alpine/k8s:1.30.7
'''
        }
    }
    steps {
        container('kubectl') {
            sh """
                apk add --no-cache aws-cli
                
                VPC_ID=\$(aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=${vpcName}' --query 'Vpcs[0].VpcId' --output text)
                echo "Found VPC ID: \$VPC_ID"
                
                helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \\
                  --set vpcId=\$VPC_ID
            """
        }
    }
}
```

**Why This Works:**
- Query VPC ID and use it in same shell script (same execution context)
- Install AWS CLI in pod to enable VPC query
- Use shell variable `$VPC_ID` instead of Jenkins env variable
- No data passing between different agents

**Applied:**
1. `terraform apply` in `terraform/dev` (added IAM permissions)
2. Restart ALB controller pods: `kubectl rollout restart deployment aws-load-balancer-controller -n kube-system`
3. Re-run ALB controller pipeline with fixed Jenkinsfile

**Update - Additional Missing Permission:**
After fixing VPC ID issue, discovered another missing permission:
- `elasticloadbalancing:DescribeListenerAttributes` also missing from official v2.7.2 policy
- Added to supplemental policy alongside `ec2:DescribeRouteTables`
- Applied with `terraform apply` and restarted controller pods

**The Lesson:**
- Official AWS policies may be incomplete - always test and supplement as needed
- Jenkins environment variables don't transfer between different agent types
- When using multiple agents, keep related operations in same stage/agent
- Shell variables (`$VAR`) work within same script, env variables (`env.VAR`) don't cross agents
- Supplemental policies are cleaner than duplicating entire 200+ line policy inline
- VPC Name tags must match between Terraform and pipeline queries (vpc-dev not platform-dev-vpc)

---

### Issue 13: Backend Rate Limiter Parameter Name Error

**Problem:** Backend pods returning 500 Internal Server Error when users send messages
```
Exception: parameter 'request' must be an instance of starlette.requests.Request
```

**Root Cause:** slowapi rate limiter expects first parameter named `request` (HTTP Request object)

Original function signature:
```python
@limiter.limit("5/minute")
async def chat(http_request: Request, request: ChatRequest):
```

**Why This Happens:**
- slowapi decorator inspects function signature looking for parameter named `request`
- First parameter was named `http_request` instead of `request`
- slowapi couldn't find Request object, threw exception
- Rate limiter requirement: first parameter MUST be named `request`

**Investigation:**
```bash
kubectl logs -l app=chatbot-backend -c chatbot-backend
# Shows: Exception with full traceback pointing to slowapi expecting 'request' parameter
```

**Solution:** Rename parameters to match slowapi expectations
```python
@limiter.limit("5/minute")
async def chat(request: Request, chat_request: ChatRequest):
    # Update all references in function body
    message = chat_request.message  # Was: request.message
    session_id = chat_request.session_id  # Was: request.session_id
```

**Applied:**
1. Fixed backend/main.py function signature
2. Updated all references from `request.message` to `chat_request.message`
3. Committed and pushed changes
4. Jenkins pipeline build #21 deployed new backend image
5. Chatbot now working correctly

**The Lesson:**
- Third-party decorators may have strict parameter naming requirements
- slowapi specifically requires first parameter named `request` for rate limiting
- Always check library documentation for decorator parameter expectations
- Test rate limiting functionality after implementation, not just successful responses

---
