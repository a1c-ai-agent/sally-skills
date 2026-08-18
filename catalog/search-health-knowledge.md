# `search_health_knowledge` — cited passages from Sally's clinical library

**FREE. No account required.** Ask a metabolic-health question, get back the
source passages that answer it. This skill retrieves; it does not generate.

Sally's library is A1C-authored clinical writing plus supplement monographs
from NCCIH and 21 CFR 101.36. Every passage carries a citation that resolves
to [almanac.a1c.io](https://almanac.a1c.io), so a reader can open the article
and check the claim themselves.

---

## Why retrieval, not an answer

Your agent already writes well. What it lacks is grounding: health claims age
badly, and a confident paragraph of model recall is indistinguishable from a
correct one. This skill hands your agent the underlying text so the reasoning
happens in your context window, with sources attached.

---

## What you get back

```ts
{
  results: Array<{
    text: string;      // the retrieved passage
    source: string;    // citation, resolves to almanac.a1c.io
    score: number;     // semantic relevance
  }>
}
```

Search with the user's actual question rather than keywords. The corpus is
embedded semantically, so a full clinical question retrieves better than a
bag of terms.

---

## Boundaries

Educational reference material, not medical advice. Sally does not diagnose
and does not prescribe. If retrieval comes back thin, say so rather than
filling the gap from memory.
