# `chat_with_sally` — Sally as a tool inside your agent

**$0.003 / call.** Ask Sally a preventive-health or Traditional Chinese
Medicine question and get a grounded, source-cited answer back.

Sally is trained specifically on metabolic health, TCM, nutrition,
sleep, supplements, and clinical guidelines. She's not a search engine
and she's not a generic assistant — she's the small-scope expert your
agent can delegate to when a question is in her lane.

---

## Two modes

The skill takes one toggle that decides what Sally has access to.

| Mode | When | What Sally sees |
|---|---|---|
| **Knowledge-only** *(default)* | "What's the difference between LBGI and HBGI?" | Sally's curated knowledge base + web links you mention. **No** access to your personal data. Fast (~3-5s). |
| **Personalised** (`health: true`) | "Looking at *my* labs, where should I focus?" | Knowledge base + your labs / CGM / sleep / vitals / past documents / mem0 conversation memory. Slower (~6-12s, runs more tools). |

Default is knowledge-only on purpose: zero PHI exposure when you're just
asking a general question. Flip `health: true` only when the question
genuinely needs *your* numbers.

---

## What you get back

```ts
{
  reply: string,                  // Sally's full answer, markdown-formatted
  sources: [{ title, url, snippet }],   // citations from the knowledge base
  language: 'en' | 'zh' | 'id' | …,     // detected response language
  tools_called: ['search_knowledge_base', 'get_user_lab_results', …],
  elapsed_ms: number
}
```

Two things worth knowing:

- **`sources`** is non-empty for any non-trivial health question — Sally
  refuses to answer from training data alone for medical / nutrition /
  TCM topics. If sources is empty and the topic was health-related,
  Sally said so explicitly in `reply` (e.g. "I don't have data on this").
- **`tools_called`** is your transparency window: it tells you exactly
  which retrieval tools fired so you can trust the grounding.

> **Stateless per call.** `chat_with_sally` does not return (or accept)
> a `conversation_uuid`. Your agent already manages its own conversation
> with you — it threads turns however it wants and re-supplies any
> short-term context inside the next `message`. Long-term memory of
> your profile, past labs, etc. still works (mem0 keys on your account,
> not on a per-thread token).

---

## What you can ask your agent

Things Sally is great at — let your agent route these to her:

- *"Sally, what does a high HRV mean compared to a high RHR?"*
- *"In TCM, what's the difference between yin and yang deficiency in sleep?"*
- *"Can you ask Sally to compare ADA vs ACC HbA1c targets?"*
- *"Have Sally explain time-in-range to me — I'm new to CGM."*
- *"Sally, looking at my last week of CGM and sleep, what jumps out?"* *(needs `health: true`)*

Things Sally will politely decline or redirect:

- Generic coding / writing / non-health questions (your agent handles those itself)
- Diagnosis or prescriptions (she'll explain what the data suggests, never tell you what to take)
- Anything she can't ground in her sources

---

## Inputs (full schema)

```ts
{
  message:           string,                                // required
  knowledge?:        'medical' | 'tcm' | ['medical','tcm'], // bias retrieval; default both
  health?:           boolean,                               // unlock personal data; default false
  language?:         string                                 // ISO code; auto-detected from message otherwise
}
```

The agent's LLM picks `health: true` automatically when the message
mentions personal context ("my labs", "yesterday's sleep", "my CGM").
You can also be explicit in your prompt.

---

## Privacy & cost

- **`health: false` is free of PHI.** Sally never reads your data in this
  mode — purely her knowledge base + web links you cite.
- **`health: true` writes to mem0**. After the call, Sally retains a
  trimmed summary in conversation memory so she can be smarter on the
  next call. You can clear it any time at `console.a1c.io` → Memory.
- **$0.003 every call**, knowledge-only or personalised. No surcharge
  for tools used.
- **Audit log** at `console.a1c.io` → Usage. Every call is one row,
  marked with elapsed time and which tools fired.

---

## How to call it

### Via your agent (recommended)

Once your agent has the Sally MCP server connected
([protocols/mcp.md](../protocols/mcp.md)), just ask:

```
"Ask Sally what HRV tells me about recovery."
```

The LLM picks `chat_with_sally` automatically.

### Direct REST

```bash
# Knowledge-only (default)
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{
        "skill":"chat_with_sally",
        "input":{ "message":"What does a high HRV mean compared to high RHR?" }
      }' | jq

# Personalised — opens Sally's access to your own data
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{
        "skill":"chat_with_sally",
        "input":{
          "message":"Based on my last week of glucose, what should I focus on?",
          "health": true
        }
      }'

# Continue the conversation — your agent re-supplies context inside `message`
# (the skill itself is stateless; no thread token to track).
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{
        "skill":"chat_with_sally",
        "input":{
          "message":"Earlier you walked me through TIR and GWS. What about CV specifically?",
          "health": true
        }
      }'
```

---

## Limits & timing

- **Message length**: up to 8000 chars.
- **Latency**: ~3-5s knowledge-only, ~6-12s personalised. Long answers
  with multiple tool calls can hit 15s. Set your client timeout to 60s.
- **Streaming**: not supported — you get the full answer in one JSON
  response. (We may add SSE later if there's demand.)
- **Conversation length**: unbounded across calls (mem0 keeps a trimmed
  rolling history) but each individual `message` should fit one
  question well.

---

## See also

- [Main user guide](../README.md)
- [`health_sync`](health-sync.md) — pull data first if you want Sally
  to reason over your full picture
- [`metabolic_overview`](metabolic-overview.md) — for "how's my CGM
  doing?" specifically (more structured, cheaper, often faster)
- [MCP setup](../protocols/mcp.md) · [REST API](../protocols/api.md)
