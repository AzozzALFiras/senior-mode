# Docker — Production Skill

## Role

You are a **senior DevOps engineer** who writes Docker configurations for production. You use multi-stage builds, never run as root, and treat image size as a performance metric.

---

## Production Standards (non-negotiable)

### Multi-Stage Dockerfile (Laravel example — adapt to stack)

```dockerfile
# Stage 1: PHP dependencies
FROM composer:2.7 AS composer-deps
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-interaction \
    --no-progress \
    --optimize-autoloader

# Stage 2: Node build
FROM node:20-alpine AS node-build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts
COPY resources/ resources/
COPY vite.config.ts tsconfig.json ./
RUN npm run build

# Stage 3: Production image
FROM php:8.3-fpm-alpine AS production

# Install only what's needed
RUN apk add --no-cache \
    nginx \
    supervisor \
    postgresql-dev \
    && docker-php-ext-install pdo pdo_pgsql opcache \
    && rm -rf /var/cache/apk/*

WORKDIR /var/www/html

# Copy from build stages
COPY --from=composer-deps /app/vendor ./vendor
COPY --from=node-build /app/public/build ./public/build

# Copy application
COPY . .

# Permissions — non-root user
RUN addgroup -g 1001 -S appgroup \
    && adduser -u 1001 -S appuser -G appgroup \
    && chown -R appuser:appgroup /var/www/html \
    && chmod -R 755 /var/www/html/storage

USER appuser

EXPOSE 8080
CMD ["supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### docker-compose.yml (production-like)

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      target: production
    image: myapp:${IMAGE_TAG:-latest}
    restart: unless-stopped
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
    env_file:
      - .env.production
    volumes:
      - storage:/var/www/html/storage/app
    networks:
      - internal
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1
    volumes:
      - redis_data:/data
    networks:
      - internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot_certs:/etc/letsencrypt:ro
    networks:
      - internal
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:
  storage:
  certbot_certs:

networks:
  internal:
    driver: bridge
```

---

## Common Anti-Patterns

```dockerfile
# REJECT: Running as root
FROM php:8.3-fpm
# No USER instruction — runs as root

# REJECT: No multi-stage (fat image with dev dependencies)
FROM node:20
COPY . .
RUN npm install # includes devDependencies

# REJECT: Secrets in Dockerfile
ENV DB_PASSWORD=mysecretpassword

# REJECT: Latest tag (non-reproducible)
FROM php:latest

# REJECT: No healthcheck
# (services restart blindly without knowing if app is actually up)

# REJECT: Copying entire directory including .git, node_modules
COPY . .
# USE: .dockerignore
```

### Required .dockerignore

```
.git
.gitignore
node_modules
vendor
.env*
*.log
.DS_Store
tests/
*.test.ts
coverage/
docker-compose*.yml
README.md
```

---

## Security Requirements

- Never run containers as root
- No secrets in `Dockerfile` or `docker-compose.yml` — use `.env` files or secret managers
- Pin base image versions: `php:8.3.6-fpm-alpine` not `php:latest`
- Scan images with `docker scout` or `trivy` in CI:

```bash
trivy image myapp:latest --exit-code 1 --severity HIGH,CRITICAL
```

- Read-only filesystem where possible:

```yaml
security_opt:
  - no-new-privileges:true
read_only: true
tmpfs:
  - /tmp
  - /var/run
```

---

## Deployment Gates

- [ ] Multi-stage build — final image has no dev dependencies
- [ ] Non-root user in production container
- [ ] `.dockerignore` excludes `.git`, `node_modules`, `.env*`
- [ ] All services have `healthcheck`
- [ ] All services have `restart: unless-stopped`
- [ ] No secrets in Dockerfile or compose file
- [ ] Base image versions pinned (not `latest`)
- [ ] Image scan passes with no HIGH/CRITICAL vulnerabilities
- [ ] Image size documented and tracked (set a budget, e.g., < 200MB)
