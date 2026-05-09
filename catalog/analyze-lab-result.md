# `analyze_lab_result` — drop a lab PDF, get a clinical read

**$0.008 / call.** Send a lab panel as a PDF or image; get the
extracted text plus a markdown clinical analysis written by an LLM
that knows ADA / ACC / ESC reference ranges.

Works on lipid panels, HbA1c, CBC, CMP, thyroid panels, hormone
panels, micronutrients, and most multi-marker reports. Also handles
hand-photographed paper printouts via the same VLM pipeline.

---

## What you get back

```ts
{
  document_type:   "medical_lab_report" | "prescription" | "general",
  page_count:      number,
  ocr_text:        string,                  // raw extracted text (markdown-formatted by the OCR model)
  analysis: {
    success:       boolean,
    markdown:      string | null,           // the clinical interpretation
    model_used:    string | null,           // e.g. "google/gemini-2.5-flash-lite"
    error?:        string                   // present only on analysis failure
  },
  processing_time_ms: number,
  filename:        string | null
}
```

The `analysis.markdown` field is the meat — a structured walk-through
of every meaningful marker on the report, what's in / out of range,
and what it might mean *in context with other markers*. Example shape:

```markdown
**Out-of-range markers**

- **HbA1c — 5.9%** (above optimal <5.7%, ADA prediabetes range 5.7–6.4%)
  Combined with your fasting glucose of 102, this suggests early
  insulin resistance rather than poor short-term control.

- **LDL-C — 142 mg/dL** (above ACC optimal <100)
  Your HDL of 58 is healthy, ratio 2.4 is acceptable but the absolute
  LDL is the pressure point. Consider…

**In range, worth noting**

- TSH 2.1 (normal but trending), free T4 1.0…
```

The `ocr_text` is the raw extracted content — useful if your agent
wants to do its own follow-up parsing or quote specific values.

---

## What you can ask your agent

Drop the lab PDF into the chat (most agents — Claude Desktop, Manus,
etc. — accept attachments) and the LLM picks the skill on its own:

- *"Analyse this lab PDF"*
- *"What's concerning in this report?"*
- *"Walk me through these results assuming I'm new to lab numbers"*
- *"Compare this lab panel to the one from January"* *(works if the agent
  has both PDFs in context)*

If your agent doesn't accept file attachments, you can base64-encode
the PDF locally and paste it into the call (see the curl below).

---

## Inputs

```ts
{
  pdf_b64:    string,    // REQUIRED. base64-encoded PDF or image (≤10MB raw, ≤14MB base64)
  filename?:  string,    // optional hint, e.g. "labs-jan-2026.pdf". Defaults to "document.pdf".
  llm_model?: string     // OpenRouter model id override. Defaults to a Sally-tuned cheap model.
}
```

Most agents handle the base64 step for you when you drop a file in.
For scripts, see the curl below.

---

## Privacy & data lifecycle

- **Bytes never persist**. The PDF flows agent → Sally's gateway →
  OCR service → response. No GCS upload, no S3 upload, no Milvus
  indexing of the input. The OCR engine writes the bytes to a temp
  file purely so the existing extraction pipeline can read them, then
  removes the file in a `finally` block.
- **OCR engine sees the bytes**. Mistral OCR + Llama 4 Scout (for PDFs
  / Word) or Qwen VL (for images) are routed via OpenRouter. Their
  retention is governed by OpenRouter's privacy policy (zero retention
  by default for the models Sally uses).
- **Cleanup is automatic**. Even if the OCR pipeline crashes, the
  temp file is removed. No partial PDF survives.
- **Allowlist response**. The seven fields above are *the only thing*
  the skill returns. Internal source paths, Milvus document IDs, OCR
  metadata — none of those leak.

---

## How to call it

### Via your agent (recommended)

Once your agent has the Sally MCP server connected
([protocols/mcp.md](../protocols/mcp.md)):

```
"Analyse this lab PDF" [drag PDF into the message in Claude Desktop / Manus / Cursor]
```

The agent's LLM base64-encodes the file and calls
`analyze_lab_result` on its own.

### Direct REST (scripts, your own apps)

```bash
B64=$(base64 < ~/Downloads/labs.pdf | tr -d '\n')

curl -sS https://sally.a1c.io/v1/call \
  -H "Authorization: Bearer sk-sally-…" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg b64 "$B64" '{
        skill: "analyze_lab_result",
        input: { pdf_b64: $b64, filename: "labs.pdf" }
      }')" \
  | jq '.data.analysis.markdown'
```

The `jq` at the end pulls just the analysis out so it pretty-prints in
your terminal. Drop it to see the full envelope.

---

## Limits & timing

- **Max input size**: 10 MB raw PDF (≈14 MB base64). Bigger labs
  return `invalid_input`. Compress with Preview / pdftk if needed.
- **Latency**: 15–30 s for a typical 1–3 page lab. 4-page panels
  occasionally hit 40 s. Increase your agent's tool timeout to 90 s.
- **Streaming**: not supported. The full analysis comes back in one
  JSON.
- **OCR quality**: hand photos work, but lighting matters — direct
  flash, no glare, full page in frame. If OCR mis-reads a value, you'll
  see it in `ocr_text` and Sally will reason from the wrong number.
  In that case, retry with a clearer photo.

---

## What if the OCR misses something?

The `ocr_text` field is your debug surface. If the analysis seems
wrong, check the raw extraction first — usually a low-quality scan
or unusual layout. Then either:

- Re-take the photo / re-export the PDF and retry
- Pass `chat_with_sally` the `ocr_text` directly with `health: true`
  and ask follow-up questions

---

## See also

- [Main user guide](../README.md)
- [`chat_with_sally`](chat-with-sally.md) — for follow-up questions
  after the analysis
- [`health_sync`](health-sync.md) — pulls past lab values your agent
  can compare against if you've already onboarded labs in the iOS app
- [MCP setup](../protocols/mcp.md) · [REST API](../protocols/api.md)
