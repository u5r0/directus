# Backend Architecture

The Directus API is built on Express.js with a layered architecture providing REST and GraphQL APIs, real-time capabilities, and extensive customization options.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Application Bootstrap](#application-bootstrap)
- [Controller Layer](#controller-layer)
- [Service Layer](#service-layer)
- [Middleware Stack](#middleware-stack)
- [Authentication System](#authentication-system)
- [Extension System](#extension-system)
- [Event System](#event-system)

## Architecture Overview

### Layered Architecture
```
┌─────────────────────────────────────┐
│         HTTP/WebSocket              │
├─────────────────────────────────────┤
│         Middleware Stack            │
│  (CORS, Auth, Cache, Rate Limit)    │
├─────────────────────────────────────┤
│         Controller Layer            │
│    (Route Handlers, Validation)     │
├─────────────────────────────────────┤
│          Service Layer              │
│   (Business Logic, Permissions)     │
├─────────────────────────────────────┤
│         Database Layer              │
│    (Knex, Schema Inspector)         │
├─────────────────────────────────────┤
│         Database (SQL)              │
└─────────────────────────────────────┘
```

### Technology Stack
- **Express.js 4.21.2** - Web framework
- **Knex 3.1.0** - SQL query builder
- **GraphQL 16.12.0** - GraphQL API
- **WebSocket (ws 8.18.3)** - Real-time communication
- **Pino 9.7.0** - Structured logging
- **Helmet 8.1.0** - Security headers
- **Terminus 4.12.1** - Graceful shutdown

## Application Bootstrap

### Server Initialization
```typescript
// api/src/server.ts
export async function createServer(): Promise<http.Server> {
  const server = http.createServer(await createApp());
  
  // Apply server configuration from environment
  Object.assign(server, getConfigFromEnv('SERVER_'));
  
  // Request metrics tracking
  server.on('request', function (req, res) {
    const startTime = process.hrtime();
    
    const complete = once(function (finished: boolean) {
      const elapsedTime = process.hrtime(startTime);
      const elapsedMilliseconds = (elapsedTime[0] * 1e9 + elapsedTime[1]) / 1e6;
      
      // Emit response event for logging/metrics
      emitter.emitAction('response', {
        finished,
        request: { method: req.method, url: req.url },
        response: { status: res.statusCode },
        duration: elapsedMilliseconds.toFixed(),
      });
    });
    
    res.once('finish', complete.bind(null, true));
    res.once('close', complete.bind(null, false));
  });
  
  // WebSocket support
  if (env['WEBSOCKETS_ENABLED']) {
    createSubscriptionController(server);
    createWebSocketController(server);
    createLogsController(server);
  }
  
  // Graceful shutdown
  createTerminus(server, {
    timeout: env['SERVER_SHUTDOWN_TIMEOUT'],
    signals: ['SIGINT', 'SIGTERM', 'SIGHUP'],
    beforeShutdown,
    onSignal,
    onShutdown,
  });
  
  return server;
}
```

### App Configuration
```typescript
// api/src/app.ts
export default async function createApp(): Promise<express.Application> {
  // Validate database connection
  await validateDatabaseConnection();
  
  // Check installation status
  if ((await isInstalled()) === false) {
    logger.error(`Database doesn't have Directus tables installed.`);
    process.exit(1);
  }
  
  // Validate migrations
  if ((await validateMigrations()) === false) {
    logger.warn(`Database migrations have not all been run`);
  }
  
  // Initialize extensions and flows
  const extensionManager = getExtensionManager();
  const flowManager = getFlowManager();
  
  await extensionManager.initialize();
  await flowManager.initialize();
  
  const app = express();
  
  // Basic configuration
  app.disable('x-powered-by');
  app.set('trust proxy', env['IP_TRUST_PROXY']);
  app.set('query parser', (str: string) => 
    qs.parse(str, { depth: env['QUERYSTRING_MAX_PARSE_DEPTH'] })
  );
  
  // Pressure limiter (optional)
  if (env['PRESSURE_LIMITER_ENABLED']) {
    app.use(handlePressure({
      sampleInterval: env['PRESSURE_LIMITER_SAMPLE_INTERVAL'],
      maxEventLoopUtilization: env['PRESSURE_LIMITER_MAX_EVENT_LOOP_UTILIZATION'],
      maxMemoryRss: env['PRESSURE_LIMITER_MAX_MEMORY_RSS'],
    }));
  }
  
  // Security headers
  app.use(helmet.contentSecurityPolicy({
    useDefaults: true,
    directives: {
      scriptSrc: ["'self'", "'unsafe-eval'"], // Required for extensions
      upgradeInsecureRequests: null,
    },
  }));
  
  // Middleware stack
  app.use(createExpressLogger());
  app.use(cors);
  app.use(express.json({ limit: env['MAX_PAYLOAD_SIZE'] }));
  app.use(cookieParser());
  app.use(extractToken);
  
  // Rate limiting
  if (env['RATE_LIMITER_ENABLED']) {
    app.use(rateLimiter);
  }
  
  // Authentication and schema
  app.use(authenticate);
  app.use(schema);
  app.use(sanitizeQuery);
  app.use(cache);
  
  // Register routes
  registerRoutes(app);
  
  // Error handling
  app.use(notFoundHandler);
  app.use(errorHandler);
  
  // Start scheduled tasks
  await retentionSchedule();
  await telemetrySchedule();
  await metricsSchedule();
  
  return app;
}
```

## Controller Layer

### Controller Structure
Controllers handle HTTP requests, validate input, and delegate to services:

```typescript
// api/src/controllers/collections.ts
const router = express.Router();

// GET /collections - List all collections
router.get(
  '/',
  asyncHandler(async (req, res) => {
    const service = new CollectionsService({
      accountability: req.accountability,
      schema: req.schema,
    });
    
    const collections = await service.readByQuery();
    
    res.json({ data: collections });
  })
);

// GET /collections/:collection - Get single collection
router.get(
  '/:collection',
  asyncHandler(async (req, res) => {
    const service = new CollectionsService({
      accountability: req.accountability,
      schema: req.schema,
    });
    
    const collection = await service.readOne(req.params.collection);
    
    res.json({ data: collection });
  })
);

// POST /collections - Create collection
router.post(
  '/',
  asyncHandler(async (req, res) => {
    const service = new CollectionsService({
      accountability: req.accountability,
      schema: req.schema,
    });
    
    const collection = await service.createOne(req.body);
    
    res.json({ data: collection });
  })
);

// PATCH /collections/:collection - Update collection
router.patch(
  '/:collection',
  asyncHandler(async (req, res) => {
    const service = new CollectionsService({
      accountability: req.accountability,
      schema: req.schema,
    });
    
    await service.updateOne(req.params.collection, req.body);
    
    const collection = await service.readOne(req.params.collection);
    
    res.json({ data: collection });
  })
);

// DELETE /collections/:collection - Delete collection
router.delete(
  '/:collection',
  asyncHandler(async (req, res) => {
    const service = new CollectionsService({
      accountability: req.accountability,
      schema: req.schema,
    });
    
    await service.deleteOne(req.params.collection);
    
    res.status(204).end();
  })
);

export default router;
```

### Available Controllers (30+)
```
api/src/controllers/
├── access.ts              # Access control
├── activity.ts            # Activity logs
├── assets.ts              # Asset serving
├── auth.ts                # Authentication
├── collections.ts         # Collection management
├── comments.ts            # Comments
├── dashboards.ts          # Dashboard configuration
├── extensions.ts          # Extension management
├── fields.ts              # Field management
├── files.ts               # File management
├── flows.ts               # Workflow management
├── folders.ts             # Folder management
├── graphql.ts             # GraphQL endpoint
├── items.ts               # Item CRUD operations
├── mcp.ts                 # Model Context Protocol
├── metrics.ts             # Prometheus metrics
├── notifications.ts       # User notifications
├── operations.ts          # Workflow operations
├── panels.ts              # Dashboard panels
├── permissions.ts         # Permission management
├── policies.ts            # Policy management
├── presets.ts             # User presets
├── relations.ts           # Relationship management
├── revisions.ts           # Content revisions
├── roles.ts               # Role management
├── schema.ts              # Schema introspection
├── server.ts              # Server info
├── settings.ts            # System settings
├── shares.ts              # Public shares
├── translations.ts        # Translations
├── tus.ts                 # TUS file upload
├── users.ts               # User management
├── utils.ts               # Utility endpoints
├── versions.ts            # Content versioning
└── webhooks.ts            # Webhook management
```

## Service Layer

### Service Architecture
Services contain business logic and handle permissions:

```typescript
// api/src/services/collections.ts
export class CollectionsService {
  accountability: Accountability | null;
  knex: Knex;
  schema: SchemaOverview;
  
  constructor(options: AbstractServiceOptions) {
    this.accountability = options.accountability || null;
    this.knex = options.knex || getDatabase();
    this.schema = options.schema;
  }
  
  async readByQuery(): Promise<Collection[]> {
    const collectionsService = new MetaService({
      accountability: this.accountability,
      schema: this.schema,
    });
    
    const collections = await collectionsService.getCollections();
    
    // Filter based on permissions
    if (this.accountability && this.accountability.admin !== true) {
      const permissionsService = new PermissionsService({
        accountability: this.accountability,
        schema: this.schema,
      });
      
      const permissions = await permissionsService.readByQuery();
      const allowedCollections = new Set(permissions.map(p => p.collection));
      
      return collections.filter(c => allowedCollections.has(c.collection));
    }
    
    return collections;
  }
  
  async readOne(collection: string): Promise<Collection> {
    const collections = await this.readByQuery();
    const result = collections.find(c => c.collection === collection);
    
    if (!result) {
      throw new ForbiddenError();
    }
    
    return result;
  }
  
  async createOne(data: Partial<Collection>): Promise<string> {
    // Check admin permission
    if (this.accountability && this.accountability.admin !== true) {
      throw new ForbiddenError();
    }
    
    // Validate collection name
    if (!data.collection) {
      throw new InvalidPayloadError({ reason: 'Collection name is required' });
    }
    
    // Create collection in database
    await this.knex.schema.createTable(data.collection, (table) => {
      table.increments('id').primary();
      // Add other default fields
    });
    
    // Create metadata
    await this.knex('directus_collections').insert({
      collection: data.collection,
      meta: data.meta ? JSON.stringify(data.meta) : null,
    });
    
    return data.collection;
  }
  
  async updateOne(collection: string, data: Partial<Collection>): Promise<void> {
    // Check admin permission
    if (this.accountability && this.accountability.admin !== true) {
      throw new ForbiddenError();
    }
    
    // Update metadata
    await this.knex('directus_collections')
      .where({ collection })
      .update({
        meta: data.meta ? JSON.stringify(data.meta) : null,
      });
  }
  
  async deleteOne(collection: string): Promise<void> {
    // Check admin permission
    if (this.accountability && this.accountability.admin !== true) {
      throw new ForbiddenError();
    }
    
    // Delete collection
    await this.knex.schema.dropTable(collection);
    
    // Delete metadata
    await this.knex('directus_collections')
      .where({ collection })
      .delete();
  }
}
```

### Available Services (30+)
```
api/src/services/
├── access.ts              # Access control service
├── activity.ts            # Activity logging
├── assets.ts              # Asset processing
├── authentication.ts      # Authentication logic
├── collections.ts         # Collection management
├── comments.ts            # Comment management
├── dashboards.ts          # Dashboard service
├── extensions.ts          # Extension management
├── fields.ts              # Field management
├── files.ts               # File management
├── flows.ts               # Workflow service
├── folders.ts             # Folder management
├── graphql/               # GraphQL service
├── import-export.ts       # Data import/export
├── items.ts               # Item CRUD service
├── mail/                  # Email service
├── meta.ts                # Metadata service
├── notifications.ts       # Notification service
├── operations.ts          # Operation service
├── panels.ts              # Panel service
├── payload.ts             # Payload processing
├── permissions.ts         # Permission service
├── policies.ts            # Policy service
├── presets.ts             # Preset service
├── relations.ts           # Relationship service
├── revisions.ts           # Revision service
├── roles.ts               # Role service
├── schema.ts              # Schema service
├── server.ts              # Server info service
├── settings.ts            # Settings service
├── shares.ts              # Share service
├── specifications.ts      # OpenAPI specs
├── tfa.ts                 # Two-factor auth
├── translations.ts        # Translation service
├── users.ts               # User service
├── utils.ts               # Utility service
├── versions.ts            # Version service
├── webhooks.ts            # Webhook service
└── websocket.ts           # WebSocket service
```

## Middleware Stack

### Middleware Order
```typescript
// Middleware execution order
app.use(createExpressLogger());        // 1. Logging
app.use(cors);                         // 2. CORS headers
app.use(express.json());               // 3. JSON parsing
app.use(cookieParser());               // 4. Cookie parsing
app.use(extractToken);                 // 5. Extract auth token
app.use(rateLimiter);                  // 6. Rate limiting
app.use(authenticate);                 // 7. Authentication
app.use(schema);                       // 8. Schema loading
app.use(sanitizeQuery);                // 9. Query sanitization
app.use(cache);                        // 10. Response caching
```

### Key Middleware

#### Authentication Middleware
```typescript
// api/src/middleware/authenticate.ts
export default async function authenticate(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  // Extract token from request
  const token = req.token;
  
  if (!token) {
    req.accountability = {
      user: null,
      role: null,
      admin: false,
      app: false,
    };
    return next();
  }
  
  try {
    // Verify and decode token
    const payload = verifyAccessToken(token);
    
    // Load user and role
    const user = await getUserFromToken(payload);
    const role = await getRoleForUser(user.id);
    
    req.accountability = {
      user: user.id,
      role: role.id,
      admin: role.admin_access,
      app: role.app_access,
      ip: getIPFromReq(req),
    };
    
    next();
  } catch (error) {
    throw new InvalidCredentialsError();
  }
}
```

#### Schema Middleware
```typescript
// api/src/middleware/schema.ts
export default async function schema(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  // Load database schema
  const schemaInspector = getSchemaInspector();
  const schema = await schemaInspector.overview();
  
  // Attach to request
  req.schema = schema;
  
  next();
}
```

#### Cache Middleware
```typescript
// api/src/middleware/cache.ts
export default function cache(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Only cache GET requests
  if (req.method !== 'GET') {
    return next();
  }
  
  // Check cache
  const cacheKey = getCacheKey(req);
  const cached = cacheStore.get(cacheKey);
  
  if (cached) {
    res.setHeader('X-Cache', 'HIT');
    return res.json(cached);
  }
  
  // Store original json method
  const originalJson = res.json.bind(res);
  
  // Override json method to cache response
  res.json = function (data: any) {
    cacheStore.set(cacheKey, data);
    res.setHeader('X-Cache', 'MISS');
    return originalJson(data);
  };
  
  next();
}
```

## Authentication System

### Authentication Providers
```
api/src/auth/drivers/
├── local.ts               # Email/password auth
├── oauth2.ts              # OAuth 2.0 providers
├── openid.ts              # OpenID Connect
├── ldap.ts                # LDAP authentication
└── saml.ts                # SAML authentication
```

### Token Management
```typescript
// JWT token generation
export function generateAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, env['SECRET'], {
    expiresIn: env['ACCESS_TOKEN_TTL'],
    issuer: 'directus',
  });
}

// Token refresh
export async function refreshToken(refreshToken: string): Promise<Tokens> {
  const payload = verifyRefreshToken(refreshToken);
  
  const accessToken = generateAccessToken({
    id: payload.id,
    role: payload.role,
  });
  
  return {
    access_token: accessToken,
    refresh_token: refreshToken,
    expires: env['ACCESS_TOKEN_TTL'],
  };
}
```

## Extension System

### Extension Manager
```typescript
// api/src/extensions/manager.ts
export class ExtensionManager {
  private extensions: Map<string, Extension> = new Map();
  
  async initialize(): Promise<void> {
    // Load extensions from extensions directory
    const extensionsPath = getExtensionsPath();
    const extensionDirs = await fs.readdir(extensionsPath);
    
    for (const dir of extensionDirs) {
      const extension = await this.loadExtension(path.join(extensionsPath, dir));
      this.extensions.set(extension.name, extension);
    }
  }
  
  async loadExtension(extensionPath: string): Promise<Extension> {
    const manifest = await fs.readJSON(path.join(extensionPath, 'package.json'));
    const entrypoint = await import(path.join(extensionPath, manifest.main));
    
    return {
      name: manifest.name,
      type: manifest.directus?.type,
      entrypoint,
    };
  }
  
  getEndpointRouter(): express.Router {
    const router = express.Router();
    
    for (const [name, extension] of this.extensions) {
      if (extension.type === 'endpoint') {
        router.use(`/${name}`, extension.entrypoint);
      }
    }
    
    return router;
  }
}
```

## Event System

### Event Emitter
```typescript
// api/src/emitter.ts
export class Emitter {
  private events: Map<string, Function[]> = new Map();
  
  // Emit action event
  async emitAction(
    event: string,
    meta: Record<string, any>,
    context: EventContext
  ): Promise<void> {
    const handlers = this.events.get(event) || [];
    
    for (const handler of handlers) {
      await handler(meta, context);
    }
  }
  
  // Emit filter event
  async emitFilter(
    event: string,
    payload: any,
    meta: Record<string, any>,
    context: EventContext
  ): Promise<any> {
    const handlers = this.events.get(event) || [];
    
    let result = payload;
    
    for (const handler of handlers) {
      result = await handler(result, meta, context);
    }
    
    return result;
  }
  
  // Register event handler
  onAction(event: string, handler: Function): void {
    const handlers = this.events.get(event) || [];
    handlers.push(handler);
    this.events.set(event, handlers);
  }
}
```

### Event Hooks
```typescript
// Available events
emitter.onAction('items.create', async (meta, context) => {
  // Handle item creation
});

emitter.onAction('items.update', async (meta, context) => {
  // Handle item update
});

emitter.onAction('items.delete', async (meta, context) => {
  // Handle item deletion
});

emitter.onAction('auth.login', async (meta, context) => {
  // Handle user login
});

emitter.onAction('server.start', async (meta, context) => {
  // Handle server start
});
```

## Next Steps

- **Database Layer**: See [Database Layer](./05-database-layer.md) for database architecture
- **Authentication**: Check [Authentication & Security](./06-auth-security.md) for auth details
- **Extension System**: Review [Extension System](./15-extension-system.md) for customization
- **Real-time Features**: Explore [Real-time Features](./16-realtime-features.md) for WebSocket details
