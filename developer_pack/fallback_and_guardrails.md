# Guardrails, SQL safety and fallback

## Step 0: questions answered with a fixed reply (no query)

| Trigger | Reply |
|---|---|
| Blank question | "Ask a question about the Channels or ECOM data." |
| Over 500 characters | "Please keep the question under 500 characters." |
| Prompt injection, asks for the prompt or an API key | "I can only answer questions about the forecast data." |
| What-if / simulate / scenario | "What-if questions are not answered here; use the scenario planner screen. I can compare the forecast, the back-test and last year." The agent never answers what-if questions itself. |
| Period outside the data | "This data runs from <first month> to <last month>." (months read from the data, not from the year) |
| A cause with no data | "There is no <topic> data here, so I cannot test that. I can still show where <metric> moved." Topics: price, competitor, sell-in, and any driver not yet loaded |

Weather, festivals, discount and listing move from this list to normal questions once their data is loaded (evidence_test). Until then they stay here.

## SQL safety (steps 2a and 2d; kept from the current agent)

- One `SELECT` or `WITH` only; no comments, no semicolon, no second statement.
- No write or admin keywords (INSERT, UPDATE, DELETE, DROP, CREATE, ALTER, COPY, ATTACH, PRAGMA, …).
- No file, URL or system-table access.
- Only the agent's tables and views.
- Row cap 200, added or enforced in code.
- On a rejected or failed query: one retry with the error, then "I could not run a safe query for that question." No stack trace.
- No rows: "The query ran and returned no rows for that cut of the data."

## Fallback (step 2d)

Used when the question form is valid but no shape fits, or the question needs a calculation no analysis has ("Which BPs grew volume while losing ECO %?").

- The LLM writes SQL under the safety rules above, with the map's formulas in its prompt.
- The answer is labelled **Exploratory: calculated for this question, not a standard analysis.**
- No PROVEN / causal wording in the answer; it reports what the numbers show.

## Logging

Every question is logged with: the form, the shape chosen, the analyses run, fallback yes / no, and any validation failures. Questions that went to the fallback are reviewed regularly; a pattern that keeps coming back becomes a new analysis.

## API key and no-AI mode

- Key read from the sidebar, config, or an environment variable. Without a key the chat is disabled; every other screen works.
- The analyses (step 2) must run without the LLM, so the front end can call them directly.
