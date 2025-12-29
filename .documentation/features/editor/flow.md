# Editor Flow

## Initialization

The editor component (`packages/core/src/editor/index.tsx`) orchestrates the entire editor lifecycle:

### 1. Content Preparation

```typescript
// packages/core/src/editor/index.tsx:80-104
const formattedContent = useMemo(() => {
  if (contentJson) {
    // Ensure doc wrapper exists
    const json = contentJson?.type === 'doc'
      ? contentJson
      : { type: 'doc', content: contentJson };

    // Handle deprecated nodes (e.g., 'for' → 'repeat')
    return replaceDeprecatedNode(json);
  } else if (contentHtml) {
    return contentHtml;
  } else {
    // Empty default
    return { type: 'doc', content: [{ type: 'paragraph', content: [] }] };
  }
}, [contentHtml, contentJson]);
```

**Key insight:** The editor accepts either `contentJson` or `contentHtml`, never both. HTML input is for importing from other sources; JSON is the preferred format for templates.

### 2. TipTap Editor Creation

```typescript
// packages/core/src/editor/index.tsx:106-130
const editor = useEditor({
  editorProps: {
    scrollThreshold,
    scrollMargin,
    attributes: {
      class: cn('mly:prose mly:w-full', contentClassName),
      spellCheck: spellCheck ? 'true' : 'false',
    },
  },
  immediatelyRender, // SSR support
  onCreate: ({ editor }) => {
    onCreate?.(editor);  // User callback
  },
  onUpdate: ({ editor }) => {
    onUpdate?.(editor);  // User callback - typically saves editor.getJSON()
  },
  extensions: defaultExtensions({
    extensions,  // User-provided additional extensions
    blocks,      // Custom slash command blocks
  }),
  content: formattedContent,
  autofocus,
  editable,
});
```

### 3. Extension Loading

The `extensions` function (`packages/core/src/editor/extensions/index.tsx`) assembles the extension stack:

```typescript
// packages/core/src/editor/extensions/index.tsx:17-37
export function extensions(props: ExtensionsProps) {
  const { blocks, extensions = [] } = props;

  const defaultExtensions = [
    MailyKit,                        // Core bundle with all email nodes
    SlashCommandExtension.configure({
      suggestion: getSlashCommandSuggestions(blocks),
    }),
    VariableExtension.configure({
      suggestion: getVariableSuggestions(),
    }),
    HTMLCodeBlockExtension,
    InlineImageExtension,
    PlaceholderExtension,
    SelectionExtension,
  ].filter((ext) => {
    // Don't duplicate if user provided same extension
    return !extensions.some((e) => e.name === ext.name);
  });

  return [...defaultExtensions, ...extensions];
}
```

### 4. UI Composition

The editor renders with multiple bubble menus, each watching for their respective node types:

```typescript
// packages/core/src/editor/index.tsx:136-168
return (
  <MailyProvider placeholderUrl={placeholderUrl}>
    <div id="mly-editor" ref={menuContainerRef}>
      {hasMenuBar && <EditorMenuBar config={props.config} editor={editor} />}
      <div>
        {/* Bubble menus - positioned absolutely, appear when relevant node selected */}
        <TextBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <ImageBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <SpacerBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <SectionBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <ColumnsBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <VariableBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <RepeatBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <HTMLBubbleMenu editor={editor} appendTo={menuContainerRef} />
        <InlineImageBubbleMenu editor={editor} appendTo={menuContainerRef} />

        {/* The actual editor content area */}
        <EditorContent editor={editor} />

        {/* Slash command menu */}
        {!hideContextMenu && <ContentMenu editor={editor} />}
      </div>
    </div>
  </MailyProvider>
);
```

## Node Editing Flow

### 1. Selection Detection
Each bubble menu uses TipTap's `shouldShow` callback:

```typescript
// packages/core/src/editor/components/section-menu/section-bubble-menu.tsx:44-71
shouldShow: ({ editor }) => {
  const activeSectionNode = getClosestNodeByName(editor, 'section');
  // ... additional checks for nested nodes

  if (isTextSelected(editor) || !editor.isEditable) {
    return false;
  }
  return editor.isActive('section');
},
```

### 2. State Management
Each bubble menu has a companion hook that reads current node attributes:

```typescript
// Example pattern from useSectionState
function useSectionState(editor) {
  return useMemo(() => ({
    currentAlignment: editor.getAttributes('section').align || 'left',
    currentBorderRadius: editor.getAttributes('section').borderRadius || 0,
    currentBackgroundColor: editor.getAttributes('section').backgroundColor,
    // ... etc
  }), [editor.state]);
}
```

### 3. Attribute Updates
Node attributes are updated via custom commands:

```typescript
// packages/core/src/editor/nodes/section/section.ts:273-291
addCommands() {
  return {
    setSection: () => ({ commands }) => {
      return commands.insertContent({
        type: this.name,
        attrs: { type: this.name },
        content: [{ type: 'paragraph' }],
      });
    },
    updateSection: (attrs) => updateAttributes(this.name, attrs),
  };
},
```

The `updateAttributes` helper (`packages/core/src/editor/utils/update-attribute.ts`) handles updating the focused node's attributes.

## Slash Command Flow

### 1. Trigger Detection
The SlashCommandExtension uses TipTap's Suggestion plugin:

```typescript
// packages/core/src/editor/extensions/slash-command/slash-command.ts:8-28
export const SlashCommandExtension = Extension.create<SlashCommandOptions>({
  name: 'slash-command',
  addOptions() {
    return {
      suggestion: {
        char: '/',  // Trigger character
        command: ({ editor, range, props }) => {
          props.command({ editor, range });
        },
      },
    };
  },
  addProseMirrorPlugins() {
    return [
      Suggestion({
        editor: this.editor,
        ...this.options.suggestion,
      }),
    ];
  },
});
```

### 2. Command Execution
Each block item defines its `command` function:

```typescript
// packages/core/src/blocks/layout.tsx:10-26
export const columns: BlockItem = {
  title: 'Columns',
  description: 'Add columns to email.',
  searchTerms: ['layout', 'columns'],
  icon: <ColumnsIcon className="mly:h-4 mly:w-4" />,
  command: ({ editor, range }) => {
    editor
      .chain()
      .focus()
      .deleteRange(range)  // Remove the '/' trigger text
      .setColumns()        // Insert the node
      .focus(editor.state.selection.head - 2)
      .run();
  },
};
```

## Variable Insertion Flow

Variables use a similar suggestion mechanism, triggered by `@`:

```typescript
// packages/core/src/editor/nodes/variable/variable.ts:117-146
suggestion: {
  char: '@',
  pluginKey: VariablePluginKey,
  command: ({ editor, range, props }) => {
    const nodeAfter = editor.view.state.selection.$to.nodeAfter;
    const overrideSpace = nodeAfter?.text?.startsWith(' ');

    if (overrideSpace) {
      range.to += 1;  // Consume trailing space
    }

    editor
      .chain()
      .focus()
      .insertContentAt(range, [
        { type: this.name, attrs: props },
        { type: 'text', text: ' ' },  // Add space after variable
      ])
      .run();

    window.getSelection()?.collapseToEnd();
  },
},
```

## Output

The editor's output is accessed via:
- `editor.getJSON()` - Returns the JSONContent tree (preferred for storage)
- `editor.getHTML()` - Returns HTML (for preview only, NOT for email sending)

**Critical distinction:** `editor.getHTML()` is for browser preview. For production emails, use `@maily-to/render`.
