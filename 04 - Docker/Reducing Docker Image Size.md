---
tags:
  - Docker
---

To reduce Docker image size, use several techniques:

- Use a smaller base image when possible.
- Use multi-stage builds so build tools are not included in the final image.
- Use a `.dockerignore` file to avoid copying unnecessary files like `.git`, logs, tests, and local dependencies.
- Install only the packages required at runtime.
- Avoid recommended or extra packages when installing dependencies.
- Clean package manager cache and temporary files in the same layer.
- Combine related `RUN` commands to avoid leaving unnecessary data in old layers.
- Avoid copying unnecessary application files such as documentation, test data, and local caches.
- Inspect image layers with tools like `dive` to find what is making the image large.

## Examples

### APT
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

### Python
```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

### .dockerignore
```dockerignore
.git
node_modules
venv
__pycache__
*.log
tests
docs
.env
```

