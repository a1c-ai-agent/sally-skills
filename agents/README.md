# Per-agent quick-starts

Each file is a self-contained 2-minute guide for one agent. Pick yours.

| Agent | File | Setup style |
|---|---|---|
| Claude Code (terminal) | [claude-code.md](claude-code.md) | `claude mcp add` CLI |
| Claude Desktop (mac/win) | [claude-desktop.md](claude-desktop.md) | JSON config file |
| OpenClaw | [openclaw.md](openclaw.md) | JSON config file |
| Hermes | [hermes.md](hermes.md) | TOML/JSON config |
| Manus | [manus.md](manus.md) | Web UI form |
| Perplexity (Pro/Enterprise) | [perplexity.md](perplexity.md) | Web UI form |

Don't see your agent? If it supports MCP (most modern ones do), use the
universal config in [`../protocols/mcp.md`](../protocols/mcp.md). If it
doesn't, fall back to the [REST API](../protocols/api.md).
