# `lookup_food` — macros and glycemic classification for a named food

**FREE. No account required.** Name a food, optionally give a portion in
grams, get macronutrients plus a metabolic classification.

---

## What you get back

```ts
{
  food: string,
  grams: number,
  macros: { kcal: number, protein_g: number, carb_g: number, fat_g: number, fiber_g: number },
  classification: string   // smart choice vs likely glucose trap
}
```

The classification is the metabolically interesting part. Macros tell you
what is in the food; the classification tells you how it is likely to behave
in a body.

---

## Composing meals

For a multi-item meal, look up each component and reason about the
combination. Fat, fiber, and protein alongside a carbohydrate change the
glucose response, so the parts do not simply sum.

---

## Boundaries

Figures come from Open Food Facts under ODbL 1.0 and describe a typical
version of the food, not the specific dish in front of the user. Individual
glucose responses vary; a user syncing their own CGM through A1C Insights
sees their real curve, which is the one that matters for them.
