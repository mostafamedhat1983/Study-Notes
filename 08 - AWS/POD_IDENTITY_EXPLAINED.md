---
tags:
  - AWS
  - EKS
  - Kubernetes
---
# Pod Identity Flow - Simple Step-by-Step Explanation

## Scenario 1: App Pod Requests Secret from AWS Secrets Manager

### The Flow:

```
Step 1: Pod makes request
┌─────────────────────┐
│  Your App Pod       │  "I need secret: platform-db-dev-credentials"
│  (ServiceAccount:   │  
│   my-app-sa)        │───┐
└─────────────────────┘   │
                          │
                          ↓
Step 2: Pod Identity Agent intercepts
┌─────────────────────────────────────┐
│  Pod Identity Agent (on node)       │
│  "Wait, which IAM role does         │
│   my-app-sa have?"                  │
└─────────────────────────────────────┘
           │
           ↓
Step 3: Check Pod Identity Association
┌──────────────────────────────────────┐
│  Pod Identity Association            │
│  ServiceAccount: my-app-sa           │
│  → IAM Role: my-app-secrets-role     │
└──────────────────────────────────────┘
           │
           ↓
Step 4: Get temporary credentials
┌─────────────────────────────────────┐
│  AWS STS                             │
│  "Give me temp credentials for      │
│   my-app-secrets-role"               │
│                                      │
│  Returns: {                          │
│    AccessKeyId: "ASIAXXX..."         │
│    SecretAccessKey: "xxx..."         │
│    SessionToken: "xxx..."            │
│    Expiration: "1 hour"              │
│  }                                   │
└─────────────────────────────────────┘
           │
           ↓
Step 5: Make authenticated call
┌─────────────────────────────────────┐
│  AWS Secrets Manager                 │
│  Request with temp credentials       │
│  → Check IAM policy                  │
│  → Allow/Deny based on role          │
│  → Return secret value               │
└─────────────────────────────────────┘
           │
           ↓
Step 6: Secret returned to pod
┌─────────────────────┐
│  Your App Pod       │  Receives: {"username":"admin","password":"..."}
│  Can now use the    │
│  secret!            │
└─────────────────────┘
```

---

## Scenario 2: EBS CSI Driver Creates Volume

### The Flow:

```
Step 1: Pod requests persistent storage
┌─────────────────────┐
│  Your App Pod       │  apiVersion: v1
│  with PVC           │  kind: PersistentVolumeClaim
│                     │  storageClassName: gp3
│                     │  storage: 10Gi
└─────────────────────┘
           │
           ↓
Step 2: Kubernetes calls EBS CSI Driver
┌─────────────────────────────────────┐
│  EBS CSI Controller Pod             │
│  (ServiceAccount: ebs-csi-sa)       │
│  "Need to create 10Gi EBS volume"  │
└─────────────────────────────────────┘
           │
           ↓
Step 3: CSI Driver needs to call AWS EC2 API
┌─────────────────────────────────────┐
│  EBS CSI Driver makes AWS API call: │
│  ec2.CreateVolume(size=10Gi)        │
└─────────────────────────────────────┘
           │
           ↓
Step 4: Pod Identity Agent intercepts
┌─────────────────────────────────────┐
│  Pod Identity Agent                  │
│  "Which role for ebs-csi-sa?"       │
└─────────────────────────────────────┘
           │
           ↓
Step 5: Check Pod Identity Association
┌──────────────────────────────────────┐
│  Pod Identity Association            │
│  ServiceAccount: ebs-csi-sa          │
│  → IAM Role: ebs-csi-driver-role     │
│  (has AmazonEBSCSIDriverPolicy)      │
└──────────────────────────────────────┘
           │
           ↓
Step 6: Get temp credentials from STS
┌─────────────────────────────────────┐
│  AWS STS returns temp creds for     │
│  ebs-csi-driver-role                 │
└─────────────────────────────────────┘
           │
           ↓
Step 7: Create EBS volume
┌─────────────────────────────────────┐
│  AWS EC2 API                         │
│  CreateVolume() with temp creds      │
│  → Creates: vol-0abc123...           │
│  → Size: 10Gi, Type: gp3             │
└─────────────────────────────────────┘
           │
           ↓
Step 8: Attach volume to node
┌─────────────────────────────────────┐
│  EBS CSI Driver                      │
│  - Attaches vol-0abc123 to node     │
│  - Mounts to pod at /mnt/data       │
└─────────────────────────────────────┘
           │
           ↓
Step 9: Pod gets persistent storage
┌─────────────────────┐
│  Your App Pod       │  Can now read/write to /mnt/data
│  Volume mounted at  │  Data persists even if pod restarts!
│  /mnt/data          │
└─────────────────────┘
```

---

## Key Differences:

| Aspect | Secrets Store CSI | EBS CSI |
|--------|-------------------|---------|
| **What pod requests** | Secret from Secrets Manager | Persistent disk storage |
| **AWS service called** | Secrets Manager | EC2 (EBS) |
| **IAM permissions needed** | `secretsmanager:GetSecretValue` | `ec2:CreateVolume`, `ec2:AttachVolume` |
| **Result** | Secret mounted as file in pod | Disk volume attached to pod |
| **Example use case** | Database password | Database data files |

---

## The Common Pattern (Both use same Pod Identity flow):

1. ✅ **Pod** (with ServiceAccount) makes request
2. ✅ **Pod Identity Agent** (on node) intercepts
3. ✅ **Pod Identity Association** maps ServiceAccount → IAM Role
4. ✅ **AWS STS** provides temporary credentials (valid for ~1 hour)
5. ✅ **AWS Service** (Secrets Manager or EC2) validates and responds
6. ✅ **Pod** gets what it requested

---

## What is Pod Identity Agent?

**Pod Identity Agent** is a **DaemonSet** that runs on **every EKS worker node** (not inside your application pods).

### Architecture:

```
┌─────────────────────────────────────────────────────────┐
│  EKS Worker Node                                        │
│                                                         │
│  ┌──────────────────────────────────┐                  │
│  │  Pod Identity Agent (DaemonSet)  │◄─────────┐       │
│  │  - Runs on every node            │          │       │
│  │  - Intercepts AWS API calls      │          │       │
│  │  - Provides STS credentials      │          │       │
│  └──────────────────────────────────┘          │       │
│                  ↑                              │       │
│                  │ (intercepts)                 │       │
│                  │                              │       │
│  ┌───────────────────────────┐                 │       │
│  │  Your App Pod             │                 │       │
│  │  (e.g., CSI Driver Pod)   │                 │       │
│  │                           │                 │       │
│  │  Makes AWS API call ────────────────────────┘       │
│  │  (to Secrets Manager)     │                         │
│  │                           │                         │
│  │  Gets temp STS token ◄────── from Pod Identity      │
│  └───────────────────────────┘      Agent              │
│                                                         │
└─────────────────────────────────────────────────────────┘
                      │
                      └──→ AWS Secrets Manager
                           (authenticated with STS token)
```

### Key Points:

1. **Pod Identity Agent = Node-level DaemonSet** (one per worker node)
2. **Your pods don't have it inside them** - they just make normal AWS API calls
3. **Pod Identity Agent intercepts** those calls via the node
4. **Agent gets temp STS credentials** from AWS based on the **Pod Identity Association** (which maps: ServiceAccount → IAM Role)
5. **Agent injects credentials** into the pod's AWS SDK call

---

## Complete Infrastructure View:

```
┌─────────────────────────────────────────────────────────┐
│  EKS Cluster                                            │
│                                                         │
│  Pod Identity Agent DaemonSet (runs on all nodes)      │
│                                                         │
│  ┌─────────────────────┐                               │
│  │ CSI Driver Pod      │  uses ServiceAccount          │
│  │ (has SA mapped      │──────────────────┐            │
│  │  to IAM role)       │                  │            │
│  └─────────────────────┘                  ↓            │
│           │                    ┌──────────────────┐    │
│           │                    │ Pod Identity     │    │
│           └───→ AWS API call ─→│ Association      │    │
│                                │ (SA → IAM Role)  │    │
│                                └──────────────────┘    │
│                                        │                │
└────────────────────────────────────────┼────────────────┘
                                         ↓
                                  AWS Secrets Manager
                              (authenticated via temp STS)
```

---

## Benefits of Pod Identity:

1. ✅ **No credentials in pods** - Everything is temporary and automatic
2. ✅ **Fine-grained permissions** - Each ServiceAccount can have different IAM roles
3. ✅ **Automatic rotation** - STS credentials expire after ~1 hour and auto-refresh
4. ✅ **Audit trail** - CloudTrail shows which pod (via role) accessed what
5. ✅ **No OIDC complexity** - Simpler than the old IRSA method
6. ✅ **Zero trust** - Credentials never leave AWS infrastructure

---

## Real-World Example in Your Infrastructure:

### EBS CSI Driver Configuration:

```hcl
# Terraform creates the IAM role
module "ebs_csi_driver_role" {
  source = "../role"
  name   = "platform-dev-ebs-csi-driver"
  service = "pods.eks.amazonaws.com"  # Pod Identity
  policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  ]
}

# Links ServiceAccount to IAM Role
resource "aws_eks_pod_identity_association" "ebs_csi_driver" {
  cluster_name    = "platform-dev"
  namespace       = "kube-system"
  service_account = "ebs-csi-controller-sa"
  role_arn        = module.ebs_csi_driver_role.role_arn
}
```

**Result**: Any pod using `ebs-csi-controller-sa` ServiceAccount automatically gets permissions to create/attach EBS volumes!

---

## Troubleshooting:

### How to verify Pod Identity is working:

```bash
# 1. Check Pod Identity Agent is running
kubectl get daemonset -n kube-system | grep pod-identity

# 2. Check Pod Identity Associations
aws eks list-pod-identity-associations --cluster-name platform-dev

# 3. Check if pod has correct ServiceAccount
kubectl get pod <pod-name> -n <namespace> -o yaml | grep serviceAccountName

# 4. Test from inside a pod
kubectl exec -it <pod-name> -- env | grep AWS
# Should NOT see AWS_ACCESS_KEY_ID hardcoded
# Credentials are injected at runtime by Pod Identity Agent
```

---

## Summary - The Magic:

**Your pod never handles AWS credentials directly.**

The Pod Identity Agent:
- Lives on the node (not in your pod)
- Watches for AWS API calls from pods
- Checks which ServiceAccount the pod uses
- Looks up the IAM role via Pod Identity Association
- Gets temporary credentials from AWS STS
- Injects those credentials into the API call
- Your pod gets the response it needs!

**All of this happens transparently - your application code doesn't change!**

🎉 **That's the beauty of Pod Identity!** 🎉
