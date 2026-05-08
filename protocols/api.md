# Connect via REST API

If you're scripting, building your own integration, or your agent platform
doesn't speak MCP yet, talk to Sally with plain HTTPS.

---

## Endpoints

| Endpoint | Purpose |
|---|---|
| `POST https://sally.a1c.io/v1/call` | Universal dispatcher — `{ skill, input }` body |
| `POST https://sally.a1c.io/v1/skills/<name>` | Per-skill URL — input goes in the body directly |
| `GET https://sally.a1c.io/v1/skills` | List all skills + prices (no auth needed) |
| `GET https://sally.a1c.io/health` | Liveness check (no auth needed) |

All endpoints accept JSON. Authenticated routes require:

```
Authorization: Bearer sk-sally-…
Content-Type: application/json
```

---

## Quick examples

### List the catalog (no auth)

```bash
curl -sS https://sally.a1c.io/v1/skills | jq
```

### Pull your health data — `health_sync` (FREE)

```bash
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"skill":"health_sync","input":{"aggregate":true}}' | jq
```

### Ask Sally something — `chat_with_sally` ($0.003)

```bash
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"skill":"chat_with_sally","input":{"message":"What does HRV tell me about recovery?"}}'
```

### Analyse a lab PDF — `analyze_lab_result` ($0.008)

```bash
B64=$(base64 < ~/Downloads/labs.pdf | tr -d '\n')

curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d "$(jq -nc --arg b64 "$B64" \
        '{skill:"analyze_lab_result", input:{pdf_b64:$b64, filename:"labs.pdf"}}')"
```

### Snap a meal — `food_journal` ($0.004)

```bash
B64=$(base64 < ~/Pictures/lunch.jpg | tr -d '\n')

curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d "$(jq -nc --arg b64 "$B64" \
        '{skill:"food_journal", input:{image_b64:$b64}}')"
```

### Daily readout — `health_insights` ($0.003)

```bash
# Auto-detect type from current time
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"skill":"health_insights","input":{"type":"auto"}}'

# Or force a specific window
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"skill":"health_insights","input":{"type":"morning","date":"2026-05-08"}}'
```

### Metabolic overview — `metabolic_overview` ($0.005)

```bash
# Today
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"skill":"metabolic_overview","input":{}}'

# Specific date
curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"skill":"metabolic_overview","input":{"date":"2026-04-24"}}'
```

---

## Per-skill URLs (alternate)

If you prefer a cleaner URL per skill, use `/v1/skills/<name>`. Body is the
input directly (no `{skill, input}` wrapper):

```bash
curl -sS https://sally.a1c.io/v1/skills/health_sync \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d '{"aggregate": true}'

curl -sS https://sally.a1c.io/v1/skills/metabolic_overview \
  -H "Authorization: Bearer sk-sally-…" \
  -d '{"date":"2026-04-24"}'
```

Same dispatcher, same pricing, same audit log.

---

## Response envelope

Every response — REST or MCP — looks like this.

### Success

```json
{
  "ok": true,
  "skill": "metabolic_overview",
  "version": "1.0.0",
  "data": { "date": "2026-04-24", "gws": { "value": 100, "category": "EXCELLENT CONTROL", "agent_response": "…" }, "...": "..." },
  "usage": { "units": 1, "cost_usd": 0.005 }
}
```

### Error

```json
{
  "ok": false,
  "skill": "metabolic_overview",
  "error": {
    "code": "not_found",
    "message": "no CGM data for 2026-05-08. Sync your CGM via the insight-app or pick a different date.",
    "details": null
  }
}
```

### Error codes

| Code | HTTP | Reason |
|---|---|---|
| `unauthorized` | 401 | Missing / malformed / revoked key |
| `payment_required` | 402 | Wallet balance < skill price |
| `forbidden` | 403 | Key valid but not allowed for this skill (rare) |
| `not_found` | 404 | Unknown skill, or no data for the requested date / user |
| `invalid_input` | 422 | Zod input schema rejected the body |
| `rate_limited` | 429 | Per-key rate bucket exhausted |
| `upstream_error` | 502 | Sally's AI backend hiccupped — retry once |
| `gateway_error` | 500 | Bug in Sally's gateway — please report |

---

## Tips

- **Idempotent retries** are safe on 5xx. Don't auto-retry on 4xx — the
  request will fail the same way.
- **Streaming is not supported on REST** — every response is one JSON object.
  If you need a chat stream, use the agent + MCP path.
- **Per-call latency**:
  - `health_sync` (raw): ~50-200 ms
  - `health_sync` (aggregate): ~200-500 ms
  - `chat_with_sally`: ~3-8 s
  - `food_journal`: ~5-15 s (VLM + Milvus + reasoning LLM)
  - `analyze_lab_result`: ~15-30 s (PDF VLM + LLM analysis)
  - `health_insights`: ~5-12 s
  - `metabolic_overview`: ~10-20 s
- **Timeouts**: set your client to 90s minimum for AI-bound skills.
