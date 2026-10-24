# Evidence links (map layer 2)

`evidence_links.csv` holds what the EDA and tests have shown about drivers, seasonality and trends: one row per link. The agent's `evidence_test` (analyses.md, 8) reads it to turn "this factor moved in this segment" into hypotheses and to judge them. Layer 1 (`relationship_map.yaml`) is always true; this layer is evidence and changes as evidence improves.

## Columns

| Column | Meaning |
|---|---|
| `link_id` | Stable id. E = ECOM, C = channels, T = to test. Never reused. |
| `dataset` | `channels` or `ecom` |
| `link_type` | `driver` (an input moves the metric), `seasonality`, `trend`, `structural` (follows from how the metrics are built) |
| `driver` | What moves. Names map to columns through the driver data contract |
| `metric` | What it moves (a map metric, or a share / growth of one) |
| `grain` | The level the evidence was read at. The agent only applies a link at this grain or finer |
| `direction` | `+`, `-`, `none`, `mixed`, or a named pattern (e.g. `cluster index`) |
| `effect`, `effect_unit`, `effect_low`, `effect_high` | Size and its range, where there is one |
| `lag_months` | 0 = same month |
| `strength` | `strong`, `moderate`, `weak`, `untested` (rules below) |
| `status` | `use`, `use with caution`, `do not use`, `to test` |
| `driver_noise` | A driver change smaller than this counts as "did not move" |
| `period`, `data_version` | What data the evidence comes from |
| `test`, `evidence` | How it was found, in one or two sentences |
| `source_report` | The report that backs it |

No volumes, cabinets or throughput levels go in this file: only % effects, correlations, ratios and shares.

## Strength

| Strength | Rule |
|---|---|
| strong | Joint test significant (p < 0.001), same sign in most series (≥ 60%) and, where tested, it held in a back-test |
| moderate | Significant but small, held in only some fits, or rests on few readings |
| weak | Near zero, or unstable in sign or size across periods |
| untested | Not tested at the grain or form that matters |

## How the agent uses status

| Status | In an answer |
|---|---|
| use | Tested; verdict reported normally |
| use with caution | Tested; the verdict carries "evidence is limited: <one line from `evidence`>" |
| do not use | Not offered as a hypothesis. If the user proposes it: "Checked in earlier analysis, no reliable link: <evidence>" |
| to test | Not offered as a hypothesis. If the user proposes it: "Not yet tested" plus, if the driver data exists, a direct test labelled "no prior evidence" |

## How it improves over time

1. **EDA reruns.** When the standard EDA (5.1 / 5.2) or a test is rerun on new data, its export updates the rows it covers: effect, range, strength, `period`, `data_version`. Rows are updated in place, keyed by `link_id`.
2. **To test → tested.** A `T` row that has been tested gets its result and a new status. The id stays; `link_type` and the rest are filled in.
3. **New rows from the agent's own logs.** Two signals in the question log suggest new links to test:
   - a large "not explained" share for the same metric and segment across several questions;
   - user-proposed causes that come up repeatedly with no link.
   These are reviewed periodically and added as `T` rows. The agent never adds or changes rows itself.
4. **Change log.** Every change goes through git with a one-line reason, so an answer can be traced back to the evidence that was live at the time.

## Open now

- **Channel rows (C) come from the old raw file** (before the switch to ZNOVA Data 3, 14 Oct 2026), so all are `use with caution` for now, whatever their strength. Rerun the channel EDA on current data, then reset their status.
- **Weather at the right grain (T01):** temperature and hot-day anomalies against ECO % and throughput by channel × cluster (and BP × cluster), peak months separately. Cluster-level totals show little (C06), but BP × cluster throughput in Apr–May shows 0.4–0.8 for many pairs (C07): the signal looks concentrated in the peak and at finer grain.
- **Festivals and events (T02):** major-festival lift by cluster; online-sale days for MT; Durga Puja for the East.
- **ECOM rows** rest on 15 months of platform data (large cities only). Refresh with Apr 2026+.
