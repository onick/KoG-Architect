# DevOps, Kubernetes & CI/CD for KOG Monorepo

## Docker Architecture

### Multi-Stage Build Template

```dockerfile
# Base stage (shared across services)
FROM node:18-alpine AS base
WORKDIR /app
RUN apk add --no-cache dumb-init

# Dependencies stage
FROM base AS deps
COPY package.json package-lock.json ./
COPY packages/shared-types/package.json ./packages/shared-types/
COPY services/game-engine/package.json ./services/game-engine/
RUN npm ci --production=false

# Build stage
FROM deps AS build
COPY packages/ ./packages/
COPY services/game-engine/ ./services/game-engine/
COPY tsconfig.json ./
RUN npm run build -w services/game-engine

# Production stage
FROM base AS production
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/packages/shared-types/dist ./packages/shared-types/dist

USER node
EXPOSE 3000
CMD ["dumb-init", "node", "dist/main.js"]
```

### Key Rules
- ALWAYS use multi-stage builds (smaller images, no dev deps in prod)
- Use `dumb-init` for proper signal handling (SIGTERM for graceful shutdown)
- Run as non-root user (`USER node`)
- Copy only compiled output to production stage

## Docker Compose (Local Development)

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
    volumes: ['redis-data:/data']
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru

  cockroachdb:
    image: cockroachdb/cockroach:latest
    command: start-single-node --insecure
    ports: ['26257:26257', '8080:8080']
    volumes: ['crdb-data:/cockroach/cockroach-data']

  mongodb:
    image: mongo:7
    ports: ['27017:27017']
    volumes: ['mongo-data:/data/db']

  game-engine:
    build:
      context: .
      dockerfile: services/game-engine/Dockerfile
    ports: ['3001:3000']
    depends_on: [redis, cockroachdb]
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://root@cockroachdb:26257/kog?sslmode=disable
      - NODE_ENV=development
    volumes:
      - ./services/game-engine/src:/app/services/game-engine/src  # Hot reload

  clock-service:
    build:
      context: .
      dockerfile: services/clock-service/Dockerfile
    depends_on: [redis]
    environment:
      - REDIS_URL=redis://redis:6379

  websocket-gateway:
    build:
      context: .
      dockerfile: services/websocket-gateway/Dockerfile
    ports: ['3002:3000']
    depends_on: [redis, game-engine]
    environment:
      - REDIS_URL=redis://redis:6379

  anticheat-service:
    build:
      context: .
      dockerfile: services/anticheat-service/Dockerfile
    depends_on: [redis, mongodb]
    environment:
      - REDIS_URL=redis://redis:6379
      - MONGO_URL=mongodb://mongodb:27017/kog
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G  # Stockfish needs memory

  api-gateway:
    build:
      context: .
      dockerfile: services/api-gateway/Dockerfile
    ports: ['3000:3000']
    depends_on: [game-engine, websocket-gateway]

  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile
    ports: ['8080:3000']
    depends_on: [api-gateway]

volumes:
  redis-data:
  crdb-data:
  mongo-data:
```

## Kubernetes Deployment

### Service Deployment Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-engine
  labels:
    app: kog
    service: game-engine
    tier: critical
spec:
  replicas: 3
  selector:
    matchLabels:
      service: game-engine
  template:
    metadata:
      labels:
        service: game-engine
    spec:
      containers:
        - name: game-engine
          image: ghcr.io/kog/game-engine:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          env:
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: kog-secrets
                  key: redis-url
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: kog-secrets
                  key: database-url
```

### Resource Budgets per Service

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit | Replicas |
|---------|-----------|-----------|---------------|-------------|----------|
| game-engine | 250m | 1000m | 256Mi | 512Mi | 3 |
| clock-service | 100m | 500m | 128Mi | 256Mi | 2 |
| websocket-gateway | 500m | 2000m | 512Mi | 1Gi | 3+ |
| anticheat-service | 1000m | 4000m | 512Mi | 2Gi | 2 |
| matchmaking | 100m | 500m | 128Mi | 256Mi | 2 |
| api-gateway | 250m | 1000m | 256Mi | 512Mi | 3 |
| web (Next.js) | 250m | 1000m | 256Mi | 512Mi | 2 |

### WebSocket Gateway: Sticky Sessions

```yaml
apiVersion: v1
kind: Service
metadata:
  name: websocket-gateway
  annotations:
    # For AWS ALB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP  # Sticky sessions for WebSocket
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
```

## GitHub Actions CI/CD

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --coverage --ci
      - uses: codecov/codecov-action@v3

  build-and-push:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [game-engine, clock-service, websocket-gateway, anticheat-service, matchmaking, api-gateway, web]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: services/${{ matrix.service }}/Dockerfile
          push: true
          tags: ghcr.io/kog/${{ matrix.service }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    steps:
      - run: |
          # Update K8s deployment with new image tag
          kubectl set image deployment/game-engine \
            game-engine=ghcr.io/kog/game-engine:${{ github.sha }}
          kubectl rollout status deployment/game-engine --timeout=120s
```

## Monitoring: Prometheus + Grafana

### Key Metrics to Expose

```typescript
// Custom NestJS metrics middleware
import { Counter, Histogram, Gauge } from 'prom-client';

export const activeGames = new Gauge({
  name: 'kog_active_games_total',
  help: 'Number of active games',
});

export const wsConnections = new Gauge({
  name: 'kog_websocket_connections',
  help: 'Current WebSocket connections',
  labelNames: ['gateway_instance'],
});

export const moveLatency = new Histogram({
  name: 'kog_move_processing_seconds',
  help: 'Move processing latency',
  buckets: [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1],
});

export const anticheatQueueDepth = new Gauge({
  name: 'kog_anticheat_queue_depth',
  help: 'Jobs waiting in anticheat queue',
});
```

### Alert Rules

| Metric | Condition | Severity |
|--------|----------|----------|
| Move latency p99 | > 200ms for 5 min | P1 |
| WebSocket connections | Drop > 50% in 1 min | P0 |
| Anticheat queue depth | > 500 for 10 min | P2 |
| Error rate | > 5% on any service | P1 |
| Pod restarts | > 3 in 15 min | P1 |
| Redis memory | > 80% | P2 |
