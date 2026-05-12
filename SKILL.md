# SKILL.md — Sally skills, auto-routing for CLIs and agents

> **For agents and CLIs.** This file tells an LLM (or a routing layer
> in front of one) which Sally skill to invoke for any given user
> request. Drop it into an agent's system prompt, MCP `instructions`
> field, or a CLI's tool-router, and the agent will pick the right
> skill the first time without trial-and-error.
>
> Per-skill mechanics live in [`catalog/`](catalog/). This file is
> the **decision layer** above them.

---

## TL;DR — minimum agent insert

The full doc is ~430 lines, but you only need to paste **this section**
(~70 lines) into an agent's system prompt for ~95% of requests to route
correctly. Everything below it is edge-case reference for the routing
layer's authors.

```text
You have access to Sally's metabolic-health skills. Everything starts with
DATA — call health_sync first when grounding will help; reach for the paid
skills only after you've seen the numbers. Pick exactly ONE per turn:

  raw numbers / trends / sync          → health_sync (default)     (FREE — 64 daily biomarkers across vitals/CGM/sleep/TIR/activity/environment)
  glucose during/after a single event  → health_sync (cgm_minute)  (FREE)    ← raw intra-day curve
  "how's my CGM/glucose <date>"        → metabolic_overview        ($0.005)  ← narrated grade
  "morning|afternoon|evening readout"  → health_insights           ($0.003)
  "my X based on Y" (cross-source)     → chat_with_sally health:true ($0.003)
  general health/TCM/nutrition Q&A     → chat_with_sally           ($0.003)
  attached meal photo                  → food_journal              ($0.004)
  attached PDF / lab image             → analyze_lab_result        ($0.008)

Tiebreakers (the cases an LLM router gets wrong most often):

  "how's"/"grade"/"score"              → metabolic_overview
  "show me"/"plot"/"what did"/"during" → health_sync cgm_minute
  pronoun ("my"/"I"/"me") + a metric   → chat_with_sally health:true (NOT generic chat)
  mixed attachments (lab + meal)       → ASK which to process first; do NOT silently chain
  "no live equivalent" (coming_soon)   → surface roadmap; do NOT substitute another paid skill

Always:

  - One sk-sally-… key per A1C user, NO exceptions. If your agent serves N humans, mint N keys.
  - Never pass user_uuid / email / id in the body — the bearer header is the identity.
  - Never retry on 401 (bad key) or 402 (top-up needed). Surface and stop.
  - On 503 / `service_unavailable`: retry up to 2× with backoff (1s, 4s).
  - For cgm_minute: resolve the user's IANA timezone *before* constructing ISO timestamps.
  - For analyze_lab_result: redact PII (DOB, MRN, address, SSN) from `ocr_text` before
    showing the analysis to the end user. Sally returns the raw text on purpose.
  - health_sync is FREE; call it before any paid skill when health context would improve the answer.

Envelope:
  success → { ok: true,  data: {...}, usage: { cost_usd } }
  failure → { ok: false, error: { code, message } }
```

---

## How to use this file

- **Agents that support MCP** (Claude Code/Desktop, Cursor, Hermes,
  OpenClaw, Manus, …): paste the *Routing matrix* and *Decision rules*
  sections into the agent's system prompt or instructions field.
- **CLIs and routers**: implement the *Decision rules* as a deterministic
  pre-LLM matcher (regex / keyword score) and only fall through to the
  LLM router when no rule fires.
- **End-user docs / READMEs**: the *Quick lookup* one-liners are the
  same value-props you can show users; mirror them in your UI.

The agent calling these skills authenticates **once** with one bearer:
`Authorization: Bearer sk-sally-…`. There is no per-skill credential.
The same key resolves the user's identity for every skill.

> **One key = one A1C user. No exceptions.** If your agent serves multiple
> humans (a SaaS product, a multi-tenant platform, a shared family device),
> each end-user must have their own A1C account and their own
> `sk-sally-…` key. Sharing a key across users would mix CGM data, mem0
> memory, lab history, and usage billing — there is no `user_uuid`
> override that lets you pretend otherwise. Mint N keys for N humans.

---

## Quick lookup

| User intent | Skill | Cost |
|---|---|---|
| "Pull / sync / show my health data" *(64 daily biomarkers across 6 domains)* | `health_sync` | FREE |
| "How did my glucose respond to X?" *(intra-day)* | `health_sync` *(opt-in `cgm_minute`)* | FREE |
| "Morning / afternoon / evening readout" | `health_insights` | $0.003 |
| "How's my CGM today / on date X?" | `metabolic_overview` | $0.005 |
| "What's in this meal?" *(photo)* | `food_journal` | $0.004 |
| "Read / interpret this lab report" *(PDF/image)* | `analyze_lab_result` | $0.008 |
| Knowledge / TCM / preventive-health Q&A | `chat_with_sally` | $0.003 |
| Personalised health Q&A across all data | `chat_with_sally` *(`health: true`)* | $0.003 |

---

## Decision rules (deterministic, run before LLM routing)

Apply these in order. **First match wins.** The router should never
call more than one skill from a single user turn unless the rules
explicitly say to chain.

### Rule 1 — Attached PDF / lab image → `analyze_lab_result`

Triggers:
- The user attaches a PDF, a photo of a printed lab sheet, or an image
  containing tabular blood-test data.
- Keywords: "lab", "blood test", "panel", "results", "biomarker",
  "HbA1c report", "cholesterol panel", "thyroid panel", "CBC", "CMP".

Required input: `pdf_b64` (the agent must base64-encode the file before
calling). Most agent runtimes handle attachments → base64 transparently.

Do **not** route here for: meal photos (→ `food_journal`), screenshots
of CGM apps (→ `metabolic_overview`), text descriptions of past results
(→ `chat_with_sally` with `health: true`).

**Mixed attachments (lab PDF + meal photo in the same turn).** If the
user uploads both in one message, **do not silently chain** — surface a
clarifying question ("which should I look at first — the lab report or
the meal?") and process only the one they confirm. Each is a paid call
($0.008 + $0.004) and the user usually only cares about one. Silent
chaining double-bills.

**PII in lab PDFs.** Reports routinely contain DOB, address, MRN, and
sometimes SSN. Sally never persists the bytes (no GCS / S3 / Milvus
upload), and the response is allowlist-scoped — but the `ocr_text`
field returns the *raw* extracted text on purpose, so your agent can
do its own redaction pass before showing the analysis to the end user
or writing it to disk. That redaction is **your agent's
responsibility**, not Sally's.

### Rule 2 — Attached meal photo → `food_journal`

Triggers:
- The user attaches a photo of food, a beverage, or a plate.
- Keywords: "meal", "lunch", "breakfast", "dinner", "snack",
  "what's in this", "smart food", "trap food", "should I eat", "macros".

Required input: `image_b64`.

If the photo is *not* food, the skill returns `status: "not_food"` —
the agent should surface that message and not retry with another skill.

Chain pattern: after a meal photo, the user often asks "and how will
this affect my glucose?" — that's a follow-up `health_sync` (with
`cgm_minute` for the prior comparable meal) plus an LLM synthesis
turn. Don't pre-emptively chain unless the user explicitly asks.

### Rule 3 — Time-of-day readout phrasing → `health_insights`

Triggers (case-insensitive substring):
- "morning readout", "morning insight", "morning check-in"
- "afternoon readout", "afternoon check-in", "how's my day going"
- "evening readout", "evening summary", "evening insights",
  "wrap up my day"
- "daily readout", "today's readout", "give me the readout"

Use `type: 'auto'` (default) unless the user specifies a window. Pass
`timezone` if the agent knows the user's IANA zone.

Chain pattern: an evening readout may suggest "want me to dig into the
glucose dip at 5pm?" — that's a downstream `metabolic_overview` *or*
`health_sync` with `cgm_minute`. Wait for confirmation before chaining.

### Rule 4 — CGM-focused single-day grading → `metabolic_overview`

Triggers:
- "how's my CGM" / "how's my glucose" + a single date (today / yesterday
  / a specific day).
- "metabolic overview", "GWS", "glucose wellness score", "TIR today",
  "time-in-range today", "spikes today".
- "compare my GWS this week" *(call once per day, then synthesise — do
  not stream 7 calls in parallel without confirmation)*.

Required input: `date` (YYYY-MM-DD) — default today UTC.
Pass `timezone` if known.

Use this **instead of** `health_sync`+`aggregate=true` when the user
wants the LLM-narrated "what's driving" + recommendations block.
`health_sync` with aggregate gives you the numbers; `metabolic_overview`
gives you the numbers *plus* prose.

Do **not** route here for intra-day "what happened at 6pm?" questions —
that's `health_sync` with `cgm_minute`.

**Tiebreaker vs Rule 5 — easy for an LLM router to confuse.** Both rules
fire on single-day glucose questions. The difference is *what shape*
the user wants back:

| Phrasing | Route to |
|---|---|
| "how's my glucose today?", "what's my GWS?", "give me the grade", "score for yesterday" | **this rule** (`metabolic_overview` — narrated grade + scoring + recs) |
| "show me today's glucose curve", "plot today", "what did my glucose do at 6pm?", "during/after my run" | **Rule 5** (`health_sync` cgm_minute — raw series the agent reads itself) |
| "walk me through my glucose" — ambiguous; default to `metabolic_overview` (narrated, faster) unless the user says "walk me through the *curve*" or names a sub-day window |

### Rule 5 — Intra-day glucose curve → `health_sync` with `cgm_minute`

Triggers:
- "What did my glucose do during / after / at <event>"
- "Show me the postprandial response to <meal>"
- "Did exercise drop my glucose"
- "Anything weird overnight"
- "Plot a 7-day glucose trend"

Build the call as:
```jsonc
{
  "skill": "health_sync",
  "input": {
    "include": ["cgm_minute"],
    "cgm_minute_from": "<ISO start>",
    "cgm_minute_to":   "<ISO end>",
    "cgm_minute_resolution": "1m" | "5m" | "15m" | "30m" | "1h"
  }
}
```

Resolution heuristic:

| Window length | Resolution |
|---|---|
| ≤ 4 h (single meal / workout) | `1m` |
| 4–24 h | `5m` *(default)* |
| 1–3 days | `15m` |
| 3–14 days | `30m` |
| 14–30 days | `1h` |

Hard caps: 1,440 rows per call, 30-day max window. The response includes
`cgm_minute_meta.capped: true` if the agent must narrow + retry.

**Timezone handling (required).** The user almost always says "at 6pm"
or "after breakfast", not an ISO timestamp. The agent must resolve
their IANA zone *before* constructing `cgm_minute_from`/`cgm_minute_to`:

1. Get the user's IANA zone (`America/New_York`, `Asia/Jakarta`, …)
   from your agent's context — most runtimes carry it via the OS or
   user profile.
2. Anchor the user's spoken time in that zone (`2026-05-09 18:00` in
   `America/New_York` = `2026-05-09T22:00:00Z`).
3. Send the UTC ISO 8601 string with explicit `Z` suffix.

If you don't know the user's zone, **default to UTC and tell them**
("Reading your CGM in UTC — let me know if you want a different zone")
rather than silently picking a tz that may be off by ±12 hours.
Note: unlike `metabolic_overview` (which accepts a `timezone` field),
`cgm_minute` does the conversion on the agent side — Sally only sees
UTC ISO strings.

### Rule 6 — Multi-source data pull (the on-ramp) → `health_sync`

**What comes back.** One call returns **64 distinct daily biomarkers** spread across
six clinical domains — **vitals** (16 fields: HRV, RHR, VO₂ max, SpO₂, body energy,
recovery, readiness, sleep score…), **CGM glucose** (12 fields: mean, TIR %, GWS,
eHbA1c, MAGE, LBGI/HBGI, spike count…), **time-in-range bands** (8 fields covering
optimal, suboptimal, mild + severe hypo/hyper), **sleep architecture** (19 fields:
REM/deep/light/awake minutes, sleep debt + recovery, plus 10 per-axis quality scores
from respiration to circadian rhythm), an **activity event stream** (every workout +
step bucket), and **environment** (8 fields: daylight minutes, AQI, UV, vitamin-D
RDA + adequacy band). Opt into `cgm_minute` and you also get minute-resolution
glucose for postprandial / exercise / overnight curves. Allowlist-scoped, free,
typically 50-300 ms. This is the *most data-dense* skill — treat it as the agent's
default grounding step before anything that needs personalised reasoning.

Triggers:
- "Pull my health data"
- "Sync / fetch / show my <thing>" where `<thing>` ∈ {wearable, sleep,
  vitals, recovery, HRV, steps, daylight, AQI, environment, activity,
  TIR weekly, CGM weekly}.
- "How's my <metric> trending this week / month"
- Any agent reasoning step that needs grounding numbers before
  answering — call `health_sync` *first* even if the user didn't
  ask for raw data, then reason over the response.

Default call: `{}` → last 30 days, all six daily-aggregate sources.

Use `aggregate: true` when you want the same shape Sally's morning-
insight + metabolic-overview skills accept as input — handy for your
own LLM prompt rather than parsing six arrays.

`health_sync` is **FREE**. Default to calling it before any paid skill
when health context would improve the answer.

**Rule 6 vs Rule 8 — pull-and-reason vs let-Sally-synthesise.** Both
ground in the user's data; pick on latency + voice, not cost:

| You want… | Use |
|---|---|
| Numbers in your own context, your own LLM voice, fast (~100 ms data fetch + your tokens) | **Rule 6** (`health_sync`) |
| Sally's clinical voice + mem0 + chained tool calls under the hood, slower (6-12 s) | **Rule 8** (`chat_with_sally health: true`) |
| To *show* the user a chart / table / graph | **Rule 6** — you control the rendering |
| Cross-source synthesis ("is my poor sleep causing my spikes?") | **Rule 8** — Sally weaves labs + CGM + sleep + history |

Cost: Rule 6 is FREE + your own LLM tokens; Rule 8 is $0.003 flat. They
trade latency and authorial control, not money.

### Rule 7 — Knowledge or TCM question → `chat_with_sally`

Triggers (and `health: false`, default):
- "What does <metabolic / TCM / nutrition / sleep concept> mean?"
- "Explain X", "what's the difference between X and Y", "why does X
  cause Y" — when the topic is in Sally's lane (preventive health,
  TCM, nutrition, supplements, sleep, glucose, hormones, longevity).
- Any question Sally would answer from her curated knowledge base
  rather than the user's data.

Required input: `message`.
Optional: `knowledge` to bias retrieval (`'medical'`, `'tcm'`, or both).

`chat_with_sally` is **stateless per call** — there is no `conversation_uuid`
to track. The calling agent owns its own conversation thread. Re-supply any
short-term context inside the next `message`. Long-term user memory still
works (mem0 keys on the bearer-resolved user identity).

### Rule 8 — Personalised health Q&A → `chat_with_sally` with `health: true`

**Rule 7 vs Rule 8 — pronoun is the signal.** The deterministic rule:

> **First-person pronoun ("my", "I", "me", "mine") + a health metric
> or biomarker → Rule 8.** Anything else → Rule 7.

| Phrasing | Route to |
|---|---|
| "Why does poor sleep cause glucose spikes?" *(generic, third-person)* | Rule 7 |
| "Is **my** poor sleep causing **my** glucose spikes?" *(first-person + metric)* | Rule 8 |
| "What's a healthy HbA1c?" | Rule 7 |
| "What's **my** HbA1c trend?" | Rule 8 |
| "How does berberine compare to metformin?" | Rule 7 |
| "Should **I** try berberine given **my** glucose?" | Rule 8 |

Triggers:
- "Looking at *my* data / labs / numbers, …"
- "What should *I* focus on" / "what's *my* biggest issue"
- "Based on my CGM and sleep, …"
- Any request that needs Sally to read the user's data *and* generate
  a synthesised answer (not just numbers).

Required input: `message`, `health: true`.

Cost note: still $0.003 — `health: true` is the same flat price, just
slower (~6-12s) because Sally chains multiple tool calls under the hood.

Avoid this rule when a single-source skill suffices: "what's my TIR
today" should be `metabolic_overview`, not a personalised chat call.
Personalised chat is right when the *answer* needs synthesis across
sources — e.g., "is my poor sleep causing my glucose spikes?".

---

## Routing matrix (LLM-friendly compact form)

If the deterministic rules above don't match, the LLM may fall through
to this matrix. Pick the **first** row whose trigger fits.

```
trigger                                                     → skill
─────────────────────────────────────────────────────────────────────
attachment is PDF or photo of a lab result                  → analyze_lab_result
attachment is photo of food / a meal / a plate              → food_journal
asks for "morning|afternoon|evening|daily readout/insight"  → health_insights
asks "how's my CGM/glucose <today|yesterday|on DATE>"       → metabolic_overview
asks about glucose during/after a single intra-day event    → health_sync (cgm_minute)
asks for trend / sync / pull of health data                 → health_sync (default)
generic preventive-health / TCM / nutrition Q&A             → chat_with_sally
question needs cross-source synthesis of *my* data          → chat_with_sally + health:true
none of the above, but the topic is health                  → chat_with_sally (knowledge-only first)
none of the above, topic is not health                      → DO NOT call any Sally skill
```

---

## Chaining patterns

Some user requests need two skills. Chain explicitly — never silently:

| User says | Chain |
|---|---|
| "Snap-log my lunch and predict my CGM response" | 1. `food_journal` (photo) → 2. `health_sync` cgm_minute on the prior 24h → 3. agent synthesises |
| "Read my labs and tell me what to focus on" | 1. `analyze_lab_result` (PDF) → 2. `chat_with_sally` `health: true` with the analysis quoted in `message` |
| "Compare my GWS this week" | `metabolic_overview` × 7 (one per day, sequentially — not in parallel without user opt-in due to cost) |
| "Wrap up my day, anything weird in the glucose?" | 1. `health_insights` (`evening`) → if it flags a CGM anomaly: 2. `health_sync` with `cgm_minute` over the flagged window |
| "Did my training session hurt my sleep?" | 1. `health_sync` (default) covering the day in question → agent reasons over `vitals.sleep_score`, `sleep.*`, and `activity` |

When chaining, surface every call to the user — the agent should say
"I'll pull your CGM around 5pm to see the dip" before making the
second call, especially when the second call costs money.

---

## Anti-patterns (don't do this)

- **Don't accept a `user_uuid` / `email` / `id` field from the user
  and pass it through.** The Sally gateway rejects unknown input
  fields (`Zod.strict()`). The bearer key is the identity.
- **Don't call `metabolic_overview` for intra-day "what happened at
  X o'clock"** questions — those need `health_sync` with `cgm_minute`.
- **Don't include `cgm_minute` in `include` by default.** It's a
  large table; explicit opt-in keeps responses small.
- **Don't fan out 7+ paid calls in parallel** without user opt-in. A
  weekly comparison should be sequential or batched into a wider
  `health_sync` call where possible.
- **Don't call `chat_with_sally` for raw data pulls.** That skill is
  for *answers*, not data. Use `health_sync` to get the numbers, then
  reason over them in your own LLM, or use `chat_with_sally` with
  `health: true` if you also want Sally's interpretation.
- **Don't retry on `402 payment_required`.** It means the user's
  wallet is below the skill's price. Surface the message and ask the
  user to top up at `platform.a1c.io` rather than burning retries.
- **Don't retry on `401 unauthorized`.** Bad / revoked key. Tell the
  user to issue a new one at `platform.a1c.io` → API Keys.
- **Don't stuff the bearer in the request body.** It goes only in the
  `Authorization: Bearer …` header.

---

## Cost-awareness checklist (for cost-sensitive agents)

Before calling a paid skill, the router can ask:

1. Could `health_sync` (FREE) answer this on its own? If the user
   wants raw numbers or trends, yes — call it first, reason over the
   data in the LLM, skip the paid skill entirely.
2. Could the data already in conversation context answer this? If
   the agent already pulled `health_sync` earlier in the same
   conversation, don't re-pull unnecessarily.
3. Is the user's wallet healthy? Check the response of any prior call
   — `usage.cost_usd` and the `402 payment_required` error path tell
   you when to slow down.
4. **There is no proactive-balance endpoint** (yet). The only signals
   the agent has are `usage.cost_usd` (post-call, retrospective) and
   `402 payment_required` on the *next* call after balance dips below
   the skill price. Practical pattern:
   - Call optimistically.
   - On the first 402, surface the top-up message at `platform.a1c.io`
     and pause further paid skill calls for that user until the wallet
     is replenished.
   - **Do not** call `/v1/skills` (the public catalog list) as a
     "balance probe" — that endpoint is unauthenticated metadata and
     does not reflect wallet state.

`health_sync` is free precisely so this routing layer can call it
liberally without burning budget.

---

## Error handling

All skills return a uniform envelope:

```jsonc
// success
{ "ok": true,  "skill": "<name>", "version": "1.0.0", "data": {…}, "usage": { "units": 1, "cost_usd": 0.003 } }

// failure
{ "ok": false, "skill": "<name>", "error": { "code": "<code>", "message": "<human-readable>" } }
```

| `error.code` | What it means | Router behaviour |
|---|---|---|
| `unauthorized` | Bad / missing / revoked key | Stop. Ask user to re-issue key. |
| `payment_required` | Wallet < skill price | Stop. Ask user to top up. |
| `invalid_input` | Schema rejection (unknown field, bad shape) | Fix the input, retry once. |
| `not_found` | Resource (e.g. date with no data) doesn't exist | Surface message, do not retry. |
| `upstream_error` | Backend (langchain / OCR / DB) failed | Retry once after 1s; if still failing, surface. |
| `service_unavailable` *(HTTP 503)* | Gateway healthy but an upstream pod is temporarily unreachable (langchain / OCR cold-starting, DB failover) | Retry up to 2× with exponential backoff (1 s, 4 s). If still failing, surface — **do not** silently downgrade to a different skill. |
| `rate_limited` | Per-key bucket exceeded | Backoff per `Retry-After`, do not retry-storm. |

Skills do not throw — they always return the envelope. A non-200 HTTP
status is exceptional and indicates a gateway-level problem (network,
auth header missing).

---

## Per-skill cost summary (for budgeting)

| Skill | Cost / call | Typical latency | Idempotent? |
|---|---|---|---|
| `health_sync` | $0.000 | 50-300 ms | Yes |
| `chat_with_sally` (knowledge) | $0.003 | 3-5 s | Stateless — each call independent |
| `chat_with_sally` (`health: true`) | $0.003 | 6-12 s | Stateless — each call independent |
| `health_insights` | $0.003 | 4-8 s | Yes per `(type, date)` |
| `food_journal` | $0.004 | 4-7 s | No (each photo is a fresh analysis) |
| `metabolic_overview` | $0.005 | 4-8 s | Yes per `date` |
| `analyze_lab_result` | $0.008 | 6-15 s | No (each PDF is fresh OCR + LLM) |

Cache idempotent results inside a session if the agent has memory —
don't re-call `metabolic_overview` for `"yesterday"` twice in the
same conversation turn.

---

## Coming soon (route to nothing today)

These are listed in the public catalog as `coming_soon` — don't try
to call them; the gateway returns `not_found`:

- `health_report` — multi-week composite report
- `metabolic_risk_score` — composite risk model
- `supplement_grading` — supplement label evaluation
- `preventive_protocol` — personalised intervention protocol

When users ask for these, tell them they're on the roadmap and ask
whether a related live skill might address what they actually need.
**Do not silently substitute.** Specifically:

- `metabolic_risk_score` (multi-week composite) is **not** the same
  thing as `metabolic_overview` (single-day grade) — substituting
  would give the user a fundamentally different number from what they
  asked for. Surface the roadmap message instead.
- `health_report` (multi-week composite report) is **not** the same as
  any single-day skill or as a chain of `health_insights` calls.
- `supplement_grading` and `preventive_protocol` have no current
  equivalent in the catalog. `chat_with_sally` with `health: true` can
  answer adjacent questions, but it should never be presented as
  *the* `supplement_grading` or `preventive_protocol` skill — only as
  a related option the user can opt into.

---

## See also

- [`catalog/`](catalog/) — per-skill input/output detail
- [`protocols/mcp.md`](protocols/mcp.md) — MCP wiring
- [`protocols/api.md`](protocols/api.md) — raw REST recipes
- [`agents/`](agents/) — per-agent quick-starts
- [`README.md`](README.md) — onboarding, key issuance, wallet
