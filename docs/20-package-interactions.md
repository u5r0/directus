# Package Interactions & Architecture

This document provides a comprehensive overview of how the four main directories (`/api`, `/app`, `/packages`, `/sdk`) interact with each other in the Directus monorepo.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Package Dependency Graph](#package-dependency-graph)
- [The API Package](#the-api-package)
- [The App Package](#the-app-package)
- [The SDK Package](#the-sdk-package)
- [Shared Packages](#shared-packages)
- [Data Flow Patterns](#data-flow-patterns)
- [Extension System Integration](#extension-system-integration)

## Architecture Overview

### High-Level Structure
```
directus/                    # Main deployment package
├── cli.js                   # CLI entry point
└── dependencies:
    ├── @directus/api        # Backend server
    └── @directus/app        # Frontend dashboard

@directus/api/               # Express.js backend
├── Controllers              # HTTP route handlers
├── Services                 # Business logic
├── Database                 # Knex + Schema Inspector
└── dependencies:
    ├── @directus/app        # Serves the frontend
    ├── @directus/types      # Shared type definitions
    ├── @directus/schema     # Database schema tools
    ├── @directus/storage    # File storage abstraction
    ├── @directus/errors     # Error classes
    ├── @directus/env        # Environment config
    ├── @directus/utils      # Utility functions
    └── 20+ other packages

@directus/app/               # Vue 3 frontend
├── Components               # UI components
├── Modules                  # Feature modules
├── Stores                   # Pinia state management
└── dependencies:
    ├── @directus/sdk        # API client
    ├── @directus/types      # Shared type definitions
    ├── @directus/composables # Vue composables
    ├── @directus/stores     # Shared stores
    ├── @directus/utils      # Utility functions
    └── 10+ other packages

@directus/sdk/               # TypeScript SDK
├── REST Client              # REST API methods
├── GraphQL Client           # GraphQL support
├── Authentication           # Auth management
├── Realtime                 # WebSocket subscriptions
└── dependencies:
    └── @directus/system-data # System collection schemas
```

## Package Dependency Graph

### Complete Dependency Flow
```
┌─────────────────────────────────────────────────────────────┐
│                      directus (main)                        │
│                    CLI & Deployment                         │
└────────────────┬────────────────────────────┬───────────────┘
                 │                            │
        ┌────────▼────────┐          ┌───────▼────────┐
        │  @directus/api  │          │ @directus/app  │
        │  (Backend)      │◄─────────┤  (Frontend)    │
        └────────┬────────┘  Serves  └───────┬────────┘
                 │                            │
                 │                            │
        ┌────────▼────────────────────────────▼────────┐
        │         @directus/sdk (API Client)           │
        │  REST + GraphQL + Auth + Realtime            │
        └────────┬─────────────────────────────────────┘
                 │
        ┌────────▼────────────────────────────────────┐
        │         Shared Packages (30+)               │
        ├─────────────────────────────────────────────┤
        │ @directus/types      - Type definitions     │
        │ @directus/schema     - DB schema tools      │
        │ @directus/utils      - Utilities            │
        │ @directus/errors     - Error classes        │
        │ @directus/env        - Environment config   │
        │ @directus/storage    - File storage         │
        │ @directus/extensions - Extension system     │
        │ @directus/composables - Vue composables     │
        │ @directus/stores     - Pinia stores         │
        │ ... and 20+ more                            │
        └─────────────────────────────────────────────┘
```

## The API Package

### Location & Purpose
- **Path**: `/api`
- **Package Name**: `@directus/api`
- **Purpose**: Express.js backend server providing REST and GraphQL APIs
- **Entry Point**: `dist/index.js`

### Key Dependencies
```json
{
  "dependencies": {
    // Frontend (served by API)
    "@directus/app": "workspace:*",
    
    // Core shared packages
    "@directus/types": "workspace:*",
    "@directus/schema": "workspace:*",
    "@directus/utils": "workspace:*",
    "@directus/errors": "workspace:*",
    "@directus/env": "workspace:*",
    
    // Storage system
    "@directus/storage": "workspace:*",
    "@directus/storage-driver-local": "workspace:*",
    "@directus/storage-driver-s3": "workspace:*",
    "@directus/storage-driver-azure": "workspace:*",
    "@directus/storage-driver-gcs": "workspace:*",
    "@directus/storage-driver-cloudinary": "workspace:*",
    "@directus/storage-driver-supabase": "workspace:*",
    
    // Extension system
    "@directus/extensions": "workspace:*",
    "@directus/extensions-sdk": "workspace:*",
    "@directus/extensions-registry": "workspace:*",
    
    // Utilities
    "@directus/format-title": "workspace:*",
    "@directus/memory": "workspace:*",
    "@directus/pressure": "workspace:*",
    "@directus/validation": "workspace:*",
    "@directus/system-data": "workspace:*",
    "@directus/constants": "workspace:*",
    "@directus/specs": "workspace:*",
    
    // External dependencies
    "express": "4.21.2",
    "knex": "3.1.0",
    "graphql": "16.12.0",
    "ws": "8.18.3",
    // ... 50+ more
  }
}
```

### How API Uses Shared Packages

#### 1. Type Safety with @directus/types
```typescript
// api/src/services/collections.ts
import type { Collection, Field, Accountability } from '@directus/types';

export class CollectionsService {
  accountability: Accountability | null;
  
  async readByQuery(): Promise<Collection[]> {
    // Type-safe collection handling
  }
}
```

#### 2. Database Schema with @directus/schema
```typescript
// api/src/database/index.ts
import { createInspector } from '@directus/schema';
import type { SchemaInspector } from '@directus/schema';

export function getSchemaInspector(database: Knex): SchemaInspector {
  return createInspector(database);
}
```

#### 3. Error Handling with @directus/errors
```typescript
// api/src/controllers/collections.ts
import { ForbiddenError, InvalidPayloadError } from '@directus/errors';

if (!hasPermission) {
  throw new ForbiddenError();
}

if (!data.collection) {
  throw new InvalidPayloadError({ reason: 'Collection name required' });
}
```

#### 4. Storage Abstraction with @directus/storage
```typescript
// api/src/services/files.ts
import { registerDrivers } from '@directus/storage';
import { DriverLocal } from '@directus/storage-driver-local';
import { DriverS3 } from '@directus/storage-driver-s3';

registerDrivers({
  local: DriverLocal,
  s3: DriverS3,
});
```

#### 5. Serving the App
```typescript
// api/src/app.ts
if (env['SERVE_APP']) {
  const adminPath = require.resolve('@directus/app');
  
  // Serve static files from @directus/app/dist
  app.use('/admin', express.static(path.join(adminPath, '..')));
}
```

## The App Package

### Location & Purpose
- **Path**: `/app`
- **Package Name**: `@directus/app`
- **Purpose**: Vue 3 admin dashboard interface
- **Entry Point**: `dist/index.html`
- **Build Tool**: Vite

### Key Dependencies
```json
{
  "devDependencies": {
    // SDK for API communication
    "@directus/sdk": "workspace:*",
    
    // Core shared packages
    "@directus/types": "workspace:*",
    "@directus/utils": "workspace:*",
    "@directus/errors": "workspace:*",
    
    // Vue-specific packages
    "@directus/composables": "workspace:*",
    "@directus/stores": "workspace:*",
    "@directus/themes": "workspace:*",
    
    // Extension system
    "@directus/extensions": "workspace:*",
    "@directus/extensions-sdk": "workspace:*",
    "@directus/extensions-registry": "workspace:*",
    
    // Utilities
    "@directus/format-title": "workspace:*",
    "@directus/validation": "workspace:*",
    "@directus/system-data": "workspace:*",
    "@directus/constants": "workspace:*",
    
    // Vue ecosystem
    "vue": "3.5.24",
    "pinia": "2.3.1",
    "vue-router": "4.6.3",
    "vite": "7.1.12",
    // ... 50+ more
  }
}
```

### How App Uses Shared Packages

#### 1. API Communication with @directus/sdk
```typescript
// app/src/sdk.ts
import { createDirectus, rest, authentication } from '@directus/sdk';
import { getPublicURL } from '@/utils/get-root-path';

export const sdk = createDirectus(getPublicURL())
  .with(authentication('session', { credentials: 'include' }))
  .with(rest({ credentials: 'include' }));
```

#### 2. HTTP Client with Axios
```typescript
// app/src/api.ts
import axios from 'axios';
import { getRootPath } from '@/utils/get-root-path';

const api = axios.create({
  baseURL: getRootPath(),
  withCredentials: true,
});

// Used alongside SDK for specific operations
export default api;
```

#### 3. Type Safety with @directus/types
```typescript
// app/src/stores/collections.ts
import type { Collection, Field } from '@directus/types';

export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  
  async function hydrate() {
    const response = await api.get<{ data: Collection[] }>('/collections');
    collections.value = response.data.data;
  }
});
```

#### 4. Shared Composables with @directus/composables
```typescript
// app/src/modules/content/routes/item.vue
import { useCollection } from '@directus/composables';

const { info: collectionInfo } = useCollection(collection);
```

#### 5. Shared Stores with @directus/stores
```typescript
// app/src/stores/index.ts
import { useServerStore } from '@directus/stores';

// Shared store logic used across the app
const serverStore = useServerStore();
```

#### 6. Utilities with @directus/utils
```typescript
// app/src/components/v-form.vue
import { isNil, merge } from '@directus/utils';

const mergedValues = merge(defaultValues, userValues);
```

## The SDK Package

### Location & Purpose
- **Path**: `/sdk`
- **Package Name**: `@directus/sdk`
- **Purpose**: TypeScript SDK for Directus API
- **Entry Points**: 
  - ESM: `dist/index.js`
  - CJS: `dist/index.cjs`
- **License**: MIT (open source)

### Architecture
```
sdk/src/
├── client.ts              # Core client factory
├── auth/                  # Authentication
│   ├── composable.ts      # Auth composable
│   ├── static.ts          # Static token auth
│   └── types.ts           # Auth types
├── rest/                  # REST API client
│   ├── composable.ts      # REST composable
│   ├── commands/          # CRUD commands
│   └── types.ts           # REST types
├── graphql/               # GraphQL client
│   ├── composable.ts      # GraphQL composable
│   └── types.ts           # GraphQL types
├── realtime/              # WebSocket subscriptions
│   ├── composable.ts      # Realtime composable
│   ├── commands/          # Subscription commands
│   └── types.ts           # Realtime types
├── schema/                # Type-safe schemas
│   ├── collection.ts      # Collection schemas
│   ├── user.ts            # User schemas
│   ├── file.ts            # File schemas
│   └── ... (20+ schemas)
├── types/                 # SDK type definitions
│   ├── client.ts          # Client types
│   ├── query.ts           # Query types
│   ├── filters.ts         # Filter types
│   └── ... (10+ types)
└── utils/                 # SDK utilities
```

### Key Dependencies
```json
{
  "dependencies": {
    // Only system data schemas
    "@directus/system-data": "workspace:*"
  }
}
```

**Note**: The SDK has minimal dependencies to keep it lightweight for external use.

### SDK Usage Patterns

#### 1. Creating a Client
```typescript
import { createDirectus, rest, authentication } from '@directus/sdk';

// Basic client
const client = createDirectus('https://directus.example.com')
  .with(rest())
  .with(authentication('session'));

// With custom options
const client = createDirectus('https://directus.example.com', {
  globals: {
    fetch: customFetch,
    WebSocket: customWebSocket,
  },
});
```

#### 2. REST Operations
```typescript
import { readItems, createItem, updateItem, deleteItem } from '@directus/sdk';

// Read items
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name'],
    filter: { status: { _eq: 'published' } },
    sort: ['-date_created'],
  })
);

// Create item
const newArticle = await client.request(
  createItem('articles', {
    title: 'New Article',
    content: 'Content...',
  })
);
```

#### 3. Authentication
```typescript
// Login
await client.login('user@example.com', 'password');

// Refresh token
await client.refresh();

// Logout
await client.logout();
```

#### 4. Real-time Subscriptions
```typescript
import { subscribe } from '@directus/sdk';

// Subscribe to collection changes
const subscription = client.subscribe('items', {
  collection: 'articles',
  query: { fields: ['*'] },
}, (data) => {
  console.log('Item changed:', data);
});

// Unsubscribe
subscription.unsubscribe();
```

#### 5. GraphQL
```typescript
import { graphql } from '@directus/sdk';

const result = await client.request(
  graphql(`
    query {
      articles {
        id
        title
        author {
          name
        }
      }
    }
  `)
);
```

### How SDK is Used

#### In the App (@directus/app)
```typescript
// app/src/sdk.ts
import { createDirectus, rest, authentication } from '@directus/sdk';

export const sdk = createDirectus(getPublicURL())
  .with(authentication('session'))
  .with(rest());

// Used throughout the app
const user = await sdk.request(readMe());
```

#### In External Applications
```typescript
// External app using the SDK
import { createDirectus, rest } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(rest());

const items = await client.request(readItems('products'));
```

## Shared Packages

### Complete Package List (30+)

#### Core Packages
1. **@directus/types** - TypeScript type definitions
2. **@directus/utils** - Shared utilities (browser/node/shared)
3. **@directus/errors** - Standardized error classes
4. **@directus/env** - Environment variable management
5. **@directus/constants** - Shared constants

#### Database & Schema
6. **@directus/schema** - Database schema inspection
7. **@directus/schema-builder** - Schema building utilities
8. **@directus/system-data** - System collection schemas

#### Storage
9. **@directus/storage** - Storage abstraction layer
10. **@directus/storage-driver-local** - Local filesystem driver
11. **@directus/storage-driver-s3** - AWS S3 driver
12. **@directus/storage-driver-azure** - Azure Blob Storage driver
13. **@directus/storage-driver-gcs** - Google Cloud Storage driver
14. **@directus/storage-driver-cloudinary** - Cloudinary driver
15. **@directus/storage-driver-supabase** - Supabase Storage driver

#### Extensions
16. **@directus/extensions** - Extension system core
17. **@directus/extensions-sdk** - Extension development SDK
18. **@directus/extensions-registry** - Extension registry

#### Vue-Specific
19. **@directus/composables** - Vue composables
20. **@directus/stores** - Shared Pinia stores
21. **@directus/themes** - Theme system

#### Utilities
22. **@directus/format-title** - String formatting
23. **@directus/validation** - Validation utilities
24. **@directus/memory** - Memory management
25. **@directus/pressure** - Server pressure monitoring
26. **@directus/specs** - OpenAPI specifications
27. **@directus/update-check** - Update checking

#### Development Tools
28. **@directus/create-directus-extension** - Extension scaffolding
29. **@directus/create-directus-project** - Project scaffolding
30. **@directus/release-notes-generator** - Release notes generation

### Package Interaction Examples

#### Example 1: File Upload Flow
```
User uploads file in @directus/app
    ↓
App uses @directus/sdk to POST to /files
    ↓
@directus/api receives request
    ↓
API uses @directus/storage to save file
    ↓
Storage uses @directus/storage-driver-s3 (or other driver)
    ↓
File saved to S3
    ↓
API returns file metadata using @directus/types
    ↓
App updates UI with new file
```

#### Example 2: Collection Creation Flow
```
User creates collection in @directus/app
    ↓
App uses @directus/sdk to POST to /collections
    ↓
@directus/api receives request
    ↓
API validates using @directus/validation
    ↓
API uses @directus/schema to create table
    ↓
Schema uses Knex to execute SQL
    ↓
API returns collection using @directus/types
    ↓
App updates stores with new collection
```

#### Example 3: Extension Loading Flow
```
@directus/api starts up
    ↓
Uses @directus/extensions to scan extensions directory
    ↓
Loads extension manifests using @directus/extensions-registry
    ↓
Validates extensions using @directus/extensions-sdk types
    ↓
Registers extension routes/hooks
    ↓
@directus/app loads extension components
    ↓
Uses @directus/extensions-sdk to register UI extensions
    ↓
Extensions available in app
```

## Data Flow Patterns

### 1. User Authentication Flow
```
┌─────────────┐
│ @directus/  │  1. User enters credentials
│    app      │────────────────────────────┐
└─────────────┘                            │
                                           ▼
┌─────────────┐                    ┌──────────────┐
│ @directus/  │  2. SDK sends      │ @directus/   │
│    sdk      │◄───login request───│    api       │
└─────────────┘                    └──────────────┘
       │                                   │
       │ 3. Returns tokens                 │ 4. Validates credentials
       │                                   │    using @directus/errors
       ▼                                   ▼
┌─────────────┐                    ┌──────────────┐
│ App stores  │                    │ Database     │
│ tokens      │                    │ (via Knex)   │
└─────────────┘                    └──────────────┘
```

### 2. Data Fetching Flow
```
┌─────────────┐
│ Component   │  1. Needs data
│ in app      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Pinia Store │  2. Checks cache
│ (@directus/ │
│   stores)   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ @directus/  │  3. Makes API request
│    sdk      │     with type safety
└──────┬──────┘     (@directus/types)
       │
       ▼
┌─────────────┐
│ @directus/  │  4. Processes request
│    api      │     - Checks permissions
└──────┬──────┘     - Validates query
       │            - Applies filters
       ▼
┌─────────────┐
│ Database    │  5. Executes query
│ (via Knex + │     using @directus/schema
│  @directus/ │
│   schema)   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Response    │  6. Returns typed data
│ flows back  │     through the stack
│ to component│
└─────────────┘
```

### 3. Real-time Update Flow
```
┌─────────────┐
│ Database    │  1. Item updated
│ Change      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ @directus/  │  2. Emits event
│    api      │
│ (EventBus)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ WebSocket   │  3. Broadcasts to
│ Server      │     subscribed clients
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ @directus/  │  4. Receives update
│    sdk      │     via WebSocket
│ (realtime)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ @directus/  │  5. Updates store
│    app      │     and UI reactively
│ (Pinia)     │
└─────────────┘
```

## Extension System Integration

### How Extensions Use Packages

#### API Extension (Hook)
```typescript
// extensions/my-hook/index.ts
import type { HookExtensionContext } from '@directus/extensions';
import type { Item } from '@directus/types';

export default ({ filter, action }: HookExtensionContext) => {
  // Uses @directus/types for type safety
  filter('items.create', async (input: Partial<Item>) => {
    // Modify item before creation
    return input;
  });
  
  action('items.create', async ({ payload }) => {
    // Perform action after creation
  });
};
```

#### App Extension (Interface)
```typescript
// extensions/my-interface/index.ts
import { defineInterface } from '@directus/extensions-sdk';
import InterfaceComponent from './interface.vue';

export default defineInterface({
  id: 'my-interface',
  name: 'My Interface',
  icon: 'box',
  component: InterfaceComponent,
  types: ['string'],
});
```

#### Extension Using SDK
```typescript
// extensions/my-endpoint/index.ts
import type { EndpointExtensionContext } from '@directus/extensions';
import { createDirectus, rest } from '@directus/sdk';

export default (router: Router, context: EndpointExtensionContext) => {
  const { services, getSchema } = context;
  
  router.get('/', async (req, res) => {
    const schema = await getSchema();
    const itemsService = new services.ItemsService('articles', {
      schema,
      accountability: req.accountability,
    });
    
    const items = await itemsService.readByQuery({});
    res.json(items);
  });
};
```

## Summary

### Key Takeaways

1. **Directus Package** - Main deployment package that bundles API and App
2. **API Package** - Backend server that serves both REST/GraphQL APIs and the frontend app
3. **App Package** - Vue 3 frontend that uses SDK to communicate with API
4. **SDK Package** - Lightweight TypeScript client used by App and external applications
5. **Shared Packages** - 30+ packages providing utilities, types, and functionality

### Interaction Patterns

- **API ← Packages**: API depends on 20+ shared packages for functionality
- **App ← SDK**: App uses SDK as primary API client
- **App ← Packages**: App uses 10+ shared packages for utilities and types
- **SDK ← Packages**: SDK has minimal dependencies (only system-data)
- **API → App**: API serves the built App as static files
- **Extensions ← All**: Extensions can use types and utilities from shared packages

### Type Safety Flow

All packages share `@directus/types` for consistent type definitions:
```
@directus/types (source of truth)
    ↓
├── Used by @directus/api (backend types)
├── Used by @directus/app (frontend types)
├── Used by @directus/sdk (client types)
└── Used by extensions (extension types)
```

This ensures type safety across the entire stack from database to UI.
