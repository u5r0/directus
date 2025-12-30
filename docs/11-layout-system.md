# Layout System

This document covers the layout system in Directus, which provides 5 different ways to view and interact with collection data.

## Overview

Layouts are full-page components that determine how collection items are displayed and interacted with in the Directus admin app. Each layout provides a unique way to visualize and manage data.

## Layout Architecture

### Layout Structure

Each layout consists of:
- **index.ts** - Layout configuration and setup logic
- **{name}.vue** - Main layout component
- **options.vue** - Layout-specific options component
- **sidebar.vue** - Sidebar component for filters/details
- **actions.vue** - Action buttons component

### Layout Registration (`app/src/layouts/index.ts`)

```typescript
export function getInternalLayouts(): LayoutConfig[] {
  const layouts = import.meta.glob<LayoutConfig>('./*/index.ts', {
    import: 'default',
    eager: true
  });
  
  return sortBy(Object.values(layouts), 'id');
}

export function registerLayouts(layouts: LayoutConfig[], app: App): void {
  for (const layout of layouts) {
    app.component(`layout-${layout.id}`, layout.component);
    app.component(`layout-options-${layout.id}`, layout.slots.options);
    app.component(`layout-sidebar-${layout.id}`, layout.slots.sidebar);
    app.component(`layout-actions-${layout.id}`, layout.slots.actions);
  }
}
```

### Layout Configuration

```typescript
import { defineLayout } from '@directus/extensions';

export default defineLayout<LayoutOptions, LayoutQuery>({
  id: 'tabular',
  name: '$t:layouts.tabular.tabular',
  icon: 'table_rows',
  component: TabularLayout,
  slots: {
    options: TabularOptions,
    sidebar: () => undefined,
    actions: TabularActions,
  },
  headerShadow: false,
  setup(props, { emit }) {
    // Layout setup logic
    const selection = useSync(props, 'selection', emit);
    const layoutOptions = useSync(props, 'layoutOptions', emit);
    const layoutQuery = useSync(props, 'layoutQuery', emit);
    
    // Return layout state and methods
    return {
      selection,
      layoutOptions,
      layoutQuery,
      // ... other state and methods
    };
  },
});
```

## Available Layouts

### 1. Tabular Layout

Traditional table view with sorting, filtering, and pagination.

**Best For:**
- Large datasets
- Data entry and editing
- Detailed record management
- Spreadsheet-like workflows

**Features:**
- Column sorting (ascending/descending)
- Column reordering (drag & drop)
- Column resizing
- Row selection (single/multiple)
- Inline editing
- Pagination
- Search and filtering
- Export to CSV
- Manual row sorting (if sort field configured)

**Configuration Options:**
```typescript
interface TabularOptions {
  widths?: Record<string, number>;  // Column widths
  spacing?: 'comfortable' | 'cozy' | 'compact';  // Row spacing
}

interface TabularQuery {
  fields?: string[];
  sort?: string[];
  limit?: number;
  page?: number;
}
```

**Usage Example:**
```vue
<layout-tabular
  collection="articles"
  :selection="selection"
  :layout-options="{ spacing: 'comfortable' }"
  :layout-query="{ fields: ['title', 'status', 'author'], sort: ['-created_at'] }"
  @update:selection="selection = $event"
/>
```

**Table Headers:**
```typescript
const tableHeaders = computed<HeaderRaw[]>(() => [
  {
    text: 'Title',
    value: 'title',
    width: 300,
    sortable: true,
    align: 'left',
  },
  {
    text: 'Status',
    value: 'status',
    width: 120,
    sortable: true,
  },
  {
    text: 'Author',
    value: 'author.name',
    width: 200,
    sortable: true,
  },
  {
    text: 'Created',
    value: 'created_at',
    width: 150,
    sortable: true,
  },
]);
```

### 2. Cards Layout

Card-based grid layout with customizable templates.

**Best For:**
- Visual content (images, media)
- Product catalogs
- Portfolio items
- Content previews

**Features:**
- Responsive grid
- Customizable card templates
- Image thumbnails
- Drag & drop sorting
- Card size options (small, medium, large)
- Masonry layout option
- Lazy loading

**Configuration Options:**
```typescript
interface CardsOptions {
  size?: 'small' | 'medium' | 'large';
  imageFit?: 'crop' | 'contain';
  imageSource?: string;  // Field for card image
  title?: string;        // Field for card title
  subtitle?: string;     // Field for card subtitle
  icon?: string;         // Field for card icon
}
```

**Card Template:**
```vue
<template>
  <v-card
    :to="`/content/${collection}/${item[primaryKeyField.field]}`"
    class="card"
  >
    <v-image
      v-if="imageSrc"
      :src="imageSrc"
      :alt="item[titleField]"
    />
    
    <div class="card-content">
      <h3 class="card-title">{{ item[titleField] }}</h3>
      <p class="card-subtitle">{{ item[subtitleField] }}</p>
    </div>
  </v-card>
</template>
```

### 3. Calendar Layout

Calendar view for date-based content.

**Best For:**
- Events and schedules
- Content publishing calendars
- Booking systems
- Time-based data

**Features:**
- Month, week, day views
- Drag & drop to reschedule
- Event creation
- Color coding by field
- Multi-day events
- Recurring events
- Time zone support

**Configuration Options:**
```typescript
interface CalendarOptions {
  viewInfo?: {
    type: 'dayGridMonth' | 'timeGridWeek' | 'timeGridDay' | 'listWeek';
    startDateStr?: string;
  };
  firstDay?: number;  // 0 = Sunday, 1 = Monday
  template?: string;  // Event display template
  startDateField?: string;
  endDateField?: string;
}
```

**Calendar Integration:**
```typescript
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import timeGridPlugin from '@fullcalendar/timegrid';
import interactionPlugin from '@fullcalendar/interaction';

const calendarOptions = {
  plugins: [dayGridPlugin, timeGridPlugin, interactionPlugin],
  initialView: 'dayGridMonth',
  editable: true,
  selectable: true,
  events: calendarEvents.value,
  eventClick: handleEventClick,
  dateClick: handleDateClick,
  eventDrop: handleEventDrop,
};
```

### 4. Map Layout

Geographic map layout for location data.

**Best For:**
- Location-based content
- Store locators
- Geographic data visualization
- Spatial analysis

**Features:**
- Interactive map (MapBox/MapLibre)
- Marker clustering
- Custom marker icons
- Popup information
- Draw tools
- Geolocation
- Multiple map styles

**Configuration Options:**
```typescript
interface MapOptions {
  basemap?: string;  // Map style
  geometryField?: string;
  template?: string;  // Popup template
  clusterData?: boolean;
  defaultZoom?: number;
  defaultCenter?: [number, number];
}
```

**Map Integration:**
```typescript
import maplibregl from 'maplibre-gl';

const map = new maplibregl.Map({
  container: mapContainer.value,
  style: mapStyle.value,
  center: defaultCenter.value,
  zoom: defaultZoom.value,
});

// Add markers
items.value.forEach((item) => {
  const coordinates = item[geometryField.value];
  
  new maplibregl.Marker()
    .setLngLat(coordinates)
    .setPopup(new maplibregl.Popup().setHTML(popupContent))
    .addTo(map);
});
```

### 5. Kanban Layout

Kanban board for workflow management.

**Best For:**
- Task management
- Project workflows
- Status tracking
- Agile boards

**Features:**
- Drag & drop between columns
- Customizable columns
- Card templates
- Column limits
- Swimlanes
- Card sorting
- Quick actions

**Configuration Options:**
```typescript
interface KanbanOptions {
  groupField?: string;  // Field to group by (status, stage, etc.)
  title?: string;       // Card title field
  text?: string;        // Card text field
  tags?: string;        // Card tags field
  userField?: string;   // Assignee field
  dateField?: string;   // Due date field
}
```

**Kanban Structure:**
```vue
<template>
  <div class="kanban-board">
    <div
      v-for="column in columns"
      :key="column.value"
      class="kanban-column"
    >
      <div class="column-header">
        <h3>{{ column.text }}</h3>
        <span class="count">{{ column.items.length }}</span>
      </div>
      
      <draggable
        v-model="column.items"
        :group="{ name: 'kanban' }"
        @change="handleCardMove"
      >
        <kanban-card
          v-for="item in column.items"
          :key="item.id"
          :item="item"
          :fields="fields"
        />
      </draggable>
    </div>
  </div>
</template>
```

## Layout Props

All layouts receive these props:

```typescript
interface LayoutProps {
  collection: string;
  selection?: (string | number)[];
  layoutOptions?: Record<string, any>;
  layoutQuery?: Query;
  filter?: Filter;
  filterUser?: Filter;
  filterSystem?: Filter;
  search?: string;
  selectMode?: boolean;
  readonly?: boolean;
  resetPreset?: () => void;
}
```

## Layout Slots

### Options Slot

Configuration panel for layout-specific settings:

```vue
<template #options>
  <layout-options-tabular
    v-model="layoutOptions"
    :collection="collection"
  />
</template>
```

### Sidebar Slot

Sidebar for filters, details, or additional information:

```vue
<template #sidebar>
  <layout-sidebar-tabular
    :collection="collection"
    :filter="filter"
    @update:filter="$emit('update:filter', $event)"
  />
</template>
```

### Actions Slot

Action buttons for the layout:

```vue
<template #actions>
  <layout-actions-tabular
    :selection="selection"
    :collection="collection"
    @export="handleExport"
    @batch-edit="handleBatchEdit"
  />
</template>
```

## Layout State Management

### Layout Options

Persistent layout configuration:

```typescript
const layoutOptions = useSync(props, 'layoutOptions', emit);

// Tabular: column widths, spacing
layoutOptions.value = {
  widths: { title: 300, status: 120 },
  spacing: 'comfortable',
};

// Cards: card size, image settings
layoutOptions.value = {
  size: 'medium',
  imageSource: 'featured_image',
  title: 'title',
};
```

### Layout Query

Query parameters for data fetching:

```typescript
const layoutQuery = useSync(props, 'layoutQuery', emit);

layoutQuery.value = {
  fields: ['id', 'title', 'status', 'author.name'],
  sort: ['-created_at'],
  limit: 25,
  page: 1,
};
```

### Selection State

Selected items across layouts:

```typescript
const selection = useSync(props, 'selection', emit);

// Single selection
selection.value = ['item-id-1'];

// Multiple selection
selection.value = ['item-id-1', 'item-id-2', 'item-id-3'];
```

## Layout Integration

### With useItems Composable

Layouts use the `useItems` composable for data fetching:

```typescript
import { useItems } from '@directus/composables';

const {
  items,
  loading,
  error,
  totalPages,
  itemCount,
  totalCount,
  getItems,
  getItemCount,
  getTotalCount,
} = useItems(collection, {
  sort,
  limit,
  page,
  fields,
  filter,
  search,
});
```

### With useCollection Composable

Access collection metadata:

```typescript
import { useCollection } from '@directus/composables';

const {
  info,
  primaryKeyField,
  fields,
  sortField,
} = useCollection(collection);
```

### With Layout Click Handler

Handle item clicks consistently:

```typescript
import { useLayoutClickHandler } from '@/composables/use-layout-click-handler';

const { onClick } = useLayoutClickHandler({
  props,
  selection,
  primaryKeyField,
});
```

## Creating Custom Layouts

### Layout Definition

```typescript
import { defineLayout } from '@directus/extensions';
import CustomLayout from './custom.vue';
import CustomOptions from './options.vue';
import CustomActions from './actions.vue';

export default defineLayout({
  id: 'custom-layout',
  name: 'Custom Layout',
  icon: 'view_module',
  component: CustomLayout,
  slots: {
    options: CustomOptions,
    sidebar: () => undefined,
    actions: CustomActions,
  },
  setup(props, { emit }) {
    const selection = useSync(props, 'selection', emit);
    const layoutOptions = useSync(props, 'layoutOptions', emit);
    const layoutQuery = useSync(props, 'layoutQuery', emit);
    
    const { items, loading } = useItems(props.collection, {
      fields: layoutQuery.value.fields,
      sort: layoutQuery.value.sort,
      limit: layoutQuery.value.limit,
      page: layoutQuery.value.page,
      filter: props.filter,
      search: props.search,
    });
    
    return {
      items,
      loading,
      selection,
      layoutOptions,
      layoutQuery,
    };
  },
});
```

### Layout Component

```vue
<template>
  <div class="custom-layout">
    <div v-if="loading" class="loading">
      <v-progress-circular indeterminate />
    </div>
    
    <div v-else-if="items.length === 0" class="empty">
      <v-info icon="info" :title="t('no_items')" />
    </div>
    
    <div v-else class="items-grid">
      <div
        v-for="item in items"
        :key="item[primaryKeyField.field]"
        class="item-card"
        @click="onClick(item)"
      >
        <!-- Custom item rendering -->
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { toRefs } from 'vue';
import { useI18n } from 'vue-i18n';

const props = defineProps<{
  collection: string;
  selection: (string | number)[];
  items: any[];
  loading: boolean;
  primaryKeyField: any;
  onClick: (item: any) => void;
}>();

const { t } = useI18n();
</script>
```

## Layout Best Practices

### Performance

- Implement virtual scrolling for large datasets
- Use lazy loading for images and media
- Debounce search and filter inputs
- Optimize re-renders with computed properties
- Cache layout state

### User Experience

- Provide loading states
- Show empty states with helpful messages
- Support keyboard navigation
- Implement responsive design
- Maintain selection across layout changes

### Accessibility

- Use semantic HTML
- Provide ARIA labels
- Support keyboard shortcuts
- Ensure sufficient color contrast
- Test with screen readers

### Data Management

- Handle pagination efficiently
- Implement proper error handling
- Support batch operations
- Maintain filter state
- Sync with URL parameters

## Layout Switching

Users can switch between layouts:

```vue
<template>
  <div class="layout-selector">
    <v-button
      v-for="layout in availableLayouts"
      :key="layout.id"
      :active="currentLayout === layout.id"
      @click="currentLayout = layout.id"
    >
      <v-icon :name="layout.icon" />
      {{ layout.name }}
    </v-button>
  </div>
</template>
```

Layout preference is saved per collection:

```typescript
// Save layout preference
await api.patch(`/users/me/presets`, {
  collection: collection.value,
  layout: 'cards',
  layout_options: { size: 'medium' },
  layout_query: { fields: ['*'], sort: ['-created_at'] },
});
```

## Layout Presets

Presets save layout configuration:

```typescript
interface Preset {
  id: string;
  collection: string;
  layout: string;
  layout_options: Record<string, any>;
  layout_query: Query;
  filter: Filter | null;
  search: string | null;
  bookmark: string | null;
}
```

## Next Steps

- [Interface System](./09-interface-system.md) - Input components
- [Display System](./10-display-system.md) - Output components
- [UI Component System](./08-ui-components.md) - Base components
- [Frontend Overview](./07-frontend-overview.md) - Vue app architecture
