# Interface System

This document covers the input interface system in Directus, which provides 40+ field input components for the admin app.

## Overview

Interfaces are input components that allow users to interact with field data in the Directus admin app. Each interface is designed for specific data types and use cases.

## Interface Architecture

### Interface Structure

Each interface consists of:
- **index.ts** - Interface configuration and metadata
- **{name}.vue** - Vue component for rendering
- **options.vue** (optional) - Configuration options component
- **preview.svg** - Preview image for interface selector

### Interface Registration (`app/src/interfaces/index.ts`)

```typescript
export function getInternalInterfaces(): InterfaceConfig[] {
  const interfaces = import.meta.glob<InterfaceConfig>(
    ['./*/index.ts', './_system/*/index.ts'],
    { import: 'default', eager: true }
  );
  
  return sortBy(Object.values(interfaces), 'id');
}

export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
  for (const inter of interfaces) {
    // Register main component
    app.component(`interface-${inter.id}`, inter.component);
    
    // Register options component if exists
    if (inter.options && typeof inter.options !== 'function') {
      app.component(`interface-options-${inter.id}`, inter.options);
    }
  }
}
```

### Interface Configuration

```typescript
import { defineInterface } from '@directus/extensions';

export default defineInterface({
  id: 'input',
  name: '$t:interfaces.input.input',
  description: '$t:interfaces.input.description',
  icon: 'text_fields',
  component: InterfaceInput,
  types: ['string', 'uuid', 'text'],
  group: 'standard',
  options: [
    {
      field: 'placeholder',
      name: '$t:placeholder',
      type: 'string',
      meta: {
        width: 'full',
        interface: 'system-input-translated-string',
        options: {
          placeholder: '$t:enter_a_placeholder',
        },
      },
    },
    // ... more options
  ],
  recommendedDisplays: ['formatted-value'],
});
```

## Interface Categories

### Text Input Interfaces

#### input
Basic text input with validation and formatting.

**Supported Types:** `string`, `uuid`, `bigInteger`, `integer`, `float`, `decimal`, `text`

**Options:**
- `placeholder` - Placeholder text
- `iconLeft` - Icon on the left side
- `iconRight` - Icon on the right side
- `font` - Font family (sans-serif, serif, monospace)
- `trim` - Trim whitespace
- `masked` - Mask input (for passwords)
- `clear` - Show clear button
- `slug` - Auto-generate slug

**Usage:**
```vue
<interface-input
  v-model="value"
  :placeholder="'Enter text...'"
  :icon-left="'search'"
/>
```

#### input-multiline
Multi-line text input with auto-resize.

**Supported Types:** `text`, `string`

**Options:**
- `placeholder` - Placeholder text
- `trim` - Trim whitespace
- `font` - Font family
- `softLength` - Soft character limit
- `clear` - Show clear button

#### input-rich-text-html
WYSIWYG HTML editor powered by TinyMCE.

**Supported Types:** `text`, `string`

**Options:**
- `toolbar` - Toolbar buttons configuration
- `customFormats` - Custom text formats
- `tinymceOverrides` - TinyMCE configuration overrides
- `imageToken` - Static token for image uploads

**Features:**
- Rich text formatting
- Image upload and management
- Link insertion
- Code blocks
- Tables
- Media embeds

#### input-rich-text-md
Markdown editor with live preview.

**Supported Types:** `text`, `string`

**Options:**
- `placeholder` - Placeholder text
- `customSyntax` - Custom markdown syntax
- `imageToken` - Static token for image uploads

#### input-block-editor
Block-based editor powered by Editor.js.

**Supported Types:** `json`

**Options:**
- `placeholder` - Placeholder text
- `tools` - Available block tools
- `imageToken` - Static token for image uploads

**Available Tools:**
- Header
- Paragraph
- List (ordered/unordered)
- Quote
- Code
- Image
- Delimiter
- Table
- Raw HTML

#### input-code
Code editor with syntax highlighting (CodeMirror).

**Supported Types:** `text`, `string`, `json`

**Options:**
- `language` - Programming language
- `lineNumber` - Show line numbers
- `template` - Code template
- `lineWrapping` - Enable line wrapping

**Supported Languages:**
- JavaScript, TypeScript
- HTML, CSS, SCSS
- JSON, YAML
- Python, PHP
- SQL
- Markdown
- And more...

#### input-hash
Password/hash input with generation.

**Supported Types:** `hash`

**Options:**
- `placeholder` - Placeholder text
- `masked` - Mask input
- `iconLeft` - Left icon

#### input-autocomplete-api
Autocomplete input with API-powered suggestions.

**Supported Types:** `string`

**Options:**
- `url` - API endpoint URL
- `resultsPath` - Path to results in response
- `textPath` - Path to display text
- `valuePath` - Path to value
- `trigger` - Trigger character (e.g., '@')

### Selection Interfaces

#### select-dropdown
Single selection dropdown.

**Supported Types:** `string`, `integer`, `float`, `decimal`

**Options:**
- `choices` - Available choices
- `allowOther` - Allow custom values
- `allowNone` - Allow deselection
- `icon` - Dropdown icon
- `placeholder` - Placeholder text

**Choices Format:**
```json
[
  { "text": "Option 1", "value": "option1" },
  { "text": "Option 2", "value": "option2" }
]
```

#### select-multiple-dropdown
Multi-selection dropdown.

**Supported Types:** `json`, `csv`

**Options:**
- `choices` - Available choices
- `allowOther` - Allow custom values
- `closeOnSelect` - Close after selection
- `placeholder` - Placeholder text

#### select-radio
Radio button selection.

**Supported Types:** `string`, `integer`, `float`, `decimal`

**Options:**
- `choices` - Available choices
- `allowOther` - Allow custom values
- `iconOn` - Icon for selected state
- `iconOff` - Icon for unselected state
- `color` - Color for selected state

#### select-multiple-checkbox
Checkbox group selection.

**Supported Types:** `json`, `csv`

**Options:**
- `choices` - Available choices
- `allowOther` - Allow custom values
- `iconOn` - Icon for checked state
- `iconOff` - Icon for unchecked state
- `color` - Color for checked state
- `itemsShown` - Number of items to show

#### select-multiple-checkbox-tree
Hierarchical checkbox selection.

**Supported Types:** `json`, `csv`

**Options:**
- `choices` - Hierarchical choices
- `valueField` - Field for value
- `textField` - Field for display text
- `childrenField` - Field for children

#### select-color
Color picker interface.

**Supported Types:** `string`

**Options:**
- `presets` - Preset colors
- `allowAlpha` - Allow alpha channel
- `allowInput` - Allow manual input
- `width` - Picker width

#### select-icon
Icon selection interface.

**Supported Types:** `string`

**Options:**
- `placeholder` - Placeholder text

**Features:**
- Search icons
- Browse by category
- Preview selected icon

### Relationship Interfaces

#### select-dropdown-m2o
Many-to-One relationship selector.

**Supported Types:** `uuid`, `string`, `integer`, `bigInteger`

**Options:**
- `template` - Display template
- `filter` - Filter related items
- `enableCreate` - Allow creating new items
- `enableSelect` - Allow selecting existing items

**Template Syntax:**
```
{{field_name}} - {{other_field}}
```

#### list-o2m
One-to-Many relationship management.

**Supported Types:** Relational field

**Options:**
- `template` - Display template
- `enableCreate` - Allow creating items
- `enableSelect` - Allow selecting items
- `filter` - Filter related items
- `layout` - Display layout (list, table)
- `fields` - Fields to display

#### list-m2m
Many-to-Many relationship management.

**Supported Types:** Relational field

**Options:**
- `template` - Display template
- `enableCreate` - Allow creating items
- `enableSelect` - Allow selecting items
- `filter` - Filter related items
- `layout` - Display layout

#### list-m2a
Many-to-Any (polymorphic) relationship management.

**Supported Types:** Relational field

**Options:**
- `template` - Display template per collection
- `enableCreate` - Allow creating items
- `enableSelect` - Allow selecting items
- `filter` - Filter per collection

#### list-o2m-tree-view
Hierarchical One-to-Many relationship.

**Supported Types:** Relational field

**Options:**
- `template` - Display template
- `enableCreate` - Allow creating items
- `enableSelect` - Allow selecting items
- `filter` - Filter related items

#### collection-item-dropdown
Single item selection from collection.

**Supported Types:** `uuid`, `string`, `integer`, `bigInteger`

**Options:**
- `collection` - Target collection
- `template` - Display template
- `filter` - Filter items

#### collection-item-multiple-dropdown
Multiple item selection from collection.

**Supported Types:** `json`, `csv`

**Options:**
- `collection` - Target collection
- `template` - Display template
- `filter` - Filter items

### File & Media Interfaces

#### file
Single file upload and management.

**Supported Types:** `uuid`

**Options:**
- `folder` - Default upload folder
- `crop` - Enable image cropping

**Features:**
- Drag and drop upload
- File preview
- File metadata editing
- Replace file
- Remove file

#### file-image
Image-specific upload with preview.

**Supported Types:** `uuid`

**Options:**
- `folder` - Default upload folder
- `crop` - Enable cropping

**Features:**
- Image preview
- Crop and edit
- Focal point selection
- Image metadata

#### files
Multiple file upload and management.

**Supported Types:** Relational field (O2M)

**Options:**
- `folder` - Default upload folder
- `template` - Display template

### Specialized Interfaces

#### boolean
Boolean toggle/checkbox.

**Supported Types:** `boolean`

**Options:**
- `label` - Label text
- `color` - Toggle color
- `iconOn` - Icon for true state
- `iconOff` - Icon for false state

#### datetime
Date and time picker.

**Supported Types:** `date`, `time`, `datetime`, `timestamp`

**Options:**
- `includeSeconds` - Include seconds
- `use24` - Use 24-hour format
- `min` - Minimum date/time
- `max` - Maximum date/time

#### slider
Numeric range slider.

**Supported Types:** `integer`, `float`, `decimal`

**Options:**
- `min` - Minimum value
- `max` - Maximum value
- `step` - Step increment
- `minLabel` - Label for minimum
- `maxLabel` - Label for maximum
- `showValue` - Show current value

#### map
Geographic coordinate picker.

**Supported Types:** `json`, `geometry`

**Options:**
- `defaultView` - Default map view
- `geometryType` - Geometry type (Point, LineString, Polygon)
- `geometryFormat` - Format (GeoJSON, WKT)

**Features:**
- Interactive map
- Draw shapes
- Search locations
- Coordinate input

#### tags
Tag input with autocomplete.

**Supported Types:** `json`, `csv`

**Options:**
- `placeholder` - Placeholder text
- `alphabetize` - Sort tags alphabetically
- `allowCustom` - Allow custom tags
- `whitespace` - Whitespace handling
- `capitalization` - Capitalization mode
- `iconRight` - Right icon
- `presets` - Preset tags

#### translations
Multi-language content management.

**Supported Types:** Relational field (O2M)

**Options:**
- `languageField` - Language code field
- `defaultLanguage` - Default language
- `userLanguage` - Use user's language

### Grouping Interfaces

#### group-accordion
Collapsible field groups.

**Supported Types:** `alias`

**Options:**
- `start` - Start collapsed/expanded
- `headerIcon` - Header icon
- `headerColor` - Header color

#### group-detail
Detailed field groups.

**Supported Types:** `alias`

**Options:**
- `headerIcon` - Header icon
- `headerColor` - Header color

#### group-raw
Raw JSON field groups.

**Supported Types:** `json`

**Options:**
- `language` - Code language
- `template` - JSON template

### Presentation Interfaces

#### presentation-divider
Visual section dividers.

**Supported Types:** `alias`

**Options:**
- `icon` - Divider icon
- `color` - Divider color
- `title` - Divider title
- `inlineTitle` - Inline title

#### presentation-header
Section headers.

**Supported Types:** `alias`

**Options:**
- `icon` - Header icon
- `color` - Header color
- `title` - Header title
- `description` - Header description

#### presentation-links
Link collections.

**Supported Types:** `alias`

**Options:**
- `links` - Link definitions

#### presentation-notice
Informational notices.

**Supported Types:** `alias`

**Options:**
- `color` - Notice color
- `icon` - Notice icon
- `text` - Notice text

### System Interfaces

Located in `app/src/interfaces/_system/`:

- **system-collection** - Collection selector
- **system-field** - Field selector
- **system-interface** - Interface selector
- **system-display** - Display selector
- **system-filter** - Filter builder
- **system-permissions** - Permission editor
- **system-mfa-setup** - MFA configuration
- **system-token** - Token generator
- **system-theme** - Theme selector
- And 20+ more system interfaces

## Interface Integration

### Form System Integration

Interfaces integrate with the VForm component:

```vue
<v-form
  v-model="edits"
  :fields="fields"
  :initial-values="item"
  :validation-errors="validationErrors"
/>
```

The form automatically selects the appropriate interface based on field configuration:

```typescript
const interfaceId = field.meta?.interface || getDefaultInterfaceForType(field.type);
const component = `interface-${interfaceId}`;
```

### Field Configuration

Fields are configured in `directus_fields`:

```typescript
{
  collection: 'articles',
  field: 'title',
  type: 'string',
  meta: {
    interface: 'input',
    options: {
      placeholder: 'Enter article title...',
      iconLeft: 'title',
      trim: true,
    },
    width: 'full',
    required: true,
    readonly: false,
    hidden: false,
  },
  schema: {
    max_length: 255,
    is_nullable: false,
  }
}
```

### Conditional Fields

Interfaces can be shown/hidden based on conditions:

```typescript
{
  field: 'discount_amount',
  meta: {
    interface: 'input',
    options: { ... },
    conditions: [
      {
        rule: {
          _and: [
            {
              has_discount: {
                _eq: true
              }
            }
          ]
        },
        hidden: false,
        required: true,
      }
    ]
  }
}
```

## Creating Custom Interfaces

### Interface Definition

```typescript
import { defineInterface } from '@directus/extensions';
import InterfaceComponent from './interface.vue';

export default defineInterface({
  id: 'custom-interface',
  name: 'Custom Interface',
  description: 'A custom interface',
  icon: 'extension',
  component: InterfaceComponent,
  types: ['string'],
  group: 'standard',
  options: [
    {
      field: 'customOption',
      name: 'Custom Option',
      type: 'string',
      meta: {
        interface: 'input',
        width: 'half',
      },
    },
  ],
});
```

### Interface Component

```vue
<template>
  <div class="custom-interface">
    <input
      :value="value"
      @input="$emit('input', $event.target.value)"
      :placeholder="placeholder"
    />
  </div>
</template>

<script setup lang="ts">
defineProps<{
  value: string | null;
  placeholder?: string;
  disabled?: boolean;
  // ... other props
}>();

defineEmits<{
  (e: 'input', value: string): void;
}>();
</script>
```

## Best Practices

### Interface Selection

- Choose interfaces that match the data type
- Consider user experience and data entry patterns
- Use appropriate validation and constraints
- Provide helpful placeholders and labels

### Performance

- Lazy load heavy interfaces (rich text, code editor)
- Optimize API calls in autocomplete interfaces
- Use debouncing for search inputs
- Implement virtual scrolling for large lists

### Accessibility

- Provide proper labels and ARIA attributes
- Support keyboard navigation
- Ensure sufficient color contrast
- Test with screen readers

## Next Steps

- [Display System](./10-display-system.md) - Output display components
- [UI Component System](./08-ui-components.md) - Base UI components
- [Frontend Overview](./07-frontend-overview.md) - Vue app architecture
