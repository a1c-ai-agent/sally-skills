# `lookup_supplement_grade` — is this product actually worth taking

**FREE. No account required.** Give a brand and product name, get A1C's
stored quality grade with the reasoning behind it.

---

## What you get back

```ts
{
  product: string,
  grade: string,       // A-D quality band
  rationale: string,   // what earned or cost the grade
  source: string
}
```

Grades reflect evidence quality and formulation, not popularity or price. A
heavily marketed product can grade poorly and an unglamorous one can grade
well, which is the entire point of looking it up.

---

## Getting a hit

Pass both the brand and the product. "Magnesium" identifies nothing;
"Thorne Magnesium Bisglycinate" identifies a product. When a product is not
in the library you get no match, which means unknown, not bad.

---

## Boundaries

Grades are A1C's own assessment. The NIH Dietary Supplement Label Database is
one of the underlying sources and does not endorse them. Supplements interact
with medications and conditions, so anything touching a prescription belongs
with a clinician.
