# Display System

This document covers the output display system in Directus, which provides 18 display components for rendering field values in the admin app.

## Overview

Displays are read-only components that render field values in various formats throughout the Directus admin interface. They are used in tables, cards, detail views, and other layouts to present data to users.

## Display Architecture

### Display Structure

Each display consists of:
- **index.ts** - Display configuration and metadata
- **{name}.vue** - Vue component for rendering
- **options.vue** (optional) - Configuration options component

### Display Registration (`app/src/displays/index.ts`)

```typescript
export function getInternalDisplays(): DisplayConfig[] {
  const displays = import.meta.glob<DisplayConfig>('./*/index.ts', {
    import: 'default',
    eager: true
  });
  
  return sortBy(Object.values(displays), 'id');
}

export function registerDisplays(displays: DisplayConfig[], app: App): void {
  for (const display of displays) {
    // Register main component
    app.component(`display-${display.id}`, display.component);
    
    // Register options component if exists
    if (display.options && typeof display.options !== 'function') {
      app.component(`display-options-${display.id}`, display.options);
    }
  }
}
```

### Display Configuration

```typescript
import { defineDisplay } from '@directus/extensions';

export default defineDisplay({
  id: 'boolean',
  name: '$t:displays.boolean.boolean',
  description: '$t:displays.boolean.description',
  types: ['boolean'],
  icon: 'check_box',
  component: DisplayBoolean,
  options: [
    {
      field: 'labelOn',
      name: '$t:displays.boolean.label_on',
      type: 'string',
      meta: {
        interface: 'system-input-translated-string',
        width: 'half',
      },
    },
    {
      field: 'iconOn',
      name: '$t:displays.boolean.icon_on',
      type: 'string',
      meta: {
        interface: 'select-icon',
        width: 'half',
      },
      schema: {
        default_value: 'check',
      },
    },
    // ... more options
  ],
});
```

## Display Types

### Basic Displays

#### raw
Displays raw value without any formatting.

**Supported Types:** All types

**Options:** None

**Usage:**
```vue
<display-raw :value="value" />
```

**Output:**
```
"Hello World"
123
true
```

#### formatted-value
Formats value based on field type with locale-aware formatting.

**Supported Types:** All types

**Options:**
- `format` - Display format
- `prefix` - Text prefix
- `suffix` - Text suffix
- `decimals` - Decimal places (for numbers)
- `thousandsSeparator` - Thousands separator
- `iconLeft` - Left icon
- `iconRight` - Right icon
- `font` - Font family

**Features:**
- Number formatting with locale
- Date/time formatting
- Boolean formatting
- String truncation
- Custom prefixes/suffixes

**Example:**
```typescript
// Number: 1234.56 → "$1,234.56"
{ format: true, prefix: '$', decimals: 2 }

// Date: 2024-01-15 → "January 15, 2024"
{ format: 'long' }
```

#### formatted-json-value
Formats JSON with syntax highlighting.

**Supported Types:** `json`

**Options:**
- `format` - Pretty print JSON

**Output:**
```json
{
  "name": "John Doe",
  "age": 30,
  "active": true
}
```

### Content Displays

#### boolean
Displays boolean value with customizable icons and labels.

**Supported Types:** `boolean`

**Options:**
- `labelOn` - Label for true value
- `labelOff` - Label for false value
- `iconOn` - Icon for true value (default: 'check')
- `iconOff` - Icon for false value (default: 'close')
- `colorOn` - Color for true value
- `colorOff` - Color for false value

**Example:**
```vue
<!-- True: ✓ Active -->
<!-- False: ✗ Inactive -->
<display-boolean
  :value="true"
  label-on="Active"
  label-off="Inactive"
  icon-on="check"
  icon-off="close"
/>
```

#### datetime
Formats date and time values with locale support.

**Supported Types:** `date`, `time`, `datetime`, `timestamp`

**Options:**
- `format` - Date format (short, long, etc.)
- `relative` - Show relative time (e.g., "2 hours ago")

**Formats:**
- `short` - 1/15/24
- `long` - January 15, 2024
- `relative` - 2 hours ago

**Example:**
```typescript
// Absolute: "January 15, 2024 at 3:30 PM"
{ format: 'long' }

// Relative: "2 hours ago"
{ relative: true }
```

#### color
Displays color value with swatch.

**Supported Types:** `string`

**Options:**
- `defaultColor` - Default color if value is null

**Output:**
```
[■] #FF5733
```

#### icon
Displays icon from icon library.

**Supported Types:** `string`

**Options:**
- `filled` - Use filled icon variant

**Example:**
```vue
<display-icon value="home" />
<!-- Renders: 🏠 -->
```

#### image
Displays image with lightbox support.

**Supported Types:** `uuid` (file reference)

**Options:**
- `circle` - Display as circle
- `lightbox` - Enable lightbox on click

**Features:**
- Thumbnail generation
- Lazy loading
- Lightbox preview
- Fallback for missing images

#### file
Displays file information with download link.

**Supported Types:** `uuid` (file reference)

**Options:**
- `iconLeft` - Left icon
- `iconRight` - Right icon

**Output:**
```
📄 document.pdf (2.5 MB)
```

#### filesize
Displays file size in human-readable format.

**Supported Types:** `integer`, `bigInteger`

**Options:** None

**Example:**
```typescript
1024 → "1 KB"
1048576 → "1 MB"
1073741824 → "1 GB"
```

#### mime-type
Displays MIME type with icon.

**Supported Types:** `string`

**Options:** None

**Output:**
```
📄 application/pdf
🖼️ image/jpeg
📹 video/mp4
```

### Relationship Displays

#### related-values
Displays values from related items.

**Supported Types:** Relational fields

**Options:**
- `template` - Display template for related items

**Template Syntax:**
```
{{field_name}} - {{other_field}}
```

**Example:**
```typescript
// Template: "{{first_name}} {{last_name}}"
// Output: "John Doe"

// Template: "{{title}} ({{year}})"
// Output: "The Matrix (1999)"
```

#### user
Displays user information with avatar.

**Supported Types:** `uuid` (user reference)

**Options:**
- `circle` - Display avatar as circle
- `display` - Display format (avatar, name, both)

**Output:**
```
[👤] John Doe
```

**Features:**
- User avatar
- User name
- User email
- Online status indicator

#### collection
Displays collection name.

**Supported Types:** `string`

**Options:**
- `icon` - Show collection icon
- `color` - Show collection color

**Output:**
```
📚 Articles
```

### Advanced Displays

#### labels
Displays tags/labels with colors.

**Supported Types:** `json`, `csv`, `string`

**Options:**
- `format` - Display format (badge, pill, text)
- `defaultForeground` - Default text color
- `defaultBackground` - Default background color
- `choices` - Label definitions with colors

**Choices Format:**
```json
[
  {
    "text": "Published",
    "value": "published",
    "foreground": "#FFFFFF",
    "background": "#00C897"
  },
  {
    "text": "Draft",
    "value": "draft",
    "foreground": "#FFFFFF",
    "background": "#A2B5CD"
  }
]
```

**Output:**
```
[Published] [Featured] [Breaking News]
```

#### rating
Displays star rating.

**Supported Types:** `integer`, `float`, `decimal`

**Options:**
- `maxRating` - Maximum rating value (default: 5)
- `simple` - Simple display without stars

**Example:**
```vue
<!-- Value: 4 -->
★★★★☆
```

#### hash
Displays masked hash/password value.

**Supported Types:** `hash`

**Options:** None

**Output:**
```
••••••••••••
```

#### translations
Displays multi-language content.

**Supported Types:** Relational field (O2M)

**Options:**
- `template` - Display template
- `languageField` - Language code field
- `defaultLanguage` - Default language

**Output:**
```
🇺🇸 English | 🇪🇸 Spanish | 🇫🇷 French
```

## Display Usage

### In Tables

Displays are automatically used in table layouts:

```vue
<v-table
  :headers="headers"
  :items="items"
>
  <template #item.status="{ item }">
    <display-labels :value="item.status" :choices="statusChoices" />
  </template>
</v-table>
```

### In Cards

Displays render field values in card layouts:

```vue
<v-card>
  <template #title>
    <display-formatted-value :value="item.title" />
  </template>
  
  <template #subtitle>
    <display-datetime :value="item.created_at" :relative="true" />
  </template>
</v-card>
```

### In Detail Views

Displays show field values in detail/form views:

```vue
<div class="field-preview">
  <component
    :is="`display-${field.meta.display}`"
    :value="item[field.field]"
    v-bind="field.meta.display_options"
  />
</div>
```

### Dynamic Display Selection

The system automatically selects appropriate displays:

```typescript
function getDefaultDisplayForType(type: string): string {
  const displayMap: Record<string, string> = {
    'boolean': 'boolean',
    'date': 'datetime',
    'time': 'datetime',
    'datetime': 'datetime',
    'timestamp': 'datetime',
    'json': 'formatted-json-value',
    'uuid': 'formatted-value',
    'hash': 'hash',
    'string': 'formatted-value',
    'text': 'formatted-value',
    'integer': 'formatted-value',
    'bigInteger': 'formatted-value',
    'float': 'formatted-value',
    'decimal': 'formatted-value',
  };
  
  return displayMap[type] || 'raw';
}
```

## Display Configuration

### Field Display Settings

Displays are configured in field metadata:

```typescript
{
  collection: 'articles',
  field: 'status',
  type: 'string',
  meta: {
    display: 'labels',
    display_options: {
      format: 'badge',
      choices: [
        {
          text: 'Published',
          value: 'published',
          foreground: '#FFFFFF',
          background: '#00C897'
        },
        {
          text: 'Draft',
          value: 'draft',
          foreground: '#FFFFFF',
          background: '#A2B5CD'
        }
      ]
    }
  }
}
```

### Layout-Specific Displays

Different displays can be used in different layouts:

```typescript
// Table layout - compact display
{
  layout: 'tabular',
  display: 'labels',
  display_options: { format: 'badge' }
}

// Card layout - detailed display
{
  layout: 'cards',
  display: 'formatted-value',
  display_options: { font: 'serif' }
}
```

## Creating Custom Displays

### Display Definition

```typescript
import { defineDisplay } from '@directus/extensions';
import DisplayComponent from './display.vue';

export default defineDisplay({
  id: 'custom-display',
  name: 'Custom Display',
  description: 'A custom display component',
  icon: 'extension',
  component: DisplayComponent,
  types: ['string', 'text'],
  options: [
    {
      field: 'uppercase',
      name: 'Uppercase',
      type: 'boolean',
      meta: {
        interface: 'boolean',
        width: 'half',
      },
      schema: {
        default_value: false,
      },
    },
  ],
});
```

### Display Component

```vue
<template>
  <div class="custom-display">
    <span :class="{ uppercase: uppercase }">
      {{ formattedValue }}
    </span>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue';

interface Props {
  value: string | null;
  uppercase?: boolean;
  type?: string;
  collection?: string;
  field?: string;
}

const props = withDefaults(defineProps<Props>(), {
  uppercase: false,
});

const formattedValue = computed(() => {
  if (!props.value) return '—';
  return props.uppercase ? props.value.toUpperCase() : props.value;
});
</script>

<style scoped>
.custom-display {
  display: inline-block;
}

.uppercase {
  text-transform: uppercase;
}
</style>
```

## Display vs Interface

### Key Differences

| Aspect | Interface | Display |
|--------|-----------|---------|
| Purpose | Input/editing | Output/viewing |
| Interaction | Interactive | Read-only |
| Complexity | Complex with validation | Simple rendering |
| Events | Emits input events | No events |
| Use Cases | Forms, editors | Tables, cards, previews |

### When to Use Each

**Use Interfaces when:**
- User needs to input or edit data
- Validation is required
- Complex interactions are needed
- Data transformation on input

**Use Displays when:**
- Showing read-only data
- Rendering in tables or lists
- Previewing values
- Formatting output

## Display Best Practices

### Performance

- Keep displays lightweight
- Avoid heavy computations
- Use computed properties for formatting
- Implement lazy loading for images

### Accessibility

- Provide text alternatives for icons
- Use semantic HTML
- Ensure sufficient color contrast
- Support screen readers

### Consistency

- Follow Directus design patterns
- Use consistent formatting
- Match interface behavior where applicable
- Provide meaningful fallbacks for null values

### Localization

- Support i18n for labels
- Use locale-aware formatting
- Handle RTL languages
- Format dates/numbers per locale

## Common Display Patterns

### Conditional Rendering

```vue
<template>
  <div class="display-wrapper">
    <template v-if="value !== null">
      <v-icon v-if="showIcon" :name="icon" />
      <span>{{ formattedValue }}</span>
    </template>
    <span v-else class="empty">—</span>
  </div>
</template>
```

### Value Formatting

```typescript
const formattedValue = computed(() => {
  if (props.value === null) return null;
  
  switch (props.type) {
    case 'integer':
    case 'float':
    case 'decimal':
      return formatNumber(props.value, props.decimals);
    case 'date':
    case 'datetime':
      return formatDate(props.value, props.format);
    default:
      return String(props.value);
  }
});
```

### Relational Data

```vue
<template>
  <div v-if="relatedItem">
    <display-formatted-value
      :value="relatedItem"
      :template="template"
    />
  </div>
  <span v-else class="empty">—</span>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';
import { useApi } from '@directus/composables';

const props = defineProps<{
  value: string | null;
  collection: string;
  template: string;
}>();

const api = useApi();
const relatedItem = ref(null);

watch(() => props.value, async (newValue) => {
  if (!newValue) {
    relatedItem.value = null;
    return;
  }
  
  const response = await api.get(`/items/${props.collection}/${newValue}`);
  relatedItem.value = response.data.data;
}, { immediate: true });
</script>
```

## Display Integration

### With VTable

```typescript
const headers = computed<HeaderRaw[]>(() => [
  {
    text: 'Name',
    value: 'name',
    width: 200,
    display: 'formatted-value',
  },
  {
    text: 'Status',
    value: 'status',
    width: 120,
    display: 'labels',
    displayOptions: {
      choices: statusChoices,
    },
  },
  {
    text: 'Created',
    value: 'created_at',
    width: 150,
    display: 'datetime',
    displayOptions: {
      relative: true,
    },
  },
]);
```

### With Layouts

Layouts automatically use displays for field rendering:

```typescript
// Tabular layout
<layout-tabular
  :collection="collection"
  :fields="fields"
  :items="items"
/>

// Cards layout
<layout-cards
  :collection="collection"
  :fields="fields"
  :items="items"
/>
```

### With Field Templates

Use displays in custom templates:

```vue
<template>
  <div class="custom-template">
    <component
      v-for="field in fields"
      :key="field.field"
      :is="`display-${field.meta.display || 'formatted-value'}`"
      :value="item[field.field]"
      v-bind="field.meta.display_options"
    />
  </div>
</template>
```

## Next Steps

- [Interface System](./09-interface-system.md) - Input interface components
- [Layout System](./11-layout-system.md) - Content layout components
- [UI Component System](./08-ui-components.md) - Base UI components
