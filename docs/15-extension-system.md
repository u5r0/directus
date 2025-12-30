# Extension System

This document covers the Directus extension system, which allows you to customize and extend Directus functionality.

## Overview

Directus provides a powerful extension system that allows developers to add custom functionality without modifying the core codebase. Extensions can add new interfaces, displays, layouts, modules, panels, hooks, endpoints, and operations.

## Extension Types

### Frontend Extensions (App)

#### 1. Interfaces
Custom input components for field editing.

**Use Cases:**
- Custom data entry forms
- Specialized input controls
- Third-party integrations

**Example:**
```typescript
import { defineInterface } from '@directus/extensions';

export default defineInterface({
  id: 'custom-input',
  name: 'Custom Input',
  icon: 'text_fields',
  description: 'A custom input interface',
  component: CustomInputComponent,
  options: [
    {
      field: 'placeholder',
      name: 'Placeholder',
      type: 'string',
      meta: {
        interface: 'input',
        width: 'full',
      },
    },
  ],
  types: ['string', 'text'],
});
```

#### 2. Displays
Custom output components for displaying field values.

**Use Cases:**
- Custom data visualization
- Formatted output
- Rich content display

**Example:**
```typescript
import { defineDisplay } from '@directus/extensions';

export default defineDisplay({
  id: 'custom-display',
  name: 'Custom Display',
  icon: 'visibility',
  description: 'A custom display component',
  component: CustomDisplayComponent,
  options: [
    {
      field: 'format',
      name: 'Format',
      type: 'string',
      meta: {
        interface: 'select-dropdown',
        options: {
          choices: [
            { text: 'Short', value: 'short' },
            { text: 'Long', value: 'long' },
          ],
        },
      },
    },
  ],
  types: ['string', 'text'],
});
```

#### 3. Layouts
Custom collection view layouts.

**Use Cases:**
- Alternative data views
- Specialized visualizations
- Custom workflows

**Example:**
```typescript
import { defineLayout } from '@directus/extensions';

export default defineLayout({
  id: 'custom-layout',
  name: 'Custom Layout',
  icon: 'view_module',
  component: CustomLayoutComponent,
  slots: {
    options: OptionsComponent,
    sidebar: SidebarComponent,
    actions: ActionsComponent,
  },
  setup(props, { emit }) {
    // Layout setup logic
    return {
      // State and methods
    };
  },
});
```

#### 4. Modules
Custom admin modules with routes and navigation.

**Use Cases:**
- Custom admin sections
- Specialized tools
- External integrations

**Example:**
```typescript
import { defineModule } from '@directus/extensions';

export default defineModule({
  id: 'custom-module',
  name: 'Custom Module',
  icon: 'extension',
  routes: [
    {
      path: '',
      component: ModuleComponent,
    },
  ],
});
```

#### 5. Panels
Custom dashboard widgets.

**Use Cases:**
- Data visualizations
- Quick stats
- Custom metrics

**Example:**
```typescript
import { definePanel } from '@directus/extensions';

export default definePanel({
  id: 'custom-panel',
  name: 'Custom Panel',
  icon: 'dashboard',
  description: 'A custom dashboard panel',
  component: CustomPanelComponent,
  options: [
    {
      field: 'collection',
      name: 'Collection',
      type: 'string',
      meta: {
        interface: 'system-collection',
        width: 'full',
      },
    },
  ],
  minWidth: 12,
  minHeight: 8,
});
```

### Backend Extensions (API)

#### 6. Hooks
Server-side event listeners and filters.

**Use Cases:**
- Data validation
- Automated workflows
- External integrations
- Custom business logic

**Example:**
```typescript
import { defineHook } from '@directus/extensions';

export default defineHook(({ filter, action, init, schedule, embed }, context) => {
  const { services, database, getSchema, logger } = context;
  
  // Filter hook - modify data before action
  filter('items.create', async (input, meta, context) => {
    // Modify input
    input.created_by_extension = true;
    return input;
  });
  
  // Action hook - execute after action
  action('items.create', async (meta, context) => {
    logger.info(`Item created: ${meta.key}`);
  });
  
  // Init hook - run on startup
  init('app.before', async () => {
    logger.info('Extension initialized');
  });
  
  // Schedule hook - run on cron schedule
  schedule('0 0 * * *', async () => {
    logger.info('Daily task executed');
  });
  
  // Embed hook - inject HTML
  embed('head', '<script>console.log("Extension loaded")</script>');
});
```

#### 7. Endpoints
Custom API endpoints.

**Use Cases:**
- Custom API routes
- External webhooks
- Specialized operations

**Example:**
```typescript
import { defineEndpoint } from '@directus/extensions';

export default defineEndpoint((router, context) => {
  const { services, database, getSchema, logger } = context;
  
  router.get('/', async (req, res) => {
    const schema = await getSchema();
    const service = new services.ItemsService('articles', {
      schema,
      accountability: req.accountability,
    });
    
    const items = await service.readByQuery({
      limit: 10,
    });
    
    res.json(items);
  });
  
  router.post('/', async (req, res) => {
    const service = new services.ItemsService('articles', {
      schema: await getSchema(),
      accountability: req.accountability,
    });
    
    const item = await service.createOne(req.body);
    res.json(item);
  });
});
```

#### 8. Operations
Custom workflow operations for Flows.

**Use Cases:**
- Custom data transformations
- External API calls
- Specialized business logic

**Example:**
```typescript
import { defineOperationApi } from '@directus/extensions';

export default defineOperationApi({
  id: 'custom-operation',
  handler: async (options, context) => {
    const { data, accountability, database, getSchema, logger } = context;
    
    // Operation logic
    const result = await performCustomLogic(options, data);
    
    return result;
  },
});
```

### Hybrid Extensions

#### 9. Bundles
Combine multiple extensions in one package.

**Example:**
```typescript
import { defineBundle } from '@directus/extensions';

export default defineBundle({
  hooks: [
    { config: hookConfig, name: 'custom-hook' },
  ],
  endpoints: [
    { config: endpointConfig, name: 'custom-endpoint' },
  ],
  operations: [
    { config: operationConfig, name: 'custom-operation' },
  ],
});
```

## Extension Manager

### Extension Loading (`api/src/extensions/manager.ts`)

```typescript
export class ExtensionManager {
  private localExtensions: Map<string, Extension> = new Map();
  private registryExtensions: Map<string, Extension> = new Map();
  private moduleExtensions: Map<string, Extension> = new Map();
  
  async initialize(options?: ExtensionManagerOptions): Promise<void> {
    if (!this.isLoaded) {
      await this.load();
    }
  }
  
  private async load(): Promise<void> {
    // Sync extensions from remote locations
    await syncExtensions();
    
    // Get extensions from all sources
    const { local, registry, module } = await getExtensions();
    
    this.localExtensions = local;
    this.registryExtensions = registry;
    this.moduleExtensions = module;
    
    // Register extensions
    await this.registerApiExtensions();
    await this.generateExtensionBundle();
  }
}
```

### Extension Sources

**1. Local Extensions**
Located in `./extensions/` directory:
```
extensions/
├── interfaces/
│   └── custom-input/
├── displays/
│   └── custom-display/
├── hooks/
│   └── custom-hook/
└── endpoints/
    └── custom-endpoint/
```

**2. Registry Extensions**
Installed from npm registry:
```bash
# Install from registry
npm install directus-extension-custom
```

**3. Module Extensions**
Installed as npm packages:
```bash
# Install as dependency
npm install @my-org/directus-extension
```

### Extension Configuration

```bash
# Enable extensions
EXTENSIONS_PATH=./extensions
EXTENSIONS_AUTO_RELOAD=true
EXTENSIONS_MUST_LOAD=false

# Extension locations
EXTENSIONS_LOCATION=local,registry,module
```

## Creating Extensions

### Extension CLI

```bash
# Create new extension
npm init directus-extension

# Choose extension type
? Choose the extension type
  ❯ interface
    display
    layout
    module
    panel
    hook
    endpoint
    operation
    bundle
```

### Extension Structure

```
my-extension/
├── src/
│   ├── index.ts       # Main entry point
│   └── ...            # Extension code
├── package.json       # Extension metadata
└── README.md          # Documentation
```

### Extension package.json

```json
{
  "name": "directus-extension-custom",
  "version": "1.0.0",
  "keywords": ["directus", "directus-extension", "directus-custom-interface"],
  "directus:extension": {
    "type": "interface",
    "path": "dist/index.js",
    "source": "src/index.ts",
    "host": "^11.0.0"
  },
  "scripts": {
    "build": "directus-extension build",
    "dev": "directus-extension build --watch"
  },
  "devDependencies": {
    "@directus/extensions-sdk": "latest"
  }
}
```

## Extension Development

### Development Workflow

1. **Create Extension**
```bash
npm init directus-extension
cd my-extension
```

2. **Develop**
```bash
npm run dev
```

3. **Link to Directus**
```bash
# In extension directory
npm link

# In Directus directory
npm link directus-extension-custom
```

4. **Build**
```bash
npm run build
```

5. **Publish**
```bash
npm publish
```

### Hot Reload

Enable auto-reload during development:

```bash
EXTENSIONS_AUTO_RELOAD=true
```

The extension manager watches for changes and reloads automatically.

### Extension Context

Extensions receive a context object with utilities:

```typescript
interface ExtensionContext {
  services: typeof services;
  database: Knex;
  getSchema: () => Promise<SchemaOverview>;
  logger: Logger;
  env: Record<string, any>;
  emitter: Emitter;
}
```

## Extension Sandboxing

### Sandbox Mode

For security, extensions can run in isolated environments:

```json
{
  "directus:extension": {
    "type": "hook",
    "sandbox": {
      "enabled": true,
      "requestedScopes": {
        "log": {},
        "sleep": {}
      }
    }
  }
}
```

### Available Scopes

```typescript
interface SandboxScopes {
  log: {};
  sleep: {};
  request: {
    urls: string[];
    methods: string[];
  };
}
```

### Sandbox Implementation

```typescript
// Extensions run in isolated-vm
const isolate = new ivm.Isolate({
  memoryLimit: env['EXTENSIONS_SANDBOX_MEMORY'],
});

const context = await isolate.createContext();
const module = await isolate.compileModule(extensionCode);

await module.instantiate(context, (specifier) => {
  if (specifier !== 'directus:api') {
    throw new Error('Only directus:api imports allowed');
  }
  return sdkModule;
});
```

## Extension Registry

### Publishing Extensions

1. **Build Extension**
```bash
npm run build
```

2. **Test Extension**
```bash
npm pack
npm install ./directus-extension-custom-1.0.0.tgz
```

3. **Publish to npm**
```bash
npm publish
```

### Extension Metadata

```json
{
  "name": "directus-extension-custom",
  "description": "A custom Directus extension",
  "keywords": [
    "directus",
    "directus-extension",
    "directus-custom-interface"
  ],
  "directus:extension": {
    "type": "interface",
    "path": "dist/index.js",
    "source": "src/index.ts",
    "host": "^11.0.0"
  }
}
```

## Extension Examples

### Example 1: Custom Validation Hook

```typescript
import { defineHook } from '@directus/extensions';

export default defineHook(({ filter }, { logger }) => {
  filter('items.create', async (input, meta) => {
    if (meta.collection === 'articles') {
      // Validate title length
      if (input.title && input.title.length < 10) {
        throw new Error('Title must be at least 10 characters');
      }
      
      // Auto-generate slug
      if (!input.slug && input.title) {
        input.slug = input.title
          .toLowerCase()
          .replace(/[^a-z0-9]+/g, '-');
      }
    }
    
    return input;
  });
});
```

### Example 2: Custom API Endpoint

```typescript
import { defineEndpoint } from '@directus/extensions';

export default defineEndpoint((router, { services, getSchema }) => {
  router.get('/stats', async (req, res) => {
    const schema = await getSchema();
    const service = new services.ItemsService('articles', {
      schema,
      accountability: req.accountability,
    });
    
    const published = await service.readByQuery({
      filter: { status: { _eq: 'published' } },
      aggregate: { count: '*' },
    });
    
    const draft = await service.readByQuery({
      filter: { status: { _eq: 'draft' } },
      aggregate: { count: '*' },
    });
    
    res.json({
      published: published[0].count,
      draft: draft[0].count,
    });
  });
});
```

### Example 3: Custom Operation

```typescript
import { defineOperationApi } from '@directus/extensions';
import axios from 'axios';

export default defineOperationApi({
  id: 'send-slack-message',
  handler: async (options, { logger }) => {
    const { webhook_url, message } = options;
    
    try {
      await axios.post(webhook_url, {
        text: message,
      });
      
      logger.info('Slack message sent successfully');
      return { success: true };
    } catch (error) {
      logger.error('Failed to send Slack message:', error);
      throw error;
    }
  },
});
```

## Extension Best Practices

### Performance

- Keep extensions lightweight
- Avoid blocking operations
- Use async/await properly
- Cache expensive operations
- Implement proper error handling

### Security

- Validate all inputs
- Sanitize user data
- Use accountability context
- Implement proper permissions
- Avoid exposing sensitive data

### Maintainability

- Follow TypeScript best practices
- Document your code
- Write tests
- Version your extensions
- Provide clear README

### Compatibility

- Specify host version requirements
- Test with multiple Directus versions
- Handle breaking changes gracefully
- Provide migration guides

## Extension Debugging

### Logging

```typescript
export default defineHook(({ action }, { logger }) => {
  action('items.create', async (meta) => {
    logger.info('Item created:', meta.key);
    logger.debug('Full metadata:', meta);
    logger.warn('Warning message');
    logger.error('Error message');
  });
});
```

### Error Handling

```typescript
export default defineEndpoint((router, { logger }) => {
  router.get('/', async (req, res, next) => {
    try {
      // Endpoint logic
    } catch (error) {
      logger.error('Endpoint error:', error);
      next(error);
    }
  });
});
```

### Development Tools

```bash
# Watch mode
npm run dev

# Enable debug logging
LOG_LEVEL=debug

# Auto-reload extensions
EXTENSIONS_AUTO_RELOAD=true
```

## Next Steps

- [Backend Architecture](./04-backend-architecture.md) - API structure
- [Frontend Overview](./07-frontend-overview.md) - App architecture
- [Workflow Engine](./18-workflow-engine.md) - Custom operations
