---
tags:
  - Docker
  - Linux
---
Docker can run in two main modes: normal (rootful) mode and rootless mode.

## Normal Docker
- The Docker daemon runs as root.
- Containers are managed by a root-owned service.
- It has the widest feature support.
- It is the default mode on most Linux systems.

## Rootless Docker
- The Docker daemon runs as a regular user.
- Containers also run without root privileges on the host.
- It is more secure because it reduces the attack surface.
- It may have networking and feature limitations.

## Key differences
| Area | Normal Docker | Rootless Docker |
|---|---|---|
| Daemon user | root | regular user |
| Security | more powerful, less restricted | safer, less privileged |
| Setup | simpler | needs extra setup |
| Networking | full support | some limits |
| Storage | `/var/lib/docker` | user home directory paths |
| Compatibility | full | some features may be limited |

## Common config locations
- Normal Docker: `/etc/docker/daemon.json`
- Rootless Docker: `~/.config/docker/daemon.json`

## How to check mode
```bash
docker info | grep -i rootless
```

## When to use normal Docker
- Production servers.
- Full compatibility is needed.
- Advanced networking or system integration is required.

## When to use rootless Docker
- Better security is the priority.
- Docker should not run as root.
- The machine is shared or less trusted.
- Some limitations are acceptable.

## Common rootless limitations
- Limited privileged port binding.
- Some network features are restricted.
- Extra setup may be needed for systemd user services.
- Not always ideal for production workloads.

## Example commands
```bash
sudo systemctl restart docker
systemctl --user restart docker
```

## Install and use rootless Docker
Rootless Docker runs the Docker daemon as a regular user instead of root.

### Before you start
- Stop the system-wide Docker service if it is already running.
- Make sure the user has the required subordinate UID/GID ranges.
- Some systems also need `uidmap`, `dbus-user-session`, or `fuse-overlayfs`.

### Install rootless Docker
```bash
dockerd-rootless-setuptool.sh install
```

If that script is not available, the rootless install script can be used:
```bash
curl -fsSL https://get.docker.com/rootless | sh
```

### Start and manage the service
```bash
systemctl --user start docker
systemctl --user restart docker
systemctl --user status docker
```

### Enable it on boot
```bash
systemctl --user enable docker
sudo loginctl enable-linger $USER
```

### Set environment variables
Add these to `~/.bashrc` or `~/.zshrc` if needed:
```bash
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix:///run/user/1000/docker.sock
```

### Verify it works
```bash
docker info
docker run hello-world
```

### Common notes
- Rootless Docker uses user-level config and data paths.
- The default config is usually in `~/.config/docker/daemon.json`.
- The Docker socket is usually under `/run/user/<uid>/docker.sock`.
- Some networking and privileged port features are limited in rootless mode.

## Quick summary
Normal Docker = more power and compatibility.
Rootless Docker = less privilege and better security.