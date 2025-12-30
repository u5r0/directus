# Database Layer

This document covers the database architecture, schema inspection, migrations, and data access patterns in Directus.

## Overview

Directus uses **Knex.js** as its query builder and supports multiple database systems through a unified interface. The `@directus/schema` package provides database-agnostic schema inspection capabilities.

## Supported Databases

### Database Clients

```typescript
// Supported database systems
- PostgreSQL (with PostGIS for geometry)
- MySQL 8 & 5.7 (with utf8mb4 support)
- MariaDB 11.4
- SQLite3 (with foreign key support)
- Oracle DB (18 XE)
- MS SQL Server (2019+)
- CockroachDB (with serial normalization)
```

### Database Configuration

```bash
# Environment variables
DB_CLIENT=pg|mysql|sqlite3|oracledb|mssql|cockroachdb
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=directus
DB_USER=directus
DB_PASSWORD=password
DB_CONNECTION_STRING=  # Alternative to individual settings

# Connection pool settings
DB_POOL_MIN=0
DB_POOL_MAX=10
DB_POOL_IDLE_TIMEOUT=30000
DB_POOL_ACQUIRE_TIMEOUT=30000

# Exclude specific tables
DB_EXCLUDE_TABLES=table1,table2
```

## Database Initialization

### Knex Setup (`api/src/database/index.ts`)

```typescript
export function getDatabase(): Knex {
  const env = useEnv();
  
  const knexConfig: Knex.Config = {
    client,
    version,
    searchPath,
    connection: connectionString || connectionConfig,
    pool: poolConfig,
  };
  
  // Database-specific configuration
  if (client === 'sqlite3') {
    knexConfig.useNullAsDefault = true;
    poolConfig.afterCreate = (conn, callback) => {
      conn.run('PRAGMA foreign_keys = ON');
      callback(null, conn);
    };
  }
  
  if (client === 'cockroachdb') {
    poolConfig.afterCreate = (conn, callback) => {
      conn.query('SET serial_normalization = "sql_sequence"');
      conn.query('SET default_int_size = 4');
      callback(null, conn);
    };
  }
  
  database = knex(knexConfig);
  
  // Query logging and metrics
  database
    .on('query', ({ __knexUid }) => {
      times.set(__knexUid, performance.now());
    })
    .on('query-response', (_response, queryInfo) => {
      const delta = performance.now() - times.get(queryInfo.__knexUid);
      metrics?.getDatabaseResponseMetric()?.observe(delta);
      logger.trace(`[${delta.toFixed(3)}ms] ${queryInfo.sql}`);
    });
  
  return database;
}
```

## Schema Inspection

### Schema Inspector (`@directus/schema`)

The `@directus/schema` package provides database-agnostic schema inspection:

```typescript
import { createInspector } from '@directus/schema';

// Create inspector for current database
const inspector = createInspector(knex);

// Supported inspectors by database
- MySQLSchemaInspector
- PostgresSchemaInspector
- CockroachDBSchemaInspector
- SqliteSchemaInspector
- OracleDBSchemaInspector
- MSSQLSchemaInspector
```

### Schema Overview Structure

```typescript
interface SchemaOverview {
  collections: {
    [collection: string]: {
      collection: string;
      primary: string;
      singleton: boolean;
      note: string | null;
      sortField: string | null;
      accountability: 'all' | 'activity' | null;
      fields: {
        [field: string]: {
          field: string;
          defaultValue: any;
          nullable: boolean;
          generated: boolean;
          type: string;        // Local type (string, integer, etc.)
          dbType: string;      // Database-specific type
          precision: number | null;
          scale: number | null;
          special: string[];   // Special field types (m2o, o2m, etc.)
          note: string | null;
          validation: Filter | null;
          alias: boolean;
          searchable: boolean;
        };
      };
    };
  };
  relations: Relation[];
}
```

### Getting Schema (`api/src/utils/get-schema.ts`)

```typescript
export async function getSchema(options?: {
  database?: Knex;
  bypassCache?: boolean;
}): Promise<SchemaOverview> {
  // Check cache first
  if (!options?.bypassCache && env['CACHE_SCHEMA'] !== false) {
    const cached = getMemorySchemaCache();
    if (cached) return cached;
  }
  
  const database = options?.database || getDatabase();
  const schemaInspector = createInspector(database);
  
  const schema = await getDatabaseSchema(database, schemaInspector);
  
  setMemorySchemaCache(schema);
  return schema;
}

async function getDatabaseSchema(
  database: Knex,
  schemaInspector: SchemaInspector
): Promise<SchemaOverview> {
  // Get database schema overview
  const schemaOverview = await schemaInspector.overview();
  
  // Load collection metadata from directus_collections
  const collections = await database
    .select('collection', 'singleton', 'note', 'sort_field', 'accountability')
    .from('directus_collections');
  
  // Load field metadata from directus_fields
  const fields = await database
    .select('id', 'collection', 'field', 'special', 'note', 'validation')
    .from('directus_fields');
  
  // Merge database schema with Directus metadata
  // ...
  
  return result;
}
```

## Core System Collections

Directus uses system collections (prefixed with `directus_`) to manage its own data:

### User Management
```sql
directus_users          -- User accounts with authentication
directus_roles          -- User roles
directus_permissions    -- Permission rules
directus_policies       -- Access policies
directus_access         -- Policy-role-user associations
directus_sessions       -- Active user sessions
```

### Content Management
```sql
directus_collections    -- Collection metadata
directus_fields         -- Field definitions
directus_relations      -- Relationship definitions
directus_activity       -- Activity logs
directus_revisions      -- Item version history
directus_comments       -- Item comments
directus_presets        -- User presets/filters
```

### File Management
```sql
directus_files          -- File metadata
directus_folders        -- File folder structure
```

### Workflow & Automation
```sql
directus_flows          -- Workflow definitions
directus_operations     -- Workflow operations
directus_webhooks       -- Webhook definitions
```

### UI & UX
```sql
directus_dashboards     -- Dashboard definitions
directus_panels         -- Dashboard panels
directus_notifications  -- User notifications
directus_shares         -- Shared item links
directus_translations   -- Translation strings
```

### System Configuration
```sql
directus_settings       -- System settings
directus_migrations     -- Migration tracking
directus_extensions     -- Extension registry
directus_versions       -- Content versions
```

## Migration System

### Migration Structure (`api/src/database/migrations/`)

Migrations are versioned files with up/down functions:

```typescript
// Format: {version}{letter}-{name}.ts
// Example: 20240101A-add-user-fields.ts

import type { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('my_table', (table) => {
    table.uuid('id').primary();
    table.string('name');
    table.timestamps(true, true);
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTable('my_table');
}
```

### Running Migrations (`api/src/database/migrations/run.ts`)

```typescript
export default async function run(
  database: Knex,
  direction: 'up' | 'down' | 'latest',
  log = true
): Promise<void> {
  // Load migration files from core and custom paths
  let migrationFiles = await fse.readdir(__dirname);
  const customMigrationsPath = path.resolve(env['MIGRATIONS_PATH']);
  let customMigrationFiles = await fse.readdir(customMigrationsPath);
  
  // Get completed migrations from database
  const completedMigrations = await database
    .select('*')
    .from('directus_migrations')
    .orderBy('version');
  
  // Execute migrations
  if (direction === 'latest') {
    for (const migration of migrations) {
      if (!migration.completed) {
        const { up } = await import(migration.file);
        await up(database);
        await database
          .insert({ version: migration.version, name: migration.name })
          .into('directus_migrations');
      }
    }
  }
  
  await flushCaches(true);
}
```

### Custom Migrations

Place custom migrations in the configured migrations path:

```bash
# Environment variable
MIGRATIONS_PATH=./migrations

# Migration file naming
{version}-{name}.js
# Example: 20240101-add-custom-table.js
```

## Database Helpers

### Helper System (`api/src/database/helpers/`)

Database-specific helpers provide abstraction for different SQL dialects:

```typescript
export function getHelpers(database: Knex) {
  const client = getDatabaseClient(database);
  
  return {
    date: new dateHelpers[client](database),
    st: new geometryHelpers[client](database),
    schema: new schemaHelpers[client](database),
    sequence: new sequenceHelpers[client](database),
    number: new numberHelpers[client](database),
    capabilities: new capabilitiesHelpers[client](database),
  };
}
```

### Helper Categories

**Date Helpers** - Date/time operations
```typescript
helpers.date.parse(value)
helpers.date.format(column)
helpers.date.year(column)
helpers.date.month(column)
```

**Geometry Helpers (ST)** - Spatial operations
```typescript
helpers.st.supported()  // Check PostGIS/Spatialite support
helpers.st.fromGeoJSON(geojson)
helpers.st.asGeoJSON(column)
helpers.st.intersects(a, b)
```

**Schema Helpers** - Schema operations
```typescript
helpers.schema.getDatabaseSize()
helpers.schema.hasTable(table)
helpers.schema.hasColumn(table, column)
```

**Sequence Helpers** - Auto-increment operations
```typescript
helpers.sequence.nextval(sequence)
helpers.sequence.currval(sequence)
```

## Query Building

### AST-Based Query System

Directus uses an Abstract Syntax Tree (AST) approach for complex queries:

```typescript
// 1. Convert query to AST
const ast = await getAstFromQuery(collection, query, schema, {
  accountability,
  knex: database,
});

// 2. Process AST (apply permissions, filters)
const processedAst = await processAst(ast, accountability, context);

// 3. Execute AST
const result = await runAst(processedAst, schema, {
  knex: database,
  accountability,
});
```

### Query Structure

```typescript
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
  group?: string[];
  aggregate?: Aggregate;
}
```

## Field Types

### Supported Field Types

```typescript
// Text types
'string', 'text', 'uuid'

// Numeric types
'integer', 'bigInteger', 'float', 'decimal'

// Boolean
'boolean'

// Date/Time
'date', 'time', 'datetime', 'timestamp'

// JSON
'json'

// Geometry (with PostGIS/Spatialite)
'geometry'

// Special types
'hash'  // For passwords
'csv'   // Comma-separated values
'alias' // Virtual fields
```

### Special Field Types

```typescript
// Relationship types
'm2o'  // Many-to-One
'o2m'  // One-to-Many
'm2m'  // Many-to-Many
'm2a'  // Many-to-Any (polymorphic)

// File types
'file'
'files'

// Group types
'group'
'alias'
'no-data'  // Presentation-only fields
```

## Relationship Types

### Many-to-One (M2O)
```typescript
// Foreign key relationship
{
  collection: 'articles',
  field: 'author',
  related_collection: 'users',
  meta: {
    one_field: null,
    junction_field: null,
  },
  schema: {
    foreign_key_column: 'author',
    foreign_key_table: 'articles',
    constraint_name: 'articles_author_foreign',
  }
}
```

### One-to-Many (O2M)
```typescript
// Reverse of M2O
{
  collection: 'users',
  field: 'articles',
  related_collection: 'articles',
  meta: {
    one_field: 'author',
    junction_field: null,
  }
}
```

### Many-to-Many (M2M)
```typescript
// Junction table relationship
{
  collection: 'articles',
  field: 'tags',
  related_collection: 'tags',
  meta: {
    one_field: null,
    junction_field: 'tags_id',
    one_collection_field: 'articles_id',
    one_allowed_collections: null,
  },
  schema: {
    table: 'articles_tags',
    column: 'articles_id',
    foreign_key_column: 'tags_id',
  }
}
```

### Many-to-Any (M2A)
```typescript
// Polymorphic relationship
{
  collection: 'comments',
  field: 'item',
  related_collection: null,
  meta: {
    one_allowed_collections: ['articles', 'pages'],
    one_collection_field: 'collection',
    one_deselect_action: 'nullify',
  }
}
```

## Database Features

### Connection Pooling

```typescript
// Pool configuration
DB_POOL_MIN=0
DB_POOL_MAX=10
DB_POOL_IDLE_TIMEOUT=30000
DB_POOL_ACQUIRE_TIMEOUT=30000
```

### Query Logging

```typescript
// Automatic query logging with performance metrics
database.on('query-response', (_response, queryInfo) => {
  const delta = performance.now() - startTime;
  logger.trace(`[${delta.toFixed(3)}ms] ${queryInfo.sql}`);
  metrics?.getDatabaseResponseMetric()?.observe(delta);
});
```

### Charset Validation

```typescript
// MySQL charset validation
async function validateDatabaseCharset(database: Knex) {
  if (getDatabaseClient(database) === 'mysql') {
    const { collation } = await database
      .select(database.raw(`@@collation_database as collation`))
      .first();
    
    // Check tables and columns for collation mismatches
    // Warn about inconsistencies
  }
}
```

### Database Extensions

```typescript
// PostGIS for PostgreSQL
await helpers.st.supported();  // Check if PostGIS is installed

// Spatialite for SQLite
// Geometry support with Spatialite extension

// Full-text search
// MySQL: FULLTEXT indexes
// PostgreSQL: tsvector and GIN indexes
// SQLite: FTS5 extension
```

## Performance Optimization

### Schema Caching

```bash
# Enable schema caching
CACHE_SCHEMA=true
CACHE_SCHEMA_MAX_ITERATIONS=10
CACHE_SCHEMA_SYNC_TIMEOUT=5000
```

### Query Optimization

- **Field selection**: Only request needed fields
- **Limit/offset**: Paginate large result sets
- **Indexes**: Create indexes on frequently queried columns
- **Connection pooling**: Reuse database connections

### Database-Specific Optimizations

**PostgreSQL**
- Use JSONB for JSON data
- Enable PostGIS for geometry
- Use tsvector for full-text search

**MySQL**
- Use utf8mb4 collation
- Enable InnoDB for transactions
- Use FULLTEXT indexes for search

**SQLite**
- Enable foreign keys
- Use WAL mode for concurrency
- Enable FTS5 for full-text search

## Next Steps

- [Authentication & Security](./06-auth-security.md) - Authentication and authorization
- [Backend Architecture](./04-backend-architecture.md) - API structure and services
- [Package Interactions](./20-package-interactions.md) - How packages work together
