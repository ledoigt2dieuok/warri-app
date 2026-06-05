# Phase 5: CI/CD & Deployment Pipeline - Detailed Implementation Guide

**Date Created**: June 5, 2026  
**Status**: Ready for Implementation  
**Target Duration**: 7-10 weeks

---

## Overview

Phase 5 focuses on setting up automated deployment pipelines, monitoring, and production-ready infrastructure.

### Goals
1. GitHub Actions CI/CD workflows
2. Multi-environment configuration
3. Container deployment
4. Database management
5. Monitoring & logging
6. Deployment documentation

---

## 1. GitHub Actions Workflows 🔄

### 1.1 Test Workflow

**File**: `.github/workflows/test.yml`

```yaml
name: Tests

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: warri_test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run frontend tests
        run: npm run test

      - name: Run backend tests
        run: npm run test:api

      - name: Generate coverage report
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          flags: unittests
          name: codecov-umbrella

      - name: Comment PR with coverage
        if: github.event_name == 'pull_request'
        uses: romeovs/lcov-reporter-action@v0.3.1
        with:
          lcov-file: ./coverage/lcov.info
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### 1.2 Code Quality Workflow

**File**: `.github/workflows/code-quality.yml`

```yaml
name: Code Quality

on:
  pull_request:
    branches: [main, develop]

jobs:
  quality:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run type-check

      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

      - name: Security audit
        run: npm audit --audit-level=moderate

      - name: SAST scan
        uses: github/super-linter@v4
        env:
          DEFAULT_BRANCH: main
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 1.3 Staging Deployment

**File**: `.github/workflows/deploy-staging.yml`

```yaml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build backend
        run: npm run backend:build

      - name: Run migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}

      - name: Build Docker image
        run: docker build -t warri-app:staging .

      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag warri-app:staging ${{ secrets.DOCKER_REGISTRY }}/warri-app:staging
          docker push ${{ secrets.DOCKER_REGISTRY }}/warri-app:staging

      - name: Deploy to staging
        run: |
          curl -X POST ${{ secrets.STAGING_WEBHOOK_URL }} \
            -H "Authorization: Bearer ${{ secrets.STAGING_DEPLOY_TOKEN }}" \
            -d '{"ref":"develop"}'

      - name: Run health check
        run: |
          for i in {1..30}; do
            if curl -f https://staging-api.wari-app.com/health; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          exit 1

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Staging deployment ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Deploy to Staging: ${{ job.status }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View workflow>"
                  }
                }
              ]
            }
```

### 1.4 Production Deployment

**File**: `.github/workflows/deploy-production.yml`

```yaml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v3
        with:
          ref: v${{ github.event.inputs.version }}

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build backend
        run: npm run backend:build

      - name: Build Docker image
        run: docker build -t warri-app:${{ github.event.inputs.version }} .

      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag warri-app:${{ github.event.inputs.version }} ${{ secrets.DOCKER_REGISTRY }}/warri-app:latest
          docker push ${{ secrets.DOCKER_REGISTRY }}/warri-app:latest

      - name: Blue-Green Deployment
        run: |
          # Deploy to green environment
          kubectl set image deployment/warri-app-green \
            warri-app=${{ secrets.DOCKER_REGISTRY }}/warri-app:latest \
            --kubeconfig=${{ secrets.KUBECONFIG }}

          # Wait for health checks
          kubectl rollout status deployment/warri-app-green \
            --kubeconfig=${{ secrets.KUBECONFIG }} \
            --timeout=5m

          # Switch traffic to green
          kubectl patch service warri-app \
            -p '{"spec":{"selector":{"version":"green"}}}' \
            --kubeconfig=${{ secrets.KUBECONFIG }}

      - name: Health check
        run: |
          for i in {1..30}; do
            if curl -f https://api.wari-app.com/health; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          exit 1

      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.event.inputs.version }}
          release_name: Release ${{ github.event.inputs.version }}
          body: |
            Deployed to production: ${{ github.event.inputs.version }}

      - name: Notify team
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "🚀 Production deployment v${{ github.event.inputs.version }} successful"
            }
```

---

## 2. Environment Configuration 🔧

### 2.1 Environment Files

**Development** (`.env.example`):

```bash
# Frontend
API_URL=http://localhost:3000
ENVIRONMENT=development
LOG_LEVEL=debug

# Backend
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/warri_dev
REDIS_URL=redis://localhost:6379

# Security
JWT_SECRET=dev-secret-key-change-in-production
JWT_REFRESH_SECRET=dev-refresh-secret-change-in-production
ENCRYPTION_KEY=dev-encryption-key-change-in-production

# External Services
SENTRY_DSN=
STRIPE_SECRET_KEY=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

**Staging** (`scripts/env/staging.env`):

```bash
# Frontend
API_URL=https://staging-api.wari-app.com
ENVIRONMENT=staging
LOG_LEVEL=info

# Backend
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://user:password@staging-db.example.com/warri_staging
REDIS_URL=redis://staging-redis.example.com:6379

# Security
JWT_SECRET=<staging-secret>
JWT_REFRESH_SECRET=<staging-refresh-secret>
ENCRYPTION_KEY=<staging-encryption-key>

# External Services
SENTRY_DSN=https://<key>@sentry.io/<project-id>
STRIPE_SECRET_KEY=sk_test_...
AWS_ACCESS_KEY_ID=<staging-key>
AWS_SECRET_ACCESS_KEY=<staging-secret>
```

**Production** (`scripts/env/production.env` - NEVER COMMIT):

```bash
# Frontend
API_URL=https://api.wari-app.com
ENVIRONMENT=production
LOG_LEVEL=warn

# Backend
NODE_ENV=production
PORT=3000
DATABASE_URL=<production-db-url>
REDIS_URL=<production-redis-url>

# Security
JWT_SECRET=<production-secret>
JWT_REFRESH_SECRET=<production-refresh-secret>
ENCRYPTION_KEY=<production-encryption-key>

# External Services
SENTRY_DSN=<production-sentry-dsn>
STRIPE_SECRET_KEY=sk_live_...
AWS_ACCESS_KEY_ID=<prod-key>
AWS_SECRET_ACCESS_KEY=<prod-secret>
```

---

## 3. Docker Configuration 📦

### 3.1 Multi-stage Dockerfile

**File**: `Dockerfile`

```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

# Frontend build
FROM node:18 AS frontend-builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY src ./src
COPY tsconfig.json ./
RUN npm run build

# Runtime stage
FROM node:18-alpine

WORKDIR /app

# Security
RUN apk add --no-cache dumb-init
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs backend ./backend
COPY --chown=nodejs:nodejs dist ./dist
COPY --chown=nodejs:nodejs package*.json ./

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["npm", "run", "backend:prod"]
```

### 3.2 Docker Compose

**File**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://warri:password@postgres:5432/warri_dev
      REDIS_URL: redis://redis:6379
      NODE_ENV: development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src
      - ./backend:/app/backend

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: warri
      POSTGRES_PASSWORD: password
      POSTGRES_DB: warri_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U warri"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

## 4. Database Management 🗄️

### 4.1 Backup Script

**File**: `scripts/backup.sh`

```bash
#!/bin/bash

set -e

BACKUP_DIR="./backups"
DATABASE_URL="${DATABASE_URL:=postgresql://user:password@localhost/warri}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/warri_db_$TIMESTAMP.sql"

mkdir -p "$BACKUP_DIR"

echo "Starting database backup..."

# Backup database
pg_dump "$DATABASE_URL" -f "$BACKUP_FILE"

# Compress backup
gzip "$BACKUP_FILE"

echo "Backup completed: ${BACKUP_FILE}.gz"

# Keep only last 7 days of backups
find "$BACKUP_DIR" -name "warri_db_*.sql.gz" -mtime +7 -delete

echo "Cleanup completed"
```

### 4.2 Restore Script

**File**: `scripts/restore.sh`

```bash
#!/bin/bash

set -e

if [ -z "$1" ]; then
  echo "Usage: ./restore.sh <backup-file>"
  exit 1
fi

DATABASE_URL="${DATABASE_URL:=postgresql://user:password@localhost/warri}"
BACKUP_FILE="$1"

echo "Restoring database from $BACKUP_FILE..."

# Handle compressed files
if [[ "$BACKUP_FILE" == *.gz ]]; then
  gunzip -c "$BACKUP_FILE" | psql "$DATABASE_URL"
else
  psql "$DATABASE_URL" < "$BACKUP_FILE"
fi

echo "Database restored successfully"
```

---

## 5. Monitoring & Logging 📊

### 5.1 Logging Service

**File**: `backend/services/loggingService.ts`

```typescript
import winston from 'winston';
import * as Sentry from '@sentry/node';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple(),
    }),
  ],
});

export const logError = (error: Error, context?: any) => {
  logger.error({
    message: error.message,
    stack: error.stack,
    context,
  });

  Sentry.captureException(error, { contexts: { app: context } });
};

export const logInfo = (message: string, context?: any) => {
  logger.info({ message, context });
};

export const logWarning = (message: string, context?: any) => {
  logger.warn({ message, context });
};

export default logger;
```

### 5.2 Health Check Endpoint

**File**: `backend/routes/health.ts`

```typescript
import { Router } from 'express';
import { db } from '../config/database';
import redis from '../config/redis';

const router = Router();

router.get('/health', async (req, res) => {
  const checks = {
    database: false,
    redis: false,
    uptime: process.uptime(),
  };

  try {
    // Check database
    await db.raw('SELECT 1');
    checks.database = true;
  } catch (error) {
    console.error('Database health check failed:', error);
  }

  try {
    // Check Redis
    await redis.ping();
    checks.redis = true;
  } catch (error) {
    console.error('Redis health check failed:', error);
  }

  const status = checks.database && checks.redis ? 200 : 503;

  res.status(status).json({
    status: status === 200 ? 'healthy' : 'unhealthy',
    checks,
  });
});

export default router;
```

---

## 6. Deployment Checklist 📋

**Pre-Deployment**:

- [ ] All tests passing
- [ ] Code review approved
- [ ] No security vulnerabilities
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Secrets properly managed
- [ ] Documentation updated

**Deployment**:

- [ ] Deploy to staging
- [ ] Run smoke tests
- [ ] Health checks pass
- [ ] Monitor metrics
- [ ] Deploy to production
- [ ] Monitor logs & errors
- [ ] Notify stakeholders

**Post-Deployment**:

- [ ] Verify all services healthy
- [ ] Check API response times
- [ ] Monitor error rates
- [ ] Validate user flows
- [ ] Document any issues
- [ ] Plan rollback if needed

---

## Phase 5 Summary

| Task | Timeline | Status |
|------|----------|--------|
| GitHub Actions | 2-3 wks | Core workflows |
| Environment Config | 1 wk | Dev/Staging/Prod |
| Container Setup | 1 wk | Docker & Compose |
| Database Management | 1 wk | Backup/Restore |
| Monitoring | 1-2 wks | Logging & Health |
| Documentation | 1 wk | Deployment Guide |

**Total Phase 5**: 7-10 weeks

---

## Success Metrics

✅ **Automation**:
- All tests run on PR
- Automated staging deployment
- One-click production deployment

✅ **Reliability**:
- 99.9% uptime SLA
- <1 min MTTD
- <5 min MTTR

✅ **Monitoring**:
- All errors logged
- Real-time alerts
- Performance metrics tracked

---

**Ready for production deployment!** 🚀
