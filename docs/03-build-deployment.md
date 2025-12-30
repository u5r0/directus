# Build & Deployment

This document covers the build process, deployment strategies, and production configuration for Directus.

## Table of Contents
- [Build Process](#build-process)
- [Deployment Strategies](#deployment-strategies)
- [Environment Configuration](#environment-configuration)
- [Docker Deployment](#docker-deployment)
- [Production Optimization](#production-optimization)
- [Monitoring & Logging](#monitoring--logging)

## Build Process

### Development Build
```bash
# Install dependencies
pnpm install

# Build all packages
pnpm build

# Start development servers
pnpm --filter @directus/api dev
pnpm --filter @directus/app dev
```

### Production Build
```bash
# Clean build
rimraf dist

# Build for production
pnpm build

# Deploy to directory
pnpm --filter directus deploy --prod dist
```

### Build Output Structure
```
dist/
├── node_modules/          # Production dependencies
├── cli.js                 # CLI entry point
├── package.json           # Package manifest
├── api/                   # Built API code
│   ├── dist/              # Compiled JavaScript
│   └── package.json
└── app/                   # Built frontend
    ├── dist/              # Static assets
    └── index.html         # Entry HTML
```

## Deployment Strategies

### 1. Self-Hosted (Node.js)

#### Using PM2
```bash
# Install PM2 globally
npm install -g pm2

# Start Directus with PM2
pm2 start ecosystem.config.cjs

# Save PM2 configuration
pm2 save

# Setup PM2 startup script
pm2 startup
```

#### PM2 Configuration
```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [
    {
      name: 'directus',
      script: './cli.js',
      args: 'start',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 8055,
        PUBLIC_URL: 'https://directus.example.com',
      },
    },
  ],
};
```

### 2. Docker Deployment

#### Dockerfile
```dockerfile
FROM node:22-alpine AS builder

WORKDIR /directus

# Copy package files
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY packages ./packages
COPY api ./api
COPY app ./app
COPY sdk ./sdk
COPY directus ./directus

# Install dependencies and build
RUN corepack enable && corepack prepare pnpm@latest --activate
RUN pnpm install --frozen-lockfile
RUN pnpm build

# Production stage
FROM node:22-alpine

WORKDIR /directus

# Copy built application
COPY --from=builder /directus/dist ./

# Install production dependencies only
RUN corepack enable && corepack prepare pnpm@latest --activate
RUN pnpm install --prod --frozen-lockfile

# Expose port
EXPOSE 8055

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
  CMD node -e "require('http').get('http://localhost:8055/server/ping', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "cli.js", "start"]
```

#### Docker Compose
```yaml
# docker-compose.yml
version: '3.8'

services:
  directus:
    image: directus/directus:latest
    ports:
      - 8055:8055
    volumes:
      - ./uploads:/directus/uploads
      - ./extensions:/directus/extensions
    environment:
      SECRET: 'replace-with-secure-random-value'
      ADMIN_EMAIL: 'admin@example.com'
      ADMIN_PASSWORD: 'secure-password'
      DB_CLIENT: 'pg'
      DB_HOST: 'database'
      DB_PORT: '5432'
      DB_DATABASE: 'directus'
      DB_USER: 'directus'
      DB_PASSWORD: 'directus'
      WEBSOCKETS_ENABLED: 'true'
      CACHE_ENABLED: 'true'
      REDIS: 'redis://cache:6379'
    depends_on:
      - database
      - cache

  database:
    image: postgres:16-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: 'directus'
      POSTGRES_USER: 'directus'
      POSTGRES_PASSWORD: 'directus'

  cache:
    image: redis:7-alpine
    volumes:
      - cache_data:/data

volumes:
  db_data:
  cache_data:
```

### 3. Cloud Platforms

#### AWS Elastic Beanstalk
```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "directus/directus:latest",
    "Update": "true"
  },
  "Ports": [
    {
      "ContainerPort": 8055,
      "HostPort": 8055
    }
  ],
  "Environment": [
    {
      "Name": "DB_CLIENT",
      "Value": "pg"
    },
    {
      "Name": "DB_HOST",
      "Value": "your-rds-endpoint.amazonaws.com"
    }
  ]
}
```

#### Google Cloud Run
```bash
# Build and push image
gcloud builds submit --tag gcr.io/PROJECT_ID/directus

# Deploy to Cloud Run
gcloud run deploy directus \
  --image gcr.io/PROJECT_ID/directus \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars="DB_CLIENT=pg,DB_HOST=..."
```

#### Azure Container Instances
```bash
# Create container instance
az container create \
  --resource-group directus-rg \
  --name directus \
  --image directus/directus:latest \
  --dns-name-label directus \
  --ports 8055 \
  --environment-variables \
    DB_CLIENT=pg \
    DB_HOST=... \
  --secure-environment-variables \
    SECRET=... \
    DB_PASSWORD=...
```

## Environment Configuration

### Required Variables
```bash
# Security
SECRET="replace-with-random-32-char-string"
ACCESS_TOKEN_TTL="15m"
REFRESH_TOKEN_TTL="7d"

# Database
DB_CLIENT="pg"
DB_HOST="localhost"
DB_PORT="5432"
DB_DATABASE="directus"
DB_USER="directus"
DB_PASSWORD="directus"

# Server
PUBLIC_URL="https://directus.example.com"
PORT="8055"
HOST="0.0.0.0"
```

### Optional Variables
```bash
# Admin Account (first run only)
ADMIN_EMAIL="admin@example.com"
ADMIN_PASSWORD="secure-password"

# Cache
CACHE_ENABLED="true"
CACHE_STORE="redis"
REDIS="redis://localhost:6379"

# Storage
STORAGE_LOCATIONS="local,s3"
STORAGE_LOCAL_ROOT="./uploads"
STORAGE_S3_DRIVER="s3"
STORAGE_S3_KEY="your-access-key"
STORAGE_S3_SECRET="your-secret-key"
STORAGE_S3_BUCKET="directus-files"
STORAGE_S3_REGION="us-east-1"

# Email
EMAIL_FROM="no-reply@example.com"
EMAIL_TRANSPORT="smtp"
EMAIL_SMTP_HOST="smtp.example.com"
EMAIL_SMTP_PORT="587"
EMAIL_SMTP_USER="user"
EMAIL_SMTP_PASSWORD="password"

# Rate Limiting
RATE_LIMITER_ENABLED="true"
RATE_LIMITER_POINTS="50"
RATE_LIMITER_DURATION="1"

# WebSockets
WEBSOCKETS_ENABLED="true"

# Metrics
METRICS_ENABLED="true"

# Telemetry
TELEMETRY="false"
```

### Environment File
```bash
# .env
SECRET="your-secret-key-here"
DB_CLIENT="pg"
DB_HOST="localhost"
DB_PORT="5432"
DB_DATABASE="directus"
DB_USER="directus"
DB_PASSWORD="directus"
PUBLIC_URL="http://localhost:8055"
```

## Docker Deployment

### Multi-Stage Build
```dockerfile
# Build stage
FROM node:22-alpine AS builder
WORKDIR /directus
COPY . .
RUN corepack enable pnpm
RUN pnpm install --frozen-lockfile
RUN pnpm build

# Production stage
FROM node:22-alpine
WORKDIR /directus
COPY --from=builder /directus/dist ./
RUN corepack enable pnpm
RUN pnpm install --prod
EXPOSE 8055
CMD ["node", "cli.js", "start"]
```

### Docker Compose with Services
```yaml
version: '3.8'

services:
  # Directus Application
  directus:
    build: .
    ports:
      - "8055:8055"
    volumes:
      - uploads:/directus/uploads
      - extensions:/directus/extensions
    env_file:
      - .env
    depends_on:
      database:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  # PostgreSQL Database
  database:
    image: postgis/postgis:16-3.4-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: directus
      POSTGRES_USER: directus
      POSTGRES_PASSWORD: directus
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U directus"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis Cache
  cache:
    image: redis:7-alpine
    volumes:
      - cache_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - directus
    restart: unless-stopped

volumes:
  db_data:
  cache_data:
  uploads:
  extensions:
```

### Nginx Configuration
```nginx
# nginx.conf
upstream directus {
    server directus:8055;
}

server {
    listen 80;
    server_name directus.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name directus.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    client_max_body_size 100M;

    location / {
        proxy_pass http://directus;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # WebSocket support
    location /websocket {
        proxy_pass http://directus;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

## Production Optimization

### Performance Tuning

#### Database Connection Pool
```bash
DB_POOL_MIN="2"
DB_POOL_MAX="10"
DB_POOL_IDLE_TIMEOUT="30000"
DB_POOL_ACQUIRE_TIMEOUT="30000"
```

#### Cache Configuration
```bash
CACHE_ENABLED="true"
CACHE_TTL="30m"
CACHE_NAMESPACE="directus"
CACHE_AUTO_PURGE="true"
```

#### Rate Limiting
```bash
RATE_LIMITER_ENABLED="true"
RATE_LIMITER_POINTS="50"
RATE_LIMITER_DURATION="1"
RATE_LIMITER_STORE="redis"
```

### Security Hardening

#### Content Security Policy
```bash
CONTENT_SECURITY_POLICY_DIRECTIVES__SCRIPT_SRC="'self' 'unsafe-eval'"
CONTENT_SECURITY_POLICY_DIRECTIVES__CONNECT_SRC="'self' https:"
CONTENT_SECURITY_POLICY_DIRECTIVES__IMG_SRC="'self' data: https:"
```

#### CORS Configuration
```bash
CORS_ENABLED="true"
CORS_ORIGIN="https://example.com,https://app.example.com"
CORS_METHODS="GET,POST,PATCH,DELETE"
CORS_ALLOWED_HEADERS="Content-Type,Authorization"
CORS_EXPOSED_HEADERS="Content-Range"
CORS_CREDENTIALS="true"
CORS_MAX_AGE="18000"
```

#### IP Trust Proxy
```bash
IP_TRUST_PROXY="true"
IP_CUSTOM_HEADER="X-Forwarded-For"
```

### Asset Optimization

#### Image Processing
```bash
ASSETS_CACHE_TTL="30d"
ASSETS_TRANSFORM_MAX_CONCURRENT="4"
ASSETS_TRANSFORM_TIMEOUT="7500"
```

#### File Upload
```bash
MAX_PAYLOAD_SIZE="100mb"
TUS_ENABLED="true"
TUS_CHUNK_SIZE="5mb"
```

## Monitoring & Logging

### Logging Configuration
```bash
LOG_LEVEL="info"
LOG_STYLE="pretty"
```

### Prometheus Metrics
```bash
METRICS_ENABLED="true"
```

**Available Metrics**:
- HTTP request duration
- Database query duration
- Cache hit/miss ratio
- Active connections
- Memory usage
- CPU usage

### Health Checks
```bash
# Liveness probe
curl http://localhost:8055/server/ping

# Readiness probe
curl http://localhost:8055/server/health
```

### Logging Best Practices
```javascript
// Structured logging
logger.info({
  msg: 'User logged in',
  userId: user.id,
  ip: req.ip,
  userAgent: req.headers['user-agent'],
});

// Error logging
logger.error({
  msg: 'Database query failed',
  error: err.message,
  stack: err.stack,
  query: sql,
});
```

## Next Steps

- **Backend Architecture**: See [Backend Architecture](./04-backend-architecture.md) for API details
- **Database Layer**: Check [Database Layer](./05-database-layer.md) for database configuration
- **Performance**: Review [Performance & Scaling](./19-performance-scaling.md) for optimization
- **Security**: Explore [Authentication & Security](./06-auth-security.md) for security best practices
