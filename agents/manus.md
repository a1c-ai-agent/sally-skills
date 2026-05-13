# Sally + Manus

[Manus](https://manus.im) supports MCP servers via its **Tools** panel.
Connection is web-based — no config files to edit.

---

## Setup

### Step 1 — Get your key

Follow [Section 1 of the main README](../README.md#section-1--onboarding-start-here):
1. Install **A1C Insights** on iPhone, sign up.
2. Visit **<https://console.a1c.io>**, generate a key.

### Step 2 — Add Sally as an MCP tool in Manus

1. Open Manus → **Tools** (or **工具** if your UI is in Chinese).
2. Click **Add MCP Server** / **添加 MCP 服务器**.
3. Fill in:
   - **Name** / **名称**: `Sally`
   - **URL**: `https://sally.a1c.io/mcp`
   - **Authorization header**: `Bearer sk-sally-…`
4. Save.

Manus will fetch Sally's `tools/list` and show six green tool entries:
`chat_with_sally`, `health_sync`, `analyze_lab_result`, `food_journal`,
`health_insights`, `metabolic_overview`.

### Step 3 — Enable for your agent

For each Manus agent that should use Sally, toggle the Sally tools on
in the agent's **Tool permissions**.

---

## Calling Sally from Manus

Once enabled, your Manus agent picks Sally tools automatically when the
conversation makes them relevant. Example prompts:

```
帮我看看我的血糖今天怎么样   ← will call metabolic_overview
分析一下这份化验单 [上传 PDF]  ← will call analyze_lab_result
我刚吃了这个 [上传图片]        ← will call food_journal
今天早上的健康洞察是什么?      ← will call health_insights
```

English works just as well.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Tools don't appear after adding the server | Manus retries `tools/list` periodically — wait 30s or remove + re-add the server |
| `401 unauthorized` | Header format is `Bearer sk-sally-…` (with the space). Recheck the value in console.a1c.io |
| Tool call times out | Manus enforces 60s per tool — `analyze_lab_result` may exceed this on large PDFs. Consider splitting or compressing the PDF |
| China region: connection refused | Sally's gateway is on Cloudflare — confirm Cloudflare isn't blocked in your region; if so, contact us for a region-specific endpoint |

→ Full troubleshooting: [`../protocols/mcp.md`](../protocols/mcp.md#troubleshooting)
