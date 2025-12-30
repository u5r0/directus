# State Management

Directus uses Pinia for state management with 21 specialized stores that handle different aspects of the application. This document covers the complete state management architecture and data flow patterns.

## Table of Contents
- [Store Architecture](#store-architecture)
- [Core Stores](#core-stores)
- [Feature Stores](#feature-stores)
- [Store Hydration System](#store-hydration-system)
- [API Integration](#api-integration)
- [Real-time Updates](#real-time-updates)
- [Store Communication Patterns](#store-communication-patterns)
- [Performance Optimizations](#performance-optimizations)

## Store Architecture

### Pinia Configuration
```typescript
// main.ts
import { createPinia } from 'pinia';

const app = createApp(App);
const pinia = createPinia();

app.use(pinia);
```

### Store Structure Pattern
```typescript
// Example store structure
export const useExampleStore = defineStore('exampleStore', () => {
  // State
  const items = ref<Item[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  
  // Getters (computed)
  const sortedItems = computed(() => 
    items.value.sort((a, b) => a.name.localeCompare(b.name))
  );
  
  const itemCount = computed(() => items.value.length);
  
  // Actions
  async function hydrate() {
    loading.value = true;
    try {
      const response = await api.get('/items');
      items.value = response.data.data;
    } catch (err) {
      error.value = err;
    } finally {
      loading.value = false;
    }
  }
  
  async function createItem(data: Partial<Item>) {
    const response = await api.post('/items', data);
    items.value.push(response.data.data);
    return response.data.data;
  }
  
  function dehydrate() {
    items.value = [];
    loading.value = false;
    error.value = null;
  }
  
  return {
    // State
    items,
    loading,
    error,
    // Getters
    sortedItems,
    itemCount,
    // Actions
    hydrate,
    createItem,
    dehydrate,
  };
});
```

## Core Stores

### User Store (useUserStore)
Manages user authentication, profile, and preferences.

```typescript
// stores/user.ts
export const useUserStore = defineStore('userStore', () => {
  const currentUser = ref<AppUser | ShareUser | null>(null);
  const loading = ref(false);
  const error = ref(null);
  
  // Computed properties
  const fullName = computed(() => {
    const user = unref(currentUser);
    if (user === null || 'share' in user) return null;
    return userName(user);
  });
  
  const isAdmin = computed(() => 
    unref(currentUser)?.admin_access === true || false
  );
  
  const language = computed(() => {
    const user = unref(currentUser);
    
    if (user && 'language' in user && user.language !== null) {
      return user.language;
    }
    
    if (serverStore.info?.project?.default_language) {
      return serverStore.info.project.default_language;
    }
    
    return 'en-US';
  });
  
  const textDirection = computed(() => {
    const user = unref(currentUser);
    const lang = unref(language);
    
    const savedDir = (user && 'text_direction' in user && user.text_direction) ?? 'auto';
    
    let dir: 'ltr' | 'rtl';
    
    if (savedDir === 'ltr' || savedDir === 'rtl') {
      dir = savedDir;
    } else {
      dir = isIn(lang, RTL_LANGUAGES) ? 'rtl' : 'ltr';
    }
    
    return dir;
  });
  
  // Actions
  async function hydrate() {
    loading.value = true;
    
    try {
      const fields = ['*', 'role.id'];
      
      const [{ data: user }, { data: globals }, { data: roles }] = await Promise.all([
        api.get('/users/me', { params: { fields } }),
        api.get('/policies/me/globals'),
        api.get('/roles/me', { params: { fields: ['id'] } }),
      ]);
      
      currentUser.value = {
        ...user.data,
        ...(user.data?.avatar != null ? { avatar: { id: user.data?.avatar } } : {}),
        ...globals.data,
        roles: roles.data,
      };
    } catch (error: any) {
      error.value = error;
    } finally {
      loading.value = false;
    }
  }
  
  async function trackPage(to: RouteLocationNormalized) {
    if (to.path.endsWith('/preview')) return;
    
    await api.patch('/users/me/track/page', {
      last_page: to.fullPath,
    }, { measureLatency: true });
    
    const user = unref(currentUser);
    if (user && !('share' in user)) {
      user.last_page = to.fullPath;
    }
  }
  
  return {
    currentUser,
    loading,
    error,
    fullName,
    isAdmin,
    language,
    textDirection,
    hydrate,
    dehydrate,
    hydrateAdditionalFields,
    trackPage,
  };
});
```

### Collections Store (useCollectionsStore)
Manages collection metadata, translations, and CRUD operations.

```typescript
// stores/collections.ts
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const aiStore = useAiStore();
  
  // AI integration for automatic updates
  aiStore.onSystemToolResult(async (toolName) => {
    if (toolName === 'collections') {
      await hydrate();
    }
  });
  
  const collections = ref<Collection[]>([]);
  
  // Computed collections with different filters
  const sortedCollections = computed(() => 
    flattenGroupedCollections(collections.value)
  );
  
  const systemCollections = computed(() =>
    sortedCollections.value.filter(({ collection }) => 
      isSystemCollection(collection)
    )
  );
  
  const allCollections = computed(() =>
    sortedCollections.value.filter(({ collection }) => 
      isSystemCollection(collection) === false
    )
  );
  
  const configuredCollections = computed(() => 
    allCollections.value.filter((collection) => collection.meta)
  );
  
  const visibleCollections = computed(() =>
    configuredCollections.value.filter((collection) => 
      collection.meta?.hidden !== true
    )
  );
  
  const databaseCollections = computed(() => 
    allCollections.value.filter((collection) => collection.schema)
  );
  
  // Actions
  async function hydrate() {
    const response = await api.get<any>(`/collections`);
    const rawCollections: CollectionRaw[] = response.data.data;
    collections.value = rawCollections.map(prepareCollectionForApp);
  }
  
  function prepareCollectionForApp(collection: CollectionRaw): Collection {
    const icon = collection.meta?.icon || 'database';
    const color = collection.meta?.color;
    let name = formatTitle(collection.collection);
    const type = getCollectionType(collection);
    
    // Handle translations
    if (collection.meta && Array.isArray(collection.meta.translations)) {
      for (const { language, translation, singular, plural } of collection.meta.translations) {
        i18n.global.mergeLocaleMessage(language, {
          ...(translation ? {
            collection_names: {
              [collection.collection]: getLiteralInterpolatedTranslation(translation),
            },
          } : {}),
          ...(singular ? {
            collection_names_singular: {
              [collection.collection]: getLiteralInterpolatedTranslation(singular),
            },
          } : {}),
          ...(plural ? {
            collection_names_plural: {
              [collection.collection]: getLiteralInterpolatedTranslation(plural),
            },
          } : {}),
        });
      }
    }
    
    if (i18n.global.te(`collection_names.${collection.collection}`)) {
      name = i18n.global.t(`collection_names.${collection.collection}`);
    }
    
    return { ...collection, name, type, icon, color };
  }
  
  async function upsertCollection(collection: string, values: DeepPartial<Collection & { fields: Field[] }>) {
    const existing = getCollection(collection);
    const rawValues = omit(values, ['name', 'type', 'icon', 'color']);
    
    if (existing) {
      if (isEqual(existing, values)) return;
      
      const updatedCollectionResponse = await api.patch<{ data: CollectionRaw }>(
        `/collections/${collection}`,
        rawValues,
      );
      
      collections.value = collections.value.map((existingCollection: Collection) => {
        if (existingCollection.collection === collection) {
          return prepareCollectionForApp(updatedCollectionResponse.data.data);
        }
        return existingCollection;
      });
    } else {
      const createdCollectionResponse = await api.post<{ data: CollectionRaw }>('/collections', rawValues);
      collections.value = [...collections.value, prepareCollectionForApp(createdCollectionResponse.data.data)];
    }
  }
  
  function getCollection(collectionKey: string): Collection | null {
    return collections.value.find((collection) => collection.collection === collectionKey) || null;
  }
  
  return {
    collections,
    sortedCollections,
    allCollections,
    visibleCollections,
    configuredCollections,
    databaseCollections,
    systemCollections,
    hydrate,
    dehydrate,
    upsertCollection,
    updateCollection,
    deleteCollection,
    getCollection,
  };
});
```

### Fields Store (useFieldsStore)
Manages field definitions, validation, and relationships.

```typescript
// stores/fields.ts
export const useFieldsStore = defineStore('fieldsStore', () => {
  const aiStore = useAiStore();
  
  // AI integration for field updates
  aiStore.onSystemToolResult(async (toolName) => {
    if (toolName === 'collections' || toolName === 'fields') {
      await hydrate();
    }
  });
  
  const fields = ref<Field[]>([]);
  
  // Special fake field for file thumbnails
  const fakeFilesField: Field = {
    collection: 'directus_files',
    field: '$thumbnail',
    schema: null,
    name: '$thumbnail',
    type: 'integer',
    meta: {
      id: -1,
      collection: 'directus_files',
      field: '$thumbnail',
      sort: null,
      special: null,
      interface: null,
      options: null,
      display: 'file',
      display_options: null,
      hidden: false,
      translations: null,
      readonly: true,
      width: 'full',
      group: null,
      note: null,
      required: false,
      conditions: null,
      validation: null,
      validation_message: null,
    },
  };
  
  async function hydrate(options?: { skipTranslation?: boolean }) {
    const fieldsResponse = await api.get<any>(`/fields`);
    const fieldsRaw: FieldRaw[] = fieldsResponse.data.data;
    fields.value = [...fieldsRaw.map(parseField), fakeFilesField];
    
    if (options?.skipTranslation !== true) {
      translateFields();
    }
  }
  
  function parseField(field: FieldRaw): Field {
    let name = formatTitle(field.field);
    
    // Handle field translations
    if (field.meta && Array.isArray(field.meta.translations)) {
      for (const { language, translation } of field.meta.translations) {
        i18n.global.mergeLocaleMessage(language, {
          ...(translation ? {
            fields: {
              [field.collection]: {
                [field.field]: getLiteralInterpolatedTranslation(translation),
              },
            },
          } : {}),
        });
      }
    }
    
    if (i18n.global.te(`fields.${field.collection}.${field.field}`)) {
      name = i18n.global.t(`fields.${field.collection}.${field.field}`);
    }
    
    return { ...field, name };
  }
  
  function getFieldsForCollection(collection: string): Field[] {
    return orderBy(
      fields.value.filter((field) => field.collection === collection),
      [(field) => field.meta?.system === true, (field) => (field.meta?.sort ? Number(field.meta?.sort) : null)],
      ['desc', 'asc'],
    );
  }
  
  function getField(collection: string, fieldKey: string): Field | null {
    if (fieldKey.includes('.')) {
      return getRelationalField(collection, fieldKey) || null;
    } else {
      return fields.value.find((field) => field.collection === collection && field.field === fieldKey) || null;
    }
  }
  
  // Optimistic updates for field operations
  async function updateField(collectionKey: string, fieldKey: string, updates: DeepPartial<Field>) {
    const stateClone = [...fields.value];
    
    // Update locally first for immediate UI feedback
    fields.value = fields.value.map((field) => {
      if (field.collection === collectionKey && field.field === fieldKey) {
        return merge({}, field, updates);
      }
      return field;
    });
    
    try {
      const response = await api.patch<any>(`/fields/${collectionKey}/${fieldKey}`, updates);
      
      // Update with server response
      fields.value = fields.value.map((field) => {
        if (field.collection === collectionKey && field.field === fieldKey) {
          return parseField(response.data.data);
        }
        return field;
      });
    } catch (error) {
      // Revert on error
      fields.value = stateClone;
      throw error;
    }
  }
  
  return {
    fields,
    hydrate,
    dehydrate,
    parseField,
    translateFields,
    upsertField,
    createField,
    updateField,
    updateFields,
    deleteField,
    getPrimaryKeyFieldForCollection,
    getFieldsForCollection,
    getField,
    getFieldGroupChildren,
  };
});
```

### Permissions Store (usePermissionsStore)
Manages role-based access control and field-level permissions.

```typescript
// stores/permissions.ts
export const usePermissionsStore = defineStore({
  id: 'permissionsStore',
  state: () => ({
    permissions: {} as CollectionAccess,
  }),
  actions: {
    async hydrate() {
      const userStore = useUserStore();
      
      const response = await api.get('/permissions/me');
      
      // Extract dynamic variable fields for additional user data
      const fields = getNestedDynamicVariableFields(response.data.data);
      
      if (fields.length > 0) {
        await userStore.hydrateAdditionalFields(fields);
      }
      
      this.permissions = mapValues(
        response.data.data as CollectionAccess,
        (collectionPermission: CollectionPermission) => {
          Object.values(collectionPermission).forEach((actionPermission) => {
            if (actionPermission.presets) {
              actionPermission.presets = parsePreset(actionPermission.presets);
            }
          });
          
          return collectionPermission;
        },
      ) as CollectionAccess;
      
      function getNestedDynamicVariableFields(rawPermissions: Record<string, CollectionPermission>) {
        const fields = new Set<string>();
        
        const checkDynamicVariable = (value: string) => {
          if (typeof value !== 'string') return;
          
          if (value.startsWith('$CURRENT_USER.')) {
            fields.add(value.replace('$CURRENT_USER.', ''));
          } else if (value.startsWith('$CURRENT_ROLE.')) {
            fields.add(value.replace('$CURRENT_ROLE.', 'role.'));
          }
        };
        
        Object.values(rawPermissions).forEach((collectionPermission: CollectionPermission) => {
          Object.values(collectionPermission).forEach((actionPermission) => {
            deepMap(actionPermission.presets, checkDynamicVariable);
          });
        });
        
        return Array.from(fields);
      }
    },
    
    dehydrate() {
      this.$reset();
    },
    
    getPermission(collection: string, action: PermissionsAction) {
      return this.permissions[collection]?.[action] ?? null;
    },
    
    hasPermission(collection: string, action: PermissionsAction) {
      const userStore = useUserStore();
      
      if (userStore.isAdmin) return true;
      
      return (this.getPermission(collection, action)?.access ?? 'none') !== 'none';
    },
  },
});
```

## Feature Stores

### Files Store (useFilesStore)
```typescript
export const useFilesStore = defineStore('filesStore', () => {
  const files = ref<DirectusFile[]>([]);
  const uploading = ref<Map<string, UploadProgress>>(new Map());
  
  async function uploadFile(file: File, folder?: string) {
    const uploadId = nanoid();
    
    uploading.value.set(uploadId, {
      file,
      progress: 0,
      status: 'uploading',
    });
    
    try {
      const formData = new FormData();
      formData.append('file', file);
      if (folder) formData.append('folder', folder);
      
      const response = await api.post('/files', formData, {
        onUploadProgress: (progressEvent) => {
          const progress = Math.round((progressEvent.loaded * 100) / progressEvent.total);
          
          uploading.value.set(uploadId, {
            ...uploading.value.get(uploadId)!,
            progress,
          });
        },
      });
      
      files.value.push(response.data.data);
      uploading.value.delete(uploadId);
      
      return response.data.data;
    } catch (error) {
      uploading.value.set(uploadId, {
        ...uploading.value.get(uploadId)!,
        status: 'error',
        error,
      });
      
      throw error;
    }
  }
  
  return {
    files,
    uploading,
    uploadFile,
    deleteFile,
    updateFile,
  };
});
```

### Flows Store (useFlowsStore)
```typescript
export const useFlowsStore = defineStore('flowsStore', () => {
  const flows = ref<Flow[]>([]);
  const executions = ref<Map<string, FlowExecution>>(new Map());
  
  async function executeFlow(flowId: string, data?: any) {
    const executionId = nanoid();
    
    executions.value.set(executionId, {
      id: executionId,
      flowId,
      status: 'running',
      startTime: Date.now(),
      data,
    });
    
    try {
      const response = await api.post(`/flows/${flowId}/execute`, { data });
      
      executions.value.set(executionId, {
        ...executions.value.get(executionId)!,
        status: 'completed',
        endTime: Date.now(),
        result: response.data.data,
      });
      
      return response.data.data;
    } catch (error) {
      executions.value.set(executionId, {
        ...executions.value.get(executionId)!,
        status: 'failed',
        endTime: Date.now(),
        error,
      });
      
      throw error;
    }
  }
  
  return {
    flows,
    executions,
    executeFlow,
    getManualFlows,
    getFlowsForCollection,
  };
});
```

### Notifications Store (useNotificationsStore)
```typescript
export const useNotificationsStore = defineStore('notificationsStore', () => {
  const notifications = ref<Notification[]>([]);
  const unreadCount = computed(() => 
    notifications.value.filter(n => !n.read).length
  );
  
  async function markAsRead(id: string) {
    // Optimistic update
    const notification = notifications.value.find(n => n.id === id);
    if (notification) {
      notification.read = true;
    }
    
    try {
      await api.patch(`/notifications/${id}`, { read: true });
    } catch (error) {
      // Revert on error
      if (notification) {
        notification.read = false;
      }
      throw error;
    }
  }
  
  function addNotification(notification: Omit<Notification, 'id' | 'timestamp'>) {
    notifications.value.unshift({
      ...notification,
      id: nanoid(),
      timestamp: Date.now(),
      read: false,
    });
  }
  
  return {
    notifications,
    unreadCount,
    markAsRead,
    markAllAsRead,
    addNotification,
    removeNotification,
  };
});
```

## Store Hydration System

### Hydration Process
```typescript
// hydrate.ts
export async function hydrate(): Promise<void> {
  const stores = useStores([
    useCollectionsStore,
    useFieldsStore,
    useUserStore,
    useRequestsStore,
    usePresetsStore,
    useSettingsStore,
    useServerStore,
    useRelationsStore,
    usePermissionsStore,
    useInsightsStore,
    useFlowsStore,
    useNotificationsStore,
    useAiStore,
  ]);
  
  const appStore = useAppStore();
  const userStore = useUserStore();
  const permissionsStore = usePermissionsStore();
  const fieldsStore = useFieldsStore();
  
  if (appStore.hydrated) return;
  if (appStore.hydrating) return;
  
  appStore.hydrating = true;
  
  try {
    // User store must be hydrated first (required by others)
    await userStore.hydrate();
    
    const currentUser = userStore.currentUser;
    
    if (currentUser?.app_access) {
      // Parallel hydration of core stores
      await Promise.all([
        permissionsStore.hydrate(),
        fieldsStore.hydrate({ skipTranslation: true })
      ]);
      
      // Hydrate remaining stores
      const hydratedStores = ['userStore', 'permissionsStore', 'fieldsStore', 'serverStore'];
      await Promise.all(
        stores
          .filter(({ $id }) => !hydratedStores.includes($id))
          .map((store) => store.hydrate?.())
      );
      
      await onHydrateExtensions();
    }
    
    await setLanguage(userStore.language);
    
    appStore.basemap = getBasemapSources()[0].name;
  } catch (error: any) {
    appStore.error = error;
  } finally {
    appStore.hydrating = false;
  }
  
  appStore.hydrated = true;
}

export async function dehydrate(stores = useStores()): Promise<void> {
  const appStore = useAppStore();
  
  if (appStore.hydrated === false) return;
  
  for (const store of stores) {
    await store.dehydrate?.();
  }
  
  await onDehydrateExtensions();
  
  appStore.hydrated = false;
}
```

### Store Dependencies
```typescript
// Store dependency graph
const storeDependencies = {
  userStore: [], // No dependencies
  serverStore: [],
  permissionsStore: ['userStore'],
  fieldsStore: ['userStore'],
  collectionsStore: ['userStore', 'fieldsStore'],
  relationsStore: ['collectionsStore', 'fieldsStore'],
  filesStore: ['userStore', 'permissionsStore'],
  flowsStore: ['userStore', 'permissionsStore'],
  // ... other stores
};
```

## API Integration

### Request Tracking Store
```typescript
// stores/requests.ts
export const useRequestsStore = defineStore('requestsStore', () => {
  const requests = ref<Map<string, RequestInfo>>(new Map());
  
  const activeRequests = computed(() => 
    Array.from(requests.value.values()).filter(r => r.status === 'pending')
  );
  
  const isLoading = computed(() => activeRequests.value.length > 0);
  
  function startRequest(): string {
    const id = nanoid();
    
    requests.value.set(id, {
      id,
      status: 'pending',
      startTime: Date.now(),
    });
    
    return id;
  }
  
  function endRequest(id: string) {
    const request = requests.value.get(id);
    if (request) {
      request.status = 'completed';
      request.endTime = Date.now();
      request.duration = request.endTime - request.startTime;
    }
  }
  
  return {
    requests,
    activeRequests,
    isLoading,
    startRequest,
    endRequest,
  };
});
```

### Store API Integration Pattern
```typescript
// Common pattern for API integration in stores
export const useExampleStore = defineStore('exampleStore', () => {
  const items = ref<Item[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  
  async function fetchItems() {
    loading.value = true;
    error.value = null;
    
    try {
      const response = await api.get('/items');
      items.value = response.data.data;
    } catch (err) {
      error.value = err;
      console.error('Failed to fetch items:', err);
    } finally {
      loading.value = false;
    }
  }
  
  async function createItem(data: CreateItemData) {
    try {
      const response = await api.post('/items', data);
      const newItem = response.data.data;
      
      // Optimistic update
      items.value.push(newItem);
      
      // Show success notification
      notify({
        title: i18n.global.t('item_created_successfully'),
        type: 'success',
      });
      
      return newItem;
    } catch (error) {
      // Handle error
      unexpectedError(error);
      throw error;
    }
  }
  
  return {
    items,
    loading,
    error,
    fetchItems,
    createItem,
  };
});
```

## Real-time Updates

### WebSocket Integration
```typescript
// Real-time store updates via WebSocket
export const useRealtimeStore = defineStore('realtimeStore', () => {
  const connections = ref<Map<string, WebSocketConnection>>(new Map());
  
  function subscribe(collection: string, callback: (data: any) => void) {
    const subscription = sdk.subscribe('items', {
      collection,
      query: { fields: ['*'] }
    }, (data) => {
      // Update relevant stores based on changes
      if (data.event === 'create') {
        handleItemCreated(collection, data.data);
      } else if (data.event === 'update') {
        handleItemUpdated(collection, data.data);
      } else if (data.event === 'delete') {
        handleItemDeleted(collection, data.keys);
      }
      
      callback(data);
    });
    
    connections.value.set(`${collection}:items`, subscription);
  }
  
  function handleItemCreated(collection: string, item: any) {
    // Update collections store if needed
    const collectionsStore = useCollectionsStore();
    const collectionInfo = collectionsStore.getCollection(collection);
    
    if (collectionInfo) {
      // Trigger refresh of related data
      emitter.emit('item-created', { collection, item });
    }
  }
  
  return {
    subscribe,
    unsubscribe,
    connections,
  };
});
```

### Store Synchronization
```typescript
// Cross-store synchronization
export function setupStoreSync() {
  const collectionsStore = useCollectionsStore();
  const fieldsStore = useFieldsStore();
  const relationsStore = useRelationsStore();
  
  // When collections change, update related stores
  watch(
    () => collectionsStore.collections,
    async (newCollections, oldCollections) => {
      const addedCollections = newCollections.filter(
        nc => !oldCollections?.find(oc => oc.collection === nc.collection)
      );
      
      const removedCollections = oldCollections?.filter(
        oc => !newCollections.find(nc => nc.collection === oc.collection)
      ) || [];
      
      if (addedCollections.length > 0 || removedCollections.length > 0) {
        // Refresh fields and relations
        await Promise.all([
          fieldsStore.hydrate(),
          relationsStore.hydrate(),
        ]);
      }
    },
    { deep: true }
  );
}
```

## Store Communication Patterns

### Event-Driven Updates
```typescript
// Event-based store communication
export const useEventStore = defineStore('eventStore', () => {
  const events = ref<AppEvent[]>([]);
  
  function emitEvent(event: AppEvent) {
    events.value.push(event);
    
    // Notify other stores
    switch (event.type) {
      case 'collection-updated':
        emitter.emit('collection-updated', event.data);
        break;
      case 'user-updated':
        emitter.emit('user-updated', event.data);
        break;
      case 'permissions-changed':
        emitter.emit('permissions-changed', event.data);
        break;
    }
  }
  
  return {
    events,
    emitEvent,
  };
});

// Store listening to events
export const useListenerStore = defineStore('listenerStore', () => {
  onMounted(() => {
    emitter.on('collection-updated', async (collection) => {
      // Refresh data when collection changes
      await hydrate();
    });
    
    emitter.on('user-updated', (user) => {
      // Update user-related data
      updateUserData(user);
    });
  });
});
```

### Computed Cross-Store Dependencies
```typescript
// Cross-store computed properties
export const useDerivedStore = defineStore('derivedStore', () => {
  const userStore = useUserStore();
  const collectionsStore = useCollectionsStore();
  const permissionsStore = usePermissionsStore();
  
  const availableCollections = computed(() => {
    if (!userStore.currentUser) return [];
    
    return collectionsStore.visibleCollections.filter(collection => 
      permissionsStore.hasPermission(collection.collection, 'read')
    );
  });
  
  const userCollectionPreferences = computed(() => {
    const user = userStore.currentUser;
    if (!user || 'share' in user) return {};
    
    return user.preferences?.collections || {};
  });
  
  return {
    availableCollections,
    userCollectionPreferences,
  };
});
```

## Performance Optimizations

### Lazy Store Loading
```typescript
// Lazy store initialization
export const useLazyStore = defineStore('lazyStore', () => {
  const data = ref<any[]>([]);
  const initialized = ref(false);
  
  async function ensureInitialized() {
    if (initialized.value) return;
    
    await hydrate();
    initialized.value = true;
  }
  
  async function getData() {
    await ensureInitialized();
    return data.value;
  }
  
  return {
    data: readonly(data),
    getData,
    hydrate,
  };
});
```

### Store Memoization
```typescript
// Memoized store computations
export const useMemoizedStore = defineStore('memoizedStore', () => {
  const items = ref<Item[]>([]);
  
  // Memoized expensive computation
  const processedItems = computed(() => {
    return items.value.map(item => ({
      ...item,
      computed: expensiveComputation(item),
    }));
  });
  
  // Cached API calls
  const cache = new Map<string, any>();
  
  async function getCachedData(key: string) {
    if (cache.has(key)) {
      return cache.get(key);
    }
    
    const data = await api.get(`/data/${key}`);
    cache.set(key, data);
    
    return data;
  }
  
  return {
    items,
    processedItems,
    getCachedData,
  };
});
```

### Batch Updates
```typescript
// Batch store updates for performance
export const useBatchStore = defineStore('batchStore', () => {
  const items = ref<Item[]>([]);
  const pendingUpdates = ref<Map<string, Partial<Item>>>(new Map());
  
  function queueUpdate(id: string, updates: Partial<Item>) {
    pendingUpdates.value.set(id, {
      ...pendingUpdates.value.get(id),
      ...updates,
    });
    
    // Debounced batch processing
    debouncedProcessUpdates();
  }
  
  const debouncedProcessUpdates = debounce(async () => {
    const updates = Array.from(pendingUpdates.value.entries());
    pendingUpdates.value.clear();
    
    try {
      await api.patch('/items/batch', { updates });
      
      // Apply updates to local state
      for (const [id, update] of updates) {
        const item = items.value.find(i => i.id === id);
        if (item) {
          Object.assign(item, update);
        }
      }
    } catch (error) {
      // Re-queue failed updates
      for (const [id, update] of updates) {
        pendingUpdates.value.set(id, update);
      }
      throw error;
    }
  }, 500);
  
  return {
    items,
    queueUpdate,
  };
});
```

## Next Steps

- **API Integration**: See [API Integration](./13-api-integration.md) for detailed communication patterns
- **Real-time Features**: Check [Real-time Features](./16-realtime-features.md) for WebSocket integration
- **Performance**: Review [Performance & Scaling](./19-performance-scaling.md) for optimization strategies
- **Extension Development**: Explore [Extension System](./15-extension-system.md) for custom store integration
