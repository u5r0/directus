# Frontend Overview (app/)

The Directus frontend is a sophisticated Vue 3 application that provides a comprehensive admin interface for content management, user administration, and system configuration.

## Table of Contents
- [Technology Stack](#technology-stack)
- [Application Architecture](#application-architecture)
- [Directory Structure](#directory-structure)
- [Core Systems](#core-systems)
- [Module System](#module-system)
- [Component Architecture](#component-architecture)
- [State Management](#state-management)
- [Routing & Navigation](#routing--navigation)
- [Internationalization](#internationalization)
- [Performance Features](#performance-features)

## Technology Stack

### Core Framework
- **Vue 3.5.24** - Composition API with TypeScript support
- **TypeScript 5.9.3** - Type-safe development
- **Vite 7.1.12** - Fast build tool and dev server
- **Pinia 2.3.1** - Lightweight state management
- **Vue Router 4.6.3** - Client-side routing

### UI & Styling
- **SCSS** - Enhanced CSS with Sass Embedded
- **Custom Component Library** - 60+ v-* components
- **Vue I18n 11.1.12** - Internationalization
- **VueUse 14.0.0** - Composition utilities
- **Unhead** - Document head management

### Rich Content Editing
- **TinyMCE 6.8.6** - WYSIWYG HTML editor
- **Editor.js 2.31.0** - Block-based content editor
- **CodeMirror 5.65.18** - Code editor with syntax highlighting
- **Monaco Editor** - Advanced code editing capabilities

### Data Visualization
- **ApexCharts 4.7.0** - Interactive charts and graphs
- **FullCalendar 6.1.19** - Calendar components
- **MapBox GL 1.13.3** - Interactive maps
- **MapLibre GL 1.15.3** - Open-source mapping

### Development & Testing
- **Vitest 3.2.4** - Unit testing framework
- **Happy DOM** - Lightweight DOM implementation for testing
- **Histoire** - Component documentation and testing
- **Vue DevTools** - Development debugging tools

## Application Architecture

### Entry Point Flow
```
main.ts
├── App.vue (Root Component)
├── Router Setup (Vue Router)
├── Store Setup (Pinia)
├── Component Registration
├── Extension Loading
├── Language Setup
└── Theme Configuration
```

### Core Application Structure
```typescript
// main.ts - Application bootstrap
async function init() {
  console.log(DIRECTUS_LOGO);
  
  // Create Vue app
  const app = createApp(App);
  
  // Setup core services
  app.use(createPinia());
  app.use(createHead());
  app.use(router);
  app.use(i18n);
  
  // Register components and extensions
  registerComponents(app);
  registerViews(app);
  await loadExtensions();
  
  // Mount application
  app.mount('#app');
}
```

## Directory Structure

### Complete App Structure
```
app/src/
├── main.ts                    # Application entry point
├── app.vue                    # Root Vue component
├── router.ts                  # Vue Router configuration
├── api.ts                     # Axios HTTP client
├── sdk.ts                     # Directus SDK client
├── auth.ts                    # Authentication logic
├── hydrate.ts                 # Store hydration system
├── constants.ts               # Application constants
├── events.ts                  # Global event system
├── extensions.ts              # Extension loading system
├── idle.ts                    # Idle detection system
│
├── components/                # Base UI Components (60+)
│   ├── register.ts           # Component registration
│   ├── v-button.vue          # Button component
│   ├── v-table/              # Table component system
│   │   ├── v-table.vue       # Main table component
│   │   ├── table-header.vue  # Table header
│   │   ├── table-row.vue     # Table row
│   │   └── types.ts          # Table type definitions
│   ├── v-form/               # Form system
│   │   ├── v-form.vue        # Main form component
│   │   ├── components/       # Form sub-components
│   │   ├── composables/      # Form composables
│   │   ├── types.ts          # Form type definitions
│   │   └── utils/            # Form utilities
│   ├── v-icon/               # Icon system
│   ├── v-select/             # Select component
│   ├── v-checkbox-tree/      # Tree checkbox
│   ├── v-field-template/     # Field templates
│   ├── v-field-list/         # Field lists
│   └── transition/           # Transition components
│
├── views/                     # Page-level Components
│   ├── register.ts           # View registration
│   ├── private/              # Admin interface views
│   │   ├── index.ts          # Private view exports
│   │   ├── components/       # Private view components
│   │   └── private-view/     # Main private view system
│   ├── public/               # Public views (login, etc.)
│   └── shared/               # Shared item views
│
├── modules/                   # Feature Modules (7 modules)
│   ├── index.ts              # Module registration
│   ├── content/              # Content Management
│   │   ├── index.ts          # Module definition
│   │   ├── routes/           # Module routes
│   │   │   ├── collection.vue # Collection listing
│   │   │   ├── item.vue      # Item editing
│   │   │   └── not-found.vue # 404 handling
│   │   ├── components/       # Module components
│   │   └── navigation.vue    # Module navigation
│   ├── files/                # File Management
│   ├── users/                # User Management
│   ├── settings/             # System Settings
│   ├── activity/             # Activity Logs
│   ├── insights/             # Analytics Dashboard
│   └── visual/               # Visual Content Tools
│
├── stores/                    # Pinia Stores (21 stores)
│   ├── user.ts               # User authentication & profile
│   ├── server.ts             # Server info & capabilities
│   ├── collections.ts        # Collection metadata
│   ├── fields.ts             # Field definitions
│   ├── permissions.ts        # Access control
│   ├── relations.ts          # Relationship mappings
│   ├── settings.ts           # Application settings
│   ├── files.ts              # File management
│   ├── flows.ts              # Workflow definitions
│   ├── insights.ts           # Analytics data
│   ├── notifications.ts      # User notifications
│   ├── presets.ts            # User preferences
│   ├── translations.ts       # Multi-language content
│   ├── extensions.ts         # Extension registry
│   └── requests.ts           # HTTP request tracking
│
├── interfaces/                # Input Interfaces (40+ types)
│   ├── index.ts              # Interface registration
│   ├── input/                # Text input interface
│   ├── input-multiline/      # Textarea interface
│   ├── input-rich-text-html/ # WYSIWYG editor
│   ├── input-rich-text-md/   # Markdown editor
│   ├── input-block-editor/   # Block editor
│   ├── input-code/           # Code editor
│   ├── select-dropdown/      # Dropdown selection
│   ├── select-multiple-*/    # Multi-selection interfaces
│   ├── file*/                # File upload interfaces
│   ├── list-*/               # Relationship interfaces
│   ├── datetime/             # Date/time picker
│   ├── boolean/              # Boolean toggle
│   ├── map/                  # Geographic picker
│   ├── tags/                 # Tag input
│   ├── translations/         # Multi-language input
│   ├── group-*/              # Field grouping
│   ├── presentation-*/       # Presentation elements
│   └── _system/              # System interfaces
│
├── displays/                  # Output Displays (18 types)
│   ├── index.ts              # Display registration
│   ├── raw/                  # Raw value display
│   ├── formatted-value/      # Formatted display
│   ├── formatted-json-value/ # JSON formatting
│   ├── boolean/              # Boolean display
│   ├── datetime/             # Date/time formatting
│   ├── color/                # Color swatch
│   ├── icon/                 # Icon display
│   ├── image/                # Image display
│   ├── file/                 # File information
│   ├── filesize/             # File size formatting
│   ├── mime-type/            # MIME type display
│   ├── related-values/       # Relationship display
│   ├── user/                 # User information
│   ├── collection/           # Collection name
│   ├── labels/               # Tag/label display
│   ├── rating/               # Star rating
│   ├── hash/                 # Masked display
│   └── translations/         # Multi-language display
│
├── layouts/                   # Content Layouts (5 types)
│   ├── index.ts              # Layout registration
│   ├── tabular/              # Table layout
│   ├── cards/                # Card grid layout
│   ├── calendar/             # Calendar layout
│   ├── map/                  # Map layout
│   └── kanban/               # Kanban board layout
│
├── panels/                    # Dashboard Panels (11 types)
│   ├── index.ts              # Panel registration
│   ├── bar-chart/            # Bar chart panel
│   ├── line-chart/           # Line chart panel
│   ├── pie-chart/            # Pie chart panel
│   ├── time-series/          # Time series panel
│   ├── list/                 # Data list panel
│   ├── metric/               # Single metric panel
│   ├── metric-list/          # Multiple metrics panel
│   ├── meter/                # Progress meter panel
│   ├── label/                # Static label panel
│   ├── variable/             # Dynamic variable panel
│   └── relational-variable/  # Related data panel
│
├── operations/                # Workflow Operations (15 types)
│   ├── index.ts              # Operation registration
│   ├── condition/            # Conditional logic
│   ├── exec/                 # Command execution
│   ├── item-create/          # Create items
│   ├── item-read/            # Read items
│   ├── item-update/          # Update items
│   ├── item-delete/          # Delete items
│   ├── json-web-token/       # JWT operations
│   ├── log/                  # Logging operations
│   ├── mail/                 # Email operations
│   ├── notification/         # Notification operations
│   ├── request/              # HTTP requests
│   ├── sleep/                # Delay operations
│   ├── throw-error/          # Error handling
│   └── transform/            # Data transformation
│
├── composables/               # Vue Composables (30+ utilities)
│   ├── use-item/             # Item management
│   ├── use-permissions/      # Permission checking
│   ├── use-alias-fields.ts   # Field aliasing
│   ├── use-clipboard.ts      # Clipboard operations
│   ├── use-dialog-route.ts   # Dialog routing
│   ├── use-edits-guard.ts    # Unsaved changes guard
│   ├── use-extension.ts      # Extension utilities
│   ├── use-field-tree.ts     # Field hierarchy
│   ├── use-flows.ts          # Workflow management
│   ├── use-folders.ts        # Folder operations
│   ├── use-navigation-guard.ts # Route guards
│   ├── use-preset.ts         # User presets
│   ├── use-relation-*.ts     # Relationship utilities
│   ├── use-revision.ts       # Content versioning
│   ├── use-shortcut.ts       # Keyboard shortcuts
│   ├── use-system.ts         # System utilities
│   ├── use-template-data.ts  # Template rendering
│   ├── use-tfa-setup.ts      # Two-factor auth
│   ├── use-theme-configuration.ts # Theme management
│   ├── use-validation-error-details.ts # Validation
│   └── use-versions.ts       # Version management
│
├── directives/                # Vue Directives (6 directives)
│   ├── register.ts           # Directive registration
│   ├── click-outside.ts      # Outside click detection
│   ├── context-menu.ts       # Right-click menus
│   ├── focus.ts              # Focus management
│   ├── input-auto-width.ts   # Auto-sizing inputs
│   ├── markdown.ts           # Markdown rendering
│   └── tooltip.ts            # Tooltip display
│
├── routes/                    # Route Components
│   ├── accept-invite.vue     # Invitation acceptance
│   ├── logout.vue            # Logout handling
│   ├── private-not-found.vue # 404 for private routes
│   ├── tfa-setup.vue         # Two-factor setup
│   ├── login/                # Login flow
│   ├── register/             # Registration flow
│   ├── reset-password/       # Password reset
│   ├── setup/                # Initial setup
│   └── shared/               # Shared item access
│
├── utils/                     # Utility Functions (80+ utilities)
│   ├── geometry/             # Geographic utilities
│   ├── readable-mime-type/   # MIME type handling
│   ├── add-query-to-path.ts  # URL manipulation
│   ├── apply-conditions.ts   # Conditional logic
│   ├── field-utils.ts        # Field operations
│   ├── format-*.ts           # Formatting utilities
│   ├── get-*.ts              # Getter utilities
│   ├── parse-*.ts            # Parsing utilities
│   ├── validate-*.ts         # Validation utilities
│   └── ...                   # Many more utilities
│
├── lang/                      # Internationalization
│   ├── index.ts              # i18n setup
│   ├── available-languages.yaml # Supported languages
│   ├── date-formats.yaml     # Date formatting
│   ├── number-formats.yaml   # Number formatting
│   ├── translations/         # Translation files
│   ├── refresh-current-language.ts # Language refresh
│   └── set-language.ts       # Language switching
│
├── ai/                        # AI Integration
│   ├── components/           # AI UI components
│   ├── composables/          # AI composables
│   ├── stores/               # AI state management
│   ├── types/                # AI type definitions
│   └── models.ts             # AI model definitions
│
├── styles/                    # Global Styles
│   ├── main.scss             # Main stylesheet
│   ├── _base.scss            # Base styles
│   ├── _colors.scss          # Color definitions
│   ├── _fonts.scss           # Font definitions
│   ├── _variables.scss       # SCSS variables
│   ├── _type-styles.scss     # Typography
│   ├── mixins/               # SCSS mixins
│   ├── lib/                  # Style libraries
│   └── themes/               # Theme definitions
│
├── themes/                    # Theme System
│   └── register.ts           # Theme registration
│
├── types/                     # TypeScript Definitions
│   ├── activity.ts           # Activity types
│   ├── collections.ts        # Collection types
│   ├── error.ts              # Error types
│   ├── folders.ts            # Folder types
│   ├── insights.ts           # Analytics types
│   ├── interfaces.ts         # Interface types
│   ├── login.ts              # Authentication types
│   ├── notifications.ts      # Notification types
│   ├── panels.ts             # Panel types
│   ├── permissions.ts        # Permission types
│   ├── revisions.ts          # Revision types
│   └── user.ts               # User types
│
└── assets/                    # Static Assets
    ├── fonts/                # Font files
    ├── avatar-placeholder.svg # Default avatar
    ├── logo-dark.svg         # Dark theme logo
    ├── logo.svg              # Default logo
    ├── sprite.svg            # Icon sprite
    └── readme.md             # Asset documentation
```

## Core Systems

### 1. Component Registration System
```typescript
// components/register.ts
export function registerComponents(app: App): void {
  // Base components (60+)
  app.component('VButton', VButton);
  app.component('VTable', VTable);
  app.component('VForm', VForm);
  // ... all other components
  
  // Utility components
  app.component('RenderDisplay', RenderDisplay);
  app.component('RenderTemplate', RenderTemplate);
  app.component('DrawerBatch', DrawerBatch);
}
```

### 2. Extension Loading System
```typescript
// extensions.ts
export async function loadExtensions(): Promise<void> {
  // Load internal extensions
  const interfaces = getInternalInterfaces();
  const displays = getInternalDisplays();
  const layouts = getInternalLayouts();
  const panels = getInternalPanels();
  
  // Register with Vue app
  registerInterfaces(interfaces, app);
  registerDisplays(displays, app);
  registerLayouts(layouts, app);
  registerPanels(panels, app);
}
```

### 3. Store Hydration System
```typescript
// hydrate.ts
export async function hydrate(): Promise<void> {
  const stores = [
    useUserStore,
    useCollectionsStore,
    useFieldsStore,
    usePermissionsStore,
    // ... all other stores
  ];
  
  // Hydrate user store first (required by others)
  await userStore.hydrate();
  
  // Parallel hydration of remaining stores
  await Promise.all(stores.map(store => store.hydrate?.()));
  
  // Load extensions after stores are ready
  await onHydrateExtensions();
}
```

## Module System

### Module Architecture
Each module is a self-contained feature with its own:
- **Routes** - URL patterns and components
- **Components** - Module-specific UI elements
- **Navigation** - Menu items and breadcrumbs
- **Permissions** - Access control integration

### Content Module (Primary Module)
```typescript
// modules/content/index.ts
export default {
  id: 'content',
  name: '$t:content',
  icon: 'box',
  routes: [
    {
      name: 'content-collection',
      path: '/:collection',
      component: () => import('./routes/collection.vue'),
      props: true,
    },
    {
      name: 'content-item',
      path: '/:collection/:primaryKey',
      component: () => import('./routes/item.vue'),
      props: true,
    },
  ],
} as ModuleConfig;
```

### Module Registration
```typescript
// modules/index.ts
export function registerModules(modules: ModuleConfig[]) {
  const onHydrateModules = async () => {
    // Check permissions for each module
    const allowedModules = await Promise.all(
      modules.map(async (module) => {
        if (!module.preRegisterCheck) return module;
        
        const allowed = await module.preRegisterCheck(
          userStore.currentUser,
          permissionsStore.permissions
        );
        
        return allowed ? module : null;
      })
    );
    
    // Register routes for allowed modules
    for (const module of allowedModules.filter(Boolean)) {
      router.addRoute({
        name: module.id,
        path: `/${module.id}`,
        component: RouterPass,
        children: module.routes,
      });
    }
  };
}
```

## Component Architecture

### Base Component Hierarchy
```
VApp (app.vue)
├── Router (Vue Router)
├── PrivateView (Admin Interface)
│   ├── Navigation (Module Navigation)
│   ├── Header (Actions, Search, Breadcrumbs)
│   ├── Main Content
│   │   ├── Layouts (tabular, cards, calendar, map, kanban)
│   │   ├── Forms (VForm with dynamic interfaces)
│   │   └── Tables (VTable with displays)
│   └── Sidebar (Details, Options, Flows)
└── PublicView (Login, Setup, Shared)
```

### Component Communication Patterns

#### 1. Props Down, Events Up
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

#### 2. Provide/Inject for Deep Hierarchies
```typescript
// Parent provides context
provide('values', values);
provide('live-preview-active', livePreviewActive);

// Child components inject
const values = inject('values');
const livePreviewActive = inject('live-preview-active');
```

#### 3. Store-Based Communication
```typescript
// Components access shared state
const userStore = useUserStore();
const permissionsStore = usePermissionsStore();
const fieldsStore = useFieldsStore();
```

## State Management

### Store Architecture (21 Stores)
The application uses specialized Pinia stores for different concerns:

#### Core Stores
- **user** - Authentication, profile, preferences
- **server** - Server capabilities and project info
- **permissions** - Role-based access control
- **collections** - Collection metadata and operations
- **fields** - Field definitions and validation
- **relations** - Relationship mappings

#### Feature Stores
- **files** - File management and upload
- **flows** - Workflow definitions and execution
- **insights** - Analytics and dashboard data
- **notifications** - User notifications
- **presets** - User preferences and filters
- **settings** - Application configuration
- **translations** - Multi-language content
- **extensions** - Extension registry
- **requests** - HTTP request tracking

### Store Interaction Pattern
```typescript
// Store definition with API integration
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const collections = ref<Collection[]>([]);
  
  async function hydrate() {
    const response = await api.get('/collections');
    collections.value = response.data.data.map(prepareCollectionForApp);
  }
  
  async function updateCollection(collection: string, updates: DeepPartial<Collection>) {
    await api.patch(`/collections/${collection}`, updates);
    await hydrate(); // Re-fetch data
    notify({ title: i18n.global.t('update_collection_success') });
  }
  
  return { collections, hydrate, updateCollection };
});
```

## Routing & Navigation

### Route Structure
```typescript
// router.ts
export const defaultRoutes: RouteRecordRaw[] = [
  {
    path: '/',
    redirect: () => {
      const serverStore = useServerStore();
      return serverStore.info.setupCompleted ? '/login' : '/setup';
    },
  },
  {
    name: 'login',
    path: '/login',
    component: LoginRoute,
    meta: { public: true },
  },
  {
    name: 'content',
    path: '/content',
    component: () => import('./modules/content/routes/collection.vue'),
  },
  // ... module routes added dynamically
];
```

### Route Guards
```typescript
// Authentication guard
router.beforeEach(async (to, from) => {
  if (to.meta.public) return true;
  
  const userStore = useUserStore();
  if (!userStore.currentUser) {
    return '/login';
  }
  
  // Check permissions for protected routes
  const permissionsStore = usePermissionsStore();
  if (!permissionsStore.hasPermission(to.meta.collection, 'read')) {
    return '/unauthorized';
  }
});
```

## Internationalization

### Language System
```typescript
// lang/index.ts
export const i18n = createI18n({
  legacy: false,
  locale: 'en-US',
  fallbackLocale: 'en-US',
  messages: {}, // Loaded dynamically
});

// Language switching
export async function setLanguage(lang: string) {
  if (!i18n.global.availableLocales.includes(lang)) {
    await loadLanguageAsync(lang);
  }
  
  i18n.global.locale.value = lang;
  document.documentElement.setAttribute('lang', lang);
}
```

### Translation Loading
```typescript
// Dynamic translation loading
async function loadLanguageAsync(lang: string) {
  const messages = await import(`./translations/${lang}.yaml`);
  i18n.global.setLocaleMessage(lang, messages.default);
}
```

## Performance Features

### 1. Lazy Loading
```typescript
// Async component loading
const LazyComponent = defineAsyncComponent(() => 
  import('./components/HeavyComponent.vue')
);
```

### 2. Virtual Scrolling
```vue
<VirtualList
  :items="items"
  :item-height="48"
  :buffer="10"
  @scroll="handleScroll"
/>
```

### 3. Request Optimization
```typescript
// Request queuing and deduplication
export let requestQueue: PQueue = new PQueue({
  concurrency: 5,
  intervalCap: 5,
  interval: 500,
});
```

### 4. Component Optimization
- **Computed Properties** for reactive data transformation
- **Watchers** for side effects and API calls
- **Refs** for DOM manipulation and component references
- **Provide/Inject** for efficient prop drilling avoidance

## Next Steps

- **UI Components**: See [UI Component System](./08-ui-components.md) for detailed component documentation
- **State Management**: Check [State Management](./12-state-management.md) for store patterns
- **API Integration**: Review [API Integration](./13-api-integration.md) for backend communication
- **Extension Development**: Explore [Extension System](./15-extension-system.md) for customization
