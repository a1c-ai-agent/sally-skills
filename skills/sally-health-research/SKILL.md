---
name: sally-health-research
description: "Answer metabolic health, longevity, and preventive medicine questions from cited clinical sources rather than model recall. Use for questions about glucose, HbA1c, insulin sensitivity, sleep, VO2 max, cardiovascular risk, and supplement or lifestyle interventions, and whenever an answer needs citations a reader can check."
---

# Sally Health Research

Use the `search_health_knowledge` tool from this plugin to ground health answers in A1C's curated clinical library. The tool returns retrieved source passages, not a generated answer: you do the reasoning, the library supplies the evidence.

## When to use it

Reach for it whenever the question is about metabolic health, longevity, or preventive medicine and the answer would otherwise come from model recall. Health claims decay quickly and are easy to get subtly wrong, so retrieve first and write second.

## Workflow

1. Search with the user's actual clinical question, not a keyword soup. The library is embedded semantically, so a full question retrieves better than fragments.
2. Read the returned passages before answering. If they do not support a claim, do not make it.
3. Cite the sources you used. Citations resolve to `almanac.a1c.io`, so a reader can open the article and verify.
4. If retrieval comes back thin, say so plainly rather than filling the gap from memory.

## Boundaries

Sally returns educational, source-cited reference material. It does not diagnose, does not prescribe, and does not replace a clinician. Anything that sounds urgent belongs with a doctor or emergency services, not with a tool call. Say that clearly when a question crosses into diagnosis or treatment.

## Related tools

- `lookup_supplement_grade` for a specific product's quality grade.
- `lookup_food` for nutrition figures and glycemic classification.

Personal skills that read a user's own biomarkers (glucose history, labs, sleep, vitals) require an API key from `console.a1c.io` and are not part of the free tier.
