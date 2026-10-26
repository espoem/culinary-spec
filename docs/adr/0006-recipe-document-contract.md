# ADR-0006: Recipe document and provenance sidecar contract

- Status: Accepted
- Date: 2026-10-25
- Decided in: ticket *Recipe document contract* (#4), after research (#16) and three-axis adversarial challenge

## Context

A recipe in Culinary Spec is a bilingual, portion-scalable, accessible technical document. It must render into four distinct projections: a responsive reading view, an interactive cooking mode with live timers, an accessible paged A4 print sheet, and a "Cooking for Engineers" matrix table where steps span the ingredient rows they consume.

Previous recipe formats fail under these simultaneous requirements:
- **Markdown front-matter + prose** (e.g. Hugo, Jekyll, Astro content collections) divides the recipe into two disconnected worlds. Ingredients live in YAML front-matter, while method instructions live in free-form Markdown paragraphs. In a bilingual system, parallel prose (`cs` and `en`) either requires splitting into two files (`recipe.cs.md`, `recipe.en.md`) or embedding dual language headers. In both cases, referential integrity breaks: an ingredient instance cannot reliably bind to a step ID across languages, structural language parity (ADR-0002: identical step counts, durations, temperatures, and ingredient bindings) cannot be enforced by a deterministic schema, and matrix column spans cannot be verified without parsing Markdown ASTs.
- **Prose DSLs** (Cooklang, RecipeMD) embed ingredient definitions and quantities directly within sentence tokens (`@butter{30%g}`, `~{2%minutes}`). As proven in research (#16) and ADR-0005, inline tokenization fails on Czech 7-case inflection (`cibule` vs `cibuli`), destroys bilingual translation parity, prevents structured nutrition and allergen derivation, and risks silent drift when quantities are edited in prose but missed in tables.

To solve this, ADR-0001 established that the recipe document is a structured ingredient table paired with written methods, and ADR-0005 confirmed that each recipe lives in its own directory (`recipes/<id>/recipe.yaml`).

Before freezing the document schema, a **three-axis adversarial challenge** (covering schema edge cases, matrix geometry failure modes, and verification loopholes) was executed against the draft specification. The challenge exposed ten critical domain problems that required rigorous contract solutions:

1. **Intermediate step products in matrix geometry**: Naive matrix generators assume steps only consume raw ingredient instances. In actual cooking, subsequent steps consume intermediate products of prior steps (e.g. a roux consumes butter and flour, then a béchamel consumes the roux and warm milk). If step consumption of prior steps is untracked, intermediate steps collapse or become disconnected in the matrix, breaking row span continuity.
2. **Temporal anchoring of background tasks (`prep_tasks`)**: Prep tasks (preheating an oven, chilling meat, or soaking beans) are not tied to specific ingredient instances, but they cannot float unanchored in time. They must be explicitly anchored before the step where their output is needed (`before_step`), determining their exact row placement as full-width spanning cells in the matrix and phase rail.
3. **Component yield dimension bridges**: When a recipe references another recipe as a `component`, the sub-recipe's native yield might be in portions (`4 porce`), while the parent recipe requires a physical mass (`300 g`). Without an explicit physical bridge on the component's yield definition (`alternate`), cross-recipe aggregation requires guessing densities or fails verification.
4. **Zero-token prose**: Any interpolation syntax inside step prose (`{butter}`, `~{5%min}`) creates parser complexity, translation friction, and inflection bugs. Step prose must be 100% plain, human-authored text. Consumed ingredient chips, temperatures, and timers are rendered by the UI and print layouts directly from the step's structured bindings, never parsed from prose.
5. **Per-language publication state**: A recipe may be reviewed and ready in Czech, but still draft or unverified in English. A single top-level state would either prematurely expose unreviewed English content or prevent Czech readers from accessing a completed recipe. Publication state must be tracked independently per language.
6. **Orthogonality of `unit` vs `scaling`**: Indivisibility is a property of countable units (`piece`), not a scaling law. Mixing "indivisible piece" into the scaling enum creates ambiguity when an item is both an indivisible count and fixed (e.g. 1 bay leaf or 1 egg for an egg wash). Unit indivisibility belongs in the unit table; `scaling` specifies mathematical behavior (`linear`, `fixed`, `taste`, `leavening`).
7. **Multi-stage step timers**: Cooking instructions frequently require sequential heat/time phases within a single step (e.g. sear on high for 3 minutes, then braise covered on low for 45 minutes; or blind bake with weights for 15 minutes, then without weights for 10 minutes). A scalar `time_minutes` / `temp_c` pair cannot drive cook-mode timers for multi-stage steps.
8. **Variant-level laminarity and allergens**: A recipe variant that substitutes, adds, or removes ingredients alters the topological row order and allergen profile of the dish. Matrix laminarity and allergen derivation cannot be evaluated only on the base recipe; they must be verified independently on the materialized document of every variant.
9. **Deterministic CI provenance verification with `exit 2`**: Copyright law protects original recipe expression while allowing culinary formulas to be re-expressed. ADR-0002 mandated provenance sidecars, but if CI passes vacuously when scrape data is omitted, verification is an illusion. Adapted recipes must include normalized source extracts and prose fingerprints in `provenance.yaml`; missing provenance files or extracts must terminate verification with exit code 2.
10. **Explicit nutrition basis and slug redirect aliases**: Nutrition tables must declare whether values are `per_serving` or `per_100g`. Furthermore, when recipe slugs are refined, static hosting requires an explicit list of slug aliases to emit deterministic 301 static redirect files, preventing broken inbound links.

---

## Decision

We define the complete, normative specification for recipe storage, schema structure, matrix geometry, variants, allergens, nutrition, and provenance sidecars.

### 1. Directory Layout

Every recipe occupies a dedicated directory under `recipes/`:

```
recipes/<id>/
├── recipe.yaml
├── provenance.yaml
└── [assets] (e.g. hero.webp, step-01.webp)
```

- `<id>` is an immutable, lowercase ASCII kebab-case identifier (`^[a-z0-9]+(-[a-z0-9]+)*$`).
- The directory name must exactly match the `id` field in `recipe.yaml`.
- Image assets referenced in `provenance.yaml` or UI metadata live alongside `recipe.yaml`.

---

### 2. Header, Languages, and Publication State

The root of `recipes/<id>/recipe.yaml` contains identity, categorization, and bilingual metadata:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique semantic recipe identifier (`^[a-z0-9]+(-[a-z0-9]+)*$`). Must match directory name (`recipes/<id>/`). Used for internal component references (`component: mornay-sauce`). |
| `uid` | `string` | Immutable 6-character lowercase Crockford Base32 identifier (`^[0-9a-hjkmnp-tv-z]{6}$`, excluding `i, l, o, u`, e.g. `k8m2x9`). Unique corpus-wide. Used in public URLs (`/<lang>/<section>/<uid>-<slug>/`), permanent shortlinks (`/<lang>/<section>/<uid>/`), and collection query parameters (`?r=<uid1>,<uid2>`). |
| `state` | `{ cs?: PublicationState, en?: PublicationState }` | Per-language publication state: `'draft'`, `'unverified'`, or `'reviewed'`. |
| `title` | `{ cs?: string, en?: string }` | Human-readable title per language. |
| `slug` | `{ cs?: string, en?: string }` | ASCII kebab-case URL slug per language. Unique corpus-wide per language. |
| `slug_aliases` | `{ cs?: string[], en?: string[] }` | Optional. Prior slugs that emit static 301 redirect pages during build. |
| `summary` | `{ cs?: string, en?: string }` | 1–2 sentence summary. Feeds recipe cards and JSON-LD `description`. |
| `published_at` | `string (YYYY-MM-DD)` | Initial publication date. Mandatory when any language has state `'reviewed'`. |
| `parent` | `string` | Optional recipe `id`. Used when this recipe is a structural fork with altered steps. |
| `category` | `string` | Controlled vocabulary for `schema.org/recipeCategory`. Must resolve to `data/taxonomy.yaml#categories` (`TAXONOMY_CATEGORY_UNKNOWN`). |
| `cuisine` | `string` | Optional controlled vocabulary for `schema.org/recipeCuisine`. Must resolve to `data/taxonomy.yaml#cuisines` (`TAXONOMY_CUISINE_UNKNOWN`). |
| `difficulty` | `'easy' \| 'medium' \| 'hard'` | Subjective kitchen difficulty rating. |
| `tags` | `string[]` | Array of controlled kebab-case tag IDs. Must resolve to `data/taxonomy.yaml#tags` (`TAXONOMY_TAG_UNKNOWN`). Enforce no duplicate IDs (`TAG_DUPLICATE_INSTANCE`) and capped at $\le 8$ tags per recipe for card layout density and chip visual budget (`TAXONOMY_TAG_LIMIT_EXCEEDED`). |
| `diets` | `string[]` | Optional array of controlled diet IDs. Must resolve to `data/taxonomy.yaml#diets` (`TAXONOMY_DIET_UNKNOWN`). Enforce no duplicate IDs (`TAG_DUPLICATE_INSTANCE`). |
| `search_aliases` | `{ cs?: string[], en?: string[] }` | Optional bilingual inflected title synonyms and colloquial search terms used for build-time search indexing (ADR-0010). |
| `table_layout` | `'auto' \| 'matrix' \| 'phase-rail'` | Optional layout override. Defaults to `'auto'`. If `'phase-rail'`, forces phase rail layout even if matrix is laminar. If `'matrix'`, explicitly requires matrix layout and fails verification with `MATRIX_LAMINARITY` (Exit 1) if step intervals cross. |

#### Publication State Rules:
- If a language key is omitted or set to `'draft'`, that recipe is completely excluded from the public static build of that language (ADR-0001).
- `'unverified'` recipes are included in the build, tagged with `noindex`, and carry a visual unverified badge.
- `'reviewed'` recipes are indexed and require complete verification: all ingredients must link to `data/foods.yaml`, `published_at` must be present, and `provenance.yaml` must carry human reviewer sign-off.
- **Structural parity rule** (ADR-0002): Any recipe with at least one language in `'unverified'` or `'reviewed'` must provide titles, slugs, summaries, and step prose for all languages whose state is not `'draft'`. Both languages must share identical step counts, step IDs, lane assignments, timings, temperatures, and ingredient bindings.

---

### 3. Yield and Timings

```yaml
yield:
  amount: 4
  unit: porce
  label:
    cs: porce
    en: servings
  alternate:
    amount: 500
    unit: ml

timings:
  prep_minutes: 15
  cook_minutes: 20
  rest_minutes: 10
```

- `yield.amount`: Non-zero positive number. This is the **single scaling base** (ADR-0004).
- `yield.unit`: Valid `UnitId` (e.g. `porce`, `piece`, `g`, `ml`, `loaf`).
- `yield.label`: Optional bilingual display override for the unit.
- `yield.alternate`: Optional physical bridge (amount + unit). **Mandatory** when this recipe is referenced as a `component` in another recipe using a dimension different from `yield.unit` (e.g. sub-recipe yield is `4 porce`, but parent recipe consumes `300 g`).
- `timings.prep_minutes`: Non-negative integer. Active prep time before heat.
- `timings.cook_minutes`: Non-negative integer. Active or monitored cooking time.
- `timings.rest_minutes`: Optional non-negative integer. Passive resting, cooling, or proofing time.
- **Total time is never stored**: JSON-LD `totalTime`, search indexing, and card badges compute `total_minutes = prep_minutes + cook_minutes + (rest_minutes ?? 0)` at build time. Background prep tasks (`prep_tasks`) and parallel passive lanes are excluded from this scalar sum.
- **Time ledger consistency (`TIME_LEDGER_MISMATCH`)**: If `timings.cook_minutes` is less than the sequential sum of active step `time_minutes` (or sequential `timers` durations), verification emits warning `TIME_LEDGER_MISMATCH`.

---

### 4. Ingredient Instances and Components (`ingredients`)

The `ingredients` block is an ordered sequence of ingredient instances and component inclusions.

```yaml
ingredients:
  - id: butter
    food: butter
    amount: 30
    unit: g
    scaling: linear
    step: roux
  - id: nutmeg
    food: nutmeg
    amount: 1
    unit: spetka
    scaling: taste
    optional: true
    step: finish
  - id: stock
    component: vegetable-stock
    amount: 250
    unit: ml
    step: simmer
```

Each ingredient instance must specify:
1. `id`: Lowercase ASCII identifier (`^[a-z0-9_-]+$`), unique within the recipe.
2. **Identity** (exclusively one of the following two):
   - **Canonical food**: `food: string | null`. Must resolve to a valid food ID in `data/foods.yaml`. Mandatory non-null when any language state is `'reviewed'`. If `food: null` (allowed in `draft`/`unverified`), `name: { cs: string, en: string }` is mandatory. An optional `name` override may be provided alongside a non-null `food` to specify culinary display nuances (e.g. "Dijon mustard").
   - **Sub-recipe component**: `component: string`. Target recipe `id`. Composition depth adheres to a 2-level hierarchy: a parent dish (level 0) may consume component recipes (level 1), and a component recipe cannot itself consume further components (max edge depth = 1). Component reference graphs must be strictly acyclic (`COMPONENT_DAG`). Components included in search facets and badges are restricted to those actually built in that language (`COMPONENT_BUILD_AVAILABILITY`).
3. `amount`: Positive `number`, two-element range `[number, number]`, or `null`. `amount` is `null` if and only if `unit` is `null` (representing unquantified items like "podle chuti" / "to taste").
4. `unit`: Valid `UnitId` from the canonical unit registry, or `null`.
5. `alternate`: Optional `{ amount: number | [number, number], unit: UnitId }`. Authored secondary measurement (ADR-0004), rendered verbatim and suppressing automatic unit conversions.
6. `scaling`: Scaling law enum:
   - `'linear'` (default): Multiplies directly by the scaling ratio $R = \text{target\_yield} / \text{base\_yield}$.
   - `'fixed'`: Does not scale when portions change (e.g. bay leaves, parchment paper, frying oil).
   - `'taste'`: Scales linearly, but UI flags the amount with an explicit taste warning.
   - `'leavening'`: Scales linearly, never rounded up, marked approximate with a leavening warning.
   - *Note on counts*: Indivisibility (e.g. eggs scaling only to integers or exact halves) is governed by `unit: piece` in the unit table, orthogonal to `scaling`. A recipe can specify `unit: piece, scaling: fixed` (e.g. 1 egg for egg wash) without ambiguity.
7. `optional`: `boolean`. Defaults to `false`. If `true`, this instance does not contribute to mandatory allergens and can be toggled in the UI.
8. `note`: Optional bilingual mise en place instruction `{ cs?: string, en?: string }` (e.g. "pokojová teplota" / "room temperature", "jemně nastrouhaný" / "finely grated").
9. `step`: Target `steps[k].id`. Mandatory unless the recipe explicitly sets `table_layout: 'phase-rail'` and has no scaling.

---

### 5. Prep Tasks (`prep_tasks`) and Steps (`steps`)

#### Prep Tasks (`prep_tasks`)
Prep tasks describe prerequisite actions that do not consume ingredient instances (e.g. preheating an oven, lining a cake tin, chilling equipment):

```yaml
prep_tasks:
  - id: preheat-oven
    before_step: bake
    temp_c: 180
    time_minutes: 15
    prose:
      cs: Troubu předehřejte na 180 °C a plech vyložte pečicím papírem.
      en: Preheat oven to 180 °C and line a baking sheet with parchment paper.
```

- `id`: Unique identifier across all steps and prep tasks.
- `before_step`: Step `id` before which this task must be ready. Defaults to `steps[0].id`. In matrix and phase-rail views, this anchors the full-width `prep-cell` immediately above `before_step`.
- `temp_c`: Optional integer target temperature in degrees Celsius.
- `time_minutes`: Optional positive integer or range `[min, max]`.
- `prose`: Bilingual instruction text. Plain prose only.

#### Steps (`steps`)
`steps` is a non-empty ordered array representing the sequential execution of the recipe:

```yaml
steps:
  - id: roux
    phase:
      cs: Základ
      en: Base
    lane: active
    time_minutes: 2
    temp_c: 120
    short:
      cs: máslo + mouka, 2 min
      en: butter + flour, 2 min
    prose:
      cs: V rendlíku rozpusťte máslo, přisypte mouku a míchejte do světlé jíšky.
      en: Melt butter in a saucepan, add flour, and whisk into a pale roux.

  - id: bechamel
    phase:
      cs: Bešamel
      en: Béchamel
    lane: active
    consumes: [roux]
    timers:
      - duration_minutes: 2
        label: { cs: "Přilévání", en: "Pouring" }
      - duration_minutes: 5
        label: { cs: "Prováření", en: "Simmering" }
    prose:
      cs: Postupně přilévejte teplé mléko a provařte do zhoustnutí.
      en: Gradually whisk in warm milk and simmer until thickened.
```

- `id`: Unique identifier across `steps` and `prep_tasks`.
- `phase`: Optional bilingual grouping header `{ cs?: string, en?: string }`. Steps sharing the same phase header are visually grouped in the phase rail and print layouts.
- `lane`: Execution lane: `'active'` (hands-on) or `'passive'` (oven, proofing, resting). Defaults to `'active'`.
- `consumes`: Array of prior step IDs whose outputs flow into this step.
  - Defaults to `[steps[i-1].id]` for $i > 0$, and `[]` for $i = 0$.
  - Explicit `consumes: []` on $i > 0$ denotes the start of an independent parallel branch.
  - Explicit `consumes: [stepA, stepB]` denotes the merge of two branches.
- **Timing and Heat (mutually exclusive)**:
  - **Scalar**: `time_minutes?: number | [number, number]` and/or `temp_c?: number`.
  - **Multi-stage sequential timers**: `timers?: { duration_minutes: number | [number, number], temp_c?: number, label?: { cs?: string, en?: string } }[]`. Mutually exclusive with top-level `time_minutes`. Drives the cook-mode countdown engine.
- **Short cue (`short`)**: Optional bilingual label `{ cs?: string, en?: string }`, at most 48 characters per language, empty or absent when the step is self-evident. Used by the matrix cells, recipe cards and any surface too small for full prose. It must be authored, never truncated from `prose`: a truncated sentence overflows the A4 sheet and reads as broken (measured in ticket #5). `STEP_SHORT_FORMAT` fails when a present value exceeds 48 characters or is empty; `STEP_SHORT_MISSING` warns when `table_layout` resolves to `matrix` and a step has no short cue.
- **Zero-Token Prose**: `prose: { cs: string, en: string }`. Step prose must be 100% natural language prose. No `{...}` tokens, no `@ingredient` references, and no hardcoded numeric quantities. Consumed ingredient pills/chips are rendered by user interfaces and print layouts directly from the step bindings (`I.step == S.id`).

#### Matrix Geometry & Laminarity Formula:
The matrix layout is a bipartite incidence grid between ordered ingredient rows and step column spans. For any step $S$, its consumed row set $\text{rows}(S)$ is defined deterministically as:

$$\text{rows}(S) = \{ \text{row}(I) \mid I \in \text{ingredients}, I.\text{step} = S.\text{id} \} \cup \bigcup_{P \in S.\text{consumes}} \text{rows}(P)$$

1. **Non-empty check**: Every step must consume at least one ingredient instance or one prior step: $\text{rows}(S) \neq \emptyset$.
2. **Topological row ordering**: Ingredient rows are ordered by the index of the first step that consumes them, preserving authoring order within ties: $\text{order}(I) = (\min \{ \text{index}(S) \mid \text{row}(I) \in \text{rows}(S) \}, \text{instance\_index}(I))$.
3. **Laminarity check**: Under this row ordering, for every step $S$, $\text{rows}(S)$ must form a **contiguous integer interval** $[\min(\text{rows}(S)), \max(\text{rows}(S))]$. Spans must nest or sit side-by-side; they must never cross. If spans cross: when `table_layout: 'auto'` (default), the generator emits a build warning and falls back to `phase-rail` (ADR-0003); when `table_layout: 'matrix'` is explicitly authored on a recipe with crossing spans, verification fails immediately with `MATRIX_LAMINARITY` (Exit Code 1).

---

### 6. Variants (`variants`)

A variant represents an alternative formulation of the **same procedure** (e.g. gluten-free flour substitution, dairy-free milk, cheddar substitution). If the cooking steps diverge fundamentally, it is not a variant; it must be authored as a separate recipe with a `parent` reference (ADR-0001).

```yaml
variants:
  - id: cheddar-mornay
    title:
      cs: Čedarová varianta
      en: Sharp Cheddar Variant
    replace:
      - target: gruyere
        instance:
          food: cheese-cheddar
          note:
            cs: vyzrálý čedar, jemně strouhaný
            en: sharp aged cheddar, finely grated
    add:
      - id: dijon
        food: mustard-dijon
        amount: 5
        unit: g
        scaling: taste
        optional: true
        step: finish
    allergens_add:
      - mustard
    prose_overrides:
      finish:
        cs: Do teplé omáčky vsypte nastrouhaný čedar a hořčici. Míchejte do rozpuštění.
        en: Fold grated cheddar and mustard into the warm sauce. Stir until melted.
```

- `id`: Lowercase ASCII kebab-case variant ID.
- `title`: Bilingual variant title.
- `replace`: Array of patches modifying existing instances (`target` matches `ingredients[].id`).
- `add`: Array of new `IngredientInstance` objects to append.
- `remove`: Array of `id` strings of base instances to exclude.
- `prose_overrides`: Optional map of step IDs to revised bilingual prose.
- `allergens_add` / `allergens_remove`: Variant-specific allergen adjustments.
- **Verification**: For every variant, the compiler materializes the full effective recipe document and executes the complete verification suite (referential integrity, laminarity, scaling, and allergen checks) independently.

---

### 7. Allergens and Nutrition

#### Allergens, Diets, and Safety Invariants
A recipe does not require manual authoring of baseline allergens; they are derived deterministically:
1. **Mandatory allergens (`allergens.mandatory`)**: Union of EU 1169/2011 allergens from all non-optional linked `food` entries in `data/foods.yaml`, plus components' mandatory allergens, plus `allergens_add`, minus `allergens_remove`.
2. **Optional allergens (`allergens.optional`)**: Allergens introduced exclusively by instances marked `optional: true` (minus any that also appear in mandatory instances). Displayed strictly with the copy: *"Volitelná surovina obsahuje <alergen> — lze vynechat / Optional ingredient contains <allergen> — can be omitted"* (the legally loaded FIC phrase "may contain" is strictly prohibited).
3. **Allergen Union Rule**: If an allergen appears in both a mandatory instance and an optional instance, it is treated strictly as mandatory (excluded under `allergen_not`, with no optional badge).
4. **Non-Removable Detected Sets (`allergens_detected`, `origins_detected`, `animal_products_detected`)**:
   To prevent safety loopholes where `allergens_remove` or variants could launder an ingredient into a false diet claim (ADR-0010), the compiler derives:
   - `allergens_detected`: raw union of linked-food `allergens` across all base ingredients (both mandatory and optional) and components **before** any `allergens_add` or `allergens_remove`.
   - `origins_detected`: raw union of biological `origin` values (`plant`, `animal`, `fungi`, `mineral`, `synthetic`) across all linked foods.
   - `animal_products_detected`: raw union of `animal_product` values (`meat`, `poultry`, `fish`, `crustacean`, `mollusc`, `dairy`, `egg`, `honey`) across all linked foods.
5. **Allergen Removal Guard (`allergens_remove`)**: If `allergens_remove` is used, each entry must be an object specifying the allergen and a mandatory bilingual culinary reason:
   ```yaml
   allergens_remove:
     - allergen: milk
       reason:
         cs: Použito přepuštěné máslo (ghee) zbavené mléčných bílkovin.
         en: Clarified butter (ghee) used with milk solids removed.
   ```
   Even when `allergens_remove: [{ allergen: milk, reason: ... }]` is declared, `allergens_detected` retains `milk` and `animal_products_detected` retains `dairy`. `allergens_remove` affects recipe card display and `allergen_not` search filtering, but **never clears a `forbid_allergens` diet gate**. Any usage of `allergens_remove` requires explicit culinary rationale documented in `review.<lang>.notes`. Furthermore, whenever `allergens_detected` differs from `allergens.mandatory`, the rendered recipe card surfaces an explicit explanatory note to inform the reader.
6. **Diet Declarations (`diets`) and Conflict Verification (`DIET_ALLERGEN_CONFLICT`, Exit 1)**:
   - Any recipe declaring an entry in `diets` must have zero unlinked ingredients (`food !== null`), enforced by `DIET_UNLINKED_FOOD` (Exit 1).
   - Each diet in `data/taxonomy.yaml#diets` carries a mandatory bilingual disclaimer (`disclaimer: { cs, en }`, e.g. clarifying that the claim reflects recipe ingredients and does not guarantee absence of kitchen cross-contamination, enforced by `DIET_DISCLAIMER_PRESENT`, Exit 1).
   - Diet claims are evaluated deterministically against `allergens_detected`, `origins_detected`, and `animal_products_detected`. The single documented exception is `lactose-free`, which evaluates `forbid_mandatory_allergens` against `allergens.mandatory` + `allergens.optional`:
     - `gluten-free`: declares `forbid_allergens: [gluten]`, forbidding `gluten` in `allergens_detected` (mandatory and optional).
     - `dairy-free`: declares `forbid_allergens: [milk]` and `forbid_animal_products: [dairy]`, forbidding `milk` in `allergens_detected` and `dairy` in `animal_products_detected`.
     - `lactose-free`: declares `forbid_mandatory_allergens: [milk]`, forbidding `milk` in `allergens.mandatory` + `allergens.optional` (permitting clarified butter/ghee where milk solids are removed with an explicit culinary reason in `allergens_remove`, or lactose-free dairy substitutes).
     - `vegetarian`: declares `forbid_allergens: [fish, crustaceans, molluscs]` and `forbid_animal_products: [meat, poultry, fish, crustacean, mollusc]`.
     - `vegan`: declares `forbid_allergens: [milk, eggs, fish, crustaceans, molluscs]` and `forbid_origins: [animal]`.

#### Nutrition
Nutrition is optional, but when present, must adhere to a strict factual basis:

```yaml
nutrition:
  basis: per_serving
  kcal: 215
  protein_g: 9.2
  fat_g: 17.5
  carbs_g: 6.8
  fiber_g: 0.2
  salt_g: 0.8
  source: CIQUAL French Food Composition Table (code 11048)
  source_url: https://ciqual.anses.fr/#/aliments/11048
```

- `basis`: `'per_serving'` or `'per_100g'`.
- `serving_size`: Optional `{ amount: number, unit: UnitId, label?: { cs?: string, en?: string } }`. **Mandatory** if `basis == 'per_serving'` and `yield.unit !== 'porce'`.
- `kcal`, `protein_g`, `fat_g`, `carbs_g`: Required positive numbers.
- `fiber_g`, `salt_g`: Optional non-negative numbers.
- `source`: Mandatory string identifying the laboratory, official database, or published reference.
- `source_url`: Optional URL linking directly to the source database record.
- **No algorithmic rollup**: Nutrition is never calculated by summing raw ingredient values (ADR-0001).

---

### 8. Provenance Sidecar (`recipes/<id>/provenance.yaml`)

Every published recipe must have an accompanying `recipes/<id>/provenance.yaml` sidecar. This sidecar enforces copyright compliance, attribution, and re-expression verification.

```yaml
origin: adapted
license: CC-BY-SA-4.0
source_url: https://example.com/classic-french-mornay
source_title: Le Guide Culinaire - Sauce Mornay Fact Sheet
retrieved_at: "2026-10-25T12:00:00Z"
raw_scrape_sha256: 4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945
source_extract_sha256: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b

source_extract:
  quantities:
    - amount: 30
      unit: g
      food: butter
      raw: "30 g de beurre"
    - amount: 30
      unit: g
      food: flour-all-purpose
      raw: "30 g de farine"
    - amount: 500
      unit: ml
      food: milk-whole
      raw: "1/2 litre de lait"
    - amount: 1
      unit: piece
      food: egg
      raw: "1 jaune d'oeuf"
    - amount: 100
      unit: g
      food: cheese-gruyere
      raw: "100 g de gruyère râpé"
  prose_fingerprint:
    simhash: "0x8f2d4e1a9b3c5d7e"
    shingle_hashes:
      - "0x3a4f12"
      - "0x9c8b74"
      - "0x12ed5a"
    word_count: 145

review:
  cs:
    reviewed_by: Martin Šmíd
    reviewed_at: "2026-10-25T14:30:00Z"
    recipe_content_sha256: 4a2b6c8d1e3f5a7b9c0d2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    source_extract_sha256_at_review: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    notes: Verified proportions against reference; confirmed original Czech prose methods.
  en:
    reviewed_by: Martin Šmíd
    reviewed_at: "2026-10-25T14:45:00Z"
    recipe_content_sha256: 7f8e9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f
    source_extract_sha256_at_review: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    notes: Verified proportions against reference; confirmed original English prose methods.

images:
  - file: hero.webp
    origin: licensed
    license: CC-BY-SA-4.0
    source_url: https://commons.wikimedia.org/wiki/File:Sauce_Mornay.jpg
    phash: d4c3b2a10f9e8d7c
```

#### Provenance Requirements:
1. `origin`: `'original'` (author's own creation) or `'adapted'` (re-expressed from external source).
2. `license`: SPDX license identifier under which our recipe text and assets are published.
3. **Adapted recipes**: When `origin == 'adapted'`, the following are mandatory:
   - `source_url`: Canonical URL of the source.
   - `source_title`: Title of the source recipe or book reference.
   - `retrieved_at`: ISO 8601 timestamp of retrieval.
   - `raw_scrape_sha256`: SHA-256 hash of the scraped source HTML or raw text fixture cached in `.scratch/scrapes/<id>/raw.html` (gitignored).
   - `source_extract_sha256`: SHA-256 hash of the RFC 8785 canonicalized JSON representation of `source_extract` (`SHA256(JCS(source_extract))`).
   - `source_extract`: Structured digest of source facts:
     - `quantities`: List of extracted ingredients `{ amount, unit, food, raw }`. Used for continuous tolerance-matched Jaccard overlap tests ($J_\tau$, ADR-0009).
     - `prose_fingerprint`: `{ simhash, shingle_hashes, word_count }`. Used to verify that prose has been completely re-authored and not plagiarized, without requiring copyrighted source HTML in the Git repository.
4. **Deterministic Exit 2**: If an adapted recipe is missing `provenance.yaml`, missing `source_extract_sha256`, or missing `source_extract`, `mise run verify` exits with **code 2** (ADR-0002). This prevents CI from passing vacuously when scraping data is omitted.
5. `review.<lang>`: Mandatory per language when that language in `recipe.yaml` has `state: reviewed`. Must contain `reviewed_by`, `reviewed_at`, `recipe_content_sha256` (`SHA256(JCS(\Pi_{\text{lang}}(R)))`), and `source_extract_sha256_at_review` (ADR-0009).
6. `images`: List of image assets. Each image specifies `file`, `origin` (`'original'`, `'generated'`, or `'licensed'`), `license`, optional `prompt`, and a 64-bit perceptual hash (`phash`) for regression assertions.

---

## Worked Example

### `recipes/mornay-sauce/recipe.yaml`

```yaml
id: mornay-sauce
uid: k8m2x9
state:
  cs: reviewed
  en: reviewed
title:
  cs: Omáčka Mornay
  en: Mornay Sauce
slug:
  cs: omacka-mornay
  en: mornay-sauce
slug_aliases:
  cs:
    - omacka-morne
  en:
    - sauce-mornay
summary:
  cs: Klasická francouzská sýrová omáčka na bázi bešamelu obohacená žloutkem a sýrem Gruyère.
  en: A classic French cheese sauce built on a silky béchamel base, enriched with egg yolk and Gruyère.
published_at: "2026-10-25"
category: sauce
cuisine: french
difficulty: medium
tags:
  - classic
  - gratin
  - bistro
table_layout: auto

yield:
  amount: 4
  unit: porce
  label:
    cs: porce
    en: servings
  alternate:
    amount: 500
    unit: ml

timings:
  prep_minutes: 10
  cook_minutes: 15
  rest_minutes: 0

ingredients:
  - id: butter
    food: butter
    amount: 30
    unit: g
    scaling: linear
    step: roux
  - id: flour
    food: flour-all-purpose
    amount: 30
    unit: g
    scaling: linear
    step: roux
  - id: milk
    food: milk-whole
    amount: 500
    unit: ml
    scaling: linear
    note:
      cs: zahřáté na 60 °C
      en: warmed to 60 °C
    step: bechamel
  - id: egg-yolk
    food: egg
    amount: 1
    unit: piece
    scaling: linear
    note:
      cs: pouze žloutek, pokojová teplota
      en: yolk only, room temperature
    step: enrich
  - id: gruyere
    food: cheese-gruyere
    amount: 100
    unit: g
    scaling: linear
    note:
      cs: jemně nastrouhaný
      en: finely grated
    step: finish
  - id: nutmeg
    food: nutmeg
    amount: 1
    unit: spetka
    scaling: taste
    optional: true
    step: finish
  - id: salt
    food: salt
    amount: 2
    unit: g
    scaling: taste
    step: finish
  - id: white-pepper
    food: pepper-white
    amount: 1
    unit: spetka
    scaling: taste
    step: finish

prep_tasks:
  - id: warm-milk
    before_step: bechamel
    temp_c: 60
    time_minutes: 5
    prose:
      cs: V malém rendlíku mírně zahřejte plnotučné mléko těsně pod bod varu (cca 60 °C).
      en: In a small saucepan, gently warm the whole milk until hot but not boiling (around 60 °C).

steps:
  - id: roux
    phase:
      cs: Základ (Jíška)
      en: Base (Roux)
    lane: active
    temp_c: 120
    time_minutes: 2
    prose:
      cs: V rendlíku na středním plameni rozpusťte máslo, přisypte mouku a za stálého míchání restujte do světlé jíšky.
      en: Melt the butter in a heavy saucepan over medium heat, stir in the flour, and cook whisking constantly into a pale roux.

  - id: bechamel
    phase:
      cs: Bešamel
      en: Béchamel
    lane: active
    consumes:
      - roux
    timers:
      - duration_minutes: 2
        temp_c: 90
        label:
          cs: Přilévání mléka
          en: Whisking milk
      - duration_minutes: 5
        temp_c: 85
        label:
          cs: Provaření omáčky
          en: Simmering
    prose:
      cs: Za stálého intenzivního šlehání metlou postupně přilévejte teplé mléko. Zmírněte plamen a za občasného prošlehání vařte na mírném ohni do zhoustnutí a hladké konzistence.
      en: Gradually pour in the warm milk while whisking vigorously to avoid lumps. Reduce heat to low and simmer gently, whisking frequently, until thickened and silky.

  - id: enrich
    phase:
      cs: Zjemnění
      en: Enrichment
    lane: active
    consumes:
      - bechamel
    time_minutes: 1
    prose:
      cs: Rendlík odstavte z přímého ohně. Rychle a důkladně zašlehejte žloutek, aby se omáčka propojila a nesrazila.
      en: Remove saucepan from the heat. Quickly and thoroughly whisk in the egg yolk until incorporated, preventing curdling.

  - id: finish
    phase:
      cs: Dokončení a dochucení
      en: Finishing & Seasoning
    lane: active
    consumes:
      - enrich
    time_minutes: 2
    prose:
      cs: Do teplé omáčky vsypte nastrouhaný Gruyère a míchejte, dokud se sýr zcela nerozpustí. Dochuťte solí, bílým pepřem a čerstvě nastrouhaným muškátovým oříškem.
      en: Fold in the grated Gruyère, stirring until melted smoothly. Season to taste with salt, freshly ground white pepper, and a pinch of nutmeg.

variants:
  - id: cheddar-mornay
    title:
      cs: Čedarová varianta
      en: Sharp Cheddar Variant
    replace:
      - target: gruyere
        instance:
          food: cheese-cheddar
          note:
            cs: vyzrálý čedar, jemně strouhaný
            en: sharp aged cheddar, finely grated
    add:
      - id: dijon
        food: mustard-dijon
        amount: 5
        unit: g
        scaling: taste
        optional: true
        note:
          cs: pro zvýraznění chuti
          en: for tang and depth
        step: finish
    allergens_add:
      - mustard
    prose_overrides:
      finish:
        cs: Do teplé omáčky vsypte nastrouhaný čedar a lžičku dijonské hořčice. Míchejte do hladkého rozpuštění a dochuťte solí a bílým pepřem.
        en: Fold in the grated cheddar and Dijon mustard into the warm sauce. Stir until completely melted and smooth, then season to taste.

nutrition:
  basis: per_serving
  kcal: 215
  protein_g: 9.2
  fat_g: 17.5
  carbs_g: 6.8
  salt_g: 0.8
  source: CIQUAL French Food Composition Table (Sauce Mornay, code 11048)
  source_url: https://ciqual.anses.fr/#/aliments/11048
```

### `recipes/mornay-sauce/provenance.yaml`

```yaml
origin: adapted
license: CC-BY-SA-4.0
source_url: https://example.com/classic-french-mornay
source_title: Le Guide Culinaire - Sauce Mornay Fact Sheet
retrieved_at: "2026-10-25T12:00:00Z"
raw_scrape_sha256: 4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945
source_extract_sha256: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b

source_extract:
  quantities:
    - amount: 30
      unit: g
      food: butter
      raw: 30 g de beurre
    - amount: 30
      unit: g
      food: flour-all-purpose
      raw: 30 g de farine
    - amount: 500
      unit: ml
      food: milk-whole
      raw: 1/2 litre de lait
    - amount: 1
      unit: piece
      food: egg
      raw: 1 jaune d'oeuf
    - amount: 100
      unit: g
      food: cheese-gruyere
      raw: 100 g de gruyère râpé
  prose_fingerprint:
    simhash: "0x8f2d4e1a9b3c5d7e"
    shingle_hashes:
      - "0x3a4f12"
      - "0x9c8b74"
      - "0x12ed5a"
    word_count: 145

review:
  cs:
    reviewed_by: Martin Šmíd
    reviewed_at: "2026-10-25T14:30:00Z"
    recipe_content_sha256: 4a2b6c8d1e3f5a7b9c0d2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    source_extract_sha256_at_review: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    notes: Verified culinary proportions against Escoffier reference, validated Czech phrasing, verified step laminarity.
  en:
    reviewed_by: Martin Šmíd
    reviewed_at: "2026-10-25T14:45:00Z"
    recipe_content_sha256: 7f8e9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f
    source_extract_sha256_at_review: a8b3d6c19f2e4b7a8d0c2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
    notes: Verified culinary proportions against Escoffier reference, validated English phrasing, terminology matches culinary glossary.

images:
  - file: hero.webp
    origin: licensed
    license: CC-BY-SA-4.0
    source_url: https://commons.wikimedia.org/wiki/File:Sauce_Mornay.jpg
    phash: d4c3b2a10f9e8d7c
```

---

## Verification & Mutant Matrix

`mise run verify` executes the following deterministic test suite against every recipe and variant in the repository:

### 1. Verification Suite Rules

| Rule | Severity | Deterministic Condition |
|---|---|---|
| `RECIPE_SCHEMA` | Error (Exit 1) | Document fails JSON Schema / Zod structural validation. |
| `RECIPE_UID_FORMAT` | Error (Exit 1) | `uid` matches regex `^[0-9a-hjkmnp-tv-z]{6}$` (lowercase Crockford Base32) and is globally unique across the entire recipe corpus. |
| `STEP_REF_INTEGRITY` | Error (Exit 1) | Every `ingredient.step` and `prep_task.before_step` resolves to an existing step in `steps`. |
| `CONSUMES_DAG` | Error (Exit 1) | `step.consumes` contains only prior step IDs ($P < S$ in document order). No cycles, no forward references. |
| `FOOD_REGISTRY_LINK` | Error (Exit 1) | In `reviewed` recipes, every non-component instance must link to a non-deprecated `food` ID in `data/foods.yaml`. |
| `COMPONENT_DAG` | Error (Exit 1) | Target component recipe exists, is acyclic, and adheres to a 2-level hierarchy: parent (level 0) $\to$ component (level 1), max edge depth = 1. |
| `COMPONENT_BRIDGE` | Error (Exit 1) | If component is consumed in a dimension differing from component's `yield.unit`, component must declare `yield.alternate` in that dimension. |
| `COMPONENT_BUILD_AVAILABILITY` | Error (Exit 1) | Published recipe or search index entry references a component that is not published in that language. |
| `TAXONOMY_CATEGORY_UNKNOWN` | Error (Exit 1) | `category` fails to resolve to an active ID in `data/taxonomy.yaml#categories`. |
| `TAXONOMY_CUISINE_UNKNOWN` | Error (Exit 1) | `cuisine` fails to resolve to an active ID in `data/taxonomy.yaml#cuisines`. |
| `TAXONOMY_TAG_UNKNOWN` | Error (Exit 1) | Any entry in `tags` fails to resolve to an active ID in `data/taxonomy.yaml#tags`. |
| `TAXONOMY_DIET_UNKNOWN` | Error (Exit 1) | Any entry in `diets` fails to resolve to an active ID in `data/taxonomy.yaml#diets`. |
| `TAXONOMY_TAG_LIMIT_EXCEEDED` | Error (Exit 1) | Recipe declares $> 8$ tags in `tags`, violating card layout density and chip visual budget. |
| `TAG_DUPLICATE_INSTANCE` | Error (Exit 1) | Duplicate tag ID declared within `tags` or `diets` on the same recipe. |
| `DIET_ALLERGEN_CONFLICT` | Error (Exit 1) | Recipe declares a diet claim that violates conflict rules against `allergens_detected`, `origins_detected`, or `animal_products_detected`. |
| `DIET_UNLINKED_FOOD` | Error (Exit 1) | Recipe declares one or more entries in `diets` while containing unlinked ingredients (`food: null`). |
| `DIET_DISCLAIMER_PRESENT` | Error (Exit 1) | Diet entry in `data/taxonomy.yaml#diets` is missing mandatory bilingual `disclaimer: { cs, en }`. |
| `TIME_LEDGER_MISMATCH` | Warning | `timings.cook_minutes` is less than the sequential sum of active step `time_minutes` or `timers` durations. |
| `LANGUAGE_PARITY` | Error (Exit 1) | For all languages not in `'draft'`, step count, step IDs, sequence, timings, temperatures, and instance step bindings are identical. |
| `ZERO_TOKEN_PROSE` | Error (Exit 1) | Step prose matches regex `\{[^\}]+\}` or contains raw unescaped ingredient amounts. |
| `STEP_SHORT_FORMAT` | Error (Exit 1) | A present `short` value is empty or exceeds 48 characters in its language. |
| `STEP_SHORT_MISSING` | Warning | `table_layout` resolves to `matrix` and a step has no `short` cue for a published language. |
| `MATRIX_LAMINARITY` | Error (Exit 1) / Warning | If explicit `table_layout: 'matrix'` and step intervals cross, fails with Exit 1. If `table_layout: 'auto'` (default) and step intervals cross, emits build warning and falls back cleanly to `phase-rail`. |
| `EMPTY_STEP_ROWS` | Error (Exit 1) | Any step has $\text{rows}(S) = \emptyset$ (consumes neither ingredients nor prior steps). |
| `PROVENANCE_FAIL_CLOSED` | Error (Exit 2) | Adapted recipe missing `provenance.yaml`, missing `source_extract_sha256`, or missing `source_extract`. |
| `NUTRITION_SERVING_SIZE` | Error (Exit 1) | `nutrition.basis == 'per_serving'` and `yield.unit !== 'porce'`, but `serving_size` is missing. |

### 2. Representative Mutants (see ADR-0008 §5 for the complete 54-mutant matrix)

Per ADR-0002, `mise run verify` must fail if any mutant passes or is missing from the test harness:

1. `mutant-dangling-step`: An ingredient instance references `step: non-existent-step`.
2. `mutant-crossing-spans`: A recipe with explicit `table_layout: matrix` where step 1 uses ingredient A, step 2 uses ingredient B and step 1 output, and step 3 uses ingredient A and step 2 output, creating crossing intervals in the matrix. Must fail `MATRIX_LAMINARITY` with **Exit Code 1**. (When `table_layout: auto`, cleanly triggers fallback to `phase-rail` with a build warning).
3. `mutant-token-in-prose`: A step prose string containing `@butter{30%g}` or `{flour}`. Must fail `ZERO_TOKEN_PROSE`.
4. `mutant-disagreeing-step-count`: A recipe where Czech has 4 steps and English has 3 steps. Must fail `LANGUAGE_PARITY`.
5. `mutant-missing-provenance`: An adapted recipe with `origin: adapted` but no `provenance.yaml`. Must exit with **code 2**.
6. `mutant-empty-step-rows`: A step that has no ingredients assigned and `consumes: []`. Must fail `EMPTY_STEP_ROWS`.
7. `mutant-unlinked-reviewed`: A recipe with `state: { cs: reviewed }` but an ingredient with `food: null`. Must fail `FOOD_REGISTRY_LINK`.
8. `mutant-cyclic-component`: Recipe A references component B, which in turn references recipe A. Must fail `COMPONENT_DAG`.
9. `mutant-deep-component`: Recipe A references component B, which references component C, which references component D (depth 3). Must fail `COMPONENT_DAG`.
10. `mutant-unbridged-component`: Recipe A consumes component B in `g`, but component B yields `porce` and has no `yield.alternate`. Must fail `COMPONENT_BRIDGE`.
11. `mutant-invalid-uid`: A recipe with an invalid Crockford Base32 `uid` (e.g. uppercase characters, containing forbidden letters `i`, `l`, `o`, `u`, or length != 6) or a duplicate `uid` already claimed by another recipe. Must fail `RECIPE_UID_FORMAT`.

---

## Consequences

- **Determinism across projections**: The compiler derives reading prose, interactive cook mode, print sheets, and the C4E matrix from a single, verified data source without fuzzy heuristics.
- **Fail-closed legal posture**: The repository cannot accidentally publish scraped recipes without legal attribution, licence tracking, and re-expression verification.
- **Bilingual authoring speed**: Authors write natural, unencumbered prose in Czech and English. Translators and editors work on parallel strings without breaking AST bindings.
- **Independent language lifecycles**: A recipe can publish in Czech when ready (`reviewed`) while English remains unindexed (`unverified`) or excluded (`draft`).
- **Clean diffs**: Recipe modifications produce clean, readable YAML diffs where structural changes (quantities, steps, timings) are clearly separated from prose tweaks.

---

## Rejected

- **Markdown front-matter + prose bodies**: Rejected because it separates ingredient data from method instructions, destroys cross-language referential integrity, and cannot guarantee structural language parity.
- **Inline token languages (Cooklang, RecipeMD)**: Rejected due to catastrophic failure under Slavic inflection, translation friction, and inability to enforce structured allergens and nutrition.
- **Storing computed totals**: Rejected storing total cooking time, total nutrition rollups, or scaled quantities. All totals are derived dynamically at build time or render time.
- **Vacuous exit 0 on missing provenance**: Rejected allowing CI to pass when scraping data is omitted. Exit code 2 enforces test environment integrity.
- **Coupling unit indivisibility to scaling class**: Rejected treating indivisible counts as a scaling type. Indivisibility is a unit dimension property; scaling governs arithmetic behavior.
- **Global recipe-level publication state**: Rejected a single publication state that forced either leaking unreviewed translations or holding back reviewed native recipes.
