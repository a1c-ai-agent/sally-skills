# Skill catalog — what each one returns

The agent guides in [`agents/`](../agents/) tell you how to *connect*. The
protocol guides in [`protocols/`](../protocols/) tell you *how to call*.
This folder tells you exactly **what comes back**, with examples.

| Skill | Cost / call | One-liner | Detail |
|---|---|---|---|
| [`health_sync`](health-sync.md) | $0.001 (10 free / mo) | Pull your wearable + CGM + sleep + vitals + activity + environment data into the agent's context | 64 fields breakdown |
| [`chat_with_sally`](chat-with-sally.md) | $0.003 | Ask Sally preventive-health / TCM questions with source citations | knowledge-only vs personalised |
| [`analyze_lab_result`](analyze-lab-result.md) | $0.008 | Drop a lab PDF, get clinical interpretation | OCR + LLM analysis |
| [`food_journal`](food-journal.md) | $0.004 | Snap a meal photo, get macros + smart/trap food rating | VLM + macro lookup |
| [`health_insights`](health-insights.md) | $0.003 | Morning / afternoon / evening readout from the day's data | server-side payload |
| [`metabolic_overview`](metabolic-overview.md) | $0.005 | CGM snapshot for a date with clinical scoring + narrative | scoring bands |
| [`search_health_knowledge`](search-health-knowledge.md) | **FREE** | Cited passages from Sally's clinical library, sources resolve to almanac.a1c.io | no account needed |
| [`lookup_supplement_grade`](lookup-supplement-grade.md) | **FREE** | Quality grade for a named supplement product, with reasoning | no account needed |
| [`lookup_food`](lookup-food.md) | **FREE** | Macros plus glycemic classification for a named food | no account needed |

If you're new here, start at the [main README](../README.md).

---

## How these pages are structured

Every catalog page covers the same five things:

1. **One-line value prop** — what the skill is for
2. **What you get back** — example response, with the meaningful
   fields called out in plain English
3. **What you can ask your agent** — concrete prompts that route
   to this skill
4. **Privacy & data lifecycle** — what the skill does (and doesn't)
   do with your data
5. **How to call it** — agent-side prompt + direct REST curl
6. **Limits & timing** — size caps, latency, streaming notes

If you're integrating from scratch, read the catalog pages in order
of cost — `health_sync` (free) first to understand the data,
then the paid skills in whatever order matches your use case.
