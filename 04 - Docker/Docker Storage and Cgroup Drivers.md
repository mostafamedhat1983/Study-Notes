---
tags:
  - Linux
  - Docker
---
## Overview
Docker uses a **storage driver** to manage container image layers and writable data, and a **cgroup driver** to manage CPU, memory, and other resource controls.

On modern Linux hosts, the usual choices are:
- `overlay2` for storage.
- `systemd` for cgroups.

## Storage drivers
Storage drivers control how Docker stores image layers and the writable container layer on disk. They use layered filesystems and copy-on-write so containers can share base layers efficiently.

### Common storage drivers
- `overlay2`: Recommended for most modern Linux systems. Best balance of performance and compatibility.
- `overlay`: Older OverlayFS-based driver. Mostly legacy.
- `aufs`: Older union filesystem driver. Mostly found on older Ubuntu/Docker setups.
- `devicemapper`: Block-device based driver. Common in older enterprise environments, but mostly legacy now.
- `btrfs`: Uses Btrfs filesystem features such as snapshots and subvolumes.
- `zfs`: Uses ZFS features such as snapshots, compression, and integrity checks.
- `fuse-overlayfs`: Useful for rootless Docker or systems with limited kernel overlay support.
- `vfs`: Very simple fallback driver. Slow, mostly for testing or debugging.

### Differences
- `overlay2` is usually the best default choice.
- `overlay` is older and less preferred than `overlay2`.
- `btrfs` and `zfs` depend on those specific filesystems being in place.
- `devicemapper` and `aufs` are mostly legacy.
- `fuse-overlayfs` is mainly for rootless or constrained environments.
- `vfs` is simplest but slowest.

### When to use each
- Use `overlay2` for almost all modern Linux hosts.
- Use `fuse-overlayfs` for rootless Docker or limited environments.
- Use `btrfs` if your host is already built around Btrfs and you want snapshot features.
- Use `zfs` if your environment already uses ZFS and you want its storage features.
- Use `devicemapper` only if an older environment requires it.
- Use `vfs` only for troubleshooting or unsupported setups.

### Important note
On many systems, Docker reports `overlay2` as the storage driver name, while OverlayFS is the underlying technology. In practice, `overlay2` is the name you usually see in `docker info`.

### How to check
```bash
docker info | grep "Storage Driver"
```

### How to change
Edit `/etc/docker/daemon.json`:
```json
{
  "storage-driver": "overlay2"
}
```

Then restart Docker:
```bash
sudo systemctl restart docker
```

### Warning
Changing the storage driver is a disruptive change. It can affect existing images and containers, so treat it carefully on production systems.

## Cgroup drivers
Cgroup drivers control how Docker manages Linux cgroups for CPU, memory, and process isolation.

### Common cgroup drivers
- `systemd`: Docker asks systemd to manage cgroups. This is the preferred choice on most modern Linux distributions.
- `cgroupfs`: Docker manages cgroups directly through the Linux cgroup filesystem.

### Differences
- `systemd` integrates with the host init system and gives more consistent resource management.
- `cgroupfs` is more direct, but on systemd-based hosts it can create two different cgroup managers.
- That mismatch can cause instability or Kubernetes startup problems.

### When to use each
- Use `systemd` on modern Linux systems.
- Use `systemd` for Kubernetes nodes in most cases.
- Use `cgroupfs` only if your platform explicitly requires it.
- If your host uses cgroup v2, `systemd` is generally the right choice.

### How to check
```bash
docker info | grep "Cgroup Driver"
```

### How to change
Edit `/etc/docker/daemon.json`:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

Then restart Docker:
```bash
sudo systemctl restart docker
```

### Kubernetes note
If Kubernetes is involved, Docker and kubelet must use the same cgroup driver. A mismatch between kubelet and Docker is a common source of startup errors.

## Why DevOps cares
These settings matter because DevOps is about making sure the operating system, Docker, and Kubernetes all agree on how containers are stored and controlled.

If the storage driver is wrong, you may see performance issues, filesystem incompatibility, or storage-related errors.
If the cgroup driver is wrong, Kubernetes nodes may fail to start cleanly or become unstable under load.

### Practical rule
- Keep `overlay2` unless you have a specific reason to change it.
- Use `systemd` for cgroups on modern Linux and Kubernetes hosts.
- Change these settings only when you understand the platform requirement or the failure you are solving.

## Troubleshooting signs
- Storage driver issues may show up as Docker startup failures, filesystem compatibility errors, or broken container/image behavior.
- Cgroup driver issues may show up as kubelet misconfiguration errors or node bootstrap failures.
- When troubleshooting, compare `docker info` with kubelet or runtime configuration.

## Quick interview answer
- Storage driver = how Docker stores image and container layers on disk.
- Cgroup driver = how Docker manages CPU and memory controls.
- `overlay2` is the common modern storage driver.
- `systemd` is the common modern cgroup driver.
- Change them only for compatibility, performance, or orchestration requirements.