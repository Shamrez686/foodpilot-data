# foodpilot-data

Public price lists and local dishes for the Food Pilot app. No app code and no keys belong here.

The app reads https://shamrez686.github.io/foodpilot-data/countries/ at most once a day.
It downloads a country only when that country's `dataVersion` in `countries/index.json` is newer than the one it already has.
If a file is broken, the app keeps last month's data.

## Layout

```
countries/
  index.json            list of countries and their dataVersion
  US/country.json       settings (currency, regions)
  US/prices.json        price list
  GB/country.json, GB/prices.json, GB/dishes.json
  CA/..., AU/..., DE/..., ZA/...
```

The full file format is in [FORMAT.md](FORMAT.md).

## Monthly update

1. Edit the country's `prices.json` (and `dishes.json` if needed).
2. Set the new month as `dataVersion` in that country's `prices.json` / `dishes.json` (for example `"2026-12"`).
3. Set the same new `dataVersion` for that country in `countries/index.json`. The app only downloads a country when this value goes up.
4. Commit and push to `main`. GitHub Pages publishes it in a few minutes, and phones pick it up within a day.

Rules the app checks before using a download: the currency in `prices.json` must match `country.json`, item ids must be unique, every price and pack size must be above 0, and every dish needs ingredients and steps.
