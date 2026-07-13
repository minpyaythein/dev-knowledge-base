# Store money as integer minor units + a separate ISO 4217 currency column

Floats corrupt money math; store cents/yen as integers and keep the currency code beside the amount, because an amount without its currency is meaningless.

## Why it matters

`0.1 + 0.2 !== 0.3` in floating point — comparisons like "is the price at or below the target?" silently misfire on float-stored prices. And multi-currency data with an assumed currency ("it's probably USD") produces alerts that compare ¥5000 against $50.

## How it works

- **Integer minor units**: $12.34 → `1234`, ¥5000 → `5000`. All comparison and storage happens on integers; division by 10^digits happens only at display time.
- **Minor-unit digits vary by currency.** Most currencies have 2 decimal digits, but zero-decimal currencies (JPY, KRW) have 0 — ¥5000 is `5000`, not `500000`. Keep a set of zero-decimal codes and consult it both when parsing and when formatting.
- **Currency is a separate column**, validated as a plausible ISO 4217 code (three ASCII letters, uppercased). Anything else → unknown, stored as `null`.
- **Never compare across currencies.** A target-price check requires `snapshot.currency === target.currency`; otherwise skip the comparison entirely rather than convert or assume.

Two parsing gotchas worth encoding:

- When parsing a price out of free text, **require an explicit currency marker** (symbol like ¥/€/$ or a code like USD) before accepting any number. Bare numbers in page text are usually quantities, ratings, or dates — not prices.
- At display time, `Intl.NumberFormat(locale, { style: 'currency', currency })` handles symbols and decimal digits for you — wrap it in try/catch with a plain `"12.34 USD"` fallback for codes Intl doesn't know.

## Example

Page text: `"Price: ¥5,000 (tax included)"`. The ¥ symbol maps to JPY (zero-decimal), commas are stripped, `5000` is stored as `price=5000, currency='JPY'`. A user target of ¥5,500 is stored as `5500 JPY`; the alert check is a plain integer comparison `5000 <= 5500` — no float, no rounding drift.
