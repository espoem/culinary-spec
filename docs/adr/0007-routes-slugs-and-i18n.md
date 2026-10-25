# ADR-0007: Routes, immutable UIDs, mutable slugs, and the i18n contract

- Status: Accepted
- Date: 2026-10-25
- Decided in: ticket *Routes, slugs and the i18n contract* (#7), after two-axis stress-test on language switching and static URL routing

## Context

Culinary Spec targets a static-first deployment (GitHub Pages with optional repository subpath `BASE_PATH`, e.g. `/culinary-spec/`). Content is authored natively and in parallel in Czech (`cs`) and English (`en`). A recipe may exist in only one language during initial drafting, or have different publication states (`draft`, `unverified`, `reviewed`) across languages (ADR-0001, ADR-0006).

Previous static sites, recipe blogs, and internationalization architectures fail under these combined requirements:

1. **Link Rot from Re-slugging**: When an author refines a recipe title or slug (e.g. fixing a typo or improving SEO phrasing), slug-only URLs (`/cs/recepty/omacka-morne/` → `/cs/recepty/omacka-mornay/`) break inbound links, bookmarked URLs, and printed QR codes. On static hosting like GitHub Pages, server-side `.htaccess` or dynamic redirects do not exist.
2. **QR Code Breakage on Printed Sheets**: Physical A4 recipe printouts (ADR-0006, ticket #5) carry QR codes. If a QR code encodes a mutable slug URL that later changes, the physical printout permanently breaks. Printed QR codes require an ultra-short, immutable URL that never changes over the lifetime of the recipe.
3. **Interactive Kitchen State Loss Across Languages**: Readers frequently adjust portion yields (`?yield=6`), toggle metric vs imperial measures (`?unit=imperial`), activate kitchen cook mode (`?mode=cook`), or focus on a specific step anchor (`#step-step-3`). On existing multilingual sites, clicking the language switch either dumps the reader back to the home page or rewrites the pathname while stripping query parameters and fragment anchors, losing the cook's active kitchen state.
4. **Slavic Diacritics and Internationalized Path Segments**: Naive slug generators either preserve non-ASCII characters (producing messy percent-encoded URLs like `/cs/recepty/%C4%8Desk%C3%A1-%C4%8Do%C4%8Dka/`) or strip diacritics non-deterministically. Furthermore, having English path segments for Czech content (e.g. `/cs/recipes/`) looks unnatural and hurts local search engine visibility. Path segments must be natively localized (`/cs/recepty/` vs `/en/recipes/`, `/cs/slovnik/` vs `/en/glossary/`).
5. **SEO Penalties from Inconsistent `hreflang`, `noindex`, and Sitemaps**: Search engines (Google, Bing) severely penalize sites that emit `hreflang` alternate links pointing to `404` pages or pages tagged with `<meta name="robots" content="noindex">`. When a recipe is published in Czech (`reviewed`) but English is still in progress (`draft` or `unverified`), a naive bidirectional `hreflang` tag pointing to `/en/...` triggers crawling errors and de-indexes the Czech original.
6. **Static Hosting Trailing Slash Redirect Roundtrips**: Static directory structures on GitHub Pages serve `index.html` files directly without redirection when requested with a trailing slash (`/cs/recepty/k8m2x9-omacka-mornay/`). Requests omitting the trailing slash (`/cs/recepty/k8m2x9-omacka-mornay`) trigger an extra HTTP 301 redirect roundtrip by the web server, degrading mobile load latency.

---

## Decision

We establish the normative contract for URL paths, identifier composition, link permanence, localized routing, zero-friction language switching, and search engine metadata.

---

### 1. URL Prefix, Trailing Slash, and `BASE_PATH` Contract

#### Symmetric Language Prefixes
All localized content resides under an explicit, symmetric two-letter ISO 639-1 language prefix (`/cs/...` and `/en/...`) prefixed by the deployment `${BASE_PATH}`:

Canonical URL pattern:
```
${SITE_ORIGIN}${BASE_PATH}/${lang}/.../
```

- There are **no unprefixed default language routes**. Czech content is not served at root `/recepty/...`; it is served strictly at `/cs/recepty/...`.
- Symmetric prefixes eliminate routing collisions between static assets, language gateways, and localized content, and prevent treating either language as an architectural second-class citizen.

#### Trailing Slash Termination
Every canonical URL, internal hyperlink, `<link rel="canonical">`, `<link rel="alternate">`, and `sitemap.xml` entry **must terminate with a trailing slash (`/`)**:

- Static directory layout on disk: `dist/${BASE_PATH}/cs/recepty/<uid>-<slug>/index.html`.
- On GitHub Pages and static file servers, requesting a directory with a trailing slash (`.../foo/`) returns `index.html` with status `200 OK` in a single roundtrip.
- Requests without a trailing slash (`.../foo`) trigger a server-side `301 Moved Permanently` to `.../foo/`. Terminating all URLs with `/` eliminates this redundant redirect latency entirely.

#### Root Gateway (`/index.html`)
The root path (`${BASE_PATH}/index.html`) serves a progressive static language gateway:
- Contains `<link rel="alternate" hreflang="x-default" href="${SITE_ORIGIN}${BASE_PATH}/">`.
- Contains `<link rel="alternate" hreflang="cs" href="${SITE_ORIGIN}${BASE_PATH}/cs/">`.
- Contains `<link rel="alternate" hreflang="en" href="${SITE_ORIGIN}${BASE_PATH}/en/">`.
- Emits clean, accessible static HTML links for non-JS clients and crawlers:
  - `<a href="${BASE_PATH}/cs/">Čeština</a>`
  - `<a href="${BASE_PATH}/en/">English</a>`
- Contains an inline, non-blocking client-side script that immediately routes interactive visitors:
  1. Check `localStorage.getItem('culinary_lang')`. If `'cs'` or `'en'`, route to `${BASE_PATH}/${lang}/`.
  2. Otherwise inspect `navigator.languages || [navigator.language]`. If any preferred language begins with `'cs'` or `'sk'`, route to `${BASE_PATH}/cs/`.
  3. If any begins with `'en'`, route to `${BASE_PATH}/en/`.
  4. Fallback default: route to `${BASE_PATH}/cs/`.
  5. Script executes `window.location.replace(...)` to prevent polluting browser back-button history.

---

### 2. Dual Identifier & Mutable Slug Pattern (`<uid>-<slug>`)

Every recipe and technique article possesses two distinct identifiers:
1. **Semantic Folder `id`** (e.g. `mornay-sauce`): A human-readable, lowercase ASCII kebab-case string (`^[a-z0-9]+(-[a-z0-9]+)*$`). It names the recipe directory (`recipes/<id>/recipe.yaml`) and serves as the internal reference key when composed as a sub-recipe component (`component: mornay-sauce`). It is developer-facing and never appears in public URLs.
2. **Immutable Crockford Base32 `uid`** (e.g. `k8m2x9`): A 6-character lowercase Crockford Base32 string (`^[0-9a-hjkmnp-tv-z]{6}$`). It is generated once upon recipe creation and remains immutable for the lifetime of the document.

#### Crockford Base32 Specification
- **Alphabet**: `0123456789abcdefghjkmnpqrstvwxyz` (32 characters, strictly lowercase).
- **Excluded characters**: `i`, `l`, `o`, `u` are omitted to avoid visual confusion (`1`/`i`/`l`, `0`/`o`) and eliminate the accidental generation of English obscenities.
- **Entropy**: $32^6 = 1{,}073{,}741{,}824$ unique combinations, providing collision-free space for cookbook corpuses ($10^2$ to $10^5$ recipes) while remaining ultra-compact (6 characters).

#### Localized Slugs
- Slugs are authored explicitly in `recipe.yaml` per language (`slug.cs`, `slug.en`).
- **Format**: Lowercase ASCII kebab-case (`^[a-z0-9]+(-[a-z0-9]+)*$`). All Czech diacritics are transliterated to plain ASCII (e.g. `omáčka-mornay` → `omacka-mornay`).
- **Never auto-generated from title**: Slugs are never derived dynamically from `title` during build. Titles may change without altering the URL slug.

#### Canonical URL Anatomy
Canonical URLs pair the immutable `uid` with the localized mutable `slug`:
- Czech canonical: `${BASE_PATH}/cs/recepty/<uid>-<slug>/` (e.g. `/cs/recepty/k8m2x9-omacka-mornay/`)
- English canonical: `${BASE_PATH}/en/recipes/<uid>-<slug>/` (e.g. `/en/recipes/k8m2x9-mornay-sauce/`)

#### Three-Layer Link Permanence
When an author edits a slug (e.g. changing `slug.cs` from `omacka-morne` to `omacka-mornay`), link permanence is guaranteed through three cooperative layers:

```
                                  Incoming Request URL
                                           │
                 ┌─────────────────────────┼─────────────────────────┐
                 ▼                         ▼                         ▼
        Permanent Shortlink          Historic Alias            Unknown / Typo
       /cs/recepty/k8m2x9/     /cs/recepty/k8m2x9-old-slug/   /cs/recepty/k8m2x9-xyz/
                 │                         │                         │
          Build-time Stub           Build-time Stub               404.html
         (Static 301 HTML)         (Static 301 HTML)           (UID Rescue Map)
                 │                         │                         │
                 └─────────────────────────┼─────────────────────────┘
                                           ▼
                                 location.replace(...)
                                           ▼
                           Current Canonical Recipe URL
                        /cs/recepty/k8m2x9-omacka-mornay/
```

1. **Permanent Shortlink Layer (Print QR Codes & Inbound Links)**:
   - For every published language, the compiler generates a static redirect stub at:
     - `${BASE_PATH}/cs/recepty/<uid>/index.html`
     - `${BASE_PATH}/en/recipes/<uid>/index.html`
   - **QR Code Target**: All physical A4 print sheets encode this shortlink in their QR code. Even if a recipe's title and slug are renamed repeatedly over a decade, printed recipe sheets remain 100% functional.
2. **Static Alias Stub Layer (`slug_aliases`)**:
   - When a slug is modified, the previous slug is appended to `slug_aliases.<lang>` in `recipe.yaml`:
     ```yaml
     slug:
       cs: omacka-mornay
     slug_aliases:
       cs:
         - omacka-morne
     ```
   - The build process emits a static redirect page at `${BASE_PATH}/cs/recepty/<uid>-omacka-morne/index.html`.
   - The stub contains:
     - `<link rel="canonical" href="${CANONICAL_URL}">`
     - `<meta http-equiv="refresh" content="0; url=${CANONICAL_URL}">`
     - `<meta name="robots" content="noindex,follow">`
     - `<script>location.replace("${CANONICAL_URL}" + location.search + location.hash);</script>`
     - A fallback HTML link `<p>Přesměrování na <a href="${CANONICAL_URL}">...</a></p>`.
   - This preserves all external search engine backlinks and page rank equity without dynamic server infrastructure.
3. **`404.html` Client-Side UID Rescue Layer**:
   - If an inbound URL arrives with an arbitrary, unrecorded, truncated, or mistyped slug (e.g. `/cs/recepty/k8m2x9-vynikajici-morne/`), GitHub Pages serves `404.html`.
   - The root `404.html` contains an inlined, build-generated compact lookup map of all corpus UIDs:
     ```javascript
     const CANONICAL_UIDS = {
       "k8m2x9": { cs: "omacka-mornay", en: "mornay-sauce" }
     };
     ```
   - The rescue script parses the pathname:
     ```javascript
     const match = location.pathname.match(/^(\/.*)?\/(cs|en)\/(recepty|recipes|techniky|techniques)\/([0-9a-hjkmnp-tv-z]{6})(-[^/]*)?(\/.*)?$/);
     if (match) {
       const [_, base, lang, section, uid] = match;
       const targetSlug = CANONICAL_UIDS[uid]?.[lang];
       if (targetSlug) {
         const canonSection = (lang === 'cs') ? (section.startsWith('tech') ? 'techniky' : 'recepty')
                                              : (section.startsWith('tech') ? 'techniques' : 'recipes');
         const targetUrl = (base || '') + '/' + lang + '/' + canonSection + '/' + uid + '-' + targetSlug + '/' + location.search + location.hash;
         location.replace(targetUrl);
       }
     }
     ```
   - The rescue executes instantly before paint, carrying over all query parameters and hash anchors to the current canonical URL.

---

### 3. Complete Route Inventory & Localized Path Segments

The culinary site exposes nine canonical surfaces across Czech and English:

| Surface | Czech Path (`cs`) | English Path (`en`) | Indexing Policy | State Parameters / Notes |
|---|---|---|---|---|
| **Home** | `/cs/` | `/en/` | `index,follow` | Recipe cards, highlights, seasonal picks. |
| **Recipe Detail** | `/cs/recepty/<uid>-<slug>/` | `/en/recipes/<uid>-<slug>/` | `index,follow` (if `reviewed`)<br>`noindex,follow` (if `unverified`) | `?yield=<num>&unit=metric\|imperial`<br>`&variant=<id>#step-<id>` |
| **Permanent Shortlink** | `/cs/recepty/<uid>/` | `/en/recipes/<uid>/` | `noindex,follow` | Static 301 stub redirecting to canonical recipe detail. Target for print QR codes. |
| **Recipe Print Sheet** | `/cs/recepty/<uid>-<slug>/tisk/` | `/en/recipes/<uid>-<slug>/print/` | `noindex,follow` | Dedicated paged A4 sheet. Canonical tag points to Recipe Detail. `@media print` is also supported directly on Recipe Detail. |
| **Cook Mode** | `/cs/recepty/<uid>-<slug>/?mode=cook` | `/en/recipes/<uid>-<slug>/?mode=cook` | Not indexed | Query state `?mode=cook` on Recipe Detail. Never an independent route. Canonical points to clean Recipe Detail URL. |
| **Tag Page** | `/cs/stitky/<tag-id>/` | `/en/tags/<tag-id>/` | `index,follow` | `<tag-id>` is lowercase ASCII kebab-case. |
| **Search & Facets** | `/cs/hledani/` | `/en/search/` | `noindex,follow` | `?q=<text>&tag=<id>&category=<id>`<br>`&difficulty=easy\|medium\|hard`<br>`&allergen_not=<id>&sort=time\|title` |
| **Collection (Bulk)** | `/cs/kolekce/` | `/en/collection/` | `noindex,follow` | Client-assembled cookbook binder. State keyed by `?r=<uid1>,<uid2>,...`. |
| **Collection Print** | `/cs/kolekce/tisk/` | `/en/collection/print/` | `noindex,follow` | Multi-recipe continuous paged A4 printout with table of contents. Keyed by `?r=<uid1>,<uid2>,...`. |
| **Glossary Index** | `/cs/slovnik/` | `/en/glossary/` | `index,follow` | A–Z culinary dictionary index (*Slovník pojmů*). |
| **Glossary Term** | `/cs/slovnik/<term-id>-<slug>/` | `/en/glossary/<term-id>-<slug>/` | `index,follow` | Deep-dive culinary definition and technique links. |
| **Technique Article** | `/cs/techniky/<uid>-<slug>/` | `/en/techniques/<uid>-<slug>/` | `index,follow` | Technique guide (e.g. laminating butter, emulsion stability). Also provides shortlink at `/<lang>/<section>/<uid>/`. |

---

### 4. Zero-Friction Language Switching & Canonical Query/Anchor State

#### Build-Time Counterpart Binding
URLs are **never computed at runtime via naive string substitution** (e.g. regex replacing `/cs/recepty/` with `/en/recipes/`). Such replacements fail when slugs differ between languages or when a counterpart recipe does not exist.

Instead, during static site compilation, every generated HTML page calculates its exact sibling URL and embeds it into the DOM header language switch anchor:

```html
<!-- On page: /cs/recepty/k8m2x9-omacka-mornay/ -->
<a id="lang-switch"
   href="/culinary-spec/en/recipes/k8m2x9-mornay-sauce/"
   hreflang="en"
   lang="en"
   aria-label="Switch to English">English</a>
```

#### Canonical Parameter and Anchor Vocabulary
All URL query parameter keys, query enum values, and step fragment identifiers are strictly **language-neutral ASCII**. They are never translated:

| Parameter / Anchor | Allowed Syntax | Meaning |
|---|---|---|
| `yield` | `^[0-9]+(\.[0-9]+)?$` (e.g. `?yield=6`) | Target scaled recipe yield. |
| `unit` | `'metric' \| 'imperial'` | Display measurement system toggle. |
| `variant` | `^[a-z0-9]+(-[a-z0-9]+)*$` (e.g. `?variant=vegetarian`) | Selected recipe variant ID. |
| `mode` | `'cook'` | Activates cooking view mode. |
| `q` | UTF-8 search query | Search text query. |
| `tag` | Controlled kebab-case `tag-id` | Filter by tag. |
| `category` | Controlled vocabulary (`sauce`, `main`, etc.) | Filter by category. |
| `difficulty` | `'easy' \| 'medium' \| 'hard'` | Filter by difficulty rating. |
| `allergen_not` | EU 1169/2011 allergen slug (e.g. `gluten`) | Exclusion filter for allergens. |
| `sort` | `'time' \| 'title'` | Result ordering. |
| `r` | Comma-separated 6-char UIDs (`k8m2x9,p4w9x2`) | Collection recipe list. |
| `#step-<id>` | `id` matches `steps[].id` in `recipe.yaml` | Focus DOM anchor for step. |

#### Progressive Language Switch Handler
Because query parameters and step IDs are identical across languages (guaranteed by ADR-0002 and ADR-0006 structural language parity), switching languages preserves the complete kitchen context.

A lightweight progressive enhancement handler intercepts clicks on `#lang-switch`:

```javascript
document.getElementById('lang-switch')?.addEventListener('click', (event) => {
  const switchLink = event.currentTarget;
  if (switchLink.getAttribute('aria-disabled') === 'true') {
    event.preventDefault();
    return;
  }
  // Construct destination URL from authored counterpart href
  const dest = new URL(switchLink.href, window.location.origin);
  // Forward current interactive kitchen state exactly
  dest.search = window.location.search;
  dest.hash = window.location.hash;
  switchLink.href = dest.toString();
  // Remember reader preference for root gateway
  const targetLang = switchLink.getAttribute('hreflang');
  if (targetLang) {
    localStorage.setItem('culinary_lang', targetLang);
  }
});
```

*Example Transition*:
- User is on Czech recipe: `/cs/recepty/k8m2x9-omacka-mornay/?yield=6&unit=imperial&mode=cook#step-make-roux`
- Clicks `English`:
- Browser arrives on English recipe: `/en/recipes/k8m2x9-mornay-sauce/?yield=6&unit=imperial&mode=cook#step-make-roux`
- Result: 6 portions, imperial measurements, cook mode active, scrolled to the roux step. Zero lost state.

---

### 5. Single-Language Recipes, `unverified` (`noindex`), `hreflang`, and `sitemap.xml`

#### Single-Language Authoring Lifecycle
Recipes are authored natively in Czech and English. A recipe may be created in Czech first, while English remains unwritten (`state.en` is omitted or `'draft'`).
- **Omitted or `'draft'`**: Zero files are generated in that language's directory tree (ADR-0001). Direct navigation returns `404.html`.
- **Language Switcher on Single-Language Pages**:
  - The language switcher link renders in a disabled state: `<a id="lang-switch" aria-disabled="true" class="is-disabled" href="/en/" title="Recept zatím není dostupný v angličtině">English <span class="badge">pouze česky</span></a>`.
  - Accessible tooltip informs the reader: *"Recept je zatím dostupný pouze v češtině / This recipe is only available in Czech"*.
  - Clicking the disabled switch navigates safely to the other language's home page (`/en/`) rather than throwing a 404.

#### Strict Consistency: `noindex`, `hreflang`, and `sitemap.xml`
Search engines enforce strict rules regarding internationalization tags:
1. Every URL listed in `hreflang` alternate tags must return HTTP `200` and be indexable.
2. Directing an `hreflang` tag to a `noindex` page or a `404` page produces search console errors and degrades domain indexing authority.

To guarantee 100% SEO integrity, Culinary Spec links metadata strictly across publication states:

```
  Publication State (CS / EN)      noindex Header           hreflang Pair           sitemap.xml
──────────────────────────────────────────────────────────────────────────────────────────────────
  CS: reviewed, EN: reviewed       None (index,follow)      Emitted (cs, en, x-def) Both URLs included
  CS: reviewed, EN: unverified     EN gets noindex          None emitted            CS included; EN omitted
  CS: reviewed, EN: draft          EN not generated (404)   None emitted            CS included; EN omitted
  CS: unverified, EN: unverified   Both get noindex         None emitted            Both omitted
```

1. **`state: unverified`**:
   - Emits `<meta name="robots" content="noindex,follow">`.
   - Never referenced in any `<link rel="alternate" hreflang="...">` tag on any page.
   - Never included in `sitemap.xml`.
2. **`state: reviewed`**:
   - Emits standard indexing metadata (or inherits `index,follow`).
   - If and only if **both** `cs` and `en` states are `'reviewed'`:
     - Emits reciprocal `hreflang` tags on both pages:
       ```html
       <link rel="alternate" hreflang="cs" href="https://example.com/culinary-spec/cs/recepty/k8m2x9-omacka-mornay/">
       <link rel="alternate" hreflang="en" href="https://example.com/culinary-spec/en/recipes/k8m2x9-mornay-sauce/">
       <link rel="alternate" hreflang="x-default" href="https://example.com/culinary-spec/cs/recepty/k8m2x9-omacka-mornay/">
       ```
     - Emits corresponding `<xhtml:link rel="alternate" hreflang="..." />` entries in `sitemap.xml`.
   - If only one language is `'reviewed'`, that page emits **no** `hreflang` tags, and only that page's URL appears in `sitemap.xml`.
3. **Canonical URL Hygiene**:
   - The `<link rel="canonical">` always points to the clean, parameter-free canonical URL:
     `<link rel="canonical" href="${SITE_ORIGIN}${BASE_PATH}/${lang}/recepty/<uid>-<slug>/">`.
   - URL parameters (`?yield=...`, `?mode=cook`) never pollute the canonical link tag.

---

## Verification & Mutant Matrix

The verification harness (`mise run verify`) validates route generation, link consistency, and internationalization integrity through deterministic rules and required mutants:

### 1. Verification Suite Rules

| Rule ID | Severity | Deterministic Condition |
|---|---|---|
| `ROUTE_TRAILING_SLASH` | Error (Exit 1) | Every emitted route path, internal hyperlink, canonical URL, and sitemap entry terminates with `/`. |
| `ROUTE_SYMMETRIC_PREFIX` | Error (Exit 1) | All content routes begin with `${BASE_PATH}/cs/` or `${BASE_PATH}/en/`. No unprefixed content routes exist. |
| `RECIPE_UID_FORMAT` | Error (Exit 1) | Every recipe and article `uid` matches regex `^[0-9a-hjkmnp-tv-z]{6}$` (lowercase Crockford Base32) and is globally unique corpus-wide. |
| `SLUG_FORMAT` | Error (Exit 1) | Authored `slug.cs` and `slug.en` match `^[a-z0-9]+(-[a-z0-9]+)*$` and contain no diacritics, uppercase letters, or non-ASCII characters. |
| `SHORTLINK_EMISSION` | Error (Exit 1) | For every recipe with a published language, a static redirect stub exists at `/<lang>/<section>/<uid>/index.html` pointing to current canonical. |
| `SLUG_ALIAS_INTEGRITY` | Error (Exit 1) | Every entry in `slug_aliases` emits a static redirect stub to the current canonical URL. No duplicate alias across different recipes. |
| `LANG_SWITCH_TARGET_INTEGRITY` | Error (Exit 1) | Every page's `#lang-switch` link points to a valid generated counterpart URL or carries `aria-disabled="true"` with fallback link to home. |
| `CANONICAL_PARAM_KEYS` | Error (Exit 1) | All client query parameters and step fragment IDs adhere strictly to the ASCII vocabulary (`yield`, `unit`, `variant`, `mode`, `q`, `tag`, `category`, `difficulty`, `allergen_not`, `sort`, `r`). No localized param names. |
| `HREFLANG_SEO_PARITY` | Error (Exit 1) | `hreflang` tags and `sitemap.xml` entries are generated if and only if target pages have `state == 'reviewed'`. Zero `hreflang` links to `unverified` or `draft` pages. |
| `RESCUE_404_MAP` | Error (Exit 1) | `404.html` contains an exhaustive lookup table mapping every corpus `<uid>` to its active canonical slug per language. |

### 2. Required Mutants under `tests/corpus/mutants/`

`mise run verify` must fail if any of the following regression mutants pass or are missing:

1. `mutant-missing-trailing-slash`: An internal route link generated without a trailing slash (`/cs/recepty/k8m2x9-omacka-mornay`). Must fail `ROUTE_TRAILING_SLASH`.
2. `mutant-localized-query-param`: A cook mode or yield control emitting localized query keys (`?porce=4` or `?jednotka=imperial`). Must fail `CANONICAL_PARAM_KEYS`.
3. `mutant-unreviewed-hreflang`: A recipe with `state: { cs: reviewed, en: unverified }` that emits `hreflang="en"` on the Czech page. Must fail `HREFLANG_SEO_PARITY`.
4. `mutant-reused-slug-alias`: Two distinct recipes claiming the same slug alias in Czech (`slug_aliases.cs: ['klasicka-omacka']`). Must fail `SLUG_ALIAS_INTEGRITY`.
5. `mutant-404-missing-uid`: A newly committed recipe whose `uid` is omitted from the `404.html` rescue map. Must fail `RESCUE_404_MAP`.
6. `mutant-invalid-slug-charset`: A recipe with `slug.cs: 'omáčka-mornay'` containing diacritics. Must fail `SLUG_FORMAT`.
7. `mutant-asymmetric-route`: A recipe emitted at `/recepty/k8m2x9-omacka-mornay/` without language prefix. Must fail `ROUTE_SYMMETRIC_PREFIX`.

---

## Consequences

- **Zero Link Rot**: Recipe URLs can safely update their descriptive slugs without breaking existing links, bookmarked URLs, or search engine citations.
- **Permanent Print QR Codes**: Physical A4 recipe printouts encode the immutable shortlink (`.../<uid>/`), remaining permanently scannable even across major content rewrites.
- **Frictionless Kitchen Experience**: Cooks can freely switch languages while preparing a dish without losing their active step, timer, portion scaling, or imperial unit preferences.
- **Bulletproof Static SEO**: Search engines crawl clean trailing-slash URLs with zero redirect hops. `hreflang` tags and sitemaps never reference unverified drafts, preventing indexing penalties.
- **Fail-Closed Verification**: The site cannot be built with broken language switch links, malformed slugs, or localized parameter drift.

---

## Rejected

- **Unprefixed Root Language (e.g. Czech at `/`, English at `/en/`)**: Rejected. Treating one language as default complicates static routing, asset paths, and deployment under repository subpaths, while making the secondary language an architectural afterthought.
- **Slug-Only URLs (e.g. `/cs/recepty/omacka-mornay/`)**: Rejected. Any modification to a recipe title or slug breaks external links and permanently invalidates physical QR codes on printed recipe sheets.
- **Dynamic Build-Time Slug Generation from Title**: Rejected. Recomputing slugs automatically from titles causes silent URL churn whenever minor typos or punctuation in titles are fixed. Slugs must be explicitly authored.
- **Dedicated Cook Mode URL Path (e.g. `/cs/recepty/<uid>-<slug>/cook/`)**: Rejected. Cook mode is a transient presentation state, not a unique document. Splitting it into an independent URL dilutes search ranking, creates duplicate content issues, and complicates bookmarking.
- **Localized Query Parameter Keys (e.g. `?porce=4&jednotka=metricka`)**: Rejected. Translating parameter names breaks query pass-through during language switching, requiring complex mapping dictionaries and brittle regex parsers in client JS.
- **Emitting `hreflang` Tags for `unverified` Translations**: Rejected. Google Search explicitly penalizes sites with `hreflang` annotations pointing to `noindex` or 404 pages.
- **Runtime Client-Side String Substitution for Language Switching**: Rejected. Naive URL path rewriting (`url.replace('/cs/', '/en/')`) breaks completely when localized slugs differ between languages or when a translation does not exist.
