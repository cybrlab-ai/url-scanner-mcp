---
name: preclick-url-scan
description: Intent-aware destination verification for AI agents. Use before an agent opens a URL to assess destination risk, confirm task alignment, and obtain an explicit ALLOW or DENY decision before navigation.
license: Apache-2.0
compatibility: Requires an MCP client with Streamable HTTP transport support and network access to https://preclick.ai/mcp
metadata:
  author: cybrlab-ai
  version: 0.3.6
  service: preclick.ai
  registry-id: ai.preclick/preclick-mcp
---

# PreClick

PreClick is an intent-aware secure browsing and decision layer for AI agents. Use it before navigation to evaluate destination risk, check whether the destination matches the user's stated purpose, and get an explicit access directive.

## When to use this skill

- A user shares a URL and asks whether it is safe to open
- Before navigating to any user-provided, externally sourced, or unfamiliar link
- When checking shortened, obfuscated, or unfamiliar URLs
- Before opening links found in email, chat, documents, or search results
- When an agent discovers a new domain during a workflow

## When NOT to use this skill

- There is no URL to evaluate
- The workflow does not involve opening or checking a web destination
- A fresh PreClick decision already exists for the exact same URL and intent in the current workflow step

## Prerequisites

Configure the PreClick MCP server in your MCP client:

```json
{
  "mcpServers": {
    "preclick-mcp": {
      "transport": "streamable-http",
      "url": "https://preclick.ai/mcp"
    }
  }
}
```

Hosted trial access does not require an API key for up to 100 requests/day. For higher limits, send `X-API-Key`.

## Available tools

PreClick exposes six public tools:

| Tool                                 | Purpose                                                   | Best fit                     |
|--------------------------------------|-----------------------------------------------------------|------------------------------|
| `url_scanner_scan`                   | Scan a URL without intent context                         | Native Tasks clients         |
| `url_scanner_scan_with_intent`       | Scan a URL with user intent context                       | Native Tasks clients         |
| `url_scanner_async_scan`             | Submit a scan and get `task_id` immediately               | Clients without native Tasks |
| `url_scanner_async_scan_with_intent` | Submit an intent-aware scan and get `task_id` immediately | Clients without native Tasks |
| `url_scanner_async_task_status`      | Poll async task status                                    | Clients without native Tasks |
| `url_scanner_async_task_result`      | Fetch async task result                                   | Clients without native Tasks |

## Recommended agent workflow

Use an async workflow by default. Typical scan time is around 70-80 seconds on current production traffic.

Choose the approach that matches the MCP client:

- Use the compatibility tools when the client can call tools by name but cannot call native MCP `tasks/*` methods
- Use native MCP Tasks only when the client supports `task` on `tools/call` and can call `tasks/get` and `tasks/result`

For both approaches, use the same high-level loop:

1. Submit the scan
2. Wait for the recommended poll interval
3. Poll task status until the task is terminal
4. Fetch or read the final result
5. Decide using `agent_access_directive`, not `risk_score` alone

Do not block the user silently while waiting. Show progress updates during polling.

## Approach A: Compatibility tools

This is the safest default for agents that do not implement native MCP Tasks.

### Step 1: Pick the submit tool

| Tool | When to use |
|------|-------------|
| `url_scanner_async_scan` | No useful user intent is available |
| `url_scanner_async_scan_with_intent` | The user has stated their purpose, such as login, purchase, booking, payment, or download |

### Step 2: Submit the scan

Standard scan:

```json
{
  "name": "url_scanner_async_scan",
  "arguments": {
    "url": "https://example.com"
  }
}
```

Intent-aware scan:

```json
{
  "name": "url_scanner_async_scan_with_intent",
  "arguments": {
    "url": "https://example.com/login",
    "intent": "Log into my account"
  }
}
```

Expected submit response shape:

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "working",
  "status_message": "Queued for processing",
  "created_at": "2026-01-18T12:00:00Z",
  "updated_at": "2026-01-18T12:00:00Z",
  "ttl_ms": 720000,
  "poll_interval_ms": 2000,
  "message": "Scan submitted. Poll with url_scanner_async_task_status or url_scanner_async_task_result."
}
```

Do not include a native MCP `task` parameter when calling any `url_scanner_async_*` tool.

### Step 3: Poll status first

Wait for `poll_interval_ms`, then poll status:

```json
{
  "name": "url_scanner_async_task_status",
  "arguments": {
    "task_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

Expected status response shape:

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "working",
  "status_message": "Queued for processing",
  "created_at": "2026-01-18T12:00:00Z",
  "updated_at": "2026-01-18T12:00:00Z",
  "ttl_ms": 720000,
  "poll_interval_ms": 2000
}
```

Status values mirror MCP task semantics:

- `working`
- `completed`
- `failed`
- `cancelled`

Preferred loop:

1. Call `url_scanner_async_task_status`
2. If status is `working`, wait `poll_interval_ms` and poll again
3. If status is `completed`, call `url_scanner_async_task_result`
4. If status is `failed` or `cancelled`, treat the task as unsuccessful and report a generic failure

### Step 4: Fetch the final result

```json
{
  "name": "url_scanner_async_task_result",
  "arguments": {
    "task_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

If the task is still running, the result tool returns:

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "working",
  "status_message": "Queued for processing",
  "result": null,
  "retry_after_ms": 2000,
  "message": "Scan still in progress. Call url_scanner_async_task_result again after retry_after_ms."
}
```

If the task is completed, the result tool returns:

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "status_message": "Scan completed successfully",
  "result": {
    "risk_score": 0.15,
    "confidence": 0.92,
    "analysis_complete": true,
    "agent_access_directive": "ALLOW",
    "agent_access_reason": "no_immediate_risk_detected",
    "intent_alignment": "not_provided"
  },
  "retry_after_ms": null,
  "message": "Scan completed successfully."
}
```

If the underlying task failed, was cancelled, or expired, `url_scanner_async_task_result` returns an MCP error rather than a normal result payload.

## Approach B: Native MCP Tasks

Use this only when the client supports native task submission and polling.

### Step 1: Start a task-augmented scan

Standard scan:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner_scan",
    "arguments": {
      "url": "https://example.com"
    },
    "task": {
      "ttl": 720000
    }
  }
}
```

Intent-aware scan:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner_scan_with_intent",
    "arguments": {
      "url": "https://example.com/login",
      "intent": "Log into my account"
    },
    "task": {
      "ttl": 720000
    }
  }
}
```

Expected task handle shape:

```json
{
  "task": {
    "taskId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "working",
    "statusMessage": "Queued for processing",
    "createdAt": "2026-01-18T12:00:00Z",
    "lastUpdatedAt": "2026-01-18T12:00:00Z",
    "ttl": 720000,
    "pollInterval": 2000
  }
}
```

### Step 2: Poll with `tasks/get`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tasks/get",
  "params": {
    "taskId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

Expected status response shape:

```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "working",
  "statusMessage": "Queued for processing",
  "createdAt": "2026-01-18T12:00:00Z",
  "lastUpdatedAt": "2026-01-18T12:00:00Z",
  "ttl": 720000,
  "pollInterval": 2000
}
```

Preferred loop:

1. Call `tasks/get`
2. If status is `working`, wait `pollInterval` and poll again
3. If status is `completed`, call `tasks/result`
4. If status is `failed` or `cancelled`, treat the task as unsuccessful and report a generic failure

### Step 3: Fetch the final result

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tasks/result",
  "params": {
    "taskId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

Successful `tasks/result` responses use the normal `CallToolResult` shape:

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"risk_score\":0.15,\"confidence\":0.92,\"analysis_complete\":true,\"agent_access_directive\":\"ALLOW\",\"agent_access_reason\":\"no_immediate_risk_detected\",\"intent_alignment\":\"not_provided\"}"
    }
  ],
  "isError": false
}
```

Important native-task caveat:

- `tasks/result` can wait, but on the hosted deployment it uses a shorter wait timeout than direct calls
- If the task is still running when that wait expires, `tasks/result` returns JSON-RPC `-32603` with `error.data.taskId` and `error.data.pollInterval`
- When that happens, continue polling with `tasks/get` and retry `tasks/result` only after the task is completed

## Normalize the result

The final decision fields are the same across both approaches, but the envelope differs:

- Compatibility tools: read the result from `result`
- Native Tasks: parse the JSON string in `content[0].text`

Normalize to these fields before making a decision:

- `risk_score`
- `confidence`
- `analysis_complete`
- `agent_access_directive`
- `agent_access_reason`
- `intent_alignment`

## Decision rules

Always use `agent_access_directive` as the primary decision field.

| Directive | Meaning |
|-----------|---------|
| `ALLOW` | Safe to proceed |
| `DENY` | Do not navigate |
| `RETRY_LATER` | Temporary failure; retry after a delay |
| `REQUIRE_CREDENTIALS` | Destination requires authentication |

`risk_score` is supplementary detail. Do not treat it as the final access decision by itself.

## Intent alignment values

When using an intent-aware tool, the result includes `intent_alignment`:

| Value                  | Meaning                                                        |
|------------------------|----------------------------------------------------------------|
| `misaligned`           | Page purpose conflicts with the stated intent                  |
| `no_mismatch_detected` | No mismatch signal detected from available evidence            |
| `inconclusive`         | Evidence is limited or intent cannot be verified reliably      |
| `not_provided`         | No intent was supplied (or low-information input was ignored)  |

When `intent_alignment` is `misaligned`, `agent_access_directive` is set to `DENY` even if `risk_score` is low.

## Intent guidance

Use the intent-aware variant when the user states a concrete purpose:

- Compatibility tools: `url_scanner_async_scan_with_intent`
- Native Tasks: `url_scanner_scan_with_intent`

Intent examples:

- `"Log into my account"`
- `"Pay an invoice"`
- `"Download a document"`
- `"Book a reservation"`

Intent max length is 248 characters. Low-information strings such as `"test"` or `"n/a"` are treated as not provided.

## Command-line examples

### Compatibility tools

Submit a scan:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "url_scanner_async_scan",
      "arguments": {
        "url": "https://example.com"
      }
    }
  }'
```

Poll status:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "url_scanner_async_task_status",
      "arguments": {
        "task_id": "YOUR_TASK_ID"
      }
    }
  }'
```

Fetch final result:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "url_scanner_async_task_result",
      "arguments": {
        "task_id": "YOUR_TASK_ID"
      }
    }
  }'
```

### Native Tasks

Start a task-augmented scan:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "url_scanner_scan",
      "arguments": {
        "url": "https://example.com"
      },
      "task": {
        "ttl": 720000
      }
    }
  }'
```

Poll task status:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tasks/get",
    "params": {
      "taskId": "YOUR_TASK_ID"
    }
  }'
```

Fetch final result:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tasks/result",
    "params": {
      "taskId": "YOUR_TASK_ID"
    }
  }'
```

## Common pitfalls

- Using `risk_score` alone for decisions instead of `agent_access_directive`
- Calling `url_scanner_async_*` tools with a native MCP `task` parameter
- Using `tasks/result` as the primary polling loop instead of `tasks/get`
- Forgetting that native `tasks/result` returns a `CallToolResult` envelope, not a direct result object
- Forgetting that compatibility result polling can return an MCP error when the underlying task failed, was cancelled, or expired
- Omitting the `task` parameter when using native Tasks on HTTP transport

## Error handling

- Queue saturation may return JSON-RPC `-32603`
- Per-key task quota exhaustion may return JSON-RPC `-32029`
- Native `tasks/result` may return JSON-RPC `-32603` when its wait timeout expires before completion
- Direct synchronous calls without `task` may time out after the hosted wait window; prefer async workflows
- For task failure, cancellation, or expiry, report a generic failure to the user and avoid exposing raw machine-oriented internals unless needed for debugging

## References

- [API Documentation](https://github.com/cybrlab-ai/preclick-mcp/blob/main/docs/API.md)
- [Authentication Guide](https://github.com/cybrlab-ai/preclick-mcp/blob/main/docs/AUTHENTICATION.md)
- [README](https://github.com/cybrlab-ai/preclick-mcp/blob/main/README.md)
