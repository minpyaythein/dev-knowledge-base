# Generic product-data extraction — cascade from structured to guessed

You can watch *any* product page without per-site scrapers by trying structured data first (JSON-LD → meta/OpenGraph → microdata) and only falling back to text heuristics last.

## Why it matters

Writing one scraper per shop doesn't scale and breaks on every redesign. Most e-commerce pages already publish machine-readable product data for Google/social previews — the same data a scraper would dig for. A cascade gets the reliable source when it exists and degrades gracefully when it doesn't.

## How it works

Each strategy is a function `(parsedHtml) → { title?, price?, currency?, inStock? }`. Run them in fixed order; **earlier strategies win conflicts** — a later strategy only fills fields still missing:

| Order | Strategy | Source | Trust |
|---|---|---|---|
| 1 | JSON-LD | `<script type="application/ld+json">` with `@type: Product` | highest — explicit schema |
| 2 | meta | OpenGraph/meta tags (`og:price:amount`, etc.) | high |
| 3 | microdata | `itemprop` attributes | high |
| 4 | heuristics | visible text keywords ("in stock", "在庫あり", price patterns) | lowest — guessing |

Three rules make the cascade safe:

- **Missing = `undefined`/`null`, never a default.** If no strategy finds the price, the price is unknown — don't substitute 0 or "USD".
- **Price and currency travel as a pair.** A currency without its price (or vice versa) is meaningless; take both from the same strategy or neither.
- **Record provenance.** Keep a `methods[]` list of which strategies contributed. It makes results debuggable ("why does this show $0?") and shows users how trustworthy a result is.

The pipeline being an ordered list is also the upgrade seam: an AI/LLM extractor is just one more entry appended after heuristics, only paying for AI when everything cheaper failed.

## Example

A Shopify page has JSON-LD with price + availability but no useful meta tags: JSON-LD fills everything, `methods = ['json-ld']`, heuristics never influence the result. A hand-made shop page has only an `og:title` and the text "SOLD OUT": meta fills the title, heuristics set `inStock: false`, price stays `null` — honestly unknown rather than guessed.
