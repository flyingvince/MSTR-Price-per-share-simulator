# MSTR Net Sats Per Share Simulator

A single-page HTML simulator that models Strategy's (MSTR, Nasdaq) theoretical share price based on its BTC-backed balance sheet. All inputs are editable, each has a sensible default, and several are kept live via public APIs. All numbers (inputs and computed outputs) use thousand separators.

## Inputs

| Input | Default | Notes |
|---|---|---|
| BTC price | $63,948 | Whole dollars, no decimals |
| BTC held on balance sheet | ₿843,775 | |
| Total debt outstanding (preferred + convertible) | $22,218,000,000 | |
| USD cash reserve | $3,225,000,000 | |
| Fully diluted shares outstanding (FDSO) | 383,219,000 shares | |
| mNAV | 1.0 | Slider range 0.9–1.1, step 0.01, synced with an editable number field |

Share price is **not** a direct input — it's replaced by the computed "MSTR estimated price" output.

### Preset buttons

- **BTC price**: "Current price" (re-fetches the live price), "$50k", "$45k", "$40k", "$38k"
- **FDSO**: "-10%", "-5%", "Current" (= 383,219,000 baseline), "+5%", "+10%"
- **mNAV**: "2x", "3x", "4x" (jumps the number field directly, even outside the slider's visible 0.9–1.1 track)

Whichever preset button currently matches a field's value is highlighted — whether that state was reached by clicking the button, typing manually, live-fetching, or hitting Reset.

## Live data

On page load, and whenever **"Reset to defaults"** is pressed, five inputs refresh from live sources. Each fetch is independent — if one source fails, the others still resolve normally, and only the failing field falls back to its hardcoded default with its own status message ("Fetching live … data…" while in flight, cleared on success, or "Live data unavailable — using last known value." on failure). mNAV always resets to 1.0, since it has no live source.

| Field | Source | Method |
|---|---|---|
| BTC price | CoinGecko, falling back to Coinbase | Direct fetch |
| BTC held | `api.strategy.com/btc/bitcoinKpis` → `btcHoldings` | Direct fetch (CORS-enabled) |
| Total debt outstanding | `api.strategy.com/btc/mstrKpiData` → `debt` + `pref` (in $M) | Direct fetch (CORS-enabled) |
| USD cash reserve | strategy.com homepage's embedded `btcTrackerData.cash` | Routed through a CORS proxy chain, since the page itself sends no CORS header |
| FDSO | strategy.com/shares's embedded share-count data | Routed through the same CORS proxy chain, then computed client-side (see below) |

CORS proxy chain (tried in order, first success wins): `api.cors.lol` → `api.allorigins.win` → `api.codetabs.com`.

### FDSO calculation

FDSO isn't directly exposed by any API, so it's replicated from strategy.com's own logic:

```
FDSO = basic_shares_outstanding
     + options_outstanding
     + rsu_psu_unvested
     + shares of any convertible note tranche whose conversion price ≤ current MSTR price
```

Conversion prices are hardcoded to mirror strategy.com's own table:

| Tranche | Conversion price |
|---|---|
| 2028 | $183.19 |
| 2029 | $672.40 |
| 2030 | $149.77 |
| 2030B | $433.43 |
| 2031 | $232.72 |
| 2032 | $204.33 |
| STRK | $1,000.00 |

## Computed outputs

- **BTC claims from debt** = debt / BTC price
- **BTC available to common shareholders** = BTC held − BTC claims from debt
- **BTC equivalent of cash** = cash / BTC price
- **Net BTC available** = BTC available to common shareholders + BTC equivalent of cash
- **Net sats per share** = Net BTC available × 100,000,000 / FDSO
- **MSTR estimated price** = BTC price × Net sats per share / 100,000,000 × mNAV *(headline output)*

## BTC pool composition chart

A donut chart with two slices that always sum to 100% of BTC held:

- BTC available to common shareholders
- BTC claims due to debt

Below the chart:
- a summary line confirming common + claims matches the "BTC held on balance sheet" input (built-in sanity check)
- a separate additive line for "extra BTC due to cash reserves" (shown as `+X`, since it sits outside the two-slice 100% pie)

Hovering a donut segment shows its exact BTC amount and percentage. A "Show as table" toggle presents the same data accessibly (category, BTC amount, % of total BTC, USD equivalent), including the subtotal and cash "extra" row.

## Other behavior

- **Reset to defaults**: re-triggers live fetches for BTC price, BTC held, debt, cash, and FDSO (falling back to hardcoded defaults per-field on failure); resets mNAV to 1.0.
- **Warnings**: shown if BTC price is zero or negative (model can't compute), or if debt claims exceed total BTC held (negative common-shareholder BTC).
- Supports both light and dark mode.

## Usage

Open `simulator.html` directly in a browser. No build step or server required.
