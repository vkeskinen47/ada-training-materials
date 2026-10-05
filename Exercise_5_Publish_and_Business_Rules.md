# Exercise 5 — Publish Layer and Business Rules

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The task

Estimated duration: 1 hour 30 minutes

The Raw Data Vault holds the fleet data as the source sent it: timestamps in two formats, distances in two units, and no idea of what *idle* means. PackageDelivery's question cannot be answered from that yet.

In this exercise you turn business rules into a publish layer: facts and dimensions that answer the question. You state the rules, ADA designs and generates the entities, and you verify the result against known figures.

**Result of this exercise:** fleet facts and a vehicle dimension deployed to DEV and loaded, with the business rules implemented in their loads, and connected to the existing taxi zone dimension.

---

### The business rules

PackageDelivery's operations team gives you these rules:

- **R1** Movement timestamps are `YYYY-MM-DD HH24:MI:SS`. Exception: EU vans write `DD.MM.YYYY HH24:MI:SS` from 15 February 2017.
- **R2** All distances are reported in miles. Rows with `distance_unit = 'km'` are converted: 1 km = 0.621371 miles.
- **R3** A vehicle is taxi-capable if it has at least 4 passenger seats.
- **R4** Idle time is the time within a shift, from 10:00 to 15:00, that is not covered by any movement. Empty legs are movements too: a vehicle driving empty is not idle.
- **R5** Fleet capacity belongs to the vehicle's home borough. Taxi demand belongs to the borough of the taxi pickup zone. Both use the borough names in `D_TAXI_ZONE`.

Why 10:00–15:00: deliveries peak in the morning and the afternoon, so the midday window is when vehicles could be released.

---

### Part A — Design the Publish Layer

Estimated duration: 20 minutes

1) **Example prompt** (to `ada-developer`):

> Design a publish layer for the FLEETOPS Data Vault so that PackageDelivery can compare idle, taxi-capable fleet capacity per borough with taxi demand. Implement these business rules: [paste the rules R1–R5]
>
> If a rule is already implemented in an earlier layer, or conflicts with my Data Vault design, tell me before you design. Each rule must live in exactly one place. Reuse the existing `D_TAXI_ZONE` in `packages/traffic_publish_<your_alias>/` for all zone references, with the same key as `F_TRIP` uses. Store idle time with one row per shift, including the shift date and the vehicle. Write the design to `design/publish_design_fleetops.md`: for each fact and dimension, its grain, its source entities, its measures and attributes, and which rule is implemented where. Do not generate any entity YAML yet.

**Under the hood:** ADA uses its publish generation skill. It reads the Data Vault packages, the existing publish package and `config/pipeline.yaml`.

2) Review the design:

| Check | Why |
|---|---|
| Every fact states its **grain** in one sentence, for example *one row per movement* | Without a grain, nobody can tell whether a sum is right |
| Every rule R1–R5 is implemented **exactly once**, and the design says in which layer | A rule implemented twice will one day be implemented two different ways |
| Zone references point to the existing `D_TAXI_ZONE`, using the same key as `F_TRIP` | Fleet and taxi figures must be comparable per zone and per borough. A second zone dimension breaks that |
| The design says what happens to movements whose zone is not in `D_TAXI_ZONE`, such as an unknown zone from Exercise 4 | Otherwise they disappear silently in every join |
| Descriptive attributes, such as the vehicle type or the home borough, are in a dimension, not in a fact | The fact holds measures and keys |
| Idle time (R4) is stored with one row per shift | This is the key figure of the decision, and Part C checks it per borough and per day |
| If a rule changed an earlier decision, the Data Vault design document is updated too | The two documents must not contradict each other |

Tell ADA what to change. Repeat until every check passes.

#### ✅ Checkpoint A

- `design/publish_design_fleetops.md` exists and states the grain of every fact
- Every rule R1–R5 is mapped to exactly one load
- Every open question is answered or left open with a reason
- The design reuses `D_TAXI_ZONE`. No new zone dimension
- You have committed the design document to Git

---

### Part B — Generate, Deploy and Run

Estimated duration: 30 minutes

1) **Example prompt:**

> Generate the publish entities from `design/publish_design_fleetops.md`. If an open question in the design blocks the generation, stop and ask me. Validate and show me the plan.

**Under the hood:** ADA writes the entity YAML and an SQL file for each load under `packages/`, and runs `ada validate` and `ada plan`.

ℹ `ada validate` and `ada plan` check the structure, not whether the SQL calculates the right thing. That is what step 3 and Part C are for.

2) Review the plan. It must show `+` only for the new publish entities, and no changes to `F_TRIP` or any Data Vault entity. `D_TAXI_ZONE` may change only if your design decided so.

3) Read the SQL of the idle time load. **Example prompt:**

> Explain the SQL that calculates idle time, step by step, in plain language. Which shifts and movements does it read, how does it limit them to 10:00–15:00, and which timestamp rule does it use?

ℹ You do not need to write this SQL yourself. You need to understand it well enough to tell whether it implements rule R4. If ADA's explanation and the rule disagree, the SQL is wrong.

4) Push and check the code. **Example prompt:**

> Push the publish package with a dry run first. Then show me the code preview of the idle time load. Do not commit yet.

**Under the hood:** `ada push --dry-run`, `ada push`, then `ada code preview` against the design environment (no `--env`).

5) When the preview matches ADA's explanation, deploy and run. **Example prompt:**

> Commit the publish package in ADE with the message "FLEETOPS publish layer" and wait for the deployment. List the loads of the DAG `FLEETOPS_<YOUR_ALIAS>` in dev, and tell me whether the new publish loads are there. Then trigger the DAG, wait for it to finish, and show me which publish loads ran and whether they succeeded.

**Under the hood:** `ada deploy commit ... --wait`, `ada workflow describe --env dev --dag-id FLEETOPS_<YOUR_ALIAS>`, `ada dag trigger ... --wait`, and the load statistics of the run.

#### ✅ Checkpoint B

- The plan shows only new publish entities
- You can explain in one sentence how the idle time is calculated
- The deployment and the DAG succeed, and every new publish load ran in the DAG

---

### Part C — Verify Against Known Figures

Estimated duration: 25 minutes

The operations team has calculated a few figures by hand. Your publish layer must reproduce them.

1) Row counts. The expected counts are the row counts of the full FLEETOPS files in staging (the sample files in your project are smaller). **Example prompt:**

> Write a short Snowflake query that returns `entity`, `row_count` and `expected_count` for my new publish entities in `ADE_TRAINING_DEV_SAAS.PUBLISH_<YOUR_ALIAS>`, one `union all` line per entity. Expected counts: one row per vehicle 42, per shift 1981, per movement 13871. Skip the grains my design does not have.

2) The business figures. **Example prompt:**

> Write a Snowflake query on my publish layer that returns, per home borough: `fleet_mph` (total miles divided by total movement hours, all vehicles) and `idle_capable_hours_per_day` (idle hours of taxi-capable vehicles divided by the number of distinct shift dates of all the borough's vehicles). Round both to one decimal.

Run both queries in Snowflake and compare:

| Borough | `fleet_mph` | `idle_capable_hours_per_day` |
|---|---|---|
| Manhattan | 7.5 | 2.3 |
| Brooklyn | 11.0 | 15.8 |
| Queens | 14.0 | 13.2 |
| Bronx | 11.0 | 1.2 |
| Staten Island | 15.9 | 0.0 |

A difference of 0.1 is rounding. A bigger difference means a rule is wrong or missing. Paste your result to ADA and ask which rule could explain it.

<details>
<summary>Typical differences and their cause</summary>

| Symptom | Likely cause |
|---|---|
| Brooklyn `fleet_mph` about 12.0, Queens about 14.8 | Kilometres were not converted (R2). The EU vans are based in Brooklyn and Queens |
| Idle hours too high in Brooklyn and Queens | EU-format timestamps became NULL (R1), so their movements do not cover any time |
| Idle hours far too high everywhere | The movements are not limited to 10:00–15:00, or idle time is counted outside shifts (R4) |
| Manhattan or Bronx idle hours too high | All vehicles were counted, not only the taxi-capable ones (R3) |

</details>

#### ✅ Checkpoint C

- Every row count matches its expected count
- Every borough matches the table within 0.1
- If you fixed something, the fix is pushed, deployed and the DAG rerun

---

### Part D — Where Does the Business Logic Live?

Estimated duration: 15 minutes

You have implemented the rules R1–R5 inside publish loads. It works, and the decision can be made. But think about what this choice costs.

Ask yourself, or discuss with ADA:

- A second fact needs the harmonised distance. Where is rule R2 then?
- The operations team changes the idle window to 11:00–15:00 in June. Can you still show what the idle hours were in March under the old rule?
- An auditor asks why Brooklyn's idle hours were 15.8 in the report of 1 March. What can you show?

<details>
<summary>Discussion</summary>

Business logic in publish loads is **recomputed on every run** and **duplicated in every load that needs it**. When a rule changes, history is recalculated with the new rule, and the old figures cannot be reproduced.

A **Business Data Vault** solves this: the rules are computed once into historised business satellites, and every fact reads them. The rule result is stored and auditable, like any other data in the vault. This course uses the publish layer to keep the scope manageable. In real projects, put rules that are shared, that change over time, or that need auditing into a Business Data Vault.

</details>

Finally, commit your design documents. The publish package was committed to Git when you pushed it. **Example prompt:**

> Commit only the files in `design/` to Git with the message "FLEETOPS publish layer verified against reference figures".

---

### Part E — Semantic View *(optional)*

Estimated duration: 20 minutes

A semantic view describes your facts, dimensions and metrics so that people and AI tools can query them in business terms.

1) **Example prompt:**

> Generate a Snowflake semantic view on my FLEETOPS publish layer, in the schema `publish_<your_alias>`. Include two metrics per home borough:
> - `idle_capable_hours_per_day`: idle hours of taxi-capable vehicles (at least 4 passenger seats), divided by the number of distinct shift dates of all the borough's vehicles. Idle time is the time within a shift, from 10:00 to 15:00, not covered by any movement
> - `fleet_mph`: total miles divided by total movement hours of all the borough's vehicles, with kilometres converted to miles
>
> Stop and show me if a metric cannot be defined as stated. Validate and show me the plan.

**Under the hood:** ADA uses its semantic layer generation skill. It reads the publish packages, follows the references from facts to dimensions, and writes an entity with the physical type `SEMANTIC_VIEW`. It adds a schema override for the new entity type to `config/pipeline.yaml`.

2) Review each metric definition against its rules. Then deploy. **Example prompt:**

> Push the semantic view package with a dry run first, then commit it in ADE with the message "FLEETOPS semantic view" and wait for the deployment.

---

### Summary of Exercise 5

You have:

- turned business rules into a reviewed publish design, with each rule implemented exactly once
- reused the existing taxi zone dimension, so fleet and taxi figures are comparable
- verified the result against figures calculated independently, and traced differences back to rules
- considered what it costs to keep business logic in publish loads

You now have everything needed to answer PackageDelivery's question. In the next exercise, you will make the recommendation.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` |
| The design creates a new zone dimension | Point ADA to `packages/traffic_publish_<your_alias>/d_taxi_zone.yaml` |
| The plan shows changes to `F_TRIP`, or to `D_TAXI_ZONE` although your design did not decide so | ADA modified the existing taxi entities. Ask it to revert them and only reference them |
| The publish tables are empty after the DAG | The publish loads did not run in this DAG. Ask ADA to list the loads of `FLEETOPS_<YOUR_ALIAS>` (`ada workflow describe`), or check the Graph view |
| Fleet movements disappear when joined to `D_TAXI_ZONE` | Some zone keys are not in the dimension, for example an unknown zone from Exercise 4. Decide whether they should be, and record it in the design |
