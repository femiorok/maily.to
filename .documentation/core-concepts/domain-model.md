# Domain Model

## Core Entity: JSONContent (TipTap Document)

The fundamental data structure is TipTap's `JSONContent`, a tree of nodes:

```typescript
// From @tiptap/core
interface JSONContent {
  type: string;           // Node type: 'paragraph', 'button', 'section', etc.
  attrs?: Record<string, any>;  // Node-specific attributes
  content?: JSONContent[];      // Child nodes
  marks?: MarkType[];           // Inline formatting (bold, link, textStyle)
  text?: string;                // For text nodes only
}
```

A typical email document:
```json
{
  "type": "doc",
  "content": [
    { "type": "logo", "attrs": { "src": "...", "size": "md" } },
    { "type": "heading", "attrs": { "level": 1 }, "content": [{ "type": "text", "text": "Hello" }] },
    { "type": "paragraph", "content": [
      { "type": "text", "text": "Welcome, " },
      { "type": "variable", "attrs": { "id": "name", "fallback": "friend" } }
    ]},
    { "type": "button", "attrs": { "text": "Click Me", "url": "https://..." } }
  ]
}
```

## Node Categories

### Text Nodes (with content)
| Node Type | Purpose | Key Attributes |
|-----------|---------|----------------|
| `paragraph` | Standard text | `textAlign`, `showIfKey` |
| `heading` | Section titles (h1-h3 only) | `level`, `textAlign` |
| `footer` | Footer text (smaller font) | `textAlign` |
| `blockquote` | Quoted content | — |
| `bulletList` / `orderedList` | Lists | — |
| `listItem` | List item container | — |

### Atomic Nodes (no children)
| Node Type | Purpose | Key Attributes |
|-----------|---------|----------------|
| `button` | CTA button | `text`, `url`, `variant`, `borderRadius`, `alignment`, `isTextVariable`, `isUrlVariable` |
| `image` | Block image | `src`, `width`, `height`, `alignment`, `externalLink`, `isSrcVariable` |
| `logo` | Logo image | `src`, `size` (`sm`/`md`/`lg`), `alignment` |
| `spacer` | Vertical space | `height` |
| `variable` | Dynamic placeholder | `id`, `fallback`, `required` |
| `inlineImage` | Inline icon/image | `src`, `width`, `height` (default 20x20) |
| `horizontalRule` | Divider line | — |

### Container Nodes (with content)
| Node Type | Purpose | Key Attributes |
|-----------|---------|----------------|
| `section` | Styled container | `backgroundColor`, `borderRadius`, `padding*`, `margin*`, `borderWidth`, `borderColor` |
| `columns` | Multi-column layout | `gap` |
| `column` | Single column | `width` (percentage or `'auto'`), `verticalAlign` |
| `repeat` | Loop over array | `each` (variable name) |
| `htmlCodeBlock` | Raw HTML injection | — |

### Inline Marks
| Mark Type | Purpose |
|-----------|---------|
| `bold` | Bold text |
| `italic` | Italic text |
| `underline` | Underlined text |
| `strike` | Strikethrough |
| `code` | Inline code |
| `textStyle` | Color customization |
| `link` | Hyperlinks (`href`, `isUrlVariable`) |

## Key Attributes Pattern

Several attributes appear across multiple node types:

### `showIfKey`
Conditional rendering - node only renders if the key exists and is truthy in `payloadValues`:
```typescript
maily.setPayloadValues({ isPremium: true });
// { type: 'section', attrs: { showIfKey: 'isPremium' } } → renders
// { type: 'section', attrs: { showIfKey: 'isAdmin' } }   → hidden
```

### `is*Variable` Flags
When true, the corresponding attribute is treated as a variable name:
- `isTextVariable` + `text: "buttonLabel"` → renders value of `buttonLabel` variable
- `isUrlVariable` + `url: "ctaLink"` → href becomes value of `ctaLink` variable
- `isSrcVariable` + `src: "imageUrl"` → image source from variable

### Alignment
Most blocks support `alignment`: `'left'` | `'center'` | `'right'`

## Variable System

Variables are inline nodes that get replaced at render time:

```typescript
// In editor JSON
{ "type": "variable", "attrs": { "id": "userName", "fallback": "friend" } }

// Rendering without replacement
maily.render() → "{{userName,fallback=friend}}"

// Rendering with replacement
maily.setVariableValue('userName', 'Alice');
maily.render() → "Alice"
```

Variables can also be embedded in button text, URLs, and image sources via the `is*Variable` flags.

## Repeat (Loop) System

The `repeat` node iterates over arrays:

```typescript
// Template
{
  "type": "repeat",
  "attrs": { "each": "products" },
  "content": [
    { "type": "heading", "content": [
      { "type": "variable", "attrs": { "id": "name" } }
    ]},
    { "type": "paragraph", "content": [
      { "type": "variable", "attrs": { "id": "price" } }
    ]}
  ]
}

// Payload
maily.setPayloadValues({
  products: [
    { name: "Widget", price: "$10" },
    { name: "Gadget", price: "$20" }
  ]
});
```

Variables inside `repeat` blocks look up values from the current iteration's object first, then fall back to global `variableValues`.

## Theme Configuration

The renderer accepts theme configuration for styling:

```typescript
// packages/shared/src/theme.ts
interface RendererThemeOptions {
  colors?: {
    heading: string;        // Default: '#111827'
    paragraph: string;      // Default: '#374151'
    footer: string;         // Default: '#64748B'
    horizontal: string;     // Default: '#EAEAEA'
    blockquoteBorder: string;
    codeBackground: string;
    codeText: string;
    linkCardTitle: string;
    // ... more
  };
  fontSize?: {
    paragraph: { size: string; lineHeight: string };
    footer: { size: string; lineHeight: string };
  };
  container?: {
    backgroundColor: string;
    maxWidth: string;       // Default: '600px'
    padding*: string;
    border*: string;
  };
  button?: {
    backgroundColor: string;
    color: string;
    padding*: string;
  };
  font?: {
    fontFamily: string;
    fallbackFontFamily: string;
    webFont?: { url: string; format: string };
  };
}
```

Themes are deep-merged with defaults - you only need to specify what changes.

## Invariants

1. **Document root is always `type: 'doc'`** with `content` array
2. **Block nodes cannot be nested arbitrarily** - only `section`, `columns`, `repeat` can contain other blocks
3. **Columns must contain Column nodes** - `columns.content` must be `column` nodes
4. **Marks only apply to text nodes** - marks wrap `{ type: 'text', text: '...' }` nodes
5. **Variables are inline** - they appear within text content, not as block nodes
