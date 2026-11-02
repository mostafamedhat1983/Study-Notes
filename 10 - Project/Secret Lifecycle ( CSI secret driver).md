---
tags:
  - My_CV_Project
---
# Historical CSI Secret Lifecycle Design (Not the Current Deployment)

> **Status:** This note describes an earlier CSI-based design explored for the project. The project later moved to an init-container approach. Treat this as historical architecture, not as proof that the CSI design was deployed or is the current implementation. For the current implementation, use `10 - Project/secret-lifecycle-init-container.md`.

## Complete Secret Lifecycle - From Creation to Pod

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

   Result: the Terraform configuration requests EKS envelope encryption for Kubernetes Secrets with this KMS key. Verify the actual cluster configuration and KMS key policy; do not infer that all cluster secrets were encrypted merely from this design snippet.

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

   Correction: the workload Pod's ServiceAccount must be associated with the IAM role used by ASCP to retrieve the external secret. Do not assign the role to the CSI provider's ServiceAccount based on this example.

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
         usePodIdentity: "true"
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

   Result: the workload ServiceAccount exists. With EKS Pod Identity or IRSA, this workload ServiceAccount is the identity associated with the IAM role that authorizes ASCP to retrieve the secret. The CSI driver/provider does not remove the need to authorize the workload identity.

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
   •  The application Pod uses its configured Kubernetes ServiceAccount.
   •  That ServiceAccount is associated with an IAM role through EKS Pod Identity (or configured through IRSA).
   •  ASCP uses the workload identity to call Secrets Manager, subject to the IAM policy.

   **4d. CSI Driver Fetches Secret from AWS**

     ASCP provider → AWS Secrets Manager API
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
   •  The sync feature can create Kubernetes Secret: chatbot-backend-db-credentials
   •  Secret data: {username: admin, password: YourPassword, dbname: platformdb}
   •  Stored as a Kubernetes Secret; encryption at rest depends on the EKS/Kubernetes encryption configuration actually enabled for the cluster
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
   2. A synchronized Kubernetes Secret is not guaranteed to remain indefinitely; the CSI driver's sync lifecycle deletes it when no Pods mount the corresponding SecretProviderClass. Verify the actual behavior and cluster encryption configuration.
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
     2. ASCP authenticates using the workload Pod's ServiceAccount identity (Pod Identity or IRSA)
        ↓
     3. CSI Driver fetches secret via AWS API (TLS)
        ↓
     4. CSI Driver creates tmpfs volume with files
        ↓
     5. Optional sync creates a Kubernetes Secret (encryption at rest depends on cluster configuration)
        ↓
     6. Pod mounts tmpfs volume at /mnt/secrets
        ↓
     7. Pod gets env vars from K8s Secret (if configured)
        ↓
     8. Application reads credentials

   Key Security Features:
   •  ✅ Secrets never stored in plaintext on disk
   •  ✅ tmpfs is memory-only
   •  ✅ Kubernetes Secret encryption at rest only when the cluster's encryption-at-rest configuration is enabled and verified
   •  ✅ IAM-based access control
   •  ✅ TLS for all network communication
   •  ✅ Automatic rotation possible (CSI driver can refresh)

   This was a proposed/experimental project design, not a statement that every component was deployed successfully. The actual project implementation and security properties must be verified against the committed Terraform and Kubernetes manifests.
