# Docker Patterns

## Multi-Stage Build (Node.js)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app

# Install dependencies first (better caching)
COPY package*.json ./
RUN npm ci --only=production

# Copy source and build
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app

# Security: non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy only necessary files
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q --spider http://localhost:3000/health || exit 1

# Start application
CMD ["node", "dist/main.js"]
```

## Multi-Stage Build (Go)

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Production stage (distroless)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/main /
EXPOSE 8080
CMD ["/main"]
```

## Multi-Stage Build (Python)

```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt > requirements.txt

# Production stage
FROM python:3.12-slim
WORKDIR /app

# Create non-root user
RUN useradd -m -u 1001 appuser

# Install dependencies
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## .dockerignore

```
# Dependencies
node_modules
.pnpm-store

# Build output
dist
build
.next

# Development
.env*.local
.git
.gitignore

# IDE
.idea
.vscode
*.swp

# Testing
coverage
.nyc_output

# Documentation
*.md
docs

# OS
.DS_Store
Thumbs.db

# Docker
Dockerfile*
docker-compose*
.dockerignore
```

## Docker Compose (Development)

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder  # Use builder stage for hot reload
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/app
      - REDIS_URL=redis://redis:6379
    volumes:
      - .:/app
      - /app/node_modules  # Exclude node_modules
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Optional: Database admin
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  postgres_data:
  redis_data:
```

## Docker Compose (Production)

```yaml
version: '3.8'

services:
  app:
    image: ${REGISTRY}/myapp:${TAG:-latest}
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
```

## Health Check Patterns

### HTTP Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q --spider http://localhost:3000/health || exit 1
```

### TCP Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
    CMD nc -z localhost 3000 || exit 1
```

### Custom Script

```dockerfile
COPY healthcheck.sh /
RUN chmod +x /healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s CMD /healthcheck.sh
```

```bash
#!/bin/sh
# healthcheck.sh
curl -f http://localhost:3000/health || exit 1
```

## Layer Caching Optimization

```dockerfile
# GOOD: Dependencies first (changes less often)
COPY package*.json ./
RUN npm ci
COPY . .

# BAD: Everything at once (no caching benefit)
COPY . .
RUN npm ci
```

## Security Best Practices

### Run as Non-Root

```dockerfile
# Create user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Set ownership
COPY --chown=appuser:appgroup . .

# Switch user
USER appuser
```

### Read-Only Filesystem

```yaml
# docker-compose.yml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

### No New Privileges

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

### Scan for Vulnerabilities

```bash
# Docker Scout
docker scout quickview myimage:latest
docker scout cves myimage:latest

# Trivy
trivy image myimage:latest

# Snyk
snyk container test myimage:latest
```

## Build Arguments and Secrets

```dockerfile
# Build args (visible in image history)
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

# Build secrets (not in final image) - Docker BuildKit
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```

```bash
# Build with secret
docker build --secret id=npm_token,src=.npmrc -t myapp .
```

## Common Commands

```bash
# Build
docker build -t myapp:latest .
docker build --target builder -t myapp:dev .

# Run
docker run -d -p 3000:3000 --name myapp myapp:latest
docker run --rm -it myapp:latest sh  # Debug shell

# Logs
docker logs -f myapp
docker logs --tail 100 myapp

# Exec into container
docker exec -it myapp sh

# Cleanup
docker system prune -a  # Remove all unused
docker volume prune     # Remove unused volumes

# Push to registry
docker tag myapp:latest registry.example.com/myapp:latest
docker push registry.example.com/myapp:latest
```

## Multi-Platform Builds

```bash
# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 \
    -t myapp:latest --push .
```

```dockerfile
# Platform-aware Dockerfile
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"
```
