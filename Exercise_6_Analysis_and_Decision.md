# Exercise 6 — Analysis and Decision

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The task

Estimated duration: 1 hour

PackageDelivery's management asks one question: **should we offer taxi service with our idle vans — and where?**

You have the data in the publish layer. In this exercise you turn it into a recommendation per borough, written as a decision memo. ADA writes the queries and drafts the memo. You decide what the numbers mean, and you are responsible for every claim in the memo.

A good recommendation is not always *yes* or *no*. Where the data does not support a recommendation, the professional answer is to say so.

**Result of this exercise:** a decision memo in `docs/decision_memo_fleetops.md`, committed to Git.

---

### The decision model

Management has agreed on this model. Every figure in your memo must follow it. It covers the five boroughs Manhattan, Brooklyn, Queens, Bronx and Staten Island.

- **D1** Margin per idle hour = 0.5 × taxi revenue per engaged hour − average cost per mile of the borough's taxi-capable vehicles × fleet mph. Fleet mph = total miles divided by total movement hours of all the borough's vehicles. The 0.5 is an assumed utilisation: half of an idle hour carries a paying passenger. Driver cost is excluded, because idle hours fall inside shifts that are already paid.
- **D2** Taxi revenue per engaged hour = total `total_amount` divided by total trip hours, per borough of the taxi pickup zone. Only trips of 1 to 120 minutes (both inclusive) with `total_amount` above 0.
- **D3** Idle capable hours per day = idle hours of the borough's taxi-capable vehicles, divided by the number of distinct shift dates of all the borough's vehicles.
- **D4** Verdict: **Expand** if the margin per idle hour is above 0 **and** the idle capable hours per day are at least 8. Otherwise **Do not expand**. The verdict is **Expand, airport-weighted** if the borough would expand and at least 25 % of its taxi revenue (same trips as D2) comes from trips that start at JFK (zone 132) or LaGuardia (zone 138), or use ratecode 2 or 3.
- **D5** Confidence rule. A borough gets a verdict from D4 only if **all** hold: at least 5 vehicles (all types) have it as their home borough; at least 500 fleet movements start in its zones; the fleet visits at least 20 of its zones (pickup or dropoff, not counting zones 264 and 265); at least 30 taxi trips (all trips, not only the D2 trips) start in its zones. Otherwise the verdict is **Insufficient evidence**.
- **D6** Monthly potential = margin per idle hour × idle capable hours per day × 26 working days.

The fleet terms *taxi-capable*, *idle* and *home borough* are the rules R3–R5 from Exercise 5.

---

### Part A — Hypotheses First

Estimated duration: 10 minutes

Before you query anything, write down your expectation for each borough — in a note, not to ADA. You have seen the data in the earlier exercises.

| Borough | Your expected verdict | Why |
|---|---|---|
| Manhattan | | |
| Brooklyn | | |
| Queens | | |
| Bronx | | |
| Staten Island | | |

ℹ A hypothesis written before the analysis protects you from reading into the numbers whatever you hoped to see. Where the result differs from your hypothesis, you have learned something — note what.

---

### Part B — Compute the Metrics

Estimated duration: 20 minutes

1) **Example prompt** (to `ada-developer`):

> Write a Snowflake query on my publish layer in `ADE_TRAINING_DEV_SAAS` that applies this decision model per borough: [paste the decision model D1–D6]
>
> Use `F_TRIP` and `D_TAXI_ZONE` in `PUBLISH_<YOUR_ALIAS>` for the taxi figures, and my FLEETOPS publish entities for the fleet figures. Return one row per borough with every input of D1–D5, the results of the four confidence checks, the margin, the monthly potential and the verdict. Comment each part of the query with the rule it implements.

**Under the hood:** ADA reads the publish YAML to find the tables and columns. It cannot run the query in Snowflake — you do.

2) Before you run it, ask ADA to show which line implements which rule. Check D5 especially: the vehicles count by **home borough**, but the movements, zones and taxi pickups count by the borough of the **zone**.

3) Run the query in Snowflake. If it fails, paste the error to ADA.

ℹ When you paste results to ADA, copy them from the Snowflake result grid. They paste as tab-separated text that ADA can read. A table copied from this page loses its column breaks.

#### ✅ Checkpoint B

Compare your result with the reference. Small differences (±1 in the last digit) are rounding.

| Borough | Taxi $ / engaged h | Airport share | Idle capable h / day | Margin / idle h | Monthly potential | Confidence | Verdict |
|---|---|---|---|---|---|---|---|
| Manhattan | 79.3 | 8 % | 2.3 | 34.4 | 2,035 | passes | **Do not expand** |
| Brooklyn | 72.4 | 0 % | 15.8 | 28.9 | 11,898 | passes | **Expand** |
| Queens | 86.1 | 38 % | 13.2 | 33.6 | 11,551 | passes | **Expand, airport-weighted** |
| Bronx | 73.5 | 0 % | 1.2 | 29.2 | — | fails: 2 vehicles, 16 zones | **Insufficient evidence** |
| Staten Island | — | — | 0.0 | — | — | fails: 1 vehicle, 313 movements, 2 zones, no taxi pickups | **Insufficient evidence** |

If a figure differs, paste your result to ADA, tell it which figure differs from which reference value, and ask which rule could explain the difference. Fix the query, not the reference.

---

### Part C — Interpret

Estimated duration: 15 minutes

Numbers do not make a decision. Answer these questions **in a note** — you will give your answers to ADA in Part D. Where you need extra queries, ADA writes them and you run them in Snowflake.

1) Manhattan has the **highest** margin per idle hour. Why is the verdict still *do not expand*?

2) The Bronx margin looks as good as Brooklyn's. Why no recommendation?

3) How sensitive is the result to the assumed utilisation of 0.5? At what utilisation would Brooklyn's margin turn negative?

4) Does the data support the utilisation of 0.5 for a fleet that is idle between 10:00 and 15:00? Look at when taxi trips start. **Example prompt:**

> I want to know whether the reference taxi data shows demand between 10:00 and 15:00. Write a Snowflake query on `F_TRIP` that counts the trips per pickup hour.

What does the result mean for the model?

5) Did the data issues you handled earlier — the kilometres, the NULL dropoffs, the unknown zones — change any verdict?

<details markdown="1">
<summary markdown="span">Discussion</summary>

1) Manhattan's fleet is almost fully busy: 2.3 idle hours per day against the required 8, and 9 of its 14 vehicles are box trucks that cannot carry passengers. There is nothing to convert. Entering Manhattan would need new vehicles and drivers — a different decision from the one asked. The naive per-hour analysis says yes; that is the trap.

2) The confidence rule. Two vehicles and 16 zones are not enough evidence. *Insufficient evidence* is not *no* — it means the data cannot answer the question. Staten Island fails every check.

3) Brooklyn's margin turns negative at a utilisation of about 10 % (0.10 × 72.4 ≈ 0.66 × 11.0). The verdicts for Brooklyn and Queens hold over a wide range. The memo should still name 0.5 as an assumption.

4) Almost all reference taxi trips start at night or early in the morning. Only 9 of about 8,000 start between 10:00 and 15:00. The model therefore uses the taxi **price level** per borough, not taxi **demand at midday**. The data neither supports nor refutes the 0.5 utilisation at midday — this is the most important limitation of the memo.

5) No. Without the kilometre conversion, Brooklyn's margin would be about 28.2 instead of 28.9. Not every data quality issue changes the decision — and knowing which ones do is part of the job. The memo should still state how each issue was handled.

</details>

---

### Part D — Write the Memo

Estimated duration: 15 minutes

1) **Example prompt:**

> Draft a decision memo for PackageDelivery's management in `docs/decision_memo_fleetops.md`, at most 600 words. Structure: the recommendation per borough in one table; the key figures behind each verdict; the decision model and its assumptions; what the data does not tell us; how data quality issues were handled.
>
> Use only figures from my query results. The data quality section may describe the issues without figures. The verdicts follow the model; where the data questions an assumption of the model, say so under *what the data does not tell us*, but do not change the verdicts — management decides whether to change the model.
>
> My query results: [paste your query results]
>
> My interpretation: [paste your answers from Part C]

2) Let ADA check the draft against the results. **Example prompt:**

> Check the memo: list every figure in it and the query result it comes from, and check that every verdict follows D4 and D5. Report anything that does not match.

3) Review the draft yourself, as if you had to defend it in front of management:

| Check | Why |
|---|---|
| Every figure comes from your query results | An agent can produce plausible numbers. The memo may only contain numbers you can show |
| Every verdict follows D4 and D5. For example, Manhattan has a positive margin but less than 8 idle hours, so it is *Do not expand* | The model, not the writer, decides the verdict |
| The assumptions are named: utilisation 0.5, driver cost excluded, 8 idle hours per day | Management must be able to disagree with them |
| Bronx and Staten Island say **Insufficient evidence**, with the reason — not *no* | The difference matters for the next decision |
| The midday demand limitation is stated | It is the biggest uncertainty in the recommendation |
| Your NULL dropoff decision from Exercise 4 is stated | The figures depend on it |

Tell ADA what to change until every check passes.

4) Commit the memo. **Example prompt:**

> Commit only `docs/decision_memo_fleetops.md` to Git with the message "FLEETOPS decision memo".

#### ✅ Checkpoint D

Your memo:

- recommends **Expand** in Brooklyn, **Expand, airport-weighted** in Queens, and **Do not expand** in Manhattan
- marks Bronx and Staten Island as **Insufficient evidence**, and says why
- defines every metric it uses, and names the assumptions
- states the midday demand limitation
- states how the NULL dropoffs, the distance units and the unknown zones were handled
- contains only figures you can trace to a query
- is committed to Git

Compare the result with your hypotheses from Part A. Where were you wrong, and why?

---

### Summary of Exercise 6

You have:

- written hypotheses before looking at the results
- applied an agreed decision model with ADA, and checked every rule in the query
- interpreted the numbers, including what they cannot tell
- written a recommendation that separates *no* from *insufficient evidence*

---

## End of the course

Over six exercises you took a new source from raw files to a management decision — without writing the YAML, the loads or the queries yourself. What you did instead:

- **Specified** — business questions, rules and decisions, precise enough for an agent to act on
- **Reviewed** — designs, plans, SQL and numbers, and rejected them when they were wrong
- **Verified** — against row counts, known figures and your own hypotheses

ADA did the construction. The decisions, and the responsibility for them, stayed with you. That is the job of a data engineer working with an agent.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` |
| Taxi revenue per hour is far too high for one borough | Trips of 0 minutes or with negative duration are included. Check the 1–120 minute filter (D2) |
| The query shows a borough called `Unknown` or `EWR` | These come from the taxi zones 264, 265 and Newark Airport. They are not PackageDelivery boroughs — leave them out of the recommendation |
| Movements per borough differ from the reference | The movements are counted by home borough, not by the borough of the pickup zone (D5) |
| The memo contains a figure you cannot find in your results | Ask ADA where it comes from. If there is no source, remove it |
