---
name: acontext-agent-integration
description: "Build AI chatbots with Acontext SDK integration. Use when implementing: conversation history storage, disk file tools (read/write/grep), Python sandbox execution, artifact management with disk:: protocol, or context compression. Keywords: Acontext, session management, disk tools, sandbox, Python execution."
---

# Acontext Agent Integration

Integration patterns for building AI agents with Acontext SDK - session management, disk tools, sandbox execution, and artifact storage.

## Architecture

**Core Principle: One Session = One Conversation = One Disk (1:1:1)**

```
┌─────────────────────────────────────────────────────┐
│                  Your Chatbot                        │
├─────────────────────────────────────────────────────┤
│  LLM Client  │  Tool Router  │  Session Manager     │
└──────┬───────┴───────┬───────┴────────┬─────────────┘
       │               │                │
       ▼               ▼                ▼
┌─────────────────────────────────────────────────────┤
│              Acontext SDK (Storage Layer)            │
├─────────────────────────────────────────────────────┤
│  sessions.*        disks.*          sandboxes.*     │
│  - storeMessage    - create         - create        │
│  - getMessages     - artifacts.*    - execCommand   │
│  - getTokenCounts    - upsert       - kill          │
│  - delete            - get                          │
└─────────────────────────────────────────────────────┘
```

**Do NOT use database for messages!** All messages via `sessions.storeMessage()` and `sessions.getMessages()`.

## Quick Start

```ts
import { AcontextClient, DISK_TOOLS, SANDBOX_TOOLS } from "@acontext/acontext";

// 1. Initialize
const client = new AcontextClient({ apiKey: process.env.ACONTEXT_API_KEY });

// 2. Create session + disk (1:1 binding)
const session = await client.sessions.create({ user: "user@example.com" });
const disk = await client.disks.create();

// 3. Store message (MUST include format option!)
await client.sessions.storeMessage(session.id, {
  role: "user",
  content: "Hello!"
}, { format: "openai" });

// 4. Load messages
const messages = await client.sessions.getMessages(session.id, { format: "openai" });
```

## Reference Documentation

| Topic | File | When to Read |
|-------|------|--------------|
| **API Reference** | [api-reference.md](references/api-reference.md) | SDK methods, parameters, return types |
| **Implementation** | [implementation.md](references/implementation.md) | Full code examples, Next.js routes |
| **Session Management** | [session-management.md](references/session-management.md) | Lifecycle, 1:1 binding, list management |
| **Context Compression** | [context-compression.md](references/context-compression.md) | Token monitoring, auto-compression |
| **Disk Path Mapping** | [disk-path-mapping.md](references/disk-path-mapping.md) | `disk::` protocol, URL expiration |
| **Vision API** | [vision-api.md](references/vision-api.md) | Multimodal content, image processing |
| **Streaming** | [streaming.md](references/streaming.md) | SSE server and client implementation |
| **Skill System** | [skill-system.md](references/skill-system.md) | Mount Acontext skills into sandboxes |
| **Session Cleanup** | [session-cleanup.md](references/session-cleanup.md) | Delete sessions, batch cleanup |
| **Error Handling** | [error-handling.md](references/error-handling.md) | API errors, retry strategies |
| **Debugging** | [debugging.md](references/debugging.md) | Logging format, common issues |
| **Troubleshooting** | [troubleshooting.md](references/troubleshooting.md) | 404 errors, tool sequence errors |

## Tool Categories

### Disk Tools (`DISK_TOOLS`)

| Tool | Purpose |
|------|---------|
| `write_file_disk` | Create/overwrite files |
| `read_file_disk` | Read file contents |
| `replace_string_disk` | Find and replace |
| `list_disk` | List directory |
| `grep_disk` / `glob_disk` | Search files |
| `download_file_disk` | Get public URL |

```ts
const ctx = DISK_TOOLS.format_context(client, diskId);
const result = await DISK_TOOLS.execute_tool(ctx, "write_file_disk", {
  filename: "notes.txt",
  content: "Hello",
  file_path: "/"
});
```

### Sandbox Tools (`SANDBOX_TOOLS`)

| Tool | Purpose |
|------|---------|
| `bash_execution_sandbox` | Execute commands |
| `text_editor_sandbox` | Create/edit files |
| `export_file_sandbox` | Export to disk with URL |

```ts
const ctx = SANDBOX_TOOLS.format_context(client, sandboxId, diskId);
await SANDBOX_TOOLS.execute_tool(ctx, "text_editor_sandbox", {
  command: "create",
  path: "script.py",
  file_text: "print('Hello')"
});
```

## Critical Patterns

### Message Sequence for Tool Calls

OpenAI API requires strict ordering: `User → Assistant(tool_calls) → Tool Response`

```ts
// 1. User message
await client.sessions.storeMessage(sessionId, { role: "user", content: "Draw" }, { format: "openai" });

// 2. Assistant with tool_calls (MUST save!)
await client.sessions.storeMessage(sessionId, {
  role: "assistant",
  content: "",
  tool_calls: [{ id: "call_xxx", type: "function", function: {...} }]
}, { format: "openai" });

// 3. Tool response (MUST match tool_call_id!)
await client.sessions.storeMessage(sessionId, {
  role: "tool",
  tool_call_id: "call_xxx",
  content: JSON.stringify({ result: "success" })
}, { format: "openai" });
```

**Common Error:** `400 An assistant message with 'tool_calls' must be followed by tool messages`

### Sandbox: No Heredoc Support

```ts
// WRONG - will hang!
command: "python - << 'EOF'\ncode\nEOF"

// CORRECT - create file first, then execute
await SANDBOX_TOOLS.execute_tool(ctx, "text_editor_sandbox", {
  command: "create", path: "script.py", file_text: "..."
});
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py"
});
```

### Sandbox: Reuse sandbox_id

```ts
// CORRECT: Same sandbox_id across tool calls
let sandboxId: string | undefined;
const ctx1 = SANDBOX_TOOLS.format_context(client, sandboxId, diskId);
await SANDBOX_TOOLS.execute_tool(ctx1, "text_editor_sandbox", {...});
sandboxId = ctx1.sandbox_id;  // Save for reuse!

const ctx2 = SANDBOX_TOOLS.format_context(client, sandboxId, diskId);
await SANDBOX_TOOLS.execute_tool(ctx2, "bash_execution_sandbox", {...});
// Now can find the file created earlier!
```

### disk:: Protocol

Use `disk::` for permanent image references (S3 URLs expire in ~1 hour):

```ts
// Tool returns
{ diskPath: "disk::artifacts/chart.png" }

// Frontend transforms for rendering
![chart](/api/images/proxy?path=artifacts%2Fchart.png&diskId=xxx)
```

See [disk-path-mapping.md](references/disk-path-mapping.md) for implementation.

## Tool Routing Pattern

```ts
const DISK_TOOL_NAMES = ["write_file_disk", "read_file_disk", "list_disk", ...];
const SANDBOX_TOOL_NAMES = ["bash_execution_sandbox", "text_editor_sandbox", ...];

async function executeToolCall(toolCall, context) {
  const { name } = toolCall.function;
  const args = JSON.parse(toolCall.function.arguments || "{}");

  if (DISK_TOOL_NAMES.includes(name)) {
    const ctx = DISK_TOOLS.format_context(context.client, context.diskId);
    return DISK_TOOLS.execute_tool(ctx, name, args);
  }

  if (SANDBOX_TOOL_NAMES.includes(name)) {
    const ctx = SANDBOX_TOOLS.format_context(context.client, context.sandboxId, context.diskId);
    return SANDBOX_TOOLS.execute_tool(ctx, name, args);
  }

  throw new Error(`Unknown tool: ${name}`);
}

// Register with LLM
const tools = [
  ...DISK_TOOLS.to_openai_tool_schema(),
  ...SANDBOX_TOOLS.to_openai_tool_schema(),
];
```

## Integration Checklist

1. **Setup**: Install `@acontext/acontext`, configure `ACONTEXT_API_KEY`
2. **Sessions**: Create on new chat, store every message with `{ format: "openai" }`
3. **Session + Disk**: 1:1 binding, each session gets its own disk
4. **Message Sequence**: Always save tool responses after tool_calls
5. **disk::**: Tools return diskPath, frontend rewrites to API URL
6. **Compression**: Monitor tokens, apply strategies when >80K
7. **Cleanup**: Delete both Session AND Disk when deleting conversation

## Common Mistakes

```ts
// WRONG: Store messages in database
await supabase.from('messages').insert({ content: "Hello" });

// CORRECT: Store messages in Acontext
await client.sessions.storeMessage(sessionId, { role: "user", content: "Hello" }, { format: "openai" });

// WRONG: Multiple sessions sharing one disk
const sharedDisk = await client.disks.create();
// Don't share this disk across sessions!

// CORRECT: Each session has its own disk
const session = await client.sessions.create({ user });
const disk = await client.disks.create(); // Exclusive to this session
```

## Scripts

Copy these files to your project:

| File | Purpose |
|------|---------|
| `scripts/types.ts` | Core type definitions |
| `scripts/config.ts` | Environment configuration |
| `scripts/acontext-client.ts` | SDK wrapper utilities |
| `scripts/disk-tools.ts` | Disk tool schemas and execution |
| `scripts/sandbox-tool.ts` | Python sandbox execution |
| `scripts/openai-client.ts` | OpenAI client with tool routing |
