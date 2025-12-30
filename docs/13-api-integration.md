# API Integration

This document covers how the Directus frontend communicates with the backend API, including HTTP client configuration, SDK integration, request patterns, and real-time communication.

## Table of Contents
- [HTTP Client Architecture](#http-client-architecture)
- [SDK Integration](#sdk-integration)
- [Request Queue Management](#request-queue-management)
- [Authentication Flow](#authentication-flow)
- [API Communication Patterns](#api-communication-patterns)
- [Error Handling](#error-handling)
- [Real-time Communication](#real-time-communication)
- [Performance Optimizations](#performance-optimizations)

## HTTP Client Architecture

### Axios Configuration
```typescript
// api.ts - Main HTTP client
import axios, { AxiosError, AxiosResponse, InternalAxiosRequestConfig } from 'axios';
import PQueue from 'p-queue';
import { useRequestsStore } from '@/stores/requests';
import { getRootPath } from '@/utils/get-root-path';

const api = axios.create({
  baseURL: getRootPath(),
  withCredentials: true,
});

// Request queue for rate limiting and performance
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,        // Max 5 concurrent requests
  intervalCap: 5,        // Max 5 requests per interval
  interval: 500,         // 500ms interval
  carryoverConcurrencyCount: true,
});
```

### Request/Response Interceptors
```typescript
// Request interceptor with queue management
export const onRequest = (config: InternalAxiosRequestConfig): Promise<InternalAxiosRequestConfig> => {
  const requestsStore = useRequestsStore();
  const id = requestsStore.startRequest();
  
  const requestConfig: InternalRequestConfig = {
    ...config,
    id,
  };
  
  return new Promise((resolve) => {
    requestQueue.add(async () => {
      // Wait for token refresh if needed
      await sdk.getToken().catch(() => {
        /* fail gracefully */
      });
      
      if (requestConfig.measureLatency) {
        requestConfig.start = performance.now();
      }
      
      return resolve(requestConfig);
    });
  });
};

// Response interceptor
export const onResponse = (response: AxiosResponse | Response): AxiosResponse | Response => {
  onRequestEnd(response);
  return response;
};

// Error interceptor
export const onError = async (error: RequestError): Promise<RequestError> => {
  onRequestEnd(error.response);
  return Promise.reject(error);
};

// Register interceptors
api.interceptors.request.use(onRequest);
api.interceptors.response.use(onResponse, onError);
```

### Request Tracking
```typescript
function onRequestEnd(response?: AxiosResponse | Response) {
  const config = response?.config as InternalRequestConfig | undefined;
  
  if (config?.id) {
    const requestsStore = useRequestsStore();
    requestsStore.endRequest(config.id);
    
    // Track latency if enabled
    if (config.measureLatency && config.start) {
      const latency = performance.now() - config.start;
      console.log(`Request latency: ${latency}ms`);
    }
  }
}
```

## SDK Integration

### Directus SDK Configuration
```typescript
// sdk.ts - Directus SDK client
import { getPublicURL } from '@/utils/get-root-path';
import type { AuthenticationClient, DirectusClient, RestClient } from '@directus/sdk';
import { authentication, createDirectus, rest } from '@directus/sdk';
import { ofetch, type FetchContext } from 'ofetch';

export type SdkClient = DirectusClient<any> & AuthenticationClient<any> & RestClient<any>;

// Custom fetch client with request tracking
const baseClient = ofetch.create({
  retry: 0,
  ignoreResponseError: true,
  async onRequest({ request, options }) {
    const requestsStore = useRequestsStore();
    const id = requestsStore.startRequest();
    (options as OptionsWithId).id = id;
    
    const path = getUrlPath(request);
    
    return new Promise((resolve) => {
      // Special handling for auth refresh
      if (path && path === '/auth/refresh') {
        requestQueue.pause();
        return resolve();
      }
      
      requestQueue.add(() => resolve());
    });
  },
  async onResponse({ options }) {
    const requestsStore = useRequestsStore();
    const id = (options as OptionsWithId).id;
    if (id) requestsStore.endRequest(id);
  },
  async onResponseError({ options }) {
    const requestsStore = useRequestsStore();
    const id = (options as OptionsWithId).id;
    if (id) requestsStore.endRequest(id);
  },
});

// Create SDK client
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
// Type-safe API calls with SDK
import { readItems, createItem, updateItem, deleteItem } from '@directus/sdk';

// Read items with type safety
const articles = await sdk.request(
  readItems('articles', {
    fields: ['id', 'title', 'content', 'author.name'],
    filter: { status: { _eq: 'published' } },
    sort: ['-date_created'],
    limit: 10,
  })
);

// Create item
const newArticle = await sdk.request(
  createItem('articles', {
    title: 'New Article',
    content: 'Article content...',
    status: 'draft',
  })
);

// Update item
const updatedArticle = await sdk.request(
  updateItem('articles', articleId, {
    status: 'published',
    date_published: new Date().toISOString(),
  })
);

// Delete item
await sdk.request(deleteItem('articles', articleId));
```

## Request Queue Management

### Queue Configuration
```typescript
// Dynamic queue management
export function resumeQueue() {
  if (!requestQueue.isPaused) return;
  requestQueue.start();
}

export async function replaceQueue(options?: Options<any, QueueAddOptions>) {
  await requestQueue.onIdle();
  requestQueue = new PQueue(options);
}

// Pause queue during authentication
export function pauseQueueForAuth() {
  requestQueue.pause();
}

// Resume queue after authentication
export function resumeQueueAfterAuth() {
  requestQueue.start();
}
```

### Priority Queue Implementation
```typescript
// Priority-based request handling
interface PriorityRequest {
  request: () => Promise<any>;
  priority: 'high' | 'normal' | 'low';
}

class PriorityRequestQueue {
  private highPriorityQueue = new PQueue({ concurrency: 2 });
  private normalPriorityQueue = new PQueue({ concurrency: 3 });
  private lowPriorityQueue = new PQueue({ concurrency: 1 });
  
  async add(request: PriorityRequest) {
    switch (request.priority) {
      case 'high':
        return this.highPriorityQueue.add(request.request);
      case 'normal':
        return this.normalPriorityQueue.add(request.request);
      case 'low':
        return this.lowPriorityQueue.add(request.request);
    }
  }
  
  pause() {
    this.highPriorityQueue.pause();
    this.normalPriorityQueue.pause();
    this.lowPriorityQueue.pause();
  }
  
  start() {
    this.highPriorityQueue.start();
    this.normalPriorityQueue.start();
    this.lowPriorityQueue.start();
  }
}
```

## Authentication Flow

### Login Process
```typescript
// auth.ts - Authentication management
import { sdk } from '@/sdk';
import { resumeQueue } from '@/api';
import { hydrate, dehydrate } from '@/hydrate';

export async function login({ credentials, provider, share }: LoginParams): Promise<void> {
  const appStore = useAppStore();
  const serverStore = useServerStore();
  
  let response: AuthenticationData;
  
  if (share) {
    // Share authentication
    const { share, password } = credentials;
    if (!share) throw new Error('Missing share ID');
    
    await sdk.request(authenticateShare(share, password, 'session'));
    response = await sdk.refresh();
  } else {
    // Regular authentication
    const { email, identifier, password, otp } = credentials;
    if (!password) throw new Error('Missing password');
    
    const loginOptions: LoginOptions = { 
      otp, 
      ...(provider !== DEFAULT_AUTH_PROVIDER && { provider }) 
    };
    
    if (email) {
      response = await sdk.login({ email, password }, loginOptions);
    } else if (identifier) {
      // Custom identifier login
      const login = <Schema extends object>(): RestCommand<AuthenticationData, Schema> => () => {
        const path = getAuthEndpoint(loginOptions.provider);
        const data = { identifier, password, otp, mode: 'session' };
        return { path, method: 'POST', body: JSON.stringify(data) };
      };
      
      await sdk.request(login());
      response = await sdk.refresh();
    } else {
      throw new Error('Missing email or identifier');
    }
  }
  
  // Update app state
  appStore.accessTokenExpiry = Date.now() + (response.expires ?? 0);
  appStore.authenticated = true;
  
  // Reload server store and hydrate all stores
  serverStore.hydrate();
  await hydrate();
}
```

### Token Refresh Management
```typescript
let idle = false;
let firstRefresh = true;

// Pause refresh when app is idle
emitter.on(Events.tabIdle, () => {
  sdk.stopRefreshing();
  idle = true;
});

// Resume refresh when app becomes active
emitter.on(Events.tabActive, () => {
  if (idle === true) {
    refresh();
    idle = false;
  }
});

export async function refresh({ navigate }: LogoutOptions = { navigate: true }): Promise<void> {
  const appStore = useAppStore();
  
  // Skip if not logged in (except for initial page load)
  if (!firstRefresh && !appStore.authenticated) return;
  
  try {
    // Skip refresh if token is still fresh
    if (appStore.accessTokenExpiry && 
        Date.now() < appStore.accessTokenExpiry - SDK_AUTH_REFRESH_BEFORE_EXPIRES) {
      await sdk.request(readMe({ fields: ['id'] }));
      return;
    }
    
    const response = await sdk.refresh();
    
    appStore.accessTokenExpiry = Date.now() + (response.expires ?? 0);
    appStore.authenticated = true;
    firstRefresh = false;
  } catch {
    await logout({ navigate, reason: LogoutReason.SESSION_EXPIRED });
  } finally {
    resumeQueue();
  }
}
```

### Logout Process
```typescript
export async function logout(options: LogoutOptions = {}): Promise<void> {
  const appStore = useAppStore();
  
  const defaultOptions: Required<LogoutOptions> = {
    navigate: true,
    reason: LogoutReason.SIGN_OUT,
  };
  
  const logoutOptions = { ...defaultOptions, ...options };
  
  // Stop token refresh
  sdk.stopRefreshing();
  
  // Call logout endpoint if user manually signed out
  if (logoutOptions.reason === LogoutReason.SIGN_OUT) {
    try {
      await sdk.logout();
    } catch {
      // User already signed out
    }
  }
  
  // Update app state
  appStore.authenticated = false;
  
  // Dehydrate all stores
  await dehydrate();
  
  // Navigate to login if needed
  if (logoutOptions.navigate === true) {
    const location: RouteLocationRaw = {
      path: `/login`,
      query: { reason: logoutOptions.reason },
    };
    
    router.push(location);
  }
}
```

## API Communication Patterns

### Store-Based API Integration
```typescript
// Example: Collections Store API Integration
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  
  // GET - Fetch collections
  async function hydrate() {
    loading.value = true;
    error.value = null;
    
    try {
      const response = await api.get<any>(`/collections`);
      const rawCollections: CollectionRaw[] = response.data.data;
      collections.value = rawCollections.map(prepareCollectionForApp);
    } catch (err) {
      error.value = err;
      console.error('Failed to fetch collections:', err);
    } finally {
      loading.value = false;
    }
  }
  
  // POST - Create collection
  async function createCollection(data: DeepPartial<Collection & { fields: Field[] }>) {
    try {
      const rawValues = omit(data, ['name', 'type', 'icon', 'color']);
      const response = await api.post<{ data: CollectionRaw }>('/collections', rawValues);
      
      const newCollection = prepareCollectionForApp(response.data.data);
      collections.value = [...collections.value, newCollection];
      
      notify({
        title: i18n.global.t('collection_created_successfully'),
        type: 'success',
      });
      
      return newCollection;
    } catch (error) {
      unexpectedError(error);
      throw error;
    }
  }
  
  // PATCH - Update collection
  async function updateCollection(collection: string, updates: DeepPartial<Collection>) {
    const stateClone = [...collections.value];
    
    // Optimistic update
    collections.value = collections.value.map((existingCollection: Collection) => {
      if (existingCollection.collection === collection) {
        return { ...existingCollection, ...updates };
      }
      return existingCollection;
    });
    
    try {
      const response = await api.patch<{ data: CollectionRaw }>(
        `/collections/${collection}`,
        updates
      );
      
      // Update with server response
      collections.value = collections.value.map((existingCollection: Collection) => {
        if (existingCollection.collection === collection) {
          return prepareCollectionForApp(response.data.data);
        }
        return existingCollection;
      });
      
      notify({
        title: i18n.global.t('update_collection_success'),
        type: 'success',
      });
    } catch (error) {
      // Revert optimistic update on error
      collections.value = stateClone;
      unexpectedError(error);
      throw error;
    }
  }
  
  // DELETE - Delete collection
  async function deleteCollection(collection: string) {
    const relationsStore = useRelationsStore();
    const stateClone = [...collections.value];
    const relationsStateClone = [...relationsStore.relations];
    
    // Optimistic update
    collections.value = collections.value.filter(c => c.collection !== collection);
    
    try {
      await api.delete(`/collections/${collection}`);
      
      // Refresh related stores
      await Promise.all([hydrate(), relationsStore.hydrate()]);
      
      notify({
        title: i18n.global.t('delete_collection_success'),
        type: 'success',
      });
    } catch (error) {
      // Revert optimistic updates
      collections.value = stateClone;
      relationsStore.relations = relationsStateClone;
      unexpectedError(error);
      throw error;
    }
  }
  
  return {
    collections,
    loading,
    error,
    hydrate,
    createCollection,
    updateCollection,
    deleteCollection,
  };
});
```

### Component API Integration
```typescript
// Component-level API usage
export default defineComponent({
  setup() {
    const loading = ref(false);
    const items = ref<Item[]>([]);
    const error = ref<Error | null>(null);
    
    // Fetch data on mount
    onMounted(async () => {
      await fetchItems();
    });
    
    async function fetchItems() {
      loading.value = true;
      error.value = null;
      
      try {
        const response = await api.get('/items', {
          params: {
            fields: ['id', 'name', 'status'],
            filter: { status: { _eq: 'active' } },
            sort: ['-date_created'],
          },
        });
        
        items.value = response.data.data;
      } catch (err) {
        error.value = err;
        notify({
          title: 'Failed to load items',
          text: err.message,
          type: 'error',
        });
      } finally {
        loading.value = false;
      }
    }
    
    async function createItem(data: CreateItemData) {
      try {
        const response = await api.post('/items', data);
        
        // Add to local state
        items.value.unshift(response.data.data);
        
        notify({
          title: 'Item created successfully',
          type: 'success',
        });
        
        return response.data.data;
      } catch (error) {
        unexpectedError(error);
        throw error;
      }
    }
    
    return {
      loading,
      items,
      error,
      fetchItems,
      createItem,
    };
  },
});
```

### Batch Operations
```typescript
// Batch API operations
export async function batchUpdateItems(
  collection: string, 
  updates: Array<{ id: string; data: any }>
) {
  const batchSize = 50; // Process in chunks
  const results = [];
  
  for (let i = 0; i < updates.length; i += batchSize) {
    const batch = updates.slice(i, i + batchSize);
    
    try {
      const response = await api.patch(`/items/${collection}`, {
        keys: batch.map(u => u.id),
        data: batch.reduce((acc, u) => ({ ...acc, [u.id]: u.data }), {}),
      });
      
      results.push(...response.data.data);
    } catch (error) {
      console.error(`Batch ${i / batchSize + 1} failed:`, error);
      throw error;
    }
  }
  
  return results;
}

// Batch delete with confirmation
export async function batchDeleteItems(
  collection: string, 
  ids: string[]
) {
  const confirmed = await confirm({
    title: 'Delete Items',
    text: `Are you sure you want to delete ${ids.length} items?`,
  });
  
  if (!confirmed) return;
  
  try {
    await api.delete(`/items/${collection}`, {
      data: ids,
    });
    
    notify({
      title: `${ids.length} items deleted successfully`,
      type: 'success',
    });
  } catch (error) {
    unexpectedError(error);
    throw error;
  }
}
```

## Error Handling

### Global Error Handler
```typescript
// Global error handling
export function unexpectedError(error: any): void {
  console.error('Unexpected error:', error);
  
  let message = 'An unexpected error occurred';
  
  if (error.response?.data?.errors) {
    const errors = error.response.data.errors;
    message = errors.map((e: any) => e.message).join(', ');
  } else if (error.message) {
    message = error.message;
  }
  
  notify({
    title: 'Error',
    text: message,
    type: 'error',
    persist: true,
  });
}

// Validation error handling
export function handleValidationErrors(errors: ValidationError[]): void {
  const fieldErrors = errors.reduce((acc, error) => {
    acc[error.field] = error.message;
    return acc;
  }, {} as Record<string, string>);
  
  // Show field-specific errors
  Object.entries(fieldErrors).forEach(([field, message]) => {
    notify({
      title: `Field "${field}"`,
      text: message,
      type: 'error',
    });
  });
}
```

### Retry Logic
```typescript
// Automatic retry for failed requests
export async function retryRequest<T>(
  requestFn: () => Promise<T>,
  maxRetries: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error) {
      lastError = error;
      
      // Don't retry on client errors (4xx)
      if (error.response?.status >= 400 && error.response?.status < 500) {
        throw error;
      }
      
      if (attempt < maxRetries) {
        console.warn(`Request failed (attempt ${attempt}/${maxRetries}), retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay *= 2; // Exponential backoff
      }
    }
  }
  
  throw lastError;
}

// Usage with retry
async function fetchWithRetry() {
  return retryRequest(
    () => api.get('/items'),
    3, // Max 3 retries
    1000 // Initial delay 1s
  );
}
```

### Network Status Handling
```typescript
// Network status monitoring
export const useNetworkStatus = () => {
  const online = ref(navigator.onLine);
  const retryQueue = ref<Array<() => Promise<any>>>([]);
  
  function handleOnline() {
    online.value = true;
    
    // Process retry queue
    const queue = [...retryQueue.value];
    retryQueue.value = [];
    
    queue.forEach(async (request) => {
      try {
        await request();
      } catch (error) {
        console.error('Retry failed:', error);
      }
    });
  }
  
  function handleOffline() {
    online.value = false;
    
    notify({
      title: 'Connection Lost',
      text: 'You are currently offline. Changes will be synced when connection is restored.',
      type: 'warning',
      persist: true,
    });
  }
  
  onMounted(() => {
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
  });
  
  onUnmounted(() => {
    window.removeEventListener('online', handleOnline);
    window.removeEventListener('offline', handleOffline);
  });
  
  function queueForRetry(request: () => Promise<any>) {
    retryQueue.value.push(request);
  }
  
  return {
    online,
    queueForRetry,
  };
};
```

## Real-time Communication

### WebSocket Integration
```typescript
// Real-time updates via WebSocket
export const useRealtime = () => {
  const connections = ref<Map<string, any>>(new Map());
  
  function subscribe(
    collection: string, 
    callback: (data: any) => void,
    options?: SubscriptionOptions
  ) {
    const subscription = sdk.subscribe('items', {
      collection,
      query: options?.query || { fields: ['*'] },
    }, (data) => {
      // Handle different event types
      switch (data.event) {
        case 'create':
          handleItemCreated(collection, data.data);
          break;
        case 'update':
          handleItemUpdated(collection, data.data);
          break;
        case 'delete':
          handleItemDeleted(collection, data.keys);
          break;
      }
      
      callback(data);
    });
    
    connections.value.set(`${collection}:items`, subscription);
    
    return () => {
      subscription.unsubscribe();
      connections.value.delete(`${collection}:items`);
    };
  }
  
  function handleItemCreated(collection: string, item: any) {
    // Update relevant stores
    const collectionsStore = useCollectionsStore();
    const collectionInfo = collectionsStore.getCollection(collection);
    
    if (collectionInfo) {
      // Emit event for components to handle
      emitter.emit('item-created', { collection, item });
    }
  }
  
  function handleItemUpdated(collection: string, item: any) {
    emitter.emit('item-updated', { collection, item });
  }
  
  function handleItemDeleted(collection: string, keys: string[]) {
    emitter.emit('item-deleted', { collection, keys });
  }
  
  return {
    subscribe,
    connections,
  };
};
```

### Real-time Store Updates
```typescript
// Store with real-time updates
export const useRealtimeItemsStore = defineStore('realtimeItemsStore', () => {
  const items = ref<Item[]>([]);
  const { subscribe } = useRealtime();
  
  let unsubscribe: (() => void) | null = null;
  
  function startRealtime(collection: string) {
    unsubscribe = subscribe(collection, (data) => {
      switch (data.event) {
        case 'create':
          items.value.unshift(data.data);
          break;
        case 'update':
          const updateIndex = items.value.findIndex(item => item.id === data.data.id);
          if (updateIndex > -1) {
            items.value[updateIndex] = { ...items.value[updateIndex], ...data.data };
          }
          break;
        case 'delete':
          items.value = items.value.filter(item => !data.keys.includes(item.id));
          break;
      }
    });
  }
  
  function stopRealtime() {
    if (unsubscribe) {
      unsubscribe();
      unsubscribe = null;
    }
  }
  
  return {
    items,
    startRealtime,
    stopRealtime,
  };
});
```

## Performance Optimizations

### Request Deduplication
```typescript
// Deduplicate identical requests
const requestCache = new Map<string, Promise<any>>();

export async function cachedRequest<T>(
  key: string,
  requestFn: () => Promise<T>,
  ttl: number = 5000
): Promise<T> {
  if (requestCache.has(key)) {
    return requestCache.get(key);
  }
  
  const promise = requestFn();
  requestCache.set(key, promise);
  
  // Clear cache after TTL
  setTimeout(() => {
    requestCache.delete(key);
  }, ttl);
  
  try {
    const result = await promise;
    return result;
  } catch (error) {
    // Remove failed request from cache immediately
    requestCache.delete(key);
    throw error;
  }
}

// Usage
async function fetchUser(id: string) {
  return cachedRequest(
    `user:${id}`,
    () => api.get(`/users/${id}`),
    10000 // 10 second cache
  );
}
```

### Request Cancellation
```typescript
// Cancel requests when component unmounts
export const useCancelableRequest = () => {
  const controllers = ref<AbortController[]>([]);
  
  function makeRequest<T>(requestFn: (signal: AbortSignal) => Promise<T>): Promise<T> {
    const controller = new AbortController();
    controllers.value.push(controller);
    
    return requestFn(controller.signal).finally(() => {
      const index = controllers.value.indexOf(controller);
      if (index > -1) {
        controllers.value.splice(index, 1);
      }
    });
  }
  
  function cancelAll() {
    controllers.value.forEach(controller => controller.abort());
    controllers.value = [];
  }
  
  onUnmounted(() => {
    cancelAll();
  });
  
  return {
    makeRequest,
    cancelAll,
  };
};

// Usage in component
export default defineComponent({
  setup() {
    const { makeRequest } = useCancelableRequest();
    
    async function fetchData() {
      try {
        const response = await makeRequest((signal) => 
          api.get('/items', { signal })
        );
        
        // Handle response
      } catch (error) {
        if (error.name === 'AbortError') {
          console.log('Request was cancelled');
        } else {
          console.error('Request failed:', error);
        }
      }
    }
    
    return { fetchData };
  },
});
```

### Optimistic Updates
```typescript
// Optimistic update pattern
export async function optimisticUpdate<T>(
  updateFn: () => Promise<T>,
  optimisticState: () => void,
  revertState: () => void
): Promise<T> {
  // Apply optimistic update immediately
  optimisticState();
  
  try {
    // Perform actual API call
    const result = await updateFn();
    return result;
  } catch (error) {
    // Revert on error
    revertState();
    throw error;
  }
}

// Usage in store
async function updateItem(id: string, updates: Partial<Item>) {
  const originalItem = items.value.find(item => item.id === id);
  if (!originalItem) throw new Error('Item not found');
  
  return optimisticUpdate(
    () => api.patch(`/items/${id}`, updates),
    () => {
      // Optimistic update
      const index = items.value.findIndex(item => item.id === id);
      if (index > -1) {
        items.value[index] = { ...items.value[index], ...updates };
      }
    },
    () => {
      // Revert on error
      const index = items.value.findIndex(item => item.id === id);
      if (index > -1) {
        items.value[index] = originalItem;
      }
    }
  );
}
```

## Next Steps

- **State Management**: See [State Management](./12-state-management.md) for store integration patterns
- **Real-time Features**: Check [Real-time Features](./16-realtime-features.md) for WebSocket details
- **Performance**: Review [Performance & Scaling](./19-performance-scaling.md) for optimization strategies
- **Error Handling**: Explore error handling patterns and user feedback systems
