# ADR-0002: What `verify` may and may not claim

- Status: Accepted
- Date: 2026-10-25
- Decided in: grilling rounds 3-4, after two adversarial reviews

## Context

The first plan made `mise run verify` the outside judge: corpus schema
validation, scaling parity against the `cook` CLI, JSON-LD validity, axe
violations, Lighthouse budgets, PDF page-count assertions, an 8-word
shingle-overlap test, and hand-written scaling fixtures as ground truth.

An adversarial review showed five of those can be green while the product is
wrong: parity is agreement with an unverified reference binary on exactly the
modifier subset that is unverified; a schema written by the same author validates
its own corpus; shingle overlap catches copying but not adaptation and cannot
fail when the scraped source is absent; a page-load axe scan says nothing about
interaction or a closed dialog; `page.pdf()` defaults to Letter with
`preferCSSPageSize: false`, so `@page { size: A4 }` never controls the box and the
page count measures one host's font fallback.

## Decision

Each check must fail for a reason a reviewer can recompute, and must not be able
to pass vacuously.

1. **Arithmetic is literal, in-test.** A hand-built incidence graph with literal
   expectations (flour 200 g linear → 400 g at 2×, oil 30 ml fixed → 30 ml) is the
   scaling judge. Kitchen fixtures are inputs and regression material, never the
   oracle. When a fixture disagrees with the scaler, the fixture must be edited in
   a separate commit with a note.
2. **A mutant must fail.** Each structural rule ships a deliberately broken input
   under `tests/corpus/mutants/` (dangling ingredient reference, crossing spans,
   missing language field) and verify fails if the mutant passes or is not wired.
3. **Copyright is provenance, not shingles.** Every published recipe carries a
   provenance sidecar (licence, source, retrieval date, source digest). Missing
   scrape data exits **2**, never 0. Quantity-tuple Jaccard against the source
   guards a re-expressed formula; prose n-grams are a secondary signal only.
   Images get a perceptual hash check.
4. **Accessibility is a contract, not a count.** A checked-in axe rule list is the
   contract: an empty run or a disabled `color-contrast` fails. The scan runs on
   the post-hydration DOM, inside an opened dialog, and after a servings change,
   asserting a live-region update and preserved focus.
5. **Performance is measured on an interaction.** Lighthouse budgets cover the
   load path only and are a floor, not the NFR. Event Timing entries from a
   scripted interaction (servings 4 → 6) carry the INP assertion.
6. **Print assertions name the sheet.** `preferCSSPageSize: true`, MediaBox width
   595 pt for A4, a pinned font file with `pdffonts` asserted, and no page-break
   inside a step or a table row. Page count alone is not an assertion.
7. **Language parity is structural.** CZ and EN versions must agree on ingredient
   identities, amounts, units, temperatures, durations and step count; only text
   may differ.
8. **Same command everywhere.** `mise run verify` runs locally through `hk` and in
   GitHub Actions as a PR gate; deploy runs from `main` only after it is green.

## Consequences

- Verify is slower than a lint sweep and is split into fast (types, schema,
  arithmetic, mutants) and slow (browser, print, performance) layers.
- Every new feature adds a rule, a mutant, or neither — never a screenshot.
- A green run is evidence of the properties named above and nothing else; it does
  not claim the cookbook reads well or that the food works.

## Rejected

- `cook`/cookcli parity as the scaling oracle.
- Kitchen fixtures as ground truth.
- Shingle overlap as the copyright gate.
- `0 axe violations` as the accessibility NFR.
- Lighthouse budgets as the field performance NFR.
- PDF page-count assertions without `preferCSSPageSize` and a pinned font.
