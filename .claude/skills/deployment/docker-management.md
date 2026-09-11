---
name: docker-management
description: Docker best practices for the DevOps Engineer. Covers Dockerfile authoring, multi-stage builds, docker-compose, image security, and container runtime patterns.
---

# Docker Management

## Dockerfile Best Practices

### Base Images
- Always pin to a specific version tag — never `:latest` in production (`node:20-alpine`, not `node:latest`)
- Prefer minimal base images: `alpine` variants reduce attack surface and image size
- For compiled languages (Go, Rust), use a build stage with the full SDK and a runtime stage from `scratch` or `distroless`
- For interpreted languages (Python, Node), use the slim or alpine variant for the runtime stage

### Multi-Stage Builds
Split every Dockerfile into at minimum two stages:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime
FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
```

Benefits: build tools don't ship in the final image, secrets used during build don't persist, final image is smaller.

### Layer Caching
Order instructions from least to most frequently changed:
1. Base image
2. System dependencies (`apt-get`, `apk add`)
3. Dependency manifest (package.json, requirements.txt, go.mod)
4. Install dependencies (`npm ci`, `pip install`, `go mod download`)
5. Application source code (`COPY . .`)

This ensures that a code change doesn't invalidate the dependency installation layer.

### Security
- **Non-root user** — always create and switch to a non-root user for the runtime stage:
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ```
- **Read-only filesystem** — set `--read-only` at runtime where possible; use named volumes for writable paths
- **No secrets in image** — build args and ENV values are visible in image history; inject secrets at runtime via environment variables
- **Minimal packages** — do not install debugging tools in production images

### HEALTHCHECK
Always include a HEALTHCHECK directive that matches the application's health endpoint:

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:${PORT:-8080}/health || exit 1
```

### .dockerignore
Always create a `.dockerignore` at the project root. Minimum contents:
```
.git
.gitignore
.env
.env.*
!.env.example
node_modules
__pycache__
*.pyc
.pytest_cache
dist
build
*.log
.DS_Store
README.md
_specs
_retrospective
```

### EXPOSE and PORT
```dockerfile
ARG PORT=8080
ENV PORT=${PORT}
EXPOSE ${PORT}
```

---

## docker-compose.yml

### Structure
```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:local
    ports:
      - "${PORT:-8080}:${PORT:-8080}"
    env_file:
      - .env
    environment:
      - PORT=${PORT:-8080}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:${PORT:-8080}/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - app_data:/app/data   # only if persistent storage is needed
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  app_data:
  db_data:
```

### .env.example
Always create `.env.example` alongside `docker-compose.yml`. List every variable with a description but no real value:
```
# Application
PORT=8080
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://user:password@db:5432/dbname
DB_NAME=myapp
DB_USER=myapp
DB_PASSWORD=changeme
```

Never commit `.env` — add it to `.gitignore`. Always commit `.env.example`.

---

## Running Containers

### Naming
Use consistent naming: `<project>-<service>-<environment>` (e.g., `myapp-api-staging`).

### Logs
```bash
docker logs <container> --follow --tail 100
```

For production, configure a log driver that ships to your logging infrastructure:
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

### Resource Limits
Set memory and CPU limits in production to prevent a single container from exhausting the host:
```yaml
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 512M
    reservations:
      memory: 256M
```

### Image Tagging Convention
- Development/local: `:local`
- CI builds: `:<git-commit-sha>` (first 8 characters)
- Releases: `:<semver>` and also `:latest` as an alias
- Never deploy `:latest` to production — always use a specific commit or version tag
