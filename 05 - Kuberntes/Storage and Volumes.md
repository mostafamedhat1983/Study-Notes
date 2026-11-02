---
tags:
  - Kubernetes
---
Kubernetes volumes provide storage to containers. PersistentVolumes (PVs), PersistentVolumeClaims (PVCs), and StorageClasses support storage lifecycle and dynamic provisioning.

## Main concepts

- A container's writable layer is tied to that container instance; a volume can outlive an individual container restart.
- A Pod volume is available to containers in that Pod when mounted.
- A PV represents a storage resource in the cluster.
- A PVC is a request for storage by a workload.
- A StorageClass describes a provisioner and parameters for dynamic volume provisioning.

## `emptyDir`

- Created for a Pod on the node where it is scheduled.
- Shared by containers in that Pod when mounted.
- Survives a container restart while the Pod remains, but is deleted when the Pod is removed from the node.
- Can be memory-backed with `medium: Memory`; memory-backed data counts against memory limits/usage.

```yaml
spec:
  volumes:
    - name: scratch
      emptyDir: {}
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: scratch
          mountPath: /tmp/data
```

## `hostPath`

Mounts a file or directory from the node's filesystem into a Pod. It is node-specific and can expose host data or resources. Use it only when the workload genuinely requires host access (for example, some node agents); it is generally unsuitable for portable application data.

## PersistentVolumes and Claims

A PV can be provisioned statically by an administrator or dynamically through a StorageClass. A PVC requests capacity and access modes; Kubernetes binds it to a matching PV or triggers dynamic provisioning.

Example PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 10Gi
```

A Pod references the claim:

```yaml
spec:
  volumes:
    - name: app-data
      persistentVolumeClaim:
        claimName: app-data
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: app-data
          mountPath: /data
```

## Access modes

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Volume can be mounted read-write by a single node. Multiple Pods on that same node may be able to use it. |
| `ReadOnlyMany` (ROX) | Volume can be mounted read-only by many nodes, if the storage backend supports it. |
| `ReadWriteMany` (RWX) | Volume can be mounted read-write by many nodes, if the backend supports it. |
| `ReadWriteOncePod` (RWOP) | Volume can be mounted read-write by a single Pod; CSI support and Kubernetes version requirements apply. |

Access modes are used for PV/PVC matching and do not universally enforce filesystem-level write protection after a volume is mounted. Check the CSI driver and storage backend capabilities.

## Reclaim policy

| Policy | Behavior after claim release |
|---|---|
| `Retain` | PV and underlying data are retained for manual recovery/cleanup. |
| `Delete` | PV and the provisioned storage asset are deleted by the provisioner, if supported. |
| `Recycle` | Deprecated legacy behavior; do not use for new configurations. |

## StorageClass example: Amazon EBS CSI

For EKS with the Amazon EBS CSI driver, use the CSI provisioner name:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

The old in-tree provisioner name `kubernetes.io/aws-ebs` is deprecated; use the EBS CSI driver. `WaitForFirstConsumer` delays binding/provisioning until a Pod is scheduled, helping the provisioner choose a topology compatible with the Pod.

Amazon EBS volumes are block storage with attachment constraints; do not describe EBS as general multi-node RWX storage. For shared filesystem access across nodes, evaluate a backend such as Amazon EFS and its CSI driver.

## Good practices

- Use PVCs for application storage and a StorageClass for dynamic provisioning.
- Choose access modes based on the backend's real capabilities.
- Understand the reclaim policy before deleting claims.
- Use StatefulSets and `volumeClaimTemplates` when each replica needs its own claim.
- Avoid `hostPath` for portable application data.
