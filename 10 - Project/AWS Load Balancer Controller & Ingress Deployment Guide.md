
## Overview

  

This guide explains how to deploy the AWS Load Balancer Controller on EKS and create an Ingress resource to expose your application with HTTPS using the ACM certificate.

  

---

  

## Prerequisites

  

- ✅ EKS cluster running (created via Terraform)

- ✅ kubectl configured to access the cluster

- ✅ Helm installed

- ✅ ACM certificate issued (from `01-acm-certificate-setup.md`)

- ✅ Certificate ARN saved

  

---

  

## Architecture Overview

  

```

Internet

    ↓

Application Load Balancer (ALB)

    ↓ (HTTPS terminated here with ACM certificate)

Ingress Resource (Kubernetes)

    ↓

Service (ClusterIP)

    ↓

Pods (Frontend/Backend)

```

  

**Traffic Flow:**

1. User accesses `https://chatbot.mostafa-medhat.online`

2. DNS resolves to ALB public IP

3. ALB terminates HTTPS using ACM certificate

4. ALB forwards HTTP traffic to Kubernetes Service

5. Service routes to healthy pods

  

---

  

## Part 1: Understanding AWS Load Balancer Controller

  

### What is AWS Load Balancer Controller?

  

A Kubernetes controller that:

- Watches for Ingress resources

- Automatically provisions AWS Application Load Balancers (ALB)

- Configures ALB to route traffic to Kubernetes services

- Manages ALB lifecycle (create, update, delete)

  

**Why we need it:**

- EKS doesn't create ALBs automatically

- Native Kubernetes Ingress controller doesn't know about AWS

- AWS Load Balancer Controller bridges Kubernetes and AWS

  

---

  

### Components:

  

1. **IAM Policy** - Permissions for controller to manage ALBs

2. **IAM Role** - Assumed by controller pods via Pod Identity

3. **Controller Deployment** - Kubernetes pods running the controller

4. **Ingress Resource** - Your configuration (which service, which certificate, etc.)

  

---

  

## Part 2: Add AWS Load Balancer Controller to Terraform

  

### Why Terraform Instead of Manual kubectl?

  

✅ **Repeatable** - Can recreate in any environment  

✅ **Version controlled** - Changes tracked in Git  

✅ **Consistent** - Same setup across dev/prod  

✅ **Documented** - Infrastructure as code  

  

---

  

### Step 1: Add IAM Policy for Load Balancer Controller

  

The controller needs permissions to create/manage ALBs.

  

**Location:** `terraform-aws-eks/terraform/modules/eks/main.tf`

  

**Add after EBS CSI Driver section:**

  

```hcl

# ========================================

# AWS Load Balancer Controller

# ========================================

# Enables automatic ALB provisioning for Kubernetes Ingress resources

# Authentication: Pod Identity (modern approach, no OIDC needed)

  

# Download official IAM policy from AWS

data "http" "aws_load_balancer_controller_policy" {

  url = "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json"

}

  

# Create IAM policy for Load Balancer Controller

resource "aws_iam_policy" "aws_load_balancer_controller" {

  name        = "${var.cluster_name}-aws-load-balancer-controller"

  description = "IAM policy for AWS Load Balancer Controller"

  policy      = data.http.aws_load_balancer_controller_policy.response_body

}

  

# IAM role for Load Balancer Controller using Pod Identity

module "aws_load_balancer_controller_role" {

  source = "../role"

  name    = "${var.cluster_name}-aws-load-balancer-controller"

  service = "pods.eks.amazonaws.com"

  policy_arns = [

    aws_iam_policy.aws_load_balancer_controller.arn

  ]

}

  

# Link IAM role to Load Balancer Controller service account

resource "aws_eks_pod_identity_association" "aws_load_balancer_controller" {

  cluster_name    = aws_eks_cluster.this.name

  namespace       = "kube-system"

  service_account = "aws-load-balancer-controller"

  role_arn        = module.aws_load_balancer_controller_role.role_arn

  

  depends_on = [

    aws_eks_addon.pod_identity_agent,

    module.aws_load_balancer_controller_role

  ]

}

```

  

---

  

### Step 2: Install AWS Load Balancer Controller via Helm

  

**Option A: Add to Terraform (Recommended)**

  

Use Terraform Helm provider to install the controller:

  

```hcl

# Install AWS Load Balancer Controller via Helm

resource "helm_release" "aws_load_balancer_controller" {

  name       = "aws-load-balancer-controller"

  repository = "https://aws.github.io/eks-charts"

  chart      = "aws-load-balancer-controller"

  namespace  = "kube-system"

  version    = "1.7.2"

  

  set {

    name  = "clusterName"

    value = aws_eks_cluster.this.name

  }

  

  set {

    name  = "serviceAccount.create"

    value = "true"

  }

  

  set {

    name  = "serviceAccount.name"

    value = "aws-load-balancer-controller"

  }

  

  depends_on = [

    aws_eks_pod_identity_association.aws_load_balancer_controller

  ]

}

```

  

**Option B: Manual Installation (For Testing)**

  

```bash

# Add Helm repo

helm repo add eks https://aws.github.io/eks-charts

helm repo update

  

# Install controller

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \

  -n kube-system \

  --set clusterName=platform-dev \

  --set serviceAccount.create=true \

  --set serviceAccount.name=aws-load-balancer-controller

```

  

---

  

### Step 3: Apply Terraform Changes

  

```bash

cd terraform-aws-eks/terraform/dev  # or prod

  

terraform plan   # Review changes

terraform apply  # Deploy controller

```

  

**What gets created:**

- IAM policy for ALB management

- IAM role for controller

- Pod Identity association

- AWS Load Balancer Controller deployment in kube-system namespace

  

---

  

### Step 4: Verify Controller Installation

  

```bash

# Check controller pods are running

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

  

# Expected output:

NAME                                            READY   STATUS    RESTARTS   AGE

aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

```

  

**If pods are not running:**

```bash

# Check logs

kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

  

# Check events

kubectl get events -n kube-system --sort-by='.lastTimestamp'

```

  

---

  

## Part 3: Create Ingress Resource

  

### Step 1: Create IngressClass

  

**Location:** `platform-ai-chatbot/k8s/templates/ingress-class.yaml`

  

```yaml

apiVersion: networking.k8s.io/v1

kind: IngressClass

metadata:

  name: alb

  annotations:

    ingressclass.kubernetes.io/is-default-class: "true"

spec:

  controller: ingress.k8s.aws/alb

```

  

**What this does:**

- Defines "alb" as the Ingress controller type

- Sets it as default (all Ingress resources use this unless specified)

- Tells Kubernetes to use AWS Load Balancer Controller

  

---

  

### Step 2: Create Ingress Resource

  

**Location:** `platform-ai-chatbot/k8s/templates/ingress.yaml`

  

```yaml

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: chatbot-ingress

  annotations:

    # ALB Configuration

    alb.ingress.kubernetes.io/scheme: internet-facing

    alb.ingress.kubernetes.io/target-type: ip

    alb.ingress.kubernetes.io/healthcheck-path: /health

    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"

    # HTTPS Configuration

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'

    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # ACM Certificate (REPLACE WITH YOUR CERTIFICATE ARN)

    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:123456789012:certificate/your-certificate-id

    # Tagging

    alb.ingress.kubernetes.io/tags: Environment=dev,Project=chatbot

spec:

  ingressClassName: alb

  rules:

    - host: chatbot.mostafa-medhat.online

      http:

        paths:

          - path: /

            pathType: Prefix

            backend:

              service:

                name: chatbot-frontend-service

                port:

                  number: 8501

```

  

---

  

### Key Annotations Explained:

  

| Annotation | Value | Purpose |

|------------|-------|---------|

| `scheme` | `internet-facing` | ALB has public IP (accessible from internet) |

| `target-type` | `ip` | Route traffic directly to pod IPs (not node ports) |

| `listen-ports` | `HTTP: 80, HTTPS: 443` | ALB listens on both ports |

| `ssl-redirect` | `443` | Redirect HTTP to HTTPS automatically |

| `certificate-arn` | Your ACM ARN | Use this certificate for HTTPS |

| `healthcheck-path` | `/health` | Check pod health at this endpoint |

  

---

  

### Step 3: Parameterize Certificate ARN in Helm

  

**Update `platform-ai-chatbot/k8s/values.yaml`:**

  

```yaml

ingress:

  enabled: true

  host: chatbot.mostafa-medhat.online

  certificateArn: ""  # Override in values-dev.yaml or values-prod.yaml

```

  

**Update `platform-ai-chatbot/k8s/values-dev.yaml`:**

  

```yaml

ingress:

  certificateArn: "arn:aws:acm:us-east-2:123456789012:certificate/your-cert-id"

```

  

**Update Ingress template to use value:**

  

```yaml

metadata:

  annotations:

    alb.ingress.kubernetes.io/certificate-arn: {{ .Values.ingress.certificateArn }}

```

  

---

  

### Step 4: Deploy Ingress

  

```bash

cd platform-ai-chatbot/k8s

  

# Deploy with Helm

helm upgrade --install chatbot . -f values-dev.yaml

  

# Verify Ingress created

kubectl get ingress

  

# Check ALB provisioning status

kubectl describe ingress chatbot-ingress

```

  

---

  

### Step 5: Get ALB DNS Name

  

```bash

# Get ALB address

kubectl get ingress chatbot-ingress

  

# Output:

NAME               CLASS   HOSTS                            ADDRESS                                    PORTS   AGE

chatbot-ingress    alb     chatbot.mostafa-medhat.online    k8s-default-chatbot-1234567890-1234567890.us-east-2.elb.amazonaws.com   80, 443   5m

```

  

**Copy the ADDRESS** - this is your ALB DNS name.

  

---

  

## Part 4: Point Domain to ALB

  

### Step 1: Add CNAME Record in Namecheap

  

1. Log in to **Namecheap**

2. Go to **Domain List** → **mostafa-medhat.online** → **Manage**

3. Click **"Advanced DNS"** tab

4. Click **"Add New Record"**

  

**Record details:**

- **Type:** CNAME Record

- **Host:** `chatbot`

- **Value:** `k8s-default-chatbot-1234567890-1234567890.us-east-2.elb.amazonaws.com` (your ALB DNS name)

- **TTL:** Automatic

  

5. Click **green checkmark** to save

  

---

  

### Step 2: Wait for DNS Propagation

  

**Timeline:**

- 1-5 minutes: DNS updates propagate

- Test with: `nslookup chatbot.mostafa-medhat.online`

  

**Expected result:**

```

chatbot.mostafa-medhat.online

CNAME: k8s-default-chatbot-1234567890.us-east-2.elb.amazonaws.com

Address: 18.x.x.x (ALB IP)

```

  

---

  

### Step 3: Test HTTPS Access

  

```bash

# Test HTTP (should redirect to HTTPS)

curl -I http://chatbot.mostafa-medhat.online

  

# Expected: 301 Redirect to https://

  

# Test HTTPS

curl https://chatbot.mostafa-medhat.online

  

# Should return your frontend HTML

```

  

**Open in browser:**

```

https://chatbot.mostafa-medhat.online

```

  

✅ Should show your chatbot with **valid HTTPS** (green padlock)

  

---

  

## Part 5: Troubleshooting

  

### ALB Not Created

  

**Check Ingress events:**

```bash

kubectl describe ingress chatbot-ingress

```

  

**Common issues:**

- Controller not running: `kubectl get pods -n kube-system`

- IAM permissions missing: Check controller logs

- Subnet tagging: ALB needs subnets tagged with `kubernetes.io/role/elb=1`

  

---

  

### Certificate Not Working

  

**Check certificate ARN:**

```bash

kubectl get ingress chatbot-ingress -o yaml | grep certificate-arn

```

  

**Verify:**

- ARN is correct (copy from ACM)

- Certificate is in same region as ALB (us-east-2)

- Certificate status is "Issued" (not "Pending validation")

  

---

  

### 502 Bad Gateway Error

  

**Causes:**

- Backend pods not healthy

- Wrong service name in Ingress

- Health check path incorrect

  

**Debug:**

```bash

# Check pods

kubectl get pods

  

# Check service

kubectl get svc chatbot-frontend-service

  

# Check health endpoint

kubectl exec -it <pod-name> -- curl localhost:8501/health

```

  

---

  

### DNS Not Resolving

  

**Check CNAME record:**

```bash

nslookup chatbot.mostafa-medhat.online

```

  

**If not working:**

- Wait 5-10 more minutes for DNS propagation

- Verify CNAME record in Namecheap (Host: `chatbot`, not `chatbot.mostafa-medhat.online`)

- Try with Google DNS: `nslookup chatbot.mostafa-medhat.online 8.8.8.8`

  

---

  

## Key Concepts Explained

  

### Why Use Ingress Instead of LoadBalancer Service?

  

**LoadBalancer Service:**

- Creates one ALB per service

- Can't do path-based routing

- Can't do SSL termination

- Expensive (multiple ALBs)

  

**Ingress:**

- One ALB for multiple services

- Path-based routing (`/api`, `/frontend`)

- Centralized SSL/TLS

- Cost-effective

  

---

  

### Pod Identity vs IRSA

  

**Old way (IRSA):**

- Create OIDC provider

- Create IAM role with trust policy

- Annotate service account

- Complex setup

  

**New way (Pod Identity):**

- Create IAM role

- Create Pod Identity association

- Done!

- Simpler, modern

  

---

  

### ALB Target Types: IP vs Instance

  

**IP mode (recommended for EKS):**

- Traffic goes directly to pod IPs

- Faster (no extra hop)

- Better for containers

  

**Instance mode:**

- Traffic goes to node, then pod

- Extra network hop

- Better for traditional VMs

  

---

  

## Cost Breakdown

  

| Resource | Cost |

|----------|------|

| **ALB** | ~$16-22/month (fixed) |

| **Data transfer** | $0.008/GB (out to internet) |

| **LCU hours** | $0.008/hour per LCU |

| **AWS Load Balancer Controller** | FREE (runs on your nodes) |

| **ACM Certificate** | FREE |

  

**Estimated total: $20-30/month** for low-traffic chatbot

  

---

  

## Security Best Practices

  

1. ✅ **Always use HTTPS** (ACM certificate)

2. ✅ **Redirect HTTP to HTTPS** (ssl-redirect annotation)

3. ✅ **Use IP target type** (direct pod routing)

4. ✅ **Configure health checks** (detect unhealthy pods)

5. ✅ **Tag resources** (for cost tracking and organization)

6. ✅ **Use Pod Identity** (no static credentials)

  

---

  

## Monitoring & Maintenance

  

### Check ALB Health

  

```bash

# AWS Console → EC2 → Load Balancers

# Find your ALB (k8s-default-chatbot-*)

# Check: Listeners, Rules, Target Groups, Monitoring

```

  

### Check Target Health

  

```bash

# In ALB console

# Target Groups → Select your target group

# Targets tab → Check health status

  

# Should show: Healthy, Initial, Unhealthy

```

  

### View ALB Logs (Optional)

  

Enable access logs to S3:

```yaml

alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=my-alb-logs

```

  

---

  

## Cleanup

  

### To Delete ALB:

  

```bash

# Delete Ingress resource

kubectl delete ingress chatbot-ingress

  

# ALB will be automatically deleted by controller

```

  

### To Uninstall Controller:

  

```bash

# If installed via Helm

helm uninstall aws-load-balancer-controller -n kube-system

  

# Or via Terraform

terraform destroy

```

  

---

  

## Reference Architecture

  

### Complete Traffic Flow:

  

```

User Browser

    ↓

DNS Query: chatbot.mostafa-medhat.online

    ↓

Namecheap DNS: Returns ALB DNS

    ↓

HTTPS Request to ALB (port 443)

    ↓

ALB: Terminates SSL with ACM certificate

    ↓

ALB: Checks target health

    ↓

ALB: Forwards HTTP to Pod IP (port 8501)

    ↓

Kubernetes Service: Routes to healthy pod

    ↓

Pod: Processes request, returns response

    ↓

ALB: Encrypts response with ACM certificate

    ↓

User Browser: Receives HTTPS response

```

  

---

  

## Summary Checklist

  

**Infrastructure Setup:**

- [ ] IAM policy created for Load Balancer Controller

- [ ] IAM role created with Pod Identity

- [ ] Pod Identity association created

- [ ] AWS Load Balancer Controller installed (Helm or Terraform)

- [ ] Controller pods running in kube-system namespace

  

**Ingress Configuration:**

- [ ] IngressClass created (alb)

- [ ] Ingress resource created with certificate ARN

- [ ] Ingress points to correct service

- [ ] Health check path configured

  

**DNS Configuration:**

- [ ] CNAME record added in Namecheap

- [ ] DNS propagation verified (nslookup)

- [ ] HTTPS accessible in browser

- [ ] HTTP redirects to HTTPS

  

**Verification:**

- [ ] ALB created in AWS Console

- [ ] Target group shows healthy targets

- [ ] Certificate shows in ALB listeners

- [ ] Application accessible via https://chatbot.mostafa-medhat.online

  

---

  

## Next Steps

  

After successful deployment:

1. ✅ Monitor ALB metrics in CloudWatch

2. ✅ Set up CloudWatch alarms for ALB errors

3. ✅ Configure WAF for DDoS protection (optional)

4. ✅ Enable ALB access logs for troubleshooting

5. ✅ Add multiple services to same ALB (path-based routing)

  

---

  

## Additional Resources

  

**Official Documentation:**

- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/

- Ingress Annotations: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/

- ACM User Guide: https://docs.aws.amazon.com/acm/

  

**Troubleshooting:**

- Controller logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`

- Ingress events: `kubectl describe ingress chatbot-ingress`

- ALB Target Health: AWS Console → EC2 → Target Groups

  

---

  

**Document Version:** 1.0  

**Last Updated:** 2025-12-02  

**Project:** AI Chatbot Platform  

**Domain:** chatbot.mostafa-medhat.online