# ADR-0004: Units, rounding and safe scaling

- Status: Accepted
- Date: 2026-10-25
- Decided in: ticket *Units, rounding and the imperial toggle*, after two research passes

## Context

SI is canonical and imperial is a display toggle (ADR-0001), kitchen measures
(`lžíce`, `lžička`, `hrnek`, `špetka`) are authoring units, and a recipe may author
an explicit alternate unit. The open questions were how conversion and rounding
work, how ranges and indivisible pieces behave, and how a verify script catches an
obviously wrong unit.

Research found that the platform does not do this job. `Intl.NumberFormat`'s unit
style accepts a closed list of sanctioned units; `cup`, `tablespoon` and
`teaspoon` are not reliably in it, a custom id like `lzice` is implementation
defined, and `lžíce` must throw. It has no conversion, no magnitude rounding, no
approximate marker and no measurement-system getter. Czech does get four plural
categories for free (`one`, `few`, `many` for fractions, `other`), but the words
for our units are authored either way. No library in the shortlist carries Czech
inflection, custom units, authored-alternate precedence and kitchen rounding
together, and the conversion factors we need are four constants.

Research also found that published recipes get this wrong: a US baking site ships
three disagreeing yield figures for one recipe (24 servings, 4.5 cups, and 1/8 cup
per serving imply 36), keeps both units inside one string, and has no yield
control; a Czech site has a real integer stepper that scales the ingredient list
but leaves the method text and the print card at the authored yield, and attaches
the nutrition panel to the same controller.

## Decision

1. **A hand-written table, no library.** One `units` module holds, per unit id: the
   dimension, the factor to the base unit, a flag for eighth-based display, a flag
   for vague units, and the display name in four plural categories per language.
   `Intl.PluralRules` selects the category; `Intl.NumberFormat` formats the number.
   Unit names are never taken from `Intl`.
2. **Kitchen measures are units with fixed factors**: `lžička` 5 ml, `lžíce` 15 ml,
   `hrnek` 250 ml (not a US cup, 236.6 ml), `špetka` ≈ 1 g and vague. A US
   tablespoon is 14.79 ml, so an English rendering of `lžíce` is a conversion and
   is marked approximate; it is never aliased.
3. **The authored amount wins.** An instance stores `amount` and `unit`, plus an
   optional authored alternate. An authored alternate is rendered verbatim and
   suppresses conversion. Conversion fills only what is missing, and every
   converted value is marked approximate. Ingredient density makes this necessary:
   one published recipe has 3 tbsp = 19 g of dried milk, 12 tbsp = 170 g of butter
   and 4 cups = 454 g of sugar — no factor table can invent those.
4. **Rounding is display-only, by magnitude.** Exact SI is stored and summed;
   displays round. Mass and volume: `<10` → 0.5, `<50` → 1, `<200` → 5, `<1000` →
   10, otherwise 25. Spoon and cup displays round to eighths. A spoon display is
   used only when it stays within 5 % and 0.5 g of the exact amount; otherwise the
   amount is shown in grams and marked approximate. Rounding never produces zero.
5. **Approximation is per quantity.** `asi` / `about` (or `~`) marks a value that is
   not the authored one. There is no blanket "values are approximate" label on a
   card, because both surveyed sites use one to cover a real arithmetic error.
6. **Counts and eggs.** Counts scale only to an integer or an exact half: `3`, or
   `2 ½` with an explicit note that this splits an egg. Any other fraction makes
   the recipe not scalable at the requested yield, and the UI says so instead of
   showing `1.6 eggs`.
7. **Ranges scale at both ends and stay ranges.** Rounding goes outward, so the
   scaled recipe never promises less than the original.
8. **Salt and leavening scale linearly, marked approximate, with a taste
   warning.** No silent 0.75 factor — a reader who asked for double gets double,
   and is told to taste. Leavening is never rounded up. An amount below a
   measurable spoon is shown in grams, not collapsed into a pinch.
9. **What never scales**: fixed instances, unquantified lines (they stay visible as
   authored, no number invented), pan dimensions and item sizes, temperatures,
   method times, yeast unless the author marks the instance scalable, and
   nutrition.
10. **One yield base.** The authored yield is the only base. Prose like "two dozen"
    or "4 1/2 cups" is text, never a second base — the surveyed recipe disagrees
    with itself precisely because it has three.
11. **The method never carries its own amount.** A quantity that appears in a step
    is a reference to the ingredient instance, not a repeated number. The surveyed
    Czech recipe hardcodes six spoons in step eight, so its scaler lies.
12. **Refusal is visible.** When a yield cannot be served safely, nothing is
    scaled, the page shows the authored yield, and one line names the constraint.
    A scaler is never hidden on a recipe that has no step-to-instance assignment;
    it is absent, and the recipe says so.

## Verification

`mise run verify` carries a `unit` layer:

- **Errors** (block): a unit id outside the closed list, a non-positive or
  non-numeric amount, a mass/volume dimension mismatch on an authored alternate.
- **Warnings** (never block): `unit-suspect` when an authored primary and alternate
  diverge by more than 15 %, and when a mass/volume pair is implausible against the
  ingredient reference table (grams per millilitre, grams per piece).
- **Fixtures**: literal expectations per rounding band, eighth rounding, range
  scaling, egg refusal, sub-spoon salt, and an authored alternate that must not be
  converted.

## Consequences

- Four plural strings per unit per language must be authored and reviewed; a wrong
  string is not caught by `Intl.PluralRules`, only by reading.
- Cross-dimension gaps stay empty; a grams-only recipe grows no cup side.
- English "cup" for `hrnek` is approximate by about 5 %, and the UI says so.
- The refusal path is a first-class UI state, not an error page.

## Rejected

- Any unit library, and `Intl.NumberFormat` unit names as the source of Czech words.
- A single factor per ingredient for mass ↔ volume.
- Rounding each line before summing.
- Silent sublinear scaling of salt or leavening.
- Decimal eggs, and a blanket approximation label.
- Deriving the scaler base from prose yield text.
