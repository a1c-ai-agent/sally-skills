# Sally + OpenClaw

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source agent
runtime that speaks MCP. Configuration is JSON-based, similar to Claude
Desktop.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here):
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://platform.a1c.io>**, generate a key.

### Step 2 — Add Sally to your OpenClaw config

OpenClaw reads `~/.openclaw/config.json` (Linux/macOS) or
`%APPDATA%\openclaw\config.json` (Windows). Add Sally to the `mcpServers`
block:

```json
{
  "mcpServers": {
    "sally": {
      "url": "https://sally.a1c.io/mcp",
      "headers": {
        "Authorization": "Bearer sk-sally-…"
      }
    }
  }
}
```

If your OpenClaw build uses a different config path, check
`openclaw config show` or the version's docs.

### Step 3 — Restart the agent

```bash
openclaw restart
# or just kill the running process and relaunch
```

Verify Sally is connected:

```bash
openclaw mcp list
# expect: sally   https://sally.a1c.io/mcp   ✓
```

And confirm tools are visible:

```bash
openclaw tools list --server sally
# expect: chat_with_sally, health_sync, analyze_lab_result, food_journal,
#         health_insights, metabolic_overview
```

---

## Calling Sally

Simply chat with the agent — it picks the right tool:

```
$ openclaw chat
> use sally to pull my latest health data
> analyze the lab pdf I downloaded yesterday
> what does my metabolic overview look like for last Friday?
```

If you want to invoke a tool by name explicitly:

```
> /tool sally.health_sync {}
> /tool sally.metabolic_overview {"date": "2026-04-24"}
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `openclaw mcp list` doesn't show sally | Config file isn't being read — check the exact path with `openclaw config which` |
| Connection succeeds but `tools list --server sally` is empty | The bearer token is missing or wrong — copy it again from platform.a1c.io |
| Tool calls return 422 | OpenClaw may be sending unexpected fields; report the issue with the raw request |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)

---

## Notes

OpenClaw evolves quickly — if the config schema changes between versions,
check the latest docs. The Sally side is stable: same URL + bearer header,
forever.
