---
tags:
  - My_CV_Project
---
 Complete Secret Lifecycle - From Creation to Pod

   Let me trace the entire journey of your database credentials:

   ──────────────────────────────────────────

   **Step 1: Manual Secret Creation in AWS Secrets Manager**

   Location: AWS Console or AWS CLI
   Action: You create the secret

   bash
     aws secretsmanager create-secret \
       --name platform-db-dev-credentials \
       --secret-string '{"username":"admin","password":"YourPassword","dbname":"platformdb"}' \
       --region us-east-2

   Result: Secret stored in AWS Secrets Manager, encrypted by AWS-managed keys

   ──────────────────────────────────────────

   **Step 2: Infrastructure Setup (Terraform)**

   Location: my-project-Infra/terraform/modules/eks/main.tf

   2a. KMS Key Creation

   hcl
     resource "aws_kms_key" "this" {
       description = "EKS cluster encryption key"
       enable_key_rotation = true
     }

   Result: Customer-managed KMS key created for encrypting Kubernetes Secrets

   2b. EKS Cluster with Encryption

   hcl
     encryption_config {
       resources = ["secrets"]
       provider {
         key_arn = aws_kms_key.this.arn
       }
     }

   Result: EKS cluster configured to encrypt all Kubernetes Secrets with KMS key

   2c. Secrets Store CSI Driver IAM Policy

   hcl
     resource "aws_iam_policy" "secrets_store_csi" {
       # Allows access to platform-*-dev-* secrets
     }

   Result: IAM policy allowing CSI driver to read Secrets Manager

   2d. Secrets Store CSI Driver Installed

   hcl
     resource "aws_eks_addon" "secrets_store_csi_driver"

   Result: CSI Driver running in kube-system namespace

   2e. CSI Driver IAM Role & Pod Identity

   hcl
     resource "aws_eks_pod_identity_association" "secrets_store_csi" {
       service_account = "secrets-store-csi-driver-provider-aws"
       role_arn = module.secrets_store_csi_role.role_arn
     }

   Result: CSI driver pods can assume IAM role to access Secrets Manager

   ──────────────────────────────────────────

   **Step 3: Application Deployment (Helm/Kubernetes)**

   Location: platform-ai-chatbot/k8s/templates/

   3a. SecretProviderClass Created

   yaml
     # chatbot-backend-secret-provider-class.yaml
     apiVersion: secrets-store.csi.x-k8s.io/v1
     kind: SecretProviderClass
     metadata:
       name: chatbot-backend-secrets
     spec:
       provider: aws
       parameters:
         objects: |
           - objectName: "platform-db-dev-credentials"
             jmesPath:
               - path: "username"
                 objectAlias: "db_user"

   Result: Configuration telling CSI driver WHAT to fetch and HOW to map it

   3b. Service Account Created

   yaml
     # chatbot-backend-service-account
     apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: chatbot-backend-service-account
       namespace: default

   Result: Service account exists (but doesn't need IAM access for secret reading - CSI driver handles that)

   3c. Deployment References SecretProviderClass

   yaml
     # chatbot-backend-deployment.yaml
     spec:
       serviceAccountName: chatbot-backend-service-account
       volumes:
         - name: secrets-store
           csi:
             driver: secrets-store.csi.k8s.io
             volumeAttributes:
               secretProviderClass: "chatbot-backend-secrets"
       containers:
         - volumeMounts:
             - name: secrets-store
               mountPath: "/mnt/secrets"

   Result: Pod definition tells Kubernetes to mount secrets using CSI driver

   ──────────────────────────────────────────

   **Step 4: Pod Startup - The Magic Happens! 🪄**

   When the pod starts, here's the sequence:

   **4a. Kubelet Requests Volume Mount**
   •  Kubelet sees the CSI volume definition
   •  Contacts Secrets Store CSI Driver: "I need secrets from chatbot-backend-secrets"

   **4b. CSI Driver Reads SecretProviderClass**
   •  CSI Driver reads the SecretProviderClass
   •  Sees it needs: platform-db-dev-credentials from AWS Secrets Manager
   •  Sees jmesPath mappings: username → db_user, etc.

   **4c. CSI Driver Authenticates to AWS**
   •  CSI Driver pod is running in kube-system namespace
   •  Has service account: secrets-store-csi-driver-provider-aws
   •  Pod Identity gives it IAM role with secretsmanager:GetSecretValue permission

   **4d. CSI Driver Fetches Secret from AWS**

     CSI Driver → AWS Secrets Manager API
     Request: GetSecretValue for "platform-db-dev-credentials"
     Response: {"username":"admin","password":"YourPassword","dbname":"platformdb"}

   **4e. CSI Driver Processes Secret**
   •  Parses JSON response
   •  Applies jmesPath mappings:
     •  username → file: db_user (content: "admin")
     •  password → file: db_password (content: "YourPassword")
     •  dbname → file: db_name (content: "platformdb")

   **4f. CSI Driver Creates tmpfs Volume**
   •  Creates an in-memory tmpfs volume (not written to disk!)
   •  Writes files to tmpfs:
     •  /mnt/secrets/db_user
     •  /mnt/secrets/db_password
     •  /mnt/secrets/db_name

   **4g. (Optional) CSI Driver Creates Kubernetes Secret**

   Because secretObjects is defined in SecretProviderClass:
   •  CSI Driver creates Kubernetes Secret: chatbot-backend-db-credentials
   •  Secret data: {username: admin, password: YourPassword, dbname: platformdb}
   •  Stored in etcd, encrypted with KMS key automatically
   •  Can now be referenced as environment variables

   **4h. Volume Mounted to Pod**
   •  Kubelet mounts the tmpfs volume to pod at /mnt/secrets
   •  Pod container starts
   •  Application can now read files OR use env vars from K8s Secret

   ──────────────────────────────────────────

   **Step 5: Application Reads Credentials**

   Option A: Read from Files

   python
     DB_USER = open('/mnt/secrets/db_user').read().strip()
     DB_PASSWORD = open('/mnt/secrets/db_password').read().strip()

   Option B: Read from Environment Variables (if using secretObjects)

   yaml
     env:
     - name: DB_USER
       valueFrom:
         secretKeyRef:
           name: chatbot-backend-db-credentials
           key: username

   Backend code already has: DB_USER = os.getenv('DB_USER')

   ──────────────────────────────────────────

   **Step 6: Pod Deletion - Cleanup**

   When pod is deleted:
   1. tmpfs volume is unmounted and destroyed (secrets removed from memory)
   2. Kubernetes Secret remains in etcd (still encrypted with KMS)
   3. AWS Secrets Manager secret remains unchanged

   ──────────────────────────────────────────

   Security at Each Layer:

   Layer                        │ Encryption                │ Access Control                
   -----------------------------+---------------------------+-------------------------------
   **AWS Secrets Manager**      │ AWS-managed encryption    │ IAM policy (CSI driver role)
   **In-Transit (AWS → Pod)**   │ TLS                       │ IAM authentication
   **tmpfs Volume in Pod**      │ Memory-only (not on disk) │ Linux file permissions
   **Kubernetes Secret (etcd)** │ KMS encryption            │ RBAC (who can read K8s Secret)
   **Environment Variables**    │ No encryption (in memory) │ Pod isolation

   ──────────────────────────────────────────

   Summary Flow Diagram:

     1. AWS Secrets Manager (encrypted at rest)
        ↓
     2. CSI Driver authenticates via Pod Identity
        ↓
     3. CSI Driver fetches secret via AWS API (TLS)
        ↓
     4. CSI Driver creates tmpfs volume with files
        ↓
     5. CSI Driver creates K8s Secret (encrypted with KMS in etcd)
        ↓
     6. Pod mounts tmpfs volume at /mnt/secrets
        ↓
     7. Pod gets env vars from K8s Secret (if configured)
        ↓
     8. Application reads credentials

   Key Security Features:
   •  ✅ Secrets never stored in plaintext on disk
   •  ✅ tmpfs is memory-only
   •  ✅ KMS encryption for K8s Secrets
   •  ✅ IAM-based access control
   •  ✅ TLS for all network communication
   •  ✅ Automatic rotation possible (CSI driver can refresh)

   This is enterprise-grade secrets management! 🔒
