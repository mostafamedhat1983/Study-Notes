---
tags:
  - Docker
  - Linux
---

To create a custom Docker image, you write a `Dockerfile` and build it with `docker build`.

A Docker image is usually created by:
- starting from a base image
- adding your application files
- installing required packages
- defining the startup command

---

## Main idea

A `Dockerfile` contains instructions for building an image.

Docker reads the file step by step and creates image layers.

Then you use:

```bash
docker build
```

to create the final image.

---

## Common Dockerfile instructions

### `FROM`
Sets the base image.

```dockerfile
FROM ubuntu:latest
```

Usually this is an official image from Docker Hub.

---

### `LABEL`
Adds metadata as key-value pairs.

```dockerfile
LABEL author="Mostafa Medhat"
LABEL project="nano"
```

Useful for project name, maintainer, version, or description.

---

### `RUN`
Executes commands during image build.

```dockerfile
RUN apt update && apt install apache2 -y
```

Used for:
- installing packages
- creating directories
- changing files
- preparing the image

> Each `RUN` instruction creates a new image layer, so combining related commands is usually better.

---

### `COPY`
Copies local files from the build context into the image.

```dockerfile
COPY app.tar.gz /tmp/
```

This is the preferred instruction for simple file copying.

---

### `ADD`
Adds local files, local archives, or remote files to the image.

```dockerfile
ADD nano.tar.gz /var/www/html/
```

`ADD` can do more than `COPY`, such as:
- automatically extracting local tar archives
- supporting remote URLs

> Use `COPY` unless you specifically need `ADD` features.

---

### `CMD`
Defines the default command that runs when the container starts.

```dockerfile
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

This is the default startup command for `docker run`.

---

### `ENTRYPOINT`
Defines the main executable for the container.

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```

If both `ENTRYPOINT` and `CMD` are present:
- `ENTRYPOINT` defines the executable
- `CMD` usually provides default arguments

---

### `EXPOSE`
Documents which port the application listens on inside the container.

```dockerfile
EXPOSE 80
```

This does **not** publish the port by itself.  
You still need `-p` in `docker run`.

---

### `VOLUME`
Marks a directory as a mount point for external storage.

```dockerfile
VOLUME ["/var/log/apache2"]
```

Useful for data or logs you may want to persist.
If you run the container without an explicit volume mapping, Docker can create an **anonymous volume** for that container path automatically.

---

### `ENV`
Sets environment variables inside the image.

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```

Useful for:
- non-interactive package installs
- application configuration
- runtime defaults

---

### `WORKDIR`
Sets the working directory for subsequent instructions like `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD`.

```dockerfile
WORKDIR /var/www/html
```

This means later commands will run relative to that directory.
If the `WORKDIR` path does not exist, Docker will generally **create it automatically** during the image build.

## Important detail

A small caveat is that the directory may be created with **root ownership**, even if you set `USER` earlier, so relying on implicit creation can sometimes cause permission issues.

## Best practice

It is safer to create it explicitly when permissions matter:

`RUN mkdir -p /app && chown appuser:appuser /app`
`WORKDIR /app`
`USER appuser`

That gives you more control over ownership and permissions.

---

### `USER`
Sets which user runs later commands or the container process.

```dockerfile
USER www-data
```

Useful for running containers as a non-root user.

---

### `ARG`
Defines build-time variables.

```dockerfile
ARG APP_VERSION=1.0
```

These values can be passed during build time.

Example:

```bash
docker build --build-arg APP_VERSION=2.0 -t myimage .
```

---

### `ONBUILD`
Adds instructions that run later when this image is used as a base image for another build.

```dockerfile
ONBUILD COPY . /app
```

Used mostly for reusable base images.

---

## Simple example Dockerfile

```dockerfile
FROM ubuntu:latest

LABEL author="Mostafa Medhat"
LABEL project="nano"

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install apache2 -y

WORKDIR /var/www/html

ADD nano.tar.gz /var/www/html/

VOLUME ["/var/log/apache2"]

EXPOSE 80

CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

---

## Why `ENV DEBIAN_FRONTEND=noninteractive` was needed

Some package installations ask interactive questions during build, which can break Docker builds.

Setting:

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```

helps prevent interactive prompts during package installation in Debian/Ubuntu-based images.

---

## Build context

When you run:

```bash
docker build -t myimage .
```

the `.` means the current directory is the **build context**.

Docker sends files from this context to the builder.

That means:
- files outside the build context cannot be copied with `COPY` or `ADD`
- unnecessary files in the context can slow down the build
- `.dockerignore` helps reduce that context

---

## .dockerignore

A `.dockerignore` file excludes files and directories from the Docker build context.

This helps with:
- faster builds
- smaller build context
- avoiding unnecessary or sensitive files
- better cache behavior

Example:

```dockerignore
.git
node_modules
.env
*.log
tmp/
```

### Notes
- `.dockerignore` works like `.gitignore`, but for Docker build context.
- It should be placed in the root of the build context.
- Ignored files are not sent to the Docker builder.

---

## Build cache

Docker builds images layer by layer.

For each Dockerfile instruction, Docker may reuse a cached layer if nothing relevant changed.

This makes rebuilds much faster.

### Important rule
If one layer changes, the layers after it usually need to be rebuilt too.

### Good practice
Put instructions that change less often earlier in the Dockerfile.

Example:
- install packages first
- copy application files later

This improves cache reuse.

### Cache-friendly example

```dockerfile
FROM ubuntu:latest

RUN apt update && apt install -y apache2

ADD nano.tar.gz /var/www/html/

CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

If only `nano.tar.gz` changes, Docker can still reuse earlier cached layers.

---

## Build the image

### Build from current directory
```bash
docker build -t nanoimage .
```

This builds the image using the `Dockerfile` in the current directory.

### Build with a tag
```bash
docker build -t nanoimage:v2 .
```

The tag is usually used for versioning.

---

## Check built images

```bash
docker images
```

Shows local images.

---

## Run the built image

```bash
docker run -d --name nanowebsite -p 9080:80 nanoimage:v2
```

This starts the container in detached mode and maps host port `9080` to container port `80`.

---

## Troubleshooting build or runtime issues

Useful commands:

```bash
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
docker inspect <image>
```

Use these when:
- the build fails
- the container exits immediately
- the startup command is wrong
- the wrong port is exposed
- required files are missing

---

## Publish image to Docker Hub

To push an image to Docker Hub, the image name should include your Docker Hub username or namespace.

Example:

```bash
docker build -t yourusername/nanoimage:v2 .
```

Then log in and push it.

---

## Login to Docker Hub

### Interactive login
```bash
docker login
```

### Recommended: login with access token
```bash
echo "$DOCKER_TOKEN" | docker login -u yourusername --password-stdin
```

Using an access token is more secure than using your password directly.

---

## Push image

```bash
docker push yourusername/nanoimage:v2
```

This uploads the tagged image to Docker Hub.

---

## Pull and run published image

Anyone who has access to the image can pull it:

```bash
docker pull yourusername/nanoimage:v2
```

Then run it:

```bash
docker run -d --name nanowebsite -p 9080:80 yourusername/nanoimage:v2
```

---

## Important notes

- `EXPOSE` documents the container port, but does not publish it by itself.
- `COPY` is usually safer and clearer than `ADD`.
- `ADD` is useful when you want automatic archive extraction.
- `RUN` executes at image build time.
- `CMD` runs when the container starts.
- `ARG` is for build time, while `ENV` is for environment variables inside the image.
- `WORKDIR` affects later Dockerfile instructions, not just interactive shell usage.
- The build context is usually the current directory passed as `.` to `docker build`.

---

## Must memorize

```bash
docker build -t myimage .
docker build -t myimage:v1 .
docker images
docker run -d --name mycontainer -p 8080:80 myimage:v1
docker logs <container>
docker inspect <image>
docker inspect <container>
docker login
docker push yourusername/myimage:v1
docker pull yourusername/myimage:v1
```

---

## Key ideas

- A `Dockerfile` is the recipe for building a Docker image.
- Docker images are built in layers.
- You usually start from an official base image.
- You customize the image with instructions like `RUN`, `COPY`, `ENV`, and `CMD`.
- After building, you can run the image locally or push it to a registry like Docker Hub.

## Advanced image hardening

### Distroless images

A distroless image contains only the application and its runtime dependencies.

It usually does **not** include:
- a shell
- a package manager
- common debugging tools

This makes the image smaller and reduces the attack surface.

Distroless images are usually created with a **multi-stage build**:
- one stage to build the application
- one final minimal stage to run it

Example:

```dockerfile
# Build stage
FROM golang:1.18 AS build
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage
FROM gcr.io/distroless/static-debian12
COPY --from=build /app/myapp /myapp
CMD ["/myapp"]
```

### Notes
- Distroless images are best for production.
- They are harder to debug interactively because they usually do not include `sh` or `bash`.
- Use JSON-array form for `CMD` or `ENTRYPOINT`.

---

### Docker Hardened Images

Docker Hardened Images (DHI) are Docker-maintained minimal and security-focused images.

They are designed to:
- reduce vulnerabilities
- reduce attack surface
- improve supply-chain security
- provide production-ready hardened base images

Docker Hardened Images follow a distroless or minimal-image philosophy and are intended for secure production workloads.

### Notes
- Good for security-focused production environments.
- Often used when compliance, CVE reduction, and provenance matter.
- They trade some debugging convenience for stronger security.

---

## Image management commands

After building an image, you often need to manage it locally or publish it to a registry.

### Pull an image

```bash
docker image pull ubuntu:20.04
```

Downloads an image from a registry such as Docker Hub.

If the image is not available locally, Docker will pull it automatically in many cases when you run a container.

---

### List local images

```bash
docker image ls
```

Shows images stored on your machine.

Useful columns include:
- repository
- tag
- image ID
- created time
- size

---

### Inspect an image

```bash
docker image inspect ubuntu:20.04
```

Displays detailed JSON metadata about the image.

This can help you check:
- environment variables
- default command
- entrypoint
- exposed ports
- layers
- labels

---

### Tag an image

```bash
docker image tag myimage:latest mostafamedhat1983/myimage:v1
```

Creates another reference for the same image.

This is commonly used before pushing an image to Docker Hub or another registry.

---

### Push an image

```bash
docker image push mostafamedhat1983/myimage:v1
```

Uploads the tagged image to a registry.

Before pushing, you usually need to log in:

```bash
docker login
```

---

### Remove an image

```bash
docker image rm ubuntu:20.04
```

Removes an image from the local machine.

If a container is still using the image, Docker may refuse to remove it until the container is removed first.

---

### Useful notes

- `docker image ls` shows local images.
- `docker image inspect` shows detailed metadata.
- `docker image tag` adds another name or tag to an image.
- `docker image push` uploads an image to a registry.
- `docker image rm` removes a local image.
- Images can be identified by name, tag, or image ID.

---

## Image history and commit

Docker images are built in layers.

Each significant image-building step usually creates a new layer, which is one reason Docker builds can reuse cache efficiently.

### Show image history

```bash
docker image history myimage:latest
```

This shows the history of the image layers.

It helps you understand:
- which instructions created layers
- image size growth
- whether large layers were added unexpectedly

This is useful when:
- troubleshooting large image size
- understanding how the image was built
- reviewing Dockerfile impact on layers

---

### Create an image from a container

```bash
docker container commit mycontainer myimage:snapshot
```

This creates a new image from the current state of a container.

This means Docker captures the container filesystem changes and saves them as a new image.

### Example

```bash
docker run -it --name testcontainer ubuntu bash
```

Make some changes inside the container, then from another terminal or after exiting:

```bash
docker container commit testcontainer ubuntu-custom:v1
```

Now you have a new image called `ubuntu-custom:v1`.

---

### When `commit` can be useful

`docker container commit` can be useful for:
- quick experiments
- debugging
- taking a snapshot of a modified container
- temporary testing

---

### Why Dockerfiles are usually better

Even though `docker container commit` works, it is usually **not the best long-term approach**.

A Dockerfile is usually better because it is:
- reproducible
- version-controlled
- easier to review
- easier to automate in CI/CD
- easier for other people to understand

So:

- use `docker container commit` for quick temporary work
- use a Dockerfile for real projects

---

### Simple rule

- `docker build` is the normal way to create images for real projects.
- `docker container commit` is more of a manual snapshot tool.

---
---

## Image best practices

When building Docker images, the goal is not only to make them work, but also to make them:
- smaller
- faster to build
- easier to maintain
- more secure
- more reproducible

---

### Use a trusted and minimal base image

Choose a base image from a trusted source such as:
- official images
- verified publisher images
- well-maintained vendor images

Also prefer a base image that is as small as possible for your use case.

Why this matters:
- smaller images download faster
- smaller images usually contain fewer packages
- fewer packages usually mean fewer vulnerabilities
- minimal images reduce attack surface

Examples:
- `alpine`
- slim runtime images
- distroless images for production runtime stages

---

### Avoid relying only on `latest`

Using `latest` can make builds less predictable because the image behind that tag may change over time.

Instead, prefer versioned tags such as:

```dockerfile
FROM node:20-alpine
```

instead of:

```dockerfile
FROM node:latest
```

This makes builds more stable and easier to reproduce.

---

### Pin image versions more strictly when needed

For stronger reproducibility, you can pin an image to a digest.

Example:

```dockerfile
FROM alpine:3.21@sha256:<digest>
```

This ensures Docker always uses the exact same image version, even if the tag later points to something new.

This is especially useful in:
- production builds
- security-sensitive environments
- CI/CD pipelines

---

### Use multi-stage builds

If your application needs build tools such as:
- Maven
- Gradle
- npm
- Go compiler

do not keep those tools in the final runtime image unless they are actually needed.

Use multi-stage builds to:
- build the artifact in one stage
- copy only the final output into the runtime stage

This keeps the final image smaller and cleaner.

---

### Keep the final image minimal

Only include what the application needs to run.

Avoid keeping unnecessary things in the final image, such as:
- build tools
- package caches
- source code if not needed
- temporary files
- debugging tools in production images

A smaller runtime image is usually better for:
- security
- performance
- portability

---

### Order Dockerfile instructions for better caching

Docker reuses cached layers when possible.

A good rule is:
- put stable instructions earlier
- put frequently changing instructions later

For example:
- install system packages first
- copy dependency files next
- copy application source code later

This improves rebuild speed because Docker can reuse more cached layers.

---

### Use `.dockerignore`

A `.dockerignore` file prevents unnecessary files from being sent in the build context.

This helps by:
- reducing build time
- reducing context size
- avoiding accidental inclusion of secrets
- improving cache efficiency

Example:

```dockerignore
.git
node_modules
.env
*.log
tmp
target
```

---

### Run as non-root when possible

If possible, avoid running the container as root.

Use the `USER` instruction to run the application with a non-root user.

Example:

```dockerfile
RUN useradd -m appuser
USER appuser
```

This reduces the impact of a potential container compromise.

---

### Rebuild images regularly

Even if your Dockerfile does not change, the packages in your base image may receive updates for:
- security patches
- bug fixes
- dependency improvements

So rebuilding images regularly helps keep them updated.

You may also use:

```bash
docker build --no-cache -t myimage .
```

when you want to force a fresh rebuild instead of reusing cached layers.

---

### Prefer Dockerfiles over manual image creation

Although `docker container commit` can create an image from a running container, Dockerfiles are usually better because they are:
- reproducible
- version-controlled
- easier to review
- easier to automate

Use `commit` mainly for:
- quick experiments
- temporary snapshots
- debugging

Use Dockerfiles for real projects.

---

### Simple rule of thumb

A good Docker image should be:
- minimal
- predictable
- reproducible
- secure
- easy to rebuild

---

### Must remember

- Use trusted and minimal base images.
- Prefer fixed versions over `latest`.
- Use multi-stage builds when a build step is required.
- Keep the final image as small as possible.
- Use `.dockerignore` to reduce build context.
- Order Dockerfile instructions to improve caching.
- Run as non-root when possible.
- Prefer Dockerfiles over manual image snapshots.
### Must memorize

``` bash

docker image pull ubuntu:20.04
docker image ls
docker image inspect ubuntu:20.04
docker image tag myimage:latest mostafamedhat1983/myimage:v1
docker image push mostafamedhat1983/myimage:v1
docker image rm ubuntu:20.04
docker image history myimage:latest
docker container commit mycontainer myimage:snapshot

```