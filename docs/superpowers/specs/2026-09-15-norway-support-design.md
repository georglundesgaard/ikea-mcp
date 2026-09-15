# IKEA Norway (NO) support — design

**Date:** 2026-09-15 · **Issue:** georglundesgaard/ikea-mcp#17 · **Target:** one PR to `taejin5314/ikea-mcp` from branch `feat/norway-support`

## Goal

Make Norway a first-class country for every tool, on par with `US` and `CA`: the stock tools accept `countryCode: "NO"`, `list_stores` knows the Norwegian stores, Norwegian store IDs render with a readable label, and product search defaults to Norwegian when the country is Norway.

## Non-goals

- Using the country-wide `/ru/{cc}` availabilities endpoint (issue #18).
- Exposing city/region as separate fields in `list_stores` output, or a `type` field for non-stock locations (rest of issue #10).
- Removing the dead `countryCode` parameter from `check_store_stock` / `getStoreStock` (issue #11).
- Fixing the unguarded property chain in `check_cart_availability` (issue #1). This PR does not touch that file.
- Other countries. After this change they are data-only additions, each needing its own verification pass.
- A version bump. The upstream maintainer assigns versions.

## Verified facts (live APIs, 2026-09-15)

- Search `sik.search.blue.cdtapps.com/{cc}/{lang}/search-result-page` works for `no/no`, `NO/no` and `no/en`. `no/no` returns Norwegian names and `NOK` prices. Item numbers are the global 8-digit numbers (`itemNo === itemNoGlobal` for the items checked).
- Stock `api.ingka.ikea.com/cia/availabilities/sto/{storeId}` (the path the code already uses, same public `X-Client-Id`) works for Norwegian stores. `/sto/007?itemNos=00263850` → quantity 56, `HIGH_IN_STOCK`. Multi-item requests work.
- Every availability row, in every market, carries `buyingOption.cashCarry.range.inRange: boolean`. An item that is not sold at that store or market has `inRange: false`, no `quantity`, and **no** entry in `errors`. Today that projects to `quantity: null, availableForCashCarry: false, errors: []`, which is indistinguishable from "unknown".
- Norwegian business units returned by the stock API and confirmed from the `"id":"NNN","buClassification":{"code":"…"}` blob on each `ikea.com/no/no/stores/<slug>/` page:

| Code | Name | City (label) | Postal place | Class | Cash & carry |
|---|---|---|---|---|---|
| `007` | Sørlandet | Kristiansand | Kristiansand | STORE | yes |
| `091` | Slependen | Asker | Billingstad | STORE | yes |
| `095` | Furuset | Oslo | Oslo | STORE | yes |
| `126` | Forus | Stavanger | Sandnes | STORE | yes |
| `371` | Leangen | Trondheim | Trondheim | STORE | yes |
| `390` | Ringsaker | Hamar | Furnes | STORE | yes |
| `441` | Åsane | Bergen | Nyborg | STORE | yes |
| `722` | på Karl Johan | Oslo | | PAOP | no |
| `1352` | Østfold | Rolvsøy | | SMALL | no |
| `1213` | Ski | Ski | | plan & order point | no |
| `1214` | Haugesund | | | plan & order point | no |
| `1215` | Ålesund | | | plan & order point | no |
| `1216` | Porsgrunn | | | plan & order point | no |
| `1217`, `1218` | unknown | | | | no |

The "City (label)" column uses the metropolitan city people associate the store with (IKEA's own page title for `390` is "IKEA Ringsaker, Hamar"); the postal place is recorded here for reference only.

## Design

### 1. New module `src/data/countries.ts`

Single source of truth for supported countries and their default search language.

```ts
export const COUNTRY_CODES = ["US", "CA", "NO"] as const;
export type CountryCode = (typeof COUNTRY_CODES)[number];

export const DEFAULT_LANG_CODE: Record<CountryCode, string> = { US: "en", CA: "en", NO: "no" };

/** Default search language for a country; "en" for countries outside the catalog. */
export function defaultLangCode(countryCode: string): string;
```

Adding a country later means adding one entry here, records in `stores.ts`, and README rows. Nothing else.

### 2. Structured store catalog in `src/data/stores.ts`

Replace the `Record<string, string>` of display strings and the `CA_LABEL_RE` regex with records:

```ts
export interface StoreRecord {
  storeId: string;
  /** Store name when it differs from the city (Norwegian stores). */
  name?: string;
  city: string;
  /** State or province code (US, CA). */
  region?: string;
  countryCode: CountryCode;
}
export const STORES: readonly StoreRecord[];
```

Derived, exported, and kept for backward compatibility:

- `STORE_LABELS: Record<string, string>` — `storeId → label`.
- `storeLabel(storeId): string | undefined`.
- `storeIdsByCountry(countryCode: CountryCode): string[]` — declaration order.
- New: `allStoreIds(): string[]` — declaration order; replaces `Object.keys(STORE_LABELS)` in tools.
- New: `storeCountry(storeId): CountryCode | undefined`.

**Label rule** (one function, no per-country branches):

```
parts = [name, city, region, countryCode === "US" ? undefined : countryCode].filter(present)
label = `${storeId} (${parts.join(", ")})`
```

This reproduces every existing label exactly (`"026 (Canton, MI)"`, `"003 (Richmond, BC, CA)"`) and yields `"095 (Furuset, Oslo, NO)"` for Norway. US having no country suffix is a historical quirk that is preserved, not a rule to copy.

**Data.** The 52 US and 15 CA records are mechanically derived from the current labels (city and region parsed from `"City, RG"` / `"City, RG, CA"`); the existing explanatory comments about excluded and unknown IDs stay. Norway adds the seven `STORE` rows from the table above, with `name` set. The eight non-stock Norwegian codes are listed in a comment with their classification, not as records.

**Ordering change (accepted).** `Object.keys()` on the old object returned canonical-integer keys first (`103`, `145`, …, `1129`) and then `026`, `027`, …, `003`, … in insertion order. The new helpers return declaration order (US, then CA, then NO). No documented behaviour depends on this; `compare_store_stock` without `sortBy` will list stores in catalog order instead of the JavaScript key-order quirk.

### 3. Schemas (`src/schemas/index.ts`)

- `CountryCodeEnum = z.preprocess(toUpper, z.enum(COUNTRY_CODES))`. Every stock tool that filters by country accepts `NO` with no further change.
- `CountryCode2` (free-form 2-letter code for search) is unchanged.
- `SearchProductsInput` and `GetProductDetailsInput`: `langCode` becomes optional and is filled with `defaultLangCode(countryCode)` in an object-level `.transform`, so the parsed type still has `langCode: string`. Explicit `langCode` always wins. Country outside the catalog (e.g. `SE`) falls back to `en`, as today.

### 4. `inRange` in stock projections (`src/services/ikea.ts`)

- Type `StockCashCarry` gains `range?: { inRange: boolean }`.
- `projectStock` returns a new field `inRange: boolean | null` (`a?.buyingOption?.cashCarry?.range?.inRange ?? null`) directly after `availableForCashCarry`.
- `projectMultiItemStock` rows gain the same field; `null` in the 405 and not-found branches.
- No synthetic error entry. The field plus the tool descriptions ("`inRange: false` means the item is not sold at that store or in that market") are enough for a client to explain the difference between "out of stock" and "not sold here".
- Tools that spread `projectStock` (`check_store_stock`, `compare_store_stock`) get the field for free. `check_multi_item_stock` maps fields explicitly and adds `inRange: row.inRange`. `find_best_store_for_item`, `find_best_store_for_cart` and `check_cart_availability` output shapes are unchanged.

### 5. Tools

- Hand-written JSON schemas: every `countryCode` `enum` becomes `[...COUNTRY_CODES]` so it cannot drift from the zod enum. `langCode` loses `default: "en"` and gains a `description`: `"Two-letter language code. Defaults to the country's language: en for US and CA, no for NO."`
- Descriptions: replace `('US' or 'CA')` with `('US', 'CA' or 'NO')` in `list_stores`, `compare_store_stock`, `find_best_store_for_item`, `find_best_store_for_cart`. `check_store_stock`, `compare_store_stock` and `check_multi_item_stock` descriptions mention `inRange`.
- `list_stores`: iterate `STORES` (filtered by country when given) and emit `{ storeId, storeLabel, countryCode }` from the record. The nested-ternary CA set is deleted. Output shape unchanged.
- `find_best_store_for_item`, `find_best_store_for_cart`: `Object.keys(STORE_LABELS)` → `allStoreIds()`.
- `compare_store_stock`: unchanged apart from the description; the `apiCountry` lowercasing is left for issue #11.

### 6. Tests (`test/unit.test.ts`, append; `test/fixtures/`)

- `test/fixtures/store-labels-v1.6.2.json`: the exact `STORE_LABELS` object from the current code, generated before the refactor. A test asserts that the derived US and CA labels deep-equal this fixture, so the label refactor is provably lossless.
- `storeIdsByCountry("NO")` is exactly `["007","091","095","126","371","390","441"]`; US, CA and NO partition `allStoreIds()` with no overlap; the old two-way partition test is replaced.
- `storeLabel("095") === "095 (Furuset, Oslo, NO)"`; `storeCountry("095") === "NO"`; `storeCountry("999") === undefined`.
- `CountryCodeEnum` (through `ListStoresInput`) accepts `"no"` → `"NO"` and rejects `"SE"`.
- `SearchProductsInput`: `{ query, countryCode: "NO" }` → `langCode: "no"`; `{ query }` → `"en"`; `{ query, countryCode: "NO", langCode: "en" }` → `"en"`. Same for `GetProductDetailsInput`.
- `projectStock` with `range.inRange: false` and no quantity → `inRange: false, quantity: null`; with `inRange: true` → `true`; with no `range` → `null`. `projectMultiItemStock` equivalents, including `null` on the 405 branch.
- `listStoresTool.handler({ countryCode: "NO" })` returns seven rows all tagged `NO`; `handler({})` returns 74 rows.
- `smoke.mjs`: four Norway steps (search with `countryCode: "NO"` expecting `NOK`; `check_store_stock` at `095` expecting a label ending in `NO)`; `compare_store_stock` with `countryCode: "NO"` expecting 7 rows; `list_stores` with `countryCode: "NO"` expecting 7). Smoke runs stay manual (`npm run smoke`).

### 7. Documentation (`README.md`)

Minimal, scoped edits: Norway row in *Supported countries* (7 stores) and in *Store IDs*; "US and Canada only" → "US, Canada and Norway"; the `langCode` default rule in the `search_products` and `get_product_details` tables; `inRange` in the `check_store_stock` output example and a one-line explanation; a fifth example workflow (search Norway → compare across `countryCode: "NO"`); `list_stores` description mentions `NO`. The broader README sync is issue #13.

## Backward compatibility

- All existing US and CA labels are byte-identical (enforced by the fixture test).
- All output shapes are unchanged except for the additive `inRange` field on `check_store_stock`, `compare_store_stock` and `check_multi_item_stock` rows.
- `countryCode` enums are widened, never narrowed. `langCode` defaults change only when `countryCode` is `NO`.
- Iteration order of the catalog changes as described in §2.

## Acceptance

- `compare_store_stock { itemNo: "00263850", countryCode: "NO" }` returns seven rows with real quantities and Norwegian labels.
- `find_best_store_for_item { itemNo: "00263850", countryCode: "NO" }` returns ranked Norwegian stores.
- `search_products { query: "billy", countryCode: "NO" }` returns Norwegian names and `NOK` without an explicit `langCode`.
- `check_store_stock { itemNo: "20522046", storeId: "095" }` (a US-only item) shows `inRange: false`.
- `npm test`, `npm run typecheck`, `npm run build` pass; `npm run smoke` prints OK for the four Norway steps.
