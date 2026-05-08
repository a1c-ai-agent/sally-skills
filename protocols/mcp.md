# Connect via MCP (Model Context Protocol)

MCP is the standard plug-in protocol for AI agents — Anthropic launched it
late 2024, and most major agents now support it. With one config block,
your agent discovers all six Sally skills and exposes them to its underlying
LLM as native tools.

> If your agent supports MCP, **always use this path**. The agent's LLM
> picks the right Sally tool on its own; you don't wire each skill manually.

---

## The universal config

Almost every MCP-aware agent reads a JSON config of this shape (the file
location and outer wrapper change per agent — see the [agents/](../agents/)
folder for exact paths):

```json
{
  "mcpServers": {
    "sally": {
      "url": "https://sally.a1c.io/mcp",
      "headers": {
        "Authorization": "Bearer sk-sally-YOUR-KEY-HERE"
      }
    }
  }
}
```

That's it. After saving the file and restarting your agent:

1. The agent calls `tools/list` on `https://sally.a1c.io/mcp`.
2. Sally returns the six live skills with their full input schemas.
3. The agent's LLM picks tools on its own when the user asks something
   relevant ("how was my sleep?", "analyse this lab"...).

---

## What the agent's LLM sees

After connection, your agent's LLM has these six tools available:

| Tool name | Use when… |
|---|---|
| `health_sync` | User wants to share their wearable / CGM / sleep / vitals data with the agent for context. |
| `chat_with_sally` | User asks a preventive-health or TCM question that Sally is specifically trained on. |
| `analyze_lab_result` | User uploads a lab PDF or image and asks for interpretation. |
| `food_journal` | User uploads a meal photo. |
| `health_insights` | User asks for a morning / afternoon / evening readout. |
| `metabolic_overview` | User asks "how was my glucose today/yesterday/last week?". |

The agent's LLM gets the full Zod-derived JSON Schema for each tool's input,
so it knows exactly which arguments to pass.

---

## Verifying the MCP connection

Once configured, ask your agent something simple to test:

> "What Sally tools do you have?"
> "Call `health_sync` with no arguments."

A working agent will respond with the tool list or the FREE `health_sync`
result. If you get errors, see the **Troubleshooting** section below.

You can also test directly with `curl` (skips the agent runtime):

```bash
# List available skills
curl -sS https://sally.a1c.io/mcp \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq '.result.tools[] | {name, description}'

# Call a tool manually
curl -sS https://sally.a1c.io/mcp \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"health_sync","arguments":{}}}' | jq
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Tools don't appear in your agent | Config file path wrong, or agent wasn't restarted | Confirm path per [agents/](../agents/), restart fully |
| `401 unauthorized` | Bad / revoked key | Re-issue at <https://platform.a1c.io> |
| `402 payment_required` | Wallet balance < skill price | Top up at platform.a1c.io (or wait for Stripe in phase 4) |
| `404 not_found` (specific to a skill, not the URL) | No data for that user/date (e.g. metabolic_overview needs CGM data; lab analysis needs a PDF) | Use a date with synced data, or supply the input the skill needs |
| `429 rate_limited` | Per-key per-skill bucket exhausted | Wait — defaults are generous, raise it by emailing the team |
| Tool call hangs | Sally's upstream LLM is slow on big inputs | Lab analysis can take 15-30s, food journal ~10s, metabolic_overview ~15s. Increase your agent's tool timeout if it's enforcing 10s |

---

## Notes on MCP transport

- Sally implements **Streamable HTTP MCP** (the JSON-RPC 2.0 over POST
  variant). All tool calls return a single result, not a stream.
- For chat-style use, prefer `chat_with_sally` over building your own
  streaming pipeline against the gateway. We may add an SSE variant later
  if there's demand.
- Sally's MCP server announces protocol version `2024-11-05` and supports
  `initialize`, `tools/list`, `tools/call`, plus a no-op `ping`.

---

## See also

- Per-agent quick-starts: [`agents/`](../agents/)
- Raw REST (no MCP runtime): [`api.md`](api.md)
