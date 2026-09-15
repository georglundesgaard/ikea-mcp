# IKEA Norway (NO) Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Norway (`NO`) a supported country for every tool: stock tools accept `countryCode: "NO"`, the catalog knows the seven Norwegian stores with readable labels, search defaults to Norwegian for Norway, and stock rows expose `inRange`.

**Architecture:** A new `src/data/countries.ts` is the single source of truth for supported countries and their default language. `src/data/stores.ts` changes from a map of display strings to an array of structured records from which labels and lookups are derived (existing US/CA labels stay byte-identical, proven by a fixture test). Schemas and tool JSON schemas read the country list from the new module so they cannot drift. `projectStock`/`projectMultiItemStock` gain an additive `inRange` field.

**Tech Stack:** TypeScript 5 (ESM, `Node16` module resolution, `.js` import suffixes), zod 3, `@modelcontextprotocol/sdk`, `node:test` via `tsx`. Node ≥ 20.

**Spec:** `docs/superpowers/specs/2026-09-15-norway-support-design.md`

## Global Constraints

- Work on branch `feat/norway-support` in `/Users/georg/playground/ikea-mcp/clones/ikea-mcp-agent-1`. Never commit to `master`.
- All existing US and CA store labels must remain byte-identical (e.g. `"026 (Canton, MI)"`, `"003 (Richmond, BC, CA)"`).
- Norwegian labels have the form `"<storeId> (<name>, <city>, NO)"`, e.g. `"095 (Furuset, Oslo, NO)"`.
- Do not modify `src/tools/check_cart_availability.ts`, `src/utils/http.ts`, `src/http.ts`, `src/server.ts`, or `package.json` version. Those belong to other issues.
- Imports between `src` files use the `.js` suffix (`from "../data/countries.js"`), even though the sources are `.ts`.
- Every code task ends with `npm test`, `npm run typecheck` and `npm run build` passing.
- Tests are appended to `test/unit.test.ts` (the repo's single test file). Add imports at the top of the file only when a task says so.
- Commit messages: conventional prefix (`feat:`, `refactor:`, `test:`, `docs:`) and end with the line `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- Do not run `npm run smoke` except where a task explicitly says so (it hits the live IKEA API).
- Do not open a pull request. The user decides when to send it upstream.

---

### Task 1: Countries module

**Files:**
- Create: `src/data/countries.ts`
- Test: `test/unit.test.ts` (append)

**Interfaces:**
- Consumes: nothing.
- Produces: `COUNTRY_CODES: readonly ["US","CA","NO"]`, `type CountryCode = "US" | "CA" | "NO"`, `DEFAULT_LANG_CODE: Record<CountryCode, string>`, `isCountryCode(value: string): value is CountryCode`, `defaultLangCode(countryCode: string): string`. Tasks 2, 3 and 5 import these.

- [ ] **Step 1: Add the import and failing tests**

At the top of `test/unit.test.ts`, directly after the existing line `import { storeIdsByCountry, STORE_LABELS } from "../src/data/stores.js";`, add:

```ts
import { COUNTRY_CODES, DEFAULT_LANG_CODE, defaultLangCode, isCountryCode } from "../src/data/countries.js";
```

Append at the end of `test/unit.test.ts` (after the `if (process.env.CANARY) { ... }` block):

```ts
// ── countries ─────────────────────────────────────────────────────────────────

test("countries — COUNTRY_CODES lists US, CA and NO in that order", () => {
  assert.deepEqual([...COUNTRY_CODES], ["US", "CA", "NO"]);
});

test("countries — every country has a default language", () => {
  for (const cc of COUNTRY_CODES) {
    assert.equal(typeof DEFAULT_LANG_CODE[cc], "string");
  }
  assert.equal(DEFAULT_LANG_CODE.NO, "no");
  assert.equal(DEFAULT_LANG_CODE.US, "en");
  assert.equal(DEFAULT_LANG_CODE.CA, "en");
});

test("countries — defaultLangCode falls back to en for unknown countries", () => {
  assert.equal(defaultLangCode("NO"), "no");
  assert.equal(defaultLangCode("US"), "en");
  assert.equal(defaultLangCode("SE"), "en");
  assert.equal(defaultLangCode(""), "en");
});

test("countries — isCountryCode is a type guard over COUNTRY_CODES", () => {
  assert.equal(isCountryCode("NO"), true);
  assert.equal(isCountryCode("no"), false);
  assert.equal(isCountryCode("SE"), false);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test 2>&1 | tail -20`
Expected: the run aborts with `Cannot find module '../src/data/countries.js'` (or an ERR_MODULE_NOT_FOUND error) because the module does not exist yet.

- [ ] **Step 3: Create the module**

Create `src/data/countries.ts`:

```ts
// Supported countries for the stock tools and the store catalog.
// Adding a country: add it here, add its stores to ./stores.ts, add README rows.

export const COUNTRY_CODES = ["US", "CA", "NO"] as const;

export type CountryCode = (typeof COUNTRY_CODES)[number];

/** Default language for the IKEA search API when the client does not pass langCode. */
export const DEFAULT_LANG_CODE: Record<CountryCode, string> = {
  US: "en",
  CA: "en",
  NO: "no",
};

export function isCountryCode(value: string): value is CountryCode {
  return (COUNTRY_CODES as readonly string[]).includes(value);
}

/** Default search language for a country; "en" for countries outside the catalog. */
export function defaultLangCode(countryCode: string): string {
  return isCountryCode(countryCode) ? DEFAULT_LANG_CODE[countryCode] : "en";
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test 2>&1 | tail -8 && npm run typecheck && npm run build`
Expected: `# pass 55`, `# fail 0` (51 existing + 4 new); typecheck and build exit 0 with no output.

- [ ] **Step 5: Commit**

```bash
git add src/data/countries.ts test/unit.test.ts
git commit -m "feat: add countries module with supported codes and default search language

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 2: Structured store catalog with Norway

**Files:**
- Modify: `src/data/stores.ts` (rewrite the whole file)
- Existing fixture (already committed, do not regenerate): `test/fixtures/store-labels-v1.6.2.json` — the exact `STORE_LABELS` object from before this change.
- Test: `test/unit.test.ts` (modify the `storeIdsByCountry` block at lines ~404–439, append new tests)

**Interfaces:**
- Consumes: `CountryCode` from `src/data/countries.ts` (Task 1).
- Produces: `interface StoreRecord { storeId: string; name?: string; city: string; region?: string; countryCode: CountryCode }`, `STORES: readonly StoreRecord[]`, `storeLabelFor(store: StoreRecord): string`, `STORE_LABELS: Record<string, string>` (derived), `storeLabel(storeId: string): string | undefined`, `storeCountry(storeId: string): CountryCode | undefined`, `allStoreIds(): string[]`, `storeIdsByCountry(countryCode: CountryCode): string[]`. Task 5 uses `STORES`, `storeLabelFor`, `allStoreIds`.

- [ ] **Step 1: Write the failing tests**

Change the import line at the top of `test/unit.test.ts` from

```ts
import { storeIdsByCountry, STORE_LABELS } from "../src/data/stores.js";
```

to

```ts
import { storeIdsByCountry, STORE_LABELS, STORES, storeLabel, storeLabelFor, storeCountry, allStoreIds } from "../src/data/stores.js";
import { readFileSync } from "node:fs";
```

Replace the three tests `storeIdsByCountry — CA returns only CA stores`, `storeIdsByCountry — US returns only non-CA stores` and `storeIdsByCountry — US and CA partition all stores exactly` (keep the two `known CA store 149` / `known US store 399` tests) with:

```ts
test("storeIdsByCountry — CA returns only CA stores", () => {
  const ids = storeIdsByCountry("CA");
  assert.equal(ids.length, 15);
  ids.forEach((id) => assert.equal(storeCountry(id), "CA"));
});

test("storeIdsByCountry — US returns only US stores", () => {
  const ids = storeIdsByCountry("US");
  assert.equal(ids.length, 52);
  ids.forEach((id) => assert.equal(storeCountry(id), "US"));
});

test("storeIdsByCountry — NO returns exactly the seven Norwegian stores", () => {
  assert.deepEqual(storeIdsByCountry("NO"), ["007", "091", "095", "126", "371", "390", "441"]);
});

test("storeIdsByCountry — US, CA and NO partition all stores exactly", () => {
  const us = storeIdsByCountry("US");
  const ca = storeIdsByCountry("CA");
  const no = storeIdsByCountry("NO");
  const all = allStoreIds();
  assert.equal(us.length + ca.length + no.length, all.length);
  assert.equal(new Set([...us, ...ca, ...no]).size, all.length);
  assert.equal(all.length, 74);
  assert.equal(Object.keys(STORE_LABELS).length, 74);
});
```

Append at the end of the file:

```ts
// ── store catalog labels ──────────────────────────────────────────────────────

test("stores — US and CA labels are byte-identical to the v1.6.2 catalog", () => {
  const expected = JSON.parse(
    readFileSync(new URL("./fixtures/store-labels-v1.6.2.json", import.meta.url), "utf8")
  ) as Record<string, string>;
  const actual: Record<string, string> = {};
  for (const s of STORES) {
    if (s.countryCode !== "NO") actual[s.storeId] = storeLabelFor(s);
  }
  assert.deepEqual(actual, expected);
  assert.equal(Object.keys(expected).length, 67);
});

test("stores — Norwegian labels use name, city and NO", () => {
  assert.equal(storeLabel("095"), "095 (Furuset, Oslo, NO)");
  assert.equal(storeLabel("007"), "007 (Sørlandet, Kristiansand, NO)");
  assert.equal(storeLabel("441"), "441 (Åsane, Bergen, NO)");
  assert.equal(STORE_LABELS["390"], "390 (Ringsaker, Hamar, NO)");
});

test("stores — storeLabel and storeCountry are undefined for unknown IDs", () => {
  assert.equal(storeLabel("999"), undefined);
  assert.equal(storeCountry("999"), undefined);
  assert.equal(storeCountry("095"), "NO");
  assert.equal(storeCountry("399"), "US");
  assert.equal(storeCountry("149"), "CA");
});

test("stores — no duplicate store IDs", () => {
  const ids = allStoreIds();
  assert.equal(new Set(ids).size, ids.length);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm run typecheck 2>&1 | head -5`
Expected: errors such as `Module '"../src/data/stores.js"' has no exported member 'STORES'` (and `storeLabelFor`, `storeCountry`, `allStoreIds`).

- [ ] **Step 3: Rewrite `src/data/stores.ts`**

Replace the entire file with the following. The US and CA records are a mechanical transcription of the previous labels; keep them in exactly this order.

```ts
import type { CountryCode } from "./countries.js";

// US store IDs confirmed from ikea.com/us/en/stores/<slug>/ pages (store_id= survey param).
// CA store IDs confirmed from ikea.com/ca/en/stores/<slug>/ pages (storeNo= embedded JSON).
// NO store IDs confirmed from ikea.com/no/no/stores/<slug>/ pages ("id":"NNN" next to
//   "buClassification" in the embedded JSON), all probed against the stock API 2026-09-15.
// IDs match the sto/{storeId} path used by the IKEA stock API.
// 4-digit IDs (921, 1099, 1129) are newer-format stores — confirmed working with stock API.
// 3136 (San Francisco) returns 405 from stock API — excluded.
// 698 (CA — unknown city) returns 422 from stock API — excluded.
// 659 (Toronto Downtown) returns 200 but null quantity — included conservatively.
// NO non-stock locations (no cash-and-carry range, returned by /ru/no but excluded here):
//   722 IKEA på Karl Johan (Oslo, plan & order point), 1352 IKEA Østfold (SMALL),
//   1213 Ski, 1214 Haugesund, 1215 Ålesund, 1216 Porsgrunn (plan & order points),
//   1217 and 1218 (unknown, no ikea.com page found).
// TODO: huntsville, arcadia, mcallen-pharr, san-marcos (US) — IDs not found on store pages.
// TODO: 448, 612, 634, 652, 661, 663, 692, 693 — valid API IDs, city/country unknown.

export interface StoreRecord {
  storeId: string;
  /** Store name when it differs from the city (Norwegian stores). */
  name?: string;
  city: string;
  /** State or province code (US, CA). */
  region?: string;
  countryCode: CountryCode;
}

export const STORES: readonly StoreRecord[] = [
  // ── United States ────────────────────────────────────────────────────────────
  { storeId: "026", city: "Canton", region: "MI", countryCode: "US" },
  { storeId: "027", city: "Round Rock", region: "TX", countryCode: "US" },
  { storeId: "028", city: "Portland", region: "OR", countryCode: "US" },
  { storeId: "042", city: "Tampa", region: "FL", countryCode: "US" },
  { storeId: "064", city: "Centennial", region: "CO", countryCode: "US" },
  { storeId: "067", city: "Charlotte", region: "NC", countryCode: "US" },
  { storeId: "103", city: "Draper", region: "UT", countryCode: "US" },
  { storeId: "145", city: "Orlando", region: "FL", countryCode: "US" },
  { storeId: "152", city: "Baltimore", region: "MD", countryCode: "US" },
  { storeId: "153", city: "Pittsburgh", region: "PA", countryCode: "US" },
  { storeId: "154", city: "Elizabeth", region: "NJ", countryCode: "US" },
  { storeId: "156", city: "Long Island", region: "NY", countryCode: "US" },
  { storeId: "157", city: "West Sacramento", region: "CA", countryCode: "US" },
  { storeId: "158", city: "Stoughton", region: "MA", countryCode: "US" },
  { storeId: "162", city: "Carson", region: "CA", countryCode: "US" },
  { storeId: "165", city: "Emeryville", region: "CA", countryCode: "US" },
  { storeId: "166", city: "San Diego", region: "CA", countryCode: "US" },
  { storeId: "167", city: "Costa Mesa", region: "CA", countryCode: "US" },
  { storeId: "168", city: "Woodbridge", region: "VA", countryCode: "US" },
  { storeId: "170", city: "Bolingbrook", region: "IL", countryCode: "US" },
  { storeId: "175", city: "West Chester", region: "OH", countryCode: "US" },
  { storeId: "183", city: "Frisco", region: "TX", countryCode: "US" },
  { storeId: "207", city: "Sunrise", region: "FL", countryCode: "US" },
  { storeId: "209", city: "Tempe", region: "AZ", countryCode: "US" },
  { storeId: "210", city: "Schaumburg", region: "IL", countryCode: "US" },
  { storeId: "211", city: "Conshohocken", region: "PA", countryCode: "US" },
  { storeId: "212", city: "Minneapolis", region: "MN", countryCode: "US" },
  { storeId: "213", city: "New Haven", region: "CT", countryCode: "US" },
  { storeId: "215", city: "South Philadelphia", region: "PA", countryCode: "US" },
  { storeId: "257", city: "Atlanta", region: "GA", countryCode: "US" },
  { storeId: "327", city: "Miami", region: "FL", countryCode: "US" },
  { storeId: "347", city: "East Palo Alto", region: "CA", countryCode: "US" },
  { storeId: "374", city: "Merriam", region: "KS", countryCode: "US" },
  { storeId: "379", city: "Houston", region: "TX", countryCode: "US" },
  { storeId: "399", city: "Burbank", region: "CA", countryCode: "US" },
  { storeId: "409", city: "Paramus", region: "NJ", countryCode: "US" },
  { storeId: "410", city: "St. Louis", region: "MO", countryCode: "US" },
  { storeId: "411", city: "College Park", region: "MD", countryCode: "US" },
  { storeId: "413", city: "Covina", region: "CA", countryCode: "US" },
  { storeId: "462", city: "Las Vegas", region: "NV", countryCode: "US" },
  { storeId: "488", city: "Renton", region: "WA", countryCode: "US" },
  { storeId: "508", city: "Memphis", region: "TN", countryCode: "US" },
  { storeId: "511", city: "Columbus", region: "OH", countryCode: "US" },
  { storeId: "535", city: "Grand Prairie", region: "TX", countryCode: "US" },
  { storeId: "536", city: "Fishers", region: "IN", countryCode: "US" },
  { storeId: "537", city: "Jacksonville", region: "FL", countryCode: "US" },
  { storeId: "560", city: "Oak Creek", region: "WI", countryCode: "US" },
  { storeId: "569", city: "Norfolk", region: "VA", countryCode: "US" },
  { storeId: "570", city: "Live Oak", region: "TX", countryCode: "US" },
  { storeId: "921", city: "Brooklyn", region: "NY", countryCode: "US" },
  { storeId: "1099", city: "University Park", region: "TX", countryCode: "US" },
  { storeId: "1129", city: "Syracuse", region: "NY", countryCode: "US" },
  // ── Canada ───────────────────────────────────────────────────────────────────
  // All probed against stock API with item 20522046 — all return 200.
  { storeId: "003", city: "Richmond", region: "BC", countryCode: "CA" },
  { storeId: "004", city: "Ottawa", region: "ON", countryCode: "CA" },
  { storeId: "039", city: "Montreal", region: "QC", countryCode: "CA" },
  { storeId: "040", city: "Burlington", region: "ON", countryCode: "CA" },
  { storeId: "149", city: "North York", region: "ON", countryCode: "CA" },
  { storeId: "216", city: "Calgary", region: "AB", countryCode: "CA" },
  { storeId: "249", city: "Winnipeg", region: "MB", countryCode: "CA" },
  { storeId: "256", city: "Etobicoke", region: "ON", countryCode: "CA" },
  { storeId: "313", city: "Coquitlam", region: "BC", countryCode: "CA" },
  { storeId: "349", city: "Edmonton", region: "AB", countryCode: "CA" },
  { storeId: "372", city: "Vaughan", region: "ON", countryCode: "CA" },
  { storeId: "414", city: "Boucherville", region: "QC", countryCode: "CA" },
  { storeId: "529", city: "Halifax", region: "NS", countryCode: "CA" },
  { storeId: "559", city: "Quebec City", region: "QC", countryCode: "CA" },
  { storeId: "659", city: "Toronto Downtown", region: "ON", countryCode: "CA" },
  // ── Norway ───────────────────────────────────────────────────────────────────
  // Full-size stores only. All probed against stock API with item 00263850 — all return 200
  // with a cash-and-carry quantity. City is the metropolitan area the store is known by.
  { storeId: "007", name: "Sørlandet", city: "Kristiansand", countryCode: "NO" },
  { storeId: "091", name: "Slependen", city: "Asker", countryCode: "NO" },
  { storeId: "095", name: "Furuset", city: "Oslo", countryCode: "NO" },
  { storeId: "126", name: "Forus", city: "Stavanger", countryCode: "NO" },
  { storeId: "371", name: "Leangen", city: "Trondheim", countryCode: "NO" },
  { storeId: "390", name: "Ringsaker", city: "Hamar", countryCode: "NO" },
  { storeId: "441", name: "Åsane", city: "Bergen", countryCode: "NO" },
];

/**
 * Display label: "<id> (<name>, <city>, <region>, <country>)" with absent parts omitted.
 * US labels historically carry no country suffix; that quirk is preserved so existing
 * labels stay byte-identical ("026 (Canton, MI)", "003 (Richmond, BC, CA)", "095 (Furuset, Oslo, NO)").
 */
export function storeLabelFor(store: StoreRecord): string {
  const parts = [
    store.name,
    store.city,
    store.region,
    store.countryCode === "US" ? undefined : store.countryCode,
  ].filter((p): p is string => typeof p === "string" && p.length > 0);
  return `${store.storeId} (${parts.join(", ")})`;
}

const BY_ID: ReadonlyMap<string, StoreRecord> = new Map(STORES.map((s) => [s.storeId, s]));

/** storeId → label. Kept for backward compatibility; prefer storeLabel() / STORES. */
export const STORE_LABELS: Record<string, string> = Object.fromEntries(
  STORES.map((s) => [s.storeId, storeLabelFor(s)])
);

export function storeLabel(storeId: string): string | undefined {
  const store = BY_ID.get(storeId);
  return store ? storeLabelFor(store) : undefined;
}

export function storeCountry(storeId: string): CountryCode | undefined {
  return BY_ID.get(storeId)?.countryCode;
}

/** All catalog store IDs in declaration order (US, CA, NO). */
export function allStoreIds(): string[] {
  return STORES.map((s) => s.storeId);
}

export function storeIdsByCountry(countryCode: CountryCode): string[] {
  return STORES.filter((s) => s.countryCode === countryCode).map((s) => s.storeId);
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test 2>&1 | tail -8 && npm run typecheck && npm run build`
Expected: `# pass 60`, `# fail 0` (55 from Task 1, minus 3 replaced, plus 4 replacements, plus 4 appended); typecheck and build exit 0. If the fixture test fails, the diff it prints names the label that changed: fix the record, never the fixture.

Note: `npm run typecheck` will still pass at this point because the tools call `storeIdsByCountry` with a value typed `"US" | "CA"`, which is assignable to `CountryCode`. Task 5 widens the tools.

- [ ] **Step 5: Commit**

```bash
git add src/data/stores.ts test/unit.test.ts
git commit -m "refactor: structured store catalog with Norway stores

Replace the label map and CA regex with StoreRecord entries; derive labels,
STORE_LABELS and country lookups from them. Existing US/CA labels are
byte-identical (fixture test). Adds the seven Norwegian full-size stores.

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 3: Schemas — widen the country enum, default langCode from country

**Files:**
- Modify: `src/schemas/index.ts` (lines 1, 34–53, 86–90)
- Test: `test/unit.test.ts` (append)

**Interfaces:**
- Consumes: `COUNTRY_CODES`, `defaultLangCode` from `src/data/countries.ts` (Task 1).
- Produces: `CountryCodeEnum` accepts `"NO"`; `SearchProductsInput.parse(...)` and `GetProductDetailsInput.parse(...)` always return `langCode: string`, filled from the country when omitted. The inferred `*InputType` types are unchanged in shape.

- [ ] **Step 1: Write the failing tests**

Find the existing import block for schemas in `test/unit.test.ts` (it starts `import { CompareStoreStockInput, ...` around line 364, mid-file) and make sure `SearchProductsInput`, `GetProductDetailsInput` and `ListStoresInput` are in it. If the line is

```ts
import { CompareStoreStockInput, CheckCartAvailabilityInput, FindBestStoreForCartInput, FindBestStoreInput } from "../src/schemas/index.js";
```

change it to

```ts
import { CompareStoreStockInput, CheckCartAvailabilityInput, FindBestStoreForCartInput, FindBestStoreInput, SearchProductsInput, GetProductDetailsInput, ListStoresInput } from "../src/schemas/index.js";
```

(If the existing import differs, add the three names to it rather than adding a second import of the same module.)

Append at the end of the file:

```ts
// ── country enum and langCode defaults ────────────────────────────────────────

test("ListStoresInput — accepts lowercase no and normalises to NO", () => {
  assert.equal(ListStoresInput.parse({ countryCode: "no" }).countryCode, "NO");
  assert.equal(ListStoresInput.parse({ countryCode: "CA" }).countryCode, "CA");
});

test("ListStoresInput — rejects countries outside the catalog", () => {
  assert.throws(() => ListStoresInput.parse({ countryCode: "SE" }));
});

test("CompareStoreStockInput — countryCode NO is accepted", () => {
  const input = CompareStoreStockInput.parse({ itemNo: "00263850", countryCode: "no" });
  assert.equal(input.countryCode, "NO");
  assert.equal(input.storeIds, undefined);
});

test("SearchProductsInput — langCode defaults to the country's language", () => {
  assert.equal(SearchProductsInput.parse({ query: "billy", countryCode: "NO" }).langCode, "no");
  assert.equal(SearchProductsInput.parse({ query: "billy", countryCode: "no" }).langCode, "no");
  assert.equal(SearchProductsInput.parse({ query: "billy", countryCode: "CA" }).langCode, "en");
  assert.equal(SearchProductsInput.parse({ query: "billy" }).langCode, "en");
  assert.equal(SearchProductsInput.parse({ query: "billy", countryCode: "SE" }).langCode, "en");
});

test("SearchProductsInput — explicit langCode wins over the country default", () => {
  const input = SearchProductsInput.parse({ query: "billy", countryCode: "NO", langCode: "en" });
  assert.equal(input.langCode, "en");
  assert.equal(input.countryCode, "NO");
  assert.equal(input.size, 10);
});

test("GetProductDetailsInput — langCode defaults to the country's language", () => {
  assert.equal(GetProductDetailsInput.parse({ itemNo: "00263850", countryCode: "NO" }).langCode, "no");
  assert.equal(GetProductDetailsInput.parse({ itemNo: "00263850" }).langCode, "en");
  assert.equal(GetProductDetailsInput.parse({ itemNo: "00263850", countryCode: "NO", langCode: "en" }).langCode, "en");
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test 2>&1 | grep -E "^not ok|# (pass|fail)"`
Expected: `not ok` for the four tests that use `NO` or expect `"no"` (`ListStoresInput — accepts lowercase no`, `CompareStoreStockInput — countryCode NO`, both `langCode defaults` tests); `# fail 4`.

- [ ] **Step 3: Implement**

In `src/schemas/index.ts`:

Change line 1 from

```ts
import { z } from "zod";
```

to

```ts
import { z } from "zod";
import { COUNTRY_CODES, defaultLangCode } from "../data/countries.js";
```

Replace the `CountryCodeEnum` definition (and its comment) with:

```ts
// Uppercase-normalised enum restricted to the countries in the store catalog.
// Accepts "us"/"ca"/"no" and normalises to "US"/"CA"/"NO".
const CountryCodeEnum = z.preprocess(toUpper, z.enum(COUNTRY_CODES));

// Two-letter language code for the search API. Optional on input; filled from the
// country's default (see defaultLangCode) when omitted.
const LangCode = z.string().length(2);
```

Replace `SearchProductsInput` with:

```ts
export const SearchProductsInput = z
  .object({
    query: z.string().min(1),
    countryCode: CountryCode2.default("US"),
    langCode: LangCode.optional(),
    size: z.number().int().min(1).max(50).default(10),
  })
  .transform((d) => ({ ...d, langCode: d.langCode ?? defaultLangCode(d.countryCode) }));
```

Replace `GetProductDetailsInput` with:

```ts
export const GetProductDetailsInput = z
  .object({
    itemNo: ItemNo,
    countryCode: CountryCode2.default("US"),
    langCode: LangCode.optional(),
  })
  .transform((d) => ({ ...d, langCode: d.langCode ?? defaultLangCode(d.countryCode) }));
```

Leave every other schema unchanged. `CountryCode2` (free-form two letters) stays as it is.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test 2>&1 | tail -8 && npm run typecheck && npm run build`
Expected: `# pass 66`, `# fail 0`; typecheck and build exit 0. If typecheck complains that `d.countryCode` may be `unknown`, the `CountryCode2` preprocess type has changed: keep `CountryCode2 = z.preprocess(toUpper, z.string().length(2))` and use `String(d.countryCode)` in the transform.

- [ ] **Step 5: Commit**

```bash
git add src/schemas/index.ts test/unit.test.ts
git commit -m "feat: accept NO in country enum and default langCode from country

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 4: `inRange` in stock projections

**Files:**
- Modify: `src/services/ikea.ts` (type `StockCashCarry` ~line 76; `projectStock` ~line 128; `projectMultiItemStock` ~line 139)
- Modify: `src/tools/check_multi_item_stock.ts:20-28`
- Test: `test/unit.test.ts` (update the test at line ~37, append)

**Interfaces:**
- Consumes: nothing new.
- Produces: `projectStock(...)` and each row of `projectMultiItemStock(...)` include `inRange: boolean | null`. `check_multi_item_stock` rows include `inRange`.

- [ ] **Step 1: Update the shape test and add failing tests**

In `test/unit.test.ts`, the test `projectStock — stocked item returns correct shape` (line ~37) asserts the whole object. Change its expected object to include `inRange: null` right after `availableForCashCarry` (the fixture has no `range`, so `null` is correct):

```ts
  assert.deepEqual(result, {
    availableForCashCarry: true,
    inRange: null,
    quantity: 42,
    messageType: "HIGH_IN_STOCK",
    eligibleForStockNotification: false,
    errors: [],
  });
```

Append at the end of the file:

```ts
// ── inRange ───────────────────────────────────────────────────────────────────

function withRange(inRange: boolean, quantity?: number): StockResponse {
  const base = makeStockResponse();
  const a = base.availabilities![0];
  const cashCarry: typeof a.buyingOption.cashCarry = {
    ...a.buyingOption.cashCarry,
    range: { inRange },
  };
  if (quantity === undefined) {
    // Not in range: the API omits availability entirely.
    delete (cashCarry as { availability?: unknown }).availability;
  } else {
    cashCarry.availability = { ...a.buyingOption.cashCarry.availability, quantity };
  }
  return {
    ...base,
    availabilities: [{ ...a, availableForCashCarry: inRange, buyingOption: { ...a.buyingOption, cashCarry } }],
  };
}

test("projectStock — inRange false with no quantity means item not sold here", () => {
  const result = projectStock(withRange(false));
  assert.equal(result.inRange, false);
  assert.equal(result.quantity, null);
  assert.equal(result.availableForCashCarry, false);
  assert.deepEqual(result.errors, []);
});

test("projectStock — inRange true passes through with quantity", () => {
  const result = projectStock(withRange(true, 7));
  assert.equal(result.inRange, true);
  assert.equal(result.quantity, 7);
});

test("projectStock — inRange is null when the API omits range", () => {
  assert.equal(projectStock(makeStockResponse()).inRange, null);
  assert.equal(projectStock(makeStockResponse({ availabilities: [] })).inRange, null);
});

test("projectMultiItemStock — inRange per row, null on 405 and not-found", () => {
  const stocked = projectMultiItemStock(
    { availabilities: [makeAvailability("20522046", 3)], timestamp: "t" },
    ["20522046", "99999999"]
  );
  assert.equal(stocked[0].inRange, null); // makeAvailability has no range
  assert.equal(stocked[1].inRange, null);
  assert.equal(stocked[1].errors[0].code, 404);

  const storeError = projectMultiItemStock(
    { availabilities: null, errors: [{ code: 405, message: "Store not found" }], timestamp: "t" },
    ["20522046"]
  );
  assert.equal(storeError[0].inRange, null);

  const inRange = projectMultiItemStock(withRange(true, 7), ["20522046"]);
  assert.equal(inRange[0].inRange, true);
  const notInRange = projectMultiItemStock(withRange(false), ["20522046"]);
  assert.equal(notInRange[0].inRange, false);
  assert.equal(notInRange[0].quantity, null);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm run typecheck 2>&1 | head -5`
Expected: type errors that `range` does not exist on the cash-carry type and that `inRange` does not exist on the projection results.

- [ ] **Step 3: Implement**

In `src/services/ikea.ts`:

In `interface StockCashCarry`, add after `updateDateTime: string;`:

```ts
  /** inRange false = item not sold at this store / in this market; availability is then omitted. */
  range?: { inRange: boolean };
```

In `projectStock`, add `inRange` directly after `availableForCashCarry`:

```ts
export function projectStock(result: StockResponse) {
  const a = result.availabilities?.[0];
  return {
    availableForCashCarry: a?.availableForCashCarry ?? false,
    inRange: a?.buyingOption?.cashCarry?.range?.inRange ?? null,
    quantity: a?.buyingOption?.cashCarry?.availability?.quantity ?? null,
    messageType: a?.buyingOption?.cashCarry?.availability?.probability?.thisDay?.messageType ?? null,
    eligibleForStockNotification: a?.buyingOption?.cashCarry?.eligibleForStockNotification ?? null,
    errors: annotateStockErrors(result.errors) ?? [],
  };
}
```

In `projectMultiItemStock`, add `inRange` to all three returned row shapes, directly after `availableForCashCarry`:

- storeError branch: `inRange: null as boolean | null,`
- found branch (`if (a)`): `inRange: a.buyingOption?.cashCarry?.range?.inRange ?? null,`
- not-found branch: `inRange: null as boolean | null,`

In `src/tools/check_multi_item_stock.ts`, add `inRange: row.inRange,` directly after `availableForCashCarry: row.availableForCashCarry,` in the row mapping.

Update the description in the same file to:

```ts
  description:
    "Check cash-and-carry stock for multiple items at a single IKEA store. inRange: false on a row means the item is not sold at that store or in that country's market.",
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test 2>&1 | tail -8 && npm run typecheck && npm run build`
Expected: `# pass 70`, `# fail 0`; typecheck and build exit 0.

- [ ] **Step 5: Commit**

```bash
git add src/services/ikea.ts src/tools/check_multi_item_stock.ts test/unit.test.ts
git commit -m "feat: expose inRange in stock projections

Distinguishes 'not sold at this store / market' (inRange false, no
quantity, no error) from 'unknown'. Additive field on check_store_stock,
compare_store_stock and check_multi_item_stock rows.

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 5: Tools — enums from COUNTRY_CODES, descriptions, list_stores from records

**Files:**
- Modify: `src/tools/list_stores.ts` (rewrite)
- Modify: `src/tools/compare_store_stock.ts:1-24`
- Modify: `src/tools/find_best_store_for_item.ts:1-24`
- Modify: `src/tools/find_best_store_for_cart.ts:1-35`
- Modify: `src/tools/search_products.ts:6-15`
- Modify: `src/tools/get_product_details.ts:6-14`
- Modify: `src/tools/check_store_stock.ts:7`
- Test: `test/unit.test.ts` (append)

**Interfaces:**
- Consumes: `COUNTRY_CODES`, `CountryCode` (Task 1); `STORES`, `storeLabelFor`, `allStoreIds`, `storeIdsByCountry` (Task 2).
- Produces: `listStoresTool.handler(rawInput)` returns `{ content: [{ type: "text", text }] }` where `text` is JSON `{ stores: { storeId, storeLabel, countryCode }[] }` (unchanged shape, now includes NO).

- [ ] **Step 1: Write the failing tests**

Add to the top of `test/unit.test.ts`, after the `countries.js` import from Task 1:

```ts
import { listStoresTool } from "../src/tools/list_stores.js";
import { compareStoreStockTool } from "../src/tools/compare_store_stock.js";
import { findBestStoreForItemTool } from "../src/tools/find_best_store_for_item.js";
import { findBestStoreForCartTool } from "../src/tools/find_best_store_for_cart.js";
import { searchProductsTool } from "../src/tools/search_products.js";
import { getProductDetailsTool } from "../src/tools/get_product_details.js";
```

Append at the end of the file:

```ts
// ── list_stores handler and tool schemas ──────────────────────────────────────

async function listStores(args: Record<string, unknown>) {
  const res = await listStoresTool.handler(args);
  return JSON.parse(res.content[0].text) as { stores: { storeId: string; storeLabel: string; countryCode: string }[] };
}

test("list_stores — countryCode NO returns the seven Norwegian stores", async () => {
  const { stores } = await listStores({ countryCode: "NO" });
  assert.deepEqual(stores.map((s) => s.storeId), ["007", "091", "095", "126", "371", "390", "441"]);
  assert.ok(stores.every((s) => s.countryCode === "NO"));
  assert.equal(stores[2].storeLabel, "095 (Furuset, Oslo, NO)");
});

test("list_stores — no filter returns all 74 stores with correct country tags", async () => {
  const { stores } = await listStores({});
  assert.equal(stores.length, 74);
  assert.equal(stores.filter((s) => s.countryCode === "US").length, 52);
  assert.equal(stores.filter((s) => s.countryCode === "CA").length, 15);
  assert.equal(stores.filter((s) => s.countryCode === "NO").length, 7);
  assert.equal(stores.find((s) => s.storeId === "149")?.countryCode, "CA");
  assert.equal(stores.find((s) => s.storeId === "399")?.countryCode, "US");
});

test("list_stores — countryCode CA still returns 15 stores", async () => {
  const { stores } = await listStores({ countryCode: "ca" });
  assert.equal(stores.length, 15);
  assert.ok(stores.every((s) => s.countryCode === "CA"));
});

test("tool schemas — every countryCode enum lists US, CA and NO", () => {
  for (const tool of [listStoresTool, compareStoreStockTool, findBestStoreForItemTool, findBestStoreForCartTool]) {
    const prop = (tool.inputSchema.properties as unknown as Record<string, { enum?: string[] }>).countryCode;
    assert.deepEqual(prop.enum, ["US", "CA", "NO"], `${tool.name} countryCode enum`);
    assert.match(tool.description, /'US', 'CA' or 'NO'/, `${tool.name} description`);
  }
});

test("tool schemas — search tools document the langCode default and no longer hardcode en", () => {
  for (const tool of [searchProductsTool, getProductDetailsTool]) {
    const props = tool.inputSchema.properties as unknown as Record<string, { default?: unknown; description?: string }>;
    assert.equal(props.langCode.default, undefined, `${tool.name} langCode default`);
    assert.match(props.langCode.description ?? "", /no for NO/, `${tool.name} langCode description`);
    assert.equal(props.countryCode.default, "US", `${tool.name} countryCode default`);
  }
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test 2>&1 | grep -E "^not ok|# (pass|fail)"`
Expected: `not ok` for `list_stores — countryCode NO` (currently returns zero rows because the handler tags via the old CA set), `list_stores — no filter returns all 74` (tags NO stores as US), both `tool schemas` tests; `# fail 4`. If `npm run typecheck` fails first because `listStoresTool.handler` returns a union, that is also an acceptable red state.

- [ ] **Step 3: Implement**

Replace `src/tools/list_stores.ts` with:

```ts
import { ListStoresInput } from "../schemas/index.js";
import { STORES, storeLabelFor } from "../data/stores.js";
import { COUNTRY_CODES } from "../data/countries.js";

export const listStoresTool = {
  name: "list_stores",
  description:
    "List known IKEA store IDs and display names. Use countryCode ('US', 'CA' or 'NO') to filter by country, or omit to return all stores. Use the returned storeId values with the stock tools.",
  inputSchema: {
    type: "object",
    properties: {
      countryCode: { type: "string", enum: [...COUNTRY_CODES] },
    },
  },
  async handler(rawInput: unknown) {
    const input = ListStoresInput.parse(rawInput);
    const records = input.countryCode
      ? STORES.filter((s) => s.countryCode === input.countryCode)
      : STORES;
    const stores = records.map((s) => ({
      storeId: s.storeId,
      storeLabel: storeLabelFor(s),
      countryCode: s.countryCode,
    }));
    return { content: [{ type: "text", text: JSON.stringify({ stores }, null, 2) }] };
  },
};
```

In `src/tools/compare_store_stock.ts`:

- Add after the existing imports: `import { COUNTRY_CODES, type CountryCode } from "../data/countries.js";`
- Description → `"Compare cash-and-carry stock for an item across multiple IKEA stores. Provide explicit storeIds, or use countryCode ('US', 'CA' or 'NO') to compare all catalog stores for that country. inRange: false on a row means the item is not sold at that store or in that country's market."`
- In `inputSchema.properties`: `countryCode: { type: "string", enum: [...COUNTRY_CODES] },`
- Line ~23: `?? storeIdsByCountry(input.countryCode as "US" | "CA");` → `?? storeIdsByCountry(input.countryCode as CountryCode);`

In `src/tools/find_best_store_for_item.ts`:

- Change `import { STORE_LABELS, storeLabel, storeIdsByCountry } from "../data/stores.js";` to `import { allStoreIds, storeLabel, storeIdsByCountry } from "../data/stores.js";`
- Add: `import { COUNTRY_CODES } from "../data/countries.js";`
- Description: replace `('US' or 'CA')` with `('US', 'CA' or 'NO')`.
- `countryCode: { type: "string", enum: [...COUNTRY_CODES] },`
- `: Object.keys(STORE_LABELS));` → `: allStoreIds());`

In `src/tools/find_best_store_for_cart.ts`: the same four edits (import `allStoreIds` instead of `STORE_LABELS`, import `COUNTRY_CODES`, description wording, enum, `Object.keys(STORE_LABELS)` → `allStoreIds()`).

In `src/tools/search_products.ts`, replace the two schema properties:

```ts
      countryCode: { type: "string", default: "US", description: "Two-letter country code of the IKEA market to search, e.g. US, CA, NO." },
      langCode: { type: "string", description: "Two-letter language code. Defaults to the country's language: en for US and CA, no for NO." },
```

In `src/tools/get_product_details.ts`, replace the same two properties with the same two lines.

In `src/tools/check_store_stock.ts`, description → `"Check cash-and-carry stock availability for an item at a specific IKEA store. inRange: false means the item is not sold at that store or in that country's market."`

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test 2>&1 | tail -8 && npm run typecheck && npm run build`
Expected: `# pass 75`, `# fail 0`; typecheck and build exit 0. Also run `grep -rn "STORE_LABELS\|'US' or 'CA'" src/tools/` and expect no output.

- [ ] **Step 5: Commit**

```bash
git add src/tools/list_stores.ts src/tools/compare_store_stock.ts src/tools/find_best_store_for_item.ts src/tools/find_best_store_for_cart.ts src/tools/search_products.ts src/tools/get_product_details.ts src/tools/check_store_stock.ts test/unit.test.ts
git commit -m "feat: tools accept countryCode NO; list_stores built from store records

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 6: Smoke steps and README

**Files:**
- Modify: `smoke.mjs` (insert before the `} catch (e) {` line near the end)
- Modify: `README.md` (sections listed below)

**Interfaces:**
- Consumes: the behaviour delivered by Tasks 1–5.
- Produces: nothing for other tasks.

- [ ] **Step 1: Add Norway steps to `smoke.mjs`**

Insert the following immediately before the line `} catch (e) {`:

```js
  // ── Norway ──────────────────────────────────────────────────────────────────
  // search_products — countryCode NO, langCode defaults to "no", prices in NOK
  send({ jsonrpc: "2.0", id: 16, method: "tools/call", params: { name: "search_products", arguments: { query: "billy", countryCode: "NO", size: 1 } } });
  const spNO = await readNext(15000);
  if (spNO.error) {
    console.log("search_products[NO] FAIL:", JSON.stringify(spNO.error));
  } else {
    const d = JSON.parse(spNO.result.content[0].text);
    const item = d.items?.[0];
    const ok = item?.salesPrice?.currencyCode === "NOK" && String(item?.pipUrl).includes("/no/no/");
    console.log(`search_products[NO] ${ok ? "OK" : "LOCALE_MISMATCH"} — name: ${item?.name} ${item?.typeName} price: ${item?.salesPrice?.amount} ${item?.salesPrice?.currencyCode}`);
  }

  // check_store_stock — Norwegian store 095 Furuset, Norwegian BILLY item
  send({ jsonrpc: "2.0", id: 17, method: "tools/call", params: { name: "check_store_stock", arguments: { itemNo: "00263850", storeId: "095" } } });
  const csNO = await readNext(10000);
  if (csNO.error) {
    console.log("check_store_stock[095/NO] FAIL:", JSON.stringify(csNO.error));
  } else {
    const d = JSON.parse(csNO.result.content[0].text);
    const ok = String(d.storeLabel).endsWith("NO)") && d.inRange === true;
    console.log(`check_store_stock[095/NO] ${ok ? "OK" : "MISMATCH"} — label: ${d.storeLabel} qty: ${d.quantity} inRange: ${d.inRange} status: ${d.messageType}`);
  }

  // check_store_stock — US-only item at a Norwegian store → inRange false, no error
  send({ jsonrpc: "2.0", id: 18, method: "tools/call", params: { name: "check_store_stock", arguments: { itemNo: "20522046", storeId: "095" } } });
  const csNotInRange = await readNext(10000);
  if (csNotInRange.error) {
    console.log("check_store_stock[095/not-in-range] FAIL:", JSON.stringify(csNotInRange.error));
  } else {
    const d = JSON.parse(csNotInRange.result.content[0].text);
    const ok = d.inRange === false && d.quantity === null && Array.isArray(d.errors) && d.errors.length === 0;
    console.log(`check_store_stock[095/not-in-range] ${ok ? "OK" : "MISMATCH"} — inRange: ${d.inRange} qty: ${d.quantity} errors: ${d.errors?.length}`);
  }

  // compare_store_stock — countryCode NO (no storeIds) → seven Norwegian rows
  send({ jsonrpc: "2.0", id: 19, method: "tools/call", params: { name: "compare_store_stock", arguments: { itemNo: "00263850", countryCode: "NO" } } });
  const cmpNO = await readNext(30000);
  if (cmpNO.error) {
    console.log("compare_store_stock[countryCode=NO] FAIL:", JSON.stringify(cmpNO.error));
  } else {
    const d = JSON.parse(cmpNO.result.content[0].text);
    const ok = d.length === 7 && d.every((r) => String(r.storeLabel).endsWith("NO)"));
    console.log(`compare_store_stock[countryCode=NO] ${ok ? "OK" : "MISMATCH"} — ${d.length} store(s): ${d.map((r) => `${r.storeId}(${r.quantity})`).join(", ")}`);
  }

  // list_stores — countryCode NO
  send({ jsonrpc: "2.0", id: 20, method: "tools/call", params: { name: "list_stores", arguments: { countryCode: "NO" } } });
  const lsNO = await readNext(10000);
  if (lsNO.error) {
    console.log("list_stores[NO] FAIL:", JSON.stringify(lsNO.error));
  } else {
    const d = JSON.parse(lsNO.result.content[0].text);
    console.log(`list_stores[NO] ${d.stores?.length === 7 ? "OK" : "MISMATCH"} — ${d.stores?.length} store(s)`);
  }
```

- [ ] **Step 2: Build and run the smoke test once**

Run: `npm run build && npm run smoke 2>&1 | tail -8`
Expected: the last five lines start with `search_products[NO] OK`, `check_store_stock[095/NO] OK`, `check_store_stock[095/not-in-range] OK`, `compare_store_stock[countryCode=NO] OK`, `list_stores[NO] OK`. Live stock numbers vary; only the OK/MISMATCH verdicts matter. If IKEA is unreachable, note it in the commit message and continue; do not retry in a loop.

- [ ] **Step 3: Edit `README.md`**

Make exactly these edits (search for the quoted text; each occurs once):

1. In **MVP limitations**, after the line `- Canada store coverage is complete (15 stores)` insert:
   `- Norway coverage is the 7 full-size stores; planning studios and the Oslo city-centre store are excluded (no cash-and-carry stock)`

2. In the **`search_products`** input table, replace the row
   `| \`langCode\` | string | \`"en"\` | no |`
   with
   `| \`langCode\` | string | country default: \`"en"\` for US/CA, \`"no"\` for NO | no |`
   Do the same replacement in the **`get_product_details`** input table.

3. In the **`check_store_stock`** output example, after the line `  "availableForCashCarry": true,` insert `  "inRange": true,`; in the error example after `  "availableForCashCarry": false,` insert `  "inRange": null,`. After the error example's closing ```` ``` ```` add the paragraph:
   `\`inRange: false\` with an empty \`errors\` array means the item is not sold at that store or in that country's market (for example a US item number checked against a Norwegian store). Search with the matching \`countryCode\` to get a local item number.`

4. In **Example workflows**, after workflow 4's closing paragraph (`Use \`check_cart_availability\` to then verify exact quantities at the chosen store.`) and before the `---`, add:

   ````markdown
   ### 5. Norway: search in Norwegian, then compare every Norwegian store

   ```
   1. search_products       { "query": "billy bokhylle", "countryCode": "NO" }
      → Norwegian names and NOK prices; langCode defaults to "no"

   2. compare_store_stock   { "itemNo": "00263850", "countryCode": "NO" }
      → one row per Norwegian store, labelled like "095 (Furuset, Oslo, NO)"
   ```
   ````

5. In **Build and test**, replace `\`smoke.mjs\` exercises all 4 tools against the live IKEA API and prints pass/fail lines to stdout.` with `\`smoke.mjs\` (\`npm run smoke\`) exercises the tools against the live IKEA API, including five Norway steps, and prints pass/fail lines to stdout.`

6. In **Store IDs**, after the sentence ending `(all probed against the stock API).` add: ` 7 Norway stores confirmed from \`ikea.com/no/no/stores/\` pages (store code in the page's embedded business-unit JSON), all probed against the stock API.` In the format list add a line: `- Norway: \`"095"\` (Furuset, Oslo, NO), \`"007"\` (Sørlandet, Kristiansand, NO)`

7. In **Limitations**, replace `- US and Canada only — no other countries supported.` with `- US, Canada and Norway only — no other countries supported.` and in the fan-out line replace `\`"CA"\` ≈ 15)` with `\`"CA"\` ≈ 15, \`"NO"\` = 7)`.

8. In **Item numbers**, after the sentence `Values outside 6–9 digits after stripping are rejected.` add a paragraph: `Item numbers are per market. An item found with \`countryCode: "US"\` may not exist in the Norwegian range; stock tools then return \`inRange: false\`. Search with the country you intend to shop in.`

9. In **Supported countries**, add the row `| Norway | \`NO\` | 7 |` after the Canada row.

10. In **Troubleshooting**, add the row `| \`inRange: false\`, empty \`errors\` | Item not sold at that store or in that market — search with the matching \`countryCode\` |` after the `404` row.

- [ ] **Step 4: Verify**

Run: `grep -c "NO" README.md && grep -n "inRange" README.md | head && npm test 2>&1 | tail -3`
Expected: several matches for `NO`, at least four lines mentioning `inRange`, tests still `# fail 0`.

- [ ] **Step 5: Commit**

```bash
git add smoke.mjs README.md
git commit -m "docs: document Norway support, langCode defaults and inRange; add Norway smoke steps

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7: Final verification

**Files:** none modified.

- [ ] **Step 1: Full green run**

Run: `npm test 2>&1 | tail -4 && npm run typecheck && npm run build && git status --short`
Expected: `# pass 75`, `# fail 0`; typecheck and build silent; `git status --short` prints nothing (clean tree).

- [ ] **Step 2: Diff review against the spec**

Run: `git log --oneline master..HEAD && git diff master --stat`
Expected: six feature commits after the spec/plan commit; changed files limited to `src/data/countries.ts`, `src/data/stores.ts`, `src/schemas/index.ts`, `src/services/ikea.ts`, the seven tool files named in Tasks 4–5, `test/unit.test.ts`, `test/fixtures/store-labels-v1.6.2.json`, `smoke.mjs`, `README.md`, and the two docs files. `check_cart_availability.ts`, `http.ts`, `server.ts`, `utils/http.ts` and `package.json` must not appear.

- [ ] **Step 3: Spot-check the built server over stdio**

Run:

```bash
printf '%s\n%s\n%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_stores","arguments":{"countryCode":"NO"}}}' | node dist/index.js 2>/dev/null | tail -1 | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const j=JSON.parse(s);console.log(JSON.parse(j.result.content[0].text).stores.map(x=>x.storeLabel).join(" | "))})'
```

Expected output:

```
007 (Sørlandet, Kristiansand, NO) | 091 (Slependen, Asker, NO) | 095 (Furuset, Oslo, NO) | 126 (Forus, Stavanger, NO) | 371 (Leangen, Trondheim, NO) | 390 (Ringsaker, Hamar, NO) | 441 (Åsane, Bergen, NO)
```

- [ ] **Step 4: Report**

Report the commit list, the test count, and the smoke verdict lines from Task 6 to the user. Do not push and do not open a PR.
