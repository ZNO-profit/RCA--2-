# Data contracts

What the forecasting pipeline delivers to the agent's database. **Phase 1 is five CSV files** (`KW-mock/data/sample_v2/`): sample_v1's two files with three columns added, two driver files, and a metadata file. Phase 2 (at the end) is kept as the direction for later, not for building now.

All files: CSV, UTF-8, one header row, `month` as `YYYY-MM`. Missing is empty (NULL), never 0.

**Scaling:** volume, cabinets, throughput and offtake, actuals and every forecast of them, are scaled by fixed factors. ECO %, names, error ratios and all driver values are real. The factors are never delivered.

## 1. `channels.csv`

As sample_v1, one row per `BP × Channel × cluster × month`, plus three columns.

| Column | Meaning |
|---|---|
| `BP`, `BP_desc`, `Channel`, `cluster`, `month`, `kind` | keys; `kind` = `actual` or `forecast (naive)` |
| `volume`, `cabinets`, `eco_pct`, `throughput` | actuals; volume = cabinets × ECO % × throughput; `eco_pct` is a fraction |
| `business_forecast` | client's forecast, Jan–Mar 2026; a zero means missing |
| `backtest_volume`, `backtest_cabinets`, `backtest_eco_pct`, `backtest_throughput` | model back-test, Jan–Mar 2026 (Nov 2025 origin) |
| `forecast_volume`, `forecast_cabinets`, `forecast_eco_pct`, `forecast_throughput` | naive forward forecast, Apr–Dec 2026 (a placeholder: say so) |
| **`FU`, `Supply Format`, `Region`** | **new**: BP nests in FU, FU in supply format; cluster nests in region |

## 2. `ecom.csv`

As sample_v1, one row per `BP × cluster × Customer × place × month`, plus `FU`, `Supply Format`, `Region`.

| Column | Meaning |
|---|---|
| `BP`, `BP_desc`, `cluster`, `Customer` (Blinkit, Zepto, Instamart), `place` (Large cities, Other places), `month`, `kind` | keys |
| `offtake` | actual offtake |
| `backtest_offtake` | model back-test, Jan–Mar 2026 |
| `forecast_offtake` | naive forward forecast, Apr–Dec 2026 |

No business forecast for ECOM.

## 3. `channels_drivers.csv`

One row per `cluster × month`, Apr 2024 – Dec 2026. Used by both datasets (ECOM joins on `cluster`, `month`).

| Column | Meaning |
|---|---|
| `temp_mean`, `temp_max` | °C (history months only) |
| `hot_days` | days above that city's usual hot-day threshold for the calendar month (about 3 in a normal month) |
| `temp_anomaly`, `hot_days_anomaly` | difference from the same calendar month's average over all years |
| `precipitation`, `rainy_days`, `humidity` | as named |
| `festival_days`, `festival_days_anomaly` | major festival days in the month (national + the cluster's own), and the difference from that calendar month's usual |
| `online_sale_days`, `durga_puja_days`, `wedding_days` (+ `_anomaly`) | custom events: online sale days apply to MT only; Durga Puja to the East clusters |

Weather is blank for forecast months. Events cover every month, since they are a calendar.

## 4. `ecom_drivers.csv`

One row per `BP × cluster × Customer × place × month` where any value exists.

| Column | Meaning |
|---|---|
| `discount_pct`, `osa_pct`, `listing_pct`, `organic_sov_pct`, `ad_sov_pct` | platform measures, 0–100; **Large cities only**, from Jan 2025 |
| `dark_stores` | stores of that customer in the cluster's large cities or other places (same on every BP row); from May 2025 only |

## 5. `metadata.csv`

One row: `data_version`, `delivered_on`, `history_from`, `history_to`, `backtest_from`, `backtest_to`, `backtest_origin`, `forward_from`, `forward_to`, `forward_is_placeholder`, `scaled`. The agent reads its windows from here, so a new drop moves them without a code change.

## Driver names used in `evidence_links.csv`

| Evidence `driver` | Column |
|---|---|
| discount_pct, availability_osa_pct, listing_pct, organic_sov_pct, ad_sov_pct | ecom_drivers: `discount_pct`, `osa_pct`, `listing_pct`, `organic_sov_pct`, `ad_sov_pct` |
| dark_stores | ecom_drivers: `dark_stores` |
| temperature, temperature anomaly, hot-day anomaly | channels_drivers: `temp_mean`, `temp_anomaly`, `hot_days_anomaly` |
| humidity; rain | channels_drivers: `humidity`, `precipitation`, `rainy_days` |
| festivals, custom events | channels_drivers: `festival_days`, `online_sale_days`, `durga_puja_days`, `wedding_days` |
| cabinets growth, eco_count | from channels.csv (map formulas) |
| calendar month, customer, place, channel, supply format | columns of channels.csv / ecom.csv |

## Load checks (fail the load)

- Keys unique in every file.
- `eco_pct` in 0–1; platform measures in 0–100; no negative volume, cabinets, throughput or offtake.
- Months inside the windows in `metadata.csv`.
- Every BP has an FU and supply format; every cluster has one region.

## Phase 2 (later, not to build now)

When several models, origins or fitted forward forecasts need to sit side by side:
- **Forecasts in long format** (one row per model × origin × target month × series × metric), with a flag for the current model.
- **Forecast parts** (level, seasonal index, driver effects…) so the agent can say why a model produced its number, not only where the difference sits.
- **Planned driver values** (discount plan, listing plan) next to actuals.
