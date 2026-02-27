# disk:: Path Mapping Guide

## Overview

The `disk::` protocol provides permanent file references that don't expire, unlike presigned S3 URLs which expire after ~1 hour.

## Problem

When using `export_file_sandbox`, the SDK returns:
- `public_url`: A presigned S3 URL (expires in ~1 hour)
- `disk_path`: **Often empty/undefined!**

If you use `public_url` directly in your document, the image will break after 1 hour.

## Solution: disk:: Protocol

Store `disk::` paths in your content, then convert to fresh URLs when rendering.

```
Storage:  ![chart](disk::artifacts/chart.png)
                     ↓
Rendering: ![chart](/api/images/proxy?path=artifacts%2Fchart.png&documentId=xxx)
                     ↓
API calls Acontext to get fresh URL → Image displays
```

## Implementation

### 1. Backend: export_file_sandbox Handler

**CRITICAL**: SDK may not return `disk_path`, so build a default path:

```ts
// lib/acontext/sandbox-tools.ts
if (toolName === 'export_file_sandbox' && !result.error) {
  // SDK may not return disk_path - build default path
  const filename = (args.sandbox_filename || args.filename) as string | undefined;

  // Use returned disk_path, or fallback to artifacts/{filename}
  const diskPath = (result.disk_path as string) ||
    (filename ? `artifacts/${filename}` : '');

  return {
    success: true,
    diskPath: `disk::${diskPath}`,
    message: `File exported successfully. Use this path in markdown: disk::${diskPath}`,
    _meta: {
      originalDiskPath: diskPath,
      publicUrl: result.public_url, // For immediate preview only
    },
  };
}
```

### 2. Frontend: Markdown Editor Transformation

Transform `disk::` to proxy URLs for rendering, and back for storage:

```tsx
// components/editor/MDXEditorCore.tsx

/**
 * Transform disk:: URLs to proxy URLs for rendering
 */
function transformDiskUrlsToProxy(markdown: string, documentId: string): string {
  return markdown.replace(
    /!\[([^\]]*)\]\(disk::([^)]+)\)/g,
    (match, alt, path) => {
      const proxyUrl = `/api/images/proxy?path=${encodeURIComponent(path)}&documentId=${encodeURIComponent(documentId)}`;
      return `![${alt}](${proxyUrl})`;
    }
  );
}

/**
 * Transform proxy URLs back to disk:: URLs for storage
 */
function transformProxyUrlsToDisk(markdown: string, documentId: string): string {
  const escapedDocId = encodeURIComponent(documentId).replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const proxyPattern = `/api/images/proxy\\?path=([^&\\\\]+)(?:\\\\)?&documentId=${escapedDocId}`;

  return markdown.replace(
    new RegExp(`!\\[([^\\]]*)\\]\\(${proxyPattern}\\)`, 'g'),
    (match, alt, encodedPath) => {
      const path = decodeURIComponent(encodedPath);
      return `![${alt}](disk::${path})`;
    }
  );
}

// Usage in component:
const renderedContent = useMemo(() => {
  return transformDiskUrlsToProxy(content, documentId);
}, [content, documentId]);

const handleChange = (markdown: string) => {
  const originalMarkdown = transformProxyUrlsToDisk(markdown, documentId);
  onChange(originalMarkdown);
};
```

### 3. Frontend: Chat Panel Transformation

Chat messages also need to render `disk::` paths as images:

```tsx
// components/chat/ChatPanel.tsx

/**
 * Transform disk:: URLs to proxy URLs for rendering in chat
 */
function transformDiskUrlsInContent(content: string, documentId: string): string {
  return content.replace(
    /!\[([^\]]*)\]\(disk::([^)]+)\)/g,
    (match, alt, path) => {
      const proxyUrl = `/api/images/proxy?path=${encodeURIComponent(path)}&documentId=${encodeURIComponent(documentId)}`;
      return `![${alt}](${proxyUrl})`;
    }
  );
}

function ChatMessageContent({ content, documentId }: Props) {
  const [html, setHtml] = useState('');

  useEffect(() => {
    (async () => {
      // Transform BEFORE parsing markdown
      const transformedContent = documentId
        ? transformDiskUrlsInContent(content, documentId)
        : content;
      const out = await marked.parse(transformedContent);
      setHtml(sanitizeHtml(out));
    })();
  }, [content, documentId]);

  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

### 4. API: Image Proxy Endpoint

```ts
// app/api/images/proxy/route.ts

export async function GET(request: NextRequest) {
  const path = searchParams.get('path');           // e.g., "artifacts/chart.png"
  const documentId = searchParams.get('documentId');

  // Get disk ID from session
  const session = await getOrCreateChatSession({ userId, documentId, acontextClient });
  const diskId = session.acontextDiskId;

  // Parse path
  const { filePath, filename } = parseFilePath(path);
  // artifacts/chart.png → filePath: "/artifacts/", filename: "chart.png"

  // Get fresh public URL from Acontext
  const result = await client.disks.artifacts.get(diskId, {
    filePath,
    filename,
    withPublicUrl: true,
    withContent: false,
  });

  // Redirect to fresh URL (or return image data)
  return NextResponse.redirect(result.public_url);
}
```

## Pitfalls & Solutions

### Pitfall 1: SDK Doesn't Return disk_path

```ts
// Test result shows:
const result = await SANDBOX_TOOLS.executeTool(ctx, 'export_file_sandbox', {
  sandbox_path: '/workspace/',
  sandbox_filename: 'chart.png',
});
// Returns: { message: "...", public_url: "..." }
// disk_path is UNDEFINED!
```

**Solution**: Build default path from filename:
```ts
const diskPath = result.disk_path || `artifacts/${args.sandbox_filename}`;
```

### Pitfall 2: AI Uses Wrong Parameter Name

AI might call the tool with `filename` instead of `sandbox_filename`:

```json
// AI's tool call (wrong parameter name)
{ "sandbox_path": "/workspace/", "filename": "chart.png" }

// Correct parameter name
{ "sandbox_path": "/workspace/", "sandbox_filename": "chart.png" }
```

**Solution**: Handle both parameter names:
```ts
const filename = args.sandbox_filename || args.filename;
```

### Pitfall 3: Chat Panel Shows Raw Markdown

If chat panel shows `![chart](disk::artifacts/chart.png)` as text instead of image:

**Cause**: Markdown not transformed before parsing

**Solution**: Transform `disk::` to proxy URL BEFORE calling `marked.parse()`:
```ts
const transformed = transformDiskUrlsInContent(content, documentId);
const html = await marked.parse(transformed);
```

### Pitfall 4: File Not Found at artifacts/ Path

If `/api/images/proxy` returns 404, verify:

1. File was actually saved to `artifacts/`:
```ts
const artifacts = await client.disks.artifacts.list(diskId);
console.log(artifacts); // Should show "artifacts" directory
```

2. Path parsing is correct:
```ts
// Input: "artifacts/chart.png"
// Expected: filePath="/artifacts/", filename="chart.png"
```

## Testing

```bash
# Test export_file_sandbox path handling
node -e "
import { AcontextClient, SANDBOX_TOOLS } from '@acontext/acontext';

const client = new AcontextClient({ apiKey: process.env.ACONTEXT_API_KEY });
const disk = await client.disks.create();
const sandbox = await client.sandboxes.create();

// Create and export file
const ctx1 = await SANDBOX_TOOLS.formatContext(client, sandbox.sandbox_id, disk.id);
await SANDBOX_TOOLS.executeTool(ctx1, 'text_editor_sandbox', {
  command: 'create', path: '/workspace/test.txt', file_text: 'Hello'
});

const ctx2 = await SANDBOX_TOOLS.formatContext(client, sandbox.sandbox_id, disk.id);
const result = await SANDBOX_TOOLS.executeTool(ctx2, 'export_file_sandbox', {
  sandbox_path: '/workspace/', sandbox_filename: 'test.txt'
});

console.log('SDK returned disk_path:', JSON.parse(result).disk_path);
// Expected: undefined (SDK doesn't return it!)

// Verify file is at artifacts/
const artifacts = await client.disks.artifacts.list(disk.id);
console.log('Disk contents:', artifacts);

await client.sandboxes.kill(sandbox.sandbox_id);
"
```

## Summary

| Layer | Format | Purpose |
|-------|--------|---------|
| **LLM Response** | `disk::artifacts/chart.png` | Permanent reference, never expires |
| **Storage (DB)** | `disk::artifacts/chart.png` | Store permanent path |
| **Rendering** | `/api/images/proxy?path=artifacts%2Fchart.png&documentId=xxx` | Get fresh URL on demand |
| **Browser** | `<img src="https://s3.../chart.png?X-Amz-Signature=...">` | Actual image load |

This ensures images remain visible indefinitely, with URLs refreshed on every page load.
