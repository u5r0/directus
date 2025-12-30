# Directus Project Architecture Documentation

## Overview

**Directus** is a real-time API and App dashboard for managing SQL database content. It's a sophisticated, enterprise-grade headless CMS built as a **monorepo** using modern technologies including Vue 3, Express.js, and TypeScript.

### Key Characteristics
- **License**: Business Source License (BSL) 1.1 with additional use grant
- **Architecture**: Monorepo with pnpm workspaces
- **Node Version**: 22 (required)
- **Package Manager**: pnpm >= 10 < 11
- **Primary Purpose**: Headless CMS/Content Management Platform with REST/GraphQL APIs

---

## 1. Project Structure & Monorepo Organization

### Workspace Packages
```
directus-monorepo/
├── directus/           # Main CLI and deployment package
├── app/               # Vue 3 admin dashboard (v14.4.0)
├── api/               # Express.js backend API (v32.2.0)
├── sdk/               # JavaScript SDK for client integration (v20.3.0)
├── packages/          # 30+ shared utility packages
├── tests/             # Test suites
└── .changeset/        # Changeset configuration for releases
```

### Core Shared Packages (30+)
- **@directus/types** - Shared TypeScript types
- **@directus/schema** - Database schema inspection
- **@directus/extensions** - Extension system utilities
- **@directus/storage** - File storage abstraction
- **@directus/storage-driver-*** - Storage drivers (S3, Azure, GCS, Local, Cloudinary, Supabase)
- **@directus/constants** - Shared constants
- **@directus/errors** - Error definitions
- **@directus/utils** - Utility functions
- **@directus/validation** - Validation schemas
- **@directus/composables** - Vue composables
- **@directus/stores** - Pinia stores
- **@directus/themes** - Theme system
- **@directus/env** - Environment variable handling
- **@directus/pressure** - Pressure-based load shedding

---

## 2. Frontend Architecture (app/)

### Technology Stack
- **Framework**: Vue 3.5.24 with Composition API
- **Language**: TypeScript 5.9.3
- **Build Tool**: Vite 7.1.12
- **State Management**: Pinia 2.3.1
- **Routing**: Vue Router 4.6.3
- **Styling**: SCSS with Sass Embedded
- **Testing**: Vitest 3.2.4 with Happy DOM

### Key Frontend Dependencies
- **UI Components**: Custom v-* component library (60+ components)
- **Rich Editors**: TinyMCE 6.8.6, Editor.js 2.31.0, CodeMirror 5.65.18
- **Data Visualization**: ApexCharts 4.7.0, FullCalendar 6.1.19
- **Maps**: MapBox GL 1.13.3, MapLibre GL 1.15.3
- **Utilities**: VueUse 14.0.0, Vue I18n 11.1.12

### Frontend Architecture Structure
```
app/src/
├── main.ts                 # Application entry point
├── app.vue                 # Root Vue component
├── router.ts               # Vue Router configuration with auth guards
├── api.ts                  # Axios HTTP client with request queue
├── sdk.ts                  # Directus SDK client configuration
├── stores/                 # Pinia stores (21 stores)
│   ├── user.ts            # User authentication & profile
│   ├── server.ts          # Server info & settings
│   ├── collections.ts     # Collection metadata
│   ├── fields.ts          # Field definitions
│   ├── permissions.ts     # User permissions
│   └── settings.ts        # App settings
├── modules/                # Feature modules (7 modules)
│   ├── content/           # Content management
│   ├── files/             # File management
│   ├── users/             # User management
│   ├── settings/          # Settings management
│   ├── activity/          # Activity logs
│   ├── insights/          # Analytics/insights
│   └── visual/            # Visual content
├── components/             # 60+ Vue components (v-button, v-table, etc.)
├── views/                  # Page components
├── routes/                 # Route definitions
├── composables/            # Vue composables for reusable logic
├── directives/             # Custom Vue directives
├── displays/               # Display field types (18 types)
├── interfaces/             # Input field interfaces (40+ types)
├── operations/             # Workflow operations (15 types)
├── panels/                 # Dashboard panels (11 types)
├── layouts/                # Layout components (5 layouts)
├── ai/                     # AI integration components
└── styles/                 # Global SCSS styles
```

## 2.1 Core UI Component System

### Base Components (60+ v-* Components)
All components are globally registered with the `v-` prefix and provide the foundation for the entire UI:

#### Layout & Structure Components
- **VCard** - Card container with title, subtitle, text, and actions slots
- **VCardTitle, VCardSubtitle, VCardText, VCardActions** - Card sub-components
- **VSheet** - Basic surface container with elevation and color
- **VDivider** - Visual separator with customizable styling
- **VWorkspace** - Main workspace container with tile management
- **VWorkspaceTile** - Individual workspace tiles with drag/drop

#### Navigation & Menu Components
- **VBreadcrumb** - Hierarchical navigation breadcrumbs
- **VTabs, VTab, VTabItem, VTabsItems** - Tab navigation system
- **VMenu** - Dropdown menu with positioning and triggers
- **VDrawer** - Slide-out drawer with header and content areas
- **VList, VListItem, VListGroup** - Hierarchical list structures
- **VListItemContent, VListItemIcon, VListItemHint** - List item sub-components

#### Form & Input Components
- **VForm** - Dynamic form builder with field validation and conditional logic
- **VInput** - Text input with validation, prefixes, and suffixes
- **VTextarea** - Multi-line text input with auto-resize
- **VCheckbox** - Checkbox with indeterminate state support
- **VCheckboxTree** - Hierarchical checkbox selection
- **VRadio** - Radio button groups
- **VSelect** - Dropdown selection with search and multi-select
- **VFancySelect** - Enhanced select with custom styling
- **VSlider** - Range slider with step controls
- **VDatePicker** - Date/time selection with calendar
- **VEmojiPicker** - Emoji selection interface
- **VUpload** - File upload with drag/drop and progress

#### Data Display Components
- **VTable** - Advanced data table with sorting, filtering, selection, and pagination
- **VPagination** - Page navigation controls
- **VAvatar** - User/entity avatar display
- **VBadge** - Status badges and counters
- **VChip** - Removable tags and labels
- **VIcon** - Icon display with FontAwesome integration
- **VIconFile** - File type specific icons
- **VImage** - Responsive image display with lazy loading
- **VTextOverflow** - Text truncation with tooltip

#### Feedback & Status Components
- **VNotice** - Alert messages with different severity levels
- **VInfo** - Information display with icons and descriptions
- **VError** - Error state display
- **VProgressLinear** - Linear progress indicators
- **VProgressCircular** - Circular progress spinners
- **VSkeletonLoader** - Loading state placeholders

#### Interactive Components
- **VButton** - Buttons with various styles, sizes, and states
- **VDialog** - Modal dialogs with backdrop and positioning
- **VOverlay** - Backdrop overlay for modals and drawers
- **VHover** - Hover state management
- **VResizeable** - Resizable containers
- **VRemove** - Remove/delete action buttons

#### Utility Components
- **VHighlight** - Text highlighting for search results
- **VTemplateInput** - Template string input with variable support
- **VFieldTemplate** - Dynamic field template rendering
- **VCollectionFieldTemplate** - Collection-specific field templates
- **VFieldList** - Dynamic field list management
- **VDetail** - Expandable detail sections
- **VErrorBoundary** - Error boundary for component isolation

#### Transition Components
- **TransitionBounce** - Bounce animation transitions
- **TransitionDialog** - Dialog enter/exit transitions
- **TransitionExpand** - Expand/collapse transitions

### Component Interaction Patterns

#### Form System Architecture
The **VForm** component is the central hub for all data input and editing:

```typescript
// VForm handles:
- Dynamic field rendering based on collection schema
- Field validation with real-time error display
- Conditional field visibility and requirements
- Batch editing mode for multiple items
- Raw editor mode for advanced users
- AI-powered field assistance
- Nested group field management
- Field width and layout management
```

**VForm** interacts with:
- **Interface Components** (40+ types) for field input
- **Display Components** (18 types) for field output
- **Field Store** for schema and metadata
- **Permissions Store** for field-level access control
- **Collections Store** for collection configuration

#### Table System Architecture
The **VTable** component provides comprehensive data management:

```typescript
// VTable features:
- Dynamic column configuration
- Row selection (single/multiple)
- Manual sorting with drag/drop
- Header reordering and resizing
- Inline editing capabilities
- Batch operations
- Loading states and empty states
- Custom cell rendering via slots
```

**VTable** interacts with:
- **Display Components** for cell rendering
- **Layout Components** for different view modes
- **API Layer** for data fetching and updates
- **Permissions Store** for action availability

## 2.2 Interface System (40+ Input Types)

### Text Input Interfaces
- **input** - Basic text input with validation
- **input-multiline** - Multi-line text with rich formatting
- **input-rich-text-html** - WYSIWYG HTML editor (TinyMCE)
- **input-rich-text-md** - Markdown editor with preview
- **input-block-editor** - Block-based editor (Editor.js)
- **input-code** - Code editor with syntax highlighting (CodeMirror)
- **input-hash** - Hash/password input with generation
- **input-autocomplete-api** - API-powered autocomplete

### Selection Interfaces
- **select-dropdown** - Single selection dropdown
- **select-multiple-dropdown** - Multi-selection dropdown
- **select-dropdown-m2o** - Many-to-one relationship selector
- **select-radio** - Radio button selection
- **select-multiple-checkbox** - Checkbox group selection
- **select-multiple-checkbox-tree** - Hierarchical checkbox selection
- **select-color** - Color picker interface
- **select-icon** - Icon selection interface

### Relationship Interfaces
- **list-o2m** - One-to-many relationship management
- **list-m2m** - Many-to-many relationship management
- **list-m2a** - Many-to-any (polymorphic) relationships
- **list-o2m-tree-view** - Hierarchical one-to-many relationships
- **collection-item-dropdown** - Single item selection from collection
- **collection-item-multiple-dropdown** - Multiple item selection

### File & Media Interfaces
- **file** - Single file upload and management
- **file-image** - Image-specific upload with preview
- **files** - Multiple file upload and management

### Specialized Interfaces
- **boolean** - Boolean toggle/checkbox
- **datetime** - Date and time picker
- **slider** - Numeric range slider
- **map** - Geographic coordinate picker
- **tags** - Tag input with autocomplete
- **translations** - Multi-language content management
- **list** - Dynamic list/array input

### Grouping Interfaces
- **group-accordion** - Collapsible field groups
- **group-detail** - Detailed field groups
- **group-raw** - Raw JSON field groups

### Presentation Interfaces
- **presentation-divider** - Visual section dividers
- **presentation-header** - Section headers
- **presentation-links** - Link collections
- **presentation-notice** - Informational notices

### System Interfaces
- **_system/system-collection** - System collection management
- **_system/system-field** - System field configuration
- **_system/system-interface** - Interface configuration

## 2.3 Display System (18 Output Types)

### Basic Displays
- **raw** - Raw value display without formatting
- **formatted-value** - Formatted value with type-specific rendering
- **formatted-json-value** - Formatted JSON with syntax highlighting

### Content Displays
- **boolean** - Boolean value with icons/text
- **datetime** - Formatted date/time display
- **color** - Color swatch display
- **icon** - Icon display with customization
- **image** - Image display with lightbox
- **file** - File information and download links
- **filesize** - Human-readable file size
- **mime-type** - MIME type with icons

### Relationship Displays
- **related-values** - Related item values
- **user** - User information with avatar
- **collection** - Collection name display

### Advanced Displays
- **labels** - Tag/label display with colors
- **rating** - Star rating display
- **hash** - Masked hash/password display
- **translations** - Multi-language content display

## 2.4 Layout System (5 Layout Types)

### Data Layouts
- **tabular** - Traditional table layout with sorting and filtering
- **cards** - Card-based grid layout with customizable templates
- **calendar** - Calendar view for date-based content
- **map** - Geographic map layout for location data
- **kanban** - Kanban board for workflow management

Each layout provides:
- **Component Slot** - Main content rendering
- **Options Slot** - Layout-specific configuration
- **Sidebar Slot** - Additional controls and filters
- **Actions Slot** - Layout-specific actions

## 2.5 Panel System (11 Dashboard Widgets)

### Chart Panels
- **bar-chart** - Bar chart visualization
- **line-chart** - Line chart with time series
- **pie-chart** - Pie/donut chart display
- **time-series** - Time-based data visualization

### Data Panels
- **list** - Simple data list display
- **metric** - Single metric display with formatting
- **metric-list** - Multiple metrics in list format
- **meter** - Progress meter/gauge display

### Content Panels
- **label** - Static text/label display
- **variable** - Dynamic variable display
- **relational-variable** - Related data variables

## 2.6 State Management & API Integration

### Pinia Store Architecture
The frontend uses 21 specialized Pinia stores for state management:

#### Core Stores
- **user** - Authentication, profile, preferences, page tracking
- **server** - Server info, project settings, capabilities
- **permissions** - Role-based access control, field-level permissions
- **collections** - Collection metadata, translations, CRUD operations
- **fields** - Field definitions, validation, relationships
- **relations** - Relationship mappings and constraints

#### Feature Stores
- **files** - File management, upload progress, metadata
- **flows** - Workflow definitions and execution state
- **insights** - Analytics data and dashboard state
- **notifications** - User notifications and alerts
- **presets** - User preferences and saved filters
- **settings** - Application configuration
- **translations** - Multi-language content management
- **extensions** - Extension registry and loading state
- **requests** - HTTP request tracking and loading states

### API Communication Layer

#### HTTP Client (api.ts)
```typescript
// Axios-based HTTP client with:
- Request/response interceptors
- Authentication token management
- Request queuing with p-queue (5 concurrent, rate limited)
- Latency measurement
- Error handling and retry logic
- CORS and credential management
```

#### SDK Client (sdk.ts)
```typescript
// Directus SDK client with:
- Type-safe API methods
- Authentication session management
- Real-time WebSocket connections
- Request queuing integration
- Automatic token refresh
- Multi-protocol support (REST/GraphQL)
```

### Component-API Interaction Flow

#### Data Fetching Pattern
```
1. Component mounts → Store action triggered
2. Store calls API via sdk/axios
3. Request queued and executed
4. Response updates store state
5. Component reactively updates via computed properties
6. Loading states managed automatically
```

#### Form Submission Pattern
```
1. VForm collects field values
2. Validation runs client-side
3. API request sent via store action
4. Optimistic updates applied to store
5. Server response confirms/reverts changes
6. Success/error feedback displayed
7. Related stores updated (collections, fields, etc.)
```

#### Real-time Updates
```
1. WebSocket connection established via SDK
2. Components subscribe to relevant collections
3. Server broadcasts changes via WebSocket
4. Store state updated automatically
5. UI reactively reflects changes
6. Conflict resolution for concurrent edits
```

### Module System Integration

#### Module Registration
Each of the 7 modules (content, files, users, settings, activity, insights, visual) provides:
- **Route definitions** with permission checks
- **Component registration** for module-specific UI
- **Store integration** for module data
- **Permission validation** for access control

#### Module Communication
Modules interact through:
- **Shared stores** for cross-module data
- **Event bus** for loose coupling
- **Router navigation** for deep linking
- **Permission system** for access control

This comprehensive component system provides a fully integrated, type-safe, and reactive user interface that seamlessly communicates with the backend API while maintaining excellent user experience through optimistic updates, real-time synchronization, and robust error handling.

---

## 3. Backend Architecture (api/)

### Technology Stack
- **Framework**: Express.js 4.21.2
- **Language**: TypeScript 5.9.3
- **Database ORM**: Knex.js 3.1.0 (query builder)
- **Schema Inspection**: @directus/schema (custom package)
- **Logging**: Pino 9.7.0 with HTTP middleware
- **Authentication**: JWT (jsonwebtoken 9.0.3)
- **Testing**: Vitest 3.2.4

### Database Support
- **PostgreSQL** (with PostGIS for geometry)
- **MySQL** 8 & 5.7 (with utf8mb4 support)
- **MariaDB** 11.4
- **SQLite3** (with foreign key support)
- **Oracle DB** (18 XE)
- **MS SQL Server** (2019+)
- **CockroachDB** (with serial normalization)

### Backend Architecture Structure
```
api/src/
├── start.ts                # Application entry point
├── server.ts               # HTTP server setup with Terminus
├── app.ts                  # Express app configuration
├── controllers/            # 35+ REST endpoint controllers
│   ├── items.ts           # CRUD operations
│   ├── collections.ts     # Collection management
│   ├── fields.ts          # Field management
│   ├── auth.ts            # Authentication endpoints
│   ├── graphql.ts         # GraphQL endpoint
│   ├── files.ts           # File upload/download
│   ├── users.ts           # User management
│   ├── webhooks.ts        # Webhook management
│   ├── flows.ts           # Workflow management
│   ├── ai/chat.ts         # AI chat endpoint
│   └── mcp.ts             # Model Context Protocol
├── services/              # 40+ business logic services
│   ├── items.ts           # Item CRUD service
│   ├── collections.ts     # Collection service
│   ├── authentication.ts  # Auth service
│   ├── files.ts           # File service
│   ├── graphql/           # GraphQL service
│   ├── mail/              # Email service
│   ├── webhooks.ts        # Webhook service
│   └── flows.ts           # Workflow service
├── middleware/            # 19 middleware functions
│   ├── authenticate.ts    # JWT authentication
│   ├── extract-token.ts   # Token extraction
│   ├── schema.ts          # Schema loading
│   ├── cache.ts           # Response caching
│   ├── cors.ts            # CORS handling
│   ├── rate-limiter-*.ts  # Rate limiting
│   └── error-handler.ts   # Error handling
├── database/              # Database layer
│   ├── index.ts           # Knex setup & connection
│   ├── migrations/        # Database migrations
│   └── helpers/           # DB-specific helpers
├── auth/                  # Authentication providers
│   ├── drivers/           # SAML, OpenID Connect, LDAP, Local
├── extensions/            # Extension system
├── websocket/             # WebSocket handlers
│   ├── controllers/       # Subscription & logs
│   └── handlers/          # Message handlers
├── schedules/             # Scheduled tasks
│   ├── retention.ts       # Data retention
│   ├── telemetry.ts       # Telemetry collection
│   └── metrics.ts         # Metrics collection
├── ai/                    # AI integration
│   ├── chat/              # AI chat service
│   ├── mcp/               # Model Context Protocol
│   └── tools/             # AI tools
├── logger/                # Logging setup
├── metrics/               # Prometheus metrics
├── cache.ts               # Cache management
├── emitter.ts             # Event emitter
└── flows.ts               # Workflow engine
```

### REST API Endpoints
- `/auth/*` - Authentication (login, logout, refresh, SSO)
- `/items/*` - CRUD operations on collections
- `/collections/*` - Collection management
- `/fields/*` - Field management
- `/files/*` - File upload/download
- `/assets/*` - Asset serving with transformations
- `/users/*` - User management
- `/roles/*` - Role management
- `/permissions/*` - Permission management
- `/webhooks/*` - Webhook management
- `/flows/*` - Workflow management
- `/extensions/*` - Extension management
- `/graphql` - GraphQL endpoint
- `/ai/chat` - AI chat endpoint
- `/mcp` - Model Context Protocol
- `/server/*` - Server info & health checks
- `/settings/*` - System settings
- `/activity/*` - Activity logs
- `/revisions/*` - Item revisions
- `/metrics` - Prometheus metrics

### Authentication & Authorization
- **JWT-based** authentication with refresh tokens
- **Multiple auth providers**: Local, SAML, OpenID Connect, LDAP
- **Role-based access control** (RBAC)
- **Policy-based permissions** with field-level granularity
- **Two-factor authentication** (TFA) support
- **Token extraction** from headers, cookies, or query params

### Advanced Features
- **GraphQL API** with schema composition
- **WebSocket support** for real-time subscriptions
- **File uploads** with TUS protocol support
- **Asset transformation** (image resizing, format conversion)
- **Workflow engine** for automation
- **Extension system** for custom functionality
- **AI integration** (Anthropic, OpenAI)
- **Model Context Protocol** (MCP) support
- **Webhooks** for external integrations
- **Rate limiting** (global & per-IP)
- **Pressure-based load shedding**
- **Metrics collection** (Prometheus)

---

## 4. Data Model & Database Schema

### Core System Collections
```sql
-- User Management
directus_users          # User accounts
directus_roles          # User roles
directus_permissions    # Permission rules
directus_policies       # Access policies

-- Content Management
directus_collections    # Collection metadata
directus_fields         # Field definitions
directus_relations      # Relationship definitions
directus_activity       # Activity logs
directus_revisions      # Item version history
directus_comments       # Item comments

-- File Management
directus_files          # File metadata
directus_folders        # File folder structure

-- System Configuration
directus_settings       # System settings
directus_migrations     # Migration tracking
directus_extensions     # Extension registry

-- Workflow & Automation
directus_flows          # Workflow definitions
directus_operations     # Workflow operations
directus_webhooks       # Webhook definitions

-- UI & UX
directus_dashboards     # Dashboard definitions
directus_panels         # Dashboard panels
directus_presets        # User presets/filters
directus_notifications  # User notifications
directus_shares         # Shared item links
directus_translations   # Translation strings
```

### Field Types Supported
- **Text**: string, text, varchar
- **Numbers**: integer, decimal, float, bigInteger
- **Boolean**: boolean
- **Date/Time**: date, time, datetime, timestamp
- **JSON**: json
- **UUID**: uuid
- **Geometry**: geometry (with PostGIS/Spatialite)
- **Files/Images**: file references
- **Relations**: M2O, O2M, M2M, M2A (polymorphic)

### Relationship Types
- **Many-to-One (M2O)**: Foreign key relationships
- **One-to-Many (O2M)**: Reverse relationships
- **Many-to-Many (M2M)**: Junction table relationships
- **Many-to-Any (M2A)**: Polymorphic relationships

### Database Features
- **Migrations**: Version-controlled schema changes
- **Revisions**: Full item version history with diffs
- **Activity tracking**: Comprehensive user action logging
- **Soft deletes**: Status-based deletion
- **Timestamps**: Created/updated tracking
- **Foreign key constraints**: Referential integrity
- **Collation support**: UTF-8 and custom collations

---

## 5. SDK Architecture (sdk/)

### Technology Stack
- **Language**: TypeScript 5.9.3
- **Build Tool**: tsdown (TypeScript bundler)
- **Output**: ESM and CommonJS formats
- **Testing**: Vitest 3.2.4

### SDK Structure
```typescript
// Core client creation
const client = createDirectus<Schema>('https://api.example.com')
  .with(authentication())
  .with(rest())
  .with(graphql())
  .with(realtime());

// Composable extensions
client
  .with(staticToken('token'))
  .with(refresh())
  .with(readItems('articles'))
  .with(createItem('articles'))
  .with(updateItem('articles'))
  .with(deleteItem('articles'));
```

### Key Features
- **Type-safe**: Full TypeScript support with schema inference
- **Composable**: Modular extension system
- **Multi-protocol**: REST, GraphQL, and WebSocket support
- **Authentication**: Multiple auth strategies
- **Real-time**: WebSocket subscriptions
- **File uploads**: TUS protocol support

---

## 6. Extension System

### Extension Types
- **Interfaces**: Custom input components
- **Displays**: Custom display components
- **Layouts**: Custom layout components
- **Modules**: Custom admin modules
- **Panels**: Custom dashboard panels
- **Hooks**: Server-side event hooks
- **Endpoints**: Custom API endpoints
- **Operations**: Custom workflow operations

### Extension Architecture
```
extensions/
├── interfaces/
├── displays/
├── layouts/
├── modules/
├── panels/
├── hooks/
├── endpoints/
└── operations/
```

### Extension Development
- **SDK**: @directus/extensions-sdk for development
- **CLI**: @directus/create-directus-extension for scaffolding
- **Registry**: @directus/extensions-registry for discovery
- **Hot reload**: Development mode with live reloading

---

## 7. Build & Deployment Configuration

### Root Configuration
- **package.json**: Monorepo root with workspace scripts
- **pnpm-workspace.yaml**: Workspace definition with catalog versioning
- **eslint.config.js**: ESLint configuration (TypeScript, Vue)
- **.prettierrc.json**: Code formatting rules
- **ecosystem.config.cjs**: PM2 process manager configuration
- **Dockerfile**: Multi-stage Docker build
- **docker-compose.yml**: Development database services

### Build Scripts
```json
{
  "build": "pnpm --recursive run build",
  "dev": "pnpm --recursive run dev",
  "test": "pnpm --recursive --filter '!tests-blackbox' test",
  "test:blackbox": "End-to-end testing with deployed instance",
  "lint": "eslint --cache .",
  "lint:style": "stylelint '**/*.{css,scss,vue}'",
  "format": "prettier --cache --check ."
}
```

### Environment Variables
```bash
# Database
DB_CLIENT=pg|mysql|sqlite3|oracledb|mssql|cockroachdb
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=directus
DB_USER=directus
DB_PASSWORD=password

# Security
SECRET=your-secret-key-min-32-bytes
PUBLIC_URL=https://your-domain.com

# Features
SERVE_APP=true
CORS_ENABLED=true
RATE_LIMITER_ENABLED=true
WEBSOCKETS_ENABLED=true
TUS_ENABLED=true
MCP_ENABLED=true
METRICS_ENABLED=true

# Storage
STORAGE_LOCATIONS=local,s3,azure,gcs
STORAGE_LOCAL_ROOT=./uploads

# Email
EMAIL_FROM=no-reply@example.com
EMAIL_TRANSPORT=smtp
EMAIL_SMTP_HOST=smtp.example.com

# Cache
CACHE_ENABLED=true
CACHE_STORE=redis
REDIS_HOST=localhost
REDIS_PORT=6379
```

---

## 8. System Interactions & Data Flow

### Request Flow
```
1. Client (Vue app) → SDK → HTTP/GraphQL request
2. Express Server → Middleware Pipeline:
   - Extract token
   - Authenticate user
   - Load schema
   - Sanitize query
   - Apply cache
3. Controller → Service → Database (Knex)
4. Response → Middleware → Client
5. Client updates Pinia store → Vue reactivity
```

### Real-Time Flow
```
1. Client subscribes via WebSocket
2. WebSocket Controller manages connection
3. Database triggers event on change
4. Event Emitter broadcasts to subscribers
5. WebSocket sends update to client
6. Client updates store and UI reactively
```

### Authentication Flow
```
1. User submits credentials
2. Auth Controller validates
3. Auth Service checks database
4. JWT Token generated
5. Token returned to client
6. Client stores token
7. Subsequent requests include token
8. Authenticate Middleware validates
9. User context attached to request
```

### Extension Flow
```
1. Extension Manager discovers extensions
2. Extensions loaded from local/module/registry
3. App registers extension routes/components
4. API registers extension endpoints
5. Extensions hook into lifecycle events
```

---

## 9. Security Architecture

### Authentication Methods
- **Local**: Username/password with Argon2 hashing
- **SAML**: Enterprise SSO integration
- **OpenID Connect**: OAuth 2.0 / OIDC providers
- **LDAP**: Active Directory integration
- **JWT**: Stateless token-based auth

### Authorization System
- **Role-Based Access Control (RBAC)**
- **Policy-Based Permissions**
- **Field-Level Granularity**
- **Collection-Level Permissions**
- **Action-Based Permissions** (create, read, update, delete)

### Security Features
- **Two-Factor Authentication (TFA)**
- **Rate Limiting** (global & per-IP)
- **CORS** support with configurable origins
- **Helmet.js** for security headers
- **Query sanitization** to prevent injection
- **IP validation** for external requests
- **Session management** with refresh tokens

---

## 10. Performance & Scalability

### Performance Features
- **Response caching** with configurable TTL
- **Database query optimization** through Knex
- **Pressure-based load shedding** to prevent overload
- **Rate limiting** to prevent abuse
- **Lazy loading** of extensions
- **Virtual scrolling** for large lists
- **Image optimization** with Sharp
- **CDN support** for assets

### Scalability Features
- **Horizontal scaling** with PM2 cluster mode
- **Database connection pooling**
- **Redis caching** for session and data
- **File storage abstraction** (local, S3, Azure, GCS)
- **WebSocket scaling** with Redis adapter
- **Metrics collection** for monitoring
- **Health checks** for load balancers

### Monitoring & Observability
- **Prometheus metrics** collection
- **Structured logging** with Pino
- **Activity tracking** for audit trails
- **Performance monitoring** with request timing
- **Error tracking** with detailed stack traces
- **Health endpoints** for uptime monitoring

---

## 11. AI Integration

### AI Features
- **AI Chat**: Integrated chat interface with AI models
- **Model Context Protocol (MCP)**: Standardized AI tool integration
- **Multiple Providers**: Anthropic, OpenAI support
- **AI Tools**: Custom tools for AI interactions
- **Context Management**: Efficient context handling for AI

### AI Architecture
```
api/src/ai/
├── chat/              # AI chat service
│   ├── router.ts      # Chat API endpoints
│   └── service.ts     # Chat business logic
├── mcp/               # Model Context Protocol
│   ├── server.ts      # MCP server implementation
│   └── tools.ts       # MCP tool definitions
└── tools/             # AI tool implementations
```

---

---

## 12. UI Component Interactions & API Communication

### Frontend-Backend Communication Architecture

#### API Communication Layer
```typescript
// app/src/api.ts - Axios-based HTTP client
const api = axios.create({
  baseURL: getRootPath(),
  withCredentials: true,
});

// Request queue management for rate limiting
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,
  intervalCap: 5,
  interval: 500,
});
```

#### SDK Integration
```typescript
// app/src/sdk.ts - Directus SDK client
export const sdk: SdkClient = createDirectus(getPublicURL())
  .with(authentication('session', { 
    credentials: 'include', 
    msRefreshBeforeExpires: SDK_AUTH_REFRESH_BEFORE_EXPIRES 
  }))
  .with(rest({ credentials: 'include' }));
```

### Component-to-API Interaction Patterns

#### 1. Store-Based Data Management
**Pattern**: Components → Pinia Stores → API → Database

```typescript
// Example: Collections Store
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  
  async function hydrate() {
    const response = await api.get<any>(`/collections`);
    collections.value = response.data.data.map(prepareCollectionForApp);
  }
  
  async function updateCollection(collection: string, updates: DeepPartial<Collection>) {
    await api.patch(`/collections/${collection}`, updates);
    await hydrate(); // Re-fetch data
    notify({ title: i18n.global.t('update_collection_success') });
  }
});
```

#### 2. Form Component Interactions
**Pattern**: v-form → Field Components → API Validation → Store Updates

```vue
<!-- v-form.vue - Main form component -->
<FormField
  v-for="fieldName in fieldNames"
  :field="fieldsMap[fieldName]"
  :model-value="values[fieldName]"
  :disabled="isDisabled(fieldsMap[fieldName])"
  @update:model-value="setValue(fieldName, $event)"
  @set-field-value="setValue($event.field, $event.value, { force: true })"
/>
```

#### 3. Real-Time Updates
**Pattern**: WebSocket → Event Emitter → Store Reactivity → Component Updates

```typescript
// Real-time subscription handling
sdk.subscribe('items', {
  collection: 'articles',
  query: { fields: ['*'] }
}, (data) => {
  // Store automatically updates
  // Components react to store changes
});
```

### Component Hierarchy & Communication

#### 1. Top-Level Architecture
```
App.vue
├── Router (Vue Router)
├── PrivateView (Admin Interface)
│   ├── Navigation (ContentNavigation)
│   ├── Header (Actions, Search, Filters)
│   ├── Main Content (Layouts, Forms, Tables)
│   └── Sidebar (Details, Options, Flows)
└── PublicView (Login, Setup, Shared)
```

#### 2. Inter-Component Communication Methods

**Props Down, Events Up**:
```vue
<!-- Parent Component -->
<VTable
  :headers="headers"
  :items="items"
  :selection="selection"
  @click:row="handleRowClick"
  @update:selection="selection = $event"
/>
```

**Provide/Inject for Deep Hierarchies**:
```typescript
// Parent provides context
provide('values', values);
provide('live-preview-active', livePreviewActive);

// Child components inject
const values = inject('values');
const livePreviewActive = inject('live-preview-active');
```

**Store-Based Communication**:
```typescript
// Components access shared state
const userStore = useUserStore();
const permissionsStore = usePermissionsStore();
const fieldsStore = useFieldsStore();
```

#### 3. Event System
```typescript
// Global event emitter for cross-component communication
import { Events, emitter } from './events';

// Emit events
emitter.emit(Events.tabIdle);
emitter.emit(Events.tabActive);

// Listen to events
emitter.on(Events.tabIdle, () => {
  sdk.stopRefreshing();
});
```

---

## 13. Frontend UI Studio - Detailed Architecture

### Core UI Studio Components

#### 1. Layout System
The UI studio uses a flexible layout system with 5 main layouts:

```typescript
// Layout registration system
export function registerLayouts(layouts: LayoutConfig[], app: App): void {
  for (const layout of layouts) {
    app.component(`layout-${layout.id}`, layout.component);
    app.component(`layout-options-${layout.id}`, layout.slots.options);
    app.component(`layout-sidebar-${layout.id}`, layout.slots.sidebar);
    app.component(`layout-actions-${layout.id}`, layout.slots.actions);
  }
}
```

**Available Layouts**:
- **Tabular**: Traditional table view with sorting, filtering, pagination
- **Cards**: Card-based grid layout for visual content
- **Calendar**: Calendar view for date-based content
- **Map**: Geographic visualization for location data
- **Kanban**: Board view for workflow management

#### 2. Interface System (40+ Input Types)
**Interface Architecture**:
```typescript
// Interface registration
export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
  for (const inter of interfaces) {
    app.component(`interface-${inter.id}`, inter.component);
    
    if (inter.options !== null) {
      app.component(`interface-options-${inter.id}`, inter.options);
    }
  }
}
```

**Key Interface Categories**:
- **Input Interfaces**: text, number, email, password, textarea
- **Selection Interfaces**: dropdown, radio, checkbox, tags
- **Rich Content**: WYSIWYG, Markdown, code editor, block editor
- **Media Interfaces**: file upload, image editor, gallery
- **Relational Interfaces**: M2O, O2M, M2M, M2A selectors
- **Special Interfaces**: JSON editor, color picker, date/time
- **Group Interfaces**: accordion, tabs, raw editor

#### 3. Display System (18 Display Types)
**Display Components** render read-only field values:
```typescript
export function registerDisplays(displays: DisplayConfig[], app: App): void {
  for (const display of displays) {
    app.component(`display-${display.id}`, display.component);
  }
}
```

**Display Types**:
- **Basic**: raw, formatted-value, boolean, datetime
- **Media**: image, file, filesize, mime-type
- **Relational**: related-values, user, collection
- **Visual**: color, icon, rating, labels
- **Advanced**: formatted-json-value, translations, hash

### UI Studio Architecture Patterns

#### 1. Module-Based Organization
```
app/src/modules/
├── content/           # Content management interface
│   ├── routes/        # Collection & item views
│   ├── components/    # Content-specific components
│   └── navigation.vue # Content navigation
├── files/             # File management interface
├── users/             # User management interface
├── settings/          # System settings interface
├── activity/          # Activity logs interface
├── insights/          # Analytics dashboard
└── visual/            # Visual content tools
```

#### 2. View Composition Pattern
**PrivateView Structure**:
```vue
<PrivateView
  :title="title"
  :icon="icon"
  :show-back="!singleton"
  :show-header-shadow="showHeaderShadow"
>
  <template #headline>
    <VBreadcrumb :items="breadcrumb" />
  </template>
  
  <template #actions>
    <SearchInput v-model="search" />
    <ActionButtons />
  </template>
  
  <template #navigation>
    <ContentNavigation />
  </template>
  
  <!-- Main content area -->
  <component :is="`layout-${layout}`" v-bind="layoutState" />
  
  <template #sidebar>
    <SidebarDetails />
  </template>
</PrivateView>
```

#### 3. Form System Architecture
**Dynamic Form Generation**:
```vue
<VForm
  v-model="edits"
  :fields="fields"
  :initial-values="item"
  :validation-errors="validationErrors"
  :disabled="isFormDisabled"
  @update:model-value="handleFormUpdate"
>
  <!-- Form fields are dynamically generated based on collection schema -->
</VForm>
```

**Field Rendering Logic**:
```typescript
// Form field component selection
const component = computed(() => {
  if (field.meta?.special?.includes('group')) {
    return `interface-${field.meta?.interface || 'group-standard'}`;
  }
  return 'FormField';
});
```

### Advanced UI Studio Features

#### 1. Live Preview System
**Split Panel Architecture**:
```vue
<SplitPanel
  v-model:size="livePreviewSize"
  v-model:collapsed="livePreviewCollapsed"
  primary="end"
  size-unit="%"
>
  <template #start>
    <VForm /> <!-- Edit form -->
  </template>
  
  <template #end>
    <LivePreview :url="previewUrl" /> <!-- Live preview -->
  </template>
</SplitPanel>
```

#### 2. Batch Operations
**Batch Edit Interface**:
```vue
<DrawerBatch
  v-model:active="batchEditActive"
  :primary-keys="selection"
  :collection="collection"
  @refresh="batchRefresh"
/>
```

#### 3. Version Management
**Content Versioning UI**:
```vue
<VersionMenu
  :collection="collection"
  :primary-key="primaryKey"
  :current-version="currentVersion"
  :versions="versions"
  @switch="currentVersion = $event"
/>
```

#### 4. Workflow Integration
**Flow Execution Interface**:
```vue
<FlowDialogs v-bind="flowDialogsContext" />
<FlowSidebarDetail :manual-flows />
```

### UI Studio State Management

#### 1. Hydration System
**Store Hydration Process**:
```typescript
export async function hydrate(): Promise<void> {
  const stores = useStores([
    useCollectionsStore,
    useFieldsStore,
    useUserStore,
    usePermissionsStore,
    // ... other stores
  ]);

  // Hydrate user store first (required by others)
  await userStore.hydrate();
  
  // Parallel hydration of remaining stores
  await Promise.all(stores.map(store => store.hydrate?.()));
  
  // Load extensions
  await onHydrateExtensions();
}
```

#### 2. Permission-Based UI
**Dynamic UI Based on Permissions**:
```typescript
const {
  createAllowed,
  updateAllowed,
  deleteAllowed,
  archiveAllowed
} = useCollectionPermissions(collection);

// UI elements conditionally rendered
const showCreateButton = computed(() => createAllowed.value);
const showDeleteButton = computed(() => deleteAllowed.value);
```

#### 3. Reactive Field System
**Dynamic Field Visibility**:
```typescript
const fieldsWithConditions = computed(() => {
  return formFields.value.map(field => 
    applyConditions(values.value, field, version.value)
  );
});
```

### UI Studio Extension Points

#### 1. Custom Interfaces
```typescript
// Custom interface registration
app.component('interface-custom-editor', CustomEditorInterface);
app.component('interface-options-custom-editor', CustomEditorOptions);
```

#### 2. Custom Layouts
```typescript
// Custom layout registration
app.component('layout-custom-grid', CustomGridLayout);
app.component('layout-options-custom-grid', CustomGridOptions);
app.component('layout-sidebar-custom-grid', CustomGridSidebar);
```

#### 3. Custom Displays
```typescript
// Custom display registration
app.component('display-custom-format', CustomFormatDisplay);
```

### Performance Optimizations

#### 1. Virtual Scrolling
**Large Dataset Handling**:
```vue
<VirtualList
  :items="items"
  :item-height="48"
  :buffer="10"
  @scroll="handleScroll"
/>
```

#### 2. Lazy Loading
**Component Lazy Loading**:
```typescript
const LazyComponent = defineAsyncComponent(() => 
  import('./components/HeavyComponent.vue')
);
```

#### 3. Request Optimization
**Request Queuing & Deduplication**:
```typescript
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,
  intervalCap: 5,
  interval: 500,
  carryoverConcurrencyCount: true,
});
```

---

## Summary

Directus represents a sophisticated, enterprise-grade headless CMS architecture that combines:

- **Modern Frontend**: Vue 3 + TypeScript with comprehensive component system
- **Robust Backend**: Express.js + Knex with multi-database support
- **Type-Safe SDK**: Composable TypeScript client
- **Extensible Platform**: Plugin architecture for customization
- **Enterprise Security**: Multiple auth providers with fine-grained permissions
- **Real-Time Capabilities**: WebSocket support for live updates
- **AI Integration**: Modern AI capabilities with MCP support
- **Production Ready**: Docker deployment with monitoring and scaling features

### UI Studio Highlights:
- **40+ Interface Types**: Comprehensive input components for all data types
- **5 Layout Systems**: Flexible content presentation options
- **18 Display Types**: Rich data visualization components
- **Live Preview**: Real-time content preview with split-panel interface
- **Batch Operations**: Efficient multi-item editing capabilities
- **Version Management**: Content versioning with visual diff tools
- **Permission-Based UI**: Dynamic interface based on user permissions
- **Extension System**: Pluggable architecture for custom components

The monorepo architecture with 30+ shared packages ensures code reusability and maintainability while providing a complete platform for content management, API generation, and custom application development. The UI studio provides a powerful, extensible interface that adapts to any data model while maintaining enterprise-grade performance and security.
