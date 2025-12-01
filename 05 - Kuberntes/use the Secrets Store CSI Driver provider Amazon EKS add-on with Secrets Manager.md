---
tags:
  - AWS
  - Kubernetes
---


### Phase 1: Infrastructure & Identity
1.  **Create Cluster**: Provision an EKS cluster using `eksctl` and a variable for the cluster name.
    ```
    CLUSTER_NAME="my-test-cluster”
    eksctl create cluster $CLUSTER_NAME
    ```
2.  **Install Add-ons**: Install the required EKS add-ons.
    -   Install **AWS Secrets Store CSI Driver Provider**:
        ```
        eksctl create addon \
          --cluster $CLUSTER_NAME \
          --name aws-secrets-store-csi-driver-provider
        ```
    -   Install **EKS Pod Identity Agent** (replaces legacy IRSA):
        ```
        eksctl create addon \
        --cluster $CLUSTER_NAME \
        --name eks-pod-identity-agent
        ```
3.  **Create IAM Role**: Create an IAM role with a trust policy for the EKS Pod Identity service principal (`pods.eks.amazonaws.com`).
    ```
    ROLE_ARN=$(aws --region <region> --query Role.Arn --output text iam create-role --role-name nginx-deployment-role --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "pods.eks.amazonaws.com"
                },
                "Action": [
                    "sts:AssumeRole",
                    "sts:TagSession"
                ]
            }
        ]
    }')
    ```
4.  **Assign Permissions**: Attach the managed policy (or a scoped-down custom policy) to the role.
    ```
    aws iam attach-role-policy \
      --role-name nginx-deployment-role \
      --policy-arn arn:aws:iam::aws:policy/AWSSecretsManagerClientReadOnlyAccess
    ```
5.  **Bind Identity**: Link the IAM role to the Kubernetes Service Account.
    ```
    eksctl create podidentityassociation \
    --cluster $CLUSTER_NAME \
    --namespace default \
    --region <region> \
    --service-account-name nginx-pod-identity-deployment-sa \
    --role-arn $ROLE_ARN \
    --create-service-account true
    ```

### Phase 2: Secret Definition
6.  **Create AWS Secret**: Create a test secret in AWS Secrets Manager.
    ```
    aws secretsmanager create-secret \
      --name addon_secret \
      --secret-string "super secret!"
    ```
7.  **Define SecretProviderClass**: Create `spc.yaml` to map the AWS Secret to the driver.
    ```
    apiVersion: secrets-store.csi.x-k8s.io/v1
    kind: SecretProviderClass
    metadata:
    name: nginx-pod-identity-deployment-aws-secrets
    spec:
    provider: aws
    parameters:
    objects: |
    - objectName: "addon_secret"
      objectType: "secretsmanager"
      usePodIdentity: "true"
    ```
    Deploy it:
    ```
    kubectl apply -f spc.yaml
    ```

### Phase 3: Workload Deployment
8.  **Deploy Pod**: Apply a deployment manifest that uses the Service Account and mounts the secret.
    ```
    kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/examples/ExampleDeployment-PodIdentity.yaml
    ```
9.  **Verification**: Exec into the pod to verify the secret file exists.
    ```
    kubectl exec -it $(kubectl get pods | awk '/nginx-pod-identity-deployment/{print $1}' | head -1) -- cat /mnt/secrets-store/addon_secret
    ```
    *Expected Output:* `super secret!`

---

## Source of Truth (Official Doc Links)

- **Official Blog Post**: [attached_file:1]
- **AWS Secrets Store CSI Driver Provider Docs**: [GitHub - aws/secrets-store-csi-driver-provider-aws](https://github.com/aws/secrets-store-csi-driver-provider-aws) [web:16]

---

## Architectural Reasoning

**Why use this over Bitnami Sealed Secrets?**
Unlike Sealed Secrets, which stores encrypted static blobs in Git, this approach fetches secrets **dynamically** at runtime.
1.  **Centralization**: Secrets remain in AWS Secrets Manager, allowing for centralized rotation, auditing (CloudTrail), and revocation. [web:10][web:11]
2.  **No Secrets in Git**: Even encrypted secrets are removed from the repo, satisfying stricter compliance requirements (e.g., PCI-DSS, FedRAMP) that forbid any credential artifacts in source control. [web:11]
3.  **Identity Federation**: By using EKS Pod Identity, the pod trades a short-lived K8s token for AWS credentials without hardcoding Access Keys. [web:11][web:16]

**The Cost of Complexity:**
You are trading the simplicity of `kubeseal` (a binary and a CRD) for a chain of dependencies: *CSI Driver DaemonSet + AWS Provider DaemonSet + IAM Role + Pod Identity Agent + SecretProviderClass + Volume Mounts*. If any link in this chain breaks (e.g., Pod Identity agent fails), your pods will fail to start with `ContainerCreating` errors. [web:17]

---

## Security & Best Practice Check

I need to challenge the "File Mount" pattern recommended in this tutorial.

**1. File System vs. Memory (Tmpfs)**
The blog explicitly notes: *"Security best practice recommends caching secrets in memory where possible."* [attached_file:1]
By default, the CSI driver mounts secrets as **files** on the container's filesystem. While these are usually tmpfs (RAM-backed) volumes, they are technically visible to any process with root access to the file system namespace.
*   **Challenge**: If you have high-security requirements, consider using the **AWS Secrets and Configuration Provider (ASCP)** or native SDKs to fetch secrets directly into application memory, bypassing the filesystem entirely.

**2. Synchronization with Kubernetes Secrets**
The steps above mounts the secret as a file. If your application expects an environment variable (`envFrom: secretRef`), this CSI driver setup requires an extra configuration (`syncSecret`) to sync the CSI volume into a native Kubernetes Secret object. This creates a duplicate copy of the secret in etcd, potentially negating the security benefit of keeping it external. [web:15]
*   **Verification**: Ensure your etcd is encrypted at rest if you enable the sync feature.

**3. Least Privilege**
The tutorial uses the managed policy `AWSSecretsManagerClientReadOnlyAccess`.
*   **Critique**: This grants the pod permission to read **ALL** secrets in the account. In a production environment, you **must** write a custom inline IAM policy that allows `secretsmanager:GetSecretValue` on *only* the specific ARNs that the application needs. Never use the wildcard managed policy for production workloads. [web:10][attached_file:1]
