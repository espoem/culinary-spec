# ADR-0005: Ingredient identity and canonical food registry

- Status: Accepted
- Date: 2026-10-25
- Decided in: ticket *Ingredient identity and canonicalisation*, after research ticket *Structured recipe languages and the ingredient identity they imply* (#16)

## Context

A bilingual CZ/EN cookbook with composition depth ≤ 2, allergen derivation,
and `unit-suspect` verification requires a durable ingredient identity. Previous
recipe formats fail this requirement under our constraints:

- Cooklang and RecipeMD tie identity to literal tokens or typographical strings
  (`@cibule` vs `@cibuli`, `- *1* onion`). Under Czech 7-case inflection, `cibule`,
  `cibuli`, and `cibulí` become three disconnected entities, bilingual linking
  (`hladká mouka` = `all-purpose flour`) is impossible without an external index,
  and cross-recipe composition cannot aggregate amounts.
- Open Recipe Format (ORF) restricts ingredient keys to ASCII (`^[a-zA-Z ]*$`),
  breaking immediately on Czech diacritics (`č`, `ř`, `á`, `í`, `ě`), and relies on
  pre-computed yield arrays rather than algorithmic scaling.
- Relational self-hosted tools (Mealie, Tandoor) use database primary keys (UUIDs or
  auto-increment IDs) with trigram matching or case-insensitive name comparison.
  For a static-first Git repository, database UUIDs add unnecessary coordination
  overhead, while trigram heuristics risk false merges across inflected Slavic words.
- Runtime NLP lemmatizers (e.g. morphological taggers) introduce large, non-deterministic
  runtime dependencies that violate static generation principles and still fail on
  culinary edge cases and loanwords.

We need a static, Git-versioned identity model that supports Czech inflection and
bilingual display, enables safe aggregation across sub-recipes, derives allergens
deterministically, and never blocks drafting on unknown foods.

## Decision

1. **Canonical registry in Git (`data/foods.yaml`) & Allergen Registry (`data/allergens.yaml`).**
   The food registry lives in a single versioned YAML file at the repository root (`data/foods.yaml`).
   Food entries use stable, human-readable kebab-case English slugs (`onion`, `flour-all-purpose`, `butter`).
   No opaque UUIDs. The file is compiled at build and verify time into a lookup index.
   Renames and merges use an explicit `deprecated_in_favor_of: <id>` pointer, flagged by `mise run verify`.
   Allergens are authoritatively defined in `data/allergens.yaml` with the 14 EU 1169/2011 tokens
   and bilingual labels (`cs`, `en`).
2. **Food schema.** Each entry in `data/foods.yaml` contains:
   - `id`: unique kebab-case slug.
   - `origin`: `'plant' | 'animal' | 'fungi' | 'mineral' | 'synthetic'`. Mandatory biological
     classification used for deterministic diet verification (ADR-0006, ADR-0010).
   - `animal_product` (optional): `'meat' | 'poultry' | 'fish' | 'crustacean' | 'mollusc' | 'dairy' | 'egg' | 'honey'`.
     Mandatory when `origin == 'animal'` to enable precise vegetarian vs vegan enforcement without
     resorting to supermarket aisle classifications.
   - `default_dimension`: `mass`, `volume`, or `count`.
   - `reference_density_g_per_ml` (optional) and `reference_piece_g` (optional):
     used strictly for `unit-suspect` verification warnings (ADR-0004), never for
     silent automatic cross-dimension conversions.
   - `allergens`: subset of canonical EU 1169/2011 allergen token IDs defined in `data/allergens.yaml`:
     `gluten`, `crustaceans`, `eggs`, `fish`, `peanuts`, `soybeans`, `milk`, `nuts`, `celery`,
     `mustard`, `sesame`, `sulphites`, `lupin`, `molluscs`.
   - `names.cs`: four plural categories (`one`, `few`, `many`, `other`) for
     countable items, or a single nominative string for mass/volume items.
   - `names.en`: two plural forms (`one`, `other`).
   - `aliases.cs` and `aliases.en`: list of explicit inflected forms and synonyms
     used for import matching and search lookup. In the search index, all plural forms
     from `names.cs`, all forms from `names.en`, plus `aliases.cs` and `aliases.en`
     are pre-folded and indexed into top-level `food_aliases` for word-boundary matching.
   - `deprecated_in_favor_of` (optional): target food `id`.
   - Supermarket aisle and shopping metadata remain strictly omitted.
3. **Czech inflection and display.** No runtime NLP lemmatizer.
   - For countable items (`unit: piece`), `Intl.PluralRules` selects the appropriate
     form from `names.cs` (`1 cibule`, `2 cibule`, `5 cibulí`).
   - For measured units (`g`, `ml`, `lžíce`, `hrnek`), UI and print layouts render
     the nominative string (`names.cs.one`) separated from the unit and quantity,
     or use an optional authored `display_name` on the instance. Runtime genitive
     inflection is never performed.
   - Import and CLI tools resolve ingredient strings via exact lookup against
     `names` and `aliases` in `data/foods.yaml`.
4. **Unknown ingredients never block publication.** An ingredient instance on a recipe
   treats `food` as optional (`food: null` or omitted), providing an inline bilingual
   fallback `name: { cs, en }`. Recipes in `draft` and `unverified` states publish
   freely with unlinked ingredients. The `reviewed` state requires every instance to
   resolve to a valid, non-deprecated food `id` in `data/foods.yaml`.
5. **Composition aggregation across sub-recipes.** When rolling up ingredients across
   a composed recipe (depth ≤ 2), instances group by `food` id (or normalized inline
   name if unlinked) and sum strictly within the same physical dimension (mass in `g`,
   volume in `ml`, counts in `piece`). Incompatible dimensions (e.g. `250 g` + `2 lžíce`
   of butter) never convert via density; they remain separate sub-lines under the same
   ingredient. Unquantified instances (`podle chuti`) attach as notes.
6. **Allergen derivation and detected sets.** A recipe's allergens are derived at build
   time as the union of `allergens` from all linked food entries and referenced sub-recipes
   against `data/allergens.yaml`. A recipe may author explicit `allergens_add` (for unlinked
   ingredients or processing contaminants) or `allergens_remove` (for culinary elimination,
   with mandatory note). Crucially, to eliminate safety loopholes in diet claims (ADR-0006,
   ADR-0010), build verification additionally computes `allergens_detected` as the raw,
   non-removable union of allergens across all linked ingredients and components before any
   `allergens_add` or `allergens_remove`. Diet validation gates (such as `vegan`, `vegetarian`, `gluten-free`, or `dairy-free`)
   evaluate `forbid_allergens` strictly against `allergens_detected`, `origins_detected`, and `animal_products_detected`. The single documented
   exception is `lactose-free`, which evaluates `forbid_mandatory_allergens` against `allergens.mandatory` +
   `allergens.optional`, permitting clarified butter/ghee where milk solids are removed with an explicit
   culinary reason in `allergens_remove` (ADR-0006, ADR-0010).
7. **Authoring format boundary.** Each recipe lives in its own directory
   (`recipes/<slug>/recipe.yaml`) as pure YAML pairing structured ingredient
   instances (`food`, `amount`, `unit`, `step: <step-id>`) with steps carrying
   shared metadata (`id`, `time`, `temp`) and parallel prose methods (`prose.cs`,
   `prose.en`).

## Verification

`mise run verify` validates the food registry, allergen tokens, and recipe references:

- **Integrity Failures** (Exit 2):
  - `ALLERGEN_REGISTRY_PRESENT`: `data/allergens.yaml` missing, empty, or unparseable.
- **Errors** (block, Exit 1):
  - `FOOD_DUPLICATE_ID`: Duplicate food `id` in `data/foods.yaml`.
  - `FOOD_SCHEMA_VALIDATION`: Schema violations in `data/foods.yaml`.
  - `FOOD_ORIGIN_MISSING`: Food entry missing mandatory `origin` field (`plant`, `animal`, `fungi`, `mineral`, `synthetic`).
  - `ALLERGEN_CANONICAL_TOKEN`: Unknown allergen tokens outside authoritative 14 EU tokens in `data/allergens.yaml`.
  - `FOOD_DEPRECATION_ACYCLIC`: Cyclic `deprecated_in_favor_of` chains in `data/foods.yaml`.
  - `FOOD_DEPRECATION_TARGET`: Dangling target in `deprecated_in_favor_of`.
  - `FOOD_REGISTRY_LINK`: Unresolved `food` references or references to deprecated food IDs in any recipe
    with `state: reviewed`.
- **Warnings** (never block):
  - `unit-suspect`: instance unit dimension disagrees with food `default_dimension`,
    or divergence >15% against reference density or piece weight.
  - Unlinked ingredient instances (`food: null`) in `draft` or `unverified` recipes.
  - References to deprecated food IDs in `draft` or `unverified` recipes.
- **Representative Mutants (see ADR-0008 §5 for the complete 54-mutant matrix)**:
  - Food entry missing mandatory biological origin (`mutant-food-origin-missing`).
  - Deliberately broken food reference in a reviewed fixture (`mutant-unlinked-reviewed`).
  - Circular deprecation loop in food registry fixture (`mutant-food-cycle-deprecation`).
  - Invalid allergen token fixture (`mutant-food-unknown-allergen`).

## Consequences

- Ingredient identity is completely decoupled from natural language prose and runtime
  linguistic guessing.
- Inflected Czech aliases must be explicitly authored in `data/foods.yaml` rather
  than guessed at runtime.
- Recipe aggregation across sub-recipes remains mathematically sound and free from
  erroneous density rounding.
- Drafting and rapid iteration remain frictionless: new ingredients do not gate
  `draft` or `unverified` publishing.

## Rejected

- Shopping lists, store aisle grouping, and supermarket categories.
- Runtime NLP lemmatization or grammatical case inference.
- Opaque UUIDs or database IDs as canonical food keys.
- Inline prose DSLs (Cooklang) and typography-based parsing (RecipeMD).
- ASCII-only ingredient keys (Open Recipe Format).
- Silent automatic cross-dimension conversions via density tables.
- Blocking `draft` or `unverified` publication on unknown foods.
