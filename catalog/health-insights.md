# `health_insights` — your morning, afternoon, or evening readout

**$0.003 / call.** Ask Sally to write you a narrative snapshot of
the day so far — what your sleep / vitals / activity / environment
data is saying, framed for the time of day you're asking.

The same daily-readout that Sally's iOS app shows you in the morning,
afternoon, and evening, but exposed as a tool any agent can pull on
demand.

---

## Three time windows, auto-detected

The skill knows what time of day it is for you, so the framing
shifts:

| Type | Window (your local time) | Framing |
|---|---|---|
| `morning` | 05:00–11:59 | "How did you sleep? What's your readiness?" — sleep architecture, recovery, HRV, body energy |
| `afternoon` | 12:00–17:59 | "How's the day going?" — activity since morning, glucose, hydration, fatigue check |
| `evening` | 18:00–23:59 | "How was the day, and what should set up tomorrow?" — full-day vitals + activity + sleep prep |
| `auto` *(default)* | uses your timezone | Picks the right one from the three above |

Late-night calls (00:00–04:59) roll back to `evening` — the
last meaningful window before you sleep.

---

## What you get back

```ts
{
  type:        'morning' | 'afternoon' | 'evening',  // the window the readout was written for
  date:        'YYYY-MM-DD',                          // the day it covers
  response:    string,    // the full readout, markdown-formatted (~300-700 words)
  summary:     string,    // 1-2 sentence headline (~150 chars), suitable for notifications
  elapsed_ms:  number
}
```

The `response` is the agent-shareable narrative — your agent can quote
it directly or summarise it further. Example (truncated):

```markdown
**Morning readout · 2026-05-09**

Recovery is **74** today, ahead of your 14-day average of 68. HRV held
steady at 52 ms; resting HR was 56 bpm — both within your usual band.

Sleep was on the lighter side: 6h 41m total, with REM at 19% (your
target: 22%) and deep at 14% (target: 18%). The drop in REM lines up
with the tough training session you logged yesterday afternoon — that
post-9pm cortisol bump tends to push REM later.

Vitamin D contribution from yesterday's daylight was 38% RDA…
```

The `summary` is the dashboard-line:

```
"Recovery 74, REM light at 19%. HRV holding."
```

---

## What you can ask your agent

These all route to `health_insights` automatically:

- *"What's my morning readout?"* → forces `type: morning`
- *"Give me the afternoon check-in"* → `type: afternoon`
- *"How's my day going?"* → `type: auto` based on local time
- *"Read me yesterday's evening insights"* → `type: evening, date: <yesterday>`

The agent picks the type from the language. You can also be explicit
when scripting.

---

## Inputs

```ts
{
  type?:     'morning' | 'afternoon' | 'evening' | 'auto',  // default 'auto'
  date?:     'YYYY-MM-DD',                                   // default: today in your timezone
  timezone?: string,                                         // IANA name, e.g. "America/New_York". Default: UTC.
  language?: string                                          // ISO code; auto-detected from the agent's locale
}
```

The skill **never accepts** health values from your agent. Everything
your readout reasons over is fetched server-side from `insight.*`
using the user_uuid resolved from your bearer.

---

## Privacy

- **Identity-from-key only.** Sally's gateway resolves your sk-sally
  key to one A1C user. No `user_uuid` ever travels in the request.
- **Server-side payload.** The data going into the LLM (sleep stages,
  vitals, activity, environment) is read by Sally's gateway from
  `insight.*` directly. Your agent never sees the raw rows — only the
  finished narrative.
- **Allowlist response.** Five fields, named above. Internal source
  paths, raw vitals arrays, weekly-trend metadata — none of those
  leak.
- **No mem0 writes.** Unlike `chat_with_sally`, this skill doesn't
  retain anything in conversation memory. Each call is independent.

---

## How to call it

### Via your agent

```
"Read me my morning insights."
```

The LLM picks the skill and `type: morning`. Same for afternoon /
evening / "what's my day looking like?".

### Direct REST

```bash
# Auto-detect from current time
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{ "skill":"health_insights", "input":{} }' | jq '.data.summary, .data.response'

# Force a specific type and date
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{ "skill":"health_insights", "input":{ "type":"evening", "date":"2026-05-08" } }'

# Localised — Bahasa
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{ "skill":"health_insights", "input":{ "language":"id" } }'
```

---

## When `health_insights` returns "no data"

If your iOS app hasn't synced the relevant block (e.g. you're asking
for `morning` but last night's sleep didn't sync yet), you'll get a
`response` that explains the gap and a `summary` like *"Sleep data
hasn't synced yet — try again in a few minutes."*

This isn't an error per se — your agent can still use the response to
decide what to do next.

---

## Limits & timing

- **One readout per call.** If you want morning + afternoon + evening
  all in one go, your agent makes three calls (the LLM will batch
  them).
- **Latency**: 5–12 s typical. Set client timeout to 60 s.
- **Streaming**: not supported.
- **Date range**: any date your iOS app has synced. Beyond ~90 days
  back the data tends to thin out.

---

## See also

- [Main user guide](../README.md)
- [`health_sync`](health-sync.md) — the raw daily series this readout
  is summarising
- [`metabolic_overview`](metabolic-overview.md) — narrower CGM-only
  view with structured scoring (often a better partner with this
  skill)
- [`chat_with_sally`](chat-with-sally.md) — for "what does the readout
  mean?" follow-ups
- [MCP setup](../protocols/mcp.md) · [REST API](../protocols/api.md)
