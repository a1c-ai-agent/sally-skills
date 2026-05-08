# Sally + Hermes

[Hermes](https://github.com/HermesProject/hermes) is an MCP-aware agent
runtime designed for autonomous workflows. Sally connects via the standard
MCP HTTP transport.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here):
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://platform.a1c.io>**, generate a key.

### Step 2 — Register Sally with Hermes

Most Hermes deployments read MCP servers from `~/.hermes/mcp.toml` or the
project's `hermes.toml`:

```toml
[mcp.sally]
url = "https://sally.a1c.io/mcp"
[mcp.sally.headers]
Authorization = "Bearer sk-sally-…"
```

If your version uses JSON instead:

```json
{
  "mcpServers": {
    "sally": {
      "url": "https://sally.a1c.io/mcp",
      "headers": { "Authorization": "Bearer sk-sally-…" }
    }
  }
}
```

### Step 3 — Reload Hermes

```bash
hermes reload
# or restart the daemon: hermes restart
```

Confirm Sally is up:

```bash
hermes mcp ls
hermes tools list --provider sally
```

---

## Using Sally in a Hermes workflow

Hermes is workflow-oriented, so you'll typically invoke Sally inside a
plan step. Example workflow snippet (TOML):

```toml
[workflow.morning_check]
description = "Daily metabolic check-in"

[[workflow.morning_check.steps]]
tool = "sally.health_insights"
args = { type = "auto" }

[[workflow.morning_check.steps]]
tool = "sally.metabolic_overview"
args = {}

[[workflow.morning_check.steps]]
tool = "sally.chat_with_sally"
args = { message = "Based on the above, what should I focus on today?" }
```

Run it:

```bash
hermes run morning_check
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `hermes mcp ls` says sally is "disconnected" | Hermes proxies all MCP traffic — check egress rules; sally.a1c.io must be reachable |
| Tools listed but invocations 401 | Bearer token typo or revoked — reissue at platform.a1c.io |
| Workflow step times out | Increase per-step timeout: `[workflow.…steps].timeout_seconds = 90` |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)
