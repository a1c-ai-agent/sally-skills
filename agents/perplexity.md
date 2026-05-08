# Sally + Perplexity

[Perplexity](https://www.perplexity.ai) supports MCP servers in **Pro**
and **Enterprise** tiers. The integration is identical to other MCP-aware
agents — one config block, restart, done.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here):
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://platform.a1c.io>**, generate a key.

### Step 2 — Add Sally to Perplexity

In the Perplexity desktop app or web UI:

1. Open **Settings** → **Connectors** → **MCP servers** (Pro or Enterprise
   tier required).
2. Click **Add server**.
3. Fill in:
   - **Name**: `Sally`
   - **Transport**: `HTTP (Streamable)`
   - **URL**: `https://sally.a1c.io/mcp`
   - **Headers**:
     - `Authorization`: `Bearer sk-sally-…`
4. Save.

Perplexity validates the connection by calling `tools/list`. You should see
"Connected ✓" with the six Sally tools listed.

### Step 3 — Enable Sally for your queries

Use the **@** mentions or tool toggle in the chat input:

```
@Sally what's my metabolic overview today?
```

Or just ask in plain English — Perplexity's reasoning model will pick
Sally tools when relevant:

```
analyze the lab PDF I just uploaded
how was my sleep last night?
how does my CGM compare to last week?
```

---

## What Perplexity adds on top

Because Perplexity also has live web search, you can chain queries:

```
look up the latest 2026 ADA guidelines for HbA1c, then have @Sally
analyze my lab PDF against those targets
```

This works because Perplexity sees both its own search tools and Sally's
six skills as available — its router picks each at the right moment.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| MCP server option missing in Settings | You need Pro or Enterprise — free tier doesn't support custom MCP |
| Connection fails with TLS error | Perplexity validates the cert; sally.a1c.io is publicly trusted, so this is rare. Update the Perplexity app |
| Tool calls return 401 | Header must be `Authorization: Bearer sk-sally-…` (capitalisation matters) |
| Slow first call after idle | First request after a quiet period can take ~5s due to cold connection — subsequent calls are fast |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)
