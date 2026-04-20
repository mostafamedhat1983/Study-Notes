---
tags:
  - Docker
  - Linux
---

Docker volumes are used to persist container data outside the container itself.  
They are important when a container stores data that should survive container deletion or recreation.

---

## Why volumes matter

Containers are temporary by nature.

If a container writes data inside its own writable layer and the container is removed, that data may be lost.

To preserve data, Docker allows you to mount storage from the host into the container.

This is useful for:

- databases
- application data
- uploaded files
- configuration files
- development code syncing

---

## Two common ways to mount data

### Bind mount
Maps a specific host directory into the container.

### Docker volume
Uses a Docker-managed volume stored under Docker’s internal storage.

Both allow data to persist outside the container lifecycle.

---

## Bind mount vs Docker volume

| Feature | Bind mount | Docker volume |
|---|---|---|
| Source | Specific host path | Docker-managed storage |
| Managed by | User / host OS | Docker |
| Best for | Development, injecting files, direct host control | Persistent app or database data |
| Host path required | Yes | No |
| Storage location | Any path you choose | Usually under `/var/lib/docker/volumes` |
| Portability | Less portable | More portable inside Docker workflows |

### Rule of thumb
- Use **bind mounts** when you want to map a specific host directory.
- Use **Docker volumes** when you want Docker-managed persistent storage.

---

## tmpfs mounts

Docker provides a third storage option besides volumes and bind mounts: **tmpfs mounts**.

A tmpfs mount stores data in the host machine's memory instead of on disk.

This means:
- data is temporary
- data is not persisted after the container stops
- data does not live in the container writable layer
- data is not stored as a normal host directory like a bind mount

tmpfs is useful when the container needs fast temporary storage that should not remain after shutdown.

---

### When tmpfs is useful

Use tmpfs when:
- the data is temporary
- the data should not persist
- the application writes lots of short-lived files
- you want to avoid writing sensitive temporary data to disk
- you want fast in-memory storage

Good examples:
- cache directories
- temporary processing files
- session-like temporary data
- short-lived secrets or temporary credentials
- application scratch space

---

### Important behavior

A tmpfs mount is:
- stored in memory
- removed when the container stops
- non-persistent

So unlike a volume or bind mount, data inside tmpfs does **not** survive container removal or restart.

This makes tmpfs very different from normal persistent storage.

---

### Basic example

```bash
docker run -d --name myapp --mount type=tmpfs,target=/app/cache nginx
```

This creates a tmpfs mount at `/app/cache` inside the container.

Anything written there is stored in memory, not on disk.

---

### Example with options

```bash
docker run -d \
  --name myapp \
  --mount type=tmpfs,target=/app/cache,tmpfs-size=64m,tmpfs-mode=1777 \
  nginx
```

This creates a tmpfs mount with:
- size limit of 64 MB
- mode `1777`

This is useful when you want more control over memory-backed temporary storage.

---

### Short syntax

Docker also supports a shorter form:

```bash
docker run -d --tmpfs /app/cache nginx
```

This is convenient for simple cases.

---

### tmpfs vs volume vs bind mount

| Type | Stored where | Persistent | Best for |
|---|---|---|---|
| Bind mount | Specific host path | Yes | Development, host-controlled files |
| Docker volume | Docker-managed storage | Yes | Databases, application data, persistent state |
| tmpfs | Host memory | No | Temporary files, caches, sensitive short-lived data |

---

### Best practices

- Use tmpfs only for data that does not need to survive container stop or removal.
- Set size limits when possible to avoid uncontrolled memory usage.
- Do not use tmpfs for important database or application state.
- Use volumes for persistent data.
- Use bind mounts when you need a specific host path.

---

### Simple rule of thumb

- Use a **volume** for persistent app data.
- Use a **bind mount** when the container must access a real host path.
- Use **tmpfs** when the data is temporary and should live only in memory.

---

### Must memorize

```bash
docker run --mount type=tmpfs,target=/app/cache nginx
docker run --tmpfs /app/cache nginx
```
## How to find volume paths and ports

Before running a container, you can inspect the image to find useful metadata.

### Inspect image
```bash
docker inspect mysql:5.7
```

This helps you find:

- exposed ports
- volume paths
- environment variables
- `CMD`
- `ENTRYPOINT`

For MySQL, the important container data path is commonly:

```bash
/var/lib/mysql
```

That is the directory where MySQL stores its database files inside the container.

---

## Required environment variable

Some images require environment variables to start properly.

For MySQL, a root password must be set:

```bash
-e MYSQL_ROOT_PASSWORD=secretpass
```

Without this, the container may fail to start.

---

## Bind mount

A bind mount maps a host directory directly into the container.

### Create a host directory
```bash
mkdir ~/vprodbdata
```

### Run MySQL with bind mount
```bash
docker run -d \
  --name vprodb \
  -e MYSQL_ROOT_PASSWORD=secretpass \
  -p 3030:3306 \
  -v /home/ubuntu/vprodbdata:/var/lib/mysql \
  mysql:5.7
```

### Explanation
- `-d` = detached mode
- `--name vprodb` = container name
- `-e MYSQL_ROOT_PASSWORD=secretpass` = set root password
- `-p 3030:3306` = map host port 3030 to container port 3306
- `-v /home/ubuntu/vprodbdata:/var/lib/mysql` = bind mount host directory to MySQL data directory

### Result
Data written by MySQL inside `/var/lib/mysql` is actually stored in:

```bash
/home/ubuntu/vprodbdata
```

on the host machine.

---

## Verify bind mount

### Check host directory
```bash
ls ~/vprodbdata
```

You should see MySQL data files there.

### Open shell inside container
```bash
docker exec -it vprodb /bin/bash
```

### Check MySQL data directory
```bash
cd /var/lib/mysql
ls
```

The contents should match the host directory.

---

## Bind mount behavior

If you stop and remove the container, the host directory still exists.

### Stop container
```bash
docker stop vprodb
```

### Remove container
```bash
docker rm vprodb
```

### Check host data
```bash
ls ~/vprodbdata
```

The data remains because it is stored on the host, not inside the deleted container.

---

## When bind mounts are useful

Bind mounts are commonly used when:

- you want to inject host files into the container
- developers want live code changes reflected inside the container
- you want direct control over the host path

They are often used in local development.

---

## Docker-managed volumes

Docker volumes are managed by Docker itself.

They are usually better than bind mounts for persistent application data.

### Create a volume
```bash
docker volume create mydbdata
```

### List volumes
```bash
docker volume ls
```

### Inspect a volume
```bash
docker volume inspect mydbdata
```

This shows details such as:

- volume name
- driver
- mountpoint on the host

### Run MySQL with Docker volume
```bash
docker run -d \
  --name vprodb \
  -e MYSQL_ROOT_PASSWORD=secretpass \
  -p 3030:3306 \
  -v mydbdata:/var/lib/mysql \
  mysql:5.7
```
***Docker will create the named volume automatically** if `mydbdata` does not already exist*
### Explanation
Instead of providing a full host path, you give only the volume name:

```bash
mydbdata:/var/lib/mysql
```

Docker stores this volume under its internal storage area.

---

## Where Docker volume data is stored

On Linux, Docker volumes are commonly stored under:

```bash
/var/lib/docker/volumes
```

For example:

```bash
/var/lib/docker/volumes/mydbdata/_data
```

That `_data` directory contains the actual files stored in the volume.

---

## Verify Docker volume

### List Docker volumes directory
```bash
sudo ls /var/lib/docker/volumes
```

### Check specific volume data
```bash
sudo ls /var/lib/docker/volumes/mydbdata/_data
```

You should see MySQL data files there.

---

## Why volumes are preferred

Docker volumes are generally preferred for persistent container data because:

- they are managed by Docker
- they are easier to reuse across containers
- they are cleaner than hardcoded host paths
- they work well for databases and stateful services

Bind mounts are more common for development workflows, while Docker volumes are more common for preserving service data.

---

## Inspect a running container

You can inspect a container to see its runtime details.

### Inspect container
```bash
docker inspect vprodb
```

This gives detailed JSON metadata about the container.

Useful fields include:

- container ID
- running status
- image used
- mounted volumes
- bind mounts
- environment variables
- exposed ports
- port bindings
- IP address
- `CMD`
- `ENTRYPOINT`

---

## What to look for in inspect output

### Mounts / Binds
Shows where host storage is connected to container storage.

### Ports
Shows which container ports are mapped to host ports.

### Env
Shows environment variables passed to the container.

### CMD and ENTRYPOINT
Shows what command/script the container runs at startup.

### IP address
Shows the internal IP address of the container inside Docker’s network.

---

## Logs and troubleshooting

You can also use logs to troubleshoot container startup issues.

### Show logs
```bash
docker logs vprodb
```

Logs show the output of the container’s main process.

This is useful when:

- the container exits immediately
- the startup command fails
- required environment variables are missing
- the application inside the container crashes

---

## Example: full volume workflow

### Pull image
```bash
docker pull mysql:5.7
```

### Inspect image
```bash
docker inspect mysql:5.7
```

### Create volume
```bash
docker volume create mydbdata
```

### Run container
```bash
docker run -d \
  --name vprodb \
  -e MYSQL_ROOT_PASSWORD=secretpass \
  -p 3030:3306 \
  -v mydbdata:/var/lib/mysql \
  mysql:5.7
```

### Check container
```bash
docker ps
```

### Check logs
```bash
docker logs vprodb
```

### Inspect running container
```bash
docker inspect vprodb
```

### Inspect volume
```bash
docker volume inspect mydbdata
```

---

## Connect to the containerized MySQL service

Once the container is running, you can connect to MySQL using the mapped host port.

Example:

```bash
mysql -h 127.0.0.1 -P 3030 -u root -p
```

Then enter the password:

```bash
secretpass
```

This works because host port `3030` is mapped to container port `3306`.

---

## Cleanup

### Stop container
```bash
docker stop vprodb
```

### Remove container
```bash
docker rm vprodb
```

### Remove image
```bash
docker rmi mysql:5.7
```

### Remove a volume
```bash
docker volume rm mydbdata
```

### Remove unused anonymous volumes
```bash
docker volume prune
```

### Remove all unused volumes
```bash
docker volume prune -a
```

Be careful: removing a volume deletes the stored data.

---

## Must memorize

```bash
docker inspect <image>
docker inspect <container>
docker run -d --name <name> -e VAR=value -p host:container -v source:target <image>
docker volume create <volume>
docker volume ls
docker volume inspect <volume>
docker volume rm <volume>
docker volume prune
docker volume prune -a
docker logs <container>
docker stop <container>
docker rm <container>
```

---

## Key ideas

- Volumes preserve data outside the container.
- Bind mounts use a real host path.
- Docker volumes use Docker-managed storage.
- `docker inspect` helps find ports, mounts, env vars, and startup commands.
- `docker logs` helps troubleshoot container startup problems.
- Removing a container does not remove external bind mount or volume data.

---

## Notes

- MySQL commonly stores data in `/var/lib/mysql` inside the container.
- Bind mounts are useful for development and direct host access.
- Docker volumes are usually better for persistent service data.
- Volume data on Linux is commonly stored under `/var/lib/docker/volumes`.