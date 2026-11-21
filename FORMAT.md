# Food Pilot country data format (v1)

Goal: adding a country = adding one folder of JSON + one line in index.json. No code change.

Layout (bundled in app assets as `countries/`, same tree hosted free later on GitHub Pages):

countries/
  index.json                 list of countries
  US/country.json            config
  US/prices.json             US fallback price list (Kroger live stays primary)
  GB/country.json, GB/prices.json, GB/dishes.json
  CA/..., AU/..., DE/..., ZA/...

## index.json
{ "schema": 1, "updated": "2026-11-21",
  "countries": [ { "code": "GB", "name": "United Kingdom", "dataVersion": "2026-11" }, ... ] }

## country.json
{ "schema": 1,
  "code": "GB", "name": "United Kingdom",
  "currency": "GBP", "currencySymbol": "£", "locale": "en-GB",
  "units": "metric",                       // "us" for US
  "livePriceProvider": null,               // "kroger" for US only
  "priceLabel": "Estimated price",
  "defaultRegion": "rest",
  "regions": [
    { "id": "london", "name": "London", "multiplier": 1.08,
      "match": { "admin1": ["England"], "localities": ["London"], "bbox": [51.28,-0.51,51.69,0.33] } },
    { "id": "rest", "name": "Rest of UK", "multiplier": 1.00, "match": { "default": true } }
  ] }
Region from phone location: Android Geocoder adminArea/locality matched against `match`, bbox [south,west,north,east] as fallback; else `defaultRegion`. User can override in Settings.

## prices.json
{ "schema": 1, "country": "GB", "currency": "GBP",
  "dataVersion": "2026-11", "updated": "2026-11-21",
  "sources": ["Hand-estimated from public supermarket shelf prices, Nov 2026"],
  "items": [
    { "id": "milk-semi-skimmed",
      "name": "Semi-skimmed milk",
      "aliases": ["milk", "semi skimmed milk"],   // lower-case match terms for ingredient names
      "category": "Dairy",                         // Produce, Meat & Protein, Dairy, Bakery, Grains & Pasta, Canned Goods, Spices & Oils, Frozen, Drinks, Other
      "pack": { "qty": 2.272, "unit": "l" },      // units: g, kg, ml, l, count (US file may use oz, lb, fl oz, gal)
      "price": 1.75,                               // base price (multiplier 1.00 region)
      "regionPrices": { "london": 1.85 },          // optional exact overrides; otherwise price * region multiplier
      "local": false }                             // true = country-specific item (crumpets, Vegemite...)
  ] }

## dishes.json
Same recipe fields as the built-in US recipe catalog in the app, plus "country": "GB".

{ "schema": 1, "country": "GB", "dataVersion": "2026-11",
  "dishes": [
    { "id": "gb-shepherds-pie",                    // unique; the app adds "gb-" if it's missing
      "title": "Shepherd's Pie",
      "description": "Lamb mince and vegetables in gravy under a mashed potato top.",
      "prepTimeMinutes": 20, "cookTimeMinutes": 45,
      "baseServings": 4,
      "tags": ["Family Classic", "Oven Bake"],
      "dietaryTags": ["Gluten-Free"],              // only: Vegetarian, Vegan, Gluten-Free, Dairy-Free, Low Carb
      "allergens": ["Dairy"],                      // only: Peanuts, Shellfish, Dairy, Gluten, Tree Nuts, Eggs, Soy, Fish, Sesame
      "ingredients": [
        { "normalizedName": "lamb mince",          // lower-case; should equal a prices.json name or alias so it gets a price
          "displayName": "Lamb mince",             // what the cook reads
          "quantityPerServing": 125,               // ONE serving
          "unit": "g",                             // g, kg, ml, l, tsp, tbsp, count, clove, can (US units also work: cup, oz, lb)
          "category": "PROTEIN",                   // PRODUCE, PROTEIN, DAIRY_AND_ALTERNATIVES, GRAINS_AND_PASTA, CANNED_GOODS, SPICES_AND_OILS, OTHER
          "optional": false }                      // optional, default false
      ],
      "instructions": ["Heat the oven to 200°C (180°C fan).", "..."],   // °C; give a safe inside temperature for meat/fish (e.g. 75°C)
      "emoji": "🥧",                                // optional, default 🍲
      "supportedSlots": ["DINNER"],                // BREAKFAST, LUNCH, DINNER (several allowed); default DINNER
      "cuisine": "British",                        // optional
      "defaultFitReason": "…" }                    // optional one-line "why this fits" text
  ] }
No photos: dishes without a built-in photo show the app's placeholder. German dishes may use German title/description/steps.

## How the app uses the files (notes from the app side, Nov 2026)
- Files live in app/src/main/assets/countries/. A hosted copy (gradle property COUNTRY_DATA_BASE_URL in gradle.properties,
  empty for now, https only, e.g. https://<owner>.github.io/foodpilot-data/countries/) is checked at most once a day when the
  app opens, and again when the country changes. Only countries whose hosted index.json dataVersion is newer (text compare,
  so "2026-12" > "2026-11") are downloaded. A country is saved only when all its files download and pass the checks
  (schema 1, codes match, currency matches country.json, unique non-empty item ids, price > 0, pack qty > 0, known pack
  units, every dish has ingredients and steps). Offline or on any error the phone keeps its saved copy, else the bundled files.
- Monthly update, no new app: edit the files, raise "dataVersion" for each changed country in index.json (e.g. "2026-12"),
  upload the same tree. A country whose files are rejected keeps last month's data and is tried again the next day.
  A missing dishes.json online (HTTP 404) means the country has no dishes; a failed download is never read as "missing".
- prices.json "currency" must equal country.json "currency", or the price list is ignored. In bundled files, items with
  price <= 0 or pack.qty <= 0 are dropped; in a download, one such item rejects that country's update.
- Region match: names are compared without accents/case (München = Munchen = munchen). A region with both admin1 and
  localities needs both; with only localities, the city may come back as locality, subAdminArea or (city-states) adminArea.
  "codes" (e.g. "QC", "NSW") count like admin1. Most specific match wins; then bbox; then the region with
  "default": true; with no location at all, defaultRegion. List English and local spellings (e.g. "Munich", "München").
- Ingredient matching: exact name/alias/id first (plurals ignored: "potato" = "potatoes"), else the longest alias found as
  whole words inside the ingredient name ("boneless chicken thighs" -> "chicken thighs"). Put common recipe words in aliases.
- Unit conversion reuses the app's existing tables: weight<->weight and volume<->volume always; cup/can/count to weight only
  for foods the app knows typical weights for. Use g/ml packs where you can.
- Priority per ingredient: live store quote (US Kroger) > the user's own receipts/entries > prices.json.
- A country with fewer than 12 own dishes also shows the other countries' dishes after its own in plans.
- The AI recipe request sends country + currency + priced food names (only names matching ^[a-z0-9][a-z0-9 '-]{1,59}$,
  so umlauts are skipped; give German items an English alias). AI outside the US is off until the Worker is deployed
  (build flag AI_RECIPES_INTERNATIONAL).
