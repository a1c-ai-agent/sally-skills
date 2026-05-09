# `food_journal` — snap a meal, get the metabolic read

**$0.004 / call.** Send a photo of what you're about to eat (or just
ate). Get a structured assessment back: dish name, macronutrient
breakdown, and Sally's smart-vs-trap food categorisation.

This isn't a calorie-counter — it's a meal *quality* judgement
grounded in Sally's curated food database. The output tells your
agent enough to give a useful "should I eat this?" answer.

---

## What you get back

```ts
{
  status:  "ok" | "not_food" | "error",

  // populated when status === "ok"
  dish_name?:        "Rice bowl with grilled salmon and avocado",
  food_description?: "Mixed grain bowl with protein and healthy fats…",

  macronutrient?: {
    protein:      { weight: "32g", kal: 128 },
    carbohydrate: { weight: "55g", kal: 220 },
    fat:          { weight: "18g", kal: 162 },
    fiber:        { weight: "8g",  kal: 0   }
    // shape varies — keys come from what the model identifies
  },

  food_category?: "smart_food" | "trap_food",
  reason_category?: "Balanced macros, low glycemic load, omega-3 source from salmon…",

  // populated when status !== "ok"
  message?: "The image does not appear to contain food or a beverage.",
  error?:   string
}
```

Three things to call out:

- **`food_category`** is the key. `smart_food` means the meal supports
  metabolic health for someone with your profile (low glycemic load,
  good macro balance, anti-inflammatory). `trap_food` means it's the
  kind of meal that tends to spike CGM, drive inflammation, or pile
  on cheap calories — even if it looks innocent.
- **`reason_category`** is the *why* in plain language. Short,
  agent-friendly, suitable for direct quotation back to the user.
- **`status: "not_food"`** is what comes back if you photograph a
  receipt, a wall, or a person. Sally's VLM refuses to fabricate food
  details — `message` will explain.

---

## What you can ask your agent

Drop a meal photo into the chat — most agents (Claude Desktop, Manus,
Cursor) support image attachments and the LLM picks the skill:

- *"What's in this meal?"* [photo]
- *"Should I eat this?"* [photo]
- *"Rate this meal for someone trying to keep glucose stable"* [photo]
- *"What would you swap to make this a smart food?"* [photo]
- *"Snap-log this lunch"* [photo]

If your agent supports CGM data, you can chain:

- *"Analyse this meal and predict how my CGM will respond based on
  yesterday's similar meal"* — the agent calls `food_journal` then
  `health_sync` for context.

---

## Inputs

```ts
{
  image_b64: string,    // REQUIRED. base64 JPEG/PNG/WebP/GIF (≤10MB raw, ≤14MB base64)
  mime?:     string     // optional override (e.g. "image/png"). Auto-detected from magic bytes otherwise.
}
```

Phase 1 supports image-only — text descriptions (e.g. "I had a
cheeseburger") aren't a separate input mode here. For text-only meal
questions, use `chat_with_sally` with `health: true`.

---

## Privacy & data lifecycle

- **Image bytes never persist**. The flow is agent → Sally's gateway
  → langchain's VLM service → response. No GCS upload of your meal
  photos, no Milvus storage of the input image, no disk write that
  outlives the request.
- **What Sally's database does have**: a curated reference index of
  foods + their macros + smart/trap categorisation. That's the
  "Milvus lookup" stage — it's a *read* of reference data, not a
  *write* of your photo.
- **VLM sees the image once** during analysis. Sally's VLM
  (Qwen VL via OpenRouter) is governed by OpenRouter's zero-retention
  default for the models Sally uses.
- **Allowlist response**. Only the fields above ever leave the
  boundary. Internal item arrays, Milvus match metadata, ingredient
  lists — none of those leak.

---

## How to call it

### Via your agent

Drop a meal photo into the chat. If your agent supports MCP
([protocols/mcp.md](../protocols/mcp.md)), it picks `food_journal`
on its own.

### Direct REST

```bash
B64=$(base64 < ~/Pictures/lunch.jpg | tr -d '\n')

curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg b64 "$B64" '{
        skill: "food_journal",
        input: { image_b64: $b64 }
      }')" \
  | jq '.data | {status, dish_name, food_category, reason_category, macronutrient}'
```

---

## Tips for a useful read

- **Plate the food first**. A clear top-down or 45° shot of the
  finished plate beats a shot of ingredients in the pan.
- **Daylight beats artificial light.** The VLM identifies foods
  better when colour is accurate.
- **One meal per call.** If you're snapping a multi-course meal, send
  separate calls — the smart/trap judgement is per-meal.
- **Re-shoot if `not_food`** — usually a framing issue (food too small
  in the frame, or angle hides the dish).

---

## Limits & timing

- **Max input**: 10 MB raw image (≈14 MB base64).
- **Latency**: 5–15 s typical (VLM + Milvus lookup + reasoning LLM
  in sequence). Set client timeout to 60 s.
- **Streaming**: not supported. One JSON response.
- **Languages**: dish names come back in English regardless of cuisine
  source — Sally's reference index is English-keyed.

---

## See also

- [Main user guide](../README.md)
- [`metabolic_overview`](metabolic-overview.md) — pair with this to
  see how a smart/trap food affects your actual CGM response
- [`chat_with_sally`](chat-with-sally.md) — for "why is this a trap
  food?" follow-ups
- [MCP setup](../protocols/mcp.md) · [REST API](../protocols/api.md)
