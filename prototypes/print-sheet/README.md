# Print sheet prototype

One self-contained HTML file. Open it in a browser, press ⌘P / Ctrl P, enable
background graphics, and compare the printed sheet with the on-screen one.
Everything is generated from one in-file document per recipe; no number is typed
twice.

Review links:

- `index.html?grammar=paragraph&lang=cs&servings=4&recipe=svickova`
- `index.html?grammar=rail&lang=cs&servings=4&recipe=svickova`
- `index.html?grammar=table&lang=cs&servings=4&recipe=svickova`
- `index.html?grammar=table&lang=en&servings=8&recipe=mornay-sauce`

## What it proves

- One document renders three grammars: paragraph, phase rail, and a matrix whose
  row spans are computed from each step's consumed instances (ADR-0003), with the
  laminarity test and an automatic fallback to the rail when spans cross.
- Amounts come from a miniature of ADR-0004: four authored Czech plural forms,
  `Intl.PluralRules` for the slot, magnitude rounding, exact halves rather than
  decimals for counts. Serving changes propagate to every grammar at once.
- `@page { size: A4 }` is honoured by the print engine (measured below), and the
  running head repeats on every page through a fixed-position element.

## Measured in headless Edge (`--print-to-pdf`, MediaBox from the PDF)

| Projection | Pages | MediaBox |
|---|---|---|
| Paragraph, Svíčková, 13 ingredients, 4 servings | 3 (cover + 2) | 595 × 842 pt (A4) |
| Phase rail, same recipe | 2 (cover + 1) | A4 |
| Matrix, same recipe, with authored short cues | 2 (cover + 1) | A4 |
| Matrix, same recipe, with truncated prose in cells | 3 (cover + 2) | A4 |
| Matrix, English, 8 servings | 2 | A4 |

## Typographic decisions this prototype forces

1. **Base type**: 10.5 pt serif on 1.5 leading, text width 174 mm. The paragraph
   grammar still needs two pages for an ordinary braise; either the sheet accepts
   two pages or the type drops to 9.5 pt and loses reading comfort at arm's length.
2. **Quantity column**: tabular figures, non-breaking space before the unit, bold
   lead and regular food name. This is what makes a column of amounts scannable.
3. **Ingredient list**: two columns with hairline rules; notes in parentheses,
   optionality in a lighter cut. Fits 13 instances in about a third of a page.
4. **Step gutter**: 8 pt sans numeral in a 7 mm column, prose at full size. The
   numeral is deliberately quiet; the prose is the content.
5. **Per-step amount chips**: in print they duplicate the list. Keep them only in
   the rail and the matrix, or drop them from print entirely.
6. **Matrix cells must carry an authored short cue.** Truncated prose overflows a
   page and reads badly. Consequence for the contract: a step gains an optional
   `short` label per language, used by the matrix and by card layouts.
7. **Matrix width**: one column per step, in document order. Seven steps fit A4;
   the reference product breaks down at about twenty.
8. **Phase rail**: 44 mm amount column, prose right. The most legible of the three
   at arm's length in a kitchen, and the only one that survives 10+ steps.
9. **Running head and folio**: the head repeats via `position: fixed` in print;
   folios in the table of contents are placeholders until pagination exists.
10. **Fallback must be visible**: when the matrix refuses, the sheet says so
    instead of silently changing shape.

## Known limits

- No pagination engine: a real build needs folios computed after layout.
- No font pinning yet, so the PDF embeds whatever the host resolves (ADR-0002
  requires a pinned font file for the print gate).
- English uses en-GB spellings; US spellings are a locale variant later.
- The cover and table of contents are placeholders for the collection ticket.
