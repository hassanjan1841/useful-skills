---
name: tiptap-editor-setup
description: Use when user asks about setting up a TipTap rich text editor in Next.js/React. Triggers on phrases like "setup tipTap", "tipTap editor", "rich text editor react", "tiptap toolbar", "tipTap extensions", "tipTap table", "tipTap placeholder", or when building a WYSIWYG editor with formatting options.
---

# TipTap Editor Setup Skill

This skill covers setting up a full-featured TipTap rich text editor in React/Next.js.

## Core Dependencies

```json
{
  "dependencies": {
    "@tiptap/react": "^2.x",
    "@tiptap/starter-kit": "^2.x",
    "@tiptap/extension-underline": "^2.x",
    "@tiptap/extension-text-align": "^2.x",
    "@tiptap/extension-link": "^2.x",
    "@tiptap/extension-image": "^2.x",
    "@tiptap/extension-table": "^2.x",
    "@tiptap/extension-table-row": "^2.x",
    "@tiptap/extension-table-header": "^2.x",
    "@tiptap/extension-table-cell": "^2.x",
    "@tiptap/extension-highlight": "^2.x",
    "@tiptap/extension-typography": "^2.x",
    "@tiptap/extension-task-list": "^2.x",
    "@tiptap/extension-task-item": "^2.x",
    "@tiptap/extension-placeholder": "^2.x",
    "@tiptap/extension-character-count": "^2.x",
    "lucide-react": "^0.x"
  }
}
```

## Editor Component

**File**: `src/components/editor/proposal-editor.tsx`

```typescript
"use client";

import { useEditor, EditorContent } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import Underline from "@tiptap/extension-underline";
import TextAlign from "@tiptap/extension-text-align";
import Link from "@tiptap/extension-link";
import Image from "@tiptap/extension-image";
import { Table } from "@tiptap/extension-table";
import { TableRow } from "@tiptap/extension-table-row";
import { TableHeader } from "@tiptap/extension-table-header";
import { TableCell } from "@tiptap/extension-table-cell";
import Highlight from "@tiptap/extension-highlight";
import Typography from "@tiptap/extension-typography";
import TaskList from "@tiptap/extension-task-list";
import TaskItem from "@tiptap/extension-task-item";
import Placeholder from "@tiptap/extension-placeholder";
import CharacterCount from "@tiptap/extension-character-count";
import { Toolbar } from "./toolbar";
import { Trash2, ArrowDown, ArrowRight, Plus, GripVertical } from "lucide-react";
import { cn } from "@/lib/utils";
import { useState, useEffect } from "react";

interface Props {
  content: string;        // TipTap JSON string
  onChange: (json: string) => void;
}

export function ProposalEditor({ content, onChange }: Props) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({ link: false, underline: false }),
      Underline,
      TextAlign.configure({ types: ["heading", "paragraph"] }),
      Link.configure({ openOnClick: false }),
      Image,
      Table.configure({ resizable: true }),
      TableRow,
      TableHeader,
      TableCell,
      Highlight,
      Typography,
      TaskList,
      TaskItem.configure({ nested: true }),
      Placeholder.configure({ placeholder: "Start writing your proposal..." }),
      CharacterCount,
    ],
    immediatelyRender: false,
    content: (() => {
      if (!content) return "";
      try { return JSON.parse(content); } catch { return ""; }
    })(),
    onUpdate: ({ editor }) => {
      onChange(JSON.stringify(editor.getJSON()));
    },
  });

  const [showTableMenu, setShowTableMenu] = useState(false);
  const [menuPosition, setMenuPosition] = useState({ top: 0, left: 0 });

  useEffect(() => {
    if (!editor) return;
    const updateState = () => {
      const isInTable = editor.isActive("table") || editor.isActive("tableCell") || editor.isActive("tableHeader");
      setShowTableMenu(isInTable);
      if (isInTable) {
        const { view } = editor;
        const coords = view.coordsAtPos(view.state.selection.from);
        const editorRect = view.dom.getBoundingClientRect();
        setMenuPosition({
          top: coords.top - editorRect.top - 45,
          left: coords.left - editorRect.left,
        });
      }
    };
    editor.on("selectionUpdate", updateState);
    editor.on("blur", () => setTimeout(() => setShowTableMenu(false), 150));
    return () => {
      editor.off("selectionUpdate", updateState);
      editor.off("blur", () => {});
    };
  }, [editor]);

  if (!editor) return null;

  return (
    <div className="flex flex-col h-full border border-[#E5E7EB] rounded-[10px] bg-white">
      <div className="flex-shrink-0 sticky top-0 z-10 bg-white border-b border-[#E5E7EB]">
        <Toolbar editor={editor} />
      </div>
      <div className="relative flex-1 overflow-auto">
        {showTableMenu && (
          <div 
            className="absolute z-50 flex items-center gap-1 bg-white border border-[#E5E7EB] shadow-lg rounded-lg p-1"
            style={{ top: menuPosition.top, left: menuPosition.left }}
          >
            {/* Table control buttons - see full file */}
          </div>
        )}
        <EditorContent editor={editor} className="p-4 min-h-[500px] prose prose-sm max-w-none" />
      </div>
    </div>
  );
}
```

## Toolbar Component

**File**: `src/components/editor/toolbar.tsx`

```typescript
"use client";

import { type Editor } from "@tiptap/react";
import {
  Bold, Italic, Underline, Strikethrough, Code, AlignLeft, AlignCenter, AlignRight,
  List, ListOrdered, Link2, Image as ImageIcon, Table, Highlighter, Undo2, Redo2,
  RemoveFormatting, CheckSquare, Minus, Trash2, Plus, ArrowDown, ArrowRight,
} from "lucide-react";
import { cn } from "@/lib/utils";

interface ToolbarProps { editor: Editor }

function ToolBtn({
  onClick, active, title, children, disabled,
}: { onClick: () => void; active?: boolean; title: string; children: React.ReactNode; disabled?: boolean }) {
  return (
    <button
      type="button"
      onMouseDown={(e) => { e.preventDefault(); if (!disabled) onClick(); }}
      title={title}
      disabled={disabled}
      className={cn(
        "w-[30px] h-[30px] flex items-center justify-center rounded-[5px] text-[#374151] transition-colors text-[13px] font-medium",
        active ? "bg-[#EFF6FF] text-[#2563EB]" : "hover:bg-[#F3F4F6]",
        disabled && "opacity-40 cursor-not-allowed"
      )}
    >
      {children}
    </button>
  );
}

const Divider = () => <div className="w-px h-5 bg-[#E5E7EB] mx-1 flex-shrink-0" />;

export function Toolbar({ editor }: ToolbarProps) {
  function insertLink() {
    if (typeof window === "undefined") return;
    const url = window.prompt("Enter URL:");
    if (url) editor.chain().focus().setLink({ href: url }).run();
  }

  function insertImage() {
    if (typeof window === "undefined") return;
    const url = window.prompt("Enter image URL:");
    if (url) editor.chain().focus().setImage({ src: url }).run();
  }

  return (
    <div className="bg-white border-b border-[#E5E7EB] px-4 lg:px-6 py-1.5 flex items-center gap-0.5 flex-wrap">
      <ToolBtn onClick={() => editor.chain().focus().undo().run()} title="Undo"><Undo2 size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().redo().run()} title="Redo"><Redo2 size={14} /></ToolBtn>
      <Divider />

      <select
        value={
          editor.isActive("heading", { level: 1 }) ? "h1"
          : editor.isActive("heading", { level: 2 }) ? "h2"
          : editor.isActive("heading", { level: 3 }) ? "h3"
          : editor.isActive("blockquote") ? "quote"
          : editor.isActive("codeBlock") ? "code"
          : "p"
        }
        onChange={(e) => {
          const v = e.target.value;
          if (v === "p") editor.chain().focus().setParagraph().run();
          else if (v === "h1") editor.chain().focus().setHeading({ level: 1 }).run();
          else if (v === "h2") editor.chain().focus().setHeading({ level: 2 }).run();
          else if (v === "h3") editor.chain().focus().setHeading({ level: 3 }).run();
          else if (v === "quote") editor.chain().focus().setBlockquote().run();
          else if (v === "code") editor.chain().focus().setCodeBlock().run();
        }}
        className="h-[30px] px-2 border border-[#E5E7EB] rounded-[5px] text-[12.5px] text-[#374151] bg-white cursor-pointer"
      >
        <option value="p">Normal text</option>
        <option value="h1">Heading 1</option>
        <option value="h2">Heading 2</option>
        <option value="h3">Heading 3</option>
        <option value="quote">Quote</option>
        <option value="code">Code block</option>
      </select>
      <Divider />

      <ToolBtn onClick={() => editor.chain().focus().toggleBold().run()} active={editor.isActive("bold")} title="Bold"><Bold size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleItalic().run()} active={editor.isActive("italic")} title="Italic"><Italic size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleUnderline().run()} active={editor.isActive("underline")} title="Underline"><Underline size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleStrike().run()} active={editor.isActive("strike")} title="Strikethrough"><Strikethrough size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleCode().run()} active={editor.isActive("code")} title="Inline code"><Code size={14} /></ToolBtn>
      <Divider />

      <ToolBtn onClick={() => editor.chain().focus().setTextAlign("left").run()} active={editor.isActive({ textAlign: "left" })} title="Align left"><AlignLeft size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().setTextAlign("center").run()} active={editor.isActive({ textAlign: "center" })} title="Align center"><AlignCenter size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().setTextAlign("right").run()} active={editor.isActive({ textAlign: "right" })} title="Align right"><AlignRight size={14} /></ToolBtn>
      <Divider />

      <ToolBtn onClick={() => editor.chain().focus().toggleBulletList().run()} active={editor.isActive("bulletList")} title="Bullet list"><List size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleOrderedList().run()} active={editor.isActive("orderedList")} title="Ordered list"><ListOrdered size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleTaskList().run()} active={editor.isActive("taskList")} title="Task list"><CheckSquare size={14} /></ToolBtn>
      <Divider />

      <ToolBtn onClick={() => editor.chain().focus().insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run()} title="Insert table"><Table size={14} /></ToolBtn>
      {/* More table controls... */}
      <Divider />
      
      <ToolBtn onClick={insertLink} active={editor.isActive("link")} title="Insert link"><Link2 size={14} /></ToolBtn>
      <ToolBtn onClick={insertImage} title="Insert image"><ImageIcon size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().toggleHighlight().run()} active={editor.isActive("highlight")} title="Highlight"><Highlighter size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().setHorizontalRule().run()} title="Horizontal rule"><Minus size={14} /></ToolBtn>
      <ToolBtn onClick={() => editor.chain().focus().unsetAllMarks().clearNodes().run()} title="Clear formatting"><RemoveFormatting size={14} /></ToolBtn>
    </div>
  );
}
```

## Key Configuration Points

| Feature | Extension | Config |
|---------|-----------|--------|
| Basic editing | StarterKit | `configure({ link: false, underline: false })` |
| Underline | extension-underline | - |
| Text alignment | extension-text-align | `configure({ types: ["heading", "paragraph"] })` |
| Links | extension-link | `configure({ openOnClick: false })` |
| Images | extension-image | - |
| Tables | extension-table | `configure({ resizable: true })` |
| Highlight | extension-highlight | - |
| Task lists | extension-task-list, task-item | `TaskItem.configure({ nested: true })` |
| Placeholder | extension-placeholder | `configure({ placeholder: "..." })` |
| Character count | extension-character-count | - |
| Typography | extension-typography | - |

## Important Notes

1. **`immediatelyRender: false`** - Prevents hydration mismatch in Next.js
2. **Content parsing** - Parse JSON string on load, handle parse errors gracefully
3. **onChange** - Use `editor.getJSON()` and stringify for storing
4. **Toolbar** - Use `onMouseDown={(e) => e.preventDefault()}` to prevent editor blur
5. **Link/Image prompts** - Use `window.prompt()` for simple URL input (can be replaced with modal)

## Quick Reference

| Task | Command |
|------|---------|
| Toggle bold | `editor.chain().focus().toggleBold().run()` |
| Toggle italic | `editor.chain().focus().toggleItalic().run()` |
| Set heading | `editor.chain().focus().setHeading({ level: 1 }).run()` |
| Insert link | `editor.chain().focus().setLink({ href: url }).run()` |
| Insert image | `editor.chain().focus().setImage({ src: url }).run()` |
| Insert table | `editor.chain().focus().insertTable({ rows: 3, cols: 3 }).run()` |
| Check active | `editor.isActive("bold")` |