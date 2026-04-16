---
tags:
  - Docker
---
Good mental model:

- `docker` CLI sends requests to the Docker daemon (`dockerd`)
    
- `dockerd` manages Docker’s higher-level features like images, containers, networks, and volumes
    
- Docker uses **containerd**, a core container runtime daemon, to handle container lifecycle operations
    
- containerd uses a lower-level OCI runtime such as **runc** to actually create and start containers
    
- The Linux kernel provides the real isolation and resource control through namespaces, cgroups, mounts, and networking primitives
    

**Interview-friendly definition:**

> A container is a regular process isolated by the Linux kernel using namespaces and cgroups, with its own filesystem view, network stack, and process tree.

---

## High-level architecture

Main pieces:

- **Docker CLI**: the command-line client
    
- **Docker daemon (`dockerd`)**: receives Docker commands and manages higher-level Docker features
    
- **containerd**: a core container runtime daemon that manages image handling, filesystem snapshot preparation, container creation, task start/stop, and cleanup
    
- **runc**: a low-level OCI "open container initiative" runtime that creates and starts the isolated container process
    
- **Linux kernel**: provides namespaces, cgroups, mounts, and networking features
    
containerd can manage many containers over time because it is a daemon that listens for requests and tracks state.  
runc is typically invoked to create/start a container and then exits, rather than staying around as the main long-running manager.

- **containerd** is a long-running daemon. - **runc** is a low-level runtime executable invoked by containerd. - The application inside the container is the long-running process that actually runs after startup.
## Simplified flow

1. I run `docker run`
    
2. The Docker CLI sends the request to `dockerd`
    
3. `dockerd` checks whether the image exists locally
    
4. If needed, Docker pulls image content from a registry
    
5. `dockerd` asks containerd to perform the runtime work
    
6. containerd prepares the container filesystem from image layers using snapshots
    
7. containerd creates the container and starts its task
    
8. containerd uses `runc` to create the isolated process
    
9. The Linux kernel enforces isolation and resource limits
    
10. The application runs as the container’s main process
    

---

## What containerd is

**containerd** is a core container runtime daemon used by Docker and other container platforms.

Its job includes:

- pulling images
    
- managing image storage
    
- preparing filesystem snapshots
    
- creating containers
    
- starting tasks/processes
    
- stopping containers
    
- cleaning up resources
    

## Important distinction

containerd is not mainly a developer UX tool like Docker.

Instead:

- **Docker** provides the user-facing workflow and commands
    
- **containerd** provides the core runtime backend
    
- **runc** performs the low-level container process creation
    

## Easy mental model

- **Docker** = platform and workflow
    
- **containerd** = runtime manager
    
- **runc** = process launcher
    
- **kernel** = actual isolation and limits
    

This is a very good way to explain the stack in interviews.

---

## Important correction: container vs VM

Important interview point:

- A container is **not** a lightweight VM
    
- A container is an **isolated process** on the host
    
- Containers **share the host kernel**
    
- VMs virtualize hardware and run separate guest operating systems and kernels
    

## Good phrasing

> Containers isolate processes at the OS level, while VMs virtualize full machines.

This wording is better than saying “containers are like lightweight VMs.”

---

## What happens in `docker run`

`docker run` combines several steps:

- pull image if missing
    
- create container metadata
    
- prepare filesystem
    
- prepare networking
    
- apply resource limits
    
- start the main process
    

`docker run` can fail at different stages:

- image pull fails
    
- container creation fails
    
- mount setup fails
    
- network setup fails
    
- permission setup fails
    
- main process fails to start
    
- app starts, then crashes
    

## Troubleshooting mindset

Always ask:

- Did the **image** fail?
    
- Did the **runtime setup** fail?
    
- Did the **application process** fail after container start?
    

This gives a much stronger troubleshooting approach than trying random commands.

---

## Core Linux concepts behind Docker

## Namespaces

Namespaces provide isolation.

Types I should know:

- **PID namespace**: isolated process tree
    
- **Network namespace**: isolated interfaces, IPs, routes
    
- **Mount namespace**: isolated filesystem view
    
- **UTS namespace**: isolated hostname/domain name
    
- **IPC namespace**: isolated inter-process communication
    
- **User namespace**: isolated user/group IDs
    

## Important takeaway

- **Namespaces isolate**
    
- **Cgroups limit**
    

That is one of the best short interview lines to memorize.

---

## Cgroups

Cgroups control resource usage such as:

- CPU
    
- Memory
    
- Disk I/O
    
- Process counts
    

Why this matters:

- containers can be OOM-killed
    
- CPU can be throttled
    
- applications may fail because of limits, not only code bugs
    

## Good explanation

If a container crashes under load, the issue may be cgroup-enforced memory or CPU limits rather than a Docker bug.

---

## Filesystem and image layers

Docker images are built from **read-only layers**.

Key points:

- many Dockerfile instructions create new image layers
    
- containers add a **writable layer** on top of the image
    
- the writable layer is usually ephemeral
    
- deleting the container usually deletes that writable layer too
    

## Important distinction

I need to clearly separate:

- **Image layers**: read-only template
    
- **Container writable layer**: temporary runtime changes
    
- **Volumes**: Docker-managed persistent storage
    
- **Bind mounts**: host paths mounted into the container
    

## Common mistake

If data disappears after recreating a container, it probably lived in the writable container layer instead of a volume or bind mount.

---

## Volumes vs bind mounts

## Volumes

- managed by Docker
    
- good for persistent application data
    
- usually preferred for databases and app state
    

## Bind mounts

- map a host path directly into the container
    
- useful in development
    
- can create host-path dependency and permission issues
    

## Volumes vs bind mounts

## Why data inside the container layer is ephemeral

A container starts from read-only image layers plus a thin writable layer added at runtime. Anything the application writes inside the container filesystem, unless it is stored in a mounted volume or bind mount, goes into that writable layer.

That writable layer belongs to the specific container instance. If I remove the container, that layer is removed too, so the data is lost. That is why storing important data only inside the container filesystem is risky.

## When to use a volume

A volume is better when I want data to outlive the container and be managed in a Docker-friendly way.

Use volumes when:

- the data must persist even if the container is recreated
    
- the application stores state, such as a database, uploaded files, or application data
    
- I want cleaner separation between container lifecycle and data lifecycle
    
- I want a more portable and production-friendly storage approach
    

Good examples:

- PostgreSQL data directory
    
- MySQL data
    
- Redis persistence files
    
- application uploads
    
- shared persistent app data
    

Simple rule: if the data is important and should survive container replacement, I should usually use a volume.

## When bind mounts are better

A bind mount is better when I want the container to use an exact file or directory from the host.

Use bind mounts when:

- I am developing locally and want code changes on the host to appear immediately inside the container
    
- I want to mount a host configuration file into the container
    
- I want the container to write files directly to a known host path
    
- I need explicit control over the exact filesystem path being mounted
    

Good examples:

- mounting my source code into a dev container
    
- mounting an Nginx config file from the host
    
- mounting certificates stored on the host
    
- writing build artifacts to a host directory
    

Simple rule: bind mounts are usually better for code, config, and host-controlled files.

## Common permission problems with bind mounts

Bind mounts often cause permission issues because the container process and the host filesystem may not use the same user IDs, group IDs, or ownership rules.

Common problems:

- the container runs as a user that does not have permission to read or write the mounted host path
    
- files created by the container appear on the host with unexpected ownership
    
- a non-root process in the container cannot modify host files
    
- SELinux or host security settings block access
    
- the mounted file or directory already has restrictive permissions on the host
    

Example:

- On the host, a folder may be owned by user `1000`
    
- Inside the container, the app may run as user `1001`
    
- The container can mount the path successfully, but the app may get “permission denied” when trying to write
    

## Practical comparison

- **Container layer**: temporary runtime writes, lost when the container is removed
    
- **Volume**: persistent Docker-managed storage, usually best for app data
    
- **Bind mount**: direct host path mapping, usually best for development and host-managed files
    

## Good interview answer

> Data inside the container layer is ephemeral because it lives in the container’s writable layer, which is removed with the container. I use volumes for persistent application data, and I use bind mounts when I need the container to access a specific host path directly, especially in development. Bind mounts can cause permission issues when host file ownership does not match the user running inside the container.
    

---

## Networking basics

Docker creates software-defined networking for containers.

## Bridge

This is the default network mode for containers on a single Docker host.

How it works:

- the container gets its **own network namespace**
    
- the container usually gets its **own private IP**
    
- that IP is usually reachable by other containers on the same bridge network
    
- it is **not directly reachable from outside the host** unless I publish ports with `-p`
    

So in bridge mode:

- **container has its own IP**
    
- **host has its own IP**
    
- traffic from outside usually reaches the container through **port publishing/NAT**
    

## Host

In host mode, the container **does not get its own separate network namespace**.

How it works:

- the container shares the **host network stack**
    
- the container does **not get its own separate container IP**
    
- if the app listens on port 8080, it is listening directly on the host’s network
    

So in host mode:

- **container uses the host IP**
    
- **container does not have its own isolated container IP**
    
- there is no port mapping layer like normal bridge mode
    

This can improve simplicity or performance, but it reduces network isolation.

## Overlay

Overlay networking is used for **multi-host container communication**, such as in orchestration environments.

How it works:

- containers still get their **own network namespace**
    
- containers get their **own IP addresses** on the overlay network
    
- the overlay network connects containers across multiple hosts as if they are on one logical network
    

So in overlay mode:

- **container has its own IP**
    
- that IP belongs to the overlay network
    
- communication can happen across hosts
    

## None

In `none` mode:

- the container gets its **own network namespace**
    
- but Docker does **not configure network interfaces for normal communication**
    
- the container does **not have a usable external network connection**
    
- typically, only the loopback interface (`lo`) exists
    

So in `none` mode:

- **container does not have a normal network IP for communication**
    
- it cannot talk to other containers or the outside world unless networking is manually configured later
    

This is the mode you were thinking of when asking whether there is a network where the container has no IP.

---

## Simple summary

- **Bridge**: container has **its own private IP**
    
- **Host**: container uses the **host IP/network stack**
    
- **Overlay**: container has **its own IP** on a multi-host virtual network
    
- **None**: container has **no normal external network interface/IP**
    

---

## Easy interview phrasing

> In bridge mode, the container gets its own private IP. In host mode, it shares the host network stack and uses the host IP. In overlay mode, it gets its own IP on a multi-host virtual network. In none mode, it has no normal network connectivity.
    

## Very important distinction

- `EXPOSE` does **not** publish a port
    
- `EXPOSE` is metadata/documentation
    
- `-p hostPort:containerPort` actually publishes the port
    

## Common networking problems

A container can be healthy but unreachable because:

- the port is not published
    
- the app listens on `127.0.0.1` instead of `0.0.0.0`
    
- the wrong port is mapped
    
- DNS resolution fails
    
- the container is attached to the wrong network
    

---

## PID 1 and signal handling

Inside the container, the main process becomes **PID 1** in that PID namespace.

Why this matters:

- PID 1 has special signal-handling behavior
    
- PID 1 should reap zombie child processes
    
- badly written entrypoints can break shutdown behavior
    

## Practical impact

If `docker stop` hangs or the container does not shut down cleanly, one possible reason is that PID 1 is not handling signals correctly.

## Interview-ready explanation

> The container’s main process runs as PID 1, so signal handling and child reaping matter more than many people expect.

---

## Build process and cache

A Docker build is a sequence of instructions from the Dockerfile.

Important concepts:

- Docker reuses cached layers when possible
    
- changing an early layer invalidates later cache
    
- Dockerfile order affects build speed
    
- smaller images usually improve pull time and reduce attack surface
    

## Things I should know well

- **Multi-stage builds**
    
- **Layer caching**
    
- **Build context**
    
- `COPY` vs `ADD`
    
- `CMD` vs `ENTRYPOINT`
    

## Clarifications

## `COPY` vs `ADD`

- use `COPY` for normal file copying
    
- use `ADD` only when I specifically need its extra behavior
    

## `CMD` vs `ENTRYPOINT`

- `ENTRYPOINT` defines the main executable
    
- `CMD` provides default arguments or default command behavior
    

A lot of interview answers become better if I explain how they work together instead of treating them as competitors.

---

## Security basics

Containers are isolated, but they are not the same as hard isolation with separate kernels.

Important topics:

- avoid running as root when possible
    
- avoid `--privileged`
    
- understand Linux capabilities
    
- use least privilege
    
- do not bake secrets into images
    

## Practical interview angle

I should be able to explain:

- why root inside containers is risky
    
- why privileged containers are dangerous
    
- why shared-kernel architecture matters for security
    
- why secrets should be injected securely at runtime
    

---

## Practical troubleshooting workflow

When debugging Docker, follow a structured workflow.

## 1. Check container state

bash

`docker ps -a`

Look for:

- exited containers
    
- restarting containers
    
- unusual status codes
    

## 2. Check logs

bash

`docker logs <container>`

Look for:

- startup errors
    
- missing configuration
    
- permission problems
    
- application crashes
    

## 3. Inspect configuration

bash

`docker inspect <container>`

Check:

- command and entrypoint
    
- environment variables
    
- mounts
    
- ports
    
- network settings
    
- restart policy
    
- resource limits
    

## 4. Check the process model

Ask:

- Is the correct process running?
    
- Is it exiting immediately?
    
- Is the entrypoint script broken?
    
- Is PID 1 behaving correctly?
    

## 5. Check networking

Ask:

- Is port publishing correct?
    
- Is the app listening on the expected port?
    
- Is it bound to `0.0.0.0`?
    
- Is DNS working?
    
- Is the container on the correct network?
    

## 6. Check storage

Ask:

- Is the data stored in a volume, bind mount, or only in the writable layer?
    
- Is the mount target correct?
    
- Are file permissions blocking access?
    

## 7. Check resources

Ask:

- Is memory too low?
    
- Is CPU throttled?
    
- Is disk space full?
    
- Was the process OOM-killed?
    

---

## Common problem map

|Problem|Likely checks|
|---|---|
|Container exits immediately|Logs, entrypoint, CMD, app startup|
|App not reachable|Port publishing, bind address, network, DNS|
|Data disappears|Writable layer vs volume vs bind mount|
|Random crashes|Memory, CPU, OOM, disk, process limits|
|Slow builds|Layer cache, Dockerfile order, build context|
|Container won’t stop cleanly|PID 1, signal handling, entrypoint behavior|

---

## Most important interview topics

For junior/mid DevOps interviews, I should know:

- Docker architecture: CLI, `dockerd`, containerd, runc, kernel
    
- what containerd is and what it does
    
- container vs VM
    
- namespaces and cgroups
    
- images and layers
    
- writable container layer
    
- volumes vs bind mounts
    
- bridge, host, and overlay networking
    
- port publishing
    
- `EXPOSE` vs `-p`
    
- PID 1 behavior
    
- Dockerfile best practices
    
- multi-stage builds
    
- `CMD` vs `ENTRYPOINT`
    
- `COPY` vs `ADD`
    
- basic container security
    
- troubleshooting workflow
    

---

## Strong interview answer pattern

A strong answer should include:

- **what happens**
    
- **why it happens**
    
- **how I would debug it**
    

## Example

> If a container keeps restarting, I would first check its status, then logs, then inspect its config for environment variables, command, mounts, ports, restart policy, and resource limits. The root cause could be app startup failure, bad configuration, dependency failure, or OOM conditions.

This sounds much stronger than just listing commands.

---

## Best one-line explanations to memorize

- **A container is an isolated process on the host, not a VM.**
    
- **Namespaces isolate; cgroups limit.**
    
- **Docker uses containerd as its core runtime backend and runc to create container processes.**
    
- **Images are read-only layers; containers add a writable layer.**
    
- **Volumes persist data; the container writable layer is ephemeral.**
    
- **`EXPOSE` documents ports; `-p` publishes them.**
    
- **PID 1 behavior affects signal handling and clean shutdown.**
    
- **Good Docker troubleshooting means identifying which layer is failing: image, runtime setup, network, storage, resources, or app process.**
    

---

## Study order

Best learning order:

1. Docker architecture and `docker run`
    
2. What containerd is and how it fits under Docker
    
3. Namespaces and cgroups
    
4. Image layers and writable layer
    
5. Volumes and bind mounts
    
6. Networking and port publishing
    
7. PID 1 and process model
    
8. Dockerfile optimization and caching
    
9. Security basics
    
10. Troubleshooting workflow
    

---

## Personal goal

I should aim to explain Docker in terms of:

- processes
    
- kernel isolation
    
- resource limits
    
- storage layers
    
- networking
    
- runtime components
    
- debugging logic
    

If I can explain those clearly with one or two examples, I will stand out much more in junior/mid DevOps interviews.

---

## Fast recap

When I use Docker, I am not launching a mini-VM. I am asking Docker to coordinate a stack of components that eventually starts an isolated process on the host. Docker provides the user workflow, containerd manages core runtime operations, runc creates the isolated process, and the Linux kernel enforces isolation and limits.