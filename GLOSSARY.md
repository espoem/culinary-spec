# GLOSSARY

Domain vocabulary for Culinary Spec. Czech and English terms are both canonical:
the site is bilingual, so a term that exists in only one language is a modelling
bug. No implementation detail belongs here.

## Content

**Recept** / *recipe* — one cookable unit a reader can follow to a finished dish.
Has a yield, a difficulty, tags, allergens, nutrition and a publication state.

**Receptní dokument** / *recipe document* — the source form of a recipe in the
repository: a structured ingredient table plus written methods, one per language.
It is the thing authors edit and everything else is derived from it.

**Postup** / *method* — the written prose a cook reads. There are two, CZ and EN,
authored natively in parallel. Neither is a translation of the other.

**Krok** / *step* — one instruction inside a method, with a label, the ingredient
instances it consumes, and optionally a duration or target temperature.

**Přípravná úloha** / *prep task* — work that must happen before or alongside the
steps but is not tied to an ingredient (preheat, grease a pan, start rice).

**Surovina** / *ingredient* — the abstract pantry item, language-neutral, with an
identity used for the shopping list (`cibule` = `onion`).

**Instance suroviny** / *ingredient instance* — one line in one recipe: an
ingredient, an amount, a unit, an optional preparation note, and the step it
enters. The same ingredient may appear as several instances in one recipe.

**Dávka** / *yield* — what the recipe produces, with its unit (`1 loaf`,
`4 porce`, `900 g`). Scaling changes the yield, never the identity.

**Přepočet porcí** / *scaling* — recomputing amounts for a different yield.
Each instance scales linearly, stays fixed, or counts as an indivisible piece.

**Mise en place** / *mise en place* — the grouped checklist of prepared
ingredients a cook assembles before heat; a view over instances, not new data.

## Structure

**Kompozice** / *composition* — a recipe assembled from components. Depth is
bounded; a component is referenced live, not pinned to a version.

**Komponenta** / *component* — a sub-recipe used inside another recipe.

**Varianta** / *variant* — a different assembly of the same or different
components. A variant changes ingredients or quantities; if the steps change, it
is a new recipe with a parent.

**Matice** / *recipe table* — the ingredient × step grid: ingredients as rows,
steps as spanning cells. Generated from the step-to-instance assignment when
those spans nest without crossing.

**Fázová lišta** / *phase rail* — the two-column fallback: numbered phases with
amounts on the left and prose on the right. Used when a table would be too wide
to read or to print.

**Dráha** / *lane* — a parallel track over time (you / oven), the mobile
projection of the same step data.

## Provenance and lifecycle

**Provenance** / *provenance* — for every recipe and image: licence, source URL,
retrieval date, and the digest of the source when one exists.

**Stav publikace** / *publication state* — `draft`, `unverified` (live, not
indexed, badged), `reviewed` (human-checked against its source, indexable).

**Kolekce** / *collection* — a reader-assembled list of recipes for one bulk
print or one shopping trip.

## Names that must not be confused

- **Slovník pojmů** / *site glossary* — the reader-facing dictionary page.
- **`GLOSSARY.md`** / *repo glossary* — this file, for people and agents working
  on the project.
- **Krok** vs **přípravná úloha** — a step consumes ingredients, a prep task does
  not.
- **Surovina** vs **instance suroviny** — identity vs one line in one recipe.
