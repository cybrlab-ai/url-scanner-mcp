# URLCheck MCP Server

[![smithery badge](https://smithery.ai/badge/cybrlab-ai/urlcheck-mcp)](https://smithery.ai/server/cybrlab-ai/urlcheck-mcp)

> **An MCP-native URL preflight scanning service for autonomous agents. It scans links for threats and confirms they match the intended task before execution. Built for agentic workflows, it provides high-accuracy, context-aware browsing governance with adaptive learning.**

**Publisher:** [CybrLab.ai](https://cybrlab.ai) | **Service:** [URLCheck](https://urlcheck.dev)

**Hosted Trial Tier:** No API key required for up to 100 requests/day. For higher limits and stable quotas, use an API key (contact [contact@cybrlab.ai](mailto:contact@cybrlab.ai)).

---

## Repository Rename Notice

This repository was renamed from `cybrlab-ai/url-scanner-mcp` to `cybrlab-ai/urlcheck-mcp`.

- Canonical repository URL: `https://github.com/cybrlab-ai/urlcheck-mcp`
- Canonical Git remote: `git@github.com:cybrlab-ai/urlcheck-mcp.git`
- GitHub Action consumers must update `uses:` references to `cybrlab-ai/urlcheck-mcp@...` (old action paths are not guaranteed to redirect)
- Collaborators should update local remotes:
  - `git remote set-url origin git@github.com:cybrlab-ai/urlcheck-mcp.git`
  - `git fetch origin --verbose`

Do not create a new repository at the old name (`url-scanner-mcp`) to avoid breaking GitHub redirect behavior.

---

## Overview

URLCheck is an MCP server that enables AI agents and any MCP-compatible client to analyze URLs for malicious content and security threats before navigation.

## Integrations

URLCheck works with any MCP-compatible client. For framework-specific adapters:

| Integration           | Repository                                                             |
|-----------------------|------------------------------------------------------------------------|
| LangChain / LangGraph | [langchain-urlcheck](https://github.com/cybrlab-ai/langchain-urlcheck) |
| OpenClaw plugin       | [urlcheck-openclaw](https://github.com/cybrlab-ai/urlcheck-openclaw)   |

For manual MCP bridge configuration (any client), see [Quick Start](#quick-start) below.

## Authentication Modes

| Deployment                         | `X-API-Key` Requirement         | Notes                                 |
|------------------------------------|---------------------------------|---------------------------------------|
| Hosted (`https://urlcheck.ai/mcp`) | Optional up to 100 requests/day | API key recommended for higher limits |
| Hosted (`https://urlcheck.ai/mcp`) | Required above trial quota      | Contact support for provisioned keys  |

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
    "urlcheck-mcp": {
      "transport": "streamable-http",
      "url": "https://urlcheck.ai/mcp"
    }
  }
}
```

Authenticated (recommended for stable and higher-volume usage):

```json
{
  "mcpServers": {
    "urlcheck-mcp": {
      "transport": "streamable-http",
      "url": "https://urlcheck.ai/mcp",
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
curl -X POST https://urlcheck.ai/mcp \
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
curl -X POST https://urlcheck.ai/mcp \
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
curl -X POST https://urlcheck.ai/mcp \
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
Result includes `intent_alignment` (`misaligned`, `no_mismatch_detected`, `inconclusive`, or `not_provided`).

Direct-call timeout note: synchronous tool calls use a bounded server wait window (hosted default 100s). If timeout is reached, the server returns JSON-RPC `-32603` with `error.data.taskId` and `error.data.pollInterval` so you can continue via `tasks/get` / `tasks/result`.

### 4. Poll for Results

```bash
curl -X POST https://urlcheck.ai/mcp \
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

Response (completed task with agent directive):
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "contentType": "application/json",
    "value": {
      "risk_score": 0.05,
      "confidence": 0.95,
      "analysis_complete": true,
      "agent_access_directive": "ALLOW",
      "agent_access_reason": "clean",
      "intent_alignment": "not_provided"
    },
    "summary": "URL scan completed"
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

- Hosted endpoint (`https://urlcheck.ai/mcp`): API key is optional for up to 100 requests/day.
- Hosted endpoint above trial quota: API key required.

See [Authentication Guide](docs/AUTHENTICATION.md) for details on getting API keys.

---

## Technical Specifications

| Property          | Value                      |
|-------------------|----------------------------|
| Registry ID       | `ai.urlcheck/urlcheck-mcp` |
| MCP Spec          | 2025-06-18                 |
| Client Protocol   | 2025-06-18                 |
| Transport         | Streamable HTTP            |
| Endpoint          | `https://urlcheck.ai/mcp`  |
| Typical Scan Time | Varies by target           |
| Supported Schemes | HTTP, HTTPS                |
| Max URL Length    | Enforced by server         |

---

## Support

- **Publisher**: [CybrLab.ai](https://cybrlab.ai)
- **Service**: [URLCheck](https://urlcheck.dev)
- **Email**: contact@cybrlab.ai

---

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

Copyright CybrLab.ai
