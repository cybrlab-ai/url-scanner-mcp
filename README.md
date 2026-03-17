# PreClick MCP Server

[![smithery badge](https://smithery.ai/badge/cybrlab-ai/urlcheck-mcp)](https://smithery.ai/server/cybrlab-ai/urlcheck-mcp)

> **PreClick — An MCP-native URL preflight scanning service for autonomous agents. It scans links for threats and confirms they match the intended task before execution. Built for agentic workflows, it provides high-accuracy, context-aware browsing governance with adaptive learning.**

**Publisher:** [CybrLab.ai](https://cybrlab.ai) | **Service:** [PreClick](https://preclick.ai)

**Hosted Trial Tier:** No API key required for up to 100 requests/day. For higher limits and stable quotas, use an API key (contact [contact@cybrlab.ai](mailto:contact@cybrlab.ai)).

---

## Overview

PreClick is an MCP server that enables AI agents and any MCP-compatible client to analyze URLs for malicious content and security threats before navigation.

## Integrations

PreClick works with any MCP-compatible client. For framework-specific adapters:

| Integration           | Repository                                                             |
|-----------------------|------------------------------------------------------------------------|
| LangChain / LangGraph | [langchain-urlcheck](https://github.com/cybrlab-ai/langchain-urlcheck) |
| OpenClaw plugin       | [urlcheck-openclaw](https://github.com/cybrlab-ai/urlcheck-openclaw)   |

For manual MCP bridge configuration (any client), see [Quick Start](#quick-start) below.

## Authentication Modes

| Deployment                         | `X-API-Key` Requirement         | Notes                                 |
|------------------------------------|---------------------------------|---------------------------------------|
| Hosted (`https://preclick.ai/mcp`) | Optional up to 100 requests/day | API key recommended for higher limits |
| Hosted (`https://preclick.ai/mcp`) | Required above trial quota      | Contact support for provisioned keys  |

## Important Notice

This tool is intended for authorized security assessment only. Use it solely on systems or websites that you own or for which you have got explicit permission to assess. Any unauthorized, unlawful, or malicious use is strictly prohibited. You are responsible for ensuring compliance with all applicable laws, regulations, and contractual obligations.

### Use Cases

- Pre-flight URL validation for AI agents
- Automated URL security scanning in workflows
- Malicious link detection in emails/messages

---

## Quick Start

### 1. Configure Your MCP Client

Choose one option:

Trial (hosted, up to 100 requests/day without API key):

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

Authenticated (recommended for stable and higher-volume usage):

```json
{
  "mcpServers": {
    "preclick-mcp": {
      "transport": "streamable-http",
      "url": "https://preclick.ai/mcp",
      "headers": {
        "X-API-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### 2. Optional: Initialize Session (stateful mode only)

```bash
# Only required if the server is running in stateful mode
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: YOUR_API_KEY" \
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
# Response includes Mcp-Session-Id header - save it for subsequent requests
```

### 3. Start a Scan

`url_scanner_scan` supports two execution modes (the same modes apply to `url_scanner_scan_with_intent`):
- **Task-augmented (recommended)**: Include the `task` parameter for async execution
- **Direct**: Omit the `task` parameter for synchronous execution

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
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
# If stateful mode is enabled, include: -H "Mcp-Session-Id: YOUR_SESSION_ID"
```

Response (task submitted):
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
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
}
```

Optional: Provide an url visiting intent for additional context (recommended but not required):

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
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
  }'
```

Recommendation: Use `url_scanner_scan_with_intent` when you can state your purpose (login, purchase, booking, payments, file download) so intent/content mismatch can be considered as an additional signal. Otherwise use `url_scanner_scan`.
Max intent length: 248 characters.
Low-information or instruction-like intent strings are treated as not provided.
Result includes `intent_alignment` (`misaligned`, `no_mismatch_detected`, `inconclusive`, or `not_provided`).
`no_mismatch_detected` is only returned when intent analysis had sufficient evidence; if intent analysis is unavailable or evidence is limited, result is `inconclusive`.
When `intent_alignment` is `misaligned` and confirmed by successful high-confidence analysis, the response directive is `DENY` with reason `intent_inconsistent_destination` (policy gate; risk score is unchanged).
When high-confidence analysis confirms an unverified high-impact service claim with weak identity corroboration in a low-confidence context, the response directive is also `DENY` with reason `insufficient_service_verification` (policy gate; risk score is unchanged).
In additional contextual low-evidence policy cases, responses may return `DENY` with reasons such as `insufficient_service_verification` or `insufficient_trust_signals` (policy gate; risk score is unchanged).

Direct-call timeout note: synchronous tool calls use a bounded server wait window (hosted default 100s). If timeout is reached, the server returns JSON-RPC `-32603` with `error.data.taskId` and `error.data.pollInterval` so you can continue via `tasks/get` / `tasks/result`.

### 4. Poll for Results

```bash
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tasks/result",
    "params": {
      "taskId": "550e8400-e29b-41d4-a716-446655440000"
    }
  }'
# If stateful mode is enabled, include: -H "Mcp-Session-Id: YOUR_SESSION_ID"
```

Response (completed task — CallToolResult shape, same as synchronous `tools/call`):
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"risk_score\":0.05,\"confidence\":0.95,\"analysis_complete\":true,\"agent_access_directive\":\"ALLOW\",\"agent_access_reason\":\"no_immediate_risk_detected\",\"intent_alignment\":\"not_provided\"}"
      }
    ],
    "isError": false
  }
}
```

---

## Available Tools

| Tool                           | Description                              | Execution Modes             |
|--------------------------------|------------------------------------------|-----------------------------|
| `url_scanner_scan`             | Analyze URL for security threats         | Direct (sync), Task (async) |
| `url_scanner_scan_with_intent` | Analyze URL with optional intent context | Direct (sync), Task (async) |

See [Full API Documentation](docs/API.md) for detailed schemas and examples.

---

## Authentication

Authentication requirements depend on deployment mode:

- Hosted endpoint (`https://preclick.ai/mcp`): API key is optional for up to 100 requests/day.
- Hosted endpoint above trial quota: API key required.

See [Authentication Guide](docs/AUTHENTICATION.md) for details on getting API keys.

---

## Technical Specifications

| Property          | Value                      |
|-------------------|----------------------------|
| Registry ID       | `ai.preclick/preclick-mcp` |
| MCP Spec          | 2025-06-18                 |
| Client Protocol   | 2025-06-18                 |
| Transport         | Streamable HTTP            |
| Endpoint          | `https://preclick.ai/mcp`  |
| Typical Scan Time | Varies by target           |
| Supported Schemes | HTTP, HTTPS                |
| Max URL Length    | Enforced by server         |

---

## Support

- **Publisher**: [CybrLab.ai](https://cybrlab.ai)
- **Service**: [PreClick](https://preclick.ai)
- **Email**: contact@cybrlab.ai

---

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

Copyright CybrLab.ai
