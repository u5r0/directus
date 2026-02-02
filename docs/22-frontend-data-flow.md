# Frontend Data Flow & Architecture

This document explains how data flows through the Directus frontend application, from API requests to UI updates.

## Table of Contents
- [Data Flow Overview](#data-flow-overview)
- [HTTP Client Architecture](#http-client-architecture)
- [SDK Client Architecture](#sdk-client-architecture)
- [Request Queue System](#request-queue-system)
- [Store Integration](#store-integration)
- [Real-time Updates](#real-time-updates)
- [Optimistic Updates](#optimistic-updates)
- [Error Handling](#error-handling)

## Data Flow Overview

### Complete Data Flow Diagram

```
User Action
    ↓
Component Event Handler
    ↓
Store Action / Composable
    ↓
API Client (Axios) or SDK Client
    ↓
Request Queue (p-queue)
    ↓
HTTP Request to Backend
    ↓
Backend Response
    ↓
Response Interceptor
    ↓
Store State Update
    ↓
Vue Reactivity System
    ↓
Component Re-render
    ↓
UI Update
```

### Example: Fetching Collections

```typescript
// 1. Component triggers fetch
<script setup>
const collectionsStore = useCollectionsStore();

onMounted(async () => {
  await collectionsStore.hydrate();
});
</script>

// 2. Store action makes API call
// app/src/stores/collections.ts
async function hydrate() {
  const response = await api.get('/collections');
  const rawCollections = response.data.data;
  collections.value = rawCollections.map(prepareCollectionForApp);
}

// 3. API client queues request
// app/src/api.ts
api.interceptors.request.use((config) => {
  return new Promise((resolve) => {
    requestQueue.add(() => resolve(config));
  });
});

// 4. Response updates store
// collections.value is reactive
collections.value = [...newData];

// 5. Component automatically re-renders
// Vue's reactivity system detects the change
```

## HTTP Client Architecture

### Axios Configuration

```typescript
// app/src/api.ts
import axios from 'axios';
import PQueue from 'p-queue';

const api = axios.create({
  baseURL: getRootPath(),
  withCredentials: true,
});

export let requestQueue: PQueue = new PQueue({
  concurrency: 5,           // Max 5 concurrent requests
  intervalCap: 5,           // Max 5 requests per 500ms
  interval: 500,            // Rate limit window
  carryoverConcurrencyCount: true,
});
```

### Request Interceptor

```typescript
export const onRequest = (config: InternalAxiosRequestConfig): Promise<InternalAxiosRequestConfig> => {
  const requestsStore = useRequestsStore();
  const id = requestsStore.startRequest();

  const requestConfig: InternalRequestConfig = {
    ...config,
    id,
  };

  return new Promise((resolve) => {
    requestQueue.add(async () => {
      // Wait for any active token refresh
      await sdk.getToken().catch(() => {
        /* fail gracefully */
      });

      // Track latency if requested
      if (requestConfig.measureLatency) {
        requestConfig.start = performance.now();
      }

      return resolve(requestConfig);
    });
  });
};

api.interceptors.request.use(onRequest);
```

### Response Interceptor

```typescript
export const onResponse = (response: AxiosResponse): AxiosResponse => {
  const config = response.config as InternalRequestConfig;

  if (config?.id) {
    const requestsStore = useRequestsStore();
    requestsStore.endRequest(config.id);
  }

  // Calculate latency if measured
  if (config?.measureLatency && config?.start) {
    const latency = performance.now() - config.start;
    console.log(`Request latency: ${latency}ms`);
  }

  return response;
};

export const onError = async (error: RequestError): Promise<RequestError> => {
  const config = error.response?.config as InternalRequestConfig;

  if (config?.id) {
    const requestsStore = useRequestsStore();
    requestsStore.endRequest(config.id);
  }

  // Handle authentication errors
  if (error.response?.status === 401) {
    const userStore = useUserStore();
    await userStore.dehydrate();
    router.push('/login');
  }

  return Promise.reject(error);
};

api.interceptors.response.use(onResponse, onError);
```

## SDK Client Architecture

### SDK Setup

```typescript
// app/src/sdk.ts
import { createDirectus, authentication, rest } from '@directus/sdk';
import { ofetch } from 'ofetch';

const baseClient = ofetch.create({
  retry: 0,
  ignoreResponseError: true,
  async onRequest({ request, options }) {
    const requestsStore = useRequestsStore();
    const id = requestsStore.startRequest();
    (options as OptionsWithId).id = id;
    
    const path = getUrlPath(request);

    return new Promise((resolve) => {
      // Pause queue for auth refresh
      if (path && path === '/auth/refresh') {
        requestQueue.pause();
        return resolve();
      }

      // Queue all other requests
      requestQueue.add(() => resolve());
    });
  },
  async onResponse({ options }) {
    const requestsStore = useRequestsStore();
    const id = (options as OptionsWithId).id;
    if (id) requestsStore.endRequest(id);
  },
});

export const sdk: SdkClient = createDirectus(getPublicURL(), {
  globals: { fetch: baseClient.native }
})
  .with(authentication('session', {
    credentials: 'include',
    msRefreshBeforeExpires: SDK_AUTH_REFRESH_BEFORE_EXPIRES
  }))
  .with(rest({ credentials: 'include' }));
```

### SDK Usage Patterns

```typescript
// Authentication
await sdk.login(email, password);
await sdk.logout();
await sdk.refresh();

// CRUD Operations
const items = await sdk.request(readItems('articles'));
const item = await sdk.request(readItem('articles', id));
await sdk.request(createItem('articles', data));
await sdk.request(updateItem('articles', id, data));
await sdk.request(deleteItem('articles', id));

// Custom Requests
const response = await sdk.request(() => ({
  method: 'GET',
  path: '/custom/endpoint',
}));
```

## Request Queue System

### Queue Configuration

```typescript
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,                    // Max concurrent requests
  intervalCap: 5,                    // Max requests per interval
  interval: 500,                     // Interval in ms
  carryoverConcurrencyCount: true,   // Carry over count between intervals
});
```

### Queue Management

```typescript
// Pause queue (e.g., during auth refresh)
requestQueue.pause();

// Resume queue
requestQueue.start();

// Replace queue with new configuration
export async function replaceQueue(options?: Options) {
  await requestQueue.onIdle();
  requestQueue = new PQueue(options);
}

// Wait for all requests to complete
await requestQueue.onIdle();
```

### Request Tracking

```typescript
// app/src/stores/requests.ts
export const useRequestsStore = defineStore('requestsStore', {
  state: () => ({
    queue: [] as string[],
  }),
  getters: {
    queueHasItems: (state) => state.queue.length > 0,
  },
  actions: {
    startRequest() {
      const id = nanoid();
      this.queue.push(id);
      return id;
    },
    endRequest(id: string) {
      this.queue = this.queue.filter((queueID) => queueID !== id);
    },
  },
});
```

## Store Integration

### Store Hydration Pattern

```typescript
// app/src/stores/collections.ts
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);

  async function hydrate() {
    const response = await api.get<any>('/collections');
    const rawCollections: CollectionRaw[] = response.data.data;
    collections.value = rawCollections.map(prepareCollectionForApp);
  }

  async function dehydrate() {
    collections.value = [];
  }

  return {
    collections,
    hydrate,
    dehydrate,
  };
});
```

### Store Update Pattern

```typescript
// app/src/stores/fields.ts
async function updateField(collection: string, field: string, updates: DeepPartial<Field>) {
  const stateClone = [...fields.value];

  // 1. Optimistic update
  fields.value = fields.value.map((f) => {
    if (f.collection === collection && f.field === field) {
      return merge({}, f, updates);
    }
    return f;
  });

  try {
    // 2. API call
    const response = await api.patch(`/fields/${collection}/${field}`, updates);

    // 3. Update with server response
    fields.value = fields.value.map((f) => {
      if (f.collection === collection && f.field === field) {
        return parseField(response.data.data);
      }
      return f;
    });
  } catch (error) {
    // 4. Rollback on error
    fields.value = stateClone;
    throw error;
  }
}
```

### Cross-Store Dependencies

```typescript
// app/src/stores/fields.ts
async function deleteField(collection: string, field: string) {
  const relationsStore = useRelationsStore();
  const collectionsStore = useCollectionsStore();

  const stateClone = [...fields.value];
  const relationsStateClone = [...relationsStore.relations];

  // Update both stores optimistically
  fields.value = fields.value.filter((f) => {
    return !(f.field === field && f.collection === collection);
  });

  relationsStore.relations = relationsStore.relations.filter((r) => {
    return !(
      (r.collection === collection && r.field === field) ||
      (r.related_collection === collection && r.meta?.one_field === field)
    );
  });

  try {
    await api.delete(`/fields/${collection}/${field}`);
    
    // Refresh related stores
    await Promise.all([
      hydrate(),
      relationsStore.hydrate(),
      collectionsStore.hydrate(),
    ]);
  } catch (error) {
    // Rollback both stores
    fields.value = stateClone;
    relationsStore.relations = relationsStateClone;
    throw error;
  }
}
```

## Real-time Updates

### WebSocket Connection

```typescript
// app/src/sdk.ts
import { realtime } from '@directus/sdk';

export const sdk = createDirectus(getPublicURL())
  .with(authentication('session'))
  .with(rest())
  .with(realtime());

// Subscribe to collection updates
const { subscription } = await sdk.subscribe('items', {
  query: {
    collection: 'articles',
    filter: { status: { _eq: 'published' } },
  },
});

for await (const message of subscription) {
  if (message.type === 'subscription' && message.event === 'create') {
    // Handle new item
    const newItem = message.data[0];
    items.value.push(newItem);
  }
}
```

### Store Integration with WebSocket

```typescript
// app/src/stores/items.ts
export const useItemsStore = defineStore('itemsStore', () => {
  const items = ref<Item[]>([]);

  async function subscribeToUpdates(collection: string) {
    const { subscription } = await sdk.subscribe('items', {
      query: { collection },
    });

    for await (const message of subscription) {
      switch (message.event) {
        case 'create':
          items.value.push(...message.data);
          break;
        case 'update':
          items.value = items.value.map((item) => {
            const updated = message.data.find((d) => d.id === item.id);
            return updated ? { ...item, ...updated } : item;
          });
          break;
        case 'delete':
          const deletedIds = message.data.map((d) => d.id);
          items.value = items.value.filter((item) => !deletedIds.includes(item.id));
          break;
      }
    }
  }

  return {
    items,
    subscribeToUpdates,
  };
});
```

## Optimistic Updates

### Pattern Implementation

```typescript
async function updateItem(id: string, updates: Partial<Item>) {
  // 1. Save current state
  const stateClone = [...items.value];
  
  // 2. Apply optimistic update
  items.value = items.value.map((item) => {
    if (item.id === id) {
      return { ...item, ...updates };
    }
    return item;
  });

  try {
    // 3. Send to server
    const response = await api.patch(`/items/${collection}/${id}`, updates);
    
    // 4. Update with server response
    items.value = items.value.map((item) => {
      if (item.id === id) {
        return response.data.data;
      }
      return item;
    });
  } catch (error) {
    // 5. Rollback on error
    items.value = stateClone;
    
    // 6. Show error notification
    notify({
      title: i18n.global.t('update_failed'),
      type: 'error',
    });
    
    throw error;
  }
}
```

### Benefits

1. **Instant Feedback**: UI updates immediately
2. **Better UX**: No waiting for server response
3. **Error Recovery**: Automatic rollback on failure
4. **Consistency**: Server response is source of truth

## Error Handling

### API Error Handling

```typescript
// app/src/api.ts
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const status = error.response?.status;

    switch (status) {
      case 401:
        // Unauthorized - logout user
        const userStore = useUserStore();
        await userStore.dehydrate();
        router.push('/login');
        break;

      case 403:
        // Forbidden - show permission error
        notify({
          title: i18n.global.t('not_allowed'),
          text: i18n.global.t('not_allowed_description'),
          type: 'error',
        });
        break;

      case 404:
        // Not found - show error
        notify({
          title: i18n.global.t('not_found'),
          type: 'error',
        });
        break;

      case 500:
        // Server error - show generic error
        notify({
          title: i18n.global.t('server_error'),
          text: error.response?.data?.errors?.[0]?.message,
          type: 'error',
        });
        break;

      default:
        // Unknown error
        unexpectedError(error);
    }

    return Promise.reject(error);
  }
);
```

### Store Error Handling

```typescript
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);

  async function hydrate() {
    loading.value = true;
    error.value = null;

    try {
      const response = await api.get('/collections');
      collections.value = response.data.data.map(prepareCollectionForApp);
    } catch (err) {
      error.value = err as Error;
      unexpectedError(err);
    } finally {
      loading.value = false;
    }
  }

  return {
    collections,
    loading,
    error,
    hydrate,
  };
});
```

### Component Error Handling

```vue
<script setup lang="ts">
const collectionsStore = useCollectionsStore();
const { collections, loading, error } = storeToRefs(collectionsStore);

onMounted(async () => {
  try {
    await collectionsStore.hydrate();
  } catch (error) {
    // Error already handled by store
    // Component can show error state
  }
});
</script>

<template>
  <div>
    <VProgressCircular v-if="loading" />
    <VError v-else-if="error" :error="error" />
    <VList v-else :items="collections" />
  </div>
</template>
```

## Performance Optimizations

### 1. Request Batching

```typescript
// Batch multiple field updates into single request
async function updateFields(collection: string, updates: DeepPartial<Field>[]) {
  const response = await api.patch(`/fields/${collection}`, updates);
  return response.data.data;
}
```

### 2. Request Deduplication

```typescript
// Cache in-flight requests
const requestCache = new Map<string, Promise<any>>();

async function fetchCollection(id: string) {
  const cacheKey = `collection:${id}`;
  
  if (requestCache.has(cacheKey)) {
    return requestCache.get(cacheKey);
  }

  const promise = api.get(`/collections/${id}`);
  requestCache.set(cacheKey, promise);

  try {
    const result = await promise;
    return result;
  } finally {
    requestCache.delete(cacheKey);
  }
}
```

### 3. Lazy Loading

```typescript
// Load data only when needed
const items = ref<Item[]>([]);
const loaded = ref(false);

async function ensureLoaded() {
  if (loaded.value) return;
  
  await hydrate();
  loaded.value = true;
}

// Component calls ensureLoaded before accessing items
```

## Next Steps

- [State Management](./12-state-management.md) - Pinia stores in detail
- [API Integration](./13-api-integration.md) - HTTP client and SDK
- [Component System](./08-ui-components.md) - UI components
- [Real-time Features](./16-realtime-features.md) - WebSocket integration

## Related Files

- `app/src/api.ts` - Axios HTTP client
- `app/src/sdk.ts` - Directus SDK client
- `app/src/stores/requests.ts` - Request tracking store
- `app/src/stores/collections.ts` - Collections store example
- `app/src/stores/fields.ts` - Fields store example
