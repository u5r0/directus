# Frontend-Backend-SDK Interactions

This document provides a comprehensive guide to how the frontend (App), backend (API), and SDK interact in Directus, with real code examples and detailed data flow patterns.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Communication Layers](#communication-layers)
- [SDK Integration in App](#sdk-integration-in-app)
- [Axios vs SDK Usage](#axios-vs-sdk-usage)
- [Authentication Flow](#authentication-flow)
- [Data Fetching Patterns](#data-fetching-patterns)
- [Real-time Communication](#real-time-communication)
- [Request Queue Management](#request-queue-management)
- [Error Handling](#error-handling)
- [Type Safety Across Stack](#type-safety-across-stack)
- [Complete Examples](#complete-examples)

## Architecture Overview

### Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    @directus/app (Frontend)                 │
│                     Vue 3 + TypeScript                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Pinia      │  │  Components  │  │  Composables │    │
│  │   Stores     │  │              │  │              │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
│                            │                                │
└────────────────────────────┼────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              Communication Layer (Dual Client)              │
│  ┌──────────────────────┐  ┌──────────────────────┐       │
│  │   @directus/sdk      │  │   Axios (api.ts)     │       │
│  │   - Type-safe        │  │   - Direct HTTP      │       │
│  │   - Composable       │  │   - Legacy support   │       │
│  │   - Auth built-in    │  │   - Custom config    │       │
│  └──────────┬───────────┘  └──────────┬───────────┘       │
│             │                          │                    │
└─────────────┼──────────────────────────┼────────────────────┘
              │                          │
              └──────────┬───────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  @directus/api (Backend)                    │
│                  Express.js + TypeScript                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Controllers  │  │   Services   │  │   Database   │    │
│  │ (Routes)     │  │  (Logic)     │  │   (Knex)     │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```



## Communication Layers

### Layer 1: SDK Client (`app/src/sdk.ts`)

The SDK is the primary, type-safe client for API communication:

```typescript
// app/src/sdk.ts
import type { AuthenticationClient, DirectusClient, RestClient } from '@directus/sdk';
import { authentication, createDirectus, rest } from '@directus/sdk';
import { ofetch } from 'ofetch';
import { requestQueue } from './api';
import { useRequestsStore } from './stores/requests';
import { getPublicURL } from '@/utils/get-root-path';

export type SdkClient = DirectusClient<unknown> & 
                        AuthenticationClient<unknown> & 
                        RestClient<unknown>;

// Custom fetch client with request tracking
const baseClient = ofetch.create({
  retry: 0,
  ignoreResponseError: true,
  
  async onRequest({ request, options }) {
    const requestsStore = useRequestsStore();
    const id = requestsStore.startRequest();
    (options as any).id = id;
    
    const path = getUrlPath(request);
    
    // Pause queue for auth refresh
    if (path === '/auth/refresh') {
      requestQueue.pause();
      return;
    }
    
    // Queue all other requests
    return new Promise((resolve) => {
      requestQueue.add(() => resolve());
    });
  },
  
  async onResponse({ options }) {
    const requestsStore = useRequestsStore();
    const id = (options as any).id;
    if (id) requestsStore.endRequest(id);
  },
});

// Create SDK instance
export const sdk: SdkClient = createDirectus(getPublicURL(), {
  globals: { fetch: baseClient.native }
})
  .with(authentication('session', {
    credentials: 'include',
    msRefreshBeforeExpires: 10000  // Refresh 10s before expiry
  }))
  .with(rest({ credentials: 'include' }));

export default sdk;
```

**Key Features:**
- Type-safe API calls
- Built-in authentication management
- Automatic token refresh
- Request tracking integration
- Queue management for rate limiting



### Layer 2: Axios Client (`app/src/api.ts`)

Axios is used alongside SDK for specific operations and legacy support:

```typescript
// app/src/api.ts
import axios, { AxiosError, AxiosResponse, InternalAxiosRequestConfig } from 'axios';
import PQueue from 'p-queue';
import sdk from './sdk';
import { useRequestsStore } from '@/stores/requests';
import { getRootPath } from '@/utils/get-root-path';

// Create axios instance
const api = axios.create({
  baseURL: getRootPath(),
  withCredentials: true,
});

// Request queue for rate limiting
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,        // Max 5 concurrent requests
  intervalCap: 5,        // Max 5 requests per interval
  interval: 500,         // 500ms interval
  carryoverConcurrencyCount: true,
});

// Request interceptor
export const onRequest = (config: InternalAxiosRequestConfig): Promise<InternalAxiosRequestConfig> => {
  const requestsStore = useRequestsStore();
  const id = requestsStore.startRequest();
  
  const requestConfig = { ...config, id };
  
  return new Promise((resolve) => {
    requestQueue.add(async () => {
      // Wait for SDK token refresh if in progress
      await sdk.getToken().catch(() => {});
      
      if (requestConfig.measureLatency) {
        requestConfig.start = performance.now();
      }
      
      return resolve(requestConfig);
    });
  });
};

// Response interceptor
export const onResponse = (response: AxiosResponse): AxiosResponse => {
  const config = response.config as any;
  if (config?.id) {
    const requestsStore = useRequestsStore();
    requestsStore.endRequest(config.id);
  }
  return response;
};

// Error interceptor
export const onError = async (error: AxiosError): Promise<AxiosError> => {
  const config = error.response?.config as any;
  if (config?.id) {
    const requestsStore = useRequestsStore();
    requestsStore.endRequest(config.id);
  }
  return Promise.reject(error);
};

api.interceptors.request.use(onRequest);
api.interceptors.response.use(onResponse, onError);

export default api;
```

**Key Features:**
- Direct HTTP access
- Request/response interceptors
- Latency measurement
- Coordinated with SDK for auth
- Shared request queue



## SDK Integration in App

### How the App Uses SDK

The app uses SDK as the primary client for type-safe API communication:

```typescript
// Example: Using SDK in a Pinia store
// app/src/stores/collections.ts
import { defineStore } from 'pinia';
import { readCollections, readCollection, updateCollection } from '@directus/sdk';
import sdk from '@/sdk';
import type { Collection } from '@directus/types';

export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  
  // Fetch all collections
  async function hydrate() {
    try {
      const response = await sdk.request(readCollections());
      collections.value = response;
    } catch (error) {
      console.error('Failed to load collections:', error);
    }
  }
  
  // Fetch single collection
  async function getCollection(collectionKey: string) {
    try {
      return await sdk.request(readCollection(collectionKey));
    } catch (error) {
      console.error(`Failed to load collection ${collectionKey}:`, error);
      throw error;
    }
  }
  
  // Update collection
  async function updateOne(collectionKey: string, updates: Partial<Collection>) {
    try {
      await sdk.request(updateCollection(collectionKey, updates));
      await hydrate(); // Refresh data
    } catch (error) {
      console.error(`Failed to update collection ${collectionKey}:`, error);
      throw error;
    }
  }
  
  return {
    collections,
    hydrate,
    getCollection,
    updateOne,
  };
});
```

### SDK Commands

The SDK provides type-safe commands for all API operations:

```typescript
import {
  // Items
  readItems,
  readItem,
  createItem,
  createItems,
  updateItem,
  updateItems,
  deleteItem,
  deleteItems,
  
  // Collections
  readCollections,
  readCollection,
  createCollection,
  updateCollection,
  deleteCollection,
  
  // Fields
  readFields,
  readField,
  createField,
  updateField,
  deleteField,
  
  // Users
  readUsers,
  readUser,
  readMe,
  updateUser,
  updateMe,
  
  // Files
  readFiles,
  readFile,
  uploadFiles,
  updateFile,
  deleteFile,
  
  // Authentication
  login,
  logout,
  refresh,
  
  // And 50+ more commands...
} from '@directus/sdk';
```



## Axios vs SDK Usage

### When to Use SDK

Use SDK for:
- Type-safe API calls
- Standard CRUD operations
- Authentication
- Real-time subscriptions
- GraphQL queries

```typescript
// ✅ Use SDK for standard operations
import sdk from '@/sdk';
import { readItems, createItem } from '@directus/sdk';

// Type-safe item fetching
const articles = await sdk.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name'],
    filter: { status: { _eq: 'published' } },
    sort: ['-date_created'],
    limit: 10,
  })
);

// Type-safe item creation
const newArticle = await sdk.request(
  createItem('articles', {
    title: 'New Article',
    content: 'Content here...',
    status: 'draft',
  })
);
```

### When to Use Axios

Use Axios for:
- Custom endpoints
- File uploads with progress
- Request cancellation
- Custom headers/config
- Legacy code compatibility

```typescript
// ✅ Use Axios for custom operations
import api from '@/api';

// Custom endpoint
const response = await api.get('/custom/endpoint', {
  params: { filter: 'value' }
});

// File upload with progress
const formData = new FormData();
formData.append('file', file);

await api.post('/files', formData, {
  headers: { 'Content-Type': 'multipart/form-data' },
  onUploadProgress: (progressEvent) => {
    const progress = (progressEvent.loaded / progressEvent.total) * 100;
    console.log(`Upload progress: ${progress}%`);
  },
});

// Request with cancellation
const controller = new AbortController();
const response = await api.get('/items/articles', {
  signal: controller.signal
});
// Later: controller.abort();
```

### Hybrid Approach

Many stores use both clients:

```typescript
// app/src/stores/files.ts
import { defineStore } from 'pinia';
import sdk from '@/sdk';
import api from '@/api';
import { readFiles, deleteFile } from '@directus/sdk';

export const useFilesStore = defineStore('filesStore', () => {
  // Use SDK for standard operations
  async function getFiles() {
    return await sdk.request(readFiles({
      fields: ['*'],
      limit: 100,
    }));
  }
  
  // Use Axios for file upload with progress
  async function uploadFile(file: File, onProgress?: (progress: number) => void) {
    const formData = new FormData();
    formData.append('file', file);
    
    const response = await api.post('/files', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: (event) => {
        if (onProgress && event.total) {
          const progress = (event.loaded / event.total) * 100;
          onProgress(progress);
        }
      },
    });
    
    return response.data.data;
  }
  
  // Use SDK for deletion
  async function deleteFileById(id: string) {
    await sdk.request(deleteFile(id));
  }
  
  return {
    getFiles,
    uploadFile,
    deleteFileById,
  };
});
```



## Authentication Flow

### Complete Authentication Sequence

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User enters credentials in login form                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. App calls SDK login method                              │
│    await sdk.login(email, password)                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SDK sends POST /auth/login                              │
│    { email, password }                                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. API validates credentials                                │
│    - Checks user exists                                     │
│    - Verifies password with Argon2                          │
│    - Checks user status (active/suspended)                  │
│    - Validates 2FA if enabled                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. API generates tokens                                     │
│    - Access token (JWT, 15min)                              │
│    - Refresh token (JWT, 7 days)                            │
│    - Session cookie                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. API returns tokens + user data                          │
│    {                                                        │
│      access_token: "eyJ...",                                │
│      refresh_token: "eyJ...",                               │
│      expires: 900000                                        │
│    }                                                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. SDK stores tokens                                        │
│    - Access token in memory                                 │
│    - Refresh token in cookie (httpOnly)                     │
│    - Sets up auto-refresh timer                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. App hydrates user store                                 │
│    - Fetches current user data                              │
│    - Loads permissions                                      │
│    - Initializes app state                                  │
└─────────────────────────────────────────────────────────────┘
```

### Login Implementation

```typescript
// app/src/routes/login/login.vue
import { useRouter } from 'vue-router';
import sdk from '@/sdk';
import { useUserStore } from '@/stores/user';

const router = useRouter();
const userStore = useUserStore();

const email = ref('');
const password = ref('');
const otp = ref('');
const loading = ref(false);
const error = ref<string | null>(null);

async function login() {
  loading.value = true;
  error.value = null;
  
  try {
    // Step 1: Login with SDK
    await sdk.login(email.value, password.value, {
      otp: otp.value || undefined,
    });
    
    // Step 2: Hydrate user store
    await userStore.hydrate();
    
    // Step 3: Navigate to app
    router.push('/content');
  } catch (err: any) {
    // Handle specific errors
    if (err.errors?.[0]?.extensions?.code === 'INVALID_OTP') {
      error.value = 'Invalid 2FA code';
    } else if (err.errors?.[0]?.extensions?.code === 'INVALID_CREDENTIALS') {
      error.value = 'Invalid email or password';
    } else {
      error.value = 'Login failed. Please try again.';
    }
  } finally {
    loading.value = false;
  }
}
```

### Token Refresh Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SDK detects token expiring soon (10s before)            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. SDK pauses request queue                                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SDK sends POST /auth/refresh                            │
│    { refresh_token: "..." } (from cookie)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. API validates refresh token                              │
│    - Verifies JWT signature                                 │
│    - Checks token not expired                               │
│    - Validates session exists                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. API generates new access token                          │
│    { access_token: "eyJ...", expires: 900000 }              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. SDK stores new access token                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. SDK resumes request queue                                │
│    All queued requests now use new token                    │
└─────────────────────────────────────────────────────────────┘
```

### Automatic Token Refresh

The SDK handles token refresh automatically:

```typescript
// SDK configuration in app/src/sdk.ts
export const sdk = createDirectus(getPublicURL())
  .with(authentication('session', {
    credentials: 'include',
    msRefreshBeforeExpires: 10000,  // Refresh 10s before expiry
  }))
  .with(rest({ credentials: 'include' }));

// The SDK automatically:
// 1. Monitors token expiration
// 2. Refreshes before expiry
// 3. Queues requests during refresh
// 4. Retries failed requests with new token
```



## Data Fetching Patterns

### Pattern 1: Store-Based Fetching

Most data fetching happens through Pinia stores:

```typescript
// app/src/stores/collections.ts
import { defineStore } from 'pinia';
import sdk from '@/sdk';
import { readCollections } from '@directus/sdk';
import type { Collection } from '@directus/types';

export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  
  async function hydrate() {
    if (loading.value) return;
    
    loading.value = true;
    error.value = null;
    
    try {
      // Fetch from API via SDK
      collections.value = await sdk.request(readCollections());
    } catch (err) {
      error.value = err as Error;
      console.error('Failed to load collections:', err);
    } finally {
      loading.value = false;
    }
  }
  
  // Computed getter for specific collection
  const getCollection = computed(() => {
    return (key: string) => collections.value.find(c => c.collection === key);
  });
  
  return {
    collections,
    loading,
    error,
    hydrate,
    getCollection,
  };
});
```

### Pattern 2: Component-Level Fetching

For component-specific data:

```typescript
// app/src/modules/content/routes/collection.vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import sdk from '@/sdk';
import { readItems } from '@directus/sdk';
import type { Item } from '@directus/types';

const props = defineProps<{
  collection: string;
}>();

const items = ref<Item[]>([]);
const loading = ref(false);
const page = ref(1);
const limit = ref(25);
const total = ref(0);

async function fetchItems() {
  loading.value = true;
  
  try {
    const response = await sdk.request(
      readItems(props.collection, {
        fields: ['*'],
        limit: limit.value,
        offset: (page.value - 1) * limit.value,
        sort: ['-date_created'],
        meta: ['total_count', 'filter_count'],
      })
    );
    
    items.value = response;
    total.value = response.meta?.total_count || 0;
  } catch (error) {
    console.error('Failed to fetch items:', error);
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  fetchItems();
});

// Refetch when page changes
watch(page, () => {
  fetchItems();
});
</script>
```

### Pattern 3: Composable-Based Fetching

Reusable data fetching logic:

```typescript
// app/src/composables/use-items.ts
import { ref, watch } from 'vue';
import sdk from '@/sdk';
import { readItems, readItem } from '@directus/sdk';
import type { Query } from '@directus/types';

export function useItems(collection: string) {
  const items = ref([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  
  async function getItems(query?: Query) {
    loading.value = true;
    error.value = null;
    
    try {
      items.value = await sdk.request(readItems(collection, query));
    } catch (err) {
      error.value = err as Error;
    } finally {
      loading.value = false;
    }
  }
  
  async function getItem(id: string | number, query?: Query) {
    loading.value = true;
    error.value = null;
    
    try {
      return await sdk.request(readItem(collection, id, query));
    } catch (err) {
      error.value = err as Error;
      throw err;
    } finally {
      loading.value = false;
    }
  }
  
  return {
    items,
    loading,
    error,
    getItems,
    getItem,
  };
}

// Usage in component
const { items, loading, getItems } = useItems('articles');

onMounted(() => {
  getItems({
    filter: { status: { _eq: 'published' } },
    sort: ['-date_created'],
  });
});
```

### Pattern 4: Cached Fetching

Implement caching to reduce API calls:

```typescript
// app/src/stores/fields.ts
import { defineStore } from 'pinia';
import sdk from '@/sdk';
import { readFields } from '@directus/sdk';
import type { Field } from '@directus/types';

export const useFieldsStore = defineStore('fieldsStore', () => {
  const fields = ref<Field[]>([]);
  const lastFetch = ref<number>(0);
  const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes
  
  async function hydrate(force = false) {
    const now = Date.now();
    
    // Return cached data if still valid
    if (!force && fields.value.length > 0 && (now - lastFetch.value) < CACHE_DURATION) {
      return;
    }
    
    try {
      fields.value = await sdk.request(readFields());
      lastFetch.value = now;
    } catch (error) {
      console.error('Failed to load fields:', error);
    }
  }
  
  // Get fields for specific collection
  const getFieldsForCollection = computed(() => {
    return (collection: string) => {
      return fields.value.filter(f => f.collection === collection);
    };
  });
  
  return {
    fields,
    hydrate,
    getFieldsForCollection,
  };
});
```



## Real-time Communication

### WebSocket Connection

The SDK provides real-time subscriptions via WebSocket:

```typescript
// app/src/composables/use-realtime.ts
import { ref, onUnmounted } from 'vue';
import sdk from '@/sdk';
import { subscribe } from '@directus/sdk';

export function useRealtime(collection: string, callback: (data: any) => void) {
  const connected = ref(false);
  const error = ref<Error | null>(null);
  let unsubscribe: (() => void) | null = null;
  
  async function connect() {
    try {
      const subscription = await sdk.subscribe('items', {
        collection,
        query: { fields: ['*'] },
      }, (data) => {
        callback(data);
      });
      
      unsubscribe = subscription.unsubscribe;
      connected.value = true;
    } catch (err) {
      error.value = err as Error;
      console.error('WebSocket connection failed:', err);
    }
  }
  
  function disconnect() {
    if (unsubscribe) {
      unsubscribe();
      unsubscribe = null;
      connected.value = false;
    }
  }
  
  // Auto-cleanup on component unmount
  onUnmounted(() => {
    disconnect();
  });
  
  return {
    connected,
    error,
    connect,
    disconnect,
  };
}

// Usage in component
const { connected, connect } = useRealtime('articles', (data) => {
  console.log('Article updated:', data);
  // Update local state
  refreshItems();
});

onMounted(() => {
  connect();
});
```

### Real-time Updates in Stores

Integrate WebSocket updates into Pinia stores:

```typescript
// app/src/stores/items.ts
import { defineStore } from 'pinia';
import sdk from '@/sdk';
import { readItems, subscribe } from '@directus/sdk';

export const useItemsStore = defineStore('itemsStore', () => {
  const items = ref<Map<string, any[]>>(new Map());
  const subscriptions = new Map<string, () => void>();
  
  // Subscribe to collection changes
  async function subscribeToCollection(collection: string) {
    // Don't subscribe twice
    if (subscriptions.has(collection)) return;
    
    try {
      const subscription = await sdk.subscribe('items', {
        collection,
        query: { fields: ['*'] },
      }, (data) => {
        handleRealtimeUpdate(collection, data);
      });
      
      subscriptions.set(collection, subscription.unsubscribe);
    } catch (error) {
      console.error(`Failed to subscribe to ${collection}:`, error);
    }
  }
  
  // Handle real-time updates
  function handleRealtimeUpdate(collection: string, data: any) {
    const collectionItems = items.value.get(collection) || [];
    
    switch (data.event) {
      case 'create':
        collectionItems.push(data.data);
        break;
      
      case 'update':
        const updateIndex = collectionItems.findIndex(i => i.id === data.data.id);
        if (updateIndex !== -1) {
          collectionItems[updateIndex] = { ...collectionItems[updateIndex], ...data.data };
        }
        break;
      
      case 'delete':
        const deleteIndex = collectionItems.findIndex(i => i.id === data.data.id);
        if (deleteIndex !== -1) {
          collectionItems.splice(deleteIndex, 1);
        }
        break;
    }
    
    items.value.set(collection, collectionItems);
  }
  
  // Cleanup subscriptions
  function unsubscribeAll() {
    subscriptions.forEach(unsubscribe => unsubscribe());
    subscriptions.clear();
  }
  
  return {
    items,
    subscribeToCollection,
    unsubscribeAll,
  };
});
```



## Request Queue Management

### Why Request Queuing?

The app implements request queuing to:
1. Prevent overwhelming the API with concurrent requests
2. Coordinate authentication token refresh
3. Implement rate limiting on the client side
4. Track active requests for loading states

### Queue Implementation

```typescript
// app/src/api.ts
import PQueue from 'p-queue';

// Shared queue for both SDK and Axios
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,        // Max 5 concurrent requests
  intervalCap: 5,        // Max 5 requests per 500ms
  interval: 500,         // Time window
  carryoverConcurrencyCount: true,
});

// Queue management functions
export function pauseQueue() {
  requestQueue.pause();
}

export function resumeQueue() {
  if (!requestQueue.isPaused) return;
  requestQueue.start();
}

export async function replaceQueue(options?: Options) {
  await requestQueue.onIdle();
  requestQueue = new PQueue(options);
}
```

### Queue Integration with SDK

```typescript
// app/src/sdk.ts
const baseClient = ofetch.create({
  async onRequest({ request, options }) {
    const path = getUrlPath(request);
    
    // Special handling for auth refresh
    if (path === '/auth/refresh') {
      requestQueue.pause();  // Pause all other requests
      return;
    }
    
    // Queue all other requests
    return new Promise((resolve) => {
      requestQueue.add(() => resolve());
    });
  },
});
```

### Request Tracking

Track active requests for loading indicators:

```typescript
// app/src/stores/requests.ts
import { defineStore } from 'pinia';
import { nanoid } from 'nanoid';

export const useRequestsStore = defineStore('requestsStore', {
  state: () => ({
    queue: [] as string[],
  }),
  
  getters: {
    queueHasItems: (state) => state.queue.length > 0,
  },
  
  actions: {
    startRequest(): string {
      const id = nanoid();
      this.queue.push(id);
      return id;
    },
    
    endRequest(id: string) {
      const index = this.queue.indexOf(id);
      if (index !== -1) {
        this.queue.splice(index, 1);
      }
    },
  },
});

// Usage in components
const requestsStore = useRequestsStore();
const isLoading = computed(() => requestsStore.queueHasItems);
```



## Error Handling

### Error Types from API

The API returns standardized errors using `@directus/errors`:

```typescript
// Common error codes
- INVALID_CREDENTIALS      // Login failed
- INVALID_TOKEN           // Token expired/invalid
- FORBIDDEN               // No permission
- INVALID_PAYLOAD         // Validation failed
- RECORD_NOT_UNIQUE       // Duplicate entry
- VALUE_TOO_LONG          // Field value too long
- INVALID_QUERY           // Malformed query
- ROUTE_NOT_FOUND         // Endpoint doesn't exist
- SERVICE_UNAVAILABLE     // Server error
```

### Error Response Structure

```typescript
// API error response
{
  errors: [
    {
      message: "Invalid credentials",
      extensions: {
        code: "INVALID_CREDENTIALS"
      }
    }
  ]
}
```

### Handling Errors in App

```typescript
// app/src/composables/use-api-error.ts
import { ref } from 'vue';
import type { AxiosError } from 'axios';

export function useApiError() {
  const error = ref<string | null>(null);
  
  function handleError(err: unknown) {
    if (isAxiosError(err)) {
      const apiError = err.response?.data?.errors?.[0];
      
      if (apiError) {
        const code = apiError.extensions?.code;
        
        // Map error codes to user-friendly messages
        switch (code) {
          case 'INVALID_CREDENTIALS':
            error.value = 'Invalid email or password';
            break;
          case 'FORBIDDEN':
            error.value = 'You don\'t have permission to perform this action';
            break;
          case 'INVALID_PAYLOAD':
            error.value = apiError.message || 'Invalid data provided';
            break;
          case 'RECORD_NOT_UNIQUE':
            error.value = 'This record already exists';
            break;
          default:
            error.value = apiError.message || 'An error occurred';
        }
      } else {
        error.value = 'Network error. Please try again.';
      }
    } else {
      error.value = 'An unexpected error occurred';
    }
  }
  
  function clearError() {
    error.value = null;
  }
  
  return {
    error,
    handleError,
    clearError,
  };
}

function isAxiosError(error: unknown): error is AxiosError {
  return (error as AxiosError).isAxiosError === true;
}
```

### Global Error Handler

```typescript
// app/src/main.ts
import { createApp } from 'vue';
import { unexpectedError } from '@/utils/unexpected-error';

const app = createApp(App);

// Global error handler
app.config.errorHandler = (err, instance, info) => {
  console.error('Global error:', err);
  unexpectedError(err);
};

// Unhandled promise rejection
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
  unexpectedError(event.reason);
});
```

### Error Notifications

```typescript
// app/src/utils/unexpected-error.ts
import { useNotificationsStore } from '@/stores/notifications';

export function unexpectedError(error: unknown) {
  const notificationsStore = useNotificationsStore();
  
  let message = 'An unexpected error occurred';
  
  if (error instanceof Error) {
    message = error.message;
  } else if (typeof error === 'string') {
    message = error;
  }
  
  notificationsStore.add({
    title: 'Error',
    text: message,
    type: 'error',
    dialog: true,
  });
}
```



## Type Safety Across Stack

### Shared Types from @directus/types

All three layers share type definitions:

```typescript
// @directus/types provides types for:

// Collections
interface Collection {
  collection: string;
  meta: CollectionMeta | null;
  schema: SchemaOverview | null;
}

// Fields
interface Field {
  collection: string;
  field: string;
  type: string;
  schema: FieldSchema | null;
  meta: FieldMeta | null;
}

// Items
interface Item {
  [key: string]: any;
}

// Users
interface User {
  id: string;
  email: string;
  first_name: string;
  last_name: string;
  role: string | Role;
  status: 'active' | 'suspended' | 'invited';
}

// Permissions
interface Permission {
  id: string;
  policy: string;
  collection: string;
  action: 'create' | 'read' | 'update' | 'delete' | 'share';
  permissions: Filter | null;
  validation: Filter | null;
  fields: string[] | null;
}

// Query
interface Query {
  fields?: string[];
  filter?: Filter;
  search?: string;
  sort?: Sort[];
  limit?: number;
  offset?: number;
  page?: number;
  deep?: Record<string, Query>;
  alias?: Record<string, string>;
}
```

### Type-Safe API Calls

```typescript
// Backend (API) - Type-safe service
// api/src/services/collections.ts
import type { Collection, Accountability } from '@directus/types';

export class CollectionsService {
  accountability: Accountability | null;
  
  async readByQuery(): Promise<Collection[]> {
    // TypeScript ensures return type matches
    const collections = await this.knex
      .select('*')
      .from('directus_collections');
    
    return collections;
  }
}

// Frontend (App) - Type-safe store
// app/src/stores/collections.ts
import type { Collection } from '@directus/types';
import { readCollections } from '@directus/sdk';

export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  
  async function hydrate() {
    // TypeScript ensures type safety
    collections.value = await sdk.request(readCollections());
  }
  
  return { collections, hydrate };
});

// SDK - Type-safe commands
// @directus/sdk
export function readCollections(): RestCommand<Collection[], Schema> {
  return {
    path: '/collections',
    method: 'GET',
  };
}
```

### Generic Type Parameters

The SDK supports generic type parameters for custom schemas:

```typescript
// Define your schema
interface MySchema {
  articles: {
    id: number;
    title: string;
    content: string;
    author: {
      id: number;
      name: string;
    };
  };
  users: {
    id: number;
    name: string;
    email: string;
  };
}

// Create typed client
const client = createDirectus<MySchema>('https://directus.example.com')
  .with(rest())
  .with(authentication('session'));

// Type-safe queries
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name'],  // Autocomplete works!
    filter: {
      title: { _contains: 'TypeScript' }     // Type-checked!
    }
  })
);

// articles is typed as:
// Array<{ id: number; title: string; author: { name: string } }>
```



## Complete Examples

### Example 1: CRUD Operations

Complete flow from UI to database:

```typescript
// 1. Component (UI Layer)
// app/src/modules/content/routes/item.vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import sdk from '@/sdk';
import { readItem, updateItem, deleteItem } from '@directus/sdk';
import { useNotificationsStore } from '@/stores/notifications';

const props = defineProps<{
  collection: string;
  primaryKey: string;
}>();

const router = useRouter();
const notificationsStore = useNotificationsStore();

const item = ref<any>(null);
const loading = ref(false);
const saving = ref(false);

// READ
async function fetchItem() {
  loading.value = true;
  
  try {
    item.value = await sdk.request(
      readItem(props.collection, props.primaryKey, {
        fields: ['*', 'user_created.*', 'user_updated.*'],
      })
    );
  } catch (error) {
    notificationsStore.add({
      title: 'Error',
      text: 'Failed to load item',
      type: 'error',
    });
  } finally {
    loading.value = false;
  }
}

// UPDATE
async function saveItem(updates: Record<string, any>) {
  saving.value = true;
  
  try {
    await sdk.request(
      updateItem(props.collection, props.primaryKey, updates)
    );
    
    notificationsStore.add({
      title: 'Success',
      text: 'Item saved successfully',
      type: 'success',
    });
    
    await fetchItem(); // Refresh
  } catch (error) {
    notificationsStore.add({
      title: 'Error',
      text: 'Failed to save item',
      type: 'error',
    });
  } finally {
    saving.value = false;
  }
}

// DELETE
async function deleteCurrentItem() {
  if (!confirm('Are you sure you want to delete this item?')) return;
  
  try {
    await sdk.request(
      deleteItem(props.collection, props.primaryKey)
    );
    
    notificationsStore.add({
      title: 'Success',
      text: 'Item deleted successfully',
      type: 'success',
    });
    
    router.push(`/content/${props.collection}`);
  } catch (error) {
    notificationsStore.add({
      title: 'Error',
      text: 'Failed to delete item',
      type: 'error',
    });
  }
}

onMounted(() => {
  fetchItem();
});
</script>

// 2. SDK Layer
// @directus/sdk/src/rest/commands/read/items.ts
export function readItem<Schema, Collection extends keyof Schema>(
  collection: Collection,
  key: string | number,
  query?: Query<Schema, Collection>
): RestCommand<Schema[Collection], Schema> {
  return {
    path: `/items/${collection}/${key}`,
    method: 'GET',
    params: query,
  };
}

// 3. API Layer - Controller
// api/src/controllers/items.ts
router.get('/:collection/:id', asyncHandler(async (req, res) => {
  const service = new ItemsService(req.params.collection, {
    accountability: req.accountability,
    schema: req.schema,
  });
  
  const item = await service.readOne(req.params.id, req.sanitizedQuery);
  
  res.json({ data: item });
}));

// 4. API Layer - Service
// api/src/services/items.ts
export class ItemsService {
  async readOne(key: PrimaryKey, query?: Query): Promise<Item> {
    // Check permissions
    await this.checkPermissions('read', key);
    
    // Build query
    const ast = await getAstFromQuery(this.collection, query, this.schema);
    
    // Execute query
    const result = await runAst(ast, this.schema, {
      knex: this.knex,
      accountability: this.accountability,
    });
    
    if (!result || result.length === 0) {
      throw new ForbiddenError();
    }
    
    return result[0];
  }
}

// 5. Database Layer
// Knex executes SQL query
SELECT * FROM articles WHERE id = ?
```



### Example 2: File Upload with Progress

Complete file upload flow:

```typescript
// 1. Component with Upload Progress
// app/src/components/file-upload.vue
<script setup lang="ts">
import { ref } from 'vue';
import api from '@/api';
import { useNotificationsStore } from '@/stores/notifications';

const notificationsStore = useNotificationsStore();

const uploading = ref(false);
const progress = ref(0);
const uploadedFile = ref<any>(null);

async function handleFileSelect(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  
  if (!file) return;
  
  await uploadFile(file);
}

async function uploadFile(file: File) {
  uploading.value = true;
  progress.value = 0;
  
  const formData = new FormData();
  formData.append('file', file);
  formData.append('folder', 'uploads');
  
  try {
    const response = await api.post('/files', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
      },
      onUploadProgress: (progressEvent) => {
        if (progressEvent.total) {
          progress.value = Math.round(
            (progressEvent.loaded / progressEvent.total) * 100
          );
        }
      },
    });
    
    uploadedFile.value = response.data.data;
    
    notificationsStore.add({
      title: 'Success',
      text: `File "${file.name}" uploaded successfully`,
      type: 'success',
    });
  } catch (error) {
    notificationsStore.add({
      title: 'Error',
      text: 'Failed to upload file',
      type: 'error',
    });
  } finally {
    uploading.value = false;
  }
}
</script>

<template>
  <div>
    <input
      type="file"
      @change="handleFileSelect"
      :disabled="uploading"
    />
    
    <div v-if="uploading" class="progress">
      <div class="progress-bar" :style="{ width: `${progress}%` }">
        {{ progress }}%
      </div>
    </div>
    
    <div v-if="uploadedFile">
      <p>Uploaded: {{ uploadedFile.filename_download }}</p>
      <img
        v-if="uploadedFile.type.startsWith('image/')"
        :src="`/assets/${uploadedFile.id}`"
        :alt="uploadedFile.title"
      />
    </div>
  </div>
</template>

// 2. API Controller
// api/src/controllers/files.ts
import busboy from 'busboy';
import { Readable } from 'stream';

router.post('/', asyncHandler(async (req, res) => {
  const service = new FilesService({
    accountability: req.accountability,
    schema: req.schema,
  });
  
  const bb = busboy({ headers: req.headers });
  const savedFiles: any[] = [];
  
  bb.on('file', async (fieldname, fileStream, info) => {
    const { filename, mimeType } = info;
    
    try {
      const file = await service.uploadOne(fileStream, {
        filename_download: filename,
        type: mimeType,
        folder: req.body.folder,
      });
      
      savedFiles.push(file);
    } catch (error) {
      fileStream.resume(); // Drain stream
      throw error;
    }
  });
  
  bb.on('finish', () => {
    res.json({ data: savedFiles[0] });
  });
  
  req.pipe(bb);
}));

// 3. Files Service
// api/src/services/files.ts
export class FilesService {
  async uploadOne(
    stream: Readable,
    data: Partial<File>
  ): Promise<File> {
    // Generate file ID
    const fileId = uuid();
    
    // Get storage location
    const storage = await getStorage();
    
    // Upload to storage
    await storage.write(`${fileId}`, stream);
    
    // Get file stats
    const stats = await storage.stat(`${fileId}`);
    
    // Save metadata to database
    const fileData = {
      id: fileId,
      storage: 'local',
      filename_disk: fileId,
      filename_download: data.filename_download,
      type: data.type,
      filesize: stats.size,
      folder: data.folder,
      uploaded_by: this.accountability?.user,
      uploaded_on: new Date(),
    };
    
    await this.knex.insert(fileData).into('directus_files');
    
    return fileData;
  }
}
```



### Example 3: Real-time Collaboration

Complete real-time update flow:

```typescript
// 1. Component with Real-time Updates
// app/src/modules/content/routes/collection.vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import sdk from '@/sdk';
import { readItems, subscribe } from '@directus/sdk';

const props = defineProps<{
  collection: string;
}>();

const items = ref<any[]>([]);
let unsubscribe: (() => void) | null = null;

// Initial fetch
async function fetchItems() {
  items.value = await sdk.request(
    readItems(props.collection, {
      fields: ['*'],
      sort: ['-date_created'],
      limit: 50,
    })
  );
}

// Subscribe to real-time updates
async function subscribeToUpdates() {
  try {
    const subscription = await sdk.subscribe('items', {
      collection: props.collection,
      query: { fields: ['*'] },
    }, (data) => {
      handleRealtimeUpdate(data);
    });
    
    unsubscribe = subscription.unsubscribe;
  } catch (error) {
    console.error('Failed to subscribe:', error);
  }
}

// Handle real-time events
function handleRealtimeUpdate(data: any) {
  switch (data.event) {
    case 'create':
      // Add new item to list
      items.value.unshift(data.data);
      break;
    
    case 'update':
      // Update existing item
      const updateIndex = items.value.findIndex(i => i.id === data.data.id);
      if (updateIndex !== -1) {
        items.value[updateIndex] = {
          ...items.value[updateIndex],
          ...data.data,
        };
      }
      break;
    
    case 'delete':
      // Remove deleted item
      const deleteIndex = items.value.findIndex(i => i.id === data.data.id);
      if (deleteIndex !== -1) {
        items.value.splice(deleteIndex, 1);
      }
      break;
  }
}

onMounted(async () => {
  await fetchItems();
  await subscribeToUpdates();
});

onUnmounted(() => {
  if (unsubscribe) {
    unsubscribe();
  }
});
</script>

// 2. WebSocket Server
// api/src/websocket/controllers/subscriptions.ts
export function createSubscriptionController(server: http.Server) {
  const wss = new WebSocketServer({ server, path: '/websocket' });
  
  wss.on('connection', (ws, request) => {
    const accountability = getAccountabilityFromRequest(request);
    
    ws.on('message', async (message) => {
      const data = JSON.parse(message.toString());
      
      if (data.type === 'subscribe') {
        await handleSubscribe(ws, data, accountability);
      } else if (data.type === 'unsubscribe') {
        handleUnsubscribe(ws, data.uid);
      }
    });
  });
}

async function handleSubscribe(
  ws: WebSocket,
  data: any,
  accountability: Accountability
) {
  const { collection, query, uid } = data;
  
  // Check permissions
  const permissionsService = new PermissionsService({
    accountability,
    schema: await getSchema(),
  });
  
  const hasPermission = await permissionsService.checkPermission(
    collection,
    'read'
  );
  
  if (!hasPermission) {
    ws.send(JSON.stringify({
      type: 'error',
      uid,
      error: 'Forbidden',
    }));
    return;
  }
  
  // Register subscription
  subscriptions.set(uid, {
    ws,
    collection,
    query,
    accountability,
  });
  
  // Send confirmation
  ws.send(JSON.stringify({
    type: 'subscription',
    uid,
    event: 'init',
  }));
}

// 3. Event Broadcasting
// api/src/services/items.ts
export class ItemsService {
  async createOne(data: Partial<Item>): Promise<PrimaryKey> {
    // Create item in database
    const key = await this.knex
      .insert(data)
      .into(this.collection)
      .returning('id');
    
    // Emit event for WebSocket subscribers
    emitter.emitAction('items.create', {
      collection: this.collection,
      key,
      payload: data,
    });
    
    // Broadcast to WebSocket clients
    broadcastToSubscribers(this.collection, {
      event: 'create',
      data: { id: key, ...data },
    });
    
    return key;
  }
}

// Broadcast helper
function broadcastToSubscribers(collection: string, message: any) {
  for (const [uid, subscription] of subscriptions) {
    if (subscription.collection === collection) {
      subscription.ws.send(JSON.stringify({
        type: 'subscription',
        uid,
        ...message,
      }));
    }
  }
}
```




## Best Practices

### 1. Use SDK for Standard Operations

```typescript
// ✅ Good - Use SDK for type-safe operations
import sdk from '@/sdk';
import { readItems, createItem } from '@directus/sdk';

const articles = await sdk.request(readItems('articles'));
const newArticle = await sdk.request(createItem('articles', data));

// ❌ Avoid - Direct axios for standard operations
import api from '@/api';
const response = await api.get('/items/articles');
```

### 2. Centralize Data in Stores

```typescript
// ✅ Good - Use Pinia stores for shared data
const collectionsStore = useCollectionsStore();
await collectionsStore.hydrate();
const collection = collectionsStore.getCollection('articles');

// ❌ Avoid - Fetching same data in multiple components
// Component A
const articles = await sdk.request(readCollection('articles'));
// Component B
const articles = await sdk.request(readCollection('articles'));
```

### 3. Handle Errors Gracefully

```typescript
// ✅ Good - Proper error handling
try {
  await sdk.request(updateItem('articles', id, data));
  notificationsStore.add({
    title: 'Success',
    text: 'Article updated',
    type: 'success',
  });
} catch (error) {
  const { handleError } = useApiError();
  handleError(error);
}

// ❌ Avoid - Silent failures
await sdk.request(updateItem('articles', id, data));
```

### 4. Implement Loading States

```typescript
// ✅ Good - Show loading indicators
const loading = ref(false);

async function fetchData() {
  loading.value = true;
  try {
    data.value = await sdk.request(readItems('articles'));
  } finally {
    loading.value = false;
  }
}

// ❌ Avoid - No feedback to user
async function fetchData() {
  data.value = await sdk.request(readItems('articles'));
}
```

### 5. Clean Up Subscriptions

```typescript
// ✅ Good - Clean up on unmount
let unsubscribe: (() => void) | null = null;

onMounted(async () => {
  const subscription = await sdk.subscribe('items', config, callback);
  unsubscribe = subscription.unsubscribe;
});

onUnmounted(() => {
  if (unsubscribe) unsubscribe();
});

// ❌ Avoid - Memory leaks
onMounted(async () => {
  await sdk.subscribe('items', config, callback);
  // Never cleaned up!
});
```

### 6. Use Type Parameters

```typescript
// ✅ Good - Type-safe with generics
interface Article {
  id: number;
  title: string;
  content: string;
}

const articles = await sdk.request<Article[]>(
  readItems('articles', {
    fields: ['id', 'title', 'content'],
  })
);
// articles is typed as Article[]

// ❌ Avoid - Losing type safety
const articles = await sdk.request(readItems('articles'));
// articles is typed as any[]
```

### 7. Batch Related Requests

```typescript
// ✅ Good - Fetch related data in single request
const articles = await sdk.request(
  readItems('articles', {
    fields: ['*', 'author.*', 'category.*'],
  })
);

// ❌ Avoid - Multiple requests for related data
const articles = await sdk.request(readItems('articles'));
for (const article of articles) {
  article.author = await sdk.request(readUser(article.author_id));
  article.category = await sdk.request(readItem('categories', article.category_id));
}
```

## Summary

### Key Takeaways

1. **Dual Client Approach**: App uses both SDK (primary) and Axios (specific cases)
2. **Request Queue**: Shared queue manages concurrency and coordinates auth refresh
3. **Type Safety**: `@directus/types` ensures consistency across all layers
4. **Error Handling**: Standardized error codes with user-friendly messages
5. **Real-time**: WebSocket subscriptions for live updates
6. **Authentication**: Automatic token refresh with request queuing
7. **State Management**: Pinia stores centralize data and API calls

### Communication Flow

```
User Action
    ↓
Vue Component
    ↓
Pinia Store (optional)
    ↓
SDK or Axios Client
    ↓
Request Queue
    ↓
HTTP Request
    ↓
API Controller
    ↓
Service Layer
    ↓
Database (Knex)
    ↓
Response flows back up
```

### Next Steps

- [Backend Architecture](./04-backend-architecture.md) - API structure details
- [Frontend Overview](./07-frontend-overview.md) - App architecture
- [SDK Architecture](./14-sdk-architecture.md) - SDK internals
- [State Management](./12-state-management.md) - Pinia stores
- [Real-time Features](./16-realtime-features.md) - WebSocket details

