# ADR-0013: Reading and cook mode rules

- Status: Accepted, with one rule pending device verification
- Date: 2026-10-26
- Decided in: ticket *Reading and cook mode prototype* (#6), from `prototypes/cook-mode/`

## Context

A recipe has to be readable standing up and cookable with wet hands. The prototype
renders one document as a phone and a tablet screen, in three modes, in two themes,
and measured what could be measured: token contrast, touch targets and how the
passive time of a braise occupies the timeline.

Three colour pairs failed the first pass and were changed: light `ink-3` at 3.60:1,
light `oven` at 3.85:1 and dark `ink-3` on the raised surface at 4.28:1. After the
change all 18 pairs clear 4.5:1 with the lowest at 4.60.

The decisive observation is structural, not visual: 90 of Svíčková's 123 minutes
are oven time. A paper recipe cannot show that, so the screen must.

## Decision

1. **Phone reading grammar is the paragraph card list**, one step per card with the
   consumed amounts as chips, and the ingredient block collapsed above it. The
   two-column phase rail from ADR-0011 stays a print and tablet rule, because a
   44 mm amount column on a 390 px screen takes half the width. This rule is
   provisional until the device test in ticket *Reading grammar on a phone,
   verified on a device* says otherwise; the fallback is the rail behind a toggle.
2. **The yield control is a reading-mode affordance.** Cook mode hides the stepper;
   the reader sets the yield before cooking. Changing it there never silently
   re-measures a step the cook already weighed. Step position and remaining time
   survive a round trip to reading mode.
3. **Timers are per step and multi-stage.** A step renders its `timers[]` list from
   ADR-0006, running one at a time with a visible label; a step with only
   `time_minutes` renders a single timer. Passive steps say who is working and for
   how long.
4. **Tablet uses two panes from 834 px** (ingredients left, current view right) and
   three panes from 1024 px. One column below 834 px.
5. **The timeline is a first-class view**, not a debug screen: it answers "when am I
   free" and it is where the you/oven split is legible.
6. **Every interactive control is at least 44 px** in both dimensions, 52 px for the
   primary action, and the primary action sits in the bottom third of the screen.
7. **Themes are designed twice.** Dark is not an inversion of light; each theme's
   pairs are measured independently.
8. **Step changes announce themselves**: the step heading is an `aria-live` region
   and progress is a `progressbar` with real values.

## Verification

The screen layer of `verify` asserts:

- Every token pair used by a component meets 4.5:1 in both themes, computed from the
  token file, not sampled from a screenshot. A new token without a measured pair
  fails.
- Every interactive element reports a box of at least 44 × 44 px at 390 px width.
- Changing the step updates an `aria-live` region and keeps focus on the control
  that was pressed.
- The cook mode exposes no yield control, and reading mode does.
- A step with `timers[]` renders one timer per entry; a step with only
  `time_minutes` renders exactly one.

## Consequences

- The screen and print grammars diverge deliberately, and the divergence is written
  down instead of argued per component.
- The yield control has two homes, so its state has to live above both.
- A device test is still owed for rule 1; until it runs, the paragraph card list is
  the default and the rail is reachable by toggle.

## Rejected

- Inverting the light palette for dark.
- One countdown per step when the document declares several stages.
- A 36 px control anywhere in the cooking surface.
- Three panes on a tablet in portrait.
- Changing the yield mid-cook without telling the cook.
