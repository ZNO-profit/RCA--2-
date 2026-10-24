# Question form (step 1)

The LLM's only job in step 1 is to fill this form from the user's question. Code then validates every field against `relationship_map.yaml` and the values present in the data. If a required field cannot be filled, the agent asks one short follow-up question instead of guessing.

## Fields

| Field | Required | Allowed values | Notes |
|---|---|---|---|
| `type` | yes | `lookup`, `why_actual`, `why_forecast` | |
| `shape` | for the two why types | see Shapes below | `lookup` uses `none` |
| `dataset` | yes | `channels`, `ecom` | Both only for lookups that show them side by side, never summed |
| `metric` | yes | `volume`, `cabinets`, `eco_count`, `eco_pct`, `throughput`, `offtake`, `wape`, `bias`, `growth`, `share` | Must exist in `dataset` |
| `measure` | yes | `actual`, `backtest`, `forecast`, `business` | Which numbers: actuals or a forecast |
| `period` | yes | month range `YYYY-MM..YYYY-MM` | The period asked about |
| `reference` | for comparisons | `last_year`, `prior_period`, `business`, `actual`, or an explicit range | What it is compared with |
| `filters` | no | `{dimension: [values]}` | Values must exist in the data |
| `by` | no | a dimension | For "which X…" or a requested breakdown |
| `proposed_cause` | no | free text, mapped by code to a column or driver if possible | "Is it because of …?" |

## Shapes

| Shape | Typical question | Analysis (analyses.md) |
|---|---|---|
| `move` | Why is volume down vs last year? | factor_split → segment_contribution → evidence_test |
| `rate_move` | Why did ECO % fall? | rate_mix_split → evidence_test |
| `timing` | When did SS ECO % start falling? | change_timing |
| `seasonal` | Is the April drop normal? | seasonal_check |
| `breadth` | Is the fall everywhere or in a few places? | segment_contribution (breadth output) |
| `persistence` | Was March a blip? | persistence_check |
| `forecast_vs_reference` | Why is Apr–Dec above last year? Why was the back-test low? | forecast_vs_reference |
| `none` | (lookups) | — |

A question can need two shapes ("when did it start and why?"). The form then holds a list of shapes, run in order.

## Defaults (agreed 24 Oct)

| Missing | Default |
|---|---|
| `reference` on a "why down / up" question | `last_year` |
| `period` | latest 3 complete months of the measure's window |
| `measure` on a forecast question with no window named | `forecast` if the period is in Apr–Dec 2026, `backtest` if in Jan–Mar 2026 |
| `dataset` when the question names neither | ask |

The answer always states which defaults were applied ("Compared with Jan–Mar 2025, since no period was given").

## Validation in code

- `metric` exists in `dataset`; `measure` exists for `period` (e.g. no business forecast for ECOM, no forward forecast before Apr 2026).
- Every filter value exists. Close misspellings are matched to a known value and shown in the answer ("SS" → `SS`; "Bengal" → `WEST BENGAL`, TO AGREE: synonym list).
- Rejected forms go back to the LLM once with the reason; on a second failure the agent asks the user.

## Example

"Why did SS ECO % drop in the East this year?"

```json
{
  "type": "why_actual",
  "shape": ["rate_move"],
  "dataset": "channels",
  "metric": "eco_pct",
  "measure": "actual",
  "period": "2026-01..2026-03",
  "reference": "last_year",
  "filters": {"channel": ["SS"], "region": ["East"]},
  "by": null,
  "proposed_cause": null
}
```
