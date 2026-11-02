---
tags:
  - Linux
  - Docker
---
# Docker Storage and Cgroup Drivers

Docker storage drivers manage image layers and container writable layers. Cgroup drivers determine how the container runtime manages Linux control groups for resource accounting and limits.

## Storage drivers

The available storage driver depends on the Docker Engine version, Linux kernel, backing filesystem, and installation mode. On many current Linux installations, Docker uses the OverlayFS-based `overlay2` driver; newer Docker Engine releases may expose the `overlayfs` driver name. Do not assume every listed legacy driver is supported by a current installation.

Common names you may encounter:

- **overlay2 / overlayfs:** OverlayFS-based layered storage.
- **btrfs / zfs:** filesystem-specific storage backends.
- **fuse-overlayfs:** userspace OverlayFS implementation used in some rootless configurations.
- **vfs:** simple copy-based driver, useful for testing but generally inefficient.
- **aufs / devicemapper:** legacy drivers; availability depends on older distributions and Docker versions.

Check the actual configuration:

```bash
docker info
```

Do not change the storage driver casually on a host with existing images or containers. Follow the Docker documentation for the Engine version and plan for the impact on local image/container data.

## Cgroup drivers

The cgroup driver controls how a container runtime organizes and manages cgroups.

Common options:

- **systemd:** uses systemd to manage cgroups.
- **cgroupfs:** manages cgroups directly through the cgroup filesystem.

On systemd-based Linux distributions, using the systemd cgroup driver is generally recommended, especially with cgroup v2.

Check Docker's setting:

```bash
docker info | grep -i 'Cgroup Driver'
```

If Docker Engine is used as a standalone runtime, its daemon configuration may include:

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

Restarting Docker after a daemon configuration change affects running workloads; plan the change.

## Kubernetes and the runtime cgroup driver

Do not memorize “Docker and kubelet must use the same cgroup driver” as a universal modern Kubernetes rule.

- Kubelet communicates with a container runtime through the Container Runtime Interface (CRI).
- Docker Engine is not directly supported by kubelet through the removed in-tree dockershim; using Docker Engine requires an external CRI adapter such as `cri-dockerd`.
- The kubelet cgroup driver should match the cgroup driver used by the CRI runtime.
- On systemd hosts, Kubernetes recommends the `systemd` cgroup driver to avoid having two different cgroup managers.

For a Kubernetes node, inspect the actual runtime and kubelet configuration rather than assuming Docker is the runtime.

## Interview takeaway

Storage driver = how container image and writable layers are stored. Cgroup driver = how container resource-control groups are organized. In Kubernetes, align kubelet with the **CRI runtime**, not automatically with Docker Engine.
