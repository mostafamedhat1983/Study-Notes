---
tags:
  - My_CV_Project
  - Kubernetes
  - argocd
---
# ArgoCD GitOps Implementation Guide

Complete guide to migrate from Jenkins-only deployment to Jenkins (CI) + ArgoCD (CD) GitOps workflow.

---

## Overview

**Current:** Jenkins builds images AND deploys to EKS  
**Target:** Jenkins builds images, ArgoCD deploys from Git  
**Benefit:** Git becomes single source of truth for deployments

**Difficulty:** Medium (1-2 hours)  
**Prerequisites:** Working Jenkins pipeline, EKS cluster

---

## Architecture

### Before (Jenkins Only)
```
Developer → Git Push → Jenkins → Build → Push ECR → Deploy EKS
                       └──────────── CI + CD ────────────┘
```

### After (Jenkins + ArgoCD)
```
Developer → Git Push → Jenkins → Build → Push ECR → Update Git
                       └────── CI ──────┘              ↓
                                                   ArgoCD watches
                                                       ↓
                                                  Deploy EKS
                                                  └── CD ──┘
```

---

## Phase 1: Install ArgoCD

### Step 1.1: Install ArgoCD in EKS

```bash
# Connect to Jenkins EC2
aws ssm start-session --target <jenkins-instance-id> --region us-east-2

# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### Step 1.2: Expose ArgoCD UI via ALB

Create `platform-ai-chatbot/k8s/argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:586794447516:certificate/1696ca3e-e5ac-477d-b872-e579133f660c
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/group.name: platform-shared-alb
spec:
  ingressClassName: alb
  rules:
  - host: argocd.mostafa-medhat.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

Apply ingress:
```bash
kubectl apply -f k8s/argocd-ingress.yaml
```

### Step 1.3: Get ArgoCD Admin Password

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Save this password - you'll need it to login
```

### Step 1.4: Configure DNS

Add CNAME record in Namecheap:
- **Host:** argocd
- **Value:** <ALB-DNS-from-kubectl-get-ingress>
- **TTL:** Automatic

### Step 1.5: Access ArgoCD UI

```bash
# Get ALB DNS
kubectl get ingress argocd-ingress -n argocd

# Open browser
https://argocd.mostafa-medhat.online

# Login:
# Username: admin
# Password: <from step 1.3>
```

---

## Phase 2: Configure Git Repository Access

### Step 2.1: Create GitHub Personal Access Token (PAT)

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. Name: `argocd-platform-ai-chatbot`
4. Scopes: ☑️ `repo` (all)
5. Generate token and save it

### Step 2.2: Add Repository to ArgoCD

**Via UI:**
1. ArgoCD UI → Settings → Repositories → Connect Repo
2. Choose connection method: HTTPS
3. Repository URL: `https://github.com/your-username/platform-ai-chatbot`
4. Username: `your-github-username`
5. Password: `<github-pat-from-step-2.1>`
6. Click Connect

**Via CLI (Alternative):**
```bash
# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login
argocd login argocd.mostafa-medhat.online --username admin --password <password>

# Add repository
argocd repo add https://github.com/your-username/platform-ai-chatbot \
  --username your-github-username \
  --password <github-pat>
```

---

## Phase 3: Create ArgoCD Applications

### Step 3.1: Create Application Manifest for Dev

Create `platform-ai-chatbot/argocd/chatbot-dev.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chatbot-dev
  namespace: argocd
spec:
  project: default
  
  # Source: Git repository
  source:
    repoURL: https://github.com/your-username/platform-ai-chatbot
    targetRevision: main
    path: k8s
    helm:
      valueFiles:
      - values-dev.yaml
  
  # Destination: EKS cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  
  # Sync policy: Automatic
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Auto-sync if cluster state drifts
    syncOptions:
    - CreateNamespace=false  # Namespace already exists
```

### Step 3.2: Create Application Manifest for Prod

Create `platform-ai-chatbot/argocd/chatbot-prod.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chatbot-prod
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/your-username/platform-ai-chatbot
    targetRevision: main  # Or use 'prod' branch for prod
    path: k8s
    helm:
      valueFiles:
      - values-prod.yaml
  
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=false
```

### Step 3.3: Deploy ArgoCD Applications

```bash
# Apply dev application
kubectl apply -f argocd/chatbot-dev.yaml

# Apply prod application (if using same cluster)
kubectl apply -f argocd/chatbot-prod.yaml

# Verify applications created
kubectl get applications -n argocd

# Check sync status
argocd app list
```

---

## Phase 4: Modify Jenkins Pipeline

### Step 4.1: Add Git Credentials to Jenkins

1. Jenkins → Manage Jenkins → Credentials → System → Global credentials
2. Add Credentials:
   - Kind: Username with password
   - Username: `your-github-username`
   - Password: `<github-pat-from-phase-2>`
   - ID: `github-credentials`
   - Description: GitHub PAT for manifest updates

### Step 4.2: Update Jenkinsfile

Replace `platform-ai-chatbot/jenkins/Jenkinsfile`:

```groovy
// Application CI Pipeline (Build Only - ArgoCD Handles Deployment)
// Builds Docker images, pushes to ECR, updates Git manifest for ArgoCD

pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
  - name: kubectl
    image: alpine/k8s:1.30.7
    command: ['cat']
    tty: true
"""
        }
    }
    
    environment {
        AWS_REGION = 'us-east-2'
        ECR_REGISTRY = '586794447516.dkr.ecr.us-east-2.amazonaws.com'
        ECR_REPOSITORY = 'platform-app'
        BACKEND_IMAGE_TAG = "${env.BUILD_NUMBER}-backend"
        FRONTEND_IMAGE_TAG = "${env.BUILD_NUMBER}-frontend"
        GIT_CREDENTIALS = credentials('github-credentials')
    }
    
    stages {
        stage('Build Images') {
            steps {
                container('docker') {
                    sh """
                        while ! docker info >/dev/null 2>&1; do sleep 1; done
                        cd backend && docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${BACKEND_IMAGE_TAG} .
                        cd ../frontend && docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${FRONTEND_IMAGE_TAG} .
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                container('docker') {
                    sh """
                        apk add --no-cache aws-cli
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${BACKEND_IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${FRONTEND_IMAGE_TAG}
                    """
                }
            }
        }
        
        stage('Update Git Manifest') {
            steps {
                container('kubectl') {
                    sh """
                        apk add --no-cache git sed
                        
                        # Configure Git
                        git config --global user.email "jenkins@platform.local"
                        git config --global user.name "Jenkins CI"
                        
                        # Update image tags in values file
                        sed -i "s|image:.*backend.*|image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${BACKEND_IMAGE_TAG}|" k8s/values-${TARGET_ENVIRONMENT}.yaml
                        sed -i "s|image:.*frontend.*|image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${FRONTEND_IMAGE_TAG}|" k8s/values-${TARGET_ENVIRONMENT}.yaml
                        
                        # Commit and push
                        git add k8s/values-${TARGET_ENVIRONMENT}.yaml
                        git commit -m "Deploy build ${BUILD_NUMBER} to ${TARGET_ENVIRONMENT} [skip ci]"
                        git push https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/your-username/platform-ai-chatbot.git HEAD:main
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ Build ${BUILD_NUMBER} completed. ArgoCD will deploy automatically."
            echo "Monitor deployment: https://argocd.mostafa-medhat.online"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
```

**Key Changes:**
- ❌ Removed `Deploy to EKS` stage
- ❌ Removed `Verify` stage
- ✅ Added `Update Git Manifest` stage
- ✅ Uses GitHub credentials to push manifest changes
- ✅ Adds `[skip ci]` to commit message (prevents infinite loop)

### Step 4.3: Update values-dev.yaml Format

Ensure `k8s/values-dev.yaml` has placeholder images that sed can replace:

```yaml
backend:
  image: 586794447516.dkr.ecr.us-east-2.amazonaws.com/platform-app:latest-backend
  # Jenkins will replace with: 586794447516.dkr.ecr.us-east-2.amazonaws.com/platform-app:42-backend

frontend:
  image: 586794447516.dkr.ecr.us-east-2.amazonaws.com/platform-app:latest-frontend
  # Jenkins will replace with: 586794447516.dkr.ecr.us-east-2.amazonaws.com/platform-app:42-frontend
```

---

## Phase 5: Test GitOps Workflow

### Step 5.1: Trigger Jenkins Build

```bash
# Make a code change
echo "# Test GitOps" >> backend/main.py

# Commit and push
git add backend/main.py
git commit -m "Test GitOps workflow"
git push origin main
```

### Step 5.2: Monitor Jenkins

1. Jenkins builds images (build #42)
2. Jenkins pushes to ECR
3. Jenkins updates `k8s/values-dev.yaml` with new image tags
4. Jenkins pushes commit to Git

### Step 5.3: Monitor ArgoCD

1. ArgoCD detects Git change
2. ArgoCD syncs cluster state
3. ArgoCD deploys new images

**Check ArgoCD UI:**
```
https://argocd.mostafa-medhat.online
→ Applications → chatbot-dev
→ Should show "Synced" and "Healthy"
```

**Check via CLI:**
```bash
argocd app get chatbot-dev
argocd app sync chatbot-dev  # Manual sync if needed
```

### Step 5.4: Verify Deployment

```bash
# Check pods updated
kubectl get pods -l app=chatbot-backend -o jsonpath='{.items[0].spec.containers[0].image}'

# Should show: 586794447516.dkr.ecr.us-east-2.amazonaws.com/platform-app:42-backend

# Check application
curl https://chatbot.mostafa-medhat.online
```

---

## Phase 6: Rollback Testing

### Scenario: New deployment breaks production

```bash
# Option 1: Git revert (recommended)
git log  # Find last working commit
git revert HEAD
git push origin main
# ArgoCD automatically deploys previous version

# Option 2: ArgoCD UI rollback
# ArgoCD UI → chatbot-dev → History → Select previous revision → Rollback

# Option 3: ArgoCD CLI rollback
argocd app rollback chatbot-dev <revision-number>
```

---

## Phase 7: Multi-Environment Strategy

### Option A: Single Branch, Multiple Values Files (Current)

```
main branch:
├── k8s/values-dev.yaml    # Dev config
└── k8s/values-prod.yaml   # Prod config

ArgoCD Applications:
- chatbot-dev → uses values-dev.yaml
- chatbot-prod → uses values-prod.yaml
```

**Pros:** Simple, single source of truth  
**Cons:** Dev and prod deploy simultaneously

### Option B: Multiple Branches (Recommended for Prod)

```
dev branch:
└── k8s/values-dev.yaml

prod branch:
└── k8s/values-prod.yaml

Workflow:
1. Merge main → dev (auto-deploy to dev)
2. Test in dev
3. Merge dev → prod (auto-deploy to prod)
```

**Pros:** Controlled promotion, test before prod  
**Cons:** More Git complexity

---

## Troubleshooting

### Issue 1: ArgoCD Can't Sync - "ComparisonError"

**Symptom:** ArgoCD shows "Unknown" status

**Cause:** Invalid Helm chart or values file

**Solution:**
```bash
# Test Helm chart locally
helm template chatbot ./k8s -f k8s/values-dev.yaml

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Issue 2: Jenkins Can't Push to Git

**Symptom:** "Authentication failed" in Jenkins

**Cause:** Invalid GitHub credentials

**Solution:**
```bash
# Verify credentials in Jenkins
# Manage Jenkins → Credentials → github-credentials → Update

# Test Git push manually
git push https://username:token@github.com/repo.git
```

### Issue 3: ArgoCD Not Detecting Changes

**Symptom:** Git updated but ArgoCD doesn't sync

**Cause:** Sync interval (default 3 minutes)

**Solution:**
```bash
# Manual sync
argocd app sync chatbot-dev

# Or reduce sync interval
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"30s"}}'
```

### Issue 4: Infinite Loop - Jenkins Triggers Itself

**Symptom:** Jenkins builds continuously

**Cause:** Jenkins webhook triggers on its own commits

**Solution:**
Add `[skip ci]` to commit message (already in Jenkinsfile):
```bash
git commit -m "Deploy build 42 [skip ci]"
```

---

## Monitoring & Observability

### ArgoCD Metrics

```bash
# Application health
argocd app get chatbot-dev

# Sync history
argocd app history chatbot-dev

# Application logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

### Deployment Notifications (Optional)

Configure ArgoCD to send Slack/email notifications:

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: Application {{.app.metadata.name}} deployed to {{.app.spec.destination.namespace}}
```

---

## Cleanup (If Reverting to Jenkins-Only)

```bash
# Delete ArgoCD applications
kubectl delete application chatbot-dev -n argocd
kubectl delete application chatbot-prod -n argocd

# Uninstall ArgoCD
kubectl delete namespace argocd

# Revert Jenkinsfile to original (with Deploy stage)
git checkout <commit-before-argocd> jenkins/Jenkinsfile
git commit -m "Revert to Jenkins-only deployment"
git push
```

---

## Cost Impact

**ArgoCD:** Zero additional cost (runs as EKS pods)  
**ALB:** Shared with existing chatbot ALB (no extra cost)  
**Git operations:** Free (GitHub)

---

## Security Considerations

### GitHub PAT Security

- ✅ Use fine-grained PAT (not classic) for better security
- ✅ Limit scope to single repository
- ✅ Set expiration (rotate annually)
- ✅ Store in Jenkins credentials (encrypted)

### ArgoCD Access Control

```bash
# Change admin password
argocd account update-password

# Create read-only user for developers
argocd account create developer --password <password>
argocd proj role add default developer --policy "p, proj:default:developer, applications, get, default/*, allow"
```

### Git Commit Signing (Optional)

```bash
# Configure GPG signing for Jenkins commits
git config --global user.signingkey <gpg-key-id>
git config --global commit.gpgsign true
```

---

## Next Steps

1. **Test in Dev:** Run full workflow in dev environment
2. **Monitor for 1 Week:** Ensure stability before prod
3. **Enable Prod:** Apply same setup to prod environment
4. **Add Notifications:** Configure Slack/email alerts
5. **Document Runbooks:** Create rollback procedures for team

---

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://opengitops.dev/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

---

**Last Updated:** 2025-01-XX
