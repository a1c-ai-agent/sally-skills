# `metabolic_overview` — your CGM, scored and explained

**$0.005 / call.** Pull a complete metabolic snapshot for any date —
glucose wellness score, time in range, variability, postprandial
spikes, baseline / average trends — each with a clinical scoring
band ("EXCELLENT CONTROL" / "Optimal" / "Needs Attention") *and* a
short LLM-authored explanation.

This is the same engine that powers Sally's CGM detail screen in the
iOS app. Built for "how am I doing?" questions where you want both
the numbers and the read-out in one call.

---

## What you get back

```ts
{
  date:    "2026-04-24",   // the day this overview covers
  status:  "success",

  gws: {
    value:           100,
    category:        "EXCELLENT CONTROL",
    agent_response:  "Your Glucose Wellness Score has reached excellent control today, reflecting tight regulation across all axes…"
  },

  whats_driving: {
    time_in_range:        { value: 100,  category: "Optimal", agent_response: "…" },
    coefficient_variation:{ value: 2.18, category: "Optimal", agent_response: "…" },
    spikes:               { value: 0,    category: "Optimal", agent_response: "…" }
  },

  trends: {
    glucose_baseline: { value: 84.67, category: "Optimal", agent_response: "…" },
    glucose_average:  { value: 86.31, category: "Optimal", agent_response: "…" }
  },

  recommendations: "- **Structured Self-Monitoring Protocol**: Continue your weekly CGM review cadence…\n- **Sleep + Glucose**: Your TIR aligned with your best sleep nights…"
}
```

Every metric carries three things: the **value** (the number),
the **category** (clinical band), and the **agent_response** (Sally's
short prose explaining it in context).

---

## The scoring bands (verbatim from Sally's clinical engine)

| Metric | Bands |
|---|---|
| **GWS** (composite 0-100) | ≥85 → `EXCELLENT CONTROL` · 70-85 → `VERY GOOD CONTROL` · 55-70 → `GOOD CONTROL` · 40-55 → `FAIR CONTROL` · <40 → `POOR / CRITICAL CONTROL` |
| **Baseline glucose** (mg/dL) | 70-91 → `Optimal` · 91-105 → `Borderline` · ≥105 → `Elevated` |
| **Average glucose** (mg/dL) | <106 → `Optimal` · 106-118 → `Good` · 118-138 → `Borderline` · ≥138 → `Elevated` |
| **TIR** (% of day in optimal zone) | ≥90 → `Optimal` · 70-90 → `Good` · <70 → `Needs Attention` |
| **CV** (variability %) | <17 → `Optimal` · 17-25 → `Good` · 25-36 → `Fair` · ≥36 → `Needs Attention` |
| **Spikes** | severe>0 OR high>1 OR moderate>3 → `Needs Attention` · high>0 OR moderate 1–3 → `Good` · else → `Optimal` |

These bands match the ones used internally for Sally's clinical
flows — agents can treat the category strings as stable identifiers
and not just human-friendly labels.

---

## What you can ask your agent

The skill picks up automatically on these:

- *"How's my CGM today?"*
- *"Give me a metabolic overview for yesterday"*
- *"What does my glucose look like for last Friday?"*
- *"Compare my GWS this week to last week"* — agent makes 7 calls
- *"What's driving my time-in-range this week?"* — single call,
  `whats_driving` block has the answer

For broader "how am I doing" questions that include sleep + activity
+ CGM, prefer [`health_insights`](health-insights.md). Use
`metabolic_overview` when CGM is the focus.

---

## Inputs

```ts
{
  date?:     'YYYY-MM-DD',   // default: today UTC
  timezone?: string          // IANA name, e.g. "America/New_York". Default: UTC.
}
```

That's it — no CGM values from the agent. The skill pulls everything
from `insight.daily_blood_glucose` server-side, computes the scoring
bands using the same logic Sally's iOS app uses, and runs a 7-day
weekly trend fetch under the hood for the LLM context.

---

## Privacy

- **Identity-from-key only.** No `user_uuid` accepted in the request
  body — your bearer is the identity.
- **No data forgery possible.** Every CGM value comes from your own
  `insight.daily_blood_glucose` row. The agent can't claim a fake
  `gws=100` because the score is computed by the gateway, not
  supplied.
- **Server-side scoring.** The bands above are applied by Sally's
  gateway, not the LLM. The LLM only writes the `agent_response`
  prose, not the categorisation.
- **Allowlist response**. Every field returned is named above. Sensor
  IDs, raw 5-min readings, internal Milvus traces — none of those
  leak.
- **Audit log.** One row per call at `console.a1c.io` → Usage.

---

## When the skill returns 404

If you call `metabolic_overview` for a date with no CGM data on
record, the skill returns:

```json
{
  "ok": false,
  "skill": "metabolic_overview",
  "error": {
    "code": "not_found",
    "message": "no CGM data for 2026-05-09. Sync your CGM via the insight-app or pick a different date."
  }
}
```

This is *before* any LLM call — your wallet isn't charged. Pick a
date with synced data, or sync your CGM first.

---

## How to call it

### Via your agent

```
"How's my CGM today?"
"Pull a metabolic overview for last Friday."
```

The LLM picks the skill, fills in `date` from the time context.

### Direct REST

```bash
# Today (UTC)
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{ "skill":"metabolic_overview", "input":{} }' | jq '.data | {date, gws, whats_driving, trends}'

# Specific date
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{ "skill":"metabolic_overview", "input":{ "date":"2026-04-24" } }'

# With your timezone (affects postprandial timing in the LLM context)
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{
        "skill":"metabolic_overview",
        "input":{ "date":"2026-04-24", "timezone":"Asia/Jakarta" }
      }'
```

---

## Limits & timing

- **One date per call.** Multi-day comparisons are the agent's job —
  it will fan out.
- **Latency**: 10–20 s typical (LLM chain + 7-day weekly fetch). Set
  client timeout to 60 s.
- **Streaming**: not supported. Single JSON response.
- **Date range**: any date your iOS app has synced CGM for. Beyond
  ~90 days back the weekly-trend context thins out and the LLM has
  less to draw from.

---

## See also

- [Main user guide](../README.md)
- [`health_sync`](health-sync.md) — same CGM data in raw form, FREE,
  if you'd rather have your agent reason from the numbers directly
- [`health_insights`](health-insights.md) — broader "how am I doing"
  including sleep + activity + environment, not just CGM
- [`chat_with_sally`](chat-with-sally.md) — for "what does this score
  mean?" follow-ups grounded in Sally's knowledge base
- [MCP setup](../protocols/mcp.md) · [REST API](../protocols/api.md)
