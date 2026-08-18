---
name: sally-supplement-check
description: "Check whether a specific supplement product is worth taking. Use when a user names a brand and product, asks if a supplement is any good, or wants to compare products on evidence quality rather than marketing."
---

# Sally Supplement Check

Use the `lookup_supplement_grade` tool from this plugin to retrieve A1C's stored quality grade for a named supplement product.

## Workflow

1. Get both the brand and the product name. "Magnesium" is not enough to identify a product; "Thorne Magnesium Bisglycinate" is.
2. Look the product up before commenting on it.
3. Report the grade with its reasoning, not just the letter. What earned or cost the grade is the useful part.
4. When a product is not in the library, say so instead of guessing. Absence is not a bad grade.

## Interpreting a grade

Grades reflect evidence quality and formulation, not popularity or price. A well-marketed product can grade poorly, and an unglamorous one can grade well. Where a user's real question is "should I take this at all", pair the grade with `search_health_knowledge` for the evidence on the ingredient itself.

## Boundaries

Grades are A1C's own assessment and are not endorsed by the NIH, whose Dietary Supplement Label Database is one of the underlying sources. Supplements interact with medications and conditions; route anything involving an actual prescription or diagnosis to a clinician.
