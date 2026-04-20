---
tags:
  - Docker
  - Linux
---

A multi-stage Dockerfile is used to **build an application artifact in one stage** and **copy only the required output into the final runtime image**.

This helps make Docker images:
- smaller
- cleaner
- more secure
- more production-ready

---

## Why multi-stage is needed

In many applications, you need build tools such as:
- Maven
- Gradle
- npm
- Go compiler
- GCC

These tools are needed only while building the application, not while running it.

If you install all build tools inside the same final image, the image becomes very large because it also contains:
- source code
- package manager caches
- downloaded dependencies
- temporary build files
- target/build directories
- tools not needed at runtime

That makes the image heavier and less efficient to ship.

---

## The problem with a single-stage build

Suppose you have a Java application and want to build a `.war` file using Maven.

A simple but bad approach would be:
- start from a runtime image
- install JDK and Maven
- clone the source code
- run `mvn install`
- keep everything in the same image

That works, but the final image contains both:
- build tools
- runtime application

This increases image size unnecessarily.

---

## Manual build is also not ideal

Another option is:
- build the artifact manually on the host machine
- then copy only the artifact into the Docker image

This makes the final image smaller, but it has a problem:
- part of the build process is manual

That means:
- less automation
- more human error
- inconsistent builds
- weaker CI/CD flow

---

## The solution: multi-stage Dockerfile

A multi-stage Dockerfile solves both problems.

It lets you:
1. build the artifact in one stage
2. use a separate lightweight runtime image in the final stage
3. copy only the built artifact into the final image

So you get:
- full automation
- smaller runtime image
- cleaner final image

---

## Main idea

A multi-stage Dockerfile has **more than one `FROM` instruction**.

Each `FROM` starts a new stage.

You can give a stage a name using `AS`:

```dockerfile
FROM maven:3.9-eclipse-temurin-8 AS build
```

Later, you can copy files from that stage into another stage:

```dockerfile
COPY --from=build /app/target/app.war /usr/local/tomcat/webapps/ROOT.war
```

---

## Example scenario

Suppose:
- Maven builds the Java artifact
- Tomcat runs the application

Then:
- build stage = Maven + JDK
- runtime stage = Tomcat
- final image contains only Tomcat + artifact

---

## Example multi-stage Dockerfile

```dockerfile
# Stage 1: Build the artifact
FROM maven:3.9-eclipse-temurin-8 AS build

WORKDIR /app

COPY . .

RUN mvn clean install -DskipTests

# Stage 2: Runtime image
FROM tomcat:8-jre11

RUN rm -rf /usr/local/tomcat/webapps/*

COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

---

## How it works

### Stage 1
The first stage:
- uses a Maven image
- copies the source code
- builds the artifact

This stage contains:
- source code
- Maven
- JDK
- dependencies
- target directory

But this stage is only for building.

### Stage 2
The second stage:
- uses a Tomcat runtime image
- removes default Tomcat applications
- copies only the generated `.war` file from the build stage
- starts Tomcat

This final image does **not** include Maven, source code, or build dependencies.

---

## Why this is better

With multi-stage builds:
- build tools stay in the build stage
- only the final artifact goes into the runtime stage
- the final image is much smaller
- the image is easier to distribute
- the attack surface is reduced
- the build is fully automated

This is the preferred method for production-ready images when an artifact must be built first.

---

## What `COPY --from` does

This instruction copies files from one stage into another.

Example:

```dockerfile
COPY --from=build /app/target/app.war /usr/local/tomcat/webapps/ROOT.war
```

### Meaning
- `--from=build` = copy from the stage named `build`
- `/app/target/app.war` = source path inside the build stage
- `/usr/local/tomcat/webapps/ROOT.war` = destination path inside the final image

This is the key instruction in multi-stage Dockerfiles.

---

## Why image size becomes smaller

In a multi-stage build, the final image contains only what exists in the **last stage**.

It does **not** automatically include everything from previous stages.

So even if the build stage downloads:
- Maven dependencies
- source code
- build caches
- temporary files

those do not end up in the final runtime image unless you explicitly copy them.

That is why multi-stage builds reduce image size significantly.

---

## Typical use cases

Multi-stage builds are very useful for:

- Java apps built with Maven or Gradle
- Node.js apps where frontend assets are built first
- Go applications compiled into binaries
- C/C++ applications compiled in one stage and run in another
- Python apps where wheels or build artifacts are prepared separately

Any time your app needs a **build step**, a multi-stage Dockerfile is usually a good idea.

---

## Single-stage vs multi-stage

| Feature | Single-stage build | Multi-stage build |
|---|---|---|
| Build tools in final image | Yes | No |
| Final image size | Larger | Smaller |
| Automation | Possible | Yes |
| Manual artifact build needed | Sometimes | No |
| Production suitability | Lower | Better |

---

## Build the image

```bash
docker build -t appimage:v1 .
```

This runs all stages of the Dockerfile and produces the final image from the last stage.

---

## Build a specific stage

You can build only a specific stage of a multi-stage Dockerfile using `--target`.

Example:

```bash
docker build --target build -t app-build-stage .
```

This is useful when:
- you want to test only the build stage
- you want to debug intermediate stages
- you want to inspect builder output

---

## View images

```bash
docker images
```

You may notice:
- the final image has your chosen name and tag
- intermediate build stages may appear as unnamed images during or after the build

The main image you care about is the final output image.

---

## Layer caching in multi-stage builds

Docker still uses layer caching in multi-stage Dockerfiles.

That means:
- unchanged layers can be reused
- builds become faster
- ordering instructions properly improves cache usage

### Good practice
Put instructions that change less often earlier.

Example:
- install dependencies first
- copy frequently changing source code later

This helps Docker reuse cached layers more often.

---

## `.dockerignore` in multi-stage builds

A `.dockerignore` file is still important in multi-stage builds.

It helps by:
- reducing build context size
- avoiding unnecessary files
- improving build performance
- preventing accidental copying of secrets or temporary files

Example:

```dockerignore
.git
target
node_modules
*.log
.env
```

Even with multi-stage builds, sending unnecessary files in the build context is still inefficient.

---

## Advanced runtime optimization

### Distroless images

A distroless image is a minimal runtime image that contains only:
- the application
- required runtime dependencies

It usually does **not** contain:
- a shell
- a package manager
- common debugging tools

This makes the final image:
- smaller
- more secure
- harder to tamper with
- better for production use

Distroless images are commonly used as the **final stage** in a multi-stage Dockerfile.

### Example

```dockerfile
# Build stage
FROM golang:1.18 AS build
WORKDIR /app
COPY . .
RUN go build -o myapp

# Distroless runtime stage
FROM gcr.io/distroless/static-debian12
COPY --from=build /app/myapp /myapp
CMD ["/myapp"]
```

### Notes
- Distroless images are best for production.
- They do not usually include `sh`, `bash`, `curl`, or package managers.
- Use JSON-array form for `CMD` or `ENTRYPOINT`.
- Debugging is harder, so testing should happen earlier in the pipeline.

---

### Docker Hardened Images

Docker Hardened Images are minimal and security-focused base images designed for production use.

They are intended to:
- reduce vulnerabilities
- reduce attack surface
- improve compliance and supply-chain security
- provide production-ready hardened runtime images

You can use them the same way you use other runtime base images.

### Example idea

```dockerfile
FROM <your-dhi-image>
COPY --from=build /app/myapp /myapp
CMD ["/myapp"]
```

### Notes
- Docker Hardened Images fit well as a secure final runtime stage.
- They follow the same minimal-image philosophy as distroless-style runtime images.
- They are mainly useful in production and security-focused environments.

---

## Important notes

- A multi-stage Dockerfile has more than one `FROM`.
- The last stage becomes the final image unless a target stage is specified.
- `COPY --from=<stage>` is used to copy files from an earlier stage.
- The build stage can be large, but the final image can still stay small.
- This is better than manually building artifacts outside Docker.
- This is also better than keeping build tools in the runtime image.

---

## Best practices

- Use a dedicated builder image such as Maven, Gradle, Node, or Go.
- Use a lightweight runtime image for the final stage.
- Copy only the required artifact into the final image.
- Remove default files from runtime images if necessary.
- Keep the runtime image minimal.
- Use multi-stage builds whenever your application requires compilation or packaging.
- Use distroless or hardened runtime images when security and minimal footprint matter.

---

## Good mental model

Think of multi-stage builds like this:

- **Stage 1** = workshop where the product is built
- **Stage 2** = delivery box containing only the final product

You do not ship the workshop to the customer.  
You only ship the final product.

---

## Must memorize

```bash
docker build -t appimage:v1 .
docker build --target build -t app-build-stage .

FROM <builder-image> AS build
COPY . .
RUN <build-command>

FROM <runtime-image>
COPY --from=build <source> <destination>
```

---

## Key ideas

- Multi-stage Dockerfiles separate build and runtime.
- They reduce final image size.
- They automate artifact creation.
- They improve production readiness.
