---
tags:
  - AWS
  - EKS
  - Kubernetes
---
# AWS EBS CSI Driver & EKS Pod Identity

## 1. The "Why" (Problem Statement)
Kubernetes natively doesn't know how to talk to AWS services like EBS. It needs a translator.
- **Old Way (In-Tree):** The code to talk to AWS was built directly inside the Kubernetes source code. It was bloated, hard to update, and required a full K8s upgrade just to patch a storage bug.
- **New Way (CSI):** The **Container Storage Interface (CSI)** is a standardized plugin system. The **EBS CSI Driver** is the specific plugin that teaches Kubernetes how to "drive" Amazon EBS (Create, Delete, Attach, Mount volumes).

## 2. How It Works (Architecture)
The driver consists of two main components that work together:

### A. The Controller (The "Manager")
- **Runs as:** A Deployment (usually 1-2 replicas).
- **Location:** Runs anywhere in the cluster (Control Plane logic).
- **Job:** Talks to the AWS API to **provision** (create) and **delete** volumes.
- **Analogy:** The call center operator who orders the pizza (volume) from the kitchen (AWS).

### B. The Node (The "Worker")
- **Runs as:** A DaemonSet (one pod on EVERY worker node).
- **Location:** Runs on every node that needs to mount storage.
- **Job:** Performs the system-level Linux commands to **attach** the EBS device to the EC2 instance and **mount** it to the Pod.
- **Analogy:** The delivery driver who physically hands the pizza to the customer (Pod).

## 3. The Relation to EKS Pod Identity
This is strictly about **Permissions**. The CSI Driver needs permission to talk to AWS (e.g., `ec2:CreateVolume`, `ec2:AttachVolume`).

### The Problem (IAM Roles)
Your EKS Worker Nodes have an IAM role (Node Role). You *could* give this Node Role full EBS permissions, but that violates the **Principle of Least Privilege** (every pod on that node would technically inherit those rights).

### The Solution (IRSA / Pod Identity)
Instead of giving permissions to the *Node*, we give them only to the *CSI Driver Pods*.
1. **IRSA (IAM Roles for Service Accounts):** The classic way. You create an IAM Role with the `AmazonEBSCSIDriverPolicy` and associate it with the `ebs-csi-controller-sa` Service Account in Kubernetes.
2. **EKS Pod Identity (The New Standard):** The modern, simpler way.
    - You install the **EKS Pod Identity Agent** addon.
    - You create an **Association** in the EKS console: "Map the IAM Role `EBS-CSI-Role` to the Service Account `ebs-csi-controller-sa` in namespace `kube-system`."
    - The agent automatically injects temporary AWS credentials into the CSI Driver pods.

**Key Takeaway:** EKS Pod Identity is the *delivery mechanism* that securely hands the "Keys to the AWS Kingdom" (IAM credentials) specifically to the EBS CSI Driver, allowing it to create volumes without giving your worker nodes excessive power.

---
## 4. Quick Reference Table

| Component | K8s Object | Responsibility | Needs IAM Permissions? |
| :--- | :--- | :--- | :--- |
| **CSI Controller** | Deployment | Talks to AWS API (Create/Delete Vol) | **YES** (Heavy usage) |
| **CSI Node** | DaemonSet | Linux Mount/Unmount commands | **YES** (Attach/Detach usage) |

## 5. Setup Snippet (Terraform Context)
*For the EKS Pod Identity approach:*

