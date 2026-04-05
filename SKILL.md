---
name: tiptap-pdf-pipeline
description: Use when user asks about converting AI/LLM responses to TipTap editor format, or exporting TipTap content to PDF. Triggers on phrases like "AI to tipTap", "markdown to tipTap", "tipTap to PDF", "export proposal to PDF", "convert editor content to PDF", "generate PDF from rich text", or when discussing the flow from AI generation → rich text editor → PDF export.
---

# TipTap JSON Pipeline Skill

This skill covers the complete pipeline: **AI Response → TipTap JSON → PDF Export**

## Pipeline Overview

```
AI (Mistral) → Markdown → TipTap JSON → React Component → PDF Export
                  ↑              ↑
             markdown-to-    proposal-pdf.tsx
              tiptap.ts
```

## Part 1: Markdown → TipTap JSON

**File**: `src/lib/markdown-to-tiptap.ts`

### Key Functions

```typescript
import { marked } from "marked";
import { generateJSON } from "@tiptap/html";
import StarterKit from "@tiptap/starter-kit";
import Underline from "@tiptap/extension-underline";
import TextAlign from "@tiptap/extension-text-align";
import Link from "@tiptap/extension-link";
import Image from "@tiptap/extension-image";
import { Table, TableRow, TableHeader, TableCell } from "@tiptap/extension-table";
import Highlight from "@tiptap/extension-highlight";
import Typography from "@tiptap/extension-typography";
import TaskList from "@tiptap/extension-task-list";
import TaskItem from "@tiptap/extension-task-item";

const extensions = [
  StarterKit.configure({ link: false, underline: false }),
  Underline,
  TextAlign.configure({ types: ["heading", "paragraph"] }),
  Link.configure({ openOnClick: false }),
  Image,
  Table.configure({ resizable: false }),
  TableRow,
  TableHeader,
  TableCell,
  Highlight,
  Typography,
  TaskList,
  TaskItem.configure({ nested: true }),
];

export function markdownToTiptapJSON(markdown: string): string {
  if (!markdown) return "";
  
  // Handle HTML entities
  let processed = markdown
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'");
  
  // Strip markdown code fences
  if (processed.startsWith("```markdown")) processed = processed.slice(11);
  else if (processed.startsWith("```")) processed = processed.slice(3);
  if (processed.endsWith("```")) processed = processed.slice(0, -3);
  processed = processed.trim();
  
  // Convert markdown to HTML, then to TipTap JSON
  const html = marked.parse(processed, { async: false }) as string;
  const json = generateJSON(html, extensions);
  return JSON.stringify(json);
}

export function extractTitle(markdown: string): string {
  const match = markdown.match(/^#\s+(.+)$/m);
  return match?.[1]?.trim() ?? "Untitled Proposal";
}
```

### Usage in API Route

```typescript
// src/app/api/proposals/generate/route.ts
import { markdownToTiptapJSON, extractTitle } from "@/lib/markdown-to-tiptap";

const stream = new ReadableStream({
  async start(controller) {
    // Get AI response as streaming markdown
    const mistralStream = await streamProposal(jobDescription, apiKey);
    
    // Collect full markdown
    let fullMarkdown = "";
    const reader = mistralStream.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      fullMarkdown += value;
      // Send chunk to client...
    }
    
    // Convert to TipTap JSON
    const content = markdownToTiptapJSON(fullMarkdown);
    const title = extractTitle(fullMarkdown);
    
    // Save to database
    const proposal = await prisma.proposal.create({
      data: { title, content, shareCode, userId },
    });
  },
});
```

## Part 2: TipTap JSON → PDF Export

**File**: `src/components/pdf/proposal-pdf.tsx`

### Key Implementation Points

1. **Single Text Component with Nested Links** (fixes line breaks):
```typescript
// WRONG - causes line breaks
return (
  <View>
    <Text>Before </Text>
    <Link src="...">Link</Link>
    <Text> After</Text>
  </View>
);

// CORRECT - inline links in single Text
return (
  <Text style={styles.p}>
    Before <Link style={linkStyle}>Link</Link> After
  </Text>
);
```

2. **Text Styling Marks** (bold, italic, underline, strikethrough):
```typescript
const marks = child.marks || [];
const hasBold = marks.some((m: any) => m.type === "bold");
const hasItalic = marks.some((m: any) => m.type === "italic");
const hasUnderline = marks.some((m: any) => m.type === "underline");
const hasStrike = marks.some((m: any) => m.type === "strike" || m.type === "strikeThrough");

const textStyle: any = {};
if (hasBold) textStyle.fontWeight = "bold";
if (hasItalic) textStyle.fontStyle = "italic";
if (hasUnderline) textStyle.textDecoration = "underline";
if (hasStrike) textStyle.textDecoration = "line-through";
```

### Complete PDF Component

```typescript
import { Document, Page, Text, View, StyleSheet, Link, Image } from "@react-pdf/renderer";

const styles = StyleSheet.create({
  page: { padding: 48, fontFamily: "Helvetica", backgroundColor: "#ffffff" },
  title: { fontSize: 22, fontWeight: "bold", color: "#111827", marginBottom: 6 },
  meta: { fontSize: 9, color: "#9CA3AF", marginBottom: 28 },
  h2: { fontSize: 14, fontWeight: "bold", color: "#111827", marginTop: 18, marginBottom: 6 },
  h3: { fontSize: 12, fontWeight: "bold", color: "#374151", marginTop: 12, marginBottom: 4 },
  p: { fontSize: 10, color: "#374151", lineHeight: 1.6, marginBottom: 6 },
  // ... more styles
});

function getTextContent(node: any): string {
  if (!node) return "";
  if (node.text !== undefined) return node.text;
  if (node.content) return node.content.map(getTextContent).join("");
  return "";
}

function renderNode(node: any, key: number): React.ReactElement | null {
  if (!node) return null;

  switch (node.type) {
    case "heading": {
      const level = node.attrs?.level ?? 2;
      const text = getTextContent(node);
      const style = level === 1 ? styles.title : level === 2 ? styles.h2 : styles.h3;
      return <Text key={key} style={style}>{text}</Text>;
    }
    case "paragraph": {
      const content = node.content || [];
      // Render inline text with marks (bold, italic, links, etc.)
      // See full implementation for details
    }
    case "bulletList":
    case "orderedList":
    case "listItem":
    case "taskList":
    case "taskItem":
    case "codeBlock":
    case "blockquote":
    case "horizontalRule":
    case "image":
    case "table":
      // See full implementation
    default:
      return null;
  }
}

interface Props {
  title: string;
  content: string; // TipTap JSON string
  authorName: string;
  createdAt: string;
}

export function ProposalPDF({ title, content, authorName, createdAt }: Props) {
  let doc: { type: string; content: any[] };
  try {
    doc = JSON.parse(content);
  } catch {
    doc = { type: "doc", content: [] };
  }
  
  return (
    <Document>
      <Page size="A4" style={styles.page}>
        <Text style={styles.title}>{title}</Text>
        <Text style={styles.meta}>By {authorName} · Generated {createdAt}</Text>
        {doc.content?.map((node: any, i: number) => renderNode(node, i))}
      </Page>
    </Document>
  );
}
```

## Common Issues & Fixes

| Issue | Solution |
|-------|----------|
| Links cause line breaks in PDF | Use single `<Text>` with nested `<Link>` components |
| Bold/italic not showing | Check for marks: `bold`, `italic`, `underline`, `strike` |
| Tables not rendering | Handle `table`, `tableRow`, `tableHeader`, `tableCell` types |
| Task lists missing | Handle `taskList` and `taskItem` with `checked` attr |
| Images not showing | Use `@react-pdf/renderer` `<Image src={...}>` component |
| Code blocks wrong | Handle `codeBlock` with Courier font |

## Dependencies

```json
{
  "dependencies": {
    "@tiptap/html": "^2.x",
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
    "marked": "^11.x",
    "@react-pdf/renderer": "^3.x"
  }
}
```

## Quick Reference

| Step | File | Function |
|------|------|----------|
| Markdown → HTML | marked | `marked.parse()` |
| HTML → TipTap JSON | @tiptap/html | `generateJSON()` |
| TipTap JSON → PDF | @react-pdf/renderer | `<ProposalPDF />` |