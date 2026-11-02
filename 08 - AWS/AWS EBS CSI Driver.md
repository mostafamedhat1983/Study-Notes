---
tags:
  - AWS
  - EKS
  - Kubernetes
---
# Amazon EBS CSI Driver and EKS Pod Identity

## 1. Why the CSI driver is needed

Kubernetes uses the Container Storage Interface (CSI) to integrate with storage systems. The Amazon EBS CSI driver lets Kubernetes provision and attach Amazon EBS volumes and mount them into Pods.

The CSI driver replaces the legacy in-tree Kubernetes AWS EBS volume plugin. New EKS configurations should use the EBS CSI driver.

## 2. Main driver components

### Controller

The controller component is deployed as a Kubernetes Deployment. It handles control-plane storage operations and communicates with AWS APIs for tasks such as creating, deleting, attaching, and detaching EBS volumes as required by the CSI workflow.

### Node plugin

The node component runs as a DaemonSet on nodes where it is needed. It performs node-side operations such as staging, publishing, mounting, unmounting, and cleanup of the block device for Pods.

The controller and node plugin have different responsibilities. Do not assume the node plugin needs the same EC2 provisioning permissions as the controller.

## 3. IAM permissions

The controller needs AWS permissions to manage EBS resources. AWS documents the managed policy `AmazonEBSCSIDriverPolicy` for the driver. Use the current AWS installation instructions for the supported driver version and identity method.

Avoid giving broad EBS permissions to the worker-node role when a workload-specific identity is supported and appropriate.

## 4. IRSA and EKS Pod Identity

The EBS CSI controller can obtain AWS permissions through:

- **IRSA:** IAM Roles for Service Accounts, using the cluster's OIDC provider and a Kubernetes ServiceAccount.
- **EKS Pod Identity:** an association maps the cluster, namespace, and ServiceAccount to an IAM role. The Pod Identity Agent provides temporary credentials to supported SDKs through the container credential provider.

For Pod Identity, ensure the association matches the actual ServiceAccount used by the EBS CSI controller. The agent does not intercept EC2 API calls; the driver's AWS SDK obtains credentials from the agent and then calls EC2 directly.

## 5. Example Pod Identity association

The following is illustrative; verify the actual cluster name, namespace, ServiceAccount, and role ARN in your installation:

```bash
aws eks create-pod-identity-association \
  --cluster-name <cluster-name> \
  --namespace kube-system \
  --service-account ebs-csi-controller-sa \
  --role-arn arn:aws:iam::<account-id>:role/<ebs-csi-role>
```

The IAM role trust policy must allow the EKS Pod Identity service principal and the required STS actions. Follow the current AWS EKS Pod Identity documentation for the trust policy and condition keys.

## 6. Quick reference

| Component | Kubernetes object | Primary responsibility | AWS provisioning permissions |
|---|---|---|---|
| EBS CSI controller | Deployment | Provisioning and control-plane volume operations | Yes, as required by the driver |
| EBS CSI node plugin | DaemonSet | Node-side device staging, publishing, and mounting | Do not assume it needs the controller's provisioning permissions |

## 7. Troubleshooting

```bash
kubectl get pods -n kube-system
kubectl get deployment -n kube-system
kubectl get daemonset -n kube-system
kubectl describe pvc <claim-name>
kubectl describe pod <pod-name> -n <namespace>
```

Check the PVC and Pod Events first. Then inspect controller and node-plugin logs, the StorageClass provisioner, IAM permissions, and the selected identity mechanism.

## Interview takeaway

The EBS CSI controller coordinates volume lifecycle operations and calls AWS APIs. The node plugin performs node-level mount operations. IAM permissions should be assigned to the correct driver component and ServiceAccount using IRSA or EKS Pod Identity according to the official AWS installation guidance.
