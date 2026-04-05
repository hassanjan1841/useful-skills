---
name: serverless-file-parser
description: Use when user has file upload/parsing issues in Next.js serverless (Vercel/Netlify) - specifically PDF, DOCX, or TXT files failing to parse in production despite working locally. Triggers on phrases like "file upload not working", "PDF parsing fails", "can't parse DOCX in production", "serverless PDF extraction", "next.js file parser error", or when user mentions "works locally but not in production" for file parsing.
---

# Serverless File Parser Skill

## Problem Identification

When PDF/DOCX/TXT file parsing works locally but fails in production (Vercel/serverless), the root cause is **native module dependencies** that can't compile in serverless environments.

### Common Symptoms
- `Error: Cannot find module 'canvas'`
- `Error: pdfParse is not a function`
- `TypeError: Class constructors cannot be invoked without 'new'`
- Worker crashes or silent failures in production
- Build succeeds but runtime errors

### Libraries That Fail in Serverless
- `pdf-parse` (any version) - depends on @napi-rs/canvas
- `react-pdftotext` - depends on pdf-parse
- Any library requiring native compilation

## Solution: Serverless-Compatible Libraries

### PDF Parsing → unpdf
```
npm install unpdf
```

**Why unpdf works:**
- Zero native dependencies
- Built specifically for serverless (Vercel, Netlify, Cloudflare Workers)
- Pure JavaScript PDF.js implementation
- Works in Node.js, browser, and edge runtimes

**Implementation:**
```typescript
const { getDocumentProxy, extractText } = await import("unpdf");
const pdf = await getDocumentProxy(new Uint8Array(buffer));
const { text } = await extractText(pdf, { mergePages: true });
pdf.destroy();
return text.trim();
```

### DOCX Parsing → mammoth (already serverless-safe)
```
npm install mammoth
```

**Implementation:**
```typescript
const mammoth = await import("mammoth");
const result = await mammoth.extractRawText({ buffer });
return result.value.trim();
```

### TXT Parsing → Native (no dependencies)
```typescript
return buffer.toString("utf-8").trim();
```

## Configuration

### next.config.mjs
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverComponentsExternalPackages: ["unpdf", "@napi-rs/canvas"],
  },
  webpack: (config, { isServer }) => {
    if (isServer) {
      config.externals = [...(config.externals || []), "canvas", "@napi-rs/canvas"];
    }
    return config;
  },
};

export default nextConfig;
```

> Note: unpdf works without config in most cases, but the above helps with edge cases.

## File: src/lib/file-parser.ts

Complete implementation:

```typescript
import type { Buffer } from "buffer";

export async function parseFile(buffer: Buffer, filename: string): Promise<string> {
  const ext = filename.split(".").pop()?.toLowerCase();

  if (ext === "pdf") {
    try {
      const { getDocumentProxy, extractText } = await import("unpdf");
      const pdf = await getDocumentProxy(new Uint8Array(buffer as unknown as ArrayBuffer));
      const { text } = await extractText(pdf, { mergePages: true });
      pdf.destroy();
      return text.trim();
    } catch (err) {
      console.error("[parseFile] PDF ERROR:", err);
      throw new Error("Failed to parse PDF");
    }
  }

  if (ext === "docx") {
    const mammoth = await import("mammoth");
    const result = await mammoth.extractRawText({ buffer });
    return result.value.trim();
  }

  if (ext === "txt") {
    return buffer.toString("utf-8").trim();
  }

  throw new Error(`Unsupported file type: .${ext}`);
}

export function validateFile(filename: string, sizeBytes: number): string | null {
  const ext = filename.split(".").pop()?.toLowerCase();
  if (!["pdf", "docx", "txt"].includes(ext ?? "")) {
    return "Only PDF, DOCX, and TXT files are supported";
  }
  if (sizeBytes > 10 * 1024 * 1024) {
    return "File size must be under 10MB";
  }
  return null;
}
```

## Testing

### Local Test
```bash
npx tsx -e "
import { parseFile } from './src/lib/file-parser';
import { readFileSync } from 'fs';

async function test() {
  const pdfText = await parseFile(readFileSync('./test-files/sample.pdf') as any, 'sample.pdf');
  console.log('PDF OK:', pdfText.length);
  
  const docxText = await parseFile(readFileSync('./test-files/sample.docx') as any, 'sample.docx');
  console.log('DOCX OK:', docxText.length);
  
  const txtText = await parseFile(readFileSync('./test-files/sample.txt') as any, 'sample.txt');
  console.log('TXT OK:', txtText.length);
}

test();
"
```

### Production Test
Use Playwright or similar to test the full upload flow in the browser, as local Node.js tests may pass even if serverless still has issues.

## Quick Reference

| File Type | Library | Status |
|-----------|---------|--------|
| PDF | unpdf | ✅ Serverless-safe |
| DOCX | mammoth | ✅ Serverless-safe |
| TXT | Native | ✅ Serverless-safe |
| XLSX | xlsx | ⚠️ Needs testing |
| Images | sharp | ⚠️ Needs external worker |