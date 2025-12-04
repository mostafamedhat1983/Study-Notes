---
tags:
  - AWS
  - Kubernetes
  - EKS
---
The Amazon EBS CSI driver is a Kubernetes storage plugin that lets your cluster dynamically create, attach, mount, and delete Amazon EBS volumes for pods using the standard PersistentVolume (PV) and PersistentVolumeClaim (PVC) APIs. It is the supported, out‑of‑tree implementation that replaces the older in‑tree AWS EBS volume plugin and is the recommended way to use EBS as persistent block storage with EKS or any Kubernetes cluster running on AWS.
## Architectural Reasoning (What it does and why it exists)

- Role in Kubernetes:
  - Implements the Container Storage Interface (CSI) for Amazon EBS.
  - When a PVC using an EBS-backed StorageClass is created, the driver:
    - Provisions an EBS volume with the requested size/type.
    - Attaches it to the right worker node.
    - Mounts it into the pod at the requested mount path.
  - When the PVC/PV is deleted (with `reclaimPolicy: Delete`), it can automatically delete the underlying EBS volume.

- Why use it instead of the old in-tree plugin:
  - The Kubernetes project is deprecating and removing in-tree cloud volume plugins; CSI is the standard.
  - CSI drivers can be developed and released independently of Kubernetes versions, so AWS can add features/fixes without waiting for a Kubernetes release.
  - EKS best practices and AWS guidance assume you are using the EBS CSI driver for block storage.

Typical flow in EKS:
- Install/enable the EBS CSI driver as an add‑on.
- Define a `StorageClass` with `provisioner: ebs.csi.aws.com` (or EKS-specific variant).
- Create PVCs referencing that StorageClass.
- Pods mount those PVCs and get EBS-backed persistent storage.

## Security & Best Practice Check

- Always run the EBS CSI driver with a dedicated IAM role (IRSA or Pod Identity) scoped to the minimum EBS actions required (create, attach, delete, etc.).
- Encrypt EBS volumes by default (KMS-managed keys), and ensure tagging policies are in place for cost and security visibility.
- Use EBS CSI for:
  - Single-pod or single-node stateful workloads (e.g., databases, queues) that need low-latency block storage and ReadWriteOnce semantics.
- Combine with EFS/other CSI drivers when you need ReadWriteMany or cross-node shared storage; do not try to force EBS into RWX patterns.

In short: the EBS CSI driver is the Kubernetes-native bridge between your pods and Amazon EBS volumes, enabling dynamic, automated, and policy-driven block storage provisioning on AWS.
