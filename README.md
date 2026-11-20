# Gold & Silver Live Tracker

A single-file, static HTML dashboard that tracks live spot prices for gold (XAU) and silver (XAG) in USD per troy ounce and NPR per tola, with live-updating charts, price alerts, and a historical data table.

## Features

- **Live price tiles** for Gold and Silver: USD per troy ounce, and NPR per tola, with percent change since the last update, plus a per-gram (USD) / per-10-gram (NPR) equivalent shown underneath each.
- **Trend charts** (line) for Gold and Silver showing the last 30 updates.
- **Change bar chart** showing percent change since the page was loaded, in the selected currency.
- **Currency toggle** to switch chart/tile display between USD (per troy oz) and NPR (per tola).
- **Gold purity toggle** — 24K fine vs. 22K hallmark, applied to the gold NPR/tola price (Nepal's silver market isn't commonly quoted by karat, so this only affects gold).
- **Oz → tola conversion note** shown on the page with the exact calculation used, and the premiums currently in effect, plus a calibration-staleness warning if the premium looks out of date.
- **Adjustable, persisted premium controls** — editable gold/silver premium multiplier inputs with an "Update" button, so you can recalibrate the NPR/tola price against a current Nepal market quote directly in the browser. Saved to this browser's local storage, so it survives a reload.
- **Price alerts** — set a gold and/or silver NPR/tola threshold; once enabled (and notifications granted), crossing a threshold triggers a desktop notification and an audible beep. Settings persist locally.
- **Data table** (collapsible) listing every recorded update with timestamp and prices, plus a **CSV export** button.
- **Auto-refresh** every 30 seconds, with retry/backoff on failure, a locally cached last-known price shown if the feed is briefly unreachable, and status/warning indicators.
- **Light/dark theme** support via `prefers-color-scheme`.

## How it works

Everything lives in a single `index.html` file — no build step, no dependencies beyond [Chart.js](https://www.chartjs.org/) (loaded from a CDN).

On load, and every 30 seconds thereafter, the page fetches:

| Data | Source |
|---|---|
| Gold spot price (USD/troy oz) | [gold-api.com](https://gold-api.com) — `https://api.gold-api.com/price/XAU` |
| Silver spot price (USD/troy oz) | [gold-api.com](https://gold-api.com) — `https://api.gold-api.com/price/XAG` |
| USD → NPR exchange rate | [exchangerate-api.com](https://www.exchangerate-api.com) — `https://open.er-api.com/v6/latest/USD` (updates daily) |

Prices are held in-memory (last 30 readings per metal) and rendered into the tiles, charts, and table. No backend, database, or build tooling is required.

### Troy ounce → tola conversion

Gold and silver spot prices are quoted globally in USD per **troy ounce**, but Nepali markets price bullion in **tola**. The app converts between the two using their defining weights:

- 1 troy ounce = 31.1034768 g
- 1 tola = 11.6638038 g (180 grains)
- **1 troy oz ÷ tola weight = 31.1034768 ÷ 11.6638038 ≈ 2.66667 tola per troy oz**

So the NPR price shown per tola is calculated as:

```
NPR per tola = (USD per troy oz × USD→NPR rate × local premium × purity) ÷ 2.66667
```

(`purity` is 1 for gold at 24K fine, 22/24 for 22K hallmark, and always 1 for silver.) This exact formula (and the constant `2.66667`) is also displayed on the page itself, just below the currency/purity toggles, and is used consistently for the tiles, charts, change-bar baseline, the data table, and the CSV export.

### Local market premium

Raw spot price × FX rate alone **understates** the real Nepal counter price, because the actual daily rate set by FENEGOSIDA (the Nepal gold & silver dealers' association) bakes in import duty, VAT, and dealer margin on top of the international spot price.

To correct for this, the app applies a `LOCAL_PREMIUM` multiplier per metal:

| Metal | Default premium | Basis |
|---|---|---|
| Gold | ×1.2357 (+23.6%) | Calibrated 2026-11-20 against live spot ($3,544.87/oz) and a Nepal market rate of Rs 245,400/tola |
| Silver | ×1.3207 (+32.1%) | Calibrated 2026-11-20 against live spot ($42.033/oz) and a Nepal market rate of Rs 3,110/tola |

An earlier gold value of ×2.063 was calibrated against a stale spot price (effectively assuming gold around $2,123/oz) and badly overstated the live NPR/tola price once gold rallied well past that — it was reverted to ×1.2357. Silver's premium was originally a rough guess (×1.35); it's now calibrated the same way gold's is, against an actual quoted Nepal silver rate.

These defaults, along with the spot price and date each was calibrated against, live in the `CALIBRATION` constant in the script (which seeds `LOCAL_PREMIUM`). They seed the "Adjust premiums" input fields on the page. You can type in new gold/silver multipliers there and click **Update** to immediately recalculate every tile, chart, and table row — the conversion note updates to show the active multipliers and their percentage, and the new values are saved to `localStorage` (`xauxag.premiums`), so they survive a page reload. To change the *defaults* (what a fresh browser with no saved value sees), edit `CALIBRATION`/`LOCAL_PREMIUM` in the script.

**Calibration staleness warning**: because each premium is a single fixed multiplier, it drifts out of date as spot prices move relative to the Nepal market (duty, VAT, and dealer margin don't scale perfectly with the international price). The page compares the live spot price against the spot price (and date) each premium was calibrated against, and shows a warning beneath the conversion note if more than 14 days have passed or the spot price has drifted more than 15% — a prompt to re-check a current Nepal market quote and update the premium (and ideally the `CALIBRATION` constant) again.

### Gold purity (24K fine vs. 22K hallmark)

Nepal's bullion market commonly quotes gold at two purities: 24K fine (tejabi) and 22K hallmark. The purity toggle applies a 22/24 (≈91.7%) ratio to the gold NPR/tola price when "22K Hallmark" is selected; "24K Fine" (the default) applies no adjustment. This is a standard-ratio approximation, not a live quote — Nepal's actual fine/hallmark spread can vary slightly by dealer and day. The selection is saved to `localStorage` (`xauxag.purity`).

### Price alerts

The "Price alerts" row lets you set an NPR/tola threshold for gold and/or silver. Click **Enable notifications** once to grant the browser's Notification permission (also required to prime the beep's audio context, since browsers require a user gesture to start audio); check **Enable** and click **Save** to activate. On each refresh, if a metal's current NPR/tola price is at or above its threshold, the page fires a desktop notification (if permission was granted) and plays a short beep (via the Web Audio API — no external sound file). It won't re-notify again for the same crossing until the price first drops back below the threshold. Threshold values and the enabled flag are saved to `localStorage` (`xauxag.alerts`); whether a threshold has already fired is **not** persisted, so every page load can alert once more on the first crossing it sees.

### Resilience: retry, fallback hook, and offline cache

- **Retry with backoff**: each price/FX fetch is retried up to 3 times with exponential backoff (800ms, 1.6s, …) before being treated as failed.
- **Fallback source hook**: each data source is defined as a list of URLs (`GOLD_URLS`, `SILVER_URLS`, `FX_URLS`) that are tried in order, so adding a second provider later is a one-line change. No second free, keyless gold/silver price API was found to be reliable at the time of writing — `goldprice.org`'s endpoint returned `403 Forbidden` and `metals.live` was unreachable — so each list currently holds only the primary URL, and resilience instead relies on retry/backoff plus the cache below.
- **Offline cache**: every successful fetch saves the latest prices and FX rate to `localStorage` (`xauxag.lastKnown`). If the very first fetch of a page load fails (after retries), the page falls back to that cached snapshot and shows an "Offline — showing cached data from …" status instead of leaving the tiles blank.

### CSV export

Inside the "View data as a table" panel, the **Export CSV** button downloads everything currently held in memory (up to the last 30 updates) as a CSV file with the same columns as the on-screen table (USD/oz and NPR/tola for both metals, timestamped), computed with the currently active premium and purity settings.

## Usage

Just open `index.html` in a browser:

```bash
open index.html
```

Or serve it with any static file server, e.g.:

```bash
python3 -m http.server 8000
```

then visit `http://localhost:8000`.

## Notes

- Prices are **indicative only** — not an executable trading price.
- Since all requests are made client-side, the page requires outbound network access to `gold-api.com` and `exchangerate-api.com`, and no API keys are needed.
- Premiums, purity selection, and alert settings are stored in the browser's `localStorage`, so they're per-browser/per-device, not shared or synced anywhere.
