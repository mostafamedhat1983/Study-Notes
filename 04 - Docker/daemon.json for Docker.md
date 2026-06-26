---
tags:
  - Docker
  - Linux
---
`daemon.json` is Docker’s main configuration file for the Docker daemon (`dockerd`).

## What it is used for
It is usually used to set default Docker behavior on a host, such as:
- logging settings.
- data storage location.
- DNS servers.
- registry mirrors.
- insecure or private registries.
- storage and networking defaults.

## Most common use cases
- **Logging control:** set the log driver and rotate logs to avoid disk issues.
- **Registry mirrors:** speed up image pulls by using a mirror close to you.
- **Custom data location:** move Docker data to another disk with `data-root`.
- **Private registry access:** allow insecure registries or self-signed registry endpoints.
- **Network defaults:** set bridge IP, DNS, or other networking options.
- **Security/runtime defaults:** enable `live-restore` or other daemon-wide behaviors.

## Why it is useful
It lets you apply the same Docker settings every time the daemon starts, instead of typing startup flags manually.

## Common location
```bash
/etc/docker/daemon.json
```

## Rootless Docker location
```bash
~/.config/docker/daemon.json
```

## Example
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
  "live-restore": true
}
```

## Common useful keys
- `log-driver` → sets the default logging driver.
- `log-opts` → sets log rotation and logging options.
- `data-root` → changes Docker’s data directory.
- `registry-mirrors` → adds mirror endpoints for faster pulls.
- `insecure-registries` → allows HTTP or untrusted registries.
- `dns` → sets DNS servers for containers.
- `live-restore` → keeps containers running if Docker daemon stops.
- `storage-driver` → sets the storage backend.
- `bip` → sets the Docker bridge IP.
- `default-address-pools` → avoids subnet overlap issues.

## Validate config
```bash
dockerd --validate --config-file /etc/docker/daemon.json
```

## Apply changes
```bash
sudo systemctl restart docker
sudo systemctl status docker
```

## Quick verification
```bash
docker info
```

## Notes
- The file must be valid JSON.
- Restart Docker after editing it.
- Do not set the same option in both `daemon.json` and Docker startup flags.
- If the file does not exist, you can create it.
- A bad `daemon.json` can stop Docker from starting.