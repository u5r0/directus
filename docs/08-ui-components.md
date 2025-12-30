# UI Component System

The Directus frontend features a comprehensive component library with 60+ base components, providing a complete design system for the admin interface.

## Table of Contents
- [Component Architecture](#component-architecture)
- [Base Components (v-*)](#base-components-v-)
- [Component Registration](#component-registration)
- [Component Interaction Patterns](#component-interaction-patterns)
- [Form System](#form-system)
- [Table System](#table-system)
- [Component Communication](#component-communication)
- [Styling System](#styling-system)
- [Component Testing](#component-testing)

## Component Architecture

### Design Principles
- **Composable** - Components can be combined to create complex interfaces
- **Accessible** - Full ARIA support and keyboard navigation
- **Themeable** - CSS custom properties for consistent styling
- **Type-Safe** - Full TypeScript support with prop validation
- **Extensible** - Slot-based architecture for customization

### Component Hierarchy
```
Base Components (v-*)
├── Layout Components (structure)
├── Form Components (input/interaction)
├── Display Components (data presentation)
├── Navigation Components (routing/menus)
├── Feedback Components (status/alerts)
└── Utility Components (helpers)
```

## Base Components (v-*)

### Layout & Structure Components

#### VCard System
```vue
<!-- Card with all slots -->
<VCard>
  <VCardTitle>Card Title</VCardTitle>
  <VCardSubtitle>Optional subtitle</VCardSubtitle>
  <VCardText>
    Main card content goes here
  </VCardText>
  <VCardActions>
    <VButton>Action</VButton>
  </VCardActions>
</VCard>
```

**Props & Features:**
- `elevation` - Shadow depth (0-24)
- `outlined` - Border instead of shadow
- `tile` - Remove border radius
- `hover` - Hover elevation effect

#### VSheet
```vue
<!-- Basic surface container -->
<VSheet 
  :elevation="2"
  color="primary"
  rounded
>
  Content with elevation and color
</VSheet>
```

#### VWorkspace System
```vue
<!-- Workspace with draggable tiles -->
<VWorkspace v-model="tiles">
  <VWorkspaceTile
    v-for="tile in tiles"
    :key="tile.id"
    :tile="tile"
    @update="updateTile"
    @remove="removeTile"
  >
    <component :is="tile.component" v-bind="tile.props" />
  </VWorkspaceTile>
</VWorkspace>
```

### Form & Input Components

#### VInput
```vue
<!-- Advanced text input -->
<VInput
  v-model="value"
  :placeholder="$t('enter_value')"
  :disabled="loading"
  :validation-error="error"
  prefix="$"
  suffix=".00"
  clearable
  @focus="handleFocus"
  @blur="handleBlur"
>
  <template #prepend>
    <VIcon name="search" />
  </template>
  <template #append>
    <VButton icon small @click="submit">
      <VIcon name="send" />
    </VButton>
  </template>
</VInput>
```

**Features:**
- Validation integration
- Prefix/suffix text
- Icon slots (prepend/append)
- Loading states
- Clearable option
- Auto-focus support

#### VSelect
```vue
<!-- Advanced select component -->
<VSelect
  v-model="selected"
  :items="options"
  :loading="loading"
  multiple
  searchable
  clearable
  :placeholder="$t('select_option')"
  item-text="name"
  item-value="id"
  @search="handleSearch"
>
  <template #selection="{ item }">
    <VChip>{{ item.name }}</VChip>
  </template>
  <template #item="{ item, active }">
    <VListItem :active="active">
      <VListItemIcon>
        <VIcon :name="item.icon" />
      </VListItemIcon>
      <VListItemContent>
        {{ item.name }}
      </VListItemContent>
    </VListItem>
  </template>
</VSelect>
```

#### VCheckbox & VRadio
```vue
<!-- Checkbox with indeterminate state -->
<VCheckbox
  v-model="checked"
  :indeterminate="someSelected"
  :disabled="loading"
  label="Select all items"
  @change="handleChange"
/>

<!-- Radio group -->
<VRadio
  v-for="option in options"
  :key="option.value"
  v-model="selected"
  :value="option.value"
  :label="option.label"
/>
```

#### VSlider
```vue
<!-- Range slider -->
<VSlider
  v-model="value"
  :min="0"
  :max="100"
  :step="5"
  show-ticks
  thumb-label
  @change="handleChange"
>
  <template #thumb-label="{ value }">
    {{ value }}%
  </template>
</VSlider>
```

#### VDatePicker
```vue
<!-- Date/time picker -->
<VDatePicker
  v-model="date"
  :type="'datetime'"
  :disabled-dates="disabledDates"
  :locale="userLocale"
  @update:model-value="handleDateChange"
/>
```

#### VUpload
```vue
<!-- File upload with drag/drop -->
<VUpload
  v-model="files"
  :multiple="true"
  :accept="'image/*'"
  :max-size="10 * 1024 * 1024"
  @upload="handleUpload"
  @error="handleError"
>
  <template #default="{ dragover, uploading }">
    <div :class="{ dragover, uploading }">
      <VIcon name="cloud_upload" large />
      <p>{{ $t('drag_files_here') }}</p>
    </div>
  </template>
</VUpload>
```

### Data Display Components

#### VTable
```vue
<!-- Advanced data table -->
<VTable
  :headers="headers"
  :items="items"
  :loading="loading"
  :selection="selection"
  :sort="sort"
  show-select="multiple"
  show-resize
  show-manual-sort
  @click:row="handleRowClick"
  @update:selection="selection = $event"
  @update:sort="sort = $event"
  @manual-sort="handleManualSort"
>
  <!-- Custom header -->
  <template #header.name="{ header }">
    <VIcon name="person" />
    {{ header.text }}
  </template>
  
  <!-- Custom cell -->
  <template #item.status="{ item }">
    <VChip :color="getStatusColor(item.status)">
      {{ item.status }}
    </VChip>
  </template>
  
  <!-- Row actions -->
  <template #item-append="{ item }">
    <VMenu>
      <template #activator="{ toggle }">
        <VButton icon small @click="toggle">
          <VIcon name="more_vert" />
        </VButton>
      </template>
      <VList>
        <VListItem @click="editItem(item)">
          <VListItemIcon><VIcon name="edit" /></VListItemIcon>
          <VListItemContent>{{ $t('edit') }}</VListItemContent>
        </VListItem>
      </VList>
    </VMenu>
  </template>
</VTable>
```

**Table Features:**
- Column sorting and filtering
- Row selection (single/multiple)
- Manual row reordering
- Column resizing and reordering
- Custom cell rendering
- Loading and empty states
- Pagination integration
- Keyboard navigation

#### VPagination
```vue
<!-- Pagination controls -->
<VPagination
  v-model="currentPage"
  :total-pages="totalPages"
  :total-items="totalItems"
  :items-per-page="itemsPerPage"
  show-totals
  @update:items-per-page="itemsPerPage = $event"
/>
```

#### VAvatar
```vue
<!-- User avatar -->
<VAvatar
  :src="user.avatar"
  :alt="user.name"
  :color="user.color"
  size="large"
>
  <template #placeholder>
    {{ user.initials }}
  </template>
</VAvatar>
```

#### VIcon System
```vue
<!-- Icon with FontAwesome integration -->
<VIcon
  name="home"
  :size="24"
  color="primary"
  outline
  filled
  @click="handleClick"
/>

<!-- File type icons -->
<VIconFile
  :file-type="file.type"
  :size="48"
/>
```

### Navigation Components

#### VBreadcrumb
```vue
<!-- Hierarchical navigation -->
<VBreadcrumb :items="breadcrumbItems">
  <template #item="{ item, index }">
    <router-link :to="item.to">
      <VIcon v-if="item.icon" :name="item.icon" />
      {{ item.name }}
    </router-link>
  </template>
</VBreadcrumb>
```

#### VTabs System
```vue
<!-- Tab navigation -->
<VTabs v-model="activeTab">
  <VTab value="general">{{ $t('general') }}</VTab>
  <VTab value="advanced">{{ $t('advanced') }}</VTab>
  <VTab value="permissions">{{ $t('permissions') }}</VTab>
</VTabs>

<VTabsItems v-model="activeTab">
  <VTabItem value="general">
    <GeneralSettings />
  </VTabItem>
  <VTabItem value="advanced">
    <AdvancedSettings />
  </VTabItem>
  <VTabItem value="permissions">
    <PermissionSettings />
  </VTabItem>
</VTabsItems>
```

#### VMenu
```vue
<!-- Dropdown menu -->
<VMenu
  :close-on-content-click="false"
  :offset-y="true"
>
  <template #activator="{ toggle, active }">
    <VButton
      :active="active"
      @click="toggle"
    >
      Menu <VIcon name="arrow_drop_down" />
    </VButton>
  </template>
  
  <VList>
    <VListItem @click="action1">
      <VListItemIcon><VIcon name="edit" /></VListItemIcon>
      <VListItemContent>{{ $t('edit') }}</VListItemContent>
      <VListItemHint>Ctrl+E</VListItemHint>
    </VListItem>
    <VListItem @click="action2">
      <VListItemIcon><VIcon name="delete" /></VListItemIcon>
      <VListItemContent>{{ $t('delete') }}</VListItemContent>
    </VListItem>
  </VList>
</VMenu>
```

#### VList System
```vue
<!-- Hierarchical lists -->
<VList>
  <VListGroup>
    <template #activator>
      <VListItem>
        <VListItemIcon><VIcon name="folder" /></VListItemIcon>
        <VListItemContent>{{ $t('collections') }}</VListItemContent>
      </VListItem>
    </template>
    
    <VListItem
      v-for="collection in collections"
      :key="collection.id"
      :to="`/content/${collection.collection}`"
    >
      <VListItemIcon>
        <VIcon :name="collection.icon" />
      </VListItemIcon>
      <VListItemContent>
        {{ collection.name }}
      </VListItemContent>
      <VListItemHint>
        {{ collection.count }} items
      </VListItemHint>
    </VListItem>
  </VListGroup>
</VList>
```

### Feedback Components

#### VNotice
```vue
<!-- Alert messages -->
<VNotice
  type="warning"
  :title="$t('warning')"
  icon="warning"
  :closable="true"
  @close="handleClose"
>
  {{ $t('unsaved_changes_warning') }}
</VNotice>
```

**Notice Types:**
- `info` - Informational messages
- `success` - Success confirmations
- `warning` - Warning alerts
- `danger` - Error messages

#### VDialog
```vue
<!-- Modal dialogs -->
<VDialog
  v-model="dialogOpen"
  :persistent="hasUnsavedChanges"
  max-width="600px"
  @esc="handleEscape"
>
  <VCard>
    <VCardTitle>{{ $t('confirm_action') }}</VCardTitle>
    <VCardText>
      {{ $t('confirm_delete_message') }}
    </VCardText>
    <VCardActions>
      <VButton secondary @click="dialogOpen = false">
        {{ $t('cancel') }}
      </VButton>
      <VButton
        kind="danger"
        :loading="deleting"
        @click="confirmDelete"
      >
        {{ $t('delete') }}
      </VButton>
    </VCardActions>
  </VCard>
</VDialog>
```

#### VProgress Components
```vue
<!-- Linear progress -->
<VProgressLinear
  :value="uploadProgress"
  :indeterminate="loading"
  color="primary"
  height="4"
/>

<!-- Circular progress -->
<VProgressCircular
  :value="progress"
  :size="48"
  :width="4"
  color="primary"
>
  {{ Math.round(progress) }}%
</VProgressCircular>
```

#### VSkeletonLoader
```vue
<!-- Loading placeholders -->
<VSkeletonLoader
  type="card"
  :loading="loading"
>
  <VCard>
    <VCardTitle>{{ item.title }}</VCardTitle>
    <VCardText>{{ item.content }}</VCardText>
  </VCard>
</VSkeletonLoader>
```

**Skeleton Types:**
- `text` - Text lines
- `card` - Card layout
- `table` - Table structure
- `list` - List items
- `avatar` - Avatar placeholder

### Interactive Components

#### VButton
```vue
<!-- Button with all features -->
<VButton
  kind="primary"
  size="large"
  :loading="submitting"
  :disabled="!isValid"
  icon="save"
  rounded
  @click="handleSubmit"
>
  {{ $t('save_changes') }}
  
  <template #append-outer>
    <VMenu>
      <template #activator="{ toggle }">
        <VIcon name="arrow_drop_down" @click="toggle" />
      </template>
      <VList>
        <VListItem @click="saveAndContinue">
          {{ $t('save_and_continue') }}
        </VListItem>
      </VList>
    </VMenu>
  </template>
</VButton>
```

**Button Variants:**
- `kind`: `normal`, `info`, `success`, `warning`, `danger`
- `size`: `x-small`, `small`, `normal`, `large`, `x-large`
- States: `loading`, `disabled`, `active`
- Styles: `outlined`, `rounded`, `icon`, `tile`

#### VDrawer
```vue
<!-- Slide-out drawer -->
<VDrawer
  v-model="drawerOpen"
  :width="400"
  position="right"
  @close="handleClose"
>
  <template #header>
    <VDrawerHeader>
      <VIcon name="settings" />
      {{ $t('settings') }}
    </VDrawerHeader>
  </template>
  
  <div class="drawer-content">
    <SettingsForm />
  </div>
</VDrawer>
```

## Component Registration

### Global Registration System
```typescript
// components/register.ts
export function registerComponents(app: App): void {
  // Layout Components
  app.component('VCard', VCard);
  app.component('VCardTitle', VCardTitle);
  app.component('VCardSubtitle', VCardSubtitle);
  app.component('VCardText', VCardText);
  app.component('VCardActions', VCardActions);
  app.component('VSheet', VSheet);
  app.component('VDivider', VDivider);
  
  // Form Components
  app.component('VInput', VInput);
  app.component('VTextarea', VTextarea);
  app.component('VSelect', VSelect);
  app.component('VCheckbox', VCheckbox);
  app.component('VRadio', VRadio);
  app.component('VSlider', VSlider);
  app.component('VDatePicker', VDatePicker);
  app.component('VUpload', VUpload);
  
  // Display Components
  app.component('VTable', VTable);
  app.component('VPagination', VPagination);
  app.component('VAvatar', VAvatar);
  app.component('VIcon', VIcon);
  app.component('VBadge', VBadge);
  app.component('VChip', VChip);
  
  // Navigation Components
  app.component('VBreadcrumb', VBreadcrumb);
  app.component('VTabs', VTabs);
  app.component('VTab', VTab);
  app.component('VTabItem', VTabItem);
  app.component('VTabsItems', VTabsItems);
  app.component('VMenu', VMenu);
  app.component('VList', VList);
  app.component('VListItem', VListItem);
  app.component('VListGroup', VListGroup);
  
  // Feedback Components
  app.component('VNotice', VNotice);
  app.component('VDialog', VDialog);
  app.component('VProgressLinear', VProgressLinear);
  app.component('VProgressCircular', VProgressCircular);
  app.component('VSkeletonLoader', VSkeletonLoader);
  
  // Interactive Components
  app.component('VButton', VButton);
  app.component('VDrawer', VDrawer);
  app.component('VOverlay', VOverlay);
  
  // Utility Components
  app.component('VForm', VForm);
  app.component('VHighlight', VHighlight);
  app.component('VTextOverflow', VTextOverflow);
  app.component('VResizeable', VResizeable);
  app.component('VErrorBoundary', VErrorBoundary);
  
  // Transition Components
  app.component('TransitionBounce', TransitionBounce);
  app.component('TransitionDialog', TransitionDialog);
  app.component('TransitionExpand', TransitionExpand);
}
```

## Component Interaction Patterns

### 1. Props Down, Events Up
```vue
<!-- Parent Component -->
<template>
  <VTable
    :headers="headers"
    :items="items"
    :selection="selection"
    :loading="loading"
    @click:row="handleRowClick"
    @update:selection="updateSelection"
    @update:sort="updateSort"
  />
</template>

<script setup lang="ts">
const selection = ref([]);
const sort = ref({ by: 'name', desc: false });

function handleRowClick(event: { item: any, event: MouseEvent }) {
  router.push(`/items/${event.item.id}`);
}

function updateSelection(newSelection: any[]) {
  selection.value = newSelection;
  emit('selection-changed', newSelection);
}
</script>
```

### 2. Provide/Inject for Deep Hierarchies
```vue
<!-- Parent Component -->
<script setup lang="ts">
import { provide } from 'vue';

const formValues = ref({});
const formErrors = ref({});

provide('form-values', formValues);
provide('form-errors', formErrors);
</script>

<!-- Deep Child Component -->
<script setup lang="ts">
import { inject } from 'vue';

const formValues = inject('form-values');
const formErrors = inject('form-errors');
</script>
```

### 3. Composable-Based Logic Sharing
```typescript
// composables/use-table-selection.ts
export function useTableSelection(items: Ref<any[]>) {
  const selection = ref<any[]>([]);
  
  const allSelected = computed(() => 
    items.value.length > 0 && selection.value.length === items.value.length
  );
  
  const someSelected = computed(() => 
    selection.value.length > 0 && !allSelected.value
  );
  
  function toggleAll() {
    selection.value = allSelected.value ? [] : [...items.value];
  }
  
  function toggleItem(item: any) {
    const index = selection.value.findIndex(s => s.id === item.id);
    if (index > -1) {
      selection.value.splice(index, 1);
    } else {
      selection.value.push(item);
    }
  }
  
  return {
    selection,
    allSelected,
    someSelected,
    toggleAll,
    toggleItem,
  };
}
```

## Form System

### VForm Architecture
The VForm component is the central hub for all data input and editing:

```vue
<!-- Dynamic form generation -->
<VForm
  v-model="edits"
  :collection="collection"
  :primary-key="primaryKey"
  :initial-values="item"
  :loading="loading"
  :disabled="!updateAllowed"
  :validation-errors="validationErrors"
  :batch-mode="batchMode"
  :raw-editor-enabled="true"
  @update:model-value="handleFormUpdate"
>
  <!-- Form fields are automatically generated based on collection schema -->
</VForm>
```

### Form Field Rendering
```typescript
// VForm dynamically renders fields based on schema
const fieldsWithConditions = computed(() => {
  return formFields.value.map(field => 
    applyConditions(values.value, field, version.value)
  );
});

// Field component selection
const component = computed(() => {
  if (field.meta?.special?.includes('group')) {
    return `interface-${field.meta?.interface || 'group-standard'}`;
  }
  return 'FormField';
});
```

### Form Validation Integration
```vue
<!-- FormField with validation -->
<FormField
  :field="field"
  :model-value="values[fieldName]"
  :validation-error="getValidationError(fieldName)"
  :disabled="isFieldDisabled(field)"
  @update:model-value="setValue(fieldName, $event)"
  @set-field-value="setValue($event.field, $event.value, { force: true })"
/>
```

## Table System

### VTable Features
```typescript
// Table configuration
interface TableProps {
  headers: HeaderRaw[];           // Column definitions
  items: Item[];                  // Data rows
  selection?: any[];              // Selected items
  sort?: Sort;                    // Current sort state
  loading?: boolean;              // Loading state
  showSelect?: ShowSelect;        // Selection mode
  showResize?: boolean;           // Column resizing
  showManualSort?: boolean;       // Drag/drop sorting
  allowHeaderReorder?: boolean;   // Column reordering
  fixedHeader?: boolean;          // Sticky header
  clickable?: boolean;            // Row click handling
}
```

### Table Event Handling
```vue
<VTable
  @click:row="({ item, event }) => handleRowClick(item, event)"
  @update:selection="selection = $event"
  @update:sort="sort = $event"
  @manual-sort="({ item, to }) => handleManualSort(item, to)"
  @update:headers="headers = $event"
/>
```

### Custom Cell Rendering
```vue
<VTable :headers="headers" :items="items">
  <!-- Custom header -->
  <template #header.status="{ header }">
    <VIcon name="flag" />
    {{ header.text }}
  </template>
  
  <!-- Custom cell with display component -->
  <template #item.status="{ item }">
    <RenderDisplay
      :value="item.status"
      :display="'labels'"
      :options="{ choices: statusChoices }"
    />
  </template>
  
  <!-- Row actions -->
  <template #item-append="{ item }">
    <VButton icon small @click="editItem(item)">
      <VIcon name="edit" />
    </VButton>
  </template>
</VTable>
```

## Component Communication

### Event Bus System
```typescript
// events.ts
export enum Events {
  tabIdle = 'tab-idle',
  tabActive = 'tab-active',
  userUpdated = 'user-updated',
  collectionUpdated = 'collection-updated',
}

export const emitter = mitt<{
  [Events.tabIdle]: void;
  [Events.tabActive]: void;
  [Events.userUpdated]: User;
  [Events.collectionUpdated]: Collection;
}>();

// Component usage
emitter.on(Events.userUpdated, (user) => {
  // Handle user update
});

emitter.emit(Events.userUpdated, updatedUser);
```

### Store Integration
```vue
<script setup lang="ts">
import { useCollectionsStore } from '@/stores/collections';
import { usePermissionsStore } from '@/stores/permissions';

const collectionsStore = useCollectionsStore();
const permissionsStore = usePermissionsStore();

// Reactive computed properties
const collections = computed(() => collectionsStore.visibleCollections);
const canCreate = computed(() => 
  permissionsStore.hasPermission(collection.value, 'create')
);

// Store actions
async function createCollection(data: any) {
  await collectionsStore.upsertCollection(data.collection, data);
}
</script>
```

## Styling System

### CSS Custom Properties
```scss
// Component styling with CSS variables
.v-button {
  --v-button-color: var(--theme--foreground-inverted);
  --v-button-background-color: var(--theme--primary);
  --v-button-height: 44px;
  --v-button-padding: 0 19px;
  
  color: var(--v-button-color);
  background-color: var(--v-button-background-color);
  height: var(--v-button-height);
  padding: var(--v-button-padding);
}

// Theme variants
.v-button.success {
  --v-button-background-color: var(--theme--success);
}

.v-button.danger {
  --v-button-background-color: var(--theme--danger);
}
```

### Responsive Design
```scss
// Responsive utilities
@use '@/styles/mixins';

.component {
  @include mixins.breakpoint('small') {
    // Mobile styles
  }
  
  @include mixins.breakpoint('medium') {
    // Tablet styles
  }
  
  @include mixins.breakpoint('large') {
    // Desktop styles
  }
}
```

## Component Testing

### Unit Testing with Vitest
```typescript
// VButton.test.ts
import { mount } from '@vue/test-utils';
import VButton from './v-button.vue';

describe('VButton', () => {
  it('renders correctly', () => {
    const wrapper = mount(VButton, {
      props: {
        kind: 'primary',
      },
      slots: {
        default: 'Click me',
      },
    });
    
    expect(wrapper.text()).toBe('Click me');
    expect(wrapper.classes()).toContain('primary');
  });
  
  it('emits click event', async () => {
    const wrapper = mount(VButton);
    
    await wrapper.trigger('click');
    
    expect(wrapper.emitted('click')).toHaveLength(1);
  });
  
  it('shows loading state', () => {
    const wrapper = mount(VButton, {
      props: { loading: true },
    });
    
    expect(wrapper.find('.spinner').exists()).toBe(true);
    expect(wrapper.find('.content').classes()).toContain('invisible');
  });
});
```

### Component Documentation with Histoire
```vue
<!-- VButton.story.vue -->
<script setup lang="ts">
import VButton from './v-button.vue';
</script>

<template>
  <Story title="VButton" :layout="{ type: 'grid', width: 200 }">
    <Variant title="Primary">
      <VButton kind="primary">Primary Button</VButton>
    </Variant>
    
    <Variant title="Secondary">
      <VButton secondary>Secondary Button</VButton>
    </Variant>
    
    <Variant title="Loading">
      <VButton :loading="true">Loading Button</VButton>
    </Variant>
    
    <Variant title="Icon">
      <VButton icon>
        <VIcon name="add" />
      </VButton>
    </Variant>
  </Story>
</template>
```

## Next Steps

- **Interface System**: See [Interface System](./09-interface-system.md) for input components
- **Display System**: Check [Display System](./10-display-system.md) for output components
- **Layout System**: Review [Layout System](./11-layout-system.md) for content layouts
- **State Management**: Explore [State Management](./12-state-management.md) for data flow
