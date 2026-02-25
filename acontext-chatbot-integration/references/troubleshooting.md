# Troubleshooting

## 400 Parameter Error

```
APIError: parameter error
at SessionsAPI.storeMessage
```

### Cause: Missing format option

```ts
// WRONG - will throw "parameter error"
await client.sessions.storeMessage(sessionId, {
  role: "user",
  content: "Hello"
});

// CORRECT - must include format option
await client.sessions.storeMessage(sessionId, {
  role: "user",
  content: "Hello"
}, { format: "openai" });
```

### Cause: Empty content

```ts
// WRONG - empty content may cause errors
await client.sessions.storeMessage(sessionId, {
  role: "assistant",
  content: ""
}, { format: "openai" });

// CORRECT - ensure non-empty content
await client.sessions.storeMessage(sessionId, {
  role: "assistant",
  content: content || " "
}, { format: "openai" });
```

## 404 Not Found Error

```
[Acontext] Response: { status: 404, statusText: 'Not Found' }
```

### Cause 1: Wrong Import

```ts
// CORRECT
import { AcontextClient } from "@acontext/acontext";
const client = new AcontextClient({ apiKey: process.env.ACONTEXT_API_KEY });

// WRONG
import { Acontext } from "@acontext/acontext";
```

### Cause 2: Wrong Request Body Format

```ts
// CORRECT (official SDK)
const session = await client.sessions.create({
  user: "user@example.com"
});

// WRONG (non-standard)
await client.sessions.create({
  configs: { userId: "user-123" }
});
```

### Cause 3: Missing baseUrl for Self-Hosted

```ts
// For self-hosted Acontext
const client = new AcontextClient({
  baseUrl: "http://localhost:8029/api/v1",
  apiKey: "sk-ac-your-root-api-bearer-token",
});
```

### Cause 4: Wrong API Key Format

```
CORRECT: sk-ac-xxx
WRONG: acx-xxx
```

## Tool Call Sequence Error

```
400 An assistant message with 'tool_calls' must be followed by tool messages
responding to each 'tool_call_id'. The following tool_call_ids did not have
response messages: call_xxx
```

### Diagnosis

1. Check if assistant messages with `tool_calls` are saved
2. Check if corresponding `tool` response messages are saved
3. Verify `tool_call_id` matches between assistant and tool messages

### Debug Logging

```ts
const messages = await client.sessions.getMessages(sessionId);
for (const msg of messages.items) {
  if (msg.role === "assistant" && msg.tool_calls) {
    console.log("Assistant tool_calls:", msg.tool_calls.map(tc => tc.id));
  }
  if (msg.role === "tool") {
    console.log("Tool response for:", msg.tool_call_id);
  }
}
```

### Fix

Always save tool response immediately after execution:

```ts
// Execute tool
const result = await executeToolCall(toolCall, context);

// Save tool response BEFORE next LLM call
await client.sessions.storeMessage(sessionId, {
  role: "tool",
  tool_call_id: toolCall.id,
  content: JSON.stringify(result)
});
```

### Cause: History Loading Drops tool_calls

**Symptom:** Error occurs when loading history, not during current request.

```
500 Invalid parameter: messages with role 'tool' must be a response to a
preceeding message with 'tool_calls'.
```

**Root Cause:** When loading history from Acontext and building messages array for OpenAI, you must preserve `tool_calls` and `tool_call_id` fields.

```ts
// WRONG - drops tool_calls and tool_call_id!
...history.map((m) => ({
  role: m.role as 'user' | 'assistant',
  content: m.content,
})),

// CORRECT - preserve all fields
...history.map((m) => {
  const msg: ChatCompletionMessageParam = {
    role: m.role,
    content: m.content || '',
  };
  if (m.role === 'assistant' && m.tool_calls) {
    (msg as ChatCompletionAssistantMessageParam).tool_calls = m.tool_calls;
  }
  if (m.role === 'tool' && m.tool_call_id) {
    (msg as ChatCompletionToolMessageParam).tool_call_id = m.tool_call_id;
  }
  return msg;
}),
```

**Why this is tricky:** The error happens on the NEXT request after a successful tool execution, because:
1. First request: tool calls executed successfully, messages saved correctly
2. Second request: history loaded, but `tool_calls`/`tool_call_id` lost during mapping
3. OpenAI sees orphaned `tool` messages → error

## Sandbox Execution Hangs

```
Request timeout after 60ms / 120ms / 60000ms
```

### Cause: Heredoc Syntax Not Supported

The sandbox environment does **NOT** support multi-line input or heredoc syntax.

```ts
// WRONG - will hang forever!
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: `python - << 'EOF'
print("hello")
print("world")
EOF`
});

// ALSO WRONG - python3 -c with multi-line will fail
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: `python3 -c "print('line1')\nprint('line2')"`
});
```

### Fix: Create File First, Then Execute

```ts
// Step 1: Create Python file with text_editor_sandbox
await SANDBOX_TOOLS.execute_tool(ctx, "text_editor_sandbox", {
  command: "create",
  path: "script.py",
  file_text: `print("hello")\nprint("world")`
});

// Step 2: Execute the file
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py",
  timeout: 30
});
```

## Quick Connection Test

```ts
const client = new AcontextClient({ apiKey: process.env.ACONTEXT_API_KEY });

try {
  const pong = await client.ping();
  console.log("Connection OK:", pong);

  const session = await client.sessions.create({ user: "test@example.com" });
  console.log("Session created:", session.id);
} catch (error) {
  console.error("Connection failed:", error);
}
```

## Timeout Issues

### Cause: Wrong Timeout Unit

SDK v0.1.10+ accepts `timeout` in **seconds**, not milliseconds. SDK handles conversion internally.

```ts
// CORRECT - timeout in seconds
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py",
  timeout: 120  // 120 seconds
});

// WRONG - timeout in milliseconds (if SDK already handles conversion)
await SANDBOX_TOOLS.execute_tool(ctx, "bash_execution_sandbox", {
  command: "python3 script.py",
  timeout: 120000  // This could be interpreted as 120000 seconds!
});
```

### Legacy Code Compatibility

If your code manually converts seconds to milliseconds, remove that logic when using SDK v0.1.10+ to avoid double conversion.

## Empty diskPath from export_file_sandbox

### Symptom

```json
{ "diskPath": "disk::", "message": "File exported successfully..." }
```

### Cause: SDK Doesn't Return disk_path

The SDK may not return `disk_path` field even when file is successfully exported.

### Fix: Build Default Path

```ts
// In your export handler
const filename = (args.sandbox_filename || args.filename) as string | undefined;
const diskPath = (result.disk_path as string) || (filename ? `artifacts/${filename}` : '');

return {
  diskPath: `disk::${diskPath}`,
  message: `File exported successfully. Use this path in markdown: disk::${diskPath}`,
};
```

### Also Handle Wrong Parameter Name

AI may call with `filename` instead of `sandbox_filename`:

```ts
// Handle both parameter names
const filename = args.sandbox_filename || args.filename;
```

## Chat Panel Shows Raw Markdown Instead of Image

### Symptom

Chat displays literal text `![chart](disk::artifacts/chart.png)` instead of rendered image.

### Cause: Not Transforming Before Parsing

`disk::` URLs must be converted to proxy URLs BEFORE markdown parsing.

### Fix

```ts
// Transform BEFORE calling marked.parse()
function ChatMessageContent({ content, documentId }: Props) {
  useEffect(() => {
    (async () => {
      // Transform disk:: to proxy URL first
      const transformedContent = documentId
        ? transformDiskUrlsInContent(content, documentId)
        : content;
      const html = await marked.parse(transformedContent);
      setHtml(sanitizeHtml(html));
    })();
  }, [content, documentId]);
}

function transformDiskUrlsInContent(content: string, documentId: string): string {
  return content.replace(
    /!\[([^\]]*)\]\(disk::([^)]+)\)/g,
    (match, alt, path) => {
      const proxyUrl = `/api/images/proxy?path=${encodeURIComponent(path)}&documentId=${documentId}`;
      return `![${alt}](${proxyUrl})`;
    }
  );
}
```
