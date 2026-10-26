# ADR-0008: Verification suite, Oxlint a11y contract, mutant matrix, and phased print gate

- Status: Accepted
- Date: 2026-10-25
- Decided in: ticket *Verification suite definition* (#8), after research and adversarial challenge on static linting and phased print separation

## Context

ADR-0002 established that `mise run verify` is the non-vacuous outside judge of the codebase: tests must fail for reasons a reviewer can recompute, scaling arithmetic must be verified literally without relying on external kitchen fixtures as oracles, provenance sidecars must fail closed with exit code 2 when scrape data is omitted, and accessibility must be an unyielding contract rather than a transient count of zero violations.

Subsequent architectural decisions expanded the structural invariants of Culinary Spec:
- **ADR-0004** defined unit arithmetic, magnitude rounding, eighth-based display for spoons, strict bounds on range scaling, and explicit portion refusal when indivisible counts (such as eggs) cannot be partitioned cleanly.
- **ADR-0005** established the canonical food registry (`data/foods.yaml`), EU 1169/2011 allergen derivation, acyclic deprecation chains, and composition aggregation.
- **ADR-0006** formalized the recipe document contract (`recipe.yaml`), intermediate step consumption DAGs (`consumes`), acyclic component references (depth $\le 2$), matrix geometry laminarity, per-language publication states (`draft`, `unverified`, `reviewed`), and mandatory provenance extracts.
- **ADR-0007** established the URL routing and internationalization contract: symmetric `/cs/` and `/en/` language prefixes, mandatory trailing slashes, Crockford Base32 `uid` validation, static redirect stubs for mutable `slug` aliases and immutable shortlinks (`/<lang>/<section>/<uid>/`), interactive state preservation across language switching, and strict SEO parity (`hreflang` and `sitemap.xml` suppressed on unreviewed drafts).
- **ADR-0009** established the content pipeline, tolerance-matched Jaccard similarity ($J_\tau$), cryptographic review binding ($\Pi_{\text{lang}}(R)$), component state monotonicity, and out-of-band source drift audits.
- **ADR-0010** established search facets, the closed taxonomy registry (`data/taxonomy.yaml`), authoritative allergen tokens (`data/allergens.yaml`), biological origin validation, search index schema, word-boundary text matching, deterministic tie-breaking, fail-closed safety filtering, and accessible single-filter relaxation.

However, turning ADR-0002 into an executable, non-vacuous, everyday development harness revealed three critical operational challenges:

1. **Accessibility Verification Latency and Static vs Runtime Boundary**:
   Relying exclusively on full-browser Playwright runs with `axe-core` across every commit creates unacceptable latency (tens of seconds) during local authoring and pre-commit hooks (`hk`). Furthermore, runtime DOM scanners catch missing attributes only after components render in specific test states, but cannot catch JSX template syntax misuse, missing labels, or invalid ARIA properties across the entire unrendered source tree. Conversely, static linter rules (like ESLint) are historically slow (5–15 seconds for TypeScript AST parsing) and susceptible to configuration drift or silent suppression comments (`/* eslint-disable */`). We require lightning-fast (< 2.0 s) static linting via `oxlint` (v1.89+) with custom SolidJS/a11y rules, paired with a cryptographically enforced parity contract against runtime `axe-core` sweeps on hydrated interactive DOM states.
2. **Environment Brittleness of Physical PDF Print Verification**:
   ADR-0002 and ADR-0006 mandate rigorous print assertions: rendering A4 sheets via headless Chromium, asserting an exact MediaBox of $595.28 \times 841.89\text{ pt}$, confirming embedded fonts with zero system fallbacks via `pdffonts`, and asserting zero page-break splits inside recipe steps or matrix rows. However, `pdffonts` and `pdfinfo` require external native OS binaries (Poppler utilities) and pinned system font packages. Coupling daily local commits (`hk`) and general CI pull-request validation to native Poppler binaries creates fragile setup requirements, breaks headless container environments, and blocks contributors working strictly on web recipes and client UI.
3. **Vacuous Green Runs and Mutant Dispersion**:
   A test suite that passes when test fixtures are empty, when environment variables are unset, or when mutant corpus files are accidentally skipped provides zero engineering confidence. Each ADR defined specific failure modes, but without a consolidated, executable mutant runner and a unified rule matrix, regression mutants risk bitrot or silent omission.

---

## Decision

We establish the normative architecture for the Culinary Spec verification harness, defining execution tiers, deterministic exit codes, a fail-closed accessibility contract between `oxlint` and `axe-core`, an exact mutant execution manifest, and a phased separation between the web release gate and native physical print verification.

---

### 1. Three-Tier Verification Architecture & Exit Codes

Verification is structured into three discrete commands managed via `mise`:

```
                       ┌─────────────────────────┐
                       │   mise run verify:fast  │  (SLA < 2.0 s, pre-commit & CI)
                       │  Oxlint, Types, Schemas │  Zero browser/OS dependencies
                       │  Mutant Runner, Logic   │
                       └────────────┬────────────┘
                                    │ PASS
                                    ▼
                       ┌─────────────────────────┐
                       │   mise run verify:slow  │  (Playwright Chromium on dist/)
                       │   Routes, 4-Stage Axe   │  State preservation, Budgets
                       │   Print-Media Smoke     │  Zero Poppler dependencies
                       └────────────┬────────────┘
                                    │
               ┌────────────────────┴────────────────────┐
               │                                         │
               ▼                                         ▼
   ┌───────────────────────┐                 ┌───────────────────────┐
   │    mise run verify    │                 │  mise run verify:print │
   │ (Phase 1 Launch Gate) │                 │ (Phase 2 Print Gate)  │
   │   verify:fast then    │                 │  Poppler, Fonts,      │
   │      verify:slow      │                 │  MediaBox, Break-Free │
   └───────────────────────┘                 └───────────────────────┘
```

#### Tier 1: `mise run verify:fast` (SLA < 2.0 s)
Runs locally on every commit via `hk` and in CI on every push. Requires only Node.js/Bun runtime; zero browser or OS binary dependencies.
1. **Contract & Environment Integrity (`assert-a11y-contract`)**: Asserts that `.oxlintrc.json` and `tests/verify/axe-rules.json` are present, non-empty, and match the canonical 12-rule accessibility contract. Verifies zero inline suppression comments in `src/`. Exits immediately with **Exit Code 2** if tampered.
2. **Static Linting (`oxlint`)**: Executes `oxlint --deny-warnings` enabling `jsx-a11y` rules and custom SolidJS/a11y plugin rules (enforcing standard HTML `for` on `<label>`, forbidding React-specific attributes like `className` or `htmlFor`, and banning destructured props that break Solid fine-grained signal reactivity). Supplemented by `ast-grep` (`sg scan`) for structural AST guards where linter rules do not apply.
3. **TypeScript Typecheck (`tsc --noEmit`)**: Validates complete type soundness across `src/` and `scripts/`.
4. **Corpus, Schema, Taxonomy & Search Validation**:
   - Validates all recipes against JSON Schema / Zod definitions, verifies Crockford Base32 `uid` uniqueness and formatting, validates referential integrity of ingredient `step` and prep-task `before_step` IDs, verifies intermediate step consumption DAGs (`consumes`), checks component composition DAGs (acyclic, 2-level hierarchy: parent $\to$ component, max edge depth = 1), asserts component build and publication availability in referenced languages (`COMPONENT_BUILD_AVAILABILITY`, Exit 1), validates component dimension bridges (`yield.alternate`), verifies variant matrix laminarity, and asserts structural language parity across non-draft locales (`LANGUAGE_PARITY`).
   - Asserts allergen registry presence and validity (`ALLERGEN_REGISTRY_PRESENT`, Exit 2; `ALLERGEN_CANONICAL_TOKEN`, Exit 1).
   - Asserts mandatory biological `origin` in `data/foods.yaml` (`FOOD_ORIGIN_MISSING`, Exit 1).
   - Validates taxonomy registry presence, schema, global ID uniqueness, and immutability (`TAXONOMY_REGISTRY_PRESENT`, Exit 2; `TAXONOMY_SCHEMA`, Exit 1; `TAXONOMY_ID_GLOBAL_UNIQUE`, Exit 1; `TAXONOMY_ID_IMMUTABLE`, Exit 1).
   - Verifies recipe taxonomy references, duplicate tag absence, and tag count cap $\le 8$ (`TAXONOMY_CATEGORY_UNKNOWN`, `TAXONOMY_CUISINE_UNKNOWN`, `TAXONOMY_TAG_UNKNOWN`, `TAXONOMY_DIET_UNKNOWN`, `TAG_DUPLICATE_INSTANCE`, `TAXONOMY_TAG_LIMIT_EXCEEDED`, Exit 1).
   - Enforces diet claims: evaluates `forbid_allergens` against non-removable `allergens_detected`, `origins_detected`, and `animal_products_detected` (with the documented exception of `lactose-free` evaluating `forbid_mandatory_allergens` against `allergens.mandatory` + `allergens.optional`), checks zero unlinked foods on diet recipes, and verifies mandatory disclaimers (`DIET_ALLERGEN_CONFLICT`, `DIET_UNLINKED_FOOD`, `DIET_DISCLAIMER_PRESENT`, Exit 1).
   - Validates search index schema, asserts semantic bilingual token union `folded.* == fold(cs) ∪ fold(en)` against source recipes, and guarantees 100% build coverage of published recipes without draft leakage (`SEARCH_INDEX_SCHEMA`, `SEARCH_INDEX_COVERAGE`, Exit 1).
   - Validates that external vendored runtime dependencies adhere to integrity pinning (`DEPENDENCY_PINNED`, Exit 1): this is a Phase 2 contingency rule that activates only when the Phase 2 `MiniSearch` feature flag is enabled (and therefore has no Phase 1 mutant in the 54-mutant matrix).
   - Free of synthetic 500-recipe micro-benchmarks; O(recipes $\times$ tags) execution completes in $< 100\text{ ms}$, safely within the 2.0 s SLA.
5. **Literal Arithmetic & Scaling Suite (`literal_scaling_and_unit_arithmetic`)**: Executes pure-function unit conversion, magnitude rounding, eighth-based spoon formatting, outward range scaling, and portion refusal assertions against in-test literal expectations.
6. **Provenance Sidecar Verification**: Evaluates all adapted recipes in `recipes/`: verifies presence of `provenance.yaml`, `source_extract_sha256`, and normalized source extract (fails closed with **Exit Code 2** if missing). Asserts Continuous Tolerance-Matched Bipartite Jaccard index $J_\tau < 0.90$, 8-word shingle overlap $\le 0.05$, and 64-bit DCT perceptual hash (`phash`) Hamming distance $> 6$ against scraped source images when `origin !== 'licensed'` (`IMAGE_DUPLICATE_OR_COPIED`).
7. **Static Print-Readiness Checks**: Validates presence of `@page { size: A4 }` rules in print stylesheets, verifies `break-inside: avoid` on step and matrix rows, and asserts that QR code shortlink targets strictly match `/<lang>/(recepty|recipes)/<uid>/`.
8. **Consolidated Mutant Runner**: Executes all 54 mutants in `tests/corpus/mutants/` (count derived dynamically from manifest catalog), asserting 100% kill rate with exact expected rule codes and exit codes.

#### Tier 2: `mise run verify:slow` (Playwright Headless Chromium)
Runs against production static build output in `dist/`. Requires headless Chromium via Playwright, but **zero Poppler or external OS utilities**:
1. **Route, Link & Permanence Verification**: Crawls all generated static HTML files; asserts symmetric language prefixes (`/cs/`, `/en/`), trailing slash termination on all links and canonical tags, static redirect stubs for `slug_aliases`, deprecated tag aliases (`TAG_ALIAS_INTEGRITY`, Exit 1), and immutable `<uid>` shortlinks, thin tag page robot indexing assertions (`TAG_THIN_PAGE`, Exit 1), complete coverage of the `404.html` UID rescue map, bidirectional `hreflang` parity for `reviewed` pages, `<meta name="robots" content="noindex,nofollow">` on `/cs/hledani/` and `/en/search/` (`HREFLANG_SEO_PARITY`, Exit 1), `sitemap.xml` validity, and schema.org `Recipe` JSON-LD compliance.
2. **Four-Stage Post-Hydration `axe-core` Contract**: Scans rendered pages under light and dark themes across four required states:
   - *State A (Hydrated DOM)*: Asserts zero violations on `document.body` after `data-hydrated="true"` is set.
   - *State B (Opened Dialog / Cook Mode)*: Triggers cook mode or modal dialog; asserts active focus trap, accessible name on dialog, and `Escape` key returning focus cleanly to the activating button.
   - *State C (Servings Stepper Interaction)*: Triggers portion stepper ($4 \to 6$); asserts that `document.activeElement` remains pinned to the stepper button and that `aria-live="polite"` announces the updated portion yield.
   - *State D (Search & Facets)*: Navigates to `/cs/hledani/` and `/en/search/`; triggers search input, facet chip selection, and empty state; asserts `aria-live="polite"` announcements on result count, accessible names on removable filter chips, and strictly escaped text node rendering of reflected queries (`SEARCH_RESULT_ECHO_ESCAPED`, Exit 1).
3. **Language Switch State Preservation**: Navigates to a recipe with active kitchen state (`?yield=6&unit=imperial&variant=comte&mode=cook#step-roux`); activates `#lang-switch`; asserts that the target language page loads with all query parameters and fragment anchors perfectly preserved. Also verifies search query preservation on `/cs/hledani/?q=omacka&tag=quick` switching to `/en/search/?q=omacka&tag=quick` (`LANG_SWITCH_TARGET_INTEGRITY`, `CANONICAL_PARAM_KEYS`, Exit 1).
4. **Performance & INP Budgets**:
   - Asset size ceilings: Total compressed JS bundle $\le 35\text{ kB}$ gzip; total CSS $\le 20\text{ kB}$ gzip; search index payload ceiling pinned for `search-index.<lang>.json`.
   - Field floor metrics: Largest Contentful Paint (LCP) $\le 1500\text{ ms}$, Cumulative Layout Shift (CLS) $\le 0.02$.
   - Interactive INP budget: Scripted `PerformanceObserver('event')` measuring duration of portion stepper clicks, unit toggles, and cook mode triggers; asserts $\text{INP duration} \le 100\text{ ms}$. Also asserts scripted keystroke latency (`input` $\to$ `rAF`, debounced 120 ms) and facet toggle response $\le 100\text{ ms}$.
5. **Search Safety Filtering & Variant Integrity**:
   - Asserts fail-closed zero-result behavior when unrecognized tokens are passed to `allergen_not` or `diet`, and asserts that unverified recipes with `food: null` are strictly excluded under safety filters even when `?include_unverified=1` is active (`SEARCH_INVALID_SAFETY_FILTER`, Exit 1).
   - Asserts variant link integrity (`?variant=<V.id>`) when recipes match excluded allergen filters via clean substituted variants (`SEARCH_VARIANT_LINK_INTEGRITY`, Exit 1).
6. **Browser Print-Media Smoke**: Executes `page.emulateMedia({ media: 'print' })`; asserts that global site navigation and interactive controls carry `display: none`, and that print-dedicated routes (`/cs/recepty/<uid>-<slug>/tisk/` and `/en/recipes/<uid>-<slug>/print/`) render without JavaScript execution errors.

#### Tier 3: Phase 1 vs Phase 2 Separation
- **`mise run verify` (Phase 1 Launch Gate)**:
  Sequentially executes `mise run verify:fast` followed by `mise run verify:slow`.
  This is the universal release gate for Phase 1. It gates `hk` pre-commit hooks and GitHub Actions CI for all fundamental website launch functions (tickets #6, #9, #10, #11, #12). It guarantees that the public responsive site, cook mode, search, i18n, and accessibility are 100% verified without imposing any native OS binary requirements.
- **`mise run verify:print` (Phase 2 Standalone Print Gate)**:
  Dedicated verification gate for physical A4 PDF print sheets (tickets #5 and #13).
  1. *Pre-flight Environment Integrity Check*: Runs `scripts/verify/assert-print-environment.sh`. Verifies that `pdfinfo` and `pdffonts` (Poppler) exist in `$PATH`, and that pinned WOFF2/TTF font files exist on disk. If any tool or font is absent, terminates immediately with **Exit Code 2 (`PRINT_ENV_MISSING`)**. It is strictly forbidden to exit 0 or pass vacuously when print tools are missing.
  2. *A4 MediaBox Assertions*: Emits PDF via `page.pdf({ preferCSSPageSize: true, printBackground: true })`. Parses output with `pdfinfo`; asserts MediaBox width is $595.28\text{ pt} \pm 0.5\text{ pt}$ and height is $841.89\text{ pt} \pm 0.5\text{ pt}$ (exact ISO A4 $210 \times 297\text{ mm}$).
  3. *Embedded Font Assertions*: Inspects emitted PDF with `pdffonts`; asserts that 100% of fonts report `emb: yes` and that zero system fallback fonts (e.g. Arial, Times New Roman, Nimbus) appear.
  4. *Page-Break Splitting Assertions*: Inspects PDF box coordinates; asserts zero page-break splits across `.recipe-step`, `.prep-task`, or `.matrix-row` containers.

---

### 2. Deterministic Exit Codes

The verification suite strictly adheres to three distinct exit codes:

| Exit Code | Classification | Deterministic Trigger Condition |
|---|---|---|
| `0` | **PASS** | All assertions pass with strictly non-zero evidence counts ($N > 0$ recipes, $K > 0$ mutants killed, $R > 0$ axe rules checked). |
| `1` | **RULE_VIOLATION** | A deterministic rule failed, a mutant survived, a schema validation error occurred, TypeScript failed typecheck, or a performance/INP budget was exceeded. |
| `2` | **INTEGRITY_FAIL_CLOSED** | Test harness or environment integrity failure. Prevents vacuous green runs: missing `provenance.yaml` or scrape extract on adapted recipes; missing, empty, or unparseable `data/allergens.yaml` (`ALLERGEN_REGISTRY_PRESENT`) or `data/taxonomy.yaml` (`TAXONOMY_REGISTRY_PRESENT`); tampered, disabled, or empty accessibility contract; 0 recipes or 0 mutants discovered; or missing Poppler/fonts in `verify:print`. |

#### Deterministic Green Run Output
Upon successful execution of `mise run verify`, the runner must output a single, parseable evidence summary string (with the mutant total derived dynamically from the manifest count):

```
PASS verify (recipes: N, variants: M, mutants_killed: 54/54, arithmetic_assertions: A, oxlint_rules: 12, axe_rules_checked: 12 across 4 states, max_inp_ms: T)
```

If any counter is zero (e.g. `recipes: 0` or `mutants_killed: 0/54`), the run is invalid and exits with code 2.

---

### 3. Oxlint + Custom SolidJS/A11y Rules & `axe-core` Parity Contract

Accessibility is maintained through an enforced mathematical mapping between static analysis in `verify:fast` and runtime DOM verification in `verify:slow`:

```
┌────────────────────────────────────────────────────────┐
│            Static Analysis (verify:fast)               │
│  Oxlint 1.89+ (jsx-a11y/* + solid/*) + ast-grep        │
│  Compile-time AST scanning across all TSX components   │
└───────────────────────────┬────────────────────────────┘
                            │
               1:1 Parity Contract Guard
             tests/verify/a11y-contract.json
            assert-a11y-contract (Fail-Closed)
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│             Runtime Analysis (verify:slow)             │
│        Playwright + axe-core (axe-rules.json)          │
│   Rendered DOM across 4 hydrated interaction states    │
└────────────────────────────────────────────────────────┘
```

#### Canonical 12-Rule Accessibility Contract Mapping

| # | Static Rule (`oxlint` / `ast-grep`) | Runtime Rule (`axe-core`) | WAI-ARIA / WCAG Target | Description & Enforcement Seam |
|---|---|---|---|---|
| 1 | `jsx-a11y/alt-text` | `image-alt` | WCAG 2.1 1.1.1 Non-text Content | All `<img>`, `<area>`, and `<input type="image">` must have descriptive `alt` attributes. |
| 2 | `jsx-a11y/anchor-has-content` | `link-name` | WCAG 2.1 2.4.4 Link Purpose | Anchor tags `<a>` must contain accessible text content (not empty icon buttons). |
| 3 | `jsx-a11y/anchor-is-valid` | `nested-interactive` | WCAG 2.1 2.1.1 Keyboard | Anchors must carry valid `href` attributes; cannot use empty hrefs, `#` without target, or `javascript:`. |
| 4 | `jsx-a11y/aria-props` | `aria-valid-attr` | WCAG 2.1 4.1.2 Name, Role, Value | Elements must only use sanctioned WAI-ARIA 1.2 attribute names. |
| 5 | `jsx-a11y/aria-proptypes` | `aria-valid-attr-value` | WCAG 2.1 4.1.2 Name, Role, Value | ARIA attribute values must match expected schema types (boolean, integer, ID reference). |
| 6 | `jsx-a11y/aria-role` | `aria-roles` | WCAG 2.1 4.1.2 Name, Role, Value | ARIA `role` values must be valid, non-abstract WAI-ARIA roles. |
| 7 | `jsx-a11y/aria-unsupported-elements` | `aria-allowed-role` | WCAG 2.1 4.1.2 Name, Role, Value | Reserved elements (`<meta>`, `<script>`, `<head>`) must not carry ARIA attributes or roles. |
| 8 | `jsx-a11y/role-has-required-aria-props` | `aria-required-attr` | WCAG 2.1 4.1.2 Name, Role, Value | Elements with explicit roles must supply all mandatory attributes (e.g. `role="slider"` requires `aria-valuenow`, `aria-valuemin`, `aria-valuemax`). |
| 9 | `jsx-a11y/heading-has-content` | `empty-heading` | WCAG 2.1 1.3.1 Info and Relationships | Heading elements (`<h1>`–`<h6>`) must contain accessible text. |
| 10 | `jsx-a11y/html-has-lang` | `html-has-lang` | WCAG 2.1 3.1.1 Language of Page | `<html>` root must declare a non-empty `lang` attribute matching the route (`cs` or `en`). |
| 11 | `solid/label-has-for` *(custom)* | `label` | WCAG 2.1 1.3.1 Info and Relationships | Form controls must have associated `<label>` using standard HTML `for` attribute (strictly rejecting React `htmlFor`). |
| 12 | `solid/no-destructured-props` & `solid/no-react-specific-props` *(custom)* | `aria-live` & dynamic focus | WCAG 2.1 4.1.2 Name, Role, Value | Forbids destructured props and React attributes in Solid components, ensuring reactive signals update `aria-live` and ARIA attributes in real time during portion adjustments. |

#### Fail-Closed Guard: `assert-a11y-contract`
Executed as step 1 in `verify:fast`:
1. Reads `tests/verify/a11y-contract.json`.
2. Validates that `.oxlintrc.json` has all 12 static rules configured with severity `"error"` (never `"warn"` or `"off"`).
3. Validates that `tests/verify/axe-rules.json` has all 12 corresponding axe rules configured with `"enabled": true`.
4. Performs a repository-wide regex scan over `src/` for any comments matching `oxlint-disable`, `eslint-disable`, or `axe-disable`. If any rule suppression comment is found, verification immediately halts with **Exit Code 2 (`A11Y_CONTRACT_TAMPERED`)**.

---

### 4. Consolidated Mutant Manifest & Execution Contract

Per ADR-0002, structural rules cannot be verified by green tests alone; every rule must prove its lethality against a deliberately broken mutant.

Each mutant lives in a dedicated directory under `tests/corpus/mutants/<mutant-id>/` containing a `mutant.yaml` manifest:

```yaml
# tests/corpus/mutants/mutant-dangling-step/mutant.yaml
mutant_id: mutant-dangling-step
governing_adr: ADR-0006
expected_exit_code: 1
expected_rule_code: STEP_REF_INTEGRITY
expected_error_path: "recipe.yaml#/ingredients/2/step"
forbidden_codes:
  - YAML_SYNTAX_ERROR
  - RECIPE_SCHEMA
description: "Ingredient instance references step 'non-existent-step' not declared in steps array."
```

#### Mutant Runner Contract
The mutant runner in `verify:fast` iterates through every directory under `tests/corpus/mutants/`:
1. Spawns an isolated test invocation passing the mutant fixture.
2. Asserts that the process exits with **`expected_exit_code`** (1 or 2).
3. Asserts that stderr or structured JSON error output contains the exact **`expected_rule_code`**.
4. Asserts that the error pointer matches **`expected_error_path`**.
5. Asserts that none of the **`forbidden_codes`** were emitted (e.g. failing on generic YAML syntax error rather than the intended domain validation rule).
6. If any mutant passes (exit code 0), exits with the wrong code, or triggers the wrong rule, the mutant runner fails immediately with **Exit Code 1**.

---

### 5. Consolidated 54-Mutant Matrix

The complete, unified 54-row matrix spanning ADR-0004 through ADR-0010:

| # | Mutant ID | Governing ADR | Rule ID | Exit | Deterministic Trigger Condition | Tier |
|---|---|---|---|---|---|---|
| 1 | `mutant-unit-unknown` | ADR-0004 | `UNIT_CLOSED_VOCABULARY` | 1 | Ingredient authors unit not in closed vocabulary (`dimension: null`). | Fast |
| 2 | `mutant-unit-negative-amount` | ADR-0004 | `UNIT_AMOUNT_POSITIVE` | 1 | Ingredient authors `amount: -5` or `amount: 0`. | Fast |
| 3 | `mutant-unit-dimension-mismatch` | ADR-0004 | `UNIT_DIMENSION_MISMATCH` | 1 | Authored alternate unit has incompatible physical dimension with primary. | Fast |
| 4 | `mutant-scaling-egg-fraction` | ADR-0004 | `SCALING_EGG_DISCRETIZATION` | 1 | Yield scaling produces non-integer, non-half egg count without refusal state. | Fast |
| 5 | `mutant-scaling-fixed-violation` | ADR-0004 | `SCALING_FIXED_INVARIANCE` | 1 | Ingredient with `scaling: fixed` modifies amount when yield scales. | Fast |
| 6 | `mutant-rounding-zero-amount` | ADR-0004 | `ROUNDING_NON_ZERO` | 1 | Small quantity rounds to zero instead of minimum positive threshold. | Fast |
| 7 | `mutant-range-inverted` | ADR-0004 | `RANGE_SCALING_BOUNDS` | 1 | Range scaling inverts min/max bounds or rounds inward. | Fast |
| 8 | `mutant-unscalable-unassigned` | ADR-0004 | `SCALER_UNASSIGNED_REFUSAL` | 1 | Recipe lacking step-to-instance assignments renders scaler without refusal notice. | Fast |
| 9 | `mutant-food-duplicate-id` | ADR-0005 | `FOOD_DUPLICATE_ID` | 1 | `data/foods.yaml` contains duplicate food `id`. | Fast |
| 10 | `mutant-food-schema-violation` | ADR-0005 | `FOOD_SCHEMA_VALIDATION` | 1 | `data/foods.yaml` contains entry missing mandatory English/Czech names. | Fast |
| 11 | `mutant-food-unknown-allergen` | ADR-0005 | `ALLERGEN_CANONICAL_TOKEN` | 1 | Food entry declares allergen token outside canonical EU 1169/2011 list. | Fast |
| 12 | `mutant-food-cycle-deprecation` | ADR-0005 | `FOOD_DEPRECATION_ACYCLIC` | 1 | Food entry `deprecated_in_favor_of` forms a cyclic pointer loop ($A \to B \to A$). | Fast |
| 13 | `mutant-food-dangling-deprecation`| ADR-0005 | `FOOD_DEPRECATION_TARGET` | 1 | Food entry `deprecated_in_favor_of` points to non-existent food ID. | Fast |
| 14 | `mutant-unlinked-reviewed` | ADR-0005 | `FOOD_REGISTRY_LINK` | 1 | Recipe with `state: { cs: reviewed }` contains ingredient with `food: null`. | Fast |
| 15 | `mutant-recipe-schema-violation` | ADR-0006 | `RECIPE_SCHEMA` | 1 | `recipe.yaml` root missing mandatory `uid` or `title` fields. | Fast |
| 16 | `mutant-invalid-uid` | ADR-0006 | `RECIPE_UID_FORMAT` | 1 | `uid` contains forbidden Crockford chars (`i`, `l`, `o`, `u`) or length $\ne 6$. | Fast |
| 17 | `mutant-dangling-step` | ADR-0006 | `STEP_REF_INTEGRITY` | 1 | Ingredient instance references step ID not declared in `steps`. | Fast |
| 18 | `mutant-consumes-cycle` | ADR-0006 | `CONSUMES_DAG` | 1 | Step `consumes` references subsequent step ID ($P \ge S$), creating a cycle. | Fast |
| 19 | `mutant-cyclic-component` | ADR-0006 | `COMPONENT_DAG` | 1 | Recipe A references Recipe B as component, which references Recipe A. | Fast |
| 20 | `mutant-deep-component` | ADR-0006 | `COMPONENT_DAG` | 1 | Component composition references chain of depth 3 ($A \to B \to C \to D$). | Fast |
| 21 | `mutant-unbridged-component` | ADR-0006 | `COMPONENT_BRIDGE` | 1 | Recipe consumes component in mass (`g`), but sub-recipe yields `porce` without `yield.alternate`. | Fast |
| 22 | `mutant-disagreeing-step-count` | ADR-0006 | `LANGUAGE_PARITY` | 1 | Czech has 4 steps, English has 3 steps in published recipe. | Fast |
| 23 | `mutant-token-in-prose` | ADR-0006 | `ZERO_TOKEN_PROSE` | 1 | Step prose contains embedded token syntax (`@butter{30%g}` or `{flour}`). | Fast |
| 24 | `mutant-crossing-spans` | ADR-0006 | `MATRIX_LAMINARITY` | 1 | Recipe with explicit `table_layout: matrix` has crossing step intervals $\to$ fails with `MATRIX_LAMINARITY` (Exit 1); when `table_layout: auto`, falls back cleanly to `phase-rail`. | Fast |
| 25 | `mutant-empty-step-rows` | ADR-0006 | `EMPTY_STEP_ROWS` | 1 | Step consumes no ingredients and no prior step ($\text{rows}(S) = \emptyset$). | Fast |
| 26 | `mutant-missing-provenance` | ADR-0006 | `PROVENANCE_FAIL_CLOSED` | 2 | Adapted recipe with `origin: adapted` lacks `provenance.yaml` or scrape extract. | Fast |
| 27 | `mutant-food-origin-missing` | ADR-0005 | `FOOD_ORIGIN_MISSING` | 1 | Food entry in `data/foods.yaml` omits mandatory biological `origin` classification. | Fast |
| 28 | `mutant-nutrition-missing-serving` | ADR-0006 | `NUTRITION_SERVING_SIZE` | 1 | `basis: per_serving` declared on non-portion yield without explicit `serving_size`. | Fast |
| 29 | `mutant-missing-trailing-slash` | ADR-0007 | `ROUTE_TRAILING_SLASH` | 1 | Internal hyperlink emitted without trailing slash (`.../k8m2x9-mornay`). | Slow |
| 30 | `mutant-asymmetric-route` | ADR-0007 | `ROUTE_SYMMETRIC_PREFIX` | 1 | Recipe page emitted at root `/recepty/...` without `/cs/` language prefix. | Slow |
| 31 | `mutant-invalid-slug-charset` | ADR-0007 | `SLUG_FORMAT` | 1 | `slug.cs` contains uppercase characters or Slavic diacritics (`omáčka-mornay`). | Fast |
| 32 | `mutant-missing-shortlink` | ADR-0007 | `SHORTLINK_EMISSION` | 1 | Static redirect stub missing at permanent shortlink path `/<lang>/<section>/<uid>/`. | Slow |
| 33 | `mutant-reused-slug-alias` | ADR-0007 | `SLUG_ALIAS_INTEGRITY` | 1 | Two distinct recipes claim the identical slug alias string. | Fast |
| 34 | `mutant-broken-lang-switch` | ADR-0007 | `LANG_SWITCH_TARGET_INTEGRITY` | 1 | `#lang-switch` links to non-existent URL without `aria-disabled="true"`. | Slow |
| 35 | `mutant-localized-query-param` | ADR-0007 | `CANONICAL_PARAM_KEYS` | 1 | Stepper or cook mode emits localized query param key (`?porce=4`). | Slow |
| 36 | `mutant-unreviewed-hreflang` | ADR-0007 | `HREFLANG_SEO_PARITY` | 1 | `hreflang` alternate tag links to a page whose publication state is `unverified`. | Slow |
| 37 | `mutant-404-missing-uid` | ADR-0007 | `RESCUE_404_MAP` | 1 | Published recipe `uid` missing from `404.html` client rescue map lookup table. | Slow |
| 38 | `mutant-corpus-duplicate-block` | ADR-0009 | `CORPUS_DUPLICATE_BLOCK` | 1 | Candidate recipe has tolerance-matched Jaccard $J_\tau \ge 0.90$ against existing corpus recipe. | Fast |
| 39 | `mutant-corpus-unresolved-near-variant` | ADR-0009 | `CORPUS_UNRESOLVED_NEAR_VARIANT` | 1 | Recipe promoted to `reviewed` with $0.75 \le J_\tau < 0.90$ without `parent` or `not_duplicate_of` justification. | Fast |
| 40 | `mutant-prose-plagiarism-detected` | ADR-0009 | `PROSE_PLAGIARISM_DETECTED` | 1 | Adapted recipe contains verbatim 8-word phrases from source extract exceeding $5\%$ shingle overlap. | Fast |
| 41 | `mutant-image-duplicate-or-copied` | ADR-0009 | `IMAGE_DUPLICATE_OR_COPIED` | 1 | Recipe image has perceptual hash Hamming distance $\le 6$ against scrape image without licensing metadata. | Fast |
| 42 | `mutant-review-stale-content-hash` | ADR-0009 | `REVIEW_STALE_CONTENT_HASH` | 1 | Step prose or shared ingredient amount modified in `reviewed` recipe without re-executing `mise run review`. | Fast |
| 43 | `mutant-review-source-drift` | ADR-0009 | `REVIEW_SOURCE_DRIFT` | 1 | `source_extract_sha256` updated following upstream source drift, but `review[L]` still carries old extract hash. | Fast |
| 44 | `mutant-component-state-inversion` | ADR-0009 | `COMPONENT_STATE_INVERSION` | 1 | Recipe in `reviewed` state references a component whose state in that language is `unverified` or `draft`. | Fast |
| 45 | `mutant-taxonomy-duplicate-id` | ADR-0010 | `TAXONOMY_ID_GLOBAL_UNIQUE` | 1 | `data/taxonomy.yaml` declares duplicate ID across categories, cuisines, diets, or tags. | Fast |
| 46 | `mutant-taxonomy-unknown-tag` | ADR-0010 | `TAXONOMY_TAG_UNKNOWN` | 1 | Recipe declares tag ID not declared in `data/taxonomy.yaml#tags`. | Fast |
| 47 | `mutant-taxonomy-over-limit-tags` | ADR-0010 | `TAXONOMY_TAG_LIMIT_EXCEEDED` | 1 | Recipe declares $> 8$ tags in `tags`, violating card layout density. | Fast |
| 48 | `mutant-diet-allergen-conflict` | ADR-0010 | `DIET_ALLERGEN_CONFLICT` | 1 | Recipe declared `vegan` contains butter with `allergens_remove: [{ allergen: milk, reason: ... }]` (testing non-removable `allergens_detected`). | Fast |
| 49 | `mutant-allergen-variant-launder` | ADR-0010 | `SEARCH_VARIANT_LINK_INTEGRITY` | 1 | Recipe matched under `allergen_not` via variant that fails to replace all allergen-bearing foods. | Slow |
| 50 | `mutant-tag-thin-page` | ADR-0010 | `TAG_THIN_PAGE` | 1 | Tag page with $< 2$ reviewed recipes emits `index,follow` in robots metadata. | Slow |
| 51 | `mutant-search-missing-index-entry` | ADR-0010 | `SEARCH_INDEX_COVERAGE` | 1 | Published recipe in language $L$ missing from generated `search-index.<L>.json`. | Fast |
| 52 | `mutant-search-invalid-safety-filter` | ADR-0010 | `SEARCH_INVALID_SAFETY_FILTER` | 1 | Inbound safety filter (`?allergen_not=glutne` or `?diet=vegen`) fails to return zero results with safety banner, or `food: null` unverified recipe leaks under active safety filter. | Slow |
| 53 | `mutant-tag-alias-missing-stub` | ADR-0010 | `TAG_ALIAS_INTEGRITY` | 1 | Deprecated tag entry in `data/taxonomy.yaml` lacks static 301 HTML redirect stub. | Slow |
| 54 | `mutant-search-langswitch-q` | ADR-0010 | `SEARCH_INDEX_SCHEMA` | 1 | Generated search-index JSON missing bilingual folded text union (`folded.* != fold(cs) ∪ fold(en)`), causing cross-lingual query loss. | Fast |

---

## Consequences

- **Sub-2-Second Pre-commit Gate**: Developers and CI receive instantaneous feedback (< 2.0 s) via `oxlint` and in-memory schema/arithmetic checks on every commit, without spinning up headless browsers.
- **Fail-Closed Legal and Quality Posture**: The repository cannot publish scraped recipes without legal provenance (Exit 2), cannot disable accessibility rules via hidden comments (Exit 2), and cannot pass vacuously when fixtures or mutants are missing (Exit 2).
- **Zero OS Dependency for Site Launch (Phase 1)**: The core static website, interactive kitchen cook mode, bilingual navigation, search, and accessibility are 100% verified via `mise run verify` without requiring Poppler or native font installations.
- **Dedicated Physical Print Gate (Phase 2)**: Physical A4 PDF print verification is isolated in `mise run verify:print`. When run in Phase 2, it guarantees exact ISO A4 MediaBox dimensions, embedded vector fonts, and split-free page breaks with complete determinism.
- **Comprehensive Regression Barrier**: The 54-mutant matrix guarantees that every rule family defined in ADR-0004 through ADR-0010 is verified against executable failure mutants in the 54-mutant matrix.

---

## Rejected

- **Monolithic `mise run verify` Requiring Native Poppler Utilities**:
  Rejected requiring Poppler (`pdfinfo`, `pdffonts`) for everyday development and web release gating. Coupling web verification to native OS packages creates setup friction, breaks containerized CI, and blocks web contributors. Print verification is cleanly decoupled into Phase 2 (`verify:print`).
- **Vacuous Exit 0 on Missing Scrape or Print Dependencies**:
  Rejected allowing the test runner to skip assertions or exit with code 0 when provenance extract files are absent or when native print utilities are missing. Exit code 2 enforces test environment integrity.
- **Runtime-Only Accessibility Verification**:
  Rejected relying solely on `axe-core` sweeps during browser tests. Runtime scans miss unrendered code paths and do not provide immediate feedback during coding. Oxlint provides compile-time protection across 100% of components.
- **Permitting Inline Linter Suppression Comments**:
  Rejected allowing `/* oxlint-disable */` or `/* eslint-disable */` on accessibility rules. The `assert-a11y-contract` runner fails closed if any suppression comment exists in `src/`.
- **Using Kitchen Fixtures as Test Oracles**:
  Rejected using real recipe files as the ground truth for scaling arithmetic. Scaling logic is verified strictly against hand-built, literal mathematical matrices.
- **Allowing Mutants to Pass on Syntax Errors**:
  Rejected treating generic YAML syntax errors or unexpected crashes as valid mutant kills. Mutants must trigger their specific declared `expected_rule_code`.
