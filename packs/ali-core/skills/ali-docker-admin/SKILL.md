---
name: ali-docker-admin
description: |
  Docker operations with safety and security enforcement. Use when:

  PLANNING: Planning container architecture, designing docker-compose configs,
  evaluating base images, considering multi-stage builds, registry strategy

  IMPLEMENTATION: Executing docker/docker-compose commands, building images,
  managing containers, configuring networks, pushing to registries

  GUIDANCE: Asking about Docker best practices, image optimization,
  security hardening, container orchestration, resource limits

  REVIEW: Reviewing running containers, auditing images, checking
  security configurations, validating resource usage
---

# Docker Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing container architecture
- Planning docker-compose configurations
- Evaluating base images (Alpine, Ubuntu, Debian)
- Considering multi-stage builds
- Planning registry and tagging strategy
- Designing network topology

**Implementation:**
- Executing docker and docker-compose commands
- Building container images
- Managing container lifecycle (run, stop, rm)
- Configuring networks and volumes
- Pushing images to registries
- Running containers in production

**Guidance/Best Practices:**
- Container security hardening
- Image size optimization
- Dockerfile best practices
- Resource limits and health checks
- Secrets management
- Multi-stage build patterns

**Review/Validation:**
- Reviewing running containers
- Auditing container security
- Checking resource usage
- Validating image tags and versions
- Reviewing docker-compose configurations

---

## Core Principles

### 1. Security First (Non-Negotiable)

| Rule | Rationale |
|------|-----------|
| **Never run --privileged** | Grants container root access to host, major security violation |
| **Never mount Docker socket** | Container can control Docker daemon, escalation path |
| **Always run as non-root** | Principle of least privilege |
| **Never expose 0.0.0.0 on databases** | Opens database to all network interfaces |
| **Never use --net=host in production** | Bypasses container network isolation |
| **Always use specific tags** | :latest is unpredictable and breaks reproducibility |
| **Always scan images** | Check for vulnerabilities before deployment |

### 2. Image Hygiene

| Practice | Impact |
|----------|--------|
| **Minimal base images** | Smaller attack surface, faster pulls |
| **Multi-stage builds** | Separate build tools from runtime, smaller final image |
| **Specific tags** | v1.2.3, not :latest - reproducible deployments |
| **Layer caching** | Order Dockerfile commands for efficient caching |
| **No secrets in images** | Use environment variables or Docker secrets |
| **Regular updates** | Rebuild images monthly for security patches |

### 3. Resource Management

| Practice | Why |
|----------|-----|
| **Memory limits** | Prevent container from consuming all host memory |
| **CPU limits** | Fair resource sharing across containers |
| **Health checks** | Automatic restart of unhealthy containers |
| **Restart policies** | Automatic recovery from crashes |
| **Volume management** | Data persistence and separation |
| **Log rotation** | Prevent disk space exhaustion |

---

## Dangerous Operations (REQUIRE CONFIRMATION)

### Critical Risk (Data Loss)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `docker rm -f` | Forcibly stops and removes running container | "Yes, force remove container [name]" |
| `docker volume rm` | Deletes volume and all data | "Yes, remove volume [name]" |
| `docker volume prune` | Removes all unused volumes | "Yes, prune volumes" |
| `docker system prune` | Removes stopped containers, networks, images, cache | "Yes, prune system" |
| `docker-compose down -v` | Removes volumes (data loss) | "Yes, remove volumes for [project]" |
| `docker-compose down --rmi all` | Removes all images | "Yes, remove images for [project]" |

### High Risk (Security)

| Operation | Risk | Action |
|-----------|------|--------|
| `docker run --privileged` | Host system access | Block unless explicit review and justification |
| `-v /var/run/docker.sock` | Docker daemon control | Block unless explicit review and justification |
| `--user root` in production | Security violation | Block, require non-root user |
| `-p 0.0.0.0:5432` | Exposes database to all interfaces | Block, bind to localhost or specific IP |
| `--net=host` on production | Bypasses network isolation | Block, use bridge networks |

### Medium Risk (Service Impact)

| Operation | Risk | Action |
|-----------|------|--------|
| `docker stop` | Stops running service | Confirm if production container |
| `docker push` | Pushes image to registry | Confirm before pushing to production registry |
| `docker exec` on production | Direct access to running container | Confirm and log for audit |

---

## Safe Operations (No Confirmation Needed)

```bash
# Container inspection
docker ps
docker ps -a
docker logs [container]
docker inspect [container]
docker stats
docker top [container]

# Image inspection
docker images
docker image ls
docker image history [image]

# Network inspection
docker network ls
docker network inspect [network]

# Volume inspection
docker volume ls
docker volume inspect [volume]

# Docker Compose inspection
docker-compose ps
docker-compose logs
docker-compose config

# System info
docker info
docker version

# Build operations (local)
docker build -t [tag] .
docker pull [image]

# Service lifecycle (non-destructive)
docker start [container]
docker-compose up -d
```

---

## Container Lifecycle

### Building Images

**Best practices:**

```dockerfile
# Multi-stage build example
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
# Copy only installed packages, not build tools
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY . .

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

CMD ["python", "app.py"]
```

**Building:**

```bash
# Always use specific tags
docker build -t myapp:v1.2.3 .

# For development
docker build -t myapp:dev .

# With build args
docker build --build-arg VERSION=1.2.3 -t myapp:v1.2.3 .
```

### Running Containers

**Good pattern:**

```bash
docker run -d \
  --name myapp \
  --user 1000:1000 \
  --memory="512m" \
  --cpus="0.5" \
  --health-cmd="curl -f http://localhost/health || exit 1" \
  --health-interval=30s \
  --restart=unless-stopped \
  -p 127.0.0.1:8000:8000 \
  myapp:v1.2.3
```

**Bad pattern:**

```bash
# NEVER do this
docker run -d --privileged --user root -p 0.0.0.0:8000:8000 myapp:latest
```

### Stopping Containers

```bash
# Graceful stop (sends SIGTERM, waits, then SIGKILL)
docker stop myapp

# Force stop (immediate) - REQUIRES CONFIRMATION
docker rm -f myapp

# Stop and remove (safer)
docker stop myapp && docker rm myapp
```

---

## Image Management

### Tagging Strategy

```bash
# Production: Semantic versioning
myapp:v1.2.3
myapp:v1.2.3-20260214

# Staging: Environment-specific
myapp:staging
myapp:staging-v1.2.3

# Development: Branch or feature
myapp:dev
myapp:feature-auth

# NEVER in production
myapp:latest
```

### Registry Operations

```bash
# Tag for registry
docker tag myapp:v1.2.3 myregistry.io/myapp:v1.2.3

# Login to registry
docker login myregistry.io

# Push to registry - REQUIRES CONFIRMATION
docker push myregistry.io/myapp:v1.2.3

# Pull from registry
docker pull myregistry.io/myapp:v1.2.3
```

### Image Cleanup

```bash
# Remove specific image - REQUIRES CONFIRMATION
docker rmi myapp:old-version

# Remove dangling images (untagged) - REQUIRES CONFIRMATION
docker image prune

# Remove all unused images - REQUIRES CONFIRMATION
docker image prune -a

# Show disk usage
docker system df
```

---

## Docker Compose

### Project Structure

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    image: myapp:v1.2.3
    container_name: myapp-web
    restart: unless-stopped
    user: "1000:1000"
    mem_limit: 512m
    cpus: 0.5
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      - ENV=production
    env_file:
      - .env
    volumes:
      - app-data:/app/data
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    container_name: myapp-db
    restart: unless-stopped
    user: postgres
    mem_limit: 1g
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  app-data:
  db-data:

networks:
  app-network:
    driver: bridge

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Compose Operations

```bash
# Start services
docker-compose up -d

# Start specific service
docker-compose up -d web

# View logs
docker-compose logs -f web

# Stop services (preserves containers)
docker-compose stop

# Stop and remove containers (preserves volumes and images)
docker-compose down

# Stop, remove containers AND volumes - REQUIRES CONFIRMATION
docker-compose down -v

# Rebuild and start
docker-compose up -d --build

# Scale service
docker-compose up -d --scale web=3
```

---

## Volume Management

### Creating Volumes

```bash
# Create named volume
docker volume create mydata

# Create with driver options
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/dir \
  mydata
```

### Using Volumes

```bash
# Mount named volume
docker run -d -v mydata:/app/data myapp:v1.2.3

# Mount bind mount (development)
docker run -d -v $(pwd)/data:/app/data myapp:dev

# Mount read-only
docker run -d -v mydata:/app/data:ro myapp:v1.2.3
```

### Volume Inspection

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Find what containers use a volume
docker ps -a --filter volume=mydata
```

### Volume Cleanup - REQUIRES CONFIRMATION

```bash
# Remove specific volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune
```

---

## Network Management

### Creating Networks

```bash
# Create bridge network
docker network create mynetwork

# Create with specific subnet
docker network create \
  --driver bridge \
  --subnet=172.18.0.0/16 \
  --gateway=172.18.0.1 \
  mynetwork

# Create overlay network (Swarm)
docker network create \
  --driver overlay \
  --attachable \
  mynetwork
```

### Using Networks

```bash
# Connect container to network
docker network connect mynetwork myapp

# Disconnect from network
docker network disconnect mynetwork myapp

# Run container on specific network
docker run -d --network mynetwork myapp:v1.2.3
```

### Network Inspection

```bash
# List networks
docker network ls

# Inspect network
docker network inspect mynetwork
```

### Network Cleanup - REQUIRES CONFIRMATION

```bash
# Remove network
docker network rm mynetwork

# Remove unused networks
docker network prune
```

---

## Security Best Practices

### 1. Non-Root User

**Dockerfile:**

```dockerfile
# Create user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set ownership
RUN chown -R appuser:appuser /app

# Switch to user
USER appuser
```

**Run command:**

```bash
docker run -d --user 1000:1000 myapp:v1.2.3
```

### 2. Read-Only Filesystem

```bash
# Run with read-only root filesystem
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myapp:v1.2.3
```

### 3. Dropped Capabilities

```bash
# Drop all capabilities, add only what's needed
docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp:v1.2.3
```

### 4. Resource Limits

```bash
# Memory and CPU limits
docker run -d \
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="0.5" \
  --pids-limit=100 \
  myapp:v1.2.3
```

### 5. Secrets Management

```bash
# Docker secrets (Swarm mode)
echo "my-secret-password" | docker secret create db_password -

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:v1.2.3

# Environment variables (less secure, OK for dev)
docker run -d --env-file .env myapp:dev

# NEVER hardcode in Dockerfile
```

### 6. Image Scanning

```bash
# Scan image for vulnerabilities
docker scan myapp:v1.2.3

# Use Trivy for scanning
trivy image myapp:v1.2.3

# Scan during build (in CI/CD)
docker build -t myapp:v1.2.3 .
docker scan myapp:v1.2.3 || exit 1
```

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| Using :latest in production | Unpredictable, breaks reproducibility | Use semantic versioning (v1.2.3) |
| Running as root | Security vulnerability | Create non-root user in Dockerfile |
| No health checks | Can't detect unhealthy containers | Add HEALTHCHECK directive |
| Large images (>1GB) | Slow pulls, larger attack surface | Multi-stage builds, minimal base images |
| Hardcoded secrets | Credential exposure in image layers | Use Docker secrets or env files |
| No resource limits | Container can consume all host resources | Set --memory and --cpus limits |
| Exposing all ports | Unnecessary network exposure | Expose only required ports |
| docker-compose down -v without backup | Data loss | Backup volumes before removing |
| Building in production | Security risk, slow | Build in CI/CD, pull from registry |
| --net=host | Bypasses container isolation | Use bridge networks |
| --privileged without justification | Major security hole | Use specific capabilities instead |

---

## Dockerfile Best Practices

### Layer Optimization

```dockerfile
# Good: Dependencies change less often than code
FROM python:3.11-slim

# Install system dependencies (rarely changes)
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (changes occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code (changes frequently)
COPY . .

# Bad: Code changes invalidate dependency layer
FROM python:3.11-slim
COPY . .
RUN pip install -r requirements.txt
```

### Minimal Base Images

```dockerfile
# Good: Alpine (5MB)
FROM python:3.11-alpine

# Good: Slim (41MB)
FROM python:3.11-slim

# Acceptable: Standard (300MB+)
FROM python:3.11

# Bad: Full OS (900MB+)
FROM ubuntu:22.04
RUN apt-get install python3
```

### Multi-Stage Builds

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine
WORKDIR /app
# Copy only production files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

---

## Environment Detection

Identify environment from:

1. **Container name patterns:**
   - `*-dev`, `*-development` → Development
   - `*-staging`, `*-stg` → Staging
   - `*-pilot`, `*-uat` → Pilot/UAT
   - `*-prod`, `*-production` → Production
   - No environment indicator → **Assume production**

2. **Docker Compose project name:**
   ```bash
   docker-compose -p myapp-dev up -d  # Development
   docker-compose -p myapp-prod up -d  # Production
   ```

3. **Container labels:**
   ```bash
   docker inspect myapp | grep Labels -A 5
   # Look for "environment=prod" or similar
   ```

4. **Network names:**
   - `myapp_default` → No environment specified (assume prod)
   - `myapp-dev_default` → Development
   - `myapp-prod_default` → Production

---

## Audit Trail

All Docker operations are logged to:

```
~/.claude/logs/docker-operations.log  # Detailed log
~/.claude/logs/docker-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-02-14 10:30:45] ACTION=ALLOW COMMAND="docker ps" RESULT=read-only USER=donmccarty
[2026-02-14 10:31:02] ACTION=BLOCK COMMAND="docker run --privileged" RESULT=security-violation USER=donmccarty
[2026-02-14 10:32:15] ACTION=CONFIRM COMMAND="docker rm -f myapp-prod" RESULT=confirmed USER=donmccarty
```

---

## Quick Reference

### Daily Commands (Safe)

```bash
docker ps                        # List running containers
docker ps -a                     # List all containers
docker images                    # List images
docker logs myapp                # View container logs
docker logs -f myapp             # Follow logs
docker stats                     # Resource usage
docker-compose ps                # List compose services
docker-compose logs -f           # Follow compose logs
```

### Build and Run

```bash
# Build image
docker build -t myapp:v1.2.3 .

# Run container
docker run -d \
  --name myapp \
  --user 1000:1000 \
  --memory="512m" \
  --cpus="0.5" \
  -p 127.0.0.1:8000:8000 \
  myapp:v1.2.3

# Start compose stack
docker-compose up -d
```

### Cleanup Commands (Require Confirmation)

```bash
docker stop myapp                # Stop container
docker rm myapp                  # Remove container
docker rmi myapp:old             # Remove image
docker volume prune              # Remove unused volumes
docker system prune              # Remove unused everything
docker-compose down              # Stop and remove containers
docker-compose down -v           # Also remove volumes
```

### Inspection Commands

```bash
docker inspect myapp             # Container details
docker logs myapp                # Container logs
docker top myapp                 # Container processes
docker stats myapp               # Resource usage
docker exec -it myapp sh         # Shell into container
```

---

## References

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
