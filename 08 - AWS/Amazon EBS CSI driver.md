---
tags:
  - AWS
  - Kubernetes
  - EKS
---
# Amazon EBS CSI Driver

The Amazon EBS CSI driver is a Kubernetes storage plugin that lets workloads use Amazon EBS block volumes through PersistentVolumes (PVs), PersistentVolumeClaims (PVCs), and StorageClasses.

It is the CSI-based replacement for the legacy in-tree AWS EBS volume plugin.

## Typical lifecycle

1. A workload creates a PVC that references a StorageClass.
2. The CSI controller provisions an EBS volume when dynamic provisioning is requested.
3. Kubernetes schedules a Pod and coordinates attachment to a compatible node.
4. The CSI node plugin stages and mounts the device for the Pod.
5. When the claim is released, the PV reclaim policy determines whether the underlying volume is retained or deleted.

## EKS installation and provisioner names

For a standard EKS cluster using the Amazon EBS CSI driver add-on, the add-on name is `aws-ebs-csi-driver` and the StorageClass provisioner is normally:

```yaml
provisioner: ebs.csi.aws.com
```

EKS Auto Mode has a separate managed block-storage capability and uses:

```yaml
provisioner: ebs.csi.eks.amazonaws.com
```

These provisioners manage their own volumes separately. Do not mix the provisioner names or assume the standard EBS CSI add-on must be installed on EKS Auto Mode clusters.

## IAM permissions

The EBS CSI driver requires AWS IAM permissions for its AWS API operations. Follow the current EKS installation documentation for the required managed policy and identity mechanism (IRSA or EKS Pod Identity, where supported). AWS documents newer policy variants for more scoped access; choose the policy appropriate to the cluster and driver version.

The controller performs AWS volume lifecycle operations. The node plugin performs node-side operations such as staging, publishing, and mounting. Do not assume the node DaemonSet requires the same volume-provisioning permissions as the controller.

## Storage characteristics

- EBS is block storage attached to EC2 instances; it is not a general-purpose shared RWX filesystem.
- Access and attachment constraints depend on the EBS volume type and deployment configuration.
- Use a shared filesystem service such as Amazon EFS with its CSI driver when multiple nodes need shared filesystem access.
- Use encryption and appropriate KMS key policies according to the workload's security requirements.
- Understand the PV reclaim policy before deleting a PVC.

## Official references

- https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html
