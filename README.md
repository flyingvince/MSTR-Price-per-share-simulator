# MSTR - Price per share simulator

A single-page HTML simulator that models Strategy's (MSTR, Nasdaq) theoretical share price based on its BTC-backed balance sheet. All inputs are editable, each has a sensible default, and several are kept live via public APIs. All numbers (inputs and computed outputs) use thousand separators.

## Inputs

| Input | Default | Notes |
|---|---|---|
| BTC price | $63,948 | Whole dollars, no decimals |
| BTC held on balance sheet | ₿843,775 | |
| Total debt outstanding (preferred + convertible) | $22,218,000,000 | |
| USD cash reserve | $3,225,000,000 | |
| Fully diluted shares outstanding (FDSO) | 388,648,000 shares | |
| mNAV | 1.0 | Slider range 0.9–1.1, step 0.01, synced with an editable number field |

Share price is **not** a direct input — it's replaced by the computed "MSTR estimated price" output.

### Preset buttons

- **BTC price**: "Current price" (re-fetches the live price), "$50k", "$45k", "$40k", "$38k"
- **FDSO**: "-10%", "-5%", "Current" (= 388,648,000 baseline), "+5%", "+10%"
- **mNAV**: "2x", "3x", "4x" (jumps the number field directly, even outside the slider's visible 0.9–1.1 track)

Whichever preset button currently matches a field's value is highlighted — whether that state was reached by clicking the button, typing manually, live-fetching, or hitting Reset.

## Live data

On page load, and whenever **"Reset to defaults"** is pressed, BTC price, BTC held, USD cash reserve, and total debt outstanding refresh from live sources. Each fetch is independent — if one source fails, the others still resolve normally, and only the failing field falls back to its hardcoded default with its own status message ("Fetching live … data…" while in flight, cleared on success, or "Live data unavailable — using last known value." on failure).

| Field | Source | Method |
|---|---|---|
| BTC price | CoinGecko, falling back to Coinbase | Direct fetch |
| BTC held | `api.strategy.com/btc/bitcoinKpis` → `btcHoldings` | Direct fetch (CORS-enabled) |
| USD cash reserve | `api.strategy.com/btc/bitcoinKpis` → `totalAnnualDividends` × `usdMonthsOfDividends` / 12 | Direct fetch (CORS-enabled), derived (see below) |
| Total debt outstanding | `api.strategy.com/btc/mstrKpiData` → `debt` + `pref` (in $M) | Direct fetch (CORS-enabled) |

### Cash reserve derivation

`bitcoinKpis` doesn't expose the USD cash reserve directly, but it can be recovered from two fields it does return: `usdMonthsOfDividends` (how many months the cash reserve covers at the current annualized dividend run-rate) and `totalAnnualDividends`:

```
cash = usdMonthsOfDividends × (totalAnnualDividends / 12)
```

This reproduces the reserve figure exactly (validated against a known $3,225,000,000 reserve) without needing a CORS proxy.

FDSO is the only field that's **not** live-fetched — it resets to its hardcoded default on load/Reset. Strategy.com exposes no CORS-enabled API for the per-tranche convertible-note data FDSO requires (only the two KPI endpoints above allow direct browser access); routing it through a public CORS proxy was tried and dropped, since the proxies proved too unreliable in practice — aggressive rate limits (~1 request/30s), and strategy.com's Cloudflare protection blocking several proxy providers' IPs outright.

## Computed outputs

- **MSTR estimated price** = BTC price × Net sats per share / 100,000,000 × mNAV
- **MSTR actual price** — live, fetched from `api.strategy.com/btc/mstrKpiData` → `ufPrice` on load and on Reset (same request used for total debt outstanding)
- **Implied mNAV** = MSTR actual price / MSTR estimated price — how the live market price compares to the model's estimate; distinct from the **mNAV** *input*, which feeds into the estimated-price formula above rather than being derived from it

Below these, the card has a second, blank spacer row (same markup/height as a real stat row, `aria-hidden`) so the card's overall height stays constant — it doesn't currently show any fields, reserved for future metrics.

Internally, a few more figures are computed from the inputs and drive the chart below rather than being shown as their own stat tiles:

- **BTC claims from debt** = debt / BTC price
- **BTC available to common shareholders** = BTC held − BTC claims from debt
- **BTC equivalent of cash** = cash / BTC price
- **Net BTC available** = BTC available to common shareholders + BTC equivalent of cash
- **Net sats per share** = Net BTC available × 100,000,000 / FDSO *(also shown at the center of the donut chart)*

## BTC pool composition chart

A donut chart with two slices that always sum to 100% of BTC held:

- BTC available to common shareholders
- BTC claims due to debt

Below the chart:
- a summary line confirming common + claims matches the "BTC held on balance sheet" input (built-in sanity check)
- a separate additive line for "extra BTC due to cash reserves" (shown as `+X`, since it sits outside the two-slice 100% pie)

Hovering a donut segment shows its exact BTC amount and percentage. A "Show as table" toggle presents the same data accessibly (category, BTC amount, % of total BTC, USD equivalent), including the subtotal and cash "extra" row.

## Other behavior

- **Reset to defaults**: re-triggers live fetches for BTC price, BTC held, cash, and debt (falling back to hardcoded defaults per-field on failure); resets FDSO and mNAV to their hardcoded defaults.
- **Warnings**: shown if BTC price is zero or negative (model can't compute), or if debt claims exceed total BTC held (negative common-shareholder BTC).
- Supports both light and dark mode.

## Usage

Open `simulator.html` directly in a browser. No build step or server required.
