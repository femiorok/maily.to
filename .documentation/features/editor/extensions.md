# Extension System

Maily is built on TipTap, which is built on ProseMirror. Understanding the extension patterns is key to extending or modifying Maily.

## Extension Types

TipTap has three types of extensions:

1. **Node** - Block or inline content (paragraph, button, section)
2. **Mark** - Inline formatting (bold, link, textStyle)
3. **Extension** - Non-content functionality (slash-command, placeholder)

## Custom Node Pattern

Maily's custom nodes follow a consistent pattern. Here's the anatomy of the Button node:

### 1. Type Declaration

```typescript
// packages/core/src/editor/nodes/button/button.tsx:47-54
declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    button: {
      setButton: () => ReturnType;
      updateButton: (attrs: Partial<ButtonAttributes>) => ReturnType;
    };
  }
}
```

This extends TipTap's command interface for TypeScript support.

### 2. Node Creation

```typescript
// packages/core/src/editor/nodes/button/button.tsx:56-226
export const ButtonExtension = Node.create({
  name: 'button',
  group: 'block',  // Can appear where blocks go
  atom: true,      // Cannot have cursor inside
  draggable: true, // Can be dragged

  addAttributes() {
    return {
      text: {
        default: 'Button',
        parseHTML: (element) => element.getAttribute('data-text') || '',
        renderHTML: (attributes) => ({ 'data-text': attributes.text }),
      },
      // ... more attributes
    };
  },

  parseHTML() {
    return [{ tag: `div[data-type="${this.name}"]` }];
  },

  renderHTML({ HTMLAttributes }) {
    return [
      'div',
      mergeAttributes(HTMLAttributes, { 'data-type': this.name }),
    ];
  },

  addCommands() {
    return {
      setButton: () => ({ commands }) => {
        return commands.insertContent({
          type: this.name,
          attrs: {},
          content: [],
        });
      },
      updateButton: (attrs) => updateAttributes(this.name, attrs),
    };
  },

  addNodeView() {
    return ReactNodeViewRenderer(ButtonView, {
      contentDOMElementTag: 'div',
      className: 'mly:relative',
    });
  },
});
```

### 3. React NodeView

For interactive editing UI:

```typescript
// packages/core/src/editor/nodes/button/button-view.tsx
export function ButtonView(props: NodeViewProps) {
  const { node, updateAttributes } = props;
  const { text, alignment, variant, buttonColor, textColor } = node.attrs;

  return (
    <NodeViewWrapper>
      {/* Rich UI with popovers, color pickers, etc. */}
      <div className={/* styling based on variant */}>
        <ButtonLabelInput
          value={text}
          onChange={(newText) => updateAttributes({ text: newText })}
        />
      </div>
    </NodeViewWrapper>
  );
}
```

## MailyKit: The Core Bundle

MailyKit (`packages/core/src/editor/extensions/maily-kit.tsx`) bundles all email-specific extensions:

```typescript
// packages/core/src/editor/extensions/maily-kit.tsx:44-163
export const MailyKit = Extension.create<MailyKitOptions>({
  name: 'maily-kit',

  addOptions() {
    return {
      link: {
        HTMLAttributes: {
          target: '_blank',
          rel: 'noopener noreferrer nofollow',
        },
        openOnClick: false,
      },
    };
  },

  addExtensions() {
    const extensions: AnyExtension[] = [
      // Custom Document that only allows blocks and columns at root
      Document.extend({
        content: '(block|columns)+',
      }),

      // TipTap StarterKit with customizations
      StarterKit.configure({
        code: { HTMLAttributes: { class: '...' } },
        blockquote: { HTMLAttributes: { class: '...' } },
        // Disable nodes we're replacing
        heading: false,
        paragraph: false,
        horizontalRule: false,
        dropcursor: false,
        document: false,
      }),

      // Standard marks/extensions
      Underline,
      Color,
      TextStyle,
      TextAlign.configure({ types: [Paragraph.name, Heading.name, Footer.name] }),
      HorizontalRule,
      Footer,
      Focus,
      Dropcursor,

      // Custom email-optimized nodes
      HeadingExtension,
      ParagraphExtension,
    ];

    // Conditionally add optional extensions
    if (this.options.linkCard !== false) {
      extensions.push(LinkCardExtension.configure(this.options.linkCard));
    }
    if (this.options.section !== false) {
      extensions.push(SectionExtension);
    }
    // ... etc for repeat, columns, button, spacer, logo, image, link

    return extensions;
  },
});
```

**Key insight:** MailyKit is configurable - you can disable specific extensions by passing `false`:

```typescript
MailyKit.configure({
  repeat: false,      // Disable repeat blocks
  linkCard: false,    // Disable link cards
})
```

## Section Node: Table-Based Layout

The Section extension demonstrates email-specific rendering:

```typescript
// packages/core/src/editor/nodes/section/section.ts:294-333
renderHTML({ HTMLAttributes }) {
  const { marginTop, marginRight, marginBottom, marginLeft } = HTMLAttributes;

  return [
    'table',  // Tables for email compatibility
    {
      'data-type': this.name,
      border: 0,
      cellpadding: 0,
      cellspacing: 0,
      class: 'mly:w-full mly:border-separate mly:relative mly:table-fixed',
      style: `margin-top: ${marginTop}px; ...`,
    },
    [
      'tbody',
      { class: 'mly:w-full' },
      [
        'tr',
        { class: 'mly:w-full' },
        [
          'td',
          mergeAttributes(HTMLAttributes, {
            'data-type': 'section-cell',
            style: 'border-style: solid',
          }),
          0,  // Content hole - children go here
        ],
      ],
    ],
  ];
},
```

The editor renders tables for visual consistency with what the email will look like. The render package produces similar table-based output.

## Columns: Tab Navigation

The Columns extension adds keyboard navigation:

```typescript
// packages/core/src/editor/nodes/columns/columns.ts:126-135
addKeyboardShortcuts() {
  return {
    Tab: () => {
      return goToColumn(this.editor, 'next');
    },
    'Shift-Tab': () => {
      return goToColumn(this.editor, 'previous');
    },
  };
},
```

This allows Tab to move between columns, matching expected behavior.

## Variable: Suggestion Plugin

The Variable extension uses TipTap's Suggestion plugin for @ mentions:

```typescript
// packages/core/src/editor/nodes/variable/variable.ts:92-289
export const VariableExtension = Node.create<VariableOptions, VariableStorage>({
  name: 'variable',
  group: 'inline',
  inline: true,
  selectable: true,
  atom: true,  // Can't put cursor inside

  addStorage() {
    return { popover: false };  // UI state
  },

  addProseMirrorPlugins() {
    return [
      Suggestion({
        editor: this.editor,
        ...this.options.suggestion,
      }),
    ];
  },

  addKeyboardShortcuts() {
    return {
      Backspace: () =>
        this.editor.commands.command(({ tr, state }) => {
          // When backspacing into a variable, convert back to @ trigger
          let isMention = false;
          const { selection } = state;
          const { empty, anchor } = selection;

          if (!empty) return false;

          state.doc.nodesBetween(anchor - 1, anchor, (node, pos) => {
            if (node.type.name === this.name) {
              isMention = true;
              tr.insertText('@', pos, pos + node.nodeSize);
              return false;
            }
          });

          return isMention;
        }),
    };
  },
});
```

**Key insight:** Backspacing into a variable converts it back to `@`, allowing the user to modify the variable selection.

## Adding Custom Extensions

To add a custom node:

1. Create the extension following the pattern above
2. Either:
   - Add to MailyKit's `addExtensions()`
   - Pass via the `extensions` prop to `<Editor>`

```typescript
<Editor
  extensions={[
    MyCustomExtension.configure({ ... })
  ]}
/>
```

Extensions with the same name will be deduplicated - user-provided extensions take precedence over defaults.
