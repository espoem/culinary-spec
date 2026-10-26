# ADR-0012: Structured data and indexing per language

- Status: Accepted
- Date: 2026-10-26
- Decided in: ticket *SEO and JSON-LD per language* (#12)

## Context

Google's recipe documentation, read from the primary source rather than recalled:
`name` and `image` are required, and the image must be crawlable, must represent
the marked-up dish, and should be offered at 16:9, 4:3 and 1:1 above 50 000 total
pixels. Everything else is recommended, including `author`, `cookTime` only in
company with `prepTime`, `datePublished`, `description`, `keywords`, `nutrition`
(where `calories` requires `recipeYield` in servings), `recipeCategory`,
`recipeCuisine`, `recipeIngredient`, `recipeInstructions` (preferring `HowToStep`,
with `HowToSection` for groups), `recipeYield`, `totalTime`, `suitableForDiet`,
`aggregateRating` and `video`.

ADR-0007 already fixes the crawl surface: trailing slashes everywhere,
`index,follow` only for `reviewed` recipes, `noindex,follow` for `unverified`, tag
pages indexed from two reviewed recipes, search and print excluded, and `hreflang`
that must never point at a `noindex` or `404` page.

Two facts about this project shape the decisions: a recipe without an image cannot
produce a rich result, and a recipe document already carries per-language
publication state, so the markup can disagree with itself across languages.

## Decision

1. **An image is a precondition of `reviewed`.** A recipe with no image, or with an
   image lacking provenance, stays `unverified`. One gate, not two: the provenance
   rules from ADR-0006 already require a licence for every asset.
2. **`recipeInstructions` is `HowToStep[]`**, with `name` = the authored `short`
   cue, `text` = the full prose, and `url` = the canonical recipe URL plus
   `#step-<id>`. When a recipe has two or more phases the steps are wrapped in
   `HowToSection` per phase; a single-phase recipe stays flat.
3. **One `@graph` per page.** `Organization` and `WebSite` (with `SearchAction`)
   are emitted on the site root; `Recipe` references the publisher by `@id` and
   carries `mainEntityOfPage`, `inLanguage`, `author` as the `Organization` (never
   an invented person), and `NutritionInformation` only when nutrition exists.
4. **`aggregateRating` and `video` are never emitted in v1.** There are no ratings;
   a zero rating is worse than none, and a video property with no video is a lie.
5. **`suitableForDiet`** is emitted only for diets with a one-to-one schema.org
   mapping and only when `allergens_detected` does not contradict the claim.
6. **`keywords`** comes from the visible tag chips, localized, so the markup and
   the page agree.
7. **`unverified` pages carry no `Recipe` entity at all** — only the site graph and
   a `WebPage`. A stale rich result must not outlive the state that produced it.
8. **Sitemap**: one `<url>` per language, `xhtml:link` alternates only for pairs
   where both languages are `reviewed`, and `lastmod` from a new optional
   `updated_at` field falling back to `published_at`.
9. **The document gains `updated_at`** (optional, `YYYY-MM-DD`) in the header of
   ADR-0006, because a sitemap cannot invent a modification date.

## Verification

`verify` treats the structured data as a machine contract and checks it against
the rendered page, not against itself. All of the following are errors:

- The JSON-LD parses and every Google-required property is present.
- `image` resolves (HTTP 200 in the built site), meets the size and aspect floor,
  and is the image actually shown on the page.
- `recipeIngredient` equals the rendered ingredient list line by line.
- `recipeInstructions[].text` equals the rendered prose, and `name` equals `short`.
- `inLanguage` matches the route prefix.
- No `Recipe` entity on a `noindex` page.
- No `aggregateRating` and no `video` anywhere in v1.
- `nutrition.calories` implies `recipeYield` expressed in servings.
- `suitableForDiet` contradicts nothing in `allergens_detected` (`DIET_ALLERGEN_CONFLICT`
  already exists for the document; this is the markup twin).
- A `reviewed` recipe without an image fails as `IMAGE_MISSING_FOR_REVIEWED`.

## Consequences

- The content pipeline must produce a licensed image before a recipe can be
  indexed, which makes image production part of the review step, not a later polish.
- Every recipe needs a `short` cue before it can be a matrix or a rich result, which
  is already required by ADR-0011.
- Two languages can be in different states without producing an invalid graph,
  because the alternates are only emitted for reviewed pairs.

## Rejected

- Emitting `Recipe` on `unverified` pages.
- A zero or placeholder `aggregateRating`.
- A single `text` blob for instructions.
- `keywords` invented from prose instead of the visible tags.
- `hreflang` on pairs where one side is not `reviewed`.
