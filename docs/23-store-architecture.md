# Store Architecture & State Management

This document provides a comprehensive guide to Pinia stores in the Directus frontend application.

## Table of Contents
- [Store Overview](#store-overview)
- [Store Categories](#store-categories)
- [Core Stores](#core-stores)
- [Store Patterns](#store-patterns)
- [Store Dependencies](#store-dependencies)
- [Best Practices](#best-practices)

## Store Overview

Directus uses **Pinia** for state management with **11 stores** (not 24 as previously documented):

```typescript
// Actual stores in app/src/stores/
1. collections.ts      - Collection metadata
2. extensions.ts       - Extension registry
3. fields.ts          - Field definitions
4. files.ts           - File management
5. flows.ts           - Workflow definitions
6. insights.ts        - Dashboard insights
7. notifications.ts   - User notifications
8. permissions.ts     - Access control
9. presets.ts         - User preferences
10. relations.ts      - Relationship definitions
11. requests.ts       - HTTP request tracking
12. server.ts         - Server information
13. settings.ts       - System settings
14. translations.ts   - i18n translations
15. user.ts           - Current user state
```

## Store Categories

### 1. Core System Stores
- **server** - Server info, project settings
- **user** - Current user, authentication
- **permissions** - Access control, policies

### 2. Schema Stores
- **collections** - Collection metadata
- **fields** - Field definitions
- **relations** - Relationship mappings

### 3. Feature Stores
- **files** - File management
- **flows** - Workflow automation
- **insights** - Analytics dashboards
- **notifications** - User notifications

### 4. UI State Stores
- **presets** - User preferences, filters
- **requests** - Loading states
- **settings** - App configuration

### 5. Extension Stores
- **extensions** - Extension registry
- **translations** - i18n strings

## Core Stores

### User Store

```typescript
// app/src/stores/user.ts
export const useUserStore = defineStore('userStore', () => {
  const serverStore = useServerStore();

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
  const hydrate = async () => {
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
  };

  const dehydrate = async () => {
    currentUser.value = null;
    loading.value = false;
    error.value = null;
  };

  const hydrateAdditionalFields = async (fields: string[]) => {
    try {
      const { data } = await api.get('/users/me', { params: { fields } });
      currentUser.value = merge({}, unref(currentUser), data.data);
    } catch {
      // Do nothing
    }
  };

  const trackPage = async (to: RouteLocationNormalized) => {
    if (to.path.endsWith('/preview')) {
      return;
    }

    await api.patch(
      '/users/me/track/page',
      { last_page: to.fullPath },
      { measureLatency: true } as RequestConfig,
    );

    const user = unref(currentUser);
    if (user && !('share' in user)) {
      user.last_page = to.fullPath;
    }
  };

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

### Collections Store

```typescript
// app/src/stores/collections.ts
export const useCollectionsStore = defineStore('collectionsStore', () => {
  const aiStore = useAiStore();

  // React to AI tool results
  aiStore.onSystemToolResult(async (toolName) => {
    if (toolName === 'collections') {
      await hydrate();
    }
  });

  const collections = ref<Collection[]>([]);

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

  const crudSafeSystemCollections = computed(() =>
    systemCollections.value.filter((collection) => 
      !COLLECTIONS_DENY_LIST.includes(collection.collection)
    )
  );

  async function hydrate() {
    const response = await api.get<any>('/collections');
    const rawCollections: CollectionRaw[] = response.data.data;
    collections.value = rawCollections.map(prepareCollectionForApp);
  }

  async function dehydrate() {
    collections.value = [];
  }

  function prepareCollectionForApp(collection: CollectionRaw): Collection {
    const icon = collection.meta?.icon || 'database';
    const color = collection.meta?.color;
    let name = formatTitle(collection.collection);
    const type = getCollectionType(collection);

    // Handle translations
    const localesToKeep =
      collection.meta && !isNil(collection.meta.translations) && Array.isArray(collection.meta.translations)
        ? collection.meta.translations.map((translation) => translation.language)
        : [];

    for (const locale of i18n.global.availableLocales) {
      if (i18n.global.te(`collection_names.${collection.collection}`, locale) && !localesToKeep.includes(locale)) {
        i18n.global.mergeLocaleMessage(locale, { 
          collection_names: { [collection.collection]: undefined } 
        });
      }
    }

    if (collection.meta && Array.isArray(collection.meta.translations)) {
      for (const { language, translation, singular, plural } of collection.meta.translations) {
        i18n.global.mergeLocaleMessage(language, {
          ...(translation
            ? {
                collection_names: {
                  [collection.collection]: getLiteralInterpolatedTranslation(translation),
                },
              }
            : {}),
          ...(singular
            ? {
                collection_names_singular: {
                  [collection.collection]: getLiteralInterpolatedTranslation(singular),
                },
              }
            : {}),
          ...(plural
            ? {
                collection_names_plural: {
                  [collection.collection]: getLiteralInterpolatedTranslation(plural),
                },
              }
            : {}),
        });
      }
    }

    if (i18n.global.te(`collection_names.${collection.collection}`)) {
      name = i18n.global.t(`collection_names.${collection.collection}`);
    }

    return {
      ...collection,
      name,
      type,
      icon,
      color,
    };
  }

  function translateCollections() {
    collections.value = collections.value.map((collection) => {
      if (i18n.global.te(`collection_names.${collection.collection}`)) {
        collection.name = i18n.global.t(`collection_names.${collection.collection}`);
      }
      return collection;
    });
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

  async function updateCollection(collection: string, updates: DeepPartial<Collection>) {
    try {
      await api.patch(`/collections/${collection}`, updates);
      await hydrate();

      notify({
        title: i18n.global.t('update_collection_success'),
      });
    } catch (error) {
      unexpectedError(error);
    }
  }

  async function deleteCollection(collection: string) {
    const relationsStore = useRelationsStore();

    try {
      await api.delete(`/collections/${collection}`);
      await Promise.all([hydrate(), relationsStore.hydrate()]);

      notify({
        title: i18n.global.t('delete_collection_success'),
      });
    } catch (error) {
      unexpectedError(error);
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
    crudSafeSystemCollections,
    hydrate,
    dehydrate,
    prepareCollectionForApp,
    translateCollections,
    upsertCollection,
    updateCollection,
    deleteCollection,
    getCollection,
  };
});
```

### Permissions Store

```typescript
// app/src/stores/permissions.ts
export const usePermissionsStore = defineStore({
  id: 'permissionsStore',
  state: () => ({
    permissions: {} as CollectionAccess,
  }),
  actions: {
    async hydrate() {
      const userStore = useUserStore();

      const response = await api.get('/permissions/me');

      // Extract dynamic variable fields
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

### Fields Store

```typescript
// app/src/stores/fields.ts
export const useFieldsStore = defineStore('fieldsStore', () => {
  const aiStore = useAiStore();

  aiStore.onSystemToolResult(async (toolName) => {
    if (toolName === 'collections' || toolName === 'fields') {
      await hydrate();
    }
  });

  const fields = ref<Field[]>([]);

  async function hydrate(options?: HydrateOptions) {
    const fieldsResponse = await api.get<any>('/fields');
    const fieldsRaw: FieldRaw[] = fieldsResponse.data.data;
    fields.value = [...fieldsRaw.map(parseField), fakeFilesField];
    if (options?.skipTranslation !== true) translateFields();
  }

  async function dehydrate() {
    fields.value = [];
  }

  function parseField(field: FieldRaw): Field {
    let name = formatTitle(field.field);

    // Handle translations
    const localesToKeep =
      field.meta && !isNil(field.meta.translations) && Array.isArray(field.meta.translations)
        ? field.meta.translations.map((translation) => translation.language)
        : [];

    for (const locale of i18n.global.availableLocales) {
      if (
        i18n.global.te(`fields.${field.collection}.${field.field}`, locale) &&
        !localesToKeep.includes(locale) &&
        !field.meta?.system
      ) {
        i18n.global.mergeLocaleMessage(locale, { 
          fields: { [field.collection]: { [field.field]: undefined } } 
        });
      }
    }

    if (field.meta && Array.isArray(field.meta.translations)) {
      for (const { language, translation } of field.meta.translations) {
        i18n.global.mergeLocaleMessage(language, {
          ...(translation
            ? {
                fields: {
                  [field.collection]: {
                    [field.field]: getLiteralInterpolatedTranslation(translation),
                  },
                },
              }
            : {}),
        });
      }
    }

    if (i18n.global.te(`fields.${field.collection}.${field.field}`)) {
      name = i18n.global.t(`fields.${field.collection}.${field.field}`);
    }

    return {
      ...field,
      name,
    };
  }

  function translateFields() {
    fields.value = fields.value.map((field) => {
      const translations: Partial<Field> = {};

      if (i18n.global.te(`fields.${field.collection}.${field.field}`)) {
        translations.name = i18n.global.t(`fields.${field.collection}.${field.field}`);
      }

      if (field.meta) {
        translations.meta = {} as Field['meta'];
      }

      if (field.meta?.note) translations.meta!.note = translateLiteral(field.meta.note);
      if (field.meta?.options) translations.meta!.options = translate(field.meta.options);
      if (field.meta?.display_options) translations.meta!.display_options = translate(field.meta.display_options);

      if (field.meta?.validation_message) {
        translations.meta!.validation_message = translateLiteral(field.meta.validation_message);
      }

      if (!isEmpty(translations)) {
        return merge({}, field, translations);
      }

      return field;
    });
  }

  async function updateField(collectionKey: string, fieldKey: string, updates: DeepPartial<Field>) {
    const stateClone = [...fields.value];

    // Optimistic update
    fields.value = fields.value.map((field) => {
      if (field.collection === collectionKey && field.field === fieldKey) {
        return merge({}, field, updates);
      }
      return field;
    });

    try {
      const response = await api.patch<any>(`/fields/${collectionKey}/${fieldKey}`, updates);

      fields.value = fields.value.map((field) => {
        if (field.collection === collectionKey && field.field === fieldKey) {
          return parseField(response.data.data);
        }
        return field;
      });
    } catch (error) {
      fields.value = stateClone;
      throw error;
    }
  }

  function getPrimaryKeyFieldForCollection(collection: string): Field | null {
    const primaryKeyField = fields.value.find(
      (field) => field.collection === collection && field.schema?.is_primary_key === true,
    );

    return primaryKeyField ?? null;
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

  return {
    fields,
    hydrate,
    dehydrate,
    parseField,
    translateFields,
    updateField,
    getPrimaryKeyFieldForCollection,
    getFieldsForCollection,
    getField,
  };
});
```

## Store Patterns

### 1. Composition API Pattern

```typescript
export const useMyStore = defineStore('myStore', () => {
  // State
  const items = ref<Item[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);

  // Getters (computed)
  const itemCount = computed(() => items.value.length);
  const hasItems = computed(() => items.value.length > 0);

  // Actions (functions)
  async function hydrate() {
    loading.value = true;
    error.value = null;

    try {
      const response = await api.get('/items');
      items.value = response.data.data;
    } catch (err) {
      error.value = err as Error;
    } finally {
      loading.value = false;
    }
  }

  async function dehydrate() {
    items.value = [];
    loading.value = false;
    error.value = null;
  }

  return {
    items,
    loading,
    error,
    itemCount,
    hasItems,
    hydrate,
    dehydrate,
  };
});
```

### 2. Options API Pattern

```typescript
export const useMyStore = defineStore({
  id: 'myStore',
  state: () => ({
    items: [] as Item[],
    loading: false,
    error: null as Error | null,
  }),
  getters: {
    itemCount: (state) => state.items.length,
    hasItems: (state) => state.items.length > 0,
  },
  actions: {
    async hydrate() {
      this.loading = true;
      this.error = null;

      try {
        const response = await api.get('/items');
        this.items = response.data.data;
      } catch (error) {
        this.error = error as Error;
      } finally {
        this.loading = false;
      }
    },
    dehydrate() {
      this.$reset();
    },
  },
});
```

### 3. Optimistic Update Pattern

```typescript
async function updateItem(id: string, updates: Partial<Item>) {
  const stateClone = [...items.value];

  // Optimistic update
  items.value = items.value.map((item) => {
    if (item.id === id) {
      return { ...item, ...updates };
    }
    return item;
  });

  try {
    const response = await api.patch(`/items/${id}`, updates);
    
    // Update with server response
    items.value = items.value.map((item) => {
      if (item.id === id) {
        return response.data.data;
      }
      return item;
    });
  } catch (error) {
    // Rollback on error
    items.value = stateClone;
    throw error;
  }
}
```

### 4. Cross-Store Communication

```typescript
async function deleteItem(id: string) {
  const relatedStore = useRelatedStore();
  
  const stateClone = [...items.value];
  const relatedStateClone = [...relatedStore.items];

  // Update both stores
  items.value = items.value.filter((item) => item.id !== id);
  relatedStore.items = relatedStore.items.filter((item) => item.parentId !== id);

  try {
    await api.delete(`/items/${id}`);
  } catch (error) {
    // Rollback both stores
    items.value = stateClone;
    relatedStore.items = relatedStateClone;
    throw error;
  }
}
```

## Store Dependencies

### Dependency Graph

```
server (no dependencies)
  ↓
user (depends on server)
  ↓
permissions (depends on user)
  ↓
collections, fields, relations (can run in parallel)
  ↓
flows, insights, presets (depend on collections/fields)
```

### Hydration Order

```typescript
async function hydrateStores() {
  // 1. Server info
  await serverStore.hydrate();

  // 2. User info
  await userStore.hydrate();

  // 3. Permissions
  await permissionsStore.hydrate();

  // 4. Schema (parallel)
  await Promise.all([
    collectionsStore.hydrate(),
    fieldsStore.hydrate(),
    relationsStore.hydrate(),
  ]);

  // 5. Features (parallel)
  await Promise.all([
    flowsStore.hydrate(),
    insightsStore.hydrate(),
    presetsStore.hydrate(),
  ]);
}
```

## Best Practices

### 1. Always Provide Dehydrate

```typescript
async function dehydrate() {
  items.value = [];
  loading.value = false;
  error.value = null;
}
```

### 2. Handle Loading States

```typescript
const loading = ref(false);

async function hydrate() {
  loading.value = true;
  try {
    // ... fetch data
  } finally {
    loading.value = false;
  }
}
```

### 3. Handle Errors

```typescript
const error = ref<Error | null>(null);

async function hydrate() {
  error.value = null;
  try {
    // ... fetch data
  } catch (err) {
    error.value = err as Error;
    unexpectedError(err);
  }
}
```

### 4. Use Optimistic Updates

```typescript
// Save state before update
const stateClone = [...items.value];

// Apply optimistic update
items.value = newItems;

try {
  await api.patch('/items', newItems);
} catch (error) {
  // Rollback on error
  items.value = stateClone;
  throw error;
}
```

### 5. Provide Getters

```typescript
const items = ref<Item[]>([]);

const itemCount = computed(() => items.value.length);
const hasItems = computed(() => items.value.length > 0);
const activeItems = computed(() => 
  items.value.filter((item) => item.active)
);
```

## Next Steps

- [Frontend Data Flow](./22-frontend-data-flow.md) - Data flow patterns
- [API Integration](./13-api-integration.md) - HTTP client usage
- [Component System](./08-ui-components.md) - UI components

## Related Files

- `app/src/stores/user.ts` - User store
- `app/src/stores/collections.ts` - Collections store
- `app/src/stores/fields.ts` - Fields store
- `app/src/stores/permissions.ts` - Permissions store
