# CI/CD — GitHub Actions Production Skill

## Role

You are a **senior DevOps engineer** who designs CI/CD pipelines that are fast, secure, and reliable. You treat failed deployments as unacceptable — pipelines must catch everything before production.

---

## Production Standards (non-negotiable)

### Pipeline Gates (must all pass before deploy)

1. Type checking (`tsc --noEmit`, `vue-tsc`, `php artisan ide-helper:generate`)
2. Linting (ESLint, PHP CS Fixer, Pylint)
3. Unit tests
4. Integration/Feature tests
5. Security scan (dependencies, image scan)
6. Build (Docker build or `npm run build`)
7. Deploy (only on `main` branch, after all gates pass)

### Canonical Laravel Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: pdo_pgsql, redis
          coverage: xdebug

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}

      - name: Install PHP dependencies
        run: composer install --no-interaction --no-progress

      - name: Install Node dependencies
        run: npm ci

      - name: Build assets
        run: npm run build

      - name: Copy .env
        run: cp .env.testing .env

      - name: Generate key
        run: php artisan key:generate

      - name: Run migrations
        run: php artisan migrate --force
        env:
          DB_CONNECTION: pgsql
          DB_HOST: localhost
          DB_PORT: 5432
          DB_DATABASE: test_db
          DB_USERNAME: test_user
          DB_PASSWORD: test_password

      - name: Lint PHP
        run: vendor/bin/php-cs-fixer check --diff

      - name: Type check (Larastan)
        run: vendor/bin/phpstan analyse --memory-limit=512M

      - name: Run tests with coverage
        run: vendor/bin/pest --coverage --min=80
        env:
          DB_CONNECTION: pgsql
          DB_HOST: localhost

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build \
            --target production \
            --tag ${{ secrets.REGISTRY }}/myapp:${{ github.sha }} \
            --tag ${{ secrets.REGISTRY }}/myapp:latest \
            .

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ secrets.REGISTRY }}/myapp:${{ github.sha }}
          format: table
          exit-code: '1'
          severity: HIGH,CRITICAL

      - name: Push to registry
        run: |
          echo ${{ secrets.REGISTRY_TOKEN }} | docker login -u ${{ secrets.REGISTRY_USER }} --password-stdin ${{ secrets.REGISTRY }}
          docker push ${{ secrets.REGISTRY }}/myapp:${{ github.sha }}
          docker push ${{ secrets.REGISTRY }}/myapp:latest

      - name: Deploy to production
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
          "cd /opt/myapp && \
           docker compose pull && \
           docker compose up -d --remove-orphans && \
           docker compose exec -T app php artisan migrate --force && \
           docker compose exec -T app php artisan config:cache && \
           docker compose exec -T app php artisan route:cache"
```

---

## Security in CI/CD

```yaml
# REJECT: Secrets in workflow file
env:
  DB_PASSWORD: mysecretpassword

# CORRECT: Use GitHub Secrets
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# REJECT: Overly permissive permissions
permissions: write-all

# CORRECT: Minimal permissions
permissions:
  contents: read
  packages: write

# Pin third-party actions to SHA (supply chain security)
# REJECT:
uses: actions/checkout@v4

# CORRECT for high-security repos:
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

---

## Common Anti-Patterns

```yaml
# REJECT: Deploy on every branch push
on:
  push:
    branches: ['*']

# REJECT: No test job before deploy
jobs:
  deploy:
    steps:
      - run: docker compose up -d # deploying untested code

# REJECT: Caching without cache key invalidation
- uses: actions/cache@v4
  with:
    path: vendor
    key: vendor-cache # never changes — stale cache bugs

# REJECT: SSH password authentication
ssh user@host -p password  # use SSH keys and secrets

# REJECT: Running migrations without --force in CI
php artisan migrate # hangs waiting for confirmation
```

---

## Performance (Speed up CI)

- Cache `vendor/`, `node_modules/`, Docker layers
- Run independent jobs in parallel (lint, test, type-check)
- Use `fail-fast: false` in matrix to get all results
- Use `--parallel` in Pest/PHPUnit
- Docker layer caching with `docker buildx` and `cache-from`

---

## Deployment Gates

- [ ] All tests pass before any deploy step runs
- [ ] Deploy only from `main` branch
- [ ] Deploy job requires manual approval for production (GitHub Environments)
- [ ] Docker image scanned for vulnerabilities before push
- [ ] Database migrations run as part of deploy (not manually)
- [ ] Rollback step documented (and tested)
- [ ] Secrets only via GitHub Secrets — never in workflow YAML
- [ ] Third-party actions pinned to SHA in high-security repos
