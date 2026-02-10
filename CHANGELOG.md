# Changelog

## 2026-02-10

- Updated MCP registry server identifier to DNS namespace format: `ai.urlcheck/urlcheck-mcp`.
- Added explicit tool safety annotations in `server.json` for both scanner tools:
  - `readOnlyHint: true`
  - `destructiveHint: false`
  - `openWorldHint: true`
- Aligned runtime MCP metadata so `tools/list` now returns the same safety annotations for both scanner tools.
- Replaced OpenClaw-specific setup guide with Integrations section linking to downstream repos.
- Removed `docs/OPENCLAW_SETUP.md` (setup docs now live in each integration's own repository).

## 2026-02-07

- Clarified authentication model across public docs:
  - Hosted `https://urlcheck.ai/mcp`: API key optional for up to 100 requests/day.
  - Above hosted trial quota: API key required.
- Updated `server.json` remote header metadata to mark `X-API-Key` as optional for hosted trial usage.
- Added manual-first OpenClaw onboarding guide:
  - `docs/OPENCLAW_SETUP.md`
  - Linked from public `README.md` as the recommended OpenClaw setup path.

## 2026-02-05

- Added free tier notice: No API key required for up to 100 requests per day.
- Announced repository rename from `cybrlab-ai/url-scanner-mcp` to `cybrlab-ai/urlcheck-mcp`.
- Set `https://github.com/cybrlab-ai/urlcheck-mcp` as the canonical repository URL.
- Noted collaborator remote update commands:
  - `git remote set-url origin git@github.com:cybrlab-ai/urlcheck-mcp.git`
  - `git fetch origin --verbose`
- Noted that old GitHub Action `uses:` paths should be updated to `cybrlab-ai/urlcheck-mcp@...`.
- Documented that a new repo must not be created at the old name (`url-scanner-mcp`) to preserve redirects.
