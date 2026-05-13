# Sally Skills — User Guide

Plug Sally's metabolic-health intelligence into your AI agent. Six skills,
per-call pricing, one MCP server. Works with Claude Code / Desktop, OpenClaw,
Hermes, Manus, Perplexity, or any tool that speaks REST.

> Marketing & roadmap: https://loud-evidence-895891.framer.app/beta/sally-platform
> Technical (gateway internals): https://github.com/Sally-A1C/ai-sally-skills

---

## Section 1 — Onboarding (start here)

Before your agent can call any Sally skill, **you (the human) need an A1C
account and an API key**. Two steps.

### Step 1 — Install A1C Insights and create your account

1. Open the App Store on your iPhone.
2. Search for **A1C Insights** and install it.
3. Open the app and tap **Sign Up**. You can use Sign in with Apple or
   email + password.
4. Walk through the onboarding (your basics, goals, optional wearables).
   This creates your A1C user account — every skill key is bound to one
   A1C account.

> Why iOS first? Skills like `metabolic_overview`, `health_insights`, and
> `health_sync` read CGM and wearable data the iPhone app collects via
> HealthKit. You can still use `chat_with_sally`, `analyze_lab_result`, and
> `food_journal` without any wearable data, but the account itself comes
> from the iOS app.

### Step 2 — Get your API key from `console.a1c.io`

1. Visit **<https://console.a1c.io>** in any browser.
2. Sign in with the same A1C account you created in step 1.
3. Open **API Keys** → **Create new key** → name it (e.g. *"Claude Code on my
   laptop"*) → **Generate**.
4. **Copy the key now.** It starts with `sk-sally-…` and is shown once. If
   you lose it, you'll need to revoke and re-issue.

> One active key per A1C account. Creating a new key automatically revokes
> the previous one, so plan ahead if you have multiple agents using the same
> account.

That's section 1 done. You now have:
- ✅ An A1C account (linked to your iPhone app data)
- ✅ A funded wallet ($100 free credit during beta)
- ✅ An `sk-sally-…` API key on your clipboard

---

## Section 2 — Use your key from any agent

You have one key. Three ways to call Sally with it. **Most agents only need
the MCP path** — pick that unless you're scripting from the command line.

### 2a · The skill catalog

These six skills go live the moment your key works:

| Skill | Cost / call | What it does | Detail |
|---|---|---|---|
| `health_sync` | FREE | Pull your wearable + CGM + sleep + vitals + activity + environment data into the agent's context. | [64 fields](catalog/health-sync.md) |
| `chat_with_sally` | $0.003 | Talk to Sally about preventive health & TCM with source citations. | [usage](catalog/chat-with-sally.md) |
| `analyze_lab_result` | $0.008 | Upload a lab PDF/image; get parsed biomarkers + clinical interpretation. | [usage](catalog/analyze-lab-result.md) |
| `food_journal` | $0.004 | Snap a photo of your meal; get macros + smart/trap categorisation. | [usage](catalog/food-journal.md) |
| `health_insights` | $0.003 | Morning / afternoon / evening readout from your last day's data. | [usage](catalog/health-insights.md) |
| `metabolic_overview` | $0.005 | CGM snapshot for a date — TIR, variability, spikes, narrative. | [usage](catalog/metabolic-overview.md) |

→ See [`catalog/`](catalog/) for the full per-skill data breakdown.

Roadmap (visible to your agent as `coming_soon`):
`health_report`, `metabolic_risk_score`, `supplement_grading`, `preventive_protocol`.

### 2b · Three ways to call

| Path | Best for | Setup |
|---|---|---|
| **MCP** | Any modern AI agent (Claude, Cursor, OpenClaw, Hermes, Manus, Perplexity…) | One JSON config block, restart agent, tools appear automatically |
| **REST API** | Bash scripts, your own apps, anything that can `curl` | One HTTP POST per call |
| **Per-skill REST** | Same as REST, slightly cleaner URL per skill | `/v1/skills/<name>` |

→ See [`protocols/mcp.md`](protocols/mcp.md) for the universal MCP setup.
→ See [`protocols/api.md`](protocols/api.md) for raw REST calls.
→ See [`agents/`](agents/) for per-agent quick-starts.

### 2c · Auto-routing for agents and CLIs

Want your agent to pick the right Sally skill for any request without
you having to think about it? Drop [`SKILL.md`](SKILL.md) into the
agent's system prompt or instructions field — it's a single file with
deterministic routing rules, decision tables, chaining patterns, and
anti-patterns. Works with any LLM-based agent or rule-based CLI.

→ See [`SKILL.md`](SKILL.md) — the full agent decision layer.

### 2d · The 30-second smoke test

Confirm your key works before configuring any agent:

```bash
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"skill":"health_sync","input":{}}' | jq .ok
# → true
```

`true` means your key is live and can call the FREE `health_sync` skill.
You're ready.

---

## Per-agent guides

Pick the agent you actually use:

- [Claude Code](agents/claude-code.md) — terminal coding assistant
- [Claude Desktop](agents/claude-desktop.md) — macOS / Windows app
- [OpenClaw](agents/openclaw.md) — open-source agent CLI
- [Hermes](agents/hermes.md) — agent runtime
- [Manus](agents/manus.md) — Chinese agent platform
- [Perplexity](agents/perplexity.md) — search-first AI

Don't see your agent? Most modern agents support MCP — see
[`protocols/mcp.md`](protocols/mcp.md) and use the universal config.

---

## Pricing & wallet

- **Free during beta**: every new account gets a **$100** wallet credit. No
  card required.
- **Per-call pricing** (no subscription): see the table above. `health_sync`
  is permanently free.
- **Stripe top-up** lands in phase 4 — until then, ping
  [ai@sallya1c.com](mailto:ai@sallya1c.com) if you exhaust the credit.
- **Wallet check is enforced before any call**: paid skills return
  `402 payment_required` if your balance is below the price. No surprise
  charges.
- **Audit log**: every call writes an immutable row to your usage history.
  See it in `console.a1c.io` → **Usage**.

---

## Privacy & security

- **Your key is your identity.** Sally never accepts `user_uuid`, `email`,
  or any other identifier from your agent's request body — only the
  `Authorization: Bearer sk-sally-…` header. This means no agent can
  impersonate another user.
- **Lab PDFs and meal photos are never persisted** by Sally. They flow
  agent → gateway → AI service → response, with no GCS / S3 upload along
  the way.
- **Output schemas are allowlists.** Sally's response only ever contains
  the fields explicitly named in each skill's docs. No accidental column
  leak from internal databases.
- **Revoke instantly** at `console.a1c.io` → API Keys → Revoke. Old key
  becomes invalid the next request.

---

## Help

- Public catalog (technical): https://github.com/Sally-A1C/ai-sally-skills/blob/main/SKILLS.md
- Per-skill mechanics: each skill's `packages/skill-*/README.md` in that repo
- Email: [ai@sallya1c.com](mailto:ai@sallya1c.com)
