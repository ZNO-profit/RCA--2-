# Answer templates (step 3)

The LLM writes the answer from the analysis output only. It may not add numbers, causes or segments that are not in that output. Every answer:

- leads with the answer in one or two sentences;
- shows one small table (at most `top_n` rows) taken from the analysis;
- states the defaults applied, and "volumes are in scaled units" once when a volume is shown;
- is short. No headings on answers under ~6 lines.

## lookup

> **<metric> for <scope>, <period>: <value>.** <one line of context if the question asked for a comparison>.
> Table: the query result (capped).

## move

> **<Metric> is <+/-x%> vs <reference>. Most of it is <factor> (<share>%), concentrated in <segment path> (<share>% of the change).**
> Table: factor split, then the top segments.
> **Likely causes:** each hypothesis with verdict *supports* or *partly supports*: driver, its change, share explained, evidence strength.
> **Checked, not supported:** one line listing the others.
> **Not explained:** the share no tested link accounts for, stated as a number.

## rate_move

> **<Rate> moved from <A> to <B>. <Rate effect>% comes from segments' own <rate> changing, <mix effect>% from <weight> shifting between segments.**
> Then as in *move* for the rate effect.

## timing

> **<Metric> started to <rise / fall> in <month> (season-adjusted) and has stayed <x%> <above / below> since.** <Segment> turned first, in <month>.

## seasonal

> **<Month> is <x%> <above / below> what the usual seasonal pattern gives.** Same month in past years: <values>.

## breadth

> **<n> of <N> <segments> (<share>% of volume) moved the same way.** <Broad / concentrated in …>.

## persistence

> **<One-off / sustained / too early to tell>:** <months> beyond the threshold since <month>.

## forecast_vs_reference

> **The forecast is <+/-x%> vs <reference>.** <Where it sits and which factor or forecast part carries it>.
> For the back-test: WAPE <w>%, bias <b>% (negative = forecast low), on <rows>.
> If the forward forecast is the naive placeholder: "The Apr–Dec forecast is a placeholder that carries recent levels forward, not a fitted model."

## Exploratory (fallback)

> *Exploratory: calculated for this question, not a standard analysis.* <What the numbers show, no causal wording>.
