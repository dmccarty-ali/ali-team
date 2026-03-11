---
name: ali-docker
description: |
  Docker containerization best practices, patterns, and security. Use when:

  PLANNING: Designing container architectures, planning multi-stage builds, architecting
  docker-compose configurations, choosing base images, designing health check strategies,
  planning volume and network isolation

  IMPLEMENTATION: Writing Dockerfiles, implementing docker-compose configurations, building
  multi-stage images, configuring health checks, setting up volumes and networks, debugging
  container issues, optimizing image sizes

  GUIDANCE: Asking about Docker best practices, asking how to containerize applications
  securely, asking about image optimization, asking about docker-compose patterns, asking
  about volume management, asking about networking between containers

  REVIEW: Reviewing Dockerfiles for security, checking for image bloat, validating
  docker-compose configurations, assessing production-readiness, checking for security
  vulnerabilities, verifying health checks
---

# Docker Containerization

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing containerization strategies for applications
- Planning multi-stage build architectures
- Architecting docker-compose configurations for multi-service applications
- Choosing appropriate base images
- Designing health check and monitoring strategies
- Planning volume management and data persistence
- Designing network isolation between services

**Implementation:**
- Writing Dockerfiles for applications
- Implementing docker-compose configurations
- Building multi-stage Docker images
- Configuring container health checks
- Setting up volumes and bind mounts
- Configuring networks and service communication
- Debugging container startup or runtime issues
- Optimizing image sizes and build times

**Guidance/Best Practices:**
- Asking about Docker security best practices
- Asking how to containerize applications efficiently
- Asking about image size optimization techniques
- Asking about docker-compose patterns
- Asking about volume and data management
- Asking about container networking
- Asking when to use Docker vs other solutions
- Asking about production deployment patterns

**Review/Validation:**
- Reviewing Dockerfiles for security issues
- Checking for image bloat and inefficiency
- Validating docker-compose configurations
- Assessing container production-readiness
- Checking for common Docker vulnerabilities
- Verifying health check implementations
- Validating network isolation

---

## Key Principles

- **Pin base image versions** - Never use :latest in production, pin to specific versions
- **Multi-stage builds** - Separate build and runtime stages for smaller final images
- **Layer optimization** - Order instructions from least-changed to most-changed for caching
- **Non-root by default** - Always specify USER directive for security
- **Health checks required** - Every production container needs a HEALTHCHECK
- **No secrets in images** - Never bake credentials into layers, use runtime injection
- **.dockerignore always** - Exclude unnecessary files from build context
- **Prefer COPY over ADD** - Only use ADD for URL fetching or tar extraction
- **Clean up in same layer** - Remove temporary files in the same RUN command
- **Explicit is better** - Document exposed ports, volumes, and environment variables

---

## Patterns We Use

### Multi-Stage Build Pattern

Separate build dependencies from runtime dependencies for smaller images:

```dockerfile
# Build stage
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copy only runtime dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Add local pip to PATH
ENV PATH=/home/appuser/.local/bin:$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["python", "app.py"]
```

**Benefits:**
- Build stage has gcc and build tools (not in final image)
- Final image only has runtime dependencies
- Smaller image size (100MB vs 500MB+)
- Faster deployments and pulls

---

### Layer Caching Optimization

Order Dockerfile instructions from least-changed to most-changed:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# 1. Copy package files first (change infrequently)
COPY package*.json ./

# 2. Install dependencies (cached until package.json changes)
RUN npm ci --only=production

# 3. Copy application code last (changes most frequently)
COPY . .

# Build step (only re-runs if code or dependencies changed)
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Why order matters:**
- Docker caches each layer
- If a layer changes, all subsequent layers rebuild
- Package dependencies change less than application code
- Copying package.json first allows npm install to be cached

---

### Non-Root User Pattern

Always run containers as non-root for security:

```dockerfile
FROM ubuntu:22.04

# Update and install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user with specific UID
RUN useradd -m -u 1000 appuser

# Create app directory with proper ownership
RUN mkdir -p /app && chown appuser:appuser /app

WORKDIR /app

# Copy files with proper ownership
COPY --chown=appuser:appuser . .

# Switch to non-root user BEFORE running anything
USER appuser

CMD ["./app"]
```

**Security benefits:**
- If container is compromised, attacker has limited privileges
- Cannot modify system files or install packages
- Cannot bind to privileged ports (<1024)
- Follows principle of least privilege

---

### Health Check Pattern

Define health checks for container orchestration:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# HTTP endpoint health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Alternative: Python script health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python /app/healthcheck.py || exit 1

# Alternative: TCP port check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD nc -z localhost 8000 || exit 1

CMD ["python", "app.py"]
```

**Health check parameters:**
- `--interval=30s` - Check every 30 seconds
- `--timeout=5s` - Health check must complete in 5 seconds
- `--start-period=10s` - Grace period before first check
- `--retries=3` - Mark unhealthy after 3 consecutive failures

---

### .dockerignore Pattern

Exclude unnecessary files from build context:

```
# .dockerignore

# Version control
.git
.gitignore
.gitattributes

# IDE
.vscode
.idea
*.swp
*.swo

# Dependencies
node_modules
venv
__pycache__
*.pyc
.pytest_cache

# Build artifacts
dist
build
*.egg-info
target

# Environment and secrets
.env
.env.*
*.pem
*.key
secrets

# Documentation
README.md
docs
*.md

# CI/CD
.github
.gitlab-ci.yml
Jenkinsfile

# Docker files
Dockerfile*
docker-compose*.yml
.dockerignore

# Logs and temp files
*.log
logs
tmp
temp
```

**Benefits:**
- Smaller build context (faster uploads to daemon)
- Prevents secrets from being in build context
- Faster builds (less data to process)
- Prevents cache invalidation from irrelevant file changes

---

### Docker Compose Service Pattern

Complete service definition with all production concerns:

```yaml
version: '3.8'

services:
  api:
    image: myorg/api:1.2.3  # Specific version, not :latest
    container_name: api
    build:
      context: ./api
      dockerfile: Dockerfile
      args:
        BUILD_DATE: "2026-01-27"
    environment:
      - NODE_ENV=production
      - LOG_LEVEL=${LOG_LEVEL:-info}
    env_file:
      - .env  # External environment variables
    ports:
      - "3000:3000"  # Expose API port
    volumes:
      - api-data:/app/data  # Named volume for persistence
      - ./logs:/app/logs:rw  # Bind mount for logs
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy  # Wait for health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped  # Restart policy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    image: postgres:15.2-alpine  # Specific version
    container_name: postgres
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 2G

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

volumes:
  api-data:
    driver: local
  postgres-data:
    driver: local
```

**Note:** The `version` field is optional in Docker Compose v2 (the default since Docker Desktop 4.x). It's shown here for backward compatibility but can be omitted in modern setups.

---

### Cleanup in Same Layer Pattern

Remove temporary files in the same RUN command to avoid bloat:

```dockerfile
# WRONG - temp files remain in intermediate layer
RUN apt-get update
RUN apt-get install -y gcc
RUN gcc myapp.c -o myapp
RUN apt-get remove -y gcc
RUN apt-get clean

# CORRECT - cleanup in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    gcc myapp.c -o myapp && \
    apt-get remove -y gcc && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Why:**
- Each RUN creates a new layer
- Files in earlier layers remain in image (even if deleted later)
- Chaining commands with && keeps everything in one layer
- Final image size is minimized

---

### Base Image Selection Pattern

Choose appropriate base images:

```dockerfile
# Standard images (full OS)
FROM ubuntu:22.04          # Full Ubuntu (77MB)
FROM python:3.11           # Python with full OS (1GB)
FROM node:18               # Node with full OS (1GB)

# Slim images (minimal OS)
FROM python:3.11-slim      # Python with minimal Debian (180MB)
FROM node:18-slim          # Node with minimal Debian (240MB)

# Alpine images (smallest)
FROM python:3.11-alpine    # Python with Alpine Linux (50MB)
FROM node:18-alpine        # Node with Alpine Linux (170MB)

# Distroless images (Google, no shell)
FROM gcr.io/distroless/python3-debian11  # No shell, minimal attack surface (50MB)
```

**Selection criteria:**
- **Full images** - When you need full OS tools (debugging, testing)
- **Slim images** - Balance of size and compatibility (production default)
- **Alpine images** - Smallest size, but uses musl libc (may have compatibility issues)
- **Distroless** - Best security (no shell), but harder to debug

---

## Docker Compose Patterns

### Development Override Pattern

Separate development and production configurations:

```yaml
# docker-compose.yml (base, production)
version: '3.8'
services:
  api:
    image: myorg/api:1.2.3
    environment:
      - NODE_ENV=production
    volumes:
      - api-data:/app/data
    command: ["node", "dist/index.js"]
```

```yaml
# docker-compose.override.yml (development, auto-loaded)
version: '3.8'
services:
  api:
    build:
      context: ./api
      target: development  # Multi-stage build development target
    environment:
      - NODE_ENV=development
      - DEBUG=*
    volumes:
      - ./api:/app  # Bind mount for live reload
      - /app/node_modules  # Anonymous volume for dependencies
    ports:
      - "9229:9229"  # Expose debugger port
    command: ["npm", "run", "dev"]
```

**Usage:**
```bash
# Development (auto-loads override)
docker-compose up

# Production (ignore override)
docker-compose -f docker-compose.yml up

# Specify custom override
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

### Dependency Ordering with Health Checks

Ensure services start in correct order:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  migrations:
    image: myorg/migrations:latest
    depends_on:
      postgres:
        condition: service_healthy  # Wait for postgres health check
    command: ["./migrate.sh"]

  api:
    image: myorg/api:latest
    depends_on:
      migrations:
        condition: service_completed_successfully  # Wait for migrations
    command: ["node", "server.js"]
```

**Dependency conditions:**
- `service_started` - Default, just wait for container start
- `service_healthy` - Wait for health check to pass
- `service_completed_successfully` - Wait for container to exit with code 0

---

### Named Volumes vs Bind Mounts

Choose appropriate volume types:

```yaml
version: '3.8'

services:
  app:
    image: myorg/app:latest
    volumes:
      # Named volume (managed by Docker, persists data)
      - app-data:/app/data

      # Bind mount (host directory, development live reload)
      - ./src:/app/src:ro  # Read-only bind mount

      # tmpfs mount (memory-backed, temporary)
      - type: tmpfs
        target: /app/tmp

      # Anonymous volume (prevents bind mount override)
      - /app/node_modules

volumes:
  app-data:  # Declare named volume
```

**When to use:**
- **Named volumes** - Production data persistence (databases, uploads)
- **Bind mounts** - Development (live code reload, local config)
- **tmpfs** - Temporary files, cache (cleared on restart)
- **Anonymous volumes** - Prevent parent directory bind mount from overriding

---

## Container Debugging

### Common Debugging Commands

```bash
# View container logs
docker logs <container-id>
docker logs -f <container-id>              # Follow logs
docker logs --tail 100 <container-id>      # Last 100 lines
docker logs --since 10m <container-id>     # Last 10 minutes

# Execute commands in running container
docker exec -it <container-id> sh          # Interactive shell
docker exec <container-id> ls -la /app     # Run single command
docker exec <container-id> env             # View environment variables

# Inspect container configuration
docker inspect <container-id>              # Full JSON config
docker inspect -f '{{.State.Health}}' <container-id>  # Health check status
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container-id>  # IP address

# View resource usage
docker stats <container-id>                # Real-time CPU/memory usage
docker stats --no-stream                   # One-time snapshot

# Check container processes
docker top <container-id>                  # Running processes

# Copy files from container
docker cp <container-id>:/app/logs ./logs

# View container filesystem changes
docker diff <container-id>                 # Changed files since start
```

---

### Network Debugging

```bash
# List networks
docker network ls

# Inspect network
docker network inspect <network-name>

# Test connectivity between containers
docker exec <container-id> ping <other-container-name>
docker exec <container-id> curl http://<other-container-name>:port

# DNS resolution test
docker exec <container-id> nslookup <service-name>

# Check port mapping
docker port <container-id>

# Network troubleshooting container
docker run -it --network <network-name> nicolaka/netshoot
```

**Common issues:**
- Containers can't communicate - Check if on same network
- DNS not resolving - Use service name, not container name
- Port not accessible - Check if exposed in Dockerfile and mapped in docker-compose
- Connection refused - Service might not be listening on 0.0.0.0

---

### Build Debugging

```bash
# Build with no cache (force rebuild)
docker build --no-cache -t myapp:latest .

# Build and show all output
docker build --progress=plain -t myapp:latest .

# Build specific target in multi-stage
docker build --target builder -t myapp:builder .

# Check build context size
docker build --no-cache -t myapp:latest . 2>&1 | grep "Sending build context"

# Inspect layer sizes
docker history myapp:latest

# Dive into image layers
docker run --rm -it wagoodman/dive:latest myapp:latest
```

---

### Dockerfile Linting with hadolint

Run hadolint on all Dockerfiles (equivalent of shellcheck for bash):

```bash
# Install hadolint
# macOS: brew install hadolint
# Linux: wget -O /usr/local/bin/hadolint https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64 && chmod +x /usr/local/bin/hadolint
# Docker: docker run --rm -i hadolint/hadolint < Dockerfile

# Run on Dockerfile
hadolint Dockerfile

# Ignore specific rules inline
# hadolint ignore=DL3008
RUN apt-get install -y curl

# Check all Dockerfiles in project
find . -name "Dockerfile*" -exec hadolint {} +

# Output as JSON for CI/CD
hadolint --format json Dockerfile
```

**Common hadolint rules:**
- DL3008 - Pin versions in apt-get install
- DL3009 - Delete apt-get lists after install
- DL3025 - Use JSON form for CMD
- DL3027 - Do not use apt, use apt-get
- DL4006 - Set SHELL option -o pipefail before RUN with pipe

---

### Container Exit Code Debugging

| Exit Code | Meaning | Common Cause |
|-----------|---------|--------------|
| 0 | Success | Container completed normally |
| 1 | Application error | Exception, crash, bug |
| 2 | Misuse | Invalid command arguments |
| 125 | Docker daemon error | Docker engine issue |
| 126 | Command cannot execute | Permission denied, not executable |
| 127 | Command not found | Invalid CMD or ENTRYPOINT |
| 130 | Terminated by Ctrl-C | User interrupted (SIGINT) |
| 137 | Killed (OOM) | Out of memory (SIGKILL) |
| 139 | Segmentation fault | Memory access error |
| 143 | Terminated | Graceful shutdown (SIGTERM) |

```bash
# Check exit code
docker inspect -f '{{.State.ExitCode}}' <container-id>

# Check if OOM killed
docker inspect -f '{{.State.OOMKilled}}' <container-id>
```

---

## Security Best Practices

### No Secrets in Images

```dockerfile
# WRONG - secret baked into layer
ENV API_KEY=abc123xyz

# WRONG - secret in ARG (visible in docker history)
ARG API_KEY=abc123xyz

# WRONG - copying secret file
COPY secrets.json /app/

# CORRECT - pass at runtime via environment
docker run -e API_KEY=${API_KEY} myapp:latest

# CORRECT - use Docker secrets (Swarm/Compose 3.1+)
docker service create --secret api_key myapp:latest

# CORRECT - read from file mounted at runtime
docker run -v /host/secrets:/secrets:ro myapp:latest
```

**Verification:**
```bash
# Check for secrets in history
docker history myapp:latest

# Check for secrets in image
docker run --rm myapp:latest env | grep -i "key\|password\|token"
```

**Escalation:** For comprehensive security reviews beyond Docker-specific patterns, coordinate with the security-expert agent who covers OWASP, secrets management, and compliance requirements.

---

### Read-Only Filesystem

Run containers with read-only root filesystem:

```yaml
version: '3.8'
services:
  api:
    image: myorg/api:latest
    read_only: true  # Root filesystem read-only
    tmpfs:
      - /tmp  # Writable temporary directory
      - /app/cache
    volumes:
      - app-data:/app/data  # Writable data volume
```

**Benefits:**
- Prevents tampering with container filesystem
- Reduces attack surface
- Forces proper data volume management

---

### Security Scanning

```bash
# Scan image for vulnerabilities (Docker Scout)
docker scout cves myapp:latest
docker scout recommendations myapp:latest

# Scan with Trivy (open source)
trivy image myapp:latest

# Scan base image before using
docker scout cves python:3.11-slim
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Using :latest tag | Non-reproducible builds | Pin to specific version: python:3.11.2-slim |
| No .dockerignore | Large build context, slow builds | Create comprehensive .dockerignore |
| Running as root | Security vulnerability | Add USER directive with non-root user |
| Secrets in ENV/ARG | Exposed in docker history | Pass at runtime or use secrets |
| No HEALTHCHECK | No container health visibility | Define HEALTHCHECK instruction |
| COPY before dependencies | Cache invalidation on code change | Copy package files first, install, then copy code |
| Multiple RUN for cleanup | Layers remain with temp files | Chain with && in single RUN |
| Installing build tools in final image | Bloated images | Use multi-stage builds |
| ADD instead of COPY | Implicit tar extraction, URL fetching | Use COPY unless you need ADD features |
| Exposing unnecessary ports | Attack surface | Only expose required ports |
| No resource limits | Resource exhaustion | Set memory and CPU limits |
| Privileged mode | Full host access | Avoid or justify with specific capabilities |
| Hardcoded configuration | Not portable | Use environment variables and .env files |
| No version pinning | Dependency drift | Pin all package versions |
| Using shell form CMD | Signals not passed | Use exec form: CMD ["node", "app.js"] |

---

## Quick Reference

### Dockerfile Template

```dockerfile
# Choose appropriate base image
FROM python:3.11-slim AS builder

# Set working directory
WORKDIR /build

# Install build dependencies in one layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy and install dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    mkdir -p /app && \
    chown appuser:appuser /app

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Set PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Use exec form for CMD
CMD ["python", "app.py"]
```

---

### Common Docker Commands

```bash
# Build image
docker build -t myapp:1.0.0 .
docker build --no-cache -t myapp:1.0.0 .

# Run container
docker run -d -p 8000:8000 --name myapp myapp:1.0.0
docker run -it --rm myapp:1.0.0 sh

# Compose commands
docker-compose up -d
docker-compose down
docker-compose logs -f
docker-compose exec service-name sh
docker-compose restart service-name

# Image management
docker images
docker rmi image-name:tag
docker image prune  # Remove dangling images
docker system prune  # Remove unused data

# Container management
docker ps
docker ps -a
docker stop container-id
docker rm container-id
docker container prune  # Remove stopped containers

# Volume management
docker volume ls
docker volume inspect volume-name
docker volume rm volume-name
docker volume prune  # Remove unused volumes

# Network management
docker network ls
docker network inspect network-name
docker network create network-name
docker network prune  # Remove unused networks
```

---

### Image Size Optimization Checklist

- [ ] Using multi-stage build
- [ ] Minimal base image (slim or alpine)
- [ ] .dockerignore excludes unnecessary files
- [ ] Dependencies copied before application code
- [ ] Cleanup in same RUN command (rm -rf /var/lib/apt/lists/*)
- [ ] --no-cache-dir for pip/npm installs
- [ ] --no-install-recommends for apt installs
- [ ] Remove build tools after compilation
- [ ] Use .dockerignore to exclude build artifacts
- [ ] Layer order optimized for caching

---

## Your Codebase

Docker files are project-specific. Check individual project directories for:

- Dockerfiles (typically in project root or service directories)
- docker-compose.yml files (typically in project root)
- .dockerignore files (same location as Dockerfiles)

Example project locations:
- ~/ali-ai/projects/tax-practice/docker-compose.yml
- ~/ali-ai/projects/braid-manager/docker-compose.yml
- ~/ali-ai/projects/tax-ai-chat/docker-compose.yml

---

## References

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
- [Docker Security](https://docs.docker.com/engine/security/)
- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Networking](https://docs.docker.com/network/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)
- [Health Check Documentation](https://docs.docker.com/engine/reference/builder/#healthcheck)
- [hadolint (Dockerfile linter)](https://github.com/hadolint/hadolint)
- ali-mcp skill - For Docker-related MCP server integrations and container management tooling

---

**Document Version:** 1.1
**Last Updated:** 2026-02-07
**Maintained By:** ALI AI Team
