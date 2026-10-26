# ADR-0011: What the printed sheet is

- Status: Accepted
- Date: 2026-10-26
- Decided in: ticket *Printed recipe sheet prototype* (#5), from the measured prototype in `prototypes/print-sheet/`

## Context

Four recipe layouts are required and three of them also have to work on paper. The
prototype in `prototypes/print-sheet/` renders one document into all three print
grammars and was measured by printing it in Chromium and reading the PDF back.

Measurements (Svíčková: 13 ingredient instances, 7 steps, 4 servings):

| Projection | Pages | MediaBox |
|---|---|---|
| Paragraph | cover + 2 | 595 × 842 pt (A4) |
| Phase rail | cover + 1 | A4 |
| Matrix, authored short cues | cover + 1 | A4 |
| Matrix, truncated prose in cells | cover + 2 | A4 |

`@page { size: A4 }` is honoured by the engine; a page count alone would not have
proved that, which is why the MediaBox was read from the PDF.

## Decision

1. **A4 is the only print target** for v1, portrait, 18 mm margins, 174 mm text
   measure. US Letter is a later, separate issue; mixing them now would double the
   page-break tests for no reader.
2. **Base type is 10.5 pt serif on 1.5 leading.** Two sheets for a braise is
   acceptable; 9.5 pt to save a sheet is not, because the sheet is read at arm's
   length in a kitchen.
3. **Grammar selection is a rule, not a choice.** Up to 8 steps and 12 ingredient
   instances: paragraph. Above either: phase rail. `table_layout: matrix` overrides
   only when the spans are laminar; otherwise the sheet falls back and says so on
   the page.
4. **Per-step amount chips are omitted from the paragraph grammar** (they duplicate
   the list) and kept in the rail and the matrix, where the quantity is not
   otherwise adjacent to the step.
5. **Matrix cells carry the authored `short` cue** (ADR-0006), never truncated
   prose. The cue is capped at 48 characters; truncation is what pushed the sheet
   from one page to two.
6. **One matrix column per step, in document order.** Reading order is time order;
   the lane cap from ADR-0003 decides when the matrix is refused.
7. **The running head repeats on every printed page** and carries the recipe title
   and the language. Folios are computed after pagination, not authored.
8. **No content may be printed that contradicts the screen.** The refusal state, the
   yield, and the approximation markers appear identically in both projections;
   printing never reverts to the authored amounts once the reader changed the yield
   (the failure observed in research, #15).
9. **The sheet is the durability test for the document.** If a field cannot be
   rendered legibly on A4, the document contract is wrong, not the stylesheet.

## Verification

The print layer of `verify` asserts, per grammar, on a fixture set:

- MediaBox width 595 pt ± 2 and height 842 pt ± 2, read from the PDF, not assumed
  from the CSS.
- Page count equal to the committed expectation per fixture and grammar, so a
  layout regression that adds a page fails the build.
- No step split across pages and no table row split, read from the text layer.
- The pinned font file present (Poppler `pdffonts`), per ADR-0002.

## Consequences

- Two pages per recipe is normal; a collection of 50 recipes is about 100 sheets.
- The paragraph grammar cannot be the only print path, which is why the rail
  exists.
- Print layout depends on the `short` cue being authored, so the content pipeline
  must collect it at review time.

## Rejected

- Letter as an equal target.
- Shrinking type to force one page.
- Truncating prose into matrix cells.
- Printing the authored yield after the reader changed it.
- Silent fallback: the sheet states which grammar it is and why.
