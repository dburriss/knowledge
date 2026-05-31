---
description: "MCP Tools let servers expose executable actions that language models can discover and invoke — querying databases, calling APIs, running computations, etc."
tags: [mcp]
---
# MCP Tools

> Spec: https://modelcontextprotocol.io/specification/2025-11-25/server/tools

Tools let MCP servers expose executable actions that language models can discover and invoke — querying databases, calling APIs, running computations, etc.

---

## Key Design Principle

Tools are **model-controlled**: the LLM decides when to call them based on context. But implementations **SHOULD** always keep a human in the loop — show which tools are exposed, indicate when they fire, and prompt for confirmation on consequential operations.

---

## Capability Declaration

A server that supports tools must declare it during initialization:

```json
{
  "capabilities": {
    "tools": {
      "listChanged": true
    }
  }
}
```

`listChanged: true` means the server will send `notifications/tools/list_changed` when the tool list changes.

---

## Tool Definition

Each tool has:

| Field | Required | Description |
|---|---|---|
| `name` | yes | Unique identifier (1–128 chars, `[A-Za-z0-9_\-.]`, no spaces) |
| `title` | no | Human-readable display name |
| `description` | yes | What the tool does (used by the LLM to decide when to call it) |
| `inputSchema` | yes | JSON Schema (2020-12 default) describing expected arguments |
| `outputSchema` | no | JSON Schema for structured output validation |
| `icons` | no | Array of icon objects for UI display |
| `annotations` | no | Metadata about tool behavior (treat as untrusted unless from a trusted server) |
| `execution` | no | Execution properties, e.g. `taskSupport` |

### inputSchema rules

- Must be a valid JSON Schema object (not `null`).
- For tools with no parameters, use `{ "type": "object", "additionalProperties": false }` (recommended).
- Defaults to JSON Schema 2020-12; specify `$schema` explicitly to use draft-07 etc.

### Tool name rules

- 1–128 characters, case-sensitive.
- Allowed characters: `A-Z a-z 0-9 _ - .`
- No spaces, commas, or other special characters.
- Must be unique within a server.
- Examples: `getUser`, `DATA_EXPORT_v2`, `admin.tools.list`

### outputSchema

When provided, the server **MUST** return `structuredContent` conforming to the schema, and clients **SHOULD** validate against it. For backwards compatibility, also include the JSON serialized in a `TextContent` block.

---

## Protocol Messages

### Discovery — `tools/list`

```json
// Request
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": { "cursor": "..." } }

// Response
{
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": { "location": { "type": "string", "description": "City name or zip code" } },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

Supports pagination via `cursor` / `nextCursor`.

### Invocation — `tools/call`

```json
// Request
{ "jsonrpc": "2.0", "id": 2, "method": "tools/call", "params": { "name": "get_weather", "arguments": { "location": "New York" } } }

// Response
{
  "result": {
    "content": [{ "type": "text", "text": "Temperature: 72°F, Partly cloudy" }],
    "isError": false
  }
}
```

### List-changed notification

```json
{ "jsonrpc": "2.0", "method": "notifications/tools/list_changed" }
```

---

## Tool Result Content Types

Results go in the `content` array. Multiple items of different types are allowed. All types support optional `annotations` (audience, priority, lastModified).

| Type | Fields | Notes |
|---|---|---|
| `text` | `text` | Plain text output |
| `image` | `data` (base64), `mimeType` | Inline image |
| `audio` | `data` (base64), `mimeType` | Inline audio |
| `resource_link` | `uri`, `name`, `description`, `mimeType` | Pointer to a resource; client may fetch/subscribe |
| `resource` | `resource.uri`, `resource.mimeType`, `resource.text` | Embedded resource content inline |

For **structured** output, use `structuredContent` (a JSON object) alongside a serialized `TextContent` fallback.

---

## Error Handling

Two distinct mechanisms:

### Protocol errors (JSON-RPC level)
For unknown tools, malformed requests, server failures. Less actionable for the model.

```json
{ "error": { "code": -32602, "message": "Unknown tool: invalid_tool_name" } }
```

### Tool execution errors (`isError: true`)
For API failures, input validation errors, business logic errors. The LLM can use these to self-correct and retry.

```json
{
  "result": {
    "content": [{ "type": "text", "text": "Invalid departure date: must be in the future." }],
    "isError": true
  }
}
```

Clients **SHOULD** pass execution errors back to the model to enable self-correction.

---

## Task-Augmented Execution

The `execution.taskSupport` field on a tool controls long-running task support:

| Value | Meaning |
|---|---|
| `"forbidden"` | Default; tasks not supported |
| `"optional"` | Client may use task-augmented execution |
| `"required"` | Client must use task-augmented execution |

---

## Security

**Servers must:**
- Validate all tool inputs
- Implement access controls
- Rate limit invocations
- Sanitize outputs

**Clients should:**
- Prompt for user confirmation on sensitive operations
- Show tool inputs to the user before calling (prevents data exfiltration)
- Validate results before passing to the LLM
- Implement call timeouts
- Log tool usage for audit

---

## Design Guidance

- Write `description` for the LLM, not the human — it directly influences when the model chooses the tool.
- Keep `inputSchema` strict: use `required`, `additionalProperties: false`, and descriptive field descriptions.
- Use `outputSchema` whenever returning structured data — it enables validation and improves LLM parsing.
- Return execution errors with `isError: true` and actionable messages so the model can retry correctly.
- Prefer `resource_link` over embedding large resources inline when the client can fetch them.
- Declare `listChanged` and emit notifications if your tool set is dynamic.
