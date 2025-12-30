# Monorepo Structure

Directus is organized as a monorepo using pnpm workspaces, containing multiple packages that work together to provide the complete platform.

## Table of Contents
- [Workspace Organization](#workspace-organization)
- [Core Packages](#core-packages)
- [Shared Packages](#shared-packages)
- [Package Dependencies](#package-dependencies)
- [Build System](#build-system)
- [Development Workflow](#development-workflow)

## Workspace Organization

### pnpm Workspace Configuration
```yaml
# pnpm-workspace.yaml
packages:
  - directus          # Main deployment package
  - app              # Vue 3 admin interface
  - api              # Express.js backend
  - sdk              # TypeScript SDK
  - packages/*       # 30+ shared utility packages
  - tests/*          # Test suites
```

### Catalog Dependencies
Directus uses pnpm's catalog feature for centralized dependency management:
- **Strict Catalog Mode**: All dependencies must be defined in the catalog
- **Version Consistency**: Ensures all packages use the same versions
- **Easy Updates**: Update versions in one place

## Core Packages

### 1. Directus (Main Package)
**Location**: `directus/`
**Purpose**: Main deployment package and CLI

```json
{
  "name": "directus",
  "type": "module",
  "bin": {
    "directus": "cli.js"
  },
  "dependencies": {
    "@directus/api": "workspace:*",
    "@directus/app": "workspace:*"
  }
}
```

**Features**:
- CLI commands for database migrations, user management, etc.
- Bundles API and App for deployment
- Version management
- Extension scaffolding

### 2. API Package
**Location**: `api/`
**Purpose**: Express.js backend server

**Key Technologies**:
- Express.js 4.21.2 - Web framework
- Knex 3.1.0 - SQL query builder
- TypeScript 5.9.3 - Type safety
- Pino 9.7.0 - Logging
- GraphQL 16.12.0 - GraphQL API
- WebSocket (ws 8.18.3) - Real-time communication

**Structure**:
```
api/src/
├── app.ts                 # Express app configuration
├── server.ts              # HTTP server setup
├── controllers/           # Route controllers (30+ endpoints)
├── services/              # Business logic layer
├── database/              # Database abstraction
│   ├── migrations/        # Schema migrations
│   ├── helpers/           # Database-specific helpers
│   └── seeds/             # Seed data
├── middleware/            # Express middleware
├── auth/                  # Authentication providers
├── extensions/            # Extension system
├── operations/            # Workflow operations
├── websocket/             # WebSocket handlers
├── ai/                    # AI integration
└── utils/                 # Utility functions
```

### 3. App Package
**Location**: `app/`
**Purpose**: Vue 3 admin interface

**Key Technologies**:
- Vue 3.5.24 - Frontend framework
- Vite 7.1.12 - Build tool
- Pinia 2.3.1 - State management
- Vue Router 4.6.3 - Routing
- TypeScript 5.9.3 - Type safety
- SCSS - Styling

**Structure**:
```
app/src/
├── main.ts                # Application entry
├── components/            # 60+ UI components
├── modules/               # 7 feature modules
├── stores/                # 21 Pinia stores
├── interfaces/            # 40+ input interfaces
├── displays/              # 18 output displays
├── layouts/               # 5 content layouts
├── panels/                # 11 dashboard panels
├── operations/            # 15 workflow operations
└── utils/                 # 80+ utility functions
```

### 4. SDK Package
**Location**: `sdk/`
**Purpose**: TypeScript SDK for API integration

**Features**:
- Type-safe API client
- Composable query builder
- Authentication management
- Real-time subscriptions
- GraphQL support

**Usage**:
```typescript
import { createDirectus, rest, authentication } from '@directus/sdk';

const client = createDirectus('https://directus.example.com')
  .with(rest())
  .with(authentication('session'));
```

## Shared Packages

### Core Shared Packages

#### @directus/types
**Purpose**: Shared TypeScript type definitions

**Exports**:
- Collection types
- Field types
- Permission types
- API response types
- Extension types

#### @directus/schema
**Purpose**: Database schema inspection and manipulation

**Features**:
- Multi-database support (PostgreSQL, MySQL, SQLite, etc.)
- Schema introspection
- Migration generation
- Type mapping

#### @directus/utils
**Purpose**: Shared utility functions

**Categories**:
- String manipulation
- Date/time utilities
- Validation helpers
- Type guards
- Array/object utilities

#### @directus/errors
**Purpose**: Standardized error classes

**Error Types**:
- `InvalidPayloadError`
- `ForbiddenError`
- `UnauthorizedError`
- `ServiceUnavailableError`
- `ValidationError`

#### @directus/env
**Purpose**: Environment variable management

**Features**:
- Type-safe environment access
- Default values
- Validation
- Configuration loading

### Extension Packages

#### @directus/extensions
**Purpose**: Extension system core

**Features**:
- Extension loading
- Type definitions
- Build tools
- Scaffolding

#### @directus/extensions-sdk
**Purpose**: SDK for building extensions

**Provides**:
- Extension API
- Type definitions
- Helper functions
- Build configuration

### Data Packages

#### @directus/data
**Purpose**: Data abstraction layer

**Features**:
- Query builder
- Filter parsing
- Aggregation
- Sorting and pagination

#### @directus/data-driver-postgres
**Purpose**: PostgreSQL data driver

#### @directus/data-driver-mysql
**Purpose**: MySQL data driver

### Storage Packages

#### @directus/storage
**Purpose**: File storage abstraction

**Drivers**:
- Local filesystem
- AWS S3
- Google Cloud Storage
- Azure Blob Storage
- Supabase Storage

### Other Shared Packages

#### @directus/pressure
**Purpose**: Server pressure monitoring

**Features**:
- Event loop monitoring
- Memory usage tracking
- Request throttling

#### @directus/format-title
**Purpose**: String formatting utilities

#### @directus/random
**Purpose**: Random value generation

#### @directus/composables
**Purpose**: Shared Vue composables

## Package Dependencies

### Dependency Graph
```
directus (main)
├── @directus/api
│   ├── @directus/types
│   ├── @directus/schema
│   ├── @directus/utils
│   ├── @directus/errors
│   ├── @directus/env
│   ├── @directus/storage
│   ├── @directus/data
│   └── @directus/pressure
├── @directus/app
│   ├── @directus/types
│   ├── @directus/utils
│   ├── @directus/composables
│   └── @directus/extensions-sdk
└── @directus/sdk
    ├── @directus/types
    └── @directus/utils
```

### Workspace Protocol
Packages reference each other using `workspace:*`:
```json
{
  "dependencies": {
    "@directus/types": "workspace:*",
    "@directus/utils": "workspace:*"
  }
}
```

## Build System

### Build Configuration

#### Root Package Scripts
```json
{
  "scripts": {
    "build": "pnpm --recursive run build",
    "format": "prettier --cache --check .",
    "lint": "eslint --cache .",
    "lint:style": "stylelint '**/*.{css,scss,vue}'",
    "test": "pnpm --recursive --filter '!tests-blackbox' test",
    "test:blackbox": "rimraf ./dist && pnpm --filter directus deploy --prod dist && pnpm --filter tests-blackbox test"
  }
}
```

### Build Tools

#### API Build (TypeScript)
```json
{
  "scripts": {
    "build": "tsdown --dts",
    "dev": "tsdown --watch"
  }
}
```

**Tools**:
- `tsdown` - Fast TypeScript bundler
- `esbuild` - JavaScript bundler
- Type declaration generation

#### App Build (Vite)
```json
{
  "scripts": {
    "build": "vite build",
    "dev": "vite"
  }
}
```

**Tools**:
- Vite 7.1.12 - Build tool
- Vue plugin for Vite
- SCSS compilation
- Asset optimization

#### SDK Build (Rollup)
```json
{
  "scripts": {
    "build": "rollup -c"
  }
}
```

**Tools**:
- Rollup - Module bundler
- Multiple output formats (ESM, CJS)
- Type declarations

### Build Order
1. **Shared packages** (`@directus/types`, `@directus/utils`, etc.)
2. **Core packages** (`@directus/schema`, `@directus/data`, etc.)
3. **Main packages** (`api`, `app`, `sdk`)
4. **Directus package** (bundles everything)

## Development Workflow

### Local Development Setup

#### 1. Install Dependencies
```bash
# Install pnpm globally
npm install -g pnpm@10

# Install all dependencies
pnpm install
```

#### 2. Build All Packages
```bash
# Build all packages in dependency order
pnpm build
```

#### 3. Start Development Servers
```bash
# Terminal 1: Start API
cd api
pnpm dev

# Terminal 2: Start App
cd app
pnpm dev
```

### Package Development

#### Working on a Shared Package
```bash
# Navigate to package
cd packages/utils

# Watch mode for development
pnpm dev

# Run tests
pnpm test
```

#### Adding a New Package
```bash
# Create package directory
mkdir packages/my-package

# Initialize package.json
cd packages/my-package
pnpm init

# Add to workspace (automatic with pnpm-workspace.yaml)
```

### Testing Strategy

#### Unit Tests
```bash
# Run all unit tests
pnpm test

# Run tests for specific package
pnpm --filter @directus/utils test

# Run with coverage
pnpm test:coverage
```

#### Integration Tests
```bash
# Run blackbox tests
pnpm test:blackbox
```

#### Test Tools
- Vitest 3.2.4 - Test runner
- Happy DOM - DOM implementation
- @vue/test-utils - Vue component testing

### Code Quality

#### Linting
```bash
# ESLint for JavaScript/TypeScript
pnpm lint

# Stylelint for CSS/SCSS
pnpm lint:style
```

#### Formatting
```bash
# Check formatting
pnpm format

# Fix formatting
pnpm format --write
```

#### Configuration Files
- `.eslintrc.json` - ESLint configuration
- `.prettierrc.json` - Prettier configuration
- `.stylelintrc.json` - Stylelint configuration

### Version Management

#### Changesets
Directus uses Changesets for version management:

```bash
# Create a changeset
pnpm changeset

# Version packages
pnpm changeset version

# Publish packages
pnpm changeset publish
```

#### Changeset Files
```markdown
---
"@directus/api": patch
"@directus/app": patch
---

Fixed bug in collection management
```

### Deployment

#### Production Build
```bash
# Build for production
pnpm build

# Deploy to directory
pnpm --filter directus deploy --prod dist
```

#### Docker Build
```dockerfile
# Multi-stage build
FROM node:22-alpine AS builder
WORKDIR /directus
COPY . .
RUN pnpm install
RUN pnpm build

FROM node:22-alpine
COPY --from=builder /directus/dist /directus
CMD ["node", "cli.js", "start"]
```

## Package Manager Features

### pnpm Advantages
1. **Disk Space Efficiency**: Content-addressable storage
2. **Fast Installation**: Parallel downloads and linking
3. **Strict Dependencies**: No phantom dependencies
4. **Monorepo Support**: Built-in workspace support

### Catalog Mode
```yaml
catalog:
  vue: 3.5.24
  typescript: 5.9.3
  vite: 7.1.12
```

**Benefits**:
- Centralized version management
- Consistent dependencies across packages
- Easy version updates

### Overrides
```json
{
  "pnpm": {
    "overrides": {
      "tar-fs": "2.1.4",
      "glob@10.4.5": "10.5.0"
    }
  }
}
```

## Next Steps

- **Build & Deployment**: See [Build & Deployment](./03-build-deployment.md) for detailed build configuration
- **Backend Architecture**: Check [Backend Architecture](./04-backend-architecture.md) for API structure
- **Frontend Overview**: Review [Frontend Overview](./07-frontend-overview.md) for app structure
- **Extension System**: Explore [Extension System](./15-extension-system.md) for customization
