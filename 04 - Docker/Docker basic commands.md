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

### Filter images
```bash
docker images --filter dangling=true
```

Shows only dangling (untagged) images.

### Remove an image
```bash
docker rmi nginx
```

Removes an image.

> You cannot remove an image if a container is still using it.

### Show image layer history
```bash
docker history nginx
```

Shows each layer of an image, its size, and the command that created it.

### Search Docker Hub
```bash
docker search nginx
```

Searches Docker Hub for images matching the keyword.

---

## Tags and Registry

### Tag an image
```bash
docker tag nginx myrepo/nginx:v1
```

Creates a new name/tag pointing to an existing image. Does not copy or rebuild the image.

- Format: `docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`
- If no tag is given, Docker defaults to `latest`
- A Docker image reference can only have **one colon** after the repository name

### Tag for a private registry
```bash
docker tag nginx myregistry:5000/myrepo/nginx:v1
```

Prepares the image for pushing to a private registry by including the host and port in the name.

### Login to a registry
```bash
docker login
```

Authenticates with Docker Hub. You will be prompted for a username and password.

### Login to a private registry
```bash
docker login myregistry:5000
```

Authenticates with a private registry.

### Logout
```bash
docker logout
```

Removes stored credentials for the registry.

### Push an image
```bash
docker push myrepo/nginx:v1
```

Uploads the image to Docker Hub or the specified registry.

> You must be logged in and the image name must match your registry namespace.

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

### Run with restart policy
```bash
docker run -d --name myweb --restart always nginx
```

Automatically restarts the container if it stops or the host reboots.

Restart policy options:

| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `always` | Always restart |
| `unless-stopped` | Restart unless manually stopped |
| `on-failure` | Restart only on non-zero exit code |

### Run with resource limits
```bash
docker run -d --name myweb --memory 512m --cpus 1.0 nginx
```

- `--memory` = maximum RAM the container can use
- `--cpus` = maximum CPU cores the container can use

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

### Format ps output
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Shows only selected columns in a readable table format.

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

### Pause and unpause a container
```bash
docker pause myweb
docker unpause myweb
```

`pause` freezes all processes inside the container without stopping it. `unpause` resumes them.

### Rename a container
```bash
docker rename myweb mywebserver
```

Renames an existing container.

### Remove a container
```bash
docker rm myweb
```

Removes a stopped container.

> A running container must be stopped before removal.

### Wait for a container to exit
```bash
docker wait myweb
```

Blocks until the container stops, then prints its exit code. Useful in scripts.

### Create an image from a container
```bash
docker commit myweb myrepo/myweb:v1
```

Saves the current state of a container as a new image. Useful for quick snapshots, but Dockerfiles are preferred for reproducible builds.

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

### Extract a specific field
```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' myweb
```

Extracts a specific value from the JSON output using Go template syntax.

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
- `-v` = mount a volume
- `--network` = connect to a specific network
- `--restart` = restart policy
- `--memory` = memory limit
- `--cpus` = CPU limit

Example:
```bash
docker run -d --name web -p 8080:80 nginx
```

```bash
docker run -e DB_HOST=localhost -e DB_PORT=5432 myapp
```

---

## Environment variables

### Set one variable
```bash
docker run -e APP_ENV=dev nginx
```

### Set multiple variables
```bash
docker run -e APP_ENV=dev -e DB_HOST=localhost myapp
```

### Use env file
```bash
docker run --env-file .env myapp
```

### Pass host variable through
```bash
export APP_ENV=dev
docker run -e APP_ENV nginx
```

---

## Must memorize

These are the commands you should know well for interviews and hands-on practice:

```bash
docker pull
docker images
docker history
docker search
docker tag
docker login
docker logout
docker push
docker run
docker ps
docker ps -a
docker stop
docker start
docker restart
docker pause
docker unpause
docker rename
docker kill
docker wait
docker commit
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

### Tag and push to registry
```bash
docker tag nginx myrepo/nginx:v1
docker login
docker push myrepo/nginx:v1
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
- `docker commit` is not recommended for production — use a Dockerfile instead.
- A Docker image reference supports only one colon for the tag: `registry/repo:tag`.
