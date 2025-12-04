---
tags:
  - My_CV_Project
  - Jenkins
---
# Jenkins Deployment and DNS Setup Guide

Complete guide for deploying applications via Jenkins pipelines and configuring Route 53 DNS for ALB access.

---

## Prerequisites

- ✅ Infrastructure deployed (VPC, EKS, RDS, Jenkins EC2)
- ✅ Jenkins configured with Kubernetes plugin
- ✅ kubectl access to EKS cluster from Jenkins
- ✅ ACM certificate validated for your domain
- ✅ Route 53 hosted zone for your domain

---

## Deployment Order

Execute Jenkins jobs in this specific order:

```
1. ALB Controller Setup (once per environment)
   ↓
2. Monitoring Stack (once per environment)
   ↓
3. Application Deployment (automated on commits)
   ↓
4. Route 53 DNS Configuration
```

---

## Step 1: Deploy AWS Load Balancer Controller

**Purpose:** Enables automatic ALB provisioning for Kubernetes Ingress resources

**Jenkins Job Setup:**
1. Create new Pipeline job: `alb-controller-setup`
2. Configure:
   - Pipeline script from SCM
   - Repository: `platform-ai-chatbot`
   - Script Path: `jenkins/Jenkinsfile-alb-controller`
3. Save

**Execution:**
```
Run job → Select environment (dev/prod) → Build
```

**Verification:**
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Expected Output:**
```
NAME                           READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller   2/2     Running   0          2m
```

---

## Step 2: Deploy Monitoring Stack

**Purpose:** Installs Metrics Server (for HPA), Prometheus, and Grafana with ingress

**Jenkins Job Setup:**
1. Create new Pipeline job: `monitoring-stack`
2. Configure:
   - Pipeline script from SCM
   - Repository: `platform-ai-chatbot`
   - Script Path: `jenkins/Jenkinsfile-monitoring`
3. Save

**Execution:**
```
Run job → Select environment (dev/prod) → Build
```

**Verification:**
```bash
# Check Metrics Server
kubectl top nodes

# Check Prometheus/Grafana
kubectl get pods -n monitoring

# Check Grafana ingress
kubectl get ingress grafana-ingress -n monitoring
```

**Expected Output:**
```
NAME              CLASS   HOSTS                          ADDRESS                                    PORTS
grafana-ingress   alb     grafana.mostafa-medhat.online  k8s-monitori-grafanai-xxx.us-east-2.elb... 80, 443
```

---

## Step 3: Deploy Application

**Purpose:** Builds Docker images, pushes to ECR, deploys chatbot to EKS

**Jenkins Job Setup:**
1. Create new Pipeline job: `chatbot-deployment`
2. Configure:
   - Pipeline script from SCM
   - Repository: `platform-ai-chatbot`
   - Script Path: `jenkins/Jenkinsfile`
   - Build Triggers: Poll SCM or GitHub webhook
3. Update environment variables in Jenkinsfile:
   ```groovy
   ECR_REGISTRY = '<your-account-id>.dkr.ecr.us-east-2.amazonaws.com'
   ```
4. Save

**Execution:**
```
Run job → Build (or automatic on Git push)
```

**Verification:**
```bash
# Check backend deployment
kubectl get pods -l app=chatbot-backend
kubectl logs -l app=chatbot-backend -c chatbot-backend

# Check frontend deployment
kubectl get pods -l app=chatbot-frontend

# Check chatbot ingress
kubectl get ingress chatbot-ingress
```

**Expected Output:**
```
NAME              CLASS   HOSTS                          ADDRESS                                    PORTS
chatbot-ingress   alb     chatbot.mostafa-medhat.online  k8s-default-chatboti-xxx.us-east-2.elb...  80, 443
```

---

## Step 4: Get ALB DNS Names

**After all deployments complete, retrieve ALB DNS names:**

```bash
# Get all ingresses
kubectl get ingress --all-namespaces

# Get chatbot ALB DNS
kubectl get ingress chatbot-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Get Grafana ALB DNS
kubectl get ingress grafana-ingress -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Example Output:**
```
k8s-default-chatboti-a1b2c3d4-123456789.us-east-2.elb.amazonaws.com
k8s-monitori-grafanai-e5f6g7h8-987654321.us-east-2.elb.amazonaws.com
```

**Note:** In dev environment, both ingresses may share the same ALB DNS (grouped by `platform-shared-alb`).

---

## Step 5: Configure Route 53 DNS

### Option A: AWS Console (Manual)

**For Chatbot:**
1. Go to Route 53 → Hosted zones → Select your domain
2. Click "Create record"
3. Configure:
   - Record name: `chatbot`
   - Record type: `A - Routes traffic to an IPv4 address and some AWS resources`
   - Toggle: ☑️ Alias
   - Route traffic to: `Alias to Application and Classic Load Balancer`
   - Region: `us-east-2`
   - Load balancer: Select chatbot ALB from dropdown
4. Click "Create records"

**For Grafana:**
1. Click "Create record"
2. Configure:
   - Record name: `grafana`
   - Record type: `A - Routes traffic to an IPv4 address and some AWS resources`
   - Toggle: ☑️ Alias
   - Route traffic to: `Alias to Application and Classic Load Balancer`
   - Region: `us-east-2`
   - Load balancer: Select Grafana ALB from dropdown
3. Click "Create records"

---

### Option B: AWS CLI (Automated)

**Get Hosted Zone ID:**
```bash
aws route53 list-hosted-zones-by-name --dns-name mostafa-medhat.online --query 'HostedZones[0].Id' --output text
```

**Get ALB Hosted Zone ID (always same for us-east-2):**
```bash
# us-east-2 ALB hosted zone ID
Z3AADJGX6KTTL2
```

**Create Chatbot DNS Record:**
```bash
CHATBOT_ALB_DNS=$(kubectl get ingress chatbot-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
HOSTED_ZONE_ID="<your-route53-hosted-zone-id>"

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "chatbot.mostafa-medhat.online",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z3AADJGX6KTTL2",
          "DNSName": "'$CHATBOT_ALB_DNS'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

**Create Grafana DNS Record:**
```bash
GRAFANA_ALB_DNS=$(kubectl get ingress grafana-ingress -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "grafana.mostafa-medhat.online",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z3AADJGX6KTTL2",
          "DNSName": "'$GRAFANA_ALB_DNS'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

---

## Step 6: Verify DNS Resolution

**Wait 1-2 minutes for DNS propagation, then test:**

```bash
# Check DNS resolution
nslookup chatbot.mostafa-medhat.online
nslookup grafana.mostafa-medhat.online

# Test HTTPS access
curl -I https://chatbot.mostafa-medhat.online
curl -I https://grafana.mostafa-medhat.online
```

**Expected Output:**
```
HTTP/2 200
server: awselb/2.0
```

---

## Step 7: Access Applications

**Chatbot:**
- URL: https://chatbot.mostafa-medhat.online
- No authentication required

**Grafana:**
- URL: https://grafana.mostafa-medhat.online
- Username: `admin`
- Password: `admin`

---

## Environment Differences

### Development
- **ALB Strategy:** 1 shared ALB for both chatbot and Grafana
- **Cost:** ~$18/month for single ALB
- **DNS Records:** Both point to same ALB DNS name
- **Replicas:** Backend 2, Frontend 2

### Production
- **ALB Strategy:** 2 separate ALBs (chatbot + Grafana)
- **Cost:** ~$36/month for two ALBs
- **DNS Records:** Each points to different ALB DNS name
- **Replicas:** Backend 3, Frontend 3

---

## Troubleshooting

### ALB Not Created
```bash
# Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check ingress events
kubectl describe ingress chatbot-ingress
kubectl describe ingress grafana-ingress -n monitoring
```

**Common Issues:**
- IAM permissions missing (check Pod Identity association)
- Certificate ARN invalid (verify in ACM)
- Subnets not tagged for ALB (check VPC subnet tags)

### DNS Not Resolving
```bash
# Check Route 53 record
aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Name=='chatbot.mostafa-medhat.online.']"

# Check ALB status
aws elbv2 describe-load-balancers --query "LoadBalancers[?DNSName=='<alb-dns-name>'].State"
```

**Common Issues:**
- DNS propagation delay (wait 5-10 minutes)
- Wrong hosted zone ID used
- ALB not in healthy state

### Application Not Accessible
```bash
# Check target health
aws elbv2 describe-target-health --target-group-arn <target-group-arn>

# Check pod status
kubectl get pods -l app=chatbot-backend
kubectl get pods -l app=chatbot-frontend

# Check service endpoints
kubectl get endpoints chatbot-backend-service
kubectl get endpoints chatbot-frontend-service
```

**Common Issues:**
- Pods not ready (check health probes)
- Security groups blocking traffic
- Target group unhealthy (check pod logs)

---

## Cleanup

**To remove all resources:**

```bash
# Delete applications
helm uninstall chatbot
helm uninstall grafana-ingress

# Delete monitoring stack
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

# Delete ALB controller
helm uninstall aws-load-balancer-controller -n kube-system

# Delete Route 53 records (manual or CLI)
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch '{"Changes":[{"Action":"DELETE",...}]}'
```

---

## Summary

**One-Time Setup (per environment):**
1. ✅ Deploy ALB Controller
2. ✅ Deploy Monitoring Stack
3. ✅ Configure Route 53 DNS

**Ongoing Operations:**
- Application deployments via Jenkins (automated)
- Monitor via Grafana dashboard
- Scale via HPA (automatic based on CPU)

**Total Setup Time:** ~15-20 minutes per environment
