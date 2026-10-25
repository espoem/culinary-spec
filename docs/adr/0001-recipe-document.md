# ADR-0001: Recipe document, composition and units

- Status: Accepted
- Date: 2026-10-25
- Decided in: grilling rounds 1-4

## Context

A bilingual CZ/EN cookbook, one operator, static first, with portion scaling,
nutrition, allergens, print output and a cooking-for-engineers table. The
original plan made a Cooklang-shaped DSL the source of truth and compiled it into
a DAG that every feature would read.

That plan failed on its own evidence: the worked Mornay fixture contradicts
itself (Gruyère 240 g under the modifier rule vs 120 g in the sentence, and the
`~1`/`~2` step references do not resolve to the steps the sample names), allergens
and nutrition are a second dataset that tokenised prose cannot carry, Czech
inflection breaks string-keyed ingredient matching, and parallel CZ/EN authoring
removes the single-prose advantage that motivates the syntax.

## Decision

1. **The recipe document is the source of truth**: a structured ingredient table
   plus written methods, one per language. Amounts, units and preparation notes
   are fields on ingredient instances, never substrings of prose.
2. **No token language.** Prose stays prose. A graph is derived from the document
   only where a feature needs it (component totals, table layout), and never gates
   publication: a recipe that yields no graph publishes without a scaler, not as
   an error.
3. **Composition.** A recipe may be assembled from components, depth ≤ 2, live
   references without version pinning. Component aggregation sums by ingredient
   identity and dimension across the composition tree; fixed instances never scale.
4. **Variants.** A variant changes ingredients or quantities of the same
   procedure. Different steps make a new recipe with a parent reference. A
   variant may be assembled from a different set of components and may have a
   different number of ingredients.
5. **Units.** SI is canonical and stored as data (mass, volume, temperature,
   duration). Imperial/US units are a display toggle with kitchen rounding, never
   a second stored value. Ranges and indivisible pieces are first-class
   (`500-600 g`, `2 eggs`).
6. **Languages.** CZ and EN are authored natively in parallel. Text fields are
   per language; quantities, units, temperatures and step counts are shared. A
   recipe missing a language is simply absent from that language's site, never a
   half-translated page.
7. **Nutrition** is entered per serving with a cited source. No computed rollup.
8. **Provenance** is mandatory per recipe and image: licence, source, retrieval
   date, and the source digest when one exists. Imported recipes are re-expressed
   from facts with original wording and original images.
9. **Publication states**: `draft`, `unverified` (live, `noindex`, badged),
   `reviewed` (human-checked against the source, indexable).

## Consequences

- Scaling, composition totals and the table layout all need explicit step-to-instance
  assignment; that assignment is authored, not inferred.
- The JSON-LD `recipeIngredient` strings are rendered from fields, so they cannot
  drift from the displayed list.
- Import tooling must produce the table, not copy prose; the human check compares
  facts against the source.
- The site can ship before any graph machinery exists.

## Rejected

- Cooklang `.cook` as authoring source of truth, and `cook` CLI as scaling oracle.
- A DAG that every feature reads and that gates publication.
- Storing imperial values alongside SI.
- Blocking publication on a missing second language.
