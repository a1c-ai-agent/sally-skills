# Sally + Claude Code

[Claude Code](https://docs.claude.com/en/docs/claude-code) is Anthropic's
CLI coding assistant. It speaks MCP natively. Adding Sally takes ~30 seconds.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here)
if you haven't already:
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://console.a1c.io>**, generate a key (`sk-sally-…`).

### Step 2 — Add Sally as an MCP server

Run this in your terminal — it adds Sally globally so every Claude Code
session has access:

```bash
claude mcp add sally https://sally.a1c.io/mcp \
  --transport http \
  --header "Authorization: Bearer sk-sally-…"
```

Or, if you prefer to edit the config file directly, add this block to
`~/.claude.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "sally": {
      "type": "http",
      "url": "https://sally.a1c.io/mcp",
      "headers": {
        "Authorization": "Bearer sk-sally-…"
      }
    }
  }
}
```

### Step 3 — Verify

Inside any Claude Code session:

```
> /mcp
```

You should see `sally` listed as connected. Then:

```
> ask sally to pull my health data
```

Claude will call the FREE `health_sync` tool and show your wearable + CGM
+ sleep summary inline.

---

## Useful prompts to try

```
> ask sally for a metabolic overview for yesterday
> have sally analyze the lab PDF at ~/Downloads/labs.pdf
> can sally tell me what my morning insights say
> use sally chat: what does a high HRV mean compared to my baseline?
> snap food_journal on ~/Pictures/dinner.jpg
```

Claude picks the right tool from the description on its own — no need to
name the tool explicitly unless you want to.

---

## Tips

- **Per-project keys**: if you want a different key per project, put the
  block above in `<project>/.mcp.json` instead of `~/.claude.json`.
- **Scope**: `claude mcp add` defaults to the user scope. Use `--scope local`
  or `--scope project` for narrower scopes.
- **Remove Sally**: `claude mcp remove sally`
- **List configured servers**: `claude mcp list`

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/mcp` says "sally — failed" | Check the bearer token — copy it again from console.a1c.io |
| Tools listed but every call returns 401 | Token has been revoked (creating a new key auto-revokes old ones) — re-issue and re-add |
| `payment_required` | Your wallet hit zero — top up at console.a1c.io |
| Tool calls time out | Lab analysis can take 30s. Increase your `mcpTimeout` setting in `~/.claude.json` to `120000` (2min) |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)
