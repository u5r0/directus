# SDK Architecture

This document covers the Directus JavaScript/TypeScript SDK architecture, which provides a type-safe client for interacting with the Directus API.

## Overview

The Directus SDK (`@directus/sdk`) is a composable, type-safe JavaScript/TypeScript client that supports REST, GraphQL, and real-time (WebSocket) communication with Directus.

**Key Features:**
- Type-safe API with TypeScript
- Composable architecture
- Multiple authentication strategies
- REST and GraphQL support
- Real-time subscriptions via WebSocket
- Minimal dependencies
- Works in Node.js and browsers

## SDK Structure

### Package Information

```json
{
  "name": "@directus/sdk",
  "version": "20.3.0",
  "license": "MIT",
  "type": "module",
  "engines": {
    "node": ">=22"
  }
}
```

### Directory Structure

```
sdk/src/
├── auth/              # Authentication composables
│   ├── composable.ts  # Session authentication
│   ├── static.ts      # Static token authentication
│   └── utils/         # Memory storage utilities
├── rest/              # REST API composables
│   ├── commands/      # CRUD commands
│   ├── helpers/       # Helper functions
│   └── utils/         # Request utilities
├── graphql/           # GraphQL composables
│   ├── composable.ts  # GraphQL client
│   └── types.ts       # GraphQL types
├── realtime/          # WebSocket composables
│   ├── commands/      # Subscription commands
│   ├── utils/         # WebSocket utilities
│   └── composable.ts  # Real-time client
├── schema/            # Type definitions for collections
│   ├── core.ts        # Core types
│   ├── user.ts        # User schema
│   ├── file.ts        # File schema
│   └── ...            # Other system collections
├── types/             # TypeScript types
│   ├── client.ts      # Client types
│   ├── query.ts       # Query types
│   ├── filters.ts     # Filter types
│   └── ...            # Other types
├── utils/             # Utility functions
│   ├── request.ts     # HTTP request handler
│   └── ...            # Other utilities
├── client.ts          # Core client factory
└── index.ts           # Main exports
```

## Core Client

### Creating a Client

```typescript
import { createDirectus } from '@directus/sdk';

// Basic client
const client = createDirectus('https://your-directus.com');

// With TypeScript schema
interface MySchema {
  articles: Article[];
  authors: Author[];
}

const client = createDirectus<MySchema>('https://your-directus.com');
```

### Client Factory (`sdk/src/client.ts`)

```typescript
export const createDirectus = <Schema = any>(
  url: string,
  options: ClientOptions = {}
): DirectusClient<Schema> => {
  const globals = options.globals 
    ? { ...defaultGlobals, ...options.globals } 
    : defaultGlobals;
    
  return {
    globals,
    url: new globals.URL(url),
    with(createExtension) {
      return {
        ...this,
        ...createExtension(this),
      };
    },
  };
};
```

### Default Globals

```typescript
const defaultGlobals: ClientGlobals = {
  fetch: globalThis.fetch,
  WebSocket: globalThis.WebSocket,
  URL: globalThis.URL,
  logger: globalThis.console,
};
```

## Composable Architecture

The SDK uses a composable pattern where functionality is added via `.with()`:

```typescript
import { createDirectus, rest, authentication } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(authentication('session'))
  .with(rest());
```

### Available Composables

#### 1. Authentication

**Session Authentication:**
```typescript
import { authentication } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(authentication('session', {
    credentials: 'include',
    autoRefresh: true,
    msRefreshBeforeExpires: 30000,
  }));

// Login
await client.login('email@example.com', 'password');

// Logout
await client.logout();

// Refresh token
await client.refresh();
```

**Static Token:**
```typescript
import { staticToken } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(staticToken('your-static-token'));
```

#### 2. REST API

```typescript
import { rest, readItems, createItem, updateItem, deleteItem } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(rest());

// Read items
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name'],
    filter: {
      status: { _eq: 'published' }
    },
    sort: ['-created_at'],
    limit: 10,
  })
);

// Create item
const newArticle = await client.request(
  createItem('articles', {
    title: 'New Article',
    content: 'Article content...',
  })
);

// Update item
await client.request(
  updateItem('articles', 'item-id', {
    status: 'published',
  })
);

// Delete item
await client.request(
  deleteItem('articles', 'item-id')
);
```

#### 3. GraphQL

```typescript
import { graphql } from '@directus/sdk';

const client = createDirectus('https://your-directus.com')
  .with(graphql());

// Query
const result = await client.query(`
  query {
    articles(filter: { status: { _eq: "published" } }) {
      id
      title
      author {
        name
      }
    }
  }
`);

// Mutation
const result = await client.query(`
  mutation {
    create_articles_item(data: {
      title: "New Article"
      content: "Article content..."
    }) {
      id
      title
    }
  }
`);
```

#### 4. Real-time (WebSocket)

```typescript
import { realtime } from '@directus/sdk';

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

// Unsubscribe
unsubscribe();
```

## REST Commands

### Item Commands

```typescript
import {
  readItems,
  readItem,
  createItems,
  createItem,
  updateItems,
  updateItem,
  deleteItems,
  deleteItem,
} from '@directus/sdk';

// Read multiple items
const items = await client.request(readItems('articles'));

// Read single item
const item = await client.request(readItem('articles', 'item-id'));

// Create multiple items
const newItems = await client.request(createItems('articles', [
  { title: 'Article 1' },
  { title: 'Article 2' },
]));

// Create single item
const newItem = await client.request(createItem('articles', {
  title: 'New Article',
}));

// Update multiple items
await client.request(updateItems('articles', ['id1', 'id2'], {
  status: 'published',
}));

// Update single item
await client.request(updateItem('articles', 'item-id', {
  status: 'published',
}));

// Delete multiple items
await client.request(deleteItems('articles', ['id1', 'id2']));

// Delete single item
await client.request(deleteItem('articles', 'item-id'));
```

### Collection Commands

```typescript
import {
  readCollections,
  readCollection,
  createCollection,
  updateCollection,
  deleteCollection,
} from '@directus/sdk';

// Read all collections
const collections = await client.request(readCollections());

// Read single collection
const collection = await client.request(readCollection('articles'));

// Create collection
await client.request(createCollection({
  collection: 'products',
  fields: [
    {
      field: 'id',
      type: 'integer',
      schema: { is_primary_key: true },
    },
    {
      field: 'name',
      type: 'string',
    },
  ],
}));

// Update collection
await client.request(updateCollection('articles', {
  note: 'Updated note',
}));

// Delete collection
await client.request(deleteCollection('articles'));
```

### Field Commands

```typescript
import {
  readFields,
  readField,
  createField,
  updateField,
  deleteField,
} from '@directus/sdk';

// Read all fields in collection
const fields = await client.request(readFields('articles'));

// Read single field
const field = await client.request(readField('articles', 'title'));

// Create field
await client.request(createField('articles', {
  field: 'subtitle',
  type: 'string',
  meta: {
    interface: 'input',
    width: 'full',
  },
}));

// Update field
await client.request(updateField('articles', 'title', {
  meta: {
    required: true,
  },
}));

// Delete field
await client.request(deleteField('articles', 'subtitle'));
```

### File Commands

```typescript
import {
  uploadFiles,
  readFiles,
  readFile,
  updateFile,
  deleteFile,
} from '@directus/sdk';

// Upload file
const formData = new FormData();
formData.append('file', fileBlob);
formData.append('title', 'My File');

const uploadedFile = await client.request(uploadFiles(formData));

// Read files
const files = await client.request(readFiles());

// Read single file
const file = await client.request(readFile('file-id'));

// Update file
await client.request(updateFile('file-id', {
  title: 'Updated Title',
}));

// Delete file
await client.request(deleteFile('file-id'));
```

### User Commands

```typescript
import {
  readUsers,
  readUser,
  readMe,
  createUser,
  updateUser,
  deleteUser,
} from '@directus/sdk';

// Read current user
const me = await client.request(readMe());

// Read all users
const users = await client.request(readUsers());

// Read single user
const user = await client.request(readUser('user-id'));

// Create user
await client.request(createUser({
  email: 'user@example.com',
  password: 'password',
  role: 'role-id',
}));

// Update user
await client.request(updateUser('user-id', {
  first_name: 'John',
  last_name: 'Doe',
}));

// Delete user
await client.request(deleteUser('user-id'));
```

## Type Safety

### Schema Definition

Define your schema for full type safety:

```typescript
interface Article {
  id: number;
  title: string;
  content: string;
  status: 'draft' | 'published';
  author: Author;
  created_at: string;
}

interface Author {
  id: string;
  name: string;
  email: string;
}

interface MySchema {
  articles: Article[];
  authors: Author[];
}

const client = createDirectus<MySchema>('https://your-directus.com')
  .with(rest());

// TypeScript knows the return type
const articles: Article[] = await client.request(
  readItems('articles')
);
```

### Query Type Inference

```typescript
// TypeScript infers the return type based on query
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name'],
  })
);

// Type: { id: number; title: string; author: { name: string } }[]
```

### Filter Type Safety

```typescript
// TypeScript validates filter fields
const articles = await client.request(
  readItems('articles', {
    filter: {
      status: { _eq: 'published' },  // ✓ Valid
      // invalid_field: { _eq: 'value' }  // ✗ TypeScript error
    }
  })
);
```

## Query Building

### Fields Selection

```typescript
// Select specific fields
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'status'],
  })
);

// Select nested fields
const articles = await client.request(
  readItems('articles', {
    fields: ['id', 'title', 'author.name', 'author.email'],
  })
);

// Select all fields
const articles = await client.request(
  readItems('articles', {
    fields: ['*'],
  })
);

// Select all fields including nested
const articles = await client.request(
  readItems('articles', {
    fields: ['*', 'author.*'],
  })
);
```

### Filtering

```typescript
// Simple filter
const articles = await client.request(
  readItems('articles', {
    filter: {
      status: { _eq: 'published' }
    }
  })
);

// Multiple conditions (AND)
const articles = await client.request(
  readItems('articles', {
    filter: {
      _and: [
        { status: { _eq: 'published' } },
        { featured: { _eq: true } }
      ]
    }
  })
);

// OR conditions
const articles = await client.request(
  readItems('articles', {
    filter: {
      _or: [
        { status: { _eq: 'published' } },
        { status: { _eq: 'archived' } }
      ]
    }
  })
);

// Nested filters
const articles = await client.request(
  readItems('articles', {
    filter: {
      author: {
        name: { _contains: 'John' }
      }
    }
  })
);
```

### Sorting

```typescript
// Sort ascending
const articles = await client.request(
  readItems('articles', {
    sort: ['title']
  })
);

// Sort descending
const articles = await client.request(
  readItems('articles', {
    sort: ['-created_at']
  })
);

// Multiple sort fields
const articles = await client.request(
  readItems('articles', {
    sort: ['status', '-created_at']
  })
);
```

### Pagination

```typescript
// Limit
const articles = await client.request(
  readItems('articles', {
    limit: 10
  })
);

// Offset
const articles = await client.request(
  readItems('articles', {
    limit: 10,
    offset: 20
  })
);

// Page-based pagination
const articles = await client.request(
  readItems('articles', {
    limit: 10,
    page: 3
  })
);
```

### Aggregation

```typescript
// Count
const result = await client.request(
  aggregate('articles', {
    aggregate: {
      count: '*'
    }
  })
);

// Sum
const result = await client.request(
  aggregate('articles', {
    aggregate: {
      sum: ['views']
    }
  })
);

// Average
const result = await client.request(
  aggregate('articles', {
    aggregate: {
      avg: ['rating']
    }
  })
);

// Multiple aggregations
const result = await client.request(
  aggregate('articles', {
    aggregate: {
      count: '*',
      sum: ['views'],
      avg: ['rating']
    },
    groupBy: ['status']
  })
);
```

## Error Handling

### Directus Errors

```typescript
import { isDirectusError } from '@directus/sdk';

try {
  const article = await client.request(
    readItem('articles', 'invalid-id')
  );
} catch (error) {
  if (isDirectusError(error)) {
    console.error('Directus Error:', error.message);
    console.error('Error Code:', error.code);
    console.error('Status:', error.status);
  } else {
    console.error('Unknown Error:', error);
  }
}
```

### Error Types

```typescript
interface DirectusError {
  message: string;
  code: string;
  status: number;
  errors?: Array<{
    message: string;
    extensions: {
      code: string;
      [key: string]: any;
    };
  }>;
}
```

## Advanced Usage

### Custom Fetch Options

```typescript
const client = createDirectus('https://your-directus.com', {
  globals: {
    fetch: customFetch,
  },
});
```

### Request Interceptors

```typescript
// Wrap the request method
const originalRequest = client.request;
client.request = async (options) => {
  console.log('Request:', options);
  const result = await originalRequest(options);
  console.log('Response:', result);
  return result;
};
```

### Retry Logic

```typescript
async function requestWithRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
}

const articles = await requestWithRetry(() =>
  client.request(readItems('articles'))
);
```

## Best Practices

### Client Reuse

Create a single client instance and reuse it:

```typescript
// ✓ Good
const client = createDirectus('https://your-directus.com')
  .with(authentication('session'))
  .with(rest());

export default client;
```

### Type Safety

Always define your schema for type safety:

```typescript
// ✓ Good
interface MySchema {
  articles: Article[];
}

const client = createDirectus<MySchema>('...');
```

### Error Handling

Always handle errors appropriately:

```typescript
// ✓ Good
try {
  const result = await client.request(readItems('articles'));
} catch (error) {
  if (isDirectusError(error)) {
    // Handle Directus errors
  } else {
    // Handle other errors
  }
}
```

### Authentication

Use session authentication for user-facing apps:

```typescript
// ✓ Good for user apps
const client = createDirectus('...')
  .with(authentication('session', {
    autoRefresh: true,
  }));
```

Use static tokens for server-side apps:

```typescript
// ✓ Good for server apps
const client = createDirectus('...')
  .with(staticToken(process.env.DIRECTUS_TOKEN));
```

## Next Steps

- [API Integration](./13-api-integration.md) - Frontend API integration
- [Real-time Features](./16-realtime-features.md) - WebSocket subscriptions
- [Authentication & Security](./06-auth-security.md) - Authentication system
