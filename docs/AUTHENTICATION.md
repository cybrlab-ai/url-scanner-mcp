# Authentication Guide

> How to get and use API keys for the PreClick MCP Server

**Publisher:** [CybrLab.ai](https://cybrlab.ai) | **Service:** [PreClick](https://preclick.ai)

---

## API Key Authentication

Authentication requirements depend on deployment mode:

- Hosted endpoint (`https://preclick.ai/mcp`): API key is optional for up to 100 requests/day.
- Hosted endpoint above trial quota: API key required.

When using an API key, send it via the `X-API-Key` header.

### Request Format

```http
POST /mcp HTTP/1.1
Host: preclick.ai
Content-Type: application/json
Accept: application/json, text/event-stream
X-API-Key: your-api-key-here
MCP-Protocol-Version: 2025-06-18

{...}
```

If you are using hosted anonymous trial access, omit `X-API-Key` and stay within trial quota.

**Stateful mode only:** If `server.stateful_mode = true`, include `Mcp-Session-Id` header after `initialize`.
**Stateless mode (default):** Keep the POST `Accept: application/json, text/event-stream` header. Some clients may also probe `GET /mcp`; the server will return `405 Method Not Allowed` unless stateful mode is enabled.

### curl Example (Initialize Session - stateful mode only)

```bash
# First request: initialize (no session ID needed yet)
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key-here" \
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
# Response includes Mcp-Session-Id header - save it!
```

### curl Example (Subsequent Requests - stateful mode only)

```bash
# All non-initialize requests require protocol version; session ID is required only in stateful mode
curl -X POST https://preclick.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key-here" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "Mcp-Session-Id: your-session-id" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tasks/list",
    "params": {}
  }'
```

---

## Obtaining API Keys

### For Production Use

Contact CybrLab.ai to get production API keys:

- **Email**: contact@cybrlab.ai
- **Website**: [CybrLab.ai](https://cybrlab.ai)

### For Testing/Evaluation

Contact us for evaluation keys with limited quotas for testing purposes.

### Fastest Path

- Start immediately with hosted trial access (no key) for up to 100 requests/day.
- Request an API key at contact@cybrlab.ai for higher limits and stable authenticated quotas.

---

## Authentication Errors

Authentication failures return **plain HTTP responses** (not JSON-RPC). The
response body is empty.

| HTTP Status | Response Type | Body | Description                                     |
|-------------|---------------|------|-------------------------------------------------|
| 401         | Empty body    | n/a  | Missing/invalid/expired API key (when required) |
| 403         | Empty body    | n/a  | Origin not permitted                            |
| 429         | Empty body    | n/a  | Transport rate limited                          |

### Error Response Example

**Missing API Key (401):**
```
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

```

> **Note:** Per-key task quota errors return JSON-RPC error code `-32029`. Transport-level rate limits return HTTP 429 without a JSON-RPC body. See the [API Reference](./API.md#rate-limits--quotas) for details.

---

## Support

For API key issues or quota increases:

- **Publisher**: [CybrLab.ai](https://cybrlab.ai)
- **Email**: contact@cybrlab.ai
