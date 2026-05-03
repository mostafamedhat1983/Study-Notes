# Kubernetes and the Linux Kernel

Kubernetes does not implement container isolation itself — it relies entirely on Linux kernel primitives. Understanding these helps you debug resource issues, security constraints, and container behavior.

---

## Linux Namespaces

Namespaces provide **isolation** by giving each process its own view of system resources. A container is essentially a process running in a set of namespaces.

| Namespace | Isolates |
|---|---|
| `pid` | Process IDs — processes in the container only see their own PIDs |
| `net` | Network interfaces, routing, ports |
| `mnt` | Filesystem mount points |
| `uts` | Hostname and domain name |
| `ipc` | Inter-process communication (shared memory, semaphores) |
| `user` | User and group IDs (rootless containers) |
| `cgroup` | cgroup hierarchy view (added later) |

> Pods in Kubernetes share the **network namespace** across all containers in the same Pod — that's why containers in a Pod communicate via `localhost` and share the same IP.

---

## cgroups (Control Groups)

cgroups control **how much** of a resource a process can use. This is what enforces Kubernetes `requests` and `limits`.

| cgroup subsystem | Controls |
|---|---|
| `cpu` | CPU shares and quotas |
| `memory` | Memory usage and OOM kills |
| `blkio` | Block I/O bandwidth |
| `pids` | Number of processes |

- `requests` set the **cgroup share weight** — the minimum guaranteed share
- `limits` set the **cgroup hard cap** — the container is throttled (CPU) or killed (memory) if exceeded
- Memory OOMKill comes from the kernel, not Kubernetes — seen as `OOMKilled` in Pod status

```bash
# Check cgroup for a container
cat /proc/<pid>/cgroup

# Check memory limit enforced on a container
cat /sys/fs/cgroup/memory/<container-id>/memory.limit_in_bytes
```

---

## Container Runtime

Kubernetes uses the **Container Runtime Interface (CRI)** to delegate container lifecycle operations to a runtime.

| Runtime | Notes |
|---|---|
| `containerd` | Default in most modern clusters (EKS, GKE, AKS) |
| `CRI-O` | Lightweight, OCI-focused |
| `Docker` (via dockershim) | Deprecated — removed in Kubernetes 1.24 |

The runtime is responsible for:
- Pulling images
- Creating containers using Linux namespaces + cgroups
- Managing container lifecycle (start, stop, delete)

---

## OCI Standards

- **OCI Image Spec**: defines the container image format — layers, manifests
- **OCI Runtime Spec**: defines how to run a container — `runc` is the reference implementation
- `runc` is what actually calls `clone()` (Linux syscall) to create namespaces and set up cgroups

---

## seccomp and AppArmor

Additional kernel security layers that can be applied to containers:

- **seccomp** (Secure Computing Mode): restricts which syscalls a container can make
- **AppArmor**: restricts file/network access by process path

```yaml
# Apply seccomp runtime default
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

---

## eBPF

Extended Berkeley Packet Filter — a modern Linux kernel technology used by advanced Kubernetes tooling:

- **Cilium** uses eBPF for NetworkPolicy enforcement, load balancing, and observability without iptables
- **Falco** uses eBPF for runtime security detection
- **Pixie** uses eBPF for zero-instrumentation observability

eBPF programs run in kernel space safely, making them extremely performant for networking and tracing.

---

## Key Takeaways

- Containers = Linux namespaces + cgroups + a root filesystem
- Kubernetes orchestrates processes; the kernel isolates them
- OOMKill is a kernel event — always set memory limits carefully
- CPU throttling (not kill) happens when CPU limit is exceeded
- Understanding the runtime layer (containerd/runc) helps when debugging `CrashLoopBackOff` and image pull failures
