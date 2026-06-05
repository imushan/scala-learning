---
name: scala-hex-docker
description: Use when user asks to build docker image, dockerize, deploy, push to registry, or mentions docker build/ACR/containerize. Do NOT use for local compile/test only.
---

# Docker build and deploy for Scala hexagonal project

Build Docker image and deploy for Scala hexagonal architecture project.

## Usage
- User says: "build docker" or "dockerize" or "deploy" or "push to registry"

## Steps

### 1. Build Distribution
```bash
bleep my-dist
```

Output: `out/` directory with `bin/` and `lib/`

### 2. Build Docker Image
```bash
docker build -t <image-name>:<tag> .
```

### 3. Run Container Locally
```bash
docker run --rm -p 8080:8080 <image-name>:<tag>
```

### 4. Push to Registry
```bash
# Login to ACR
docker login --username=<username> <registry>

# Tag and push
docker tag <image-name>:<tag> <registry>/<namespace>/<image-name>:<tag>
docker push <registry>/<namespace>/<image-name>:<tag>
```

## Dockerfile Structure
The project uses a Dockerfile with JAR separation for layer caching:

```dockerfile
FROM eclipse-temurin:17-jre-jammy

WORKDIR /app

# Copy separated dependencies (optimized for caching)
COPY out/lib_deps/ /app/lib/
COPY out/lib_app/ /app/lib/

# Copy startup script
COPY out/bin/ /app/bin/

RUN chmod +x /app/bin/*
ENV PATH="/app/bin:${PATH}"

ENTRYPOINT ["/app/bin/app"]
```

## Jenkins CI/CD Pipeline

The Jenkinsfile includes these stages:
1. **Checkout** - Pull code
2. **Build with Bleep** - Run `bleep my-dist` and separate JARs
3. **Dockerize** - Build Docker image
4. **Push to ACR** - Push to Alibaba Cloud registry
5. **Cleanup** - Remove old images

### JAR Separation (for Docker layer caching)
```bash
mkdir -p out/lib_deps out/lib_app

# App JAR goes to lib_app
mv out/lib/app.jar out/lib_app/ 2>/dev/null || true

# Third-party deps go to lib_deps
mv out/lib/*.jar out/lib_deps/ 2>/dev/null || true
```

## Required Jenkins Credentials
| Credential ID | Type | Description |
|--------------|------|-------------|
| acr-credentials | Username/Password | ACR username/password |
| gitee-token | Secret Text | Git access token |

## Test the API
```bash
# Get all users
curl http://localhost:8080/api/users

# Create user
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# Get user by ID
curl http://localhost:8080/api/users/1

# Delete user
curl -X DELETE http://localhost:8080/api/users/1
```

## Common Issues
| Issue | Solution |
|-------|----------|
| `my-dist` not found | Check scripts module exists in bleep.yaml |
| Docker build cache miss | Verify lib_deps and lib_app separation |
| ACR push denied | Check credentials and namespace permissions |