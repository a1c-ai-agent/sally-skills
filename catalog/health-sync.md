# `health_sync` — your body's data, in your agent's hands

**FREE.** One call, your last 30 days of wearable, CGM, sleep, vitals,
activity, and environment data — already curated and ready for an LLM
to reason over.

Use it when you want your agent to *understand what your body's been
doing* before it gives advice. It's the on-ramp to every other skill.

```
"Pull my health data, then tell me how my recovery's been trending."
```

The agent calls `health_sync` and gets a tidy JSON snapshot back. No
GCS uploads, no manual entry — it pulls straight from the data your
A1C Insights iOS app collects from HealthKit, your CGM, and your
environment.

---

## What you get — 64 daily measurements

Six categories. Numbers below are how many distinct fields the skill
returns *per day* in raw mode.

### 🏃 Vitals (16 fields)

| Field | What it means | Typical units |
|---|---|---|
| Body energy | Sally's body-energy score | 0–100 |
| Resting heart rate | RHR at rest | bpm |
| Heart rate variability | HRV (autonomic balance) | ms |
| VO₂ max | Cardiovascular fitness | ml/kg/min |
| SpO₂ | Blood oxygen saturation | % |
| Recovery score | How recovered you are | 0–100 |
| Readiness score | How ready you are for load | 0–100 |
| Respiratory rate | Breaths per minute | breaths/min |
| Skin temperature | Deviation from baseline | °C |
| Strain score | Cumulative training load | scaled |
| Sleep score | Last night's sleep quality | 0–100 |
| Vitals score | Composite cardio-vital score | 0–100 |
| Sleep efficiency | Time asleep ÷ time in bed | % |
| Steps | Today's steps | count |
| Sleep debt | Accumulated sleep deficit | minutes |
| Total time asleep | Last night | minutes |

### 🩸 CGM glucose (12 fields)

| Field | What it means | Typical units |
|---|---|---|
| Mean glucose value | Today's average | mg/dL |
| Coefficient of variation | Glucose stability | % |
| Time in range | % of day in healthy zone | % |
| Estimated HbA1c | 90-day average estimate | % |
| Number of spikes | Today's glucose spikes | count |
| Min glucose | Today's lowest | mg/dL |
| Max glucose | Today's highest | mg/dL |
| Glucose Wellness Score | Sally's composite CGM grade | 0–100 |
| Low blood glucose index | LBGI risk score | scaled |
| High blood glucose index | HBGI risk score | scaled |
| Baseline glucose value | Fasting baseline | mg/dL |
| MAGE | Mean amplitude of glycemic excursions | mg/dL |

### ⏱️ Time in / above / below range (8 fields)

The clinical gold-standard CGM breakdown — what % of the day was
in the optimal zone, suboptimal, or risky.

| Field | What it captures |
|---|---|
| Optimal range % | Time spent in 70–140 mg/dL (or your personal target) |
| Suboptimal range % | Time spent slightly out of optimal |
| Time below range — level 1 | Mild hypoglycemia (54–69 mg/dL), as % of day |
| Time below range — severe | Severe hypo (<54 mg/dL), as % of day |
| Total time below range | All TBR combined |
| Time above range — level 1 | Mild hyperglycemia (180–250 mg/dL), as % of day |
| Time above range — severe | Severe hyper (>250 mg/dL), as % of day |
| Total time above range | All TAR combined |

### 😴 Sleep architecture (19 fields)

The deep-detail sleep block. Anything you'd find on an Oura / Whoop /
8sleep export, plus Sally's per-axis quality scores.

**Stage durations (minutes)**: REM, deep, light, awake during sleep,
total time in bed, total sleep duration.

**Personal benchmarks**: personal sleep baseline, sleep debt recovery
(how much debt last night closed).

**Quality axes (each 0–100)**: respiration, sleep architecture,
continuity, sleep quality, sleep quantity, circadian rhythm, HRV
during sleep, heart-rate dip, oxygenation, autonomic balance.

**Behavior**: number of awakenings.

### 🏋️ Activity (record stream)

Each row carries an activity name (e.g. "running", "step"), a numeric
value, and the source (e.g. "apple.watch"). Your agent gets the full
sequence of activity events for the window — useful when you want to
ask things like "did my heavy training block hurt my sleep?".

### 🌤️ Environment (8 fields)

Often overlooked, but matters for sleep, mood, and vitamin D.

| Field | What it means |
|---|---|
| Time in daylight | Minutes outdoors with effective UV exposure |
| AQI | Numeric Air Quality Index |
| AQI level | Word category — Excellent / Good / Moderate / Low / Poor / Hazardous |
| UV index | Numeric UV reading |
| UV index level | Low / Moderate / High / Very High / Extreme |
| Vitamin D % RDA | Estimated % of daily vitamin D produced from today's sun |
| Vitamin D adequacy | Word category for the % above |
| Solar zenith efficiency | How effective today's sun was for vitamin D production |

---

## What you can ask your agent once it's connected

These all work because the agent's LLM picks `health_sync` automatically
when the question needs the data:

- *"How's my recovery been trending this week?"*
- *"Did my workout yesterday hurt my sleep architecture?"*
- *"What's my time-in-range looking like, and which days were worst?"*
- *"Pull my health data and write me a one-paragraph summary."*
- *"Compare my HRV from this week to last week."*
- *"Was I in enough daylight for vitamin D yesterday?"*

The skill is FREE, so feel free to call it as the *first* step in any
broader question — your agent's reasoning gets dramatically better when
it's grounded in your actual numbers instead of guessing.

---

## Two output modes

When the agent calls `health_sync`, it can ask for one of two shapes
(no extra cost either way):

### Raw daily series (default)

Each of the six categories above comes back as a list of per-day
records, sliced by your date range (defaults to the last 30 days,
capped at 90). Best for "show me the trend" questions.

### Aggregated

Pass `aggregate: true` and `health_sync` reshapes the data into the
exact format Sally's own agent uses internally — a `morning_insight`
block (latest sleep + vitals + environment with Sally's scoring) and
a `metabolic_overview` block (latest CGM with band labels like
"EXCELLENT CONTROL", "Optimal", "Needs Attention"). Best when you want
the agent to answer one specific question grounded in today's numbers.

---

## Privacy & data control

- **Your key is your identity.** The skill only ever returns *your*
  data — the user the API key is bound to. No agent can ask for
  someone else's numbers, even if it tries.
- **Allowlist only.** The 64 fields above are *the only thing*
  `health_sync` returns. Internal IDs, audit columns, sensor
  fingerprints, deleted rows — none of those leak.
- **Aggregated, not raw.** Where it makes sense (CGM time-series,
  daily summaries) you get the daily aggregate, not the every-5-minute
  reading. If your agent needs more granularity, ask the Sally team
  about a higher-tier read.
- **Audit log.** Every call to `health_sync` writes a row to your
  usage history. You can see who called it (which key) and when at
  `platform.a1c.io` → **Usage**.

---

## How to call it

If your agent supports MCP (most do — see [`agents/`](../agents/)),
just ask the agent in plain English. The LLM picks the tool.

If you want to see it work first, the FREE 30-second smoke test is:

```bash
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"skill":"health_sync","input":{"aggregate":true}}' | jq
```

You'll see today's morning insight + metabolic overview blocks come
back. From there, an agent can do the rest.

---

## Limits & caps

- **Window**: defaults to last 30 days, max 90.
- **Activity rows**: heavy CGM users with weeks of activity logs can
  return thousands of rows in raw mode. Use `include: ["cgm","sleep"]`
  to slice to just what you need.
- **Rate limit**: per-key bucket, generous beta defaults. Email us if
  you hit it.
- **Data freshness**: as fresh as your iPhone last synced HealthKit
  back to Sally — usually seconds to minutes behind real-time.

---

## See also

- [Main user guide](../README.md) — A1C onboarding + the agent setup flow
- [MCP setup](../protocols/mcp.md) — universal config for any agent
- [REST API](../protocols/api.md) — curl recipes for direct calls
- [Per-agent guides](../agents/) — Claude Code, Claude Desktop, OpenClaw, Hermes, Manus, Perplexity
