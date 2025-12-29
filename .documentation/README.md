# Maily.to Architecture Documentation

This documentation provides deep technical insight into how Maily works under the hood. It's designed for developers who want to understand the implementation, extend the system, or debug issues.

## What is Maily?

Maily is a React-based email editor and renderer library with two main packages:
- **`@maily-to/core`** - A WYSIWYG editor built on TipTap for creating email templates
- **`@maily-to/render`** - Converts the editor's JSON output into production-ready HTML emails

The key insight: email HTML is notoriously difficult to get right (table layouts, inline styles, cross-client compatibility). Maily provides pre-designed components that handle these complexities, letting users focus on content rather than email quirks.

## Package Architecture

```
packages/
├── core/          # Editor component (TipTap-based)
│   ├── src/editor/
│   │   ├── extensions/    # TipTap extensions (slash-command, maily-kit, etc.)
│   │   ├── nodes/         # Custom node types (button, section, columns, variable, etc.)
│   │   ├── components/    # React UI (bubble menus, toolbars)
│   │   └── utils/         # Helper functions
│   └── src/blocks/        # Slash command definitions
│
├── render/        # JSON → HTML transformer
│   └── src/
│       ├── maily.tsx      # Core Maily class (the heart of rendering)
│       ├── render.ts      # Simple render() wrapper
│       └── preheader.ts   # Email preview text handling
│
└── shared/        # Types and constants shared between packages
    └── src/
        └── theme.ts       # Theme type definitions and defaults
```

## Navigation

### Core Concepts
- [Architecture Overview](./core-concepts/architecture.md) - High-level structure and design decisions
- [Domain Model](./core-concepts/domain-model.md) - Core entities and their relationships

### Features
- [Editor Flow](./features/editor/flow.md) - How the editor initializes and processes content
- [Extension System](./features/editor/extensions.md) - TipTap extension architecture
- [Render Pipeline](./features/render/pipeline.md) - JSON to HTML transformation
- [Variables & Dynamic Content](./features/render/variables.md) - Variable system and repeat blocks

### Boundaries
- [External APIs](./boundaries/external-apis.md) - React Email, TipTap, and other dependencies

### Open Questions
- [Open Questions](./open-questions.md) - Areas requiring further investigation
