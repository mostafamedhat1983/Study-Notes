---
tags:
  - Docker
  - Linux
---

Docker Compose is a tool used to define and run **multi-container applications**.

Instead of running many long `docker run` commands manually, you describe all containers, networks, ports, volumes, and environment variables in a single YAML file, then start everything together with one command.

---

## Why Docker Compose

When an application has multiple containers, managing them one by one becomes difficult.

For example, an application may need:
- a web container
- a database container
- a cache container
- a messaging container

If you run all of them manually with `docker run`, there is a high chance of:
- making mistakes in ports
- forgetting environment variables
- forgetting volume mappings
- creating network issues
- starting containers in the wrong order

Docker Compose solves this by letting you define everything in one file.

---

## Main idea

With Docker Compose, you create a file such as:

```bash
docker-compose.yml
```

or

```bash
compose.yaml
```

Then you run:

```bash
docker compose up
```

Docker Compose reads the file and creates:
- containers
- networks
- volumes
- images if build instructions exist

This is a **declarative** way to manage multi-container apps.

---

## Why it is useful

Docker Compose gives several benefits:

- less human error
- easier container management
- one command to start everything
- one command to stop everything
- configuration stored as code
- easy to version-control in Git
- useful for development, testing, and small environments

It is similar to how Vagrant manages VMs using a config file.

---

## Docker Compose installation note

Docker Compose used to be a separate tool, but modern Docker uses **Docker Compose v2** as a Docker CLI plugin. On Linux, Docker documents installation of the Compose plugin separately from Docker Engine, and the modern command format is `docker compose` with a space rather than the old standalone `docker-compose` form. 

### Check if Compose is installed
```bash
docker compose version
```

### Modern command style
```bash
docker compose up
```

### Older command style
```bash
docker-compose up
```

Both may still appear in tutorials, but `docker compose` is the current standard.

---

## Compose file basics

A Docker Compose file is written in YAML format.

Main sections commonly include:
- `services`
- `volumes`
- `networks`

### Example structure
```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"

  redis:
    image: "redis:alpine"
```

### Meaning
- `services` = containers
- `build` = build image from Dockerfile
- `image` = use an existing image
- `ports` = map host port to container port

---

## Common Compose keys

### `services`
Defines the containers of the application.

### `build`
Builds an image from a Dockerfile.

```yaml
build: .
```

This means: build using the Dockerfile in the current directory.

### `image`
Uses an existing image from Docker Hub or another registry.

```yaml
image: redis:alpine
```

### `ports`
Maps host ports to container ports.

```yaml
ports:
  - "8000:5000"
```

This maps:
- host port `8000`
- to container port `5000`

### `volumes`
Mounts storage into containers.

```yaml
volumes:
  - .:/code
```

This is a bind mount:
- current host directory
- mapped to `/code` inside the container

You can also use named volumes.

### `environment`
Sets environment variables inside the container.

```yaml
environment:
  FLASK_ENV: development
```

### `depends_on`
Defines startup dependency order between services. Docker Compose starts and stops services in dependency order, but if your app needs a dependency to be truly ready, you may still need `healthcheck` plus `depends_on` conditions. 

### `restart`
Defines restart behavior for the container.

Example:

```yaml
restart: unless-stopped
```

Useful for services that should restart automatically. Docker will try to restart the container if it exits or if Docker starts again, **unless you explicitly stopped that container yourself**

---

## Flask + Redis example

In the lecture example, there are two services:
- `web`
- `redis`

The `web` service is built from a Dockerfile.
The `redis` service uses the official Redis image.

### Example compose file
```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine"
```

This means:
- build the web image from the current directory
- run Redis from the official image
- expose the Flask app on port `8000` of the host

---

## Example project files

The example application uses:
- `app.py`
- `requirements.txt`
- `Dockerfile`
- `compose.yaml`

### `app.py`
A simple Flask application that connects to Redis and counts page visits.

### `requirements.txt`
Contains Python dependencies such as:
- Flask
- Redis

### `Dockerfile`
Builds the Flask web image.

### `compose.yaml`
Runs both the web container and Redis together.

---

## Example Dockerfile

```dockerfile
FROM python:3.10-alpine

WORKDIR /code

ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

EXPOSE 5000

COPY . .

CMD ["flask", "run"]
```

### What it does
- starts from a Python base image
- sets `/code` as working directory
- sets Flask environment variables
- copies dependencies file
- installs dependencies
- exposes port `5000`
- copies project files
- starts Flask

---

## Build and start services

### Start in foreground
```bash
docker compose up
```

This starts all services and shows logs in the terminal.

### Start in background
```bash
docker compose up -d
```

This runs the containers in detached mode.

---

## What happens during `docker compose up`

When you run:

```bash
docker compose up
```

Docker Compose may:
- create a default network for the project,
- build images for services that use `build`,
- pull images for services that use `image`,
- create volumes,
- create and start containers. [web:243][web:244][web:249]

By default, services in the same Compose project can talk to each other on the project network using **service names** as hostnames, so the web app can usually connect to Redis using `redis` as the hostname. [web:249][web:257]

---

## Accessing the app

If the Compose file maps:

```yaml
ports:
  - "8000:5000"
```

and the app listens on port `5000` inside the container, then you access it from the host on:

```bash
http://localhost:8000
```

If you are using a cloud VM or EC2 instance, you must also allow that host port in the firewall or security group.

---

## Useful Compose commands

### Start services
```bash
docker compose up
```

### Start in detached mode
```bash
docker compose up -d
```

### Stop services
```bash
docker compose stop
```

### Stop and remove containers, networks, and default resources
```bash
docker compose down
```

### Show running services
```bash
docker compose ps
```

### Show processes
```bash
docker compose top
```

### Show logs
```bash
docker compose logs
```

### Follow logs
```bash
docker compose logs -f
```

### Rebuild and start
```bash
docker compose up --build
```

---

## `up` vs `down`

### `docker compose up`
- creates and starts services
- may build images if needed

### `docker compose down`
- stops containers
- removes containers
- removes the default network

By default, named volumes are not removed unless you explicitly ask for it.

### Remove volumes too
```bash
docker compose down -v
```

Use this carefully because it can remove persisted data.

---

## Volumes in Compose

Compose supports both bind mounts and named volumes.

### Bind mount example
```yaml
services:
  web:
    volumes:
      - .:/code
```

This maps:
- current host directory
- to `/code` inside the container

This is useful for development because file changes on the host can appear inside the container.

### Named volume example
```yaml
services:
  web:
    volumes:
      - logvolume01:/var/log/apache2

volumes:
  logvolume01:
```

This creates a named volume called `logvolume01` and mounts it inside the container.

---

## Service communication

Containers started by the same Compose project are connected through the same default network.

That means one service can talk to another service by its **service name**.

Example:
- web service can connect to Redis using hostname `redis`

This is one of the biggest benefits of Compose.

---

## Updating the app with bind mounts

If you use a bind mount like:

```yaml
volumes:
  - .:/code
```

then changes made to files in the host project directory can appear immediately inside the container.

This is very useful in development workflows.

---

## Build vs image in Compose

### `build`
Use this when you want Compose to build an image from a Dockerfile.

```yaml
build: .
```

### `image`
Use this when you want Compose to use an already existing image.

```yaml
image: redis:alpine
```

You can also use both together in some workflows, but for beginners:
- `build` = build locally
- `image` = pull/use existing image

---

## Suggested additions for real projects

For real applications, a Compose file often also includes:

- `depends_on`
- `restart`
- `environment`
- `healthcheck`
- custom `networks`
- named `volumes`

A more practical example:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:alpine
    restart: unless-stopped
```

If the application truly requires Redis to be ready before startup, use `healthcheck` with dependency conditions instead of relying only on simple startup order. [web:249]

---

## A fuller example

```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_ENV: development
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:alpine
    restart: unless-stopped
```

---

## Important notes

- Indentation matters in YAML.
- Compose is best for multi-container applications.
- `docker compose` is the modern command form.
- Services in the same Compose project can usually communicate using service names.
- Bind mounts are useful for local development.
- Named volumes are useful for persistent data.
- `docker compose down` removes containers and the default network, but not named volumes unless `-v` is used.
- `depends_on` controls order, but not full readiness unless combined with health checks.

---

## Must memorize

```bash
docker compose version
docker compose up
docker compose up -d
docker compose up --build
docker compose ps
docker compose top
docker compose logs
docker compose logs -f
docker compose stop
docker compose down
docker compose down -v
```

---
## More useful Compose options

For real applications, a Compose file often includes extra options to control startup order, restarts, configuration, health checks, networking, and persistent storage.

### `depends_on`
Defines dependency order between services.

```yaml
depends_on:
  - redis
```

This means the dependent service is started before this one.

### Important note
`depends_on` controls **startup order**, but it does **not always guarantee that the dependency is fully ready** to accept connections.

So:
- started first does not always mean ready first
- for readiness, `healthcheck` is often needed

---

### `restart`
Defines restart behavior for a container.

```yaml
restart: unless-stopped
```

Common values:

- `no` = do not restart automatically
- `on-failure` = restart only if the container exits with an error
- `always` = always restart
- `unless-stopped` = restart automatically unless you manually stop it

This is useful for web apps, databases, and background services.

---

### `environment`
Sets environment variables inside the container.

```yaml
environment:
  FLASK_ENV: development
  DB_HOST: mysql
```

You can also write it like this:

```yaml
environment:
  - FLASK_ENV=development
  - DB_HOST=mysql
```

This is commonly used for:
- database hostnames
- usernames and passwords
- app configuration
- debug or runtime settings

---

### `healthcheck`
Checks whether a container is healthy.

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 3
```

Useful fields:
- `test` = command to run
- `interval` = how often to check
- `timeout` = how long to wait
- `retries` = how many failures before marking unhealthy
- `start_period` = optional grace period before failures count

Health checks are useful when one service depends on another service being actually ready, not just started.

---

### Custom `networks`
Lets you define your own Docker networks and attach selected services to them.

```yaml
services:
  web:
    networks:
      - appnet

  redis:
    networks:
      - appnet

networks:
  appnet:
```

This is useful for:
- isolating groups of services
- controlling communication
- creating cleaner multi-tier application designs

If you do not define a custom network, Compose creates a default network automatically.

---

### Named `volumes`
Lets containers use Docker-managed persistent storage.

```yaml
services:
  db:
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

This means:
- `dbdata` is the Docker-managed volume
- `/var/lib/mysql` is the path inside the container

Named volumes are useful for:
- databases
- logs
- uploaded files
- any data that should survive container recreation

### Note about shared external networks
If multiple Docker Compose projects need to communicate, they can attach services to the same **external Docker network**.

Example:

```yaml
networks:
  shared-net:
    external: true
```

Important notes:
- `external: true` means Compose will use an **already existing Docker network**
- Compose will **not create** that network automatically
- you must create it manually first, for example:

```bash
docker network create shared-net
```

- each Compose file must declare that external network separately
- services must explicitly attach to that network to use it

If you do **not** use `external: true`, Compose can create the network automatically, but it will usually be managed only within that Compose project.

### Note about manual container-to-container communication
There are two common manual ways to let containers use each other’s names:

- **Legacy way:** use `--link`
- **Modern way:** put both containers on the same user-defined network and use their container names as hostnames

---

### Legacy way: `--link`

Before user-defined Docker networks became the standard, Docker allowed containers to communicate using the `--link` option.

Example:

```bash
docker run -d --name redis redis
docker run -d --name web --link redis:redis myapp
```

### Explanation
- `--name redis` gives the first container the name `redis`
- `--link redis:redis` links the `redis` container to the `web` container
- inside the `web` container, the hostname `redis` can be used to reach the Redis container

In this example, the second `redis` is the alias name that will be used inside the `web` container.

You could also do:

```bash
docker run -d --name redis redis
docker run -d --name web --link redis:myredis myapp
```

Then inside the `web` container, you would use:

```bash
myredis
```

as the hostname instead of `redis`.

### Important note
`--link` is a **legacy feature** and is not the recommended modern approach.

---

### Modern way: shared user-defined network

The recommended way is to place both containers on the same custom network.

Example:

```bash
docker network create mynet

docker run -d --name redis --network mynet redis
docker run -d --name web --network mynet myapp
```

In this case, the `web` container can reach the `redis` container using:

```bash
redis
```

as the hostname.

### Why this is better
- cleaner and more flexible
- works better for multiple containers
- avoids legacy linking behavior
- fits modern Docker networking practices

> `--link` is the old way. A shared custom network is the recommended modern way.
## Key ideas

- Docker Compose is used to manage multi-container applications.
- It uses a YAML file to define services, networks, volumes, and settings.
- It reduces mistakes compared to running many manual `docker run` commands.
- It works well for local development, demos, testing, and small deployments.
- It is an infrastructure-as-code style approach for containers.