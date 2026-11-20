---
layout: exercise
module_label: "Module 3: Exercise 5"
title: "Publish Layer and Business Rules"
duration: "1 hour"
summary: "Define business rules for fleet capacity, implement them in publish entities, and verify the metrics against reference figures."
---

Databricks as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The business rules

PackageDelivery's operations team gives you these rules. They turn the raw fleet data into the terms of the business question, and the verification at the end of this exercise checks each of them.

- **R1** Movement timestamps are `YYYY-MM-DD HH24:MI:SS`. Exception: EU vans write `DD.MM.YYYY HH24:MI:SS` from 15 February 2017.
- **R2** All distances are reported in miles. Rows with `distance_unit = 'km'` are converted: 1 km = 0.621371 miles.
- **R3** A vehicle is taxi-capable if it has at least 4 passenger seats.
- **R4** Idle time is the time within a shift, from 10:00 to 15:00, that is not covered by any movement. Empty legs are movements too: a vehicle driving empty is not idle.
- **R5** Fleet capacity belongs to the vehicle's home borough. Taxi demand belongs to the borough of the taxi pickup zone. Both use the borough names in `D_TAXI_ZONE`.

Why 10:00–15:00: deliveries peak in the morning and the afternoon, so the midday window is when vehicles could be released.

---

## Design the Publish Layer

Estimated duration: 15 minutes

1) **Example prompt** (to `ada-developer`):

> Design a publish layer for the FLEETOPS Data Vault so that PackageDelivery can compare idle, taxi-capable fleet capacity per borough with taxi demand. Implement these business rules: [paste the rules R1–R5]
>
> Design exactly three entities:
> - `D_FLEET_VEHICLE`: one row per vehicle, with the vehicle type, the passenger seats, taxi capability (R3), the home borough (R5) and the cost per mile
> - `F_FLEET_MOVEMENT`: one row per movement, with the start and end time as timestamps (R1), the distance in miles (R2), and the pickup and dropoff zones as references to the existing `D_TAXI_ZONE` in `packages/traffic_publish_<your_alias>/`, with the same key as `F_TRIP` uses
> - `F_SHIFT_IDLE`: one row per shift, with the shift date, the vehicle and the idle time between 10:00 and 15:00 (R4), calculated from the movement times in `F_FLEET_MOVEMENT`
>
> Each rule lives in exactly one place. If a rule is already implemented in an earlier layer, or conflicts with my Data Vault design, tell me before you design. Keep it simple: no other entities, no exception rows, no extra tests or local test scripts, and no code preview. Ask me only if a rule cannot be implemented as written. Write the design to `design/publish_design_fleetops.md`: for each entity, its grain, its source entities, its measures and attributes, and which rule it implements. Do not generate any entity YAML yet.

**Under the hood:** ADA uses its publish generation skill. It reads the Data Vault packages, the existing publish package and `config/pipeline.yaml`.

2) Review the design:

| Check | Why |
|---|---|
| The design has the three entities and nothing else | Every extra entity is more to build, deploy and verify. Ask ADA to remove what you did not ask for |
| Every entity states its **grain** in one sentence | Without a grain, nobody can tell whether a sum is right |
| Every rule R1–R5 is implemented **exactly once**, and the design says where | A rule implemented twice will one day be implemented two different ways |
| Zone references point to the existing `D_TAXI_ZONE`, using the same key as `F_TRIP` | Fleet and taxi figures must be comparable per zone and per borough. A second zone dimension breaks that |
| The design says what happens to movements whose zone is not in `D_TAXI_ZONE`, such as an unknown zone from Exercise 4 | Otherwise they disappear silently in every join |
| If a rule changed an earlier decision, the Data Vault design document is updated too | The two documents must not contradict each other |

Tell ADA what to change. Repeat until every check passes.

### Checkpoint A

- `design/publish_design_fleetops.md` exists, with the three entities and the grain of each
- Every rule R1–R5 is mapped to exactly one load
- Every open question is answered or left open with a reason
- The design reuses `D_TAXI_ZONE`. No new zone dimension

---

## Generate, Deploy and Run

Estimated duration: 20 minutes

1) **Example prompt:**

> Generate the three publish entities from `design/publish_design_fleetops.md`. Keep to the design: no other entities, steps or tests, and no code preview. If an open question in the design blocks the generation, stop and ask me. Validate and show me the plan.

**Under the hood:** ADA writes the entity YAML and an SQL file for each load under `packages/`, and runs `ada validate` and `ada plan`.

> **Note:** `ada validate` and `ada plan` check the structure, not whether the SQL calculates the right thing. That is what step 3 and the verification are for.
{: .exercise-note}

2) Review the plan. It must show `+` only for the three new publish entities, and no changes to `F_TRIP`, `D_TAXI_ZONE` or any Data Vault entity.

3) Read the SQL of the idle time load. **Example prompt:**

> Read the local SQL sidecar for the idle-time load and explain its calculation step by step in plain language. Which shifts and movements does it read, how does it limit them to 10:00–15:00, and which timestamp rule does it use?

> **Note:** You do not need to write this SQL yourself. You need to understand it well enough to tell whether it implements rule R4. If ADA's explanation and the rule disagree, the SQL is wrong.
{: .exercise-note}

4) Push, commit and deploy. **Example prompt:**

> Push the publish package. Commit it in ADE with the message "FLEETOPS publish layer", promote it to DEV, deploy it and wait for the deployment.

**Under the hood:** `ada push`, then `ada deploy commit`, `ada deploy promote --to DEV` and `ada deploy start --to DEV --wait`.

5) Open DEV <span class="ui-action">Workflow Orchestration</span> and the DAG `FLEETOPS_<YOUR_ALIAS>` in the <span class="ui-action">Graph view</span>, and check that the three publish loads are there. If they are not there yet, wait a minute and refresh. A run that starts before ADE has updated the DAG is green, but it does not run the new loads and the publish tables stay empty.

6) Trigger the DAG. **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

Then check in Workflow Orchestration that the three publish loads ran and succeeded.

### Checkpoint B

- The plan shows only the three new publish entities
- You can explain in one sentence how the idle time is calculated
- The deployment and the DAG succeed, and every new publish load ran in the DAG

---

## Verify Against Known Figures

Estimated duration: 25 minutes

The operations team has calculated a few figures by hand. Your publish layer must reproduce them.

1) Row counts. The expected counts are the row counts of the full FLEETOPS files in staging (the sample files in your project are smaller). **Example prompt:**

> Write a short Databricks SQL query that returns `entity`, `row_count` and `expected_count` for my new publish entities in `db_datahub_dev.publish_<your_alias>`, one `union all` line per entity. Expected counts: one row per vehicle 42, per shift 1981, per movement 13871. Skip the grains my design does not have.

2) The business figures. **Example prompt:**

> Write a Databricks SQL query on my publish layer that returns, per home borough: `fleet_mph` (total miles divided by total movement hours, all vehicles) and `idle_capable_hours_per_day` (idle hours of taxi-capable vehicles divided by the number of distinct shift dates of all the borough's vehicles). Round both to one decimal.

Run both queries in Databricks and compare:

| Borough | `fleet_mph` | `idle_capable_hours_per_day` |
|---|---|---|
| Manhattan | 7.5 | 2.3 |
| Brooklyn | 11.0 | 15.8 |
| Queens | 14.0 | 13.2 |
| Bronx | 11.0 | 1.2 |
| Staten Island | 15.9 | 0.0 |

A difference of 0.1 is rounding. A bigger difference means a rule is wrong or missing. Paste your result to ADA and ask which rule could explain it.

<details markdown="1">
<summary markdown="span">Typical differences and their cause</summary>

| Symptom | Likely cause |
|---|---|
| Brooklyn `fleet_mph` about 12.0, Queens about 14.8 | Kilometres were not converted (R2). The EU vans are based in Brooklyn and Queens |
| Idle hours too high in Brooklyn and Queens | EU-format timestamps became NULL (R1), so their movements do not cover any time |
| Idle hours far too high everywhere | The movements are not limited to 10:00–15:00, or idle time is counted outside shifts (R4) |
| Manhattan or Bronx idle hours too high | All vehicles were counted, not only the taxi-capable ones (R3) |

</details>

### Checkpoint C

- Every row count matches its expected count
- Every borough matches the table within 0.1
- If you fixed something, the fix is pushed, deployed and the DAG rerun

---

## Summary of Exercise 5

You have:

- turned business rules into a reviewed publish design, with each rule implemented exactly once
- reused the existing taxi zone dimension, so fleet and taxi figures are comparable
- verified the result against figures calculated independently, and traced differences back to rules

---

## End of the course

Over five exercises you took a new source from raw files to verified business figures, without writing the YAML, the loads or the SQL yourself. What you did instead:

- **Specified** business questions, rules and decisions, precise enough for an agent to act on
- **Reviewed** designs, plans, SQL and numbers, and rejected them when they were wrong
- **Verified** the results against row counts and known figures

ADA did the construction. The decisions, and the responsibility for them, stayed with you. That is the job of a data engineer working with an agent.

**Want to finish the business story?** The optional [Exercise 6](Exercise_6_Analysis_and_Decision.md) applies PackageDelivery's decision model to your publish layer and ends with a recommendation memo for management.

---

Something not working? Ask ADA to help you debug it first, then check the [Training Q&A](Training_QA_ADA_Self_Study.md) before you ask your trainer.
