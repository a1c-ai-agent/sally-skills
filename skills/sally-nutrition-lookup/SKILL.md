---
name: sally-nutrition-lookup
description: "Look up nutrition for a named food or dish, with macronutrients and a glycemic classification. Use when a user asks what is in a food, how a meal affects blood sugar, or whether something is a smart choice."
---

# Sally Nutrition Lookup

Use the `lookup_food` tool from this plugin to retrieve macronutrients and a metabolic classification for a named food, optionally scaled to a portion in grams.

## Workflow

1. Name the food as specifically as the user described it, and pass a portion in grams when they gave one. Absent a portion, the tool answers per its default serving; say which you used.
2. Report the macros and the classification together. The classification is the metabolically interesting part: it flags foods likely to drive a glucose spike versus those that are safe repeats.
3. For a multi-item meal, look up the components and reason about the combination. Fat, fiber, and protein alongside a carbohydrate change the response.

## Boundaries

Figures come from Open Food Facts under ODbL 1.0 and describe a typical version of the food, not the specific dish in front of the user. Treat them as a good estimate, not a measurement. Individual glucose responses vary; a user tracking their own data through A1C Insights will see their real curve, which is what actually matters for them.
