---
name: devops-patterns
description: DevOps patterns for infrastructure, CI/CD, and deployment automation. Use when configuring Docker containers, CI/CD pipelines, cloud deployments, Kubernetes, and monitoring. Covers GitHub Actions, Docker, Vercel, Railway, AWS, Terraform, and observability.
---

# DevOps Patterns

Modern DevOps patterns for infrastructure and deployment automation.

## Platform Selection

### Frontend Hosting

| Platform | Best For | Pricing |
|----------|----------|---------|
| Vercel | Next.js, React | Free tier, $20/user pro |
| Netlify | Static, Jamstack | Free tier, $19/user pro |
| Cloudflare Pages | Edge-first | Generous free tier |

### Backend Hosting

| Platform | Best For | Pricing |
|----------|----------|---------|
| Railway | Rapid deployment | $5 hobby, $20 pro + usage |
| Fly.io | Global edge | Pay-as-you-go (~$3/small VM) |
| Render | Managed services | Free tier, $7+ for services |
| AWS ECS | Enterprise scale | Usage-based |

### When to Use What

| Scenario | Recommendation |
|----------|----------------|
| Startup/MVP | Railway or Render |
| Next.js app | Vercel |
| Global low-latency | Fly.io |
| Enterprise/complex | AWS or GCP |
| Cost-sensitive | Fly.io or self-hosted |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Docker | `references/docker-patterns.md` | Containerization |
| CI/CD | `references/ci-cd.md` | GitHub Actions, pipelines |
| Cloud deployment | `references/cloud-deployment.md` | Vercel, Railway, AWS |
| Kubernetes | `references/kubernetes.md` | K8s manifests, Helm |
| Monitoring | `references/monitoring.md` | Prometheus, Grafana, logging |
| IaC | `references/infrastructure-as-code.md` | Terraform, Pulumi |

## Quick Start Patterns

### Dockerfile (Node.js)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
USER nodejs
EXPOSE 3000
HEALTHCHECK CMD wget -q --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/main.js"]
```

### GitHub Actions (CI)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build
```

### Docker Compose (Development)

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/app
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

## Environment Strategy

```
┌────────────────────────────────────────┐
│ Local Development                       │
│ - docker-compose up                    │
│ - .env.local                           │
├────────────────────────────────────────┤
│ Preview/Staging                         │
│ - Auto-deploy on PR                    │
│ - Unique preview URLs                  │
│ - Separate database                    │
├────────────────────────────────────────┤
│ Production                             │
│ - Deploy on main merge                 │
│ - Blue-green or rolling                │
│ - Production secrets                   │
└────────────────────────────────────────┘
```

## CI/CD Principles

1. **Fast feedback** - Quick tests first, fail fast
2. **Parallel execution** - Speed up pipelines
3. **Dependency caching** - Reduce build time
4. **Environment secrets** - Never commit credentials
5. **Automated deployments** - Push-to-deploy workflow
6. **Preview environments** - Test before merge

## Security Checklist

- [ ] Secrets in vault (not in code)
- [ ] Non-root container user
- [ ] Image vulnerability scanning
- [ ] HTTPS/TLS everywhere
- [ ] Least privilege IAM
- [ ] Environment isolation
- [ ] Audit logging enabled

## Container Best Practices

1. **Multi-stage builds** - Smaller final images
2. **Alpine/distroless base** - Minimal attack surface
3. **Non-root user** - Security isolation
4. **Health checks** - Container health monitoring
5. **.dockerignore** - Exclude unnecessary files
6. **Layer caching** - Optimize build order
