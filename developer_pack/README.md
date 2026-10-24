# Forecast RCA agent: developer pack

Status: draft v0.1 (24 Oct 2026). Thresholds, defaults and the product / region levels agreed 24 Oct. Items still marked **TO AGREE** need a decision before build.

Data change needed for this design: the export adds `supply_format`, `fu` and `region` to both datasets (BP nests in FU, FU in supply format; cluster in region).

This pack replaces the hypothesis-loop design in `Forecast_RCA_Side_by_Side_Review.docx`. The agent answers three kinds of question about the Channels and ECOM data:

1. **Lookup**: a number, list, rank or comparison ("What was SS volume in Q1 2026?").
2. **Why did an actual move**: volume, ECO %, throughput or offtake changed ("Why is SS ECO % down on last year?", "When did it start?").
3. **Why is the forecast what it is**: forecast vs last year, vs the business forecast, or the back-test vs actual.

## Design in one line

The LLM only understands the question and writes the answer. Everything in between (the maths, the breakdowns, the hypothesis tests) is code driven by a relationship map, so answers are exact and repeatable and the same code can serve front-end screens without the chatbot.

## Structure

```
 A. DATA (loaded offline, refreshed with each data drop)
    Facts       channels, ecom actuals
    Drivers     weather, festivals, platform measures, stores    -> contract: item 8 (to come)
    Forecasts   back-test, forward, business, with named parts   -> contract: item 8 (to come)

 B. KNOWLEDGE (files in this pack, read by code)
    relationship_map.yaml   layer 1: metrics, formulas, aggregation, hierarchies, factor splits
    evidence_links.csv      layer 2: driver -> metric links from EDA      (item 7, to come)
    settings                thresholds used by the analyses (analyses.md)

 C. PER QUESTION
    0. Guardrail check         code   out of scope -> fixed reply              fallback_and_guardrails.md
    1. Understand              LLM    fills the question form                  question_form.md
                               code   validates the form against the map
    2a. Lookup                 LLM writes SQL, code validates and runs it
    2b. Why (actual)           code   analysis for the shape                   analyses.md
                               code   evidence links -> hypotheses -> tests
    2c. Why (forecast)         code   same analyses on forecast - reference,
                                      plus the forecast's named parts
    2d. Fallback               LLM writes SQL for anything no shape covers; answer labelled "exploratory"
    3. Write the answer        LLM    template per shape, numbers only from step 2   answer_templates.md
```

## What changes from the current agent

| Current module | New |
|---|---|
| Guardrails | Kept (step 0); weather / promo replies change once driver data is loaded |
| 2.1 Router (2 classes) | Step 1: 3 types + shape + full question form, validated in code |
| 2.2 Direct SQL agent | Kept (step 2a), unchanged rules |
| 2.3 Hypothesis agent | Replaced: hypotheses come from `evidence_links.csv` for the factor / segment that moved |
| 2.4 SQL agent per hypothesis | Replaced: fixed analyses + evidence tests in code. Kept only as the fallback (2d) |
| 2.5 Evaluator (LLM) | Replaced: numeric verdict rules in code (analyses.md) |
| 2.6 Validation / report writer | Kept, one template per shape (step 3) |
| SCHEMA / CAUSAL_HIERARCHY / metadata prompts | Replaced by the map + settings; the LLM sees a short generated summary of the map |

## Files

| # | File | What |
|---|---|---|
| 1 | README.md | This overview |
| 2 | question_form.md | The form step 1 fills: types, shapes, fields, allowed values, defaults |
| 3 | analyses.md | The analysis functions: inputs, method, outputs, verdict rules, settings |
| 4 | fallback_and_guardrails.md | Fixed replies, SQL safety, fallback path, logging |
| 5 | answer_templates.md | One template per shape |
| 6 | relationship_map.yaml | Layer 1 of the map |
| 7 | evidence_links.csv + contract | Layer 2, from EDA (to come) |
| 8 | data_contracts.md | Driver data and forecast parts (to come) |
| 9 | question_bank.csv | Acceptance tests by shape, with expected answers (to come) |

## Rules that hold everywhere

- Volumes, cabinets and throughput in the agent's data are scaled. ECO %, shares, growth, WAPE and bias are real. Every answer that shows a volume says it is in scaled units, once.
- Never add Channels volume to ECOM offtake.
- Rates are recomputed from sums at every grain, never averaged (see the map).
- The chatbot is optional scope: no screen may depend on it, and the analysis code must run on its own.
- Models will change. Nothing in the agent may depend on a particular model; it reads forecasts and their named parts through the contract.
