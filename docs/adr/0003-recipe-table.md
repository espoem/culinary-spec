# ADR-0003: When a recipe table is generated

- Status: Accepted
- Date: 2026-10-25
- Decided in: grilling round 4, from the live markup of the reference product and eight reference screenshots (see Context)

## Context

Four layouts are required, one of them the cooking-for-engineers table
(ingredients as rows, steps as spanning cells). A review argued the merges are
hand-drawn editorial decisions and that generating them lies on the first recipe
with parallel work.

The reference product contradicts that: its per-recipe pages emit a plain table
whose step cells carry a **computed** `rowSpan` and a label, with `prep-cell` rows
spanning full width and `blank-cell` fillers; the same component is rendered again
as a hidden share preview. The two parallel steps in the sample are disjoint spans
packed side by side, which is correct, not a false sequence. The real failure is
width: a croissant recipe reaches about twenty columns of two-word cells.

## Decision

1. **Generate the table from the step-to-instance assignment** when the spans,
   ordered by first use of each ingredient, are **laminar** — they nest or sit
   side by side without crossing.
2. **Refuse per recipe, not per site.** Crossing memberships (an ingredient used,
   then another, then the first again in an overlapping lane) and split uses fall
   back to the phase rail. The fallback is a layout choice recorded on the recipe,
   not an error.
3. **Cap the width.** Above ~10 lanes the recipe renders as the phase rail, and
   print always regroups by phase. Shrinking type is not an option.
4. **Mobile has no grid.** The phone projection is the lane timeline over time,
   drawn from the same assignment.
5. **The grid is never the accessible source of truth.** The linear step list is,
   with the table as a visual projection of it.
6. **Quantities are fields, not display strings.** The reference site stores
   "1 cup / 225 g" as one string, which cannot scale or translate; we store
   amount, unit and per-language name separately so one table renders in CZ and
   EN.

## Consequences

- The ingredient table needs an explicit `step` reference per instance at authoring
  time; that is the price of not inferring structure from paragraphs.
- Table layout is a pure function of the document plus a laminarity check, so it
  is testable without a browser.
- A recipe can deliberately opt out of the table without looking broken.

## Rejected

- Hand-drawing each table as page data.
- Generating tables for every recipe regardless of crossing spans.
- A horizontally scrolling grid on mobile.
