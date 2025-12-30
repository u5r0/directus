# Performance & Scaling

This document covers performance optimization and scaling strategies for Directus deployments.

## Overview

Directus is designed to scale from small single-server deployments to large distributed systems. This guide covers caching, database optimization, load balancing, and monitoring strategies.

## Caching

### Cache Configuration

```bash
# Enable caching
CACHE_ENABLED=true
CACHE_STORE=redis  # or memory
CACHE_NAMESPACE=directus-cache
CACHE_TTL=30m
CACHE_AUTO_PURGE=true

# System cache (for schema, permissions)
CACHE_SYSTEM_TTL=10m

# Schema caching
CACHE_SCHEMA=true
CACHE_SCHEMA_FREEZE_ENABLED=true
CACHE_SCHEMA_MAX_ITERATIONS=10
CACHE_SCHEMA_SYNC_TIMEOUT=5000

# Redis configuration
REDIS=redis://localhost:6379
# or
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=password
REDIS_DB=0
```

### Cache Implementation (`api/src/cache.ts`)

```typescript
export function getCache(): {
  cache: Keyv | null;
  systemCache: Keyv;
  localSchemaCache: Keyv;
  lockCache: Keyv;
} {
  if (env['CACHE_ENABLED'] && cache === null) {
    cache = getKeyvInstance(
      env['CACHE_STORE'],
      getMilliseconds(env['CACHE_TTL'])
    );
  }
  
  if (systemCache === null) {
    systemCache = getKeyvInstance(
      env['CACHE_STORE'],
      getMilliseconds(env['CACHE_SYSTEM_TTL']),
      '_system'
    );
  }
  
  return { cache, systemCache, localSchemaCache, lockCache };
}
```

### Cache Types

**1. Response Cache**
Caches API responses:
```typescript
// Automatic caching for GET requests
GET /items/articles?cache=true

// Cache headers
Cache-Control: max-age=1800
ETag: "abc123"
```

**2. System Cache**
Caches schema, permissions, and settings:
```typescript
// Schema cache
const schema = await getSchema();  // Cached

// Permission cache
const permissions = await fetchPermissions();  // Cached
```

**3. Schema Cache**
In-memory schema caching:
```typescript
// Memory cache for schema
export function setMemorySchemaCache(schema: SchemaOverview) {
  if (env['CACHE_SCHEMA_FREEZE_ENABLED']) {
    memorySchemaCache = freezeSchema(schema);
  } else {
    memorySchemaCache = schema;
  }
}

export function getMemorySchemaCache(): SchemaOverview | undefined {
  return memorySchemaCache;
}
```

### Cache Invalidation

```typescript
// Clear all caches
await flushCaches();

// Clear system cache
await clearSystemCache();

// Clear specific cache key
await cache.delete('key');

// Auto-purge on schema changes
messenger.subscribe('schemaChanged', async () => {
  if (env['CACHE_AUTO_PURGE']) {
    await cache.clear();
  }
});
```

### Cache Strategies

**Cache-Aside (Lazy Loading):**
```typescript
async function getData(key: string) {
  // Check cache first
  let data = await cache.get(key);
  
  if (!data) {
    // Load from database
    data = await database.select().from('table').where({ id: key });
    
    // Store in cache
    await cache.set(key, data, ttl);
  }
  
  return data;
}
```

**Write-Through:**
```typescript
async function updateData(key: string, data: any) {
  // Update database
  await database('table').where({ id: key }).update(data);
  
  // Update cache
  await cache.set(key, data, ttl);
}
```

**Write-Behind:**
```typescript
async function updateData(key: string, data: any) {
  // Update cache immediately
  await cache.set(key, data, ttl);
  
  // Queue database update
  queue.add(() => database('table').where({ id: key }).update(data));
}
```

## Database Optimization

### Connection Pooling

```bash
# Pool configuration
DB_POOL_MIN=0
DB_POOL_MAX=10
DB_POOL_IDLE_TIMEOUT=30000
DB_POOL_ACQUIRE_TIMEOUT=30000
```

```typescript
const knexConfig: Knex.Config = {
  client: 'pg',
  connection: connectionConfig,
  pool: {
    min: env['DB_POOL_MIN'],
    max: env['DB_POOL_MAX'],
    idleTimeoutMillis: env['DB_POOL_IDLE_TIMEOUT'],
    acquireTimeoutMillis: env['DB_POOL_ACQUIRE_TIMEOUT'],
  },
};
```

### Query Optimization

**1. Field Selection**
Only request needed fields:
```typescript
// ✗ Bad - fetches all fields
const items = await service.readByQuery({});

// ✓ Good - fetches specific fields
const items = await service.readByQuery({
  fields: ['id', 'title', 'status'],
});
```

**2. Pagination**
Always paginate large datasets:
```typescript
// ✓ Good
const items = await service.readByQuery({
  limit: 100,
  page: 1,
});
```

**3. Indexing**
Create indexes on frequently queried columns:
```sql
-- PostgreSQL
CREATE INDEX idx_articles_status ON articles(status);
CREATE INDEX idx_articles_author ON articles(author);
CREATE INDEX idx_articles_created ON articles(created_at);

-- Composite index
CREATE INDEX idx_articles_status_created ON articles(status, created_at);
```

**4. Query Analysis**
```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM articles WHERE status = 'published';

-- MySQL
EXPLAIN SELECT * FROM articles WHERE status = 'published';
```

### Database-Specific Optimizations

**PostgreSQL:**
```sql
-- Vacuum regularly
VACUUM ANALYZE;

-- Update statistics
ANALYZE articles;

-- Connection pooling with PgBouncer
# pgbouncer.ini
[databases]
directus = host=localhost port=5432 dbname=directus

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

**MySQL:**
```sql
-- Optimize tables
OPTIMIZE TABLE articles;

-- Analyze tables
ANALYZE TABLE articles;

-- InnoDB buffer pool
SET GLOBAL innodb_buffer_pool_size = 2147483648;  -- 2GB
```

## Rate Limiting

### Configuration

```bash
# Global rate limiter
RATE_LIMITER_ENABLED=true
RATE_LIMITER_STORE=redis
RATE_LIMITER_POINTS=50
RATE_LIMITER_DURATION=1

# IP-based rate limiter
RATE_LIMITER_GLOBAL_ENABLED=true
RATE_LIMITER_GLOBAL_POINTS=1000
RATE_LIMITER_GLOBAL_DURATION=1
```

### Rate Limiter Implementation

```typescript
import { RateLimiterRedis, RateLimiterMemory } from 'rate-limiter-flexible';

export function createRateLimiter(
  configPrefix = 'RATE_LIMITER',
  configOverrides?: IRateLimiterOptionsOverrides
): RateLimiterAbstract {
  const env = useEnv();
  
  switch (env['RATE_LIMITER_STORE']) {
    case 'redis':
      return new RateLimiterRedis(getConfig('redis', configPrefix));
    case 'memory':
    default:
      return new RateLimiterMemory(getConfig('memory', configPrefix));
  }
}
```

## Load Shedding

### Pressure-Based Load Shedding

```bash
# Enable pressure monitoring
PRESSURE_ENABLED=true
PRESSURE_SAMPLE_INTERVAL=250
PRESSURE_MAX_EVENT_LOOP_DELAY=500
PRESSURE_MAX_EVENT_LOOP_UTILIZATION=0.98
PRESSURE_MAX_MEMORY_RSS=0.98
PRESSURE_MAX_MEMORY_HEAP_USED=0.98
```

```typescript
import { createPressureMonitor } from '@directus/pressure';

const monitor = createPressureMonitor({
  sampleInterval: 250,
  maxEventLoopDelay: 500,
  maxEventLoopUtilization: 0.98,
  maxMemoryRss: 0.98,
  maxMemoryHeapUsed: 0.98,
});

// Middleware
app.use((req, res, next) => {
  if (monitor.overloaded) {
    res.status(503).json({
      error: 'Service temporarily unavailable',
    });
  } else {
    next();
  }
});
```

## Horizontal Scaling

### Multi-Instance Deployment

**Requirements:**
- Shared Redis for caching and sessions
- Shared database
- Shared file storage (S3, Azure, GCS)
- Load balancer

**Configuration:**
```bash
# Instance 1
HOST=0.0.0.0
PORT=8055
REDIS=redis://redis-server:6379
DB_HOST=postgres-server
STORAGE_LOCATIONS=s3

# Instance 2
HOST=0.0.0.0
PORT=8056
REDIS=redis://redis-server:6379
DB_HOST=postgres-server
STORAGE_LOCATIONS=s3
```

### Load Balancer Configuration

**Nginx:**
```nginx
upstream directus {
  least_conn;
  server directus1:8055 max_fails=3 fail_timeout=30s;
  server directus2:8055 max_fails=3 fail_timeout=30s;
  server directus3:8055 max_fails=3 fail_timeout=30s;
}

server {
  listen 80;
  server_name directus.example.com;
  
  location / {
    proxy_pass http://directus;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
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

**HAProxy:**
```
frontend directus_frontend
  bind *:80
  default_backend directus_backend

backend directus_backend
  balance leastconn
  option httpchk GET /server/health
  server directus1 directus1:8055 check
  server directus2 directus2:8055 check
  server directus3 directus3:8055 check
```

### Session Management

Use Redis for shared sessions:

```bash
SESSION_STORE=redis
REDIS=redis://redis-server:6379
```

### File Storage

Use cloud storage for shared files:

```bash
# S3
STORAGE_LOCATIONS=s3
STORAGE_S3_DRIVER=s3
STORAGE_S3_KEY=your-key
STORAGE_S3_SECRET=your-secret
STORAGE_S3_BUCKET=directus-files
STORAGE_S3_REGION=us-east-1

# Azure
STORAGE_LOCATIONS=azure
STORAGE_AZURE_DRIVER=azure
STORAGE_AZURE_CONTAINER_NAME=directus-files
STORAGE_AZURE_ACCOUNT_NAME=your-account
STORAGE_AZURE_ACCOUNT_KEY=your-key

# GCS
STORAGE_LOCATIONS=gcs
STORAGE_GCS_DRIVER=gcs
STORAGE_GCS_BUCKET=directus-files
STORAGE_GCS_KEY_FILENAME=/path/to/keyfile.json
```

## Monitoring

### Metrics Collection

```bash
# Enable Prometheus metrics
METRICS_ENABLED=true
```

**Metrics Endpoint:**
```
GET /metrics
```

**Available Metrics:**
- `directus_http_requests_total` - Total HTTP requests
- `directus_http_request_duration_seconds` - Request duration
- `directus_database_response_duration_seconds` - Database query duration
- `directus_cache_hit_total` - Cache hits
- `directus_cache_miss_total` - Cache misses

### Metrics Implementation

```typescript
import { useMetrics } from './metrics/index.js';

const metrics = useMetrics();

// Record request
metrics?.getHttpRequestMetric()?.inc({
  method: req.method,
  status: res.statusCode,
});

// Record duration
const duration = performance.now() - startTime;
metrics?.getHttpRequestDurationMetric()?.observe(duration);

// Record database query
metrics?.getDatabaseResponseMetric()?.observe(queryDuration);
```

### Health Checks

**Health Endpoint:**
```
GET /server/health
```

**Response:**
```json
{
  "status": "ok",
  "releaseId": "11.0.0",
  "serviceId": "directus-1",
  "checks": {
    "database": "ok",
    "cache": "ok"
  }
}
```

### Logging

```bash
# Log configuration
LOG_LEVEL=info  # error, warn, info, debug, trace
LOG_STYLE=pretty  # or raw

# Structured logging
LOG_STYLE=raw
```

**Log Levels:**
- `error` - Errors only
- `warn` - Warnings and errors
- `info` - General information
- `debug` - Detailed debugging
- `trace` - Very detailed tracing

## Performance Best Practices

### API Optimization

**1. Use Field Selection**
```typescript
// Only request needed fields
GET /items/articles?fields=id,title,status
```

**2. Implement Pagination**
```typescript
// Paginate large datasets
GET /items/articles?limit=100&page=1
```

**3. Use Filtering**
```typescript
// Filter on indexed columns
GET /items/articles?filter[status][_eq]=published
```

**4. Enable Caching**
```typescript
// Cache GET requests
GET /items/articles?cache=true
```

**5. Batch Operations**
```typescript
// Create multiple items at once
POST /items/articles
[
  { title: 'Article 1' },
  { title: 'Article 2' }
]
```

### Database Optimization

**1. Create Indexes**
```sql
CREATE INDEX idx_status ON articles(status);
CREATE INDEX idx_created ON articles(created_at);
```

**2. Use Connection Pooling**
```bash
DB_POOL_MIN=2
DB_POOL_MAX=10
```

**3. Optimize Queries**
```typescript
// Use specific fields
fields: ['id', 'title']

// Use indexed filters
filter: { status: { _eq: 'published' } }

// Limit results
limit: 100
```

**4. Regular Maintenance**
```sql
-- PostgreSQL
VACUUM ANALYZE;

-- MySQL
OPTIMIZE TABLE articles;
```

### Frontend Optimization

**1. Lazy Loading**
```typescript
// Lazy load components
const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
);
```

**2. Virtual Scrolling**
```vue
<VirtualList
  :items="items"
  :item-height="48"
  :buffer="10"
/>
```

**3. Request Deduplication**
```typescript
// Deduplicate concurrent requests
const cache = new Map();

async function fetchData(key) {
  if (cache.has(key)) {
    return cache.get(key);
  }
  
  const promise = api.get(`/items/${key}`);
  cache.set(key, promise);
  
  try {
    return await promise;
  } finally {
    cache.delete(key);
  }
}
```

**4. Image Optimization**
```typescript
// Use asset transformations
GET /assets/image-id?width=400&height=300&quality=80&format=webp
```

## Deployment Strategies

### Docker Deployment

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  directus:
    image: directus/directus:latest
    ports:
      - "8055:8055"
    environment:
      KEY: your-secret-key
      SECRET: your-secret-key
      DB_CLIENT: postgres
      DB_HOST: postgres
      DB_PORT: 5432
      DB_DATABASE: directus
      DB_USER: directus
      DB_PASSWORD: directus
      CACHE_ENABLED: "true"
      CACHE_STORE: redis
      REDIS: redis://redis:6379
    depends_on:
      - postgres
      - redis
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
  
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: directus
      POSTGRES_USER: directus
      POSTGRES_PASSWORD: directus
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Kubernetes Deployment

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: directus
spec:
  replicas: 3
  selector:
    matchLabels:
      app: directus
  template:
    metadata:
      labels:
        app: directus
    spec:
      containers:
      - name: directus
        image: directus/directus:latest
        ports:
        - containerPort: 8055
        env:
        - name: DB_CLIENT
          value: "postgres"
        - name: DB_HOST
          value: "postgres-service"
        - name: REDIS
          value: "redis://redis-service:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /server/health
            port: 8055
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /server/health
            port: 8055
          initialDelaySeconds: 10
          periodSeconds: 5
```

## Troubleshooting

### Performance Issues

**Slow Queries:**
```sql
-- PostgreSQL: Find slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- MySQL: Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;
```

**High Memory Usage:**
```bash
# Check memory usage
NODE_OPTIONS="--max-old-space-size=4096"

# Monitor with metrics
GET /metrics | grep memory
```

**Connection Pool Exhaustion:**
```bash
# Increase pool size
DB_POOL_MAX=20

# Monitor connections
SELECT count(*) FROM pg_stat_activity;
```

## Next Steps

- [Database Layer](./05-database-layer.md) - Database optimization
- [Backend Architecture](./04-backend-architecture.md) - API structure
- [Real-time Features](./16-realtime-features.md) - WebSocket scaling
