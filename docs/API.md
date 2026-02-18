# URLCheck MCP Server - API Documentation

> **An MCP-native URL preflight scanning service for autonomous agents. It scans links for threats and confirms they match the intended task before execution. Built for agentic workflows, it provides high-accuracy, context-aware browsing governance with adaptive learning.**

> **Publisher:** [CybrLab.ai](https://cybrlab.ai) | **Service:** [URLCheck](https://urlcheck.dev)

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Transport & Connection](#transport--connection)
4. [Tools Reference](#tools-reference)
   - [url_scanner_scan](#url_scanner_scan)
   - [url_scanner_scan_with_intent](#url_scanner_scan_with_intent)
   - [tasks/get](#tasksget)
   - [tasks/result](#tasksresult)
   - [tasks/list](#taskslist)
   - [tasks/cancel](#taskscancel)
5. [Risk Score](#risk-score)
6. [HTTP Status Codes](#http-status-codes)
7. [Error Codes](#error-codes)
   - [JSON-RPC Standard Errors](#json-rpc-standard-errors)
   - [Authentication & Rate Limit Errors](#authentication--rate-limit-errors)
   - [Request Validation Errors (Request-Level)](#request-validation-errors-request-level)
   - [URL Validation Early-Exit Results (Task-Level)](#url-validation-early-exit-results-task-level)
   - [Task Execution Failures](#task-execution-failures)
8. [Rate Limits & Quotas](#rate-limits--quotas)
9. [Support](#support)

---

## Overview

The URLCheck MCP Server provides AI-powered malicious URL detection through the Model Context Protocol (MCP). 

### Key Characteristics

| Property          | Value                            |
|-------------------|----------------------------------|
| Typical Scan Time | 30-90 seconds (varies by target) |
| Supported Schemes | HTTP, HTTPS                      |

---

## Authentication

Authentication requirements depend on deployment mode:

- Hosted endpoint (`https://urlcheck.ai/mcp`): API key is optional for up to 100 requests/day.
- Hosted endpoint above trial quota: API key required.

```http
X-API-Key: your-api-key-here
```

---

## Transport & Connection

### Streamable HTTP (Production)

**Endpoint**: `https://urlcheck.ai/mcp`

| Method   | Path   | Description                              |
|----------|--------|------------------------------------------|
| `POST`   | `/mcp` | JSON-RPC requests                        |
| `GET`    | `/mcp` | SSE stream (stateful mode only)          |
| `DELETE` | `/mcp` | Session termination (stateful mode only) |

### Required Headers

| Header                 | Required                                                       | Description                                                           |
|------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------|
| `Content-Type`         | POST                                                           | Must be `application/json`                                            |
| `Accept`               | POST/GET                                                       | POST: `application/json, text/event-stream`; GET: `text/event-stream` |
| `X-API-Key`            | Hosted: optional up to trial quota; required above trial quota | API key for authenticated usage tiers                                 |
| `Mcp-Session-Id`       | Stateful                                                       | Session ID returned by `initialize`                                   |
| `MCP-Protocol-Version` | All non-initialize requests                                    | Protocol version (e.g., `2025-06-18`)                                 |

### Session Management

The server supports two modes:

- **Stateless (default):** No session IDs. Only POST `/mcp` is used.
- **Stateful:** Requires `initialize`, uses `Mcp-Session-Id`, enables GET/DELETE `/mcp`.

### Example: Initialize Session (stateful mode only)

```bash
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": {"name": "my-client", "version": "1.0"}
    }
  }'
# Response includes Mcp-Session-Id header
```

---

## Tools Reference

### Tool Safety Annotations

The two scanner tools are declared as read-only in `server.json`:

- `readOnlyHint: true`
- `destructiveHint: false`
- `openWorldHint: true`

These annotations are included for:

- `url_scanner_scan`
- `url_scanner_scan_with_intent`

These same annotations are also returned by the live MCP server in `tools/list` responses.

### url_scanner_scan

Start a URL security scan. Supports both task-augmented (async) and direct (sync) execution modes.

**This is the recommended approach for production use.**

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "description": "The URL to analyze. Must be HTTP or HTTPS. If no scheme provided, https:// is assumed."
    }
  },
  "required": ["url"]
}
```

#### Output Schema (Task-Augmented)

When the `task` parameter is included, returns a task handle for polling:

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

#### Output Schema (Direct Call)

When the `task` parameter is omitted, returns the scan result directly (synchronous):

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"risk_score\":0.15,\"confidence\":0.92,\"analysis_complete\":true,\"agent_access_directive\":\"ALLOW\",\"agent_access_reason\":\"clean\",\"intent_alignment\":\"not_provided\"}"
    }
  ]
}
```

#### Output Schema (Error)

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Server busy: Queue is full. Please retry later."
  },
  "id": 1
}
```

#### Example Requests

`url_scanner_scan` supports both execution modes per the MCP specification.

**Task-Augmented (Recommended for HTTP)**

Include the `task` parameter to run asynchronously with queue management:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner_scan",
    "arguments": {
      "url": "https://suspicious-site.com/login"
    },
    "task": {
      "ttl": 720000
    }
  }
}
```

**Direct Call (stdio transport)**

Omit the `task` parameter for synchronous execution:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner_scan",
    "arguments": {
      "url": "https://example.com"
    }
  }
}
```

**Direct Call Timeout:** Direct calls have a server-side wait timeout (deployment-configured; hosted default is **100 seconds**).

On direct-call timeout, the server returns JSON-RPC `-32603` with:
- `data.taskId` (use for recovery polling)
- `data.pollInterval` (recommended polling interval, ms)

For scans that may exceed this window, prefer task-augmented calls with polling.
If an upstream edge/proxy timeout fires first (for example Cloudflare 524), this timeout payload may not reach the client.

---

### url_scanner_scan_with_intent

Start a URL security scan with optional user intent context. This tool behaves like `url_scanner_scan`
but accepts a recommended `intent` field that can improve detection when aligned with substantive page evidence.

Recommended use: provide `intent` when the user can state their purpose (e.g., login, purchase, booking, payments, file download).
This helps compare intended purpose against observed page content as an additional contextual signal.

If `intent` is omitted or empty, the scan proceeds normally.

**Max intent length:** 248 characters.

Scan results include `intent_alignment` in both direct-call and task-result responses:
- `misaligned`: page purpose appears to conflict with stated intent
- `no_mismatch_detected`: no explicit mismatch signal detected from available evidence
- `inconclusive`: evidence is limited or intent cannot be verified reliably
- `not_provided`: no intent was supplied

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "description": "The URL to analyze. Must be HTTP or HTTPS. If no scheme provided, https:// is assumed."
    },
    "intent": {
      "type": "string",
      "description": "Optional user intent for visiting the URL. Recommended for additional context."
    }
  },
  "required": ["url"]
}
```

#### Example Request (Task-Augmented)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner_scan_with_intent",
    "arguments": {
      "url": "https://example.com",
      "intent": "Book a hotel room"
    },
    "task": {
      "ttl": 720000
    }
  }
}
```

---

### tasks/get

Get task status (non-blocking). Task IDs are scoped to the API key that created them.

#### Input Schema

```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Output Schema

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

#### Task Status Values

| Status      | Description                | Terminal |
|-------------|----------------------------|----------|
| `working`   | Task in progress           | No       |
| `completed` | Task finished successfully | Yes      |
| `failed`    | Task failed with error     | Yes      |
| `cancelled` | Task cancelled by user     | Yes      |

---

### tasks/result

Wait for task completion and return the tool result.

**Note:** The server enforces the same wait timeout as direct calls (hosted default **100 seconds**). If the task is still running after this timeout, `tasks/result` returns a JSON-RPC error (`-32603`). Since the client already has the `taskId`, it can fall back to polling with `tasks/get`.

#### Input Schema

```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Output Schema (Success)

```json
{
  "contentType": "application/json",
  "value": {
    "risk_score": 0.65,
    "confidence": 0.75,
    "analysis_complete": true,
    "agent_access_directive": "DENY",
    "agent_access_reason": "suspicious",
    "intent_alignment": "not_provided"
  },
  "summary": "URL scan completed"
}
```

#### Output Schema (Error)

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Task expired"
  },
  "id": 1
}
```

---

### tasks/list

List tasks for the current API key.

#### Input Schema

```json
{}
```

#### Output Schema

```json
{
  "tasks": [
    {
      "taskId": "550e8400-e29b-41d4-a716-446655440000",
      "status": "working",
      "statusMessage": "Queued for processing",
      "createdAt": "2026-01-18T12:00:00Z",
      "lastUpdatedAt": "2026-01-18T12:00:00Z",
      "ttl": 720000,
      "pollInterval": 2000
    }
  ],
  "total": 1
}
```

---

### tasks/cancel

Cancel a queued or running task.

#### Input Schema

```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Output Schema

```json
{}
```

---

## Risk Score

A float between 0.0 and 1.0 indicating threat probability:
- **0.0** — Safe, no threats detected
- **1.0** — High threat probability

Use `agent_access_directive` for access decisions. It translates the risk score into actionable guidance:
- `ALLOW` — Safe to proceed
- `DENY` — Do not access (threat detected, invalid URL, or uncertain/undetected risk)
- `RETRY_LATER` — Temporary failure (connection error, rate limited)
- `REQUIRE_CREDENTIALS` — Authentication required

---

## HTTP Status Codes

The MCP server follows the MCP 2025-06-18 specification for HTTP status codes.

### Success Codes (2xx)

| Code | Status     | When Returned                                                   |
|------|------------|-----------------------------------------------------------------|
| 200  | OK         | Successful request with JSON or SSE response                    |
| 202  | Accepted   | Notification received (no response body)                        |
| 204  | No Content | DELETE /mcp session termination successful (stateful mode only) |

### Client Error Codes (4xx)

| Code | Status                 | When Returned                                               |
|------|------------------------|-------------------------------------------------------------|
| 400  | Bad Request            | Invalid JSON, missing headers, invalid protocol version     |
| 401  | Unauthorized           | Missing/invalid `X-API-Key` when authentication is required |
| 403  | Forbidden              | CORS origin rejected or IP blocked                          |
| 404  | Not Found              | Unknown/expired session ID, wrong endpoint                  |
| 413  | Payload Too Large      | Request body exceeds size limit                             |
| 415  | Unsupported Media Type | Missing or wrong `Content-Type` header                      |
| 429  | Too Many Requests      | Rate limit exceeded                                         |

### Server Error Codes (5xx)

| Code | Status                | When Returned                 |
|------|-----------------------|-------------------------------|
| 500  | Internal Server Error | Unhandled server exception    |

> Note: Queue saturation is reported as a JSON-RPC error (`-32603` with `"Server busy: ..."`), not an HTTP 503.

### Example Error Responses

**401 Unauthorized** (missing API key when auth is required):

Authentication failures return HTTP responses without a JSON-RPC body:
```
HTTP/1.1 401 Unauthorized
```

**429 Rate Limited** (transport rate limit):

Rate limiting at the transport layer returns HTTP 429 without a JSON-RPC body:
```
HTTP/1.1 429 Too Many Requests
```

**400 Bad Request** (missing Accept header):
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32600,
    "message": "Missing Accept header. POST requests must include 'Accept: application/json, text/event-stream'"
  },
  "id": null
}
```

---

## Error Codes

### JSON-RPC Errors

Tool invocation and task operations return JSON-RPC errors (no tool-level `isError` payloads).

Common cases:
- `url_scanner_scan` missing `url` → `-32602 Invalid params`
- `url_scanner_scan_with_intent` missing `url` → `-32602 Invalid params`
- `url_scanner_scan_with_intent` intent too long (>248 chars) → `-32602 Invalid params`
- Queue full → `-32603 Internal error` with `"Server busy: ..."`
- Per-key task quota exceeded → `-32029` with `"Rate limit exceeded: ..."`
- Direct-call wait timeout → `-32603` with timeout message and `data.taskId` / `data.pollInterval`
- Invalid `taskId` for `tasks/get`, `tasks/result`, or `tasks/cancel` → `-32602`
- Task execution failures (failed/expired/cancelled) → JSON-RPC errors with message string

**Example response (queue full):**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Server busy: Queue is full. Please retry later."
  },
  "id": 1
}
```

### URL Validation Outcomes

URL validation runs **after** task creation and produces a completed task result with `agent_access_*` fields. It does **not** return JSON-RPC errors.

Typical reasons include:
- `invalid_url`, `invalid_scheme`, `missing_host`
- `local_network_only`, `reserved_domain`, `tor_network_only`, `overlay_network_only`

Additional security validation reasons may be returned.

Malformed requests (missing `url` or wrong types) still return `-32602 Invalid params`.

### JSON-RPC Standard Errors

| Code   | Message          | HTTP Status | Description                                          |
|--------|------------------|-------------|------------------------------------------------------|
| -32700 | Parse error      | 400         | Invalid JSON syntax                                  |
| -32600 | Invalid request  | 400         | Malformed JSON-RPC envelope                          |
| -32601 | Method not found | 200         | Unknown method name                                  |
| -32602 | Invalid params   | 200         | Invalid method parameters (missing/invalid fields)   |
| -32603 | Internal error   | 200         | Server error from handler (queue/task failures)      |

### JSON-RPC Custom Errors

| Code    | Message                       | HTTP Status | Description                                     |
|---------|-------------------------------|-------------|-------------------------------------------------|
| -32000  | Server validation error       | 400/404/413 | Payload/session/batch validation                |
| -32029  | Task quota exceeded           | 200         | Per-key concurrent task quota exceeded          |

### Authentication & Rate Limit Errors

**Note:** Authentication errors return plain HTTP responses (not JSON-RPC envelopes).

| HTTP Status | Response Type | Body | Description                         |
|-------------|---------------|------|-------------------------------------|
| 401         | Empty body    | —    | No API key provided (when required) |
| 401         | Empty body    | —    | API key not recognized              |
| 401         | Empty body    | —    | API key has expired                 |
| 403         | Empty body    | —    | CORS origin rejected                |
| 429         | Empty body    | —    | Transport rate limited              |

### Request Validation Errors (Request-Level)

Request validation errors return JSON-RPC errors:

| Request Error                       | Error Code | Example Message                                                         |
|-------------------------------------|------------|-------------------------------------------------------------------------|
| Missing `url` parameter (tool call) | -32602     | `Missing 'url' parameter`                                               |
| Invalid JSON-RPC envelope           | -32600     | `Missing required field 'jsonrpc'. Must be "2.0"`                       |
| Payload too large                   | -32000     | `Payload too large: ...`                                                |
| Missing/invalid/expired session     | -32000     | `Missing Mcp-Session-Id header. Session required after initialization.` |

### URL Validation Early-Exit Results (Task-Level)

URL validation happens **after** task creation. Invalid URLs produce a **completed task result** with `agent_access_directive`, `agent_access_reason`, and `intent_alignment`:

| Validation Result                           | Example Reason         | Notes                                  |
|---------------------------------------------|------------------------|----------------------------------------|
| Too short/too long                          | `invalid_url`          | Early exit                             |
| Invalid scheme                              | `invalid_scheme`       | Early exit                             |
| Missing host                                | `missing_host`         | Early exit                             |
| `.local`/`.internal`/`.home.arpa` domain    | `local_network_only`   | Early exit (local/private network)     |
| `.test`/`.invalid`/`.example`/`.alt` domain | `reserved_domain`      | Early exit (reserved domain namespace) |
| `.onion` domain                             | `tor_network_only`     | Early exit (Tor network)               |
| `.i2p` domain                               | `overlay_network_only` | Early exit (I2P overlay network)       |

---

### Task Execution Failures

Task execution failures are returned as JSON-RPC errors with a message string only.

---

## Rate Limits & Quotas

The API enforces rate limits to ensure fair usage and service availability. Limits may be adjusted without notice based on operational requirements.

### Hosted Trial Limit (`urlcheck.ai`)

| Limit            | Description                                      |
|------------------|--------------------------------------------------|
| 100 requests/day | Anonymous access (no API key) on hosted endpoint |

Above trial quota, use an API key.

### Per-API-Key Limits

| Limit               | Description                        |
|---------------------|------------------------------------|
| Requests per minute | Enforced per API key               |
| Concurrent tasks    | Limited active tasks per API key   |

### Global Limits

| Limit          | Description                    |
|----------------|--------------------------------|
| Queue capacity | Total pending tasks is limited |
| Task timeout   | Long-running scans are stopped |

> **Note:** Your API key tier determines specific limit values. Contact support for details on available tiers.

### Rate Limit Responses

**Per-Key Task Quota (JSON-RPC -32029)**

When the per-key concurrent task quota is exceeded, the server returns a JSON-RPC error:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32029,
    "message": "Rate limit exceeded: Per-key task quota exceeded"
  },
  "id": 1
}
```

**Transport Rate Limit (HTTP 429)**

Transport-level rate limiting returns HTTP 429 without a JSON-RPC body.

---

## Support

For API keys, support, or questions:
- **Publisher**: [CybrLab.ai](https://cybrlab.ai)
- **Service**: [URLCheck](https://urlcheck.dev)
- **Email**: contact@cybrlab.ai
