---
tags:
  - Docker
  - Linux
---
# Docker Logs

Docker logs help you view the output of a container's main process.  
They are mainly used for troubleshooting when a container fails to start, exits unexpectedly, or behaves incorrectly.

---

## What are container logs?

Container logs are usually the **standard output (stdout)** and **standard error (stderr)** of the main process running inside the container.

When you start a container, Docker runs the image's configured startup process.  
Any output produced by that process becomes the container logs.

---

## Main command

### Show logs of a container
```bash
docker logs <container>
```

Example:
```bash
docker logs myweb
```

This displays the logs of the specified container.

You can use either:

- container name
- container ID

---

## Why logs matter

Logs are one of the first things to check when:

- a container exits immediately
- a container fails to start
- the application inside the container throws an error
- the startup command is misconfigured
- required environment variables are missing

---

## Detached vs foreground mode

### Run in detached mode
```bash
docker run -d nginx
```

The container runs in the background, so you do not see its output directly in your shell.

To view the output later:

```bash
docker logs <container>
```

### Run in foreground mode
```bash
docker run nginx
```

The container runs in the foreground, so its output appears directly in your terminal.

If you stop it with:

```bash
Ctrl + C
```

the main process is killed, and the container stops.

---

## Example with NGINX

### Pull image
```bash
docker pull nginx
```

### Run container in detached mode
```bash
docker run -d -P nginx
```

### Check running containers
```bash
docker ps
```

### View logs
```bash
docker logs <container>
```

### Notes
- `-d` = detached mode
- `-P` = publish all exposed container ports to random host ports

---

## Logs and startup process

When you run a container, Docker starts the image's configured startup command.

You may also see these terms in image metadata:

- **ENTRYPOINT**
- **CMD**

These define what command or script runs when the container starts.

To view image or container metadata:

```bash
docker inspect <image_or_container>
```

Logs usually come from the process started by `ENTRYPOINT` and `CMD`.

---

## Troubleshooting workflow

If a container does not stay running, use this flow:

### 1. Check running containers
```bash
docker ps
```

### 2. Check all containers
```bash
docker ps -a
```

This helps you find containers that started and then exited.

### 3. View logs
```bash
docker logs <container>
```

This is often the fastest way to identify the problem.

### 4. Inspect metadata if needed
```bash
docker inspect <container>
```

This can help you understand:

- startup command
- environment variables
- port mappings
- mount points
- exit details

---

## Example: failed MySQL container

### Run MySQL container
```bash
docker run -d mysql:5.7
```

### Check running containers
```bash
docker ps
```

You may not see it running.

### Check all containers
```bash
docker ps -a
```

You may see the container in **Exited** state.

### Check logs
```bash
docker logs <container>
```

The logs may show that MySQL requires a root password environment variable.

### Start again with required variable
```bash
docker run -d -e MYSQL_ROOT_PASSWORD=mypassword mysql:5.7
```

This is a common real-world use of `docker logs`.

---

## Useful log options

### Follow logs in real time
```bash
docker logs -f <container>
```

This streams logs continuously, similar to `tail -f`.

### Show only the last lines
```bash
docker logs --tail 50 <container>
```

Shows only the last 50 log lines.

### Show timestamps
```bash
docker logs -t <container>
```

Adds timestamps to log lines.

### Combine options
```bash
docker logs -f --tail 20 -t <container>
```

Follow the last 20 log lines with timestamps.

---

## Common troubleshooting commands

### List running containers
```bash
docker ps
```

### List all containers
```bash
docker ps -a
```

### Show logs
```bash
docker logs <container>
```

### Inspect container details
```bash
docker inspect <container>
```

### Start container again
```bash
docker start <container>
```

### Stop container
```bash
docker stop <container>
```

### Remove stopped container
```bash
docker rm <container>
```

---

## Common problems logs can reveal

- missing environment variables
- invalid startup command
- application crash on startup
- port binding issues
- permission errors
- missing files or directories
- database initialization errors
- bad configuration inside the container

---

## Key idea

If a container starts and immediately exits, the main process likely failed.  
`docker logs` helps you see the error output from that process.

That is why `docker logs` is one of the most important Docker troubleshooting commands.

---

## Must memorize

```bash
docker logs <container>
docker logs -f <container>
docker logs --tail 50 <container>
docker logs -t <container>
docker ps
docker ps -a
docker inspect <container>
```

---

## Quick troubleshooting flow

```bash
docker run -d mysql:5.7
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
```

If logs show a missing variable:

```bash
docker run -d -e MYSQL_ROOT_PASSWORD=mypassword mysql:5.7
```

---

## Notes

- Logs come from the container's main process output.
- Foreground containers show logs directly in the terminal.
- Detached containers require `docker logs` to view output.
- If the main process exits, the container stops.
- `docker logs` is one of the first commands to use for troubleshooting.