# Workflow Engine

This document covers the workflow automation system in Directus, which enables complex business logic through flows and operations.

## Overview

Directus Flows is a powerful workflow automation engine that allows you to create custom business logic without writing code. Flows consist of triggers and operations that execute in sequence.

## Flow Architecture

### Flow Manager (`api/src/flows.ts`)

The FlowManager handles flow registration, execution, and lifecycle:

```typescript
class FlowManager {
  private operations: Map<string, OperationHandler> = new Map();
  private triggerHandlers: TriggerHandler[] = [];
  private operationFlowHandlers: Record<string, any> = {};
  private webhookFlowHandlers: Record<string, any> = {};
  
  async initialize(): Promise<void> {
    await this.load();
  }
  
  async reload(): Promise<void> {
    await this.unload();
    await this.load();
  }
  
  private async load(): Promise<void> {
    // Load active flows from database
    const flows = await flowsService.readByQuery({
      filter: { status: { _eq: 'active' } },
      fields: ['*', 'operations.*'],
      limit: -1,
    });
    
    // Register flows by trigger type
    for (const flow of flows) {
      if (flow.trigger === 'event') {
        this.registerEventFlow(flow);
      } else if (flow.trigger === 'schedule') {
        this.registerScheduleFlow(flow);
      } else if (flow.trigger === 'webhook') {
        this.registerWebhookFlow(flow);
      } else if (flow.trigger === 'manual') {
        this.registerManualFlow(flow);
      }
    }
  }
}
```

### Flow Structure

```typescript
interface Flow {
  id: string;
  name: string;
  icon: string;
  color: string;
  description: string;
  status: 'active' | 'inactive';
  trigger: 'event' | 'schedule' | 'webhook' | 'operation' | 'manual';
  accountability: 'all' | 'activity' | null;
  options: Record<string, any>;
  operation: Operation | null;
  operations: Operation[];
  date_created: string;
  user_created: string;
}

interface Operation {
  id: string;
  name: string;
  key: string;
  type: string;
  position_x: number;
  position_y: number;
  options: Record<string, any>;
  resolve: Operation | null;
  reject: Operation | null;
}
```

## Trigger Types

### 1. Event Triggers

Execute flows when specific events occur in Directus.

**Event Types:**
- `items.create` - Item created
- `items.update` - Item updated
- `items.delete` - Item deleted
- `auth.login` - User logged in
- `auth.logout` - User logged out
- `files.upload` - File uploaded
- `users.create` - User created

**Configuration:**
```typescript
{
  trigger: 'event',
  options: {
    type: 'action',  // or 'filter'
    scope: ['items.create', 'items.update'],
    collections: ['articles', 'pages']
  }
}
```

**Action vs Filter:**

**Action Flows** - Execute after the event:
```typescript
emitter.onAction('items.create', async (meta, context) => {
  await executeFlow(flow, meta, context);
});
```

**Filter Flows** - Modify data before the event:
```typescript
emitter.onFilter('items.create', async (payload, meta, context) => {
  const result = await executeFlow(flow, { payload, ...meta }, context);
  return result;  // Modified payload
});
```

### 2. Schedule Triggers

Execute flows on a schedule using cron expressions.

**Configuration:**
```typescript
{
  trigger: 'schedule',
  options: {
    cron: '0 0 * * *'  // Daily at midnight
  }
}
```

**Cron Examples:**
```
0 * * * *     # Every hour
0 0 * * *     # Daily at midnight
0 0 * * 0     # Weekly on Sunday
0 0 1 * *     # Monthly on the 1st
*/5 * * * *   # Every 5 minutes
```

**Implementation:**
```typescript
if (validateCron(flow.options['cron'])) {
  const job = scheduleSynchronizedJob(flow.id, flow.options['cron'], async () => {
    await this.executeFlow(flow);
  });
  
  this.triggerHandlers.push({ id: flow.id, events: [{ type: 'schedule', job }] });
}
```

### 3. Webhook Triggers

Execute flows via HTTP requests.

**Configuration:**
```typescript
{
  trigger: 'webhook',
  options: {
    method: 'GET',  // or POST, PUT, DELETE
    async: false,   // Wait for completion
    return: '$last', // Return last operation result
    cacheEnabled: true  // Cache GET responses
  }
}
```

**Webhook URL:**
```
GET  /flows/trigger/{flow-id}
POST /flows/trigger/{flow-id}
```

**Request/Response:**
```typescript
// Request
POST /flows/trigger/abc-123
Content-Type: application/json

{
  "data": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}

// Response
{
  "result": {
    "id": "456",
    "status": "created"
  }
}
```

### 4. Manual Triggers

Execute flows manually from the UI.

**Configuration:**
```typescript
{
  trigger: 'manual',
  options: {
    collections: ['articles'],
    requireSelection: true,
    location: 'item',  // or 'collection'
    return: '$last'
  }
}
```

**Execution:**
```typescript
POST /flows/trigger/{flow-id}
Content-Type: application/json

{
  "collection": "articles",
  "keys": ["123", "456"]
}
```

### 5. Operation Triggers

Execute flows from other flows.

**Configuration:**
```typescript
{
  trigger: 'operation',
  options: {}
}
```

**Usage in Operations:**
```typescript
// Trigger Flow operation
{
  type: 'trigger',
  options: {
    flow: 'child-flow-id',
    payload: {
      data: '{{$trigger.data}}'
    }
  }
}
```

## Operations

### Built-in Operations

#### Condition
Evaluate conditions and branch execution.

```typescript
{
  type: 'condition',
  options: {
    filter: {
      status: { _eq: 'published' },
      author: { _eq: '$CURRENT_USER' }
    }
  },
  resolve: nextOperation,  // If condition passes
  reject: errorOperation   // If condition fails
}
```

#### Item Create
Create new items in a collection.

```typescript
{
  type: 'item-create',
  options: {
    collection: 'articles',
    permissions: '$trigger',  // or '$public', '$full', role UUID
    emitEvents: true,
    payload: {
      title: '{{$trigger.title}}',
      content: '{{$trigger.content}}',
      author: '{{$accountability.user}}',
      status: 'draft'
    }
  }
}
```

#### Item Read
Read items from a collection.

```typescript
{
  type: 'item-read',
  options: {
    collection: 'articles',
    permissions: '$trigger',
    emitEvents: false,
    key: '{{$trigger.id}}',
    query: {
      fields: ['*', 'author.*'],
      filter: {
        status: { _eq: 'published' }
      }
    }
  }
}
```

#### Item Update
Update existing items.

```typescript
{
  type: 'item-update',
  options: {
    collection: 'articles',
    permissions: '$trigger',
    emitEvents: true,
    key: '{{$trigger.id}}',
    payload: {
      status: 'published',
      published_date: '{{$NOW}}'
    }
  }
}
```

#### Item Delete
Delete items from a collection.

```typescript
{
  type: 'item-delete',
  options: {
    collection: 'articles',
    permissions: '$trigger',
    emitEvents: true,
    key: '{{$trigger.id}}'
  }
}
```

#### Transform
Transform data using JavaScript.

```typescript
{
  type: 'transform',
  options: {
    json: JSON.stringify({
      fullName: '{{$trigger.first_name}} {{$trigger.last_name}}',
      email: '{{$trigger.email}}',
      timestamp: '{{$NOW}}'
    })
  }
}
```

#### Log
Log messages for debugging.

```typescript
{
  type: 'log',
  options: {
    message: 'Processing article: {{$trigger.title}}'
  }
}
```

#### Mail
Send emails.

```typescript
{
  type: 'mail',
  options: {
    to: '{{$trigger.email}}',
    subject: 'Welcome to our platform!',
    body: 'Hello {{$trigger.name}}, welcome!',
    type: 'wysiwyg'  // or 'markdown'
  }
}
```

#### Notification
Create in-app notifications.

```typescript
{
  type: 'notification',
  options: {
    recipient: '{{$trigger.user}}',
    subject: 'New comment on your article',
    message: '{{$trigger.comment}}',
    collection: 'articles',
    item: '{{$trigger.article_id}}'
  }
}
```

#### Request
Make HTTP requests to external APIs.

```typescript
{
  type: 'request',
  options: {
    method: 'POST',
    url: 'https://api.example.com/webhook',
    headers: [
      {
        header: 'Authorization',
        value: 'Bearer {{$env.API_TOKEN}}'
      }
    ],
    body: JSON.stringify({
      event: 'article.published',
      data: '{{$trigger}}'
    })
  }
}
```

#### Sleep
Pause execution for a duration.

```typescript
{
  type: 'sleep',
  options: {
    milliseconds: 5000  // 5 seconds
  }
}
```

#### Exec
Execute shell commands (use with caution).

```typescript
{
  type: 'exec',
  options: {
    command: 'node scripts/process-data.js {{$trigger.id}}'
  }
}
```

#### Throw Error
Throw an error to stop execution.

```typescript
{
  type: 'throw-error',
  options: {
    message: 'Invalid data: {{$trigger.error}}'
  }
}
```

#### JWT Sign
Generate JWT tokens.

```typescript
{
  type: 'json-web-token',
  options: {
    payload: {
      user_id: '{{$trigger.id}}',
      email: '{{$trigger.email}}'
    },
    secret: '{{$env.JWT_SECRET}}',
    expiresIn: '1h'
  }
}
```

## Flow Execution

### Execution Context

Flows execute with a context containing:

```typescript
const keyedData: Record<string, unknown> = {
  $trigger: data,           // Trigger data
  $last: data,              // Last operation result
  $accountability: context?.accountability,  // User context
  $env: this.envs,          // Environment variables
  [operationKey]: result    // Each operation's result
};
```

### Variable Interpolation

Use mustache syntax to reference data:

```typescript
// Access trigger data
'{{$trigger.title}}'
'{{$trigger.author.name}}'

// Access previous operation results
'{{operation_key.result}}'

// Access accountability
'{{$accountability.user}}'
'{{$accountability.role}}'

// Access environment variables
'{{$env.API_KEY}}'

// Access last operation result
'{{$last.id}}'
```

### Operation Chaining

Operations execute in sequence:

```typescript
async function executeFlow(flow: Flow, data: unknown, context: Record<string, unknown>) {
  const keyedData = {
    $trigger: data,
    $last: data,
    $accountability: context?.accountability,
    $env: this.envs,
  };
  
  let nextOperation = flow.operation;
  
  while (nextOperation !== null) {
    const { successor, data, status } = await this.executeOperation(
      nextOperation,
      keyedData,
      context
    );
    
    // Store operation result
    keyedData[nextOperation.key] = data;
    keyedData.$last = data;
    
    // Move to next operation
    nextOperation = successor;
  }
  
  // Return result based on flow options
  if (flow.options['return'] === '$all') {
    return keyedData;
  } else if (flow.options['return']) {
    return get(keyedData, flow.options['return']);
  }
  
  return undefined;
}
```

### Error Handling

Operations can have resolve and reject paths:

```typescript
try {
  const result = await handler(options, context);
  return {
    successor: operation.resolve,
    status: 'resolve',
    data: result
  };
} catch (error) {
  return {
    successor: operation.reject,
    status: 'reject',
    data: error
  };
}
```

## Flow Examples

### Example 1: Auto-Publish Articles

Automatically publish articles when approved:

```typescript
{
  name: 'Auto-Publish Articles',
  trigger: 'event',
  options: {
    type: 'action',
    scope: ['items.update'],
    collections: ['articles']
  },
  operations: [
    {
      key: 'check_status',
      type: 'condition',
      options: {
        filter: {
          status: { _eq: 'approved' }
        }
      },
      resolve: 'publish_article',
      reject: null
    },
    {
      key: 'publish_article',
      type: 'item-update',
      options: {
        collection: 'articles',
        key: '{{$trigger.keys[0]}}',
        payload: {
          status: 'published',
          published_date: '{{$NOW}}'
        }
      },
      resolve: 'send_notification',
      reject: null
    },
    {
      key: 'send_notification',
      type: 'notification',
      options: {
        recipient: '{{$trigger.payload.author}}',
        subject: 'Article Published',
        message: 'Your article "{{$trigger.payload.title}}" has been published!'
      },
      resolve: null,
      reject: null
    }
  ]
}
```

### Example 2: Daily Report

Send daily report via email:

```typescript
{
  name: 'Daily Report',
  trigger: 'schedule',
  options: {
    cron: '0 9 * * *'  // 9 AM daily
  },
  operations: [
    {
      key: 'fetch_articles',
      type: 'item-read',
      options: {
        collection: 'articles',
        query: {
          filter: {
            created_at: {
              _gte: '$YESTERDAY'
            }
          },
          aggregate: {
            count: '*'
          }
        }
      },
      resolve: 'send_email',
      reject: null
    },
    {
      key: 'send_email',
      type: 'mail',
      options: {
        to: 'admin@example.com',
        subject: 'Daily Report - {{$NOW}}',
        body: 'Articles created yesterday: {{fetch_articles.count}}'
      },
      resolve: null,
      reject: null
    }
  ]
}
```

### Example 3: Webhook Integration

Sync data to external service:

```typescript
{
  name: 'Sync to External Service',
  trigger: 'event',
  options: {
    type: 'action',
    scope: ['items.create', 'items.update'],
    collections: ['products']
  },
  operations: [
    {
      key: 'transform_data',
      type: 'transform',
      options: {
        json: JSON.stringify({
          id: '{{$trigger.key}}',
          name: '{{$trigger.payload.name}}',
          price: '{{$trigger.payload.price}}',
          timestamp: '{{$NOW}}'
        })
      },
      resolve: 'send_webhook',
      reject: 'log_error'
    },
    {
      key: 'send_webhook',
      type: 'request',
      options: {
        method: 'POST',
        url: 'https://api.example.com/products',
        headers: [
          {
            header: 'Authorization',
            value: 'Bearer {{$env.API_TOKEN}}'
          }
        ],
        body: '{{transform_data}}'
      },
      resolve: null,
      reject: 'log_error'
    },
    {
      key: 'log_error',
      type: 'log',
      options: {
        message: 'Failed to sync product: {{$last}}'
      },
      resolve: null,
      reject: null
    }
  ]
}
```

## Creating Custom Operations

### Operation Definition

```typescript
import { defineOperationApi } from '@directus/extensions';

export default defineOperationApi({
  id: 'custom-operation',
  
  handler: async (options, { data, accountability, database, getSchema, logger }) => {
    // Operation logic
    const schema = await getSchema({ database });
    
    // Access trigger data
    const triggerData = data.$trigger;
    
    // Perform operation
    const result = await performCustomLogic(options, triggerData);
    
    // Return result
    return result;
  },
});
```

### Operation Context

```typescript
interface OperationContext {
  services: typeof services;
  env: Record<string, any>;
  database: Knex;
  logger: Logger;
  getSchema: () => Promise<SchemaOverview>;
  data: Record<string, unknown>;
  accountability: Accountability | null;
}
```

## Flow Accountability

Track flow execution:

```typescript
{
  accountability: 'all',  // Track execution and data
  // or 'activity'  // Track execution only
  // or null        // No tracking
}
```

When enabled, creates activity log entries:

```typescript
await activityService.createOne({
  action: Action.RUN,
  user: accountability?.user,
  collection: 'directus_flows',
  item: flow.id,
});

// With 'all' accountability, also creates revision
await revisionsService.createOne({
  activity: activity,
  collection: 'directus_flows',
  item: flow.id,
  data: {
    steps: executedSteps,
    data: flowData
  }
});
```

## Best Practices

### Performance

- Keep flows simple and focused
- Avoid long-running operations
- Use async flows for webhooks when possible
- Implement proper error handling
- Monitor flow execution times

### Security

- Validate input data
- Use appropriate permissions for operations
- Sanitize data before external requests
- Limit environment variable access
- Audit flow execution logs

### Maintainability

- Use descriptive names for flows and operations
- Document complex logic
- Test flows thoroughly
- Version control flow configurations
- Monitor flow failures

## Next Steps

- [Backend Architecture](./04-backend-architecture.md) - API structure
- [Authentication & Security](./06-auth-security.md) - Permission system
- [Extension System](./15-extension-system.md) - Custom operations
