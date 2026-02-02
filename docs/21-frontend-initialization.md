# Frontend Initialization & Bootstrap

This document details how the Directus Vue 3 application initializes, bootstraps, and prepares for user interaction.

## Table of Contents
- [Application Entry Point](#application-entry-point)
- [Initialization Sequence](#initialization-sequence)
- [Plugin Registration](#plugin-registration)
- [Extension Loading](#extension-loading)
- [Store Hydration](#store-hydration)
- [Router Setup](#router-setup)
- [Error Handling](#error-handling)

## Application Entry Point

### main.ts - Bootstrap Process

The application starts in `app/src/main.ts` with a carefully orchestrated initialization sequence:

```typescript
// app/src/main.ts
import { createApp } from 'vue';
import { createHead } from '@unhead/vue';
import { createPinia } from 'pinia';
import App from './app.vue';

async function init() {
  console.log(DIRECTUS_LOGO);
  console.time('🕓 Application Loaded');

  const app = createApp(App);

  // 1. Setup i18n first
  app.use(i18n);
  
  // 2. Setup Pinia state management
  app.use(createPinia());
  
  // 3. Setup Unhead for meta tags
  app.use(createHead());

  // 4. Register global error handler
  app.config.errorHandler = (err, vm, info) => {
    const source = getVueComponentName(vm);
    console.warn(`[app-${source}-error] ${info}`);
    console.warn(err);
    return false;
  };

  // 5. Register directives
  registerDirectives(app);
  
  // 6. Register components
  registerComponents(app);
  
  // 7. Register views
  registerViews(app);

  // 8. Load extensions (async)
  await loadExtensions();
  registerExtensions(app);

  // 9. Add router AFTER extensions
  app.use(router);

  // 10. Mount to DOM
  app.mount('#app');

  console.timeEnd('🕓 Application Loaded');
}
```

### Why This Order Matters

1. **i18n First**: Translation system must be available before any component renders
2. **Pinia Second**: State management needed by components and composables
3. **Unhead Third**: Meta tag management for SEO and branding
4. **Error Handler**: Catch errors early in the initialization process
5. **Directives/Components/Views**: Register global functionality
6. **Extensions**: Load custom extensions before routing
7. **Router Last**: Ensures all routes (including extension routes) are registered

## Root Component (app.vue)

### Component Structure

```vue
<script setup lang="ts">
import { useAppStore } from '@directus/stores';
import { ThemeProvider } from '@directus/themes';
import { useHead } from '@unhead/vue';

const appStore = useAppStore();
const serverStore = useServerStore();
const userStore = useUserStore();

const { darkMode, themeDark, themeLight } = useThemeConfiguration();
const { hydrating } = toRefs(appStore);

// Dynamic favicon based on project settings
useHead({
  title: 'Directus',
  titleTemplate: '%s · %projectName',
  templateParams: {
    projectName: computed(() => serverStore.info?.project?.project_name ?? 'Directus'),
  },
  htmlAttrs: computed(() => ({
    lang: userStore.language,
    dir: userStore.textDirection,
  })),
  link: computed(() => {
    let href: string;
    if (serverStore.info?.project?.public_favicon) {
      href = getAssetUrl(serverStore.info.project.public_favicon);
    } else if (serverStore.info?.project?.project_color) {
      href = generateFavicon(serverStore.info.project.project_color);
    } else {
      href = '/favicon.ico';
    }
    return [{ rel: 'icon', href }];
  }),
});

// Start idle tracking for session management
onMounted(() => startIdleTracking());
onUnmounted(() => stopIdleTracking());
</script>

<template>
  <ThemeProvider
    :dark-mode="darkMode"
    :theme-light="themeLight"
    :theme-dark="themeDark"
  />

  <div id="directus">
    <!-- Loading state during hydration -->
    <Transition name="fade">
      <div v-if="hydrating" class="hydrating">
        <VProgressCircular indeterminate />
      </div>
    </Transition>

    <!-- Error state -->
    <VInfo v-if="error" type="danger" :title="$t('unexpected_error')">
      {{ $t('unexpected_error_copy') }}
      <template #append>
        <VError :error="error" />
        <VButton @click="reload">{{ $t('reload_page') }}</VButton>
      </template>
    </VInfo>

    <!-- Main application -->
    <RouterView v-else-if="!hydrating" />
  </div>

  <!-- Custom CSS injection -->
  <Teleport to="#custom-css">{{ customCSS }}</Teleport>
</template>
```

### Key Features

1. **Theme Management**: Dynamic light/dark mode with custom overrides
2. **Meta Tag Management**: Dynamic favicon, title, language, and text direction
3. **Loading States**: Shows spinner during initial data hydration
4. **Error Boundaries**: Catches and displays unexpected errors
5. **Custom CSS**: Allows project-level CSS customization
6. **Idle Tracking**: Monitors user activity for session management

## Plugin Registration

### 1. Directives Registration

```typescript
// app/src/directives/register.ts
export function registerDirectives(app: App) {
  app.directive('tooltip', tooltip);
  app.directive('focus', focus);
  app.directive('click-outside', clickOutside);
  // ... more directives
}
```

### 2. Components Registration

```typescript
// app/src/components/register.ts
export function registerComponents(app: App) {
  // Register all v-* components globally
  app.component('VButton', VButton);
  app.component('VInput', VInput);
  app.component('VTable', VTable);
  // ... 145+ components
}
```

### 3. Views Registration

```typescript
// app/src/views/register.ts
export function registerViews(app: App) {
  app.component('PrivateView', PrivateView);
  app.component('PublicView', PublicView);
  // ... view components
}
```

## Extension Loading

### Extension Discovery

```typescript
// app/src/extensions/index.ts
export async function loadExtensions() {
  try {
    // Fetch available extensions from API
    const response = await api.get('/extensions');
    const extensions = response.data.data;

    // Load each extension type
    await Promise.all([
      loadInterfaces(extensions.interfaces),
      loadDisplays(extensions.displays),
      loadLayouts(extensions.layouts),
      loadModules(extensions.modules),
      loadPanels(extensions.panels),
      loadOperations(extensions.operations),
    ]);
  } catch (error) {
    console.error('Failed to load extensions:', error);
  }
}
```

### Extension Registration

```typescript
export function registerExtensions(app: App) {
  // Register extension components
  for (const [id, extension] of Object.entries(interfaces)) {
    app.component(`interface-${id}`, extension);
  }

  for (const [id, extension] of Object.entries(displays)) {
    app.component(`display-${id}`, extension);
  }

  // ... register other extension types
}
```

## Store Hydration

### Hydration Sequence

The application hydrates stores in a specific order to ensure dependencies are met:

```typescript
// Executed after user authentication
async function hydrateStores() {
  const serverStore = useServerStore();
  const userStore = useUserStore();
  const permissionsStore = usePermissionsStore();
  const collectionsStore = useCollectionsStore();
  const fieldsStore = useFieldsStore();
  const relationsStore = useRelationsStore();

  // 1. Server info (no dependencies)
  await serverStore.hydrate();

  // 2. User info (depends on server)
  await userStore.hydrate();

  // 3. Permissions (depends on user)
  await permissionsStore.hydrate();

  // 4. Collections, fields, relations (can run in parallel)
  await Promise.all([
    collectionsStore.hydrate(),
    fieldsStore.hydrate(),
    relationsStore.hydrate(),
  ]);

  // 5. Extension-specific stores
  await Promise.all([
    useFlowsStore().hydrate(),
    useInsightsStore().hydrate(),
    usePresetsStore().hydrate(),
  ]);
}
```

### Hydration States

```typescript
// app/src/stores/app.ts
export const useAppStore = defineStore('appStore', {
  state: () => ({
    hydrating: true,
    hydrated: false,
    error: null,
  }),
  actions: {
    async hydrate() {
      this.hydrating = true;
      try {
        await hydrateStores();
        this.hydrated = true;
      } catch (error) {
        this.error = error;
      } finally {
        this.hydrating = false;
      }
    },
  },
});
```

## Router Setup

### Router Configuration

```typescript
// app/src/router.ts
import { createRouter, createWebHistory } from 'vue-router';

export const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      redirect: '/content',
    },
    {
      path: '/login',
      component: () => import('./views/public/login.vue'),
    },
    {
      path: '/content',
      component: () => import('./views/private/content.vue'),
      meta: { requiresAuth: true },
    },
    // ... more routes
  ],
});
```

### Navigation Guards

```typescript
// Authentication guard
router.beforeEach(async (to, from) => {
  const userStore = useUserStore();
  const appStore = useAppStore();

  // Check if route requires authentication
  if (to.meta.requiresAuth && !userStore.currentUser) {
    return { path: '/login', query: { redirect: to.fullPath } };
  }

  // Hydrate stores on first navigation
  if (!appStore.hydrated && userStore.currentUser) {
    await appStore.hydrate();
  }

  // Track page view
  if (userStore.currentUser) {
    await userStore.trackPage(to);
  }
});
```

## Error Handling

### Global Error Handler

```typescript
app.config.errorHandler = (err, vm, info) => {
  const source = getVueComponentName(vm);
  console.warn(`[app-${source}-error] ${info}`);
  console.warn(err);
  
  // Don't propagate to prevent app crash
  return false;
};
```

### Component Error Boundaries

```vue
<!-- VErrorBoundary component -->
<template>
  <slot v-if="!error" />
  <div v-else class="error-boundary">
    <VError :error="error" />
    <VButton @click="reset">{{ $t('try_again') }}</VButton>
  </div>
</template>

<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue';

const error = ref(null);

onErrorCaptured((err) => {
  error.value = err;
  return false; // Prevent propagation
});

const reset = () => {
  error.value = null;
};
</script>
```

### API Error Handling

```typescript
// app/src/api.ts
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    // Handle 401 Unauthorized
    if (error.response?.status === 401) {
      const userStore = useUserStore();
      await userStore.dehydrate();
      router.push('/login');
    }

    // Handle 403 Forbidden
    if (error.response?.status === 403) {
      notify({
        title: i18n.global.t('not_allowed'),
        type: 'error',
      });
    }

    return Promise.reject(error);
  }
);
```

## Performance Optimizations

### 1. Lazy Loading

```typescript
// Routes are lazy-loaded
const routes = [
  {
    path: '/content',
    component: () => import('./views/private/content.vue'),
  },
];

// Extensions are lazy-loaded
async function loadInterfaces(interfaces) {
  for (const id of interfaces) {
    const module = await import(`./interfaces/${id}/index.vue`);
    registerInterface(id, module.default);
  }
}
```

### 2. Request Queuing

```typescript
// app/src/api.ts
export const requestQueue = new PQueue({
  concurrency: 5,        // Max 5 concurrent requests
  intervalCap: 5,        // Max 5 requests per interval
  interval: 500,         // 500ms interval
  carryoverConcurrencyCount: true,
});
```

### 3. Component Registration

Only frequently used components are registered globally. Others are imported locally:

```typescript
// Global registration (145+ components)
app.component('VButton', VButton);
app.component('VInput', VInput);

// Local registration (in specific components)
import VRareComponent from '@/components/v-rare-component.vue';
```

## Initialization Timeline

```
0ms    - main.ts starts
10ms   - Vue app created
15ms   - i18n loaded
20ms   - Pinia created
25ms   - Unhead created
30ms   - Directives registered
35ms   - Components registered (145+)
40ms   - Views registered
50ms   - Extensions loading starts
200ms  - Extensions loaded
210ms  - Router added
220ms  - App mounted
230ms  - Router navigation triggered
240ms  - Auth check
250ms  - Store hydration starts
500ms  - Stores hydrated
510ms  - First render complete
```

## Debugging Initialization

### Console Logging

The initialization process logs key milestones:

```
🐰 Starting Directus...
🕓 Application Loaded: 523ms
✨ Project Information
  Environment: production
```

### Performance Monitoring

```typescript
// Track initialization performance
console.time('🕓 Application Loaded');
// ... initialization code
console.timeEnd('🕓 Application Loaded');
```

### Error Tracking

```typescript
// All errors are logged with context
app.config.errorHandler = (err, vm, info) => {
  const source = getVueComponentName(vm);
  console.warn(`[app-${source}-error] ${info}`);
  console.warn(err);
};
```

## Next Steps

- [Frontend Data Flow](./22-frontend-data-flow.md) - How data moves through the application
- [State Management Deep Dive](./12-state-management.md) - Pinia stores in detail
- [API Integration](./13-api-integration.md) - HTTP client and SDK usage
- [Component System](./08-ui-components.md) - UI component library

## Related Files

- `app/src/main.ts` - Application entry point
- `app/src/app.vue` - Root component
- `app/src/router.ts` - Router configuration
- `app/src/api.ts` - HTTP client setup
- `app/src/sdk.ts` - SDK client setup
- `app/src/extensions/index.ts` - Extension loading
