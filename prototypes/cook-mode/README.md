# Reading and cook mode prototype

One self-contained HTML file: phone and tablet frames, light and dark tokens,
three modes, two recipes, no dependencies.

Open `index.html`, or use a review link:

- `?mode=read&device=phone&recipe=svickova`
- `?mode=cook&recipe=svickova&step=3` — the oven step, 90 min
- `?mode=timeline&recipe=svickova` — you/oven lanes over 123 min
- `?mode=cook&device=tablet&recipe=mornay-sauce`

Chrome at the top (device, mode, theme, recipe) is scaffolding, not product.

## Measured

Contrast was computed, not judged: every text/background pair used by the screen
was run through the WCAG formula in both themes. Two pairs failed on the first
pass and were changed:

| Pair | First pass | Shipped |
|---|---|---|
| light `ink-3` on `bg` (`#8b8175`) | 3.60 | `#726a60` → 5.02 |
| light `oven` on `oven-soft` (`#b4573f`) | 3.85 | `#9e4d37` → 4.71 |
| dark `ink-3` on `surface-2` (`#8d8274`) | 4.28 | `#958a7c` → 4.76 |

Final: 18 of 18 pairs at or above 4.5:1, lowest 4.60.

Touch targets: 44 px minimum for stepper, mode toggle and timer, 52 px for the
primary action. Timeline totals are computed from step durations (Svíčková: 123
min, of which 90 is oven time).

## Decisions this prototype forces

1. **Default layout per recipe shape.** The read view is paragraph cards with a
   collapsible ingredient block. Above ~8 steps (Svíčková has 7) the rail may be
   better on a phone; ADR-0011 already sets that rule for print, and the screen
   should probably follow it rather than diverge.
2. **What a serving change does while cooking.** The prototype re-renders and shows
   a toast; it does not keep a "cooking at 4 portions, now showing 6" state. If the
   reader changes yield mid-cook, either the timer state must survive (it does, by
   step index) or the control must lock during cook mode.
3. **Where the hands are.** The primary action sits at the bottom in the thumb
   zone; the timer sits above it and stays reachable without scrolling. The step
   text is 1.06 rem at 1.55 line height, which is one glance, not two.
4. **Timers** currently come from `minutes` per step. ADR-0006 allows multi-stage
   `timers[]`, so the real cook mode needs a timer list per step, not one scalar.
5. **Passive time is the product.** The oven step shows "the oven is working for
   you — 90 min" and the timeline makes the free window visible. That is the
   feature a paper recipe cannot have, so it should drive the cooking-mode layout.
6. **Tablet is not a big phone.** Two panes (ingredients left, steps right) beat a
   centred phone column; the reference product uses three (mise en place, table,
   step) and the third pane is the step you are reading, not a table.
7. **Dark theme is not a tint.** All tokens are re-picked, not inverted; contrast
   was measured separately because an inverted palette fails quietly.

## Known limits

- No screen-wake, no haptics, no audio cue on timer end — all platform calls.
- Timer state is per page load; a real implementation needs persistence.
- The schematic layout (fourth grammar) is not in this prototype.
- English is not wired here; the print prototype carries the bilingual data path.
