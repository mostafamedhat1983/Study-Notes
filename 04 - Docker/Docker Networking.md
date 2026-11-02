---
tags:
  - Linux
  - Docker
  - Networking
---
# Docker Networking

Docker network drivers define how containers communicate with one another, the host, and external networks.

## Network drivers

| Driver | Purpose |
|---|---|
| `bridge` | Connects containers on the same Docker host using a Linux bridge. |
| `host` | Shares the host network namespace; available behavior differs by platform. |
| `none` | Isolates the container from normal network connectivity. |
| `overlay` | Connects services across multiple Docker daemon hosts, commonly in Swarm. |
| `macvlan` | Gives containers their own MAC address on a physical network. |
| `ipvlan` | Provides network connectivity without assigning a unique MAC address to each container. |

## Default bridge vs user-defined bridge

- Docker creates a default `bridge` network.
- Containers on the default bridge can communicate using IP addresses; automatic container-name DNS resolution is not provided there by default (legacy links are a separate feature).
- A user-defined bridge provides automatic DNS resolution between containers attached to that network.
- User-defined bridges provide better isolation and easier network configuration for application groups.

Create and use a user-defined bridge:

```bash
docker network create app-net
docker run -d --name db --network app-net mysql:8
docker run -d --name app --network app-net nginx
```

From the `app` container, the database hostname can resolve as `db` on that network, provided the relevant service is listening and network policy/firewall settings allow it.

## Test connectivity correctly

Do not use `curl http://db:3306` as a MySQL connectivity test. MySQL uses its own protocol; port 3306 is not an HTTP endpoint.

Useful checks include:

```bash
# DNS lookup, if the tool is installed in the container
getent hosts db

# TCP reachability, if netcat is installed
nc -vz db 3306

# MySQL protocol/application test, if the client is installed
mysql -h db -P 3306 -u <user> -p
```

A successful TCP connection does not prove that authentication or the database application is healthy; use a database client or application-level health check for that.

## Common commands

```bash
docker network ls
docker network inspect app-net
docker network create app-net
docker network connect app-net app
docker network disconnect app-net app
docker network rm app-net
docker network prune
```

A network cannot be removed while containers remain connected to it. `docker network prune` removes unused networks; inspect before pruning.

## Other network modes

```bash
docker run -d --name web --network host nginx
docker run -d --name isolated --network none alpine
```

With host networking, port publishing is not used in the usual bridge-network way. On Docker Desktop, host-network behavior and support differ from native Linux. With `none`, the container has only loopback networking and no normal external network connection.

## Port publishing

```bash
docker run -d --name web -p 8080:80 nginx
```

This publishes host port 8080 to container port 80. Containers on the same user-defined network generally communicate directly over the container port and do not need host port publishing for that inter-container traffic.

## Key takeaways

- Prefer user-defined bridge networks for multi-container applications on one Docker host.
- Use container/service names for DNS on user-defined networks.
- Use a MySQL client or TCP check for MySQL, not HTTP curl.
- Use overlay networks for multi-host connectivity where appropriate.
