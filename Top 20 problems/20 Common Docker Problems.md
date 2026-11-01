---
tags:
  - Docker
  - Troubleshooting
---

Knowing these problems helps you isolate container, image, network, and storage failures.

---

## Main idea

Most Docker problems involve daemon access, image builds, process lifecycle, networking, storage, or resources.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Cannot connect to the Docker daemon

- **Why it happens:** The daemon is stopped or the CLI targets the wrong endpoint.
- **Quick sign:** docker info cannot reach the server.
- **Solution:** Check the active context and daemon; Docker Desktop and rootless installations use different service management.
- **Useful commands:** `docker context ls`, `docker info`, `systemctl status docker`

---

## 2. Docker socket permission denied

- **Why it happens:** The user lacks access to the daemon socket.
- **Quick sign:** Permission denied on docker.sock.
- **Solution:** Use approved access controls or rootless Docker; docker-group membership effectively grants root-level power. Never fix this with chmod 666.
- **Useful commands:** `id`, `ls -l /var/run/docker.sock`, `docker context inspect`

---

## 3. Image pull denied or tag not found

- **Why it happens:** The image reference is wrong or registry access is missing.
- **Quick sign:** Pull reports denied or manifest unknown.
- **Solution:** Verify registry, repository, tag, and authentication; do not expose registry credentials.
- **Useful commands:** `docker pull <image>:<tag>`, `docker login <registry>`

---

## 4. Build cannot find a COPY source

- **Why it happens:** The file is outside the build context or excluded by .dockerignore.
- **Quick sign:** COPY fails with missing-file or checksum errors.
- **Solution:** Inspect the context path and ignore rules; COPY sources are relative to the context, not the Dockerfile directory.
- **Useful commands:** `docker build --progress=plain -f path/Dockerfile .`, `cat .dockerignore`

---

## 5. Container exits immediately

- **Why it happens:** The main process finishes or crashes.
- **Quick sign:** The container appears only in docker ps -a.
- **Solution:** Inspect logs and exit status; keep the intended application running in the foreground rather than hiding failure with a sleep loop.
- **Useful commands:** `docker ps -a`, `docker logs <container>`, `docker inspect <container>`

---

## 6. Restart loop

- **Why it happens:** A restart policy repeatedly starts a failing process.
- **Quick sign:** The container continually shows Restarting.
- **Solution:** Diagnose configuration, dependencies, and exit status; fix the cause rather than just increasing restart frequency.
- **Useful commands:** `docker logs --tail 100 <container>`, `docker inspect <container>`

---

## 7. Published port is already allocated

- **Why it happens:** Another process or container owns the requested host port.
- **Quick sign:** Startup reports address already in use.
- **Solution:** Identify the listener, then change the mapping or stop the conflicting service deliberately.
- **Useful commands:** `docker ps`, `ss -lntp`, `docker compose config`

---

## 8. Application is unreachable from the host

- **Why it happens:** The port is unpublished, mapped incorrectly, blocked, or the app listens only on container loopback.
- **Quick sign:** The app works inside the container but not through the host.
- **Solution:** Check application binding and publishing; EXPOSE alone does not publish a port.
- **Useful commands:** `docker port <container>`, `docker logs <container>`, `curl -v http://127.0.0.1:8080`

---

## 9. Container cannot reach another container

- **Why it happens:** Containers use different networks or the wrong endpoint.
- **Quick sign:** Service calls time out or refuse connections.
- **Solution:** Put communicating services on an appropriate shared network and use service names with container ports; localhost refers to the calling container.
- **Useful commands:** `docker network ls`, `docker network inspect <network>`, `docker compose ps`

---

## 10. DNS resolution fails inside a container

- **Why it happens:** Resolver configuration, upstream DNS, or network attachment is wrong.
- **Quick sign:** IP connectivity works while hostname lookup fails.
- **Solution:** Compare host and container DNS and verify network membership; use diagnostic tools available in the image.
- **Useful commands:** `docker exec <container> cat /etc/resolv.conf`, `docker network inspect <network>`

---

## 11. Bind mount hides application files

- **Why it happens:** Mounting onto a populated image directory obscures its original contents.
- **Quick sign:** Built files disappear when the mount is enabled.
- **Solution:** Inspect source and target paths; mount only intended data or development directories.
- **Useful commands:** `docker inspect <container>`, `docker compose config`

---

## 12. Mounted directory permission denied

- **Why it happens:** Container UID/GID, host permissions, or security labels do not permit access.
- **Quick sign:** The application cannot write mounted data.
- **Solution:** Align ownership and least-privilege permissions; check SELinux labels where applicable instead of using chmod 777.
- **Useful commands:** `docker exec <container> id`, `ls -ln /host/path`, `getenforce`

---

## 13. Data disappears after recreation

- **Why it happens:** Persistent data was stored only in the writable container layer.
- **Quick sign:** Recreating a container resets application data.
- **Solution:** Use named volumes or appropriate bind mounts and tested backups; avoid down -v unless deletion is intended.
- **Useful commands:** `docker volume ls`, `docker inspect <container>`, `docker volume inspect <volume>`

---

## 14. Docker host runs out of disk

- **Why it happens:** Images, build cache, writable layers, volumes, or logs grow.
- **Quick sign:** Pulls, builds, or writes fail with no space left.
- **Solution:** Identify the consumer, configure log retention, and remove only verified unused objects; pruning can delete needed data.
- **Useful commands:** `docker system df -v`, `df -h`, `docker inspect <container>`

---

## 15. Container is OOM-killed

- **Why it happens:** The workload exceeds available memory or its limit.
- **Quick sign:** State.OOMKilled is true; exit code 137 alone is not proof of OOM.
- **Solution:** Inspect usage and limits, then fix memory growth or adjust capacity.
- **Useful commands:** `docker stats --no-stream`, `docker inspect --format '{{.State.OOMKilled}}' <container>`

---

## 16. Container is slow

- **Why it happens:** CPU throttling, memory pressure, storage I/O, or application bottlenecks limit performance.
- **Quick sign:** Latency increases under load.
- **Solution:** Compare container and host metrics before changing resource limits.
- **Useful commands:** `docker stats`, `docker inspect <container>`, `vmstat 1`

---

## 17. Old code still runs after a change

- **Why it happens:** The container uses an old image or a mount masks the rebuilt code.
- **Quick sign:** Source changes are missing at runtime.
- **Solution:** Verify build context, image identity, mounts, and recreation; cache bypass is a diagnostic tool, not the default fix.
- **Useful commands:** `docker compose images`, `docker compose up -d --build <service>`, `docker inspect <container>`

---

## 18. Compose starts dependencies too early

- **Why it happens:** Process startup does not mean the service is ready.
- **Quick sign:** The app initially fails to connect to a database.
- **Solution:** Add meaningful health checks and supported service_healthy dependency conditions; retain application retries.
- **Useful commands:** `docker compose ps`, `docker compose logs --tail 100`, `docker compose config`

---

## 19. Container becomes unhealthy

- **Why it happens:** The health check fails due to app failure or an incorrect probe.
- **Quick sign:** Container state is running but health is unhealthy.
- **Solution:** Inspect probe output and verify command availability, endpoint, and timing; standalone Docker does not restart a container solely because it is unhealthy.
- **Useful commands:** `docker inspect --format '{{json .State.Health}}' <container>`, `docker logs <container>`

---

## 20. Exec format error

- **Why it happens:** The executable architecture is incompatible or a script has an invalid execution format.
- **Quick sign:** Startup fails before application initialization.
- **Solution:** Check platform compatibility and script shebang/line endings; select a supported image platform.
- **Useful commands:** `uname -m`, `docker image inspect <image>`, `docker buildx imagetools inspect <image>`

---

## Common mistakes

- treating EXPOSE as port publishing
- using localhost to reach another container
- deleting volumes during routine troubleshooting
- granting unrestricted socket access
- assuming every image includes bash or curl

---

## Good practices

- inspect logs and state before recreating
- use reproducible image references
- persist and back up important data
- configure resource and logging limits
- avoid privileged containers as a generic fix

---

## Must memorize

```bash
docker ps -a
docker logs --tail 100 <container>
docker inspect <container>
docker stats --no-stream
docker system df -v
docker network inspect <network>
docker compose config
docker compose ps
docker compose logs --tail 100
```

---

## Key ideas

- A container lives as long as its main process.
- Images, containers, and volumes have different lifecycles.
- Host ports and container ports are different.
- Diagnose before pruning or recreating.
