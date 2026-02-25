---
name: acontext-chatbot-integration
description: "Build AI chatbots with Acontext integration for persistent storage, file operations, and Python sandbox execution. Use when: (1) Building chatbots with conversation history storage, (2) Implementing file read/write/list/grep tools, (3) Adding Python code execution for data analysis, (4) Creating figure generation pipelines, (5) Managing artifacts with disk:: protocol, (6) Implementing token-aware context compression."
---

# Acontext Chatbot Integration

Integration patterns for building AI chatbots with Acontext SDK - provides session management, disk tools, sandbox execution, and artifact storage.

## Architecture

**Core Principle: One Acontext Session = One Conversation, all messages stored in Acontext, no database needed for messages.**

```
┌─────────────────────────────────────────────────────┐
│                  Your Chatbot                        │
├─────────────────────────────────────────────────────┤
│  LLM Client  │  Tool Router  │  Session Manager     │
└──────┬───────┴───────┬───────┴────────┬─────────────┘
       │               │                │
       ▼               ▼                ▼
┌─────────────────────────────────────────────────────┤
│              Acontext SDK (Only Storage Layer)       │
├─────────────────────────────────────────────────────┤
│  sessions.*        disks.*          sandboxes.*     │
│  - storeMessage    - create         - create        │
│  - getMessages     - artifacts.*    - execCommand   │
│  - getTokenCounts    - upsert       - kill          │
│  - delete            - get                          │
│                      - delete                       │
└─────────────────────────────────────────────────────┘
```

**Storage Architecture:**

| Layer | Content | Required? |
|-------|---------|-----------|
| **Acontext Sessions** | Conversation messages, context, token counts | Required |
| **Acontext Disk** | Generated files, images, artifacts | Required |
| **Database (Supabase)** | User auth, session ID mapping | Optional |

**Do NOT use database for messages!** All messages managed via `sessions.storeMessage()` and `sessions.getMessages()`.

## Session + Disk Binding (1:1:1)

**One Session = One Conversation = One Disk (1:1:1 binding)**

- Each Session gets its own independent Disk
- Files are completely isolated between conversations
- Never share Disk between Sessions

```ts
// CORRECT: Each session has its own disk
const session = await acontext.sessions.create({ user });
const disk = await acontext.disks.create(); // Exclusive to this session

// WRONG: Multiple sessions sharing one disk
const sharedDisk = await acontext.disks.create();
// Don't share this disk across sessions!
```

See [references/session-management.md](references/session-management.md) for detailed lifecycle management.

## Quick Start

### 1. Initialize Clients

```ts
import { AcontextClient } from "@acontext/acontext";

// baseUrl is optional - only needed for self-hosted
const client = new AcontextClient({
  apiKey: process.env.ACONTEXT_API_KEY,
});

await client.ping();  // Test connection
```

### 2. Create Session + Disk

```ts
const session = await client.sessions.create({ user: "user@example.com" });
const disk = await client.disks.create();

console.log(`Session: ${session.id}, Disk: ${disk.id}`);
```

### 3. Store and Load Messages

```ts
// IMPORTANT: Always pass format: 'openai' option
await client.sessions.storeMessage(session.id, {
  role: "user",
  content: "Hello!"
}, { format: "openai" });

const messages = await client.sessions.getMessages(session.id, {
  format: "openai"
});
console.log(messages.items);
```

## Core Features

### Message Types

When using tools with LLMs, you must store **all message types** in the correct order:

| Message Type | Role | When to Store |
|-------------|------|---------------|
| User message | `user` | After user sends input |
| Assistant with tools | `assistant` | After LLM responds with `tool_calls` |
| Tool response | `tool` | After executing each tool call |

### Session Management

```ts
// Store message - MUST include format option!
await client.sessions.storeMessage(session.id, {
  role: "user" | "assistant" | "system" | "tool",
  content: string,              // Also supports ContentPart[] for images
  tool_calls?: ToolCall[],      // For assistant with tool calls
  tool_call_id?: string         // For tool response (MUST match tool_call.id)
}, { format: "openai" });       // REQUIRED!

// Load messages
const result = await client.sessions.getMessages(session.id, {
  format: "openai"              // Recommended for OpenAI compatibility
});

// Get token counts
const tokens = await client.sessions.getTokenCounts(session.id);
```

### Complete Tool Call Flow

```ts
// 1. Store user message
await client.sessions.storeMessage(sessionId, {
  role: "user",
  content: "Help me edit this document"
}, { format: "openai" });

// 2. Call LLM, get response with tool_calls
const llmResponse = await openai.chat.completions.create({...});

// 3. Store assistant message with tool_calls
const toolCalls = llmResponse.choices[0].message.tool_calls;
await client.sessions.storeMessage(sessionId, {
  role: "assistant",
  content: llmResponse.choices[0].message.content || "",
  tool_calls: toolCalls
}, { format: "openai" });

// 4. Execute each tool and store response
for (const tc of toolCalls) {
  const result = await executeTool(tc.function.name, JSON.parse(tc.function.arguments));

  // MUST store tool response after each execution
  await client.sessions.storeMessage(sessionId, {
    role: "tool",
    tool_call_id: tc.id,  // MUST match the tool_call id
    content: JSON.stringify(result)
  }, { format: "openai" });
}
```

### Loading History with Tool Calls

```ts
// Load messages - tool responses are included by default
const result = await client.sessions.getMessages(sessionId, {
  format: "openai",
  limit: 50
});

// Messages will include:
// - user messages
// - assistant messages with tool_calls
// - tool response messages (role: "tool")

// For display, you may want to filter out tool responses:
const displayMessages = result.items.filter(m => m.role !== "tool");
```

### Disk Tools

| Tool | Description |
|------|-------------|
| `write_file_disk` | Create or overwrite a file |
| `read_file_disk` | Read file contents |
| `replace_string_disk` | Find and replace in a file |
| `list_disk` | List files in a directory |
| `download_file_disk` | Get file with public URL |
| `grep_disk` | Search file contents |
| `glob_disk` | Match file paths by pattern |

```ts
import { DISK_TOOLS } from "@acontext/acontext";

const ctx = DISK_TOOLS.format_context(acontext, diskId);
const toolSchemas = DISK_TOOLS.to_openai_tool_schema();

const result = await DISK_TOOLS.execute_tool(ctx, "write_file_disk", {
  filename: "notes.txt",
  content: "Hello, World!",
  file_path: "/"
});
```

### Sandbox Tools

| Tool | Description |
|------|-------------|
| `bash_execution_sandbox` | Execute bash commands |
| `text_editor_sandbox` | View, create, edit text files |
| `export_file_sandbox` | Export files to disk with download URL |

**CRITICAL: Sandbox does NOT support multi-line input or heredoc syntax!**

The sandbox environment cannot handle heredoc syntax like `python - << 'EOF'`. You MUST follow this pattern:

```ts
// CORRECT: Create file first, then execute
// Step 1: Create Python file
await SANDBOX_TOOLS.execute_tool(ctx, "text_editor_sandbox", {
  command: "create",
  path: "script.py",
  file_text: "print('Hello, World!')"
});

// Step 2: Execute the file
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py",
  timeout: 30
});

// WRONG: Heredoc syntax will hang!
// DO NOT USE: command: "python - << 'EOF'\ncode\nEOF"
```

```ts
import { SANDBOX_TOOLS } from "@acontext/acontext";

const sandbox = await acontext.sandboxes.create();
const ctx = SANDBOX_TOOLS.format_context(acontext, sandbox.sandbox_id, disk.id);

const result = await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py",
  timeout: 30
});

await acontext.sandboxes.kill(sandbox.sandbox_id); // Always cleanup
```

### Artifact Storage

```ts
import { FileUpload } from "@acontext/acontext";

// Upload
const artifact = await client.disks.artifacts.upsert(disk.id, {
  file: new FileUpload("notes.md", Buffer.from("# Notes")),
  file_path: "/documents/",
  meta: { author: "alice" }
});

// Get with public URL
const result = await client.disks.artifacts.get(disk.id, {
  file_path: "/documents/",
  filename: "notes.md",
  with_public_url: true,
  with_content: true
});
```

## disk:: Protocol

Use `disk::` for permanent image references instead of expiring S3 presigned URLs (~1 hour expiry).

**Why disk::?**
- S3 presigned URLs expire after ~1 hour
- `disk::` paths are permanent references that get fresh URLs on each render
- Store `disk::` in DB, convert to proxy URL for rendering

**Tool Output:**
```ts
{ diskPath: "disk::artifacts/chart.png" }
```

**Frontend Transform:**
```ts
// Transform disk:: to proxy URL for rendering
function transformDiskUrlsToProxy(markdown: string, documentId: string): string {
  return markdown.replace(
    /!\[([^\]]*)\]\(disk::([^)]+)\)/g,
    (match, alt, path) => {
      const proxyUrl = `/api/images/proxy?path=${encodeURIComponent(path)}&documentId=${documentId}`;
      return `![${alt}](${proxyUrl})`;
    }
  );
}

// Transform proxy URL back to disk:: for storage
function transformProxyUrlsToDisk(markdown: string, documentId: string): string {
  // ... reverse transformation
}
```

**Important Pitfalls:**
1. SDK may not return `disk_path` - build default path `artifacts/{filename}`
2. AI may use wrong parameter name (`filename` vs `sandbox_filename`) - handle both
3. Transform BEFORE markdown parsing, not after

See **[Disk Path Mapping](references/disk-path-mapping.md)** for complete implementation guide.

## Critical: Message Sequence for Tool Calls

OpenAI API requires strict message ordering:

```
User → Assistant (tool_calls) → Tool Response → Assistant Response
```

```ts
// 1. User message
await client.sessions.storeMessage(sessionId, { role: "user", content: "Draw" });

// 2. Assistant with tool_calls (MUST save!)
await client.sessions.storeMessage(sessionId, {
  role: "assistant",
  content: "",
  tool_calls: [{ id: "call_xxx", type: "function", function: {...} }]
});

// 3. Tool response (MUST follow tool_call_id!)
await client.sessions.storeMessage(sessionId, {
  role: "tool",
  tool_call_id: "call_xxx",
  content: JSON.stringify({ result: "success" })
});
```

**Common Error:**
```
400 An assistant message with 'tool_calls' must be followed by tool messages
```
Cause: Assistant with `tool_calls` saved but tool response not saved.

## Tool Routing Pattern

```ts
import { DISK_TOOLS, SANDBOX_TOOLS } from "@acontext/acontext";

async function executeToolCall(toolCall, context) {
  const { name } = toolCall.function;
  const args = JSON.parse(toolCall.function.arguments || "{}");

  if (isDiskToolName(name)) {
    const ctx = DISK_TOOLS.format_context(context.acontextClient, context.diskId);
    return DISK_TOOLS.execute_tool(ctx, name, args);
  }

  if (isSandboxToolName(name)) {
    const ctx = SANDBOX_TOOLS.format_context(
      context.acontextClient, context.sandboxId, context.diskId
    );
    return SANDBOX_TOOLS.execute_tool(ctx, name, args);
  }

  throw new Error(`Unknown tool: ${name}`);
}

// Register tools
const tools = [
  ...DISK_TOOLS.to_openai_tool_schema(),
  ...SANDBOX_TOOLS.to_openai_tool_schema(),
];
```

## Integration Checklist

1. **Setup**: Install `@acontext/acontext`, configure `ACONTEXT_API_KEY`
2. **Sessions**: Create on new chat, store every message in Acontext, load on resume
3. **Session + Disk**: 1:1 binding, each session gets its own disk
4. **Message Sequence**: Always save tool responses after tool_calls
5. **disk::**: Tools return diskPath, frontend rewrites to API URL
6. **Compression**: Monitor tokens, apply strategies when >80K
7. **Cleanup**: Delete both Session AND Disk when deleting conversation

## Acontext Skill System

Acontext provides a built-in Skill system that can be mounted into sandboxes for code execution.

```ts
import { SANDBOX_TOOLS } from "@acontext/acontext";

// Create sandbox and disk
const sandbox = await acontext.sandboxes.create();
const disk = await acontext.disks.create();

// Create context with skill mounting
const ctx = SANDBOX_TOOLS.format_context(
  acontext,
  sandbox.sandbox_id,
  disk.id,
  { mount_skills: ["skill-uuid-1", "skill-uuid-2"] }
);

// Or mount skills after creation
ctx.mount_skills(["skill-uuid-3"]);

// Skills are available at /skills/{skill_name}/ in sandbox
// Example: /skills/my-skill/scripts/process.py
```

**Use Cases:**
- Provide domain-specific scripts to LLM
- Share reusable code across sessions
- Give agents access to specialized tools

**Note:** Always call `get_context_prompt()` after mounting to include skill info in system message.

## Advanced Topics

See reference files for detailed implementations:

- **[Session Management](references/session-management.md)** - Lifecycle, list management, file isolation
- **[Context Compression](references/context-compression.md)** - Token monitoring, auto/manual compression
- **[Error Handling](references/error-handling.md)** - API errors, tool errors, retry strategies
- **[Vision API](references/vision-api.md)** - Multimodal content, image processing
- **[Streaming](references/streaming.md)** - SSE server and client implementation
- **[Session Cleanup](references/session-cleanup.md)** - Delete sessions, batch cleanup
- **[Debugging](references/debugging.md)** - Logging format, common issues
- **[Troubleshooting](references/troubleshooting.md)** - 404 errors, tool sequence errors
- **[Disk Path Mapping](references/disk-path-mapping.md)** - disk:: protocol, URL expiration, frontend transformation
- **[Skill System](references/skill-system.md)** - Mount Acontext skills into sandboxes
- **[API Reference](references/api-reference.md)** - Complete SDK method reference
- **[Implementation](references/implementation.md)** - Full code examples

## Scripts

Copy these files to your project:

- `scripts/types.ts` - Core type definitions
- `scripts/config.ts` - Environment configuration
- `scripts/acontext-client.ts` - SDK wrapper utilities
- `scripts/disk-tools.ts` - Disk tool schemas and execution
- `scripts/sandbox-tool.ts` - Python sandbox execution
- `scripts/openai-client.ts` - OpenAI client with tool routing

## Common Mistakes

**Wrong Architecture:**
```
Database → Store message history  // WRONG!
Acontext → Only for tools         // WRONG!
```

**Correct Architecture:**
```
Acontext Sessions → Store all messages, context management
Acontext Disk     → Store generated files
Database (opt)    → Only for user auth, session ID mapping
```

```ts
// CORRECT: Store messages in Acontext
await client.sessions.storeMessage(sessionId, { role: "user", content: "Hello" });

// WRONG: Store messages in database
await supabase.from('messages').insert({ content: "Hello" });
```
