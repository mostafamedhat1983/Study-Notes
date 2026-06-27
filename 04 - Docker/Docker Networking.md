---
tags:
  - Linux
  - Docker
  - Networking
---
# Docker Networking

Docker networking controls how containers communicate with each other and with the outside world.  
This note covers network drivers, common commands, and how to connect containers.

---

## Core idea

- Every container gets its own network namespace by default
- Docker uses **network drivers** to define how containers connect
- Containers on the same network can reach each other by **container name** (DNS)
- Containers on different networks are isolated from each other
- The host machine and containers communicate through **port mapping** (`-p`)

---

## Network drivers

| Driver | Description |
|---|---|
| `bridge` | Default. Creates an isolated network on the host. Containers can communicate within the same bridge. |
| `host` | Removes network isolation. Container shares the host's network stack directly. |
| `none` | Disables all networking for the container. |
| `overlay` | Used in Docker Swarm for multi-host networking. |
| `macvlan` | Assigns a real MAC address to the container, making it appear as a physical device on the network. |

> The default bridge network does **not** support DNS-based container discovery. Use a **custom bridge** network instead.

---

## Network commands

### List networks
```bash
docker network ls
```

Shows all networks on the Docker host.

### Inspect a network
```bash
docker network inspect bridge
```

Shows detailed information about a network including connected containers and subnet.

### Create a network
```bash
docker network create mynetwork
```

Creates a new custom bridge network.

### Create a network with a specific driver
```bash
docker network create --driver bridge mynetwork
```

Explicitly sets the driver. Bridge is the default.

### Create a network with a custom subnet
```bash
docker network create --subnet 192.168.10.0/24 mynetwork
```

Defines a custom IP range for the network.

### Remove a network
```bash
docker network rm mynetwork
```

Removes a network.

> You cannot remove a network that has active containers connected to it.

### Remove all unused networks
```bash
docker network prune
```

Deletes all networks not used by any container.

---

## Connecting containers to networks

### Run a container on a specific network
```bash
docker run -d --name myapp --network mynetwork nginx
```

Connects the container to `mynetwork` at startup.

### Connect a running container to a network
```bash
docker network connect mynetwork myapp
```

Adds a running container to an additional network.

### Disconnect a container from a network
```bash
docker network disconnect mynetwork myapp
```

Removes a container from a network without stopping it.

---

## DNS and container discovery

- On a **custom bridge** network, containers can reach each other using their container name as a hostname
- On the **default bridge** network, you must use IP addresses

Example:
```bash
docker network create mynetwork
docker run -d --name db --network mynetwork mysql
docker run -d --name app --network mynetwork myapp
```

Inside `app`, you can reach `db` by name:
```bash
ping db
curl http://db:3306
```

---

## Host and none network

### Run with host network
```bash
docker run -d --name myweb --network host nginx
```

The container uses the host's network directly. No port mapping needed, but no network isolation either.

### Run with no network
```bash
docker run -d --name isolated --network none alpine
```

The container has no network access at all. Useful for security-sensitive workloads.

---

## Inspect container network settings

### Get container IP address
```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' myapp
```

Extracts the IP address of a container.

### Get all network info
```bash
docker inspect myapp
```

Look under `NetworkSettings` in the JSON output for full network details.

---

## Must memorize

```bash
docker network ls
docker network inspect
docker network create
docker network rm
docker network prune
docker network connect
docker network disconnect
```

---

## Quick flow

### Create a network and connect two containers
```bash
docker network create mynetwork
docker run -d --name db --network mynetwork mysql:8
docker run -d --name app --network mynetwork myapp:latest
docker network inspect mynetwork
```

### Connect an existing container to a new network
```bash
docker network create backendnet
docker network connect backendnet app
docker network inspect backendnet
```

---

## Notes

- Always use custom bridge networks in multi-container setups, not the default bridge.
- Docker Compose automatically creates a custom network for your services.
- Overlay networks require Docker Swarm mode to be initialized.
- `--network host` is Linux-only; it behaves differently on Docker Desktop (macOS/Windows).
- Port mapping (`-p`) is not needed when containers are on the same network — they communicate directly.
