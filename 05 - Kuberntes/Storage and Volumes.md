---
tags:
  - Kubernetes
---
Kubernetes storage allows Pods to persist data beyond the container lifecycle and share data between containers.

---

## Main idea

- container filesystem is ephemeral — data is lost when the container restarts
- volumes provide persistent or shared storage for Pods
- PersistentVolumes (PV) are cluster-level storage resources
- PersistentVolumeClaims (PVC) are requests for storage by Pods
- StorageClass enables dynamic provisioning of volumes

---

## Volume types

### `emptyDir`

- created when the Pod starts, deleted when the Pod is removed
- shared between all containers in the same Pod
- data survives container restarts but not Pod deletion
- use for: temp files, caches, sharing data between sidecar and main container

```yaml
spec:
  volumes:
    - name: temp-storage
      emptyDir: {}
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: temp-storage
          mountPath: /tmp/data
```

### `hostPath`

- mounts a file or directory from the host node filesystem into the Pod
- data persists as long as the node exists
- not portable — Pod must land on the same node to see the same data
- use for: node-level log access, running system agents (DaemonSets)
- avoid in production for app data

```yaml
spec:
  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory
```

---

## PersistentVolume (PV)

- a cluster-level storage resource provisioned by an admin or dynamically
- exists independently of any Pod
- has a capacity, access mode, and reclaim policy

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/my-pv
```

---

## PersistentVolumeClaim (PVC)

- a request for storage by a Pod
- Kubernetes binds the PVC to a matching PV
- the Pod uses the PVC, not the PV directly

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Using PVC in a Pod

```yaml
spec:
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-pvc
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: my-storage
          mountPath: /data
```

---

## Access modes

| Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | one node can read and write |
| `ReadOnlyMany` | ROX | many nodes can read |
| `ReadWriteMany` | RWX | many nodes can read and write |
| `ReadWriteOncePod` | RWOP | one Pod can read and write (K8s 1.22+) |

---

## Reclaim policy

| Policy | Behavior after PVC deleted |
|---|---|
| `Retain` | PV stays, data preserved, manual cleanup needed |
| `Delete` | PV and underlying storage deleted automatically |
| `Recycle` | deprecated — basic scrub and reuse |

---

## StorageClass

- enables dynamic volume provisioning
- when a PVC requests a StorageClass, Kubernetes automatically creates a PV
- different classes can offer different performance tiers (SSD, HDD, etc.)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

---

## Common mistakes

- using emptyDir for data that must survive Pod restarts
- using hostPath for application data (not portable across nodes)
- not setting a reclaim policy (data may be deleted unexpectedly)
- not matching access modes between PV and PVC
- not using StorageClass for dynamic provisioning in cloud environments
- forgetting that RWX is not supported by all storage backends (EBS only supports RWO)

---

## Good practices

- use PVCs in Pod specs, never reference PVs directly
- use StorageClass for dynamic provisioning in production
- use `Retain` reclaim policy for important data
- use `ReadWriteOnce` for most database workloads
- use `ReadWriteMany` (NFS, EFS) when multiple Pods need the same data
- use StatefulSets for stateful apps — they manage PVCs per Pod automatically

---

## Must memorize

```
emptyDir    → ephemeral, shared within Pod
hostPath    → node filesystem, not portable
PV          → cluster-level storage resource
PVC         → Pod's request for storage
StorageClass → dynamic provisioning
```

```bash
kubectl get pv
kubectl get pvc
kubectl get storageclass
kubectl describe pvc <name>
```

---

## Key ideas

- Container storage is ephemeral by default — use volumes for persistence.
- Pods use PVCs to request storage; PVCs bind to PVs.
- StorageClass enables dynamic PV provisioning without admin intervention.
- Reclaim policy controls what happens to data after a PVC is deleted.
- Use StatefulSets with volumeClaimTemplates for stateful apps.
