# Architecture Overview

## The Two-Phase Architecture

Maily separates editing from rendering into two distinct phases with different concerns:

### Phase 1: Editing (`@maily-to/core`)
- **Input:** User interactions
- **Output:** TipTap JSONContent
- **Tech:** TipTap/ProseMirror, React, Radix UI

The editor produces a JSON document that's a portable, serializable representation of the email. This JSON can be stored, sent over the wire, and rendered anywhere.

### Phase 2: Rendering (`@maily-to/render`)
- **Input:** TipTap JSONContent + configuration (variables, theme, etc.)
- **Output:** Production HTML suitable for email clients
- **Tech:** React Email, juice (CSS inlining)

The renderer consumes the JSON and produces email-compatible HTML with inline styles, table-based layouts, and proper email client compatibility.

## Why This Separation Matters

1. **Decoupled concerns:** The editor doesn't need to know about email client quirks
2. **Server-side rendering:** `@maily-to/render` can run on the server without browser APIs
3. **Multiple renderers possible:** The JSON format is the contract; alternative renderers could be built
4. **Template storage:** Store JSON templates and render with different variables at send time

## Core Extension Bundle: MailyKit

The editor loads extensions through `MailyKit` (`packages/core/src/editor/extensions/maily-kit.tsx`), which bundles:

```typescript
// packages/core/src/editor/extensions/maily-kit.tsx:60-119
const extensions: AnyExtension[] = [
  Document.extend({ content: '(block|columns)+' }),
  StarterKit.configure({
    code: { /* styling */ },
    blockquote: { /* styling */ },
    // Disable nodes we're replacing with custom versions
    heading: false,
    paragraph: false,
    horizontalRule: false,
    dropcursor: false,
    document: false,
  }),
  // Custom email-specific nodes
  SectionExtension,
  ColumnsExtension,
  ColumnExtension,
  ButtonExtension,
  LogoExtension,
  ImageExtension,
  RepeatExtension,
  // ... etc
];
```

**Key decision:** TipTap's StarterKit is used but heavily customized. Several default nodes are disabled and replaced with email-optimized versions that:
- Render as tables (for email layout compatibility)
- Support additional attributes (alignment, padding, showIfKey for conditionals)
- Have custom React views for the editor UI

## Rendering Engine: The Maily Class

`packages/render/src/maily.tsx` contains the `Maily` class - the heart of the render pipeline. It's designed as a **method-based node renderer**:

```typescript
// packages/render/src/maily.tsx:613-624
private renderNode(node: JSONContent, options: NodeOptions = {}): JSX.Element | null {
  const type = node.type || '';

  if (type in this) {
    // Call this.paragraph(), this.heading(), this.button(), etc.
    return this[type]?.(node, options) as JSX.Element;
  }

  throw new Error(`Node type "${type}" is not supported.`);
}
```

Each node type has a corresponding method (e.g., `paragraph()`, `button()`, `section()`). This pattern makes it easy to add new node types.

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              EDITOR                                      │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────────────┐     │
│  │   User      │───▶│   TipTap     │───▶│     JSONContent         │     │
│  │   Actions   │    │   Editor     │    │   (stored/transmitted)  │     │
│  └─────────────┘    └──────────────┘    └─────────────────────────┘     │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            RENDERER                                      │
│  ┌─────────────────────────┐    ┌──────────────────┐    ┌────────────┐  │
│  │      Maily Class        │───▶│   React Email    │───▶│  HTML      │  │
│  │   + variables/theme     │    │   Components     │    │  Output    │  │
│  └─────────────────────────┘    └──────────────────┘    └────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

### 1. React NodeViews for Editor Nodes
Complex nodes (Button, Section, Variable, etc.) use TipTap's `ReactNodeViewRenderer` to render React components within ProseMirror. This enables rich interactive UI (bubble menus, popovers) while maintaining ProseMirror's document model.

### 2. Bubble Menus for Contextual Editing
Each node type that needs configuration has a corresponding BubbleMenu component:
- `SectionBubbleMenu` - padding, margin, colors
- `VariableBubbleMenu` - variable name, fallback value
- `ImageBubbleMenu` - size, alignment, link

### 3. Slash Commands for Block Insertion
The `/` trigger opens a command palette defined in `packages/core/src/blocks/`. Commands are organized by category and can insert any node type.

### 4. Conditional Rendering via `showIfKey`
Most block nodes support a `showIfKey` attribute that controls visibility based on runtime data:
```typescript
// At render time
maily.setPayloadValues({ isPremium: true });
// Nodes with showIfKey="isPremium" will render only if isPremium is truthy
```

This enables templates with conditional sections without complex logic.
