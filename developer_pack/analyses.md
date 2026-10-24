# Analyses (step 2)

Each analysis is a function with parameters. Code builds the SQL from `relationship_map.yaml`, so one function covers every metric × dimension × period × filter the map allows, and the roll-up rules (sums, rates from sums, stocks averaged across months) are always applied the same way. Every function returns a small table plus a few headline fields that the answer templates use.

Settings live in one settings file, not in code.

## Phases

| Phase | Analyses |
|---|---|
| **1 (build now)** | 1 factor_split, 2 segment_contribution (with breadth), 3 rate_mix_split, 7 forecast_vs_reference (without forecast parts), 8 evidence_test in its simple form (below) |
| **2 (later)** | 4 change_timing, 5 seasonal_check, 6 persistence_check, forecast parts in 7, verdicts in 8 |

In phase 1, a question whose shape is `timing`, `seasonal` or `persistence` goes to the fallback (exploratory) path.

## 1. factor_split

**For:** volume (channels).
**Inputs:** filters, period, reference.
**Method:** compute cabinets (monthly average), ECO % and throughput for both periods, per the map. Split the change with logs, which add up exactly:
`ln(V1/V0) = ln(C1/C0) + ln(E1/E0) + ln(T1/T0)` (+ `ln(n1/n0)` if the periods differ in length).
Each factor's share of the change = its log term / the total log change. Volume in rows outside the identity is reported as its own line.
**Output:** per factor: value A, value B, % change, share of the change.

## 2. segment_contribution

**For:** additive metrics (volume, cabinets, eco_count, offtake), or a factor picked by `factor_split`.
**Inputs:** `by` dimension (or each dimension in the drill order), filters, period, reference.
**Method:** change per segment = value B − value A. Report each segment's share of the net change AND of the gross change (sum of absolute changes), because with offsetting segments the net share can exceed 100%.
**Drill-down:** if one segment holds ≥ `concentration_share` of the gross change, drill into it on the next dimension in `drill_order`, up to `max_depth`.
**Breadth:** share of segments (and of volume) moving in the same direction as the total.
**Output:** ranked segment table; drill path; breadth.

## 3. rate_mix_split

**For:** rates (eco_pct, throughput).
**Inputs:** `by` dimension, filters, period, reference.
**Method:** shift-share. With weight w (cabinets for ECO %, eco_count for throughput) and rate r per segment:
- rate effect = Σ w_share_B × (r_B − r_A): segments' own rates changed
- mix effect = Σ (w_share_B − w_share_A) × r_A: weight moved towards higher or lower-rate segments
- rate + mix = total change (exactly)
Then the rate effect is broken down by segment as in `segment_contribution`.
**Output:** total change, rate effect, mix effect, segment table.

## 4. change_timing

**For:** any metric. "When did it start / change?"
**Inputs:** filters, metric, a window (default: the full history).
**Method:** remove seasonality: year-on-year ratio per month where a prior year exists, else ratio to the seasonal norm. A change starts at the first month of a run of ≥ `sustained_months` months on the same side of the threshold (`timing_threshold`). Report the start month, the size of the change since then, and, when `by` is set, which segment's run started first.
**Output:** start month, run length, size; per segment start months.

## 5. seasonal_check

**For:** "Is this month normal?"
**Method:** compare the month's value with (a) the seasonal norm (the same calendar month's share of the year, from history) applied to the recent level, and (b) the same month in each past year.
**Output:** actual, expected, gap %, the same month in past years.

## 6. persistence_check

**For:** "Was it a blip?"
**Method:** count the months since the change that stay beyond `timing_threshold` (season-adjusted, as in change_timing). One month only = one-off; ≥ `sustained_months` = sustained; anything in between = too early to tell.
**Output:** classification and the monthly season-adjusted values.

## 7. forecast_vs_reference

**For:** forecast vs last year / business / actual.
**Method:**
- Difference = forecast − reference on the same rows. For `business`, both sides are restricted to rows where the business forecast is present.
- Errors use the pooled formulas in the map (WAPE and bias).
- Analyses 1–3 then run on the difference to show where it sits and which factor carries it.
- If the forecast row carries named parts (level, season, driver effects; see the data contracts), the difference is also split by part: e.g. "level carried from Jan–Mar actuals +x%, seasonal pattern +y%".
- If the forward forecast is the naive placeholder, the answer says so and does not explain it as a model.
**Output:** difference table, error table, part split when available.

## 8. evidence_test

**For:** testing hypotheses after analyses 1–3 have found what moved (a factor, a segment, a period).
**Inputs:** the metric and segment that moved, the period; candidate links from `evidence_links.csv` where `metric` matches and `grain` fits; plus `proposed_cause` if the user gave one.
**Method per link:**
1. Measure the driver's change in that segment and period, at the link's grain and lag.
2. Expected effect = link effect × driver change.
3. Compare with the observed change.

**Phase 1 (simple form):** for each candidate link, show the driver's change next to the metric's change in that segment and period, plus the link's `strength`, `status` and `evidence` line. No verdict and no "share explained". The answer says "possible cause" for links with status `use`, and adds "evidence is limited" for `use with caution`.

**Phase 2 verdicts (numeric, in code):**

| Verdict | Rule (thresholds in Settings) |
|---|---|
| supports | driver moved in the expected direction and the expected effect is ≥ `min_explained_share` of the observed change |
| partly supports | expected direction, but explains less than `min_explained_share` |
| does not support | driver did not move (change below `driver_noise`) |
| contradicts | driver moved the opposite way |
| cannot test | driver data missing for that segment or period |

A `proposed_cause` with no matching link and a matching column is tested by measuring the driver's change and its correlation with the metric over the segment's history; the verdict is labelled "no prior evidence, direct test only".
**Output:** one row per hypothesis: driver, change, expected effect, observed, share explained, verdict, link strength and source.

## Settings (agreed 24 Oct; kept in a settings file so they can change without code changes)

| Setting | Value | Used by |
|---|---|---|
| `concentration_share` | 0.5 of gross change | 2 |
| `drill_order` | channels: channel → region → cluster → bp; ecom: customer → place → cluster → bp | 2 |
| `max_depth` | 3 | 2 |
| `sustained_months` | 3 | 4, 6 |
| `timing_threshold` | ±5% season-adjusted | 4, 6 |
| `min_explained_share` | 0.5 | 8 |
| `driver_noise` | per driver, in `evidence_links.csv` | 8 |
| `top_n` | 5 segments in any answer table | all |
