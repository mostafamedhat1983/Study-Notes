---
tags:
  - AWS
---
Direct Answer/Solution

Amazon EBS (Elastic Block Store) is block storage for a single EC2 instance (or a few via multi-attach on specific volume types), while Amazon EFS (Elastic File System) is a managed, shared NFS file system that many instances and services can mount at the same time. In EKS terms, EBS is typically used via the EBS CSI driver for pod-local, high‑IOPS volumes, and EFS via the EFS CSI driver for shared, ReadWriteMany storage across pods and nodes.

## Architectural Reasoning (Key differences and when to use)

### 1. Storage model and access pattern

- EBS:
  - Block storage, behaves like a disk attached to an EC2 instance.
  - Typically ReadWriteOnce per node (in Kubernetes terms); one node/pod “owns” the volume at a time.
  - Good for databases and single‑instance stateful workloads that need low latency and predictable IOPS.

- EFS:
  - Network file system (NFSv4) managed by AWS.
  - ReadWriteMany; many instances or pods can mount the same file system concurrently.
  - Good for shared configs, user uploads, web content, ML models, and any workload where multiple pods need the same files.

### 2. Availability and scope

- EBS:
  - Lives in a single Availability Zone; you must handle replication or snapshots if you want cross‑AZ resilience.
  - Attached to EC2 (or via CSI to EKS worker nodes) in the same AZ.

- EFS:
  - Regional service with mount targets in multiple AZs.
  - Designed for multi‑AZ access and higher availability without managing replication yourself.

### 3. Performance characteristics

- EBS:
  - Lower latency, volume‑level performance tuning (gp3, io1/io2, etc.).
  - You pick size and IOPS/throughput; good for transactional workloads and latency‑sensitive systems.

- EFS:
  - Higher latency than local block storage but high aggregate throughput.
  - Scales automatically with data size and access; ideal for bursty or horizontally scaled workloads.

### 4. Cost model

- EBS:
  - Pay for provisioned GB (and sometimes provisioned IOPS/throughput).
  - Cheaper per GB, but each instance/volume combination is billed separately.

- EFS:
  - Pay per GB stored and (depending on mode) throughput.
  - More expensive per GB than EBS, but one file system can be shared across many consumers, which may reduce total volumes and simplify management.

### 5. EKS / Kubernetes usage pattern

- Typical patterns:
  - Use EBS (via EBS CSI driver) for:
    - Databases inside Kubernetes (if you accept that pattern).
    - Single‑pod stateful apps that need fast, AZ‑local storage.
  - Use EFS (via EFS CSI driver) for:
    - Shared storage across replicas, blue/green deployments, and multiple microservices.
    - Workloads that need ReadWriteMany and multi‑AZ mounts.

## Security & Best Practice Check

- EBS:
  - Encrypt at rest (KMS), restrict attach permissions via IAM.
  - Use snapshots and backup policies for DR; keep AZ awareness in your design.

- EFS:
  - Use mount targets in private subnets, NFS over TLS, security groups, and IAM for access control.
  - Treat EFS like any network
