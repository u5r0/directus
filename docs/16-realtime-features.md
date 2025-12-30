# Real-time Features

This document covers the WebSocket-based real-time features in Directus, enabling live updates and subscriptions.

## Overview

Directus provides real-time capabilities through WebSocket connections, allowing clients to subscribe to data changes and receive instant updates when items are created, updated, or deleted.

## WebSocket Architecture

### WebSocket Server Setup

The WebSocket server runs alongside the HTTP server:

```typescript
// api/src/server.ts
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({
  noServer: true,
  maxPayload: env['WEBSOCKETS_MAX_PAYLOAD'],
});

// Handle upgrade requests
server.on('upgrade', (request, socket, head) => {
  wss.handleUpgrade(request, socket, head, (ws) => {
    wss.emit('connection', ws, request);
  });
});
```

### WebSocket Configuration

```bash
# Enable WebSocket support
WEBSOCKETS_ENABLED=true

# WebSocket settings
WEBSOCKETS_MAX_PAYLOAD=104857600  # 100MB
WEBSOCKETS_HEARTBEAT_ENABLED=true
WEBSOCKETS_HEARTBEAT_PERIOD=30000  # 30 seconds

# Authentication
WEBSOCKETS_AUTH_MODE=handshake  # or public, strict
WEBSOCKETS_AUTH_TIMEOUT=10000   # 10 seconds
```

## Message Protocol

### Message Structure

All WebSocket messages follow a structured format:

```typescript
interface WebSocketMessage {
  type: string;
  uid?: string | number;  // Unique message ID
  [key: string]: any;
}
```

### Message Types

#### 1. Authentication Messages

**auth (Client → Server)**
```json
{
  "type": "auth",
  "access_token": "your-jwt-token",
  "refresh_token": "your-refresh-token"
}
```

**auth (Server → Client)**
```json
{
  "type": "auth",
  "status": "ok",
  "refresh_token": "new-refresh-token",
  "expires": 900000
}
```

#### 2. Subscription Messages

**subscribe (Client → Server)**
```json
{
  "type": "subscribe",
  "collection": "articles",
  "query": {
    "fields": ["id", "title", "status"],
    "filter": {
      "status": {
        "_eq": "published"
      }
    }
  },
  "uid": "sub-1"
}
```

**subscription (Server → Client)**
```json
{
  "type": "subscription",
  "event": "create",
  "data": [{
    "id": "123",
    "title": "New Article",
    "status": "published"
  }],
  "uid": "sub-1"
}
```

**unsubscribe (Client → Server)**
```json
{
  "type": "unsubscribe",
  "uid": "sub-1"
}
```

#### 3. Item Operation Messages

**items (Client → Server)**
```json
{
  "type": "items",
  "collection": "articles",
  "action": "create",
  "data": {
    "title": "New Article",
    "content": "Article content..."
  },
  "uid": "req-1"
}
```

**items (Server → Client)**
```json
{
  "type": "items",
  "status": "ok",
  "data": {
    "id": "123",
    "title": "New Article",
    "content": "Article content..."
  },
  "uid": "req-1"
}
```

#### 4. Heartbeat Messages

**ping (Client → Server)**
```json
{
  "type": "ping"
}
```

**pong (Server → Client)**
```json
{
  "type": "pong"
}
```

## Subscription System

### Subscribe Handler (`api/src/websocket/handlers/subscribe.ts`)

```typescript
export class SubscribeHandler {
  // Storage of subscriptions per collection
  subscriptions: Record<string, Set<Subscription>>;
  
  constructor() {
    this.subscriptions = {};
    this.bindWebSocket();
    
    // Listen to Redis pub/sub for distributed subscriptions
    this.messenger.subscribe('websocket.event', (message) => {
      this.dispatch(message as WebSocketEvent);
    });
  }
  
  /**
   * Register a subscription
   */
  subscribe(subscription: Subscription) {
    const { collection } = subscription;
    
    if (!this.subscriptions[collection]) {
      this.subscriptions[collection] = new Set();
    }
    
    this.subscriptions[collection].add(subscription);
  }
  
  /**
   * Remove a subscription
   */
  unsubscribe(client: WebSocketClient, uid?: string) {
    for (const collection in this.subscriptions) {
      const subscriptions = this.subscriptions[collection];
      
      for (const subscription of subscriptions) {
        if (subscription.client === client) {
          if (!uid || subscription.uid === uid) {
            subscriptions.delete(subscription);
          }
        }
      }
    }
  }
  
  /**
   * Dispatch event to matching subscriptions
   */
  dispatch(event: WebSocketEvent) {
    const { collection, action, payload, keys } = event;
    const subscriptions = this.subscriptions[collection];
    
    if (!subscriptions) return;
    
    for (const subscription of subscriptions) {
      // Check if subscription matches event
      if (this.matchesSubscription(subscription, event)) {
        this.sendUpdate(subscription, event);
      }
    }
  }
}
```

### Subscription Types

```typescript
interface Subscription {
  uid?: string | number;
  query?: Query;
  item?: string | number;
  event?: SubscriptionEvent;
  collection: string;
  client: WebSocketClient;
}

type SubscriptionEvent = 'create' | 'update' | 'delete';
```

### Subscription Patterns

**1. Collection Subscription**
Subscribe to all changes in a collection:

```typescript
{
  type: 'subscribe',
  collection: 'articles',
  uid: 'articles-all'
}
```

**2. Filtered Subscription**
Subscribe to items matching a filter:

```typescript
{
  type: 'subscribe',
  collection: 'articles',
  query: {
    filter: {
      status: { _eq: 'published' },
      author: { _eq: '$CURRENT_USER' }
    }
  },
  uid: 'my-articles'
}
```

**3. Item Subscription**
Subscribe to a specific item:

```typescript
{
  type: 'subscribe',
  collection: 'articles',
  item: '123',
  uid: 'article-123'
}
```

**4. Event-Specific Subscription**
Subscribe to specific events:

```typescript
{
  type: 'subscribe',
  collection: 'articles',
  event: 'create',
  uid: 'new-articles'
}
```

## Authentication

### WebSocket Authentication Modes

#### 1. Handshake Mode (Default)
Authenticate during WebSocket handshake:

```typescript
// Client sends token in query parameter
const ws = new WebSocket('ws://localhost:8055/websocket?access_token=your-token');
```

#### 2. Public Mode
Allow unauthenticated connections:

```bash
WEBSOCKETS_AUTH_MODE=public
```

#### 3. Strict Mode
Require authentication message after connection:

```bash
WEBSOCKETS_AUTH_MODE=strict
WEBSOCKETS_AUTH_TIMEOUT=10000
```

```typescript
// Client must send auth message within timeout
ws.send(JSON.stringify({
  type: 'auth',
  access_token: 'your-token'
}));
```

### Authentication Flow

```typescript
// api/src/websocket/authenticate.ts
export async function authenticateConnection(
  client: WebSocketClient,
  accountability: Accountability
): Promise<void> {
  // Verify token
  const tokenPayload = verifyAccessToken(accountability.token);
  
  // Load user and permissions
  const user = await getUserById(tokenPayload.id);
  
  // Set client accountability
  client.accountability = {
    user: user.id,
    role: user.role,
    roles: await getRolesForUser(user.id),
    admin: user.admin,
    app: user.app_access,
    ip: accountability.ip,
  };
  
  // Set token expiration timer
  const expiresIn = tokenPayload.exp * 1000 - Date.now();
  client.auth_timer = setTimeout(() => {
    client.close(1008, 'Token expired');
  }, expiresIn);
}
```

## Real-time Updates

### Event Emission

When items change, events are emitted:

```typescript
// api/src/services/items.ts
export class ItemsService {
  async createOne(data: Partial<Item>): Promise<PrimaryKey> {
    const key = await this.knex(this.collection).insert(data);
    
    // Emit create event
    emitter.emitAction('items.create', {
      payload: data,
      key: key[0],
      collection: this.collection,
    }, {
      database: this.knex,
      schema: this.schema,
      accountability: this.accountability,
    });
    
    return key[0];
  }
  
  async updateOne(key: PrimaryKey, data: Partial<Item>): Promise<PrimaryKey> {
    await this.knex(this.collection).where({ id: key }).update(data);
    
    // Emit update event
    emitter.emitAction('items.update', {
      payload: data,
      keys: [key],
      collection: this.collection,
    }, {
      database: this.knex,
      schema: this.schema,
      accountability: this.accountability,
    });
    
    return key;
  }
}
```

### Event Broadcasting

Events are broadcast to WebSocket clients:

```typescript
// Listen to item events
emitter.onAction('items.create', async (meta, context) => {
  const { collection, payload, key } = meta;
  
  // Broadcast to WebSocket clients
  await messenger.publish('websocket.event', {
    type: 'subscription',
    event: 'create',
    collection,
    data: [payload],
    keys: [key],
  });
});
```

### Permission Filtering

Subscriptions respect user permissions:

```typescript
async function sendUpdate(
  subscription: Subscription,
  event: WebSocketEvent
): Promise<void> {
  const { client, collection, query } = subscription;
  const { accountability } = client;
  
  // Check permissions
  const service = new ItemsService(collection, {
    accountability,
    schema: await getSchema(),
  });
  
  // Fetch items with permissions applied
  const items = await service.readMany(event.keys, query);
  
  // Send only items user has access to
  if (items.length > 0) {
    client.send(JSON.stringify({
      type: 'subscription',
      event: event.action,
      data: items,
      uid: subscription.uid,
    }));
  }
}
```

## Client Integration

### JavaScript/TypeScript SDK

```typescript
import { createDirectus, realtime } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(realtime());

// Subscribe to collection
const { subscription, unsubscribe } = await client.subscribe('articles', {
  query: {
    fields: ['id', 'title', 'status'],
    filter: {
      status: { _eq: 'published' }
    }
  }
});

// Listen for updates
for await (const message of subscription) {
  if (message.event === 'create') {
    console.log('New article:', message.data);
  } else if (message.event === 'update') {
    console.log('Updated article:', message.data);
  } else if (message.event === 'delete') {
    console.log('Deleted article:', message.data);
  }
}

// Unsubscribe when done
unsubscribe();
```

### Vue Integration

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { sdk } from '@/sdk';

const articles = ref([]);
let unsubscribe: (() => void) | null = null;

onMounted(async () => {
  // Initial load
  const response = await sdk.request(readItems('articles'));
  articles.value = response;
  
  // Subscribe to updates
  const { subscription, unsubscribe: unsub } = await sdk.subscribe('articles');
  unsubscribe = unsub;
  
  for await (const message of subscription) {
    if (message.event === 'create') {
      articles.value.push(...message.data);
    } else if (message.event === 'update') {
      message.data.forEach(updated => {
        const index = articles.value.findIndex(a => a.id === updated.id);
        if (index !== -1) {
          articles.value[index] = updated;
        }
      });
    } else if (message.event === 'delete') {
      const keys = message.data.map(d => d.id);
      articles.value = articles.value.filter(a => !keys.includes(a.id));
    }
  }
});

onUnmounted(() => {
  unsubscribe?.();
});
</script>
```

### React Integration

```typescript
import { useEffect, useState } from 'react';
import { createDirectus, realtime } from '@directus/sdk';

function useRealtimeCollection(collection: string, query?: Query) {
  const [items, setItems] = useState([]);
  
  useEffect(() => {
    const client = createDirectus('https://your-directus.com')
      .with(realtime());
    
    let unsubscribe: (() => void) | null = null;
    
    (async () => {
      const { subscription, unsubscribe: unsub } = await client.subscribe(
        collection,
        { query }
      );
      
      unsubscribe = unsub;
      
      for await (const message of subscription) {
        if (message.event === 'create') {
          setItems(prev => [...prev, ...message.data]);
        } else if (message.event === 'update') {
          setItems(prev => prev.map(item => {
            const updated = message.data.find(d => d.id === item.id);
            return updated || item;
          }));
        } else if (message.event === 'delete') {
          const keys = message.data.map(d => d.id);
          setItems(prev => prev.filter(item => !keys.includes(item.id)));
        }
      }
    })();
    
    return () => {
      unsubscribe?.();
    };
  }, [collection, query]);
  
  return items;
}
```

## Advanced Features

### GraphQL Subscriptions

WebSocket support for GraphQL subscriptions:

```graphql
subscription {
  articles_mutated {
    event
    key
    data {
      id
      title
      status
    }
  }
}
```

### Log Streaming

Stream server logs via WebSocket:

```typescript
// Subscribe to logs
client.send(JSON.stringify({
  type: 'subscribe',
  collection: 'directus_activity',
  uid: 'logs'
}));

// Receive log entries
{
  type: 'subscription',
  event: 'create',
  data: [{
    action: 'create',
    collection: 'articles',
    user: 'user-id',
    timestamp: '2024-01-15T10:30:00Z'
  }],
  uid: 'logs'
}
```

### Heartbeat/Keep-Alive

Automatic connection health monitoring:

```typescript
// Server sends ping every 30 seconds
setInterval(() => {
  clients.forEach(client => {
    if (client.isAlive === false) {
      client.terminate();
    } else {
      client.isAlive = false;
      client.ping();
    }
  });
}, 30000);

// Client responds with pong
client.on('pong', () => {
  client.isAlive = true;
});
```

## Scaling WebSockets

### Redis Pub/Sub

For multi-instance deployments, use Redis for message distribution:

```bash
# Enable Redis for WebSocket scaling
REDIS=redis://localhost:6379
WEBSOCKETS_ENABLED=true
```

```typescript
// Publish events to all instances
await messenger.publish('websocket.event', {
  type: 'subscription',
  event: 'create',
  collection: 'articles',
  data: [item],
});

// Subscribe to events from other instances
messenger.subscribe('websocket.event', (event) => {
  // Broadcast to local WebSocket clients
  broadcastToClients(event);
});
```

### Load Balancing

Configure sticky sessions for WebSocket connections:

```nginx
upstream directus {
  ip_hash;  # Sticky sessions
  server directus1:8055;
  server directus2:8055;
  server directus3:8055;
}

server {
  location /websocket {
    proxy_pass http://directus;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
  }
}
```

## Error Handling

### WebSocket Errors

```typescript
interface WebSocketError {
  type: 'error';
  code: string;
  message: string;
  uid?: string | number;
}
```

**Common Error Codes:**
- `INVALID_PAYLOAD` - Malformed message
- `INVALID_COLLECTION` - Collection doesn't exist
- `FORBIDDEN` - Permission denied
- `UNAUTHORIZED` - Authentication required
- `RATE_LIMIT_EXCEEDED` - Too many requests

### Client Error Handling

```typescript
ws.addEventListener('error', (error) => {
  console.error('WebSocket error:', error);
});

ws.addEventListener('close', (event) => {
  if (event.code === 1008) {
    console.log('Token expired, reconnecting...');
    reconnect();
  }
});

ws.addEventListener('message', (event) => {
  const message = JSON.parse(event.data);
  
  if (message.type === 'error') {
    console.error('Server error:', message.message);
  }
});
```

## Best Practices

### Connection Management

- Implement automatic reconnection with exponential backoff
- Handle token expiration and refresh
- Clean up subscriptions on component unmount
- Use heartbeat to detect dead connections

### Performance

- Limit number of subscriptions per client
- Use specific filters to reduce message volume
- Implement client-side caching
- Batch updates when possible

### Security

- Always authenticate WebSocket connections
- Validate all incoming messages
- Apply permission checks on subscriptions
- Rate limit WebSocket messages
- Use WSS (WebSocket Secure) in production

## Next Steps

- [Authentication & Security](./06-auth-security.md) - WebSocket authentication
- [API Integration](./13-api-integration.md) - REST API integration
- [SDK Architecture](./14-sdk-architecture.md) - SDK real-time features
