---
tags:
  - Linux
  - Docker
---
# Docker Basics

Docker containers are lightweight processes created from images.  
This note covers the most common commands for working with containers, inspecting them, and cleaning up local Docker resources.

---

## Core idea

- **Image** = read-only template used to create containers
- **Container** = running instance of an image
- **Docker Hub** = default registry for pulling public images
- A container runs as a process on the host
- Containers can be started, stopped, inspected, and removed

---

## Images

### Pull an image
```bash
docker pull nginx
```

Downloads an image from Docker Hub.

### Pull a specific tag
```bash
docker pull nginx:mainline-alpine-perl
```

Downloads a specific version/tag of an image.

### List local images
```bash
docker images
```

Shows images available on the local machine.

### Remove an image
```bash
docker rmi nginx
```

Removes an image.

> You cannot remove an image if a container is still using it.

---

## Containers

### Run a container
```bash
docker run hello-world
```

Creates and starts a container from an image.

### Run a container in detached mode
```bash
docker run -d --name myweb -p 7090:80 nginx
```

- `-d` = run in background
- `--name myweb` = assign a custom container name
- `-p 7090:80` = map host port `7090` to container port `80`

### Run Ubuntu interactively
```bash
docker run -it ubuntu /bin/bash
```

Starts a container and attaches your terminal to a Bash shell.

- `-i` = interactive
- `-t` = terminal

### List running containers
```bash
docker ps
```

Shows only running containers.

### List all containers
```bash
docker ps -a
```

Shows running and stopped containers.

### Stop a container
```bash
docker stop myweb
```

Gracefully stops a running container.

### Start a stopped container
```bash
docker start myweb
```

Starts an existing stopped container.

### Restart a container
```bash
docker restart myweb
```

Restarts a container.

### Force stop a container
```bash
docker kill myweb
```

Immediately kills the container process.

### Remove a container
```bash
docker rm myweb
```

Removes a stopped container.

> A running container must be stopped before removal.

---

## Execute commands inside a container

### Run a command
```bash
docker exec myweb ls /
```

Runs a command inside a running container.

### Open Bash inside a container
```bash
docker exec -it myweb /bin/bash
```

Attaches your terminal to a Bash shell inside the container.

### Use sh if bash does not exist
```bash
docker exec -it myweb sh
```

Useful for minimal images like Alpine.

---

## Inspect and monitor

### Inspect a container or image
```bash
docker inspect myweb
```

Shows detailed JSON metadata.

### Show processes inside a container
```bash
docker top myweb
```

Lists processes running inside the container.

### Show live resource usage
```bash
docker stats
```

Displays live CPU, memory, and network usage for running containers.

### Show resource usage for one container
```bash
docker stats myweb
```

Displays live stats for a specific container.

### Show Docker version
```bash
docker version
```

Displays Docker client and server version information.

### Show Docker system info
```bash
docker info
```

Displays system-wide Docker details.

---

## Cleanup and disk usage

### Show Docker disk usage
```bash
docker system df
```

Shows how much disk space Docker is using.

### Show detailed disk usage
```bash
docker system df -v
```

Shows detailed size information for images, containers, and volumes.

### Remove stopped containers
```bash
docker container prune
```

Deletes all stopped containers.

### Remove dangling images
```bash
docker image prune
```

Removes unused dangling images.

### Remove all unused images
```bash
docker image prune -a
```

Removes all images not currently used by containers.

### Remove unused Docker data
```bash
docker system prune
```

Removes unused containers, networks, and cache.

### Remove more unused Docker data
```bash
docker system prune -a
```

Also removes unused images.

> Be careful with prune commands. They can delete useful resources.

---

## Copy files

### Copy from host to container
```bash
docker cp file.txt myweb:/tmp/file.txt
```

Copies a file from the host machine into the container.

### Copy from container to host
```bash
docker cp myweb:/tmp/file.txt ./file.txt
```

Copies a file from the container to the host.

---

## Help commands

### General help
```bash
docker help
```

Shows Docker help.

### Help for a specific command
```bash
docker run --help
```

Shows available options for a command.

---

## Common flags to remember

- `-d` = detached mode
- `-it` = interactive terminal
- `--name` = container name
- `-p` = port mapping
- `--rm` = automatically remove container after it exits
- `-e` = environment variable

Example:
```bash
docker run -d --name web -p 8080:80 nginx
```

---

## Must memorize

These are the commands you should know well for interviews and hands-on practice:

```bash
docker pull
docker images
docker run
docker ps
docker ps -a
docker stop
docker start
docker restart
docker rm
docker rmi
docker exec
docker inspect
docker top
docker stats
docker system df
docker container prune
docker image prune
docker system prune
docker cp
docker version
docker info
```

---

## Quick flow

### Image to running container
```bash
docker pull nginx
docker images
docker run -d --name myweb -p 7090:80 nginx
docker ps
```

### Enter container
```bash
docker exec -it myweb /bin/bash
```

### Stop and remove
```bash
docker stop myweb
docker rm myweb
docker rmi nginx
```

---

## Notes

- Containers are not virtual machines.
- A container depends on its image.
- If the main process inside the container exits, the container stops.
- Minimal images may not contain tools like `bash`, `ps`, or `ifconfig`.
- Use cleanup commands carefully, especially `prune`.