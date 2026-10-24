# Data contracts

What the forecasting pipeline delivers to the agent's database on each data drop, and the rules the load checks. The current `sample_v1` (channels.csv, ecom.csv) is the starting point. This contract (v2) splits it into facts, dimensions, drivers and forecasts so a new model or a new driver needs no change to the agent.

All files are CSV, UTF-8, one header row, `month` as `YYYY-MM`. Missing is empty (NULL), never 0, unless a column says otherwise.

**Scaling (unchanged):** volume, cabinets, throughput and offtake (actuals and every forecast of them) are scaled by fixed factors before delivery. ECO %, shares, growth, WAPE, bias, driver values and effects are real. The factors are never delivered.

## 1. Metadata: `metadata.csv`

One row per drop.

| Column | Meaning |
|---|---|
| `data_version` | e.g. `v2-2026-10` |
| `delivered_on` | date |
| `history_from`, `history_to` | first and last month of actuals |
| `backtest_from`, `backtest_to`, `backtest_origin` | back-test window |
| `forward_from`, `forward_to` | forward forecast window |
| `scaled` | always `true` |

The agent reads its windows from here (the `windows` block in `relationship_map.yaml` becomes these values), so a new drop moves the windows without a code change.

## 2. Dimensions

**`products.csv`**: `bp`, `bp_desc`, `fu`, `supply_format`. One row per BP; BP nests in FU, FU in supply format.

**`geography.csv`**: `cluster`, `region`. One row per cluster; cluster nests in region.

The load fails if a fact or forecast row has a BP or cluster missing here.

## 3. Facts (actuals)

**`channels_actuals.csv`**: one row per `bp × channel × cluster × month`.

| Column | Notes |
|---|---|
| `bp`, `channel`, `cluster`, `month` | key |
| `volume` | scaled |
| `cabinets` | scaled; a stock (see the map) |
| `eco_pct` | fraction 0–1 |
| `throughput` | scaled; volume per active cabinet |

`eco_count` is derived by the agent (`cabinets × eco_pct`), not delivered.

**`ecom_actuals.csv`**: one row per `bp × cluster × customer × place × month`: `offtake` (scaled).

## 4. Drivers

Each driver table holds actual values for history months and, where a plan exists, planned values for forecast months, with `value_type` = `actual` or `plan`.

**`weather.csv`**: `cluster × month`.

| Column | Meaning |
|---|---|
| `temp_mean`, `temp_max` | °C |
| `hot_days` | days above the city's 90th-percentile max temperature |
| `precipitation`, `rainy_days`, `humidity` | as in the weather table |
| `temp_anomaly`, `hot_days_anomaly` | difference from the same calendar month's average over all years |

**`events.csv`**: `cluster × month × event` (long): `event`, `channel` (blank = all channels), `days`, `anomaly` (days minus the usual for that calendar month in that cluster). Events: national and regional festivals, plus custom events (online sale days for MT, Durga Puja, wedding days).

**`ecom_platform.csv`**: `bp × cluster × customer × place × month`: `discount_pct`, `osa_pct`, `listing_pct`, `organic_sov_pct`, `ad_sov_pct` (0–100). Large cities only; other places are NULL. From 2025-01.

**`ecom_stores.csv`**: `cluster × customer × place × month`: `dark_stores` (count as delivered; months before 2025-05 under-cover small cities and are NULL for those cells).

### Driver names used in `evidence_links.csv`

| Evidence `driver` | Table.column |
|---|---|
| discount_pct | ecom_platform.discount_pct |
| availability_osa_pct | ecom_platform.osa_pct |
| listing_pct | ecom_platform.listing_pct |
| organic_sov_pct, ad_sov_pct | ecom_platform.organic_sov_pct, ad_sov_pct |
| dark_stores | ecom_stores.dark_stores |
| temperature, temperature anomaly, hot-day anomaly | weather.temp_mean, temp_anomaly, hot_days_anomaly |
| humidity; rain | weather.humidity, precipitation, rainy_days |
| festivals, custom events | events (by `event`) |
| cabinets growth, eco_count | from channels_actuals (map formulas) |
| calendar month, customer, place, channel, supply format | dimensions (no table) |

A new driver means a new table or column, a row here, and its links in `evidence_links.csv`. No agent code changes.

## 5. Forecasts

Long format, so any number of models, runs and origins can sit side by side.

**`forecasts.csv`**: one row per forecast value.

| Column | Meaning |
|---|---|
| `dataset` | `channels` or `ecom` |
| `run_type` | `backtest`, `forward`, `business` |
| `model_id` | e.g. `channels_AB2_2026-10`; `business` for the client's forecast |
| `model_label` | short readable name, used in answers |
| `is_placeholder` | `true` for the naive placeholder (answers say so) |
| `origin` | last month of actuals the forecast used (`YYYY-MM`) |
| `target_month` | month forecast |
| `horizon` | months from origin to target |
| keys | channels: `bp`, `channel`, `cluster`; ecom: `bp`, `cluster`, `customer`, `place` |
| `metric` | `volume`, `cabinets`, `eco_pct`, `throughput`, `offtake` |
| `value` | the forecast (scaled where the metric is) |

Rules:
- The business forecast is `run_type = business`, `metric = volume`, channels only. A business zero is delivered as NULL (missing), and comparisons use only rows where it is present.
- Channels models that forecast cabinets, ECO % and throughput deliver all three plus volume.
- `(run_type, model_id, origin, target_month, keys, metric)` is unique.
- A `model_flags.csv` names the current model per dataset and run type (`is_current = true`). By default the agent answers with the current model and names it.

**`forecast_parts.csv`**: optional, one row per named part of a forecast value.

| Column | Meaning |
|---|---|
| the key columns of `forecasts.csv` | which value this part belongs to |
| `part` | e.g. `level`, `seasonal_index`, `growth`, `discount_effect`, `listing_effect`, `expansion` |
| `part_type` | `multiplicative` (a factor) or `additive` (an amount) |
| `value` | the factor or amount |

Rule: within one forecast value, the multiplicative parts multiply (and additive parts add) to `value` within 0.1%; the load checks this. A model that cannot be split delivers no parts, and the agent explains its forecast through the analyses only.

## 6. Load checks (fail the load)

- Keys unique in every table; every BP and cluster exists in the dimensions.
- `eco_pct` in 0–1; platform percentages in 0–100; no negative volume, cabinets, throughput or offtake.
- Months inside the windows in `metadata.csv`.
- Forecast parts reconcile to their forecast value.
- Every `driver` in `evidence_links.csv` with status `use` or `use with caution` maps to a column that exists.
