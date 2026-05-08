# Sally + Claude Desktop

The macOS / Windows Claude Desktop app supports MCP via a config file.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here):
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://platform.a1c.io>**, generate a key.

### Step 2 — Edit the Claude Desktop config

The file lives at:

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

If the file doesn't exist yet, create it. Add this block:

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

If you already have other MCP servers configured, add `"sally"` as another
entry under `mcpServers`.

### Step 3 — Restart Claude Desktop

Fully quit the app (⌘Q on Mac, right-click → Quit on Windows) and reopen.
You should see a small tools indicator in the chat input — click it to
confirm Sally's six tools are listed.

---

## Useful prompts

```
ask Sally for a metabolic overview yesterday
analyze this lab PDF [drag PDF into the message]
what's my sleep insights this morning?
```

Claude Desktop's vision means you can drag a meal photo or lab PDF into
the message — it'll call `food_journal` or `analyze_lab_result` as needed.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Tools indicator missing | Confirm config file path + JSON validity (use a linter); fully restart the app |
| 401 on every call | Revoked key — re-issue at platform.a1c.io and update the config |
| Tool call hangs | Claude Desktop's tool timeout is 60s — lab analysis can exceed that on big PDFs. Try a smaller PDF or use Claude Code (longer timeout) |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)
