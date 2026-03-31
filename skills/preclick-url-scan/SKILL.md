---
name: preclick-url-scan
description: Intent-aware destination verification for AI agents. Use before an agent opens a URL to assess destination risk, confirm task alignment, and obtain an explicit ALLOW or DENY decision before navigation.
license: Apache-2.0
compatibility: Requires an MCP client with Streamable HTTP transport support and network access to https://preclick.ai/mcp
metadata:
  author: cybrlab-ai
  version: 0.3.2
  service: preclick.ai
  registry-id: ai.preclick/preclick-mcp
---

# PreClick

PreClick is an intent-aware secure browsing and decision layer for AI agents. Use it to verify a destination before navigation, evaluate both risk and task alignment, and obtain an explicit access decision before an agent proceeds.

## When to use this skill

- A user shares a URL and asks whether it is safe
- Before navigating to any user-provided, externally sourced, or unfamiliar link
- When a workflow requires URL preflight validation
- When checking shortened, obfuscated, or unfamiliar URLs
- Before clicking any link from an email, chat message, or untrusted source
- When browsing results contain unfamiliar domains
- When an agent autonomously discovers a URL it has not visited before

## When NOT to use this skill

- There is no URL or destination to evaluate
- The workflow does not involve opening or checking a web destination
- A fresh PreClick decision already exists for the exact same URL and intent in the current workflow step

## Prerequisites

The PreClick MCP server must be configured in your MCP client settings:

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

No API key is required for the hosted trial tier (up to 100 requests/day). For higher limits, add an `X-API-Key` header.

## Command-line examples

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
      "arguments": { "url": "https://example.com" },
      "task": { "ttl": 720000 }
    }
  }'
```

Scan with intent context:

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
      "name": "url_scanner_scan_with_intent",
      "arguments": {
        "url": "https://example.com/login",
        "intent": "Log into my email account"
      },
      "task": { "ttl": 720000 }
    }
  }'
```

Poll for the final result:

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tasks/result",
    "params": { "taskId": "YOUR_TASK_ID" }
  }'
```

## How to scan a URL

### Step 1: Choose the right tool

| Tool                           | When to use                                                         |
|--------------------------------|---------------------------------------------------------------------|
| `url_scanner_scan`             | Standard scan (no intent context available)                         |
| `url_scanner_scan_with_intent` | When the user has stated their purpose (login, purchase, booking)   |

### Step 2: Start a task-augmented scan (recommended)

Call the tool with a `task` parameter for async execution:

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

The response includes a `taskId` and `pollInterval`.

### Step 3: Poll for results

Use `tasks/get` to check status, then `tasks/result` to retrieve the final result:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tasks/result",
  "params": {
    "taskId": "<taskId from step 2>"
  }
}
```

### Step 4: Interpret the result

The result contains:

| Field                    | Type   | How to use                                                                           |
|--------------------------|--------|--------------------------------------------------------------------------------------|
| `agent_access_directive` | string | **Primary decision field.** `ALLOW`, `DENY`, `RETRY_LATER`, or `REQUIRE_CREDENTIALS` |
| `risk_score`             | number | 0.0 (safe) to 1.0 (dangerous) — supplementary detail                                 |
| `confidence`             | number | Confidence (0.0–1.0)                                                                 |
| `agent_access_reason`    | string | Machine-readable reason code                                                         |
| `intent_alignment`       | string | `misaligned`, `no_mismatch_detected`, `inconclusive`, or `not_provided`              |

**Decision logic:**

- `ALLOW` → Safe to proceed
- `DENY` → Do not navigate (threat detected, invalid URL, or insufficient evidence)
- `RETRY_LATER` → Temporary failure (connection error, rate limited) — retry after a delay
- `REQUIRE_CREDENTIALS` → The destination requires authentication

Always use `agent_access_directive` for access decisions, not `risk_score` alone.

## Using intent context

When the user states their purpose, use `url_scanner_scan_with_intent` for better detection:

```json
{
  "name": "url_scanner_scan_with_intent",
  "arguments": {
    "url": "https://example.com/login",
    "intent": "Log into my email account"
  },
  "task": { "ttl": 720000 }
}
```

Intent max length: 248 characters. Low-information strings (e.g., "test", "n/a") are treated as not provided.

## Typical scan duration

- Typical scan time: around 70-80 seconds on current production traffic

Plan for an async workflow. Do not block the user while waiting — show progress updates.

## Common pitfalls

- **Using `risk_score` alone for decisions.** Always use `agent_access_directive` (`ALLOW` / `DENY` / `RETRY_LATER` / `REQUIRE_CREDENTIALS`). The directive already incorporates risk score, confidence, policy gates, and intent alignment into a single actionable decision.
- **Displaying `agent_access_reason` to end users.** Reason codes are machine-readable identifiers for logging and programmatic logic, not user-friendly labels. Show the directive, not the reason.
- **Ignoring `RETRY_LATER`.** A `RETRY_LATER` directive means a transient failure (connection error, rate limit, server error at the destination). Retry after a short delay instead of treating it as `DENY`.
- **Omitting `task` parameter on HTTP transport.** Without it, direct calls may block up to 90 seconds and still require the client to keep the connection open. Always include `"task": { "ttl": 720000 }` for production use when your client supports native Tasks.
- **Using a client without native MCP Tasks support.** If the client cannot call `tasks/get` / `tasks/result`, use the compatibility tools: `url_scanner_async_scan`, `url_scanner_async_scan_with_intent`, `url_scanner_async_task_status`, and `url_scanner_async_task_result`. Call these as ordinary tools only; do not include a native MCP `task` parameter.

## Error handling

- **Queue full / rate limit**: JSON-RPC errors `-32603` or `-32029` — retry after a short delay
- **Direct-call timeout**: If not using `task` parameter, calls time out after 90 seconds; the error includes a `taskId` for recovery polling
- **Task failure**: `tasks/get` returns `status: "failed"` — report a generic error to the user

## References

- [API Documentation](https://github.com/cybrlab-ai/preclick-mcp/blob/main/docs/API.md) — Full schema and error code reference
- [Authentication Guide](https://github.com/cybrlab-ai/preclick-mcp/blob/main/docs/AUTHENTICATION.md) — API key setup
- [README](https://github.com/cybrlab-ai/preclick-mcp/blob/main/README.md) — Quick start and configuration
