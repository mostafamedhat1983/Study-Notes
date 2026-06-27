---
tags:
  - Docker
  - Linux
---
# daemon.json for Docker

`daemon.json` is Docker's main configuration file for the Docker daemon (`dockerd`).  
It lets you apply the same Docker settings every time the daemon starts, instead of typing startup flags manually.

---

## Common location
```bash
/etc/docker/daemon.json
```

## Rootless Docker location
```bash
~/.config/docker/daemon.json
```

---

## Full example

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "registry-mirrors": ["https://mirror.example.com"],
  "data-root": "/data/docker",
  "insecure-registries": ["192.168.1.10:5000"],
  "live-restore": true,
  "dns": ["8.8.8.8", "8.8.4.4"],
  "storage-driver": "overlay2",
  "bip": "192.168.1.1/24",
  "default-address-pools": [
    { "base": "172.20.0.0/16", "size": 24 }
  ]
}
```

---

## Settings explained

### log-driver
Sets the default logging driver for all containers.

- `json-file` = store logs as JSON files on the host (default)
- Other options: `syslog`, `journald`, `fluentd`, `none`

### log-opts
Options for the selected log driver.

- `max-size` = maximum size of a single log file before rotation (`10m` = 10 MB)
- `max-file` = number of rotated log files to keep (`3` = keep 3 files max)

Without these limits, log files can grow indefinitely and fill the disk.

### registry-mirrors
Pulls images from this mirror instead of Docker Hub when possible.

- Docker tries the mirror first, falls back to the original registry if it fails
- Useful when Docker Hub is slow, rate-limited, or in air-gapped environments

### data-root
Changes where Docker stores all its data — images, containers, volumes, and build cache.

- Default is `/var/lib/docker`
- Change this when the default partition is too small or you want Docker data on a separate disk
- Requires migrating existing data manually or starting fresh

### insecure-registries
Allows Docker to connect to a private registry over HTTP instead of HTTPS.

- By default Docker refuses any registry without a valid TLS certificate
- This whitelist tells Docker to trust the address without SSL

> Never use this in production. Always use proper TLS for registries.

### live-restore
Keeps containers running even if the Docker daemon restarts or crashes.

- Default is `false` — containers stop when the daemon stops
- Set to `true` to keep containers alive during Docker upgrades or daemon crashes
- Recommended for production hosts where container uptime matters

### dns
Sets custom DNS servers for all containers.

- Overrides the host's default DNS resolution inside containers
- Useful when your network has specific internal DNS servers

```json
"dns": ["8.8.8.8", "8.8.4.4"]
```

### storage-driver
Sets the storage backend Docker uses for image and container layers.

- Default is `overlay2` on most modern Linux systems
- Other options: `devicemapper`, `btrfs`, `zfs`
- Only change this if you have a specific requirement — `overlay2` is the recommended default

### bip
Sets the IP address and subnet of the default Docker bridge network (`docker0`).

- Useful when the default bridge subnet (`172.17.0.0/16`) conflicts with your existing network
- Format: `"bip": "192.168.1.1/24"`

### default-address-pools
Defines the IP ranges Docker uses when creating new custom networks.

- Prevents subnet overlap with your host or VPN networks
- `base` = the overall pool range, `size` = the size of each subnet carved from it

```json
"default-address-pools": [
  { "base": "172.20.0.0/16", "size": 24 }
]
```

---

## Validate config

Before restarting Docker, always validate the file:

```bash
# Using dockerd built-in validator
dockerd --validate --config-file /etc/docker/daemon.json

# Using Python JSON parser (quick syntax check)
cat /etc/docker/daemon.json | python3 -m json.tool
```

---

## Apply changes
```bash
sudo systemctl restart docker
sudo systemctl status docker
```

---

## Verify settings were loaded
```bash
# Check logging driver
docker info | grep -i logging

# Check data root
docker info | grep -i "docker root"

# Check live restore
docker info | grep -i "live restore"

# Check storage driver
docker info | grep -i "storage driver"

# See everything
docker info
```

---

## Notes

- The file must be valid JSON — a single syntax error prevents Docker from starting.
- Restart Docker after every edit.
- Do not set the same option in both `daemon.json` and Docker startup flags — this causes a conflict.
- If the file does not exist, create it: `sudo nano /etc/docker/daemon.json`
- Changes apply to new containers only, except daemon-level settings like `live-restore` and `data-root`.
- A bad `daemon.json` can stop Docker from starting — always validate before restarting.
