---
layout: exercise
module_label: "Module 1: Exercise 2"
title: "Diagnose and Fix a Failed Load"
duration: "45 minutes"
summary: "Find why the FLEETOPS movement load failed, identify the timestamp pattern, and fix staging without dropping records."
---

BigQuery as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias. Use your alias in lowercase.

> **Note:** In real projects, a failed overnight load is a normal part of the job. The skill is not avoiding failures but diagnosing them quickly and fixing them in the right place.
{: .exercise-note}

---

## Find What Failed

Estimated duration: 15 minutes

Use either path, or both and compare.

### Path 1: Workflow Orchestration

1) Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`.

2) Open the latest run. Click the failed load and press <span class="ui-action">Logs</span>.

3) Find the error message and the **JobID**.

### Path 2: The ada-ops agent

`ada-ops` is ADA's operations agent. It reads workflow runs, load statistics and errors from ADE. Select the `ada-ops` agent instead of `ada-developer`.

**Example prompt:**

> Why did the latest run of the DAG `FLEETOPS_<YOUR_ALIAS>` in dev fail? Show me which loads succeeded, which failed, and the full error message.

**Under the hood:** `ada workflow diagnose --env dev --dag-id FLEETOPS_<YOUR_ALIAS>`. It finds the failed run, lists the failed loads with their error messages, and tries to fetch the generated SQL of the failing load.

> **Note:** Under `SQL:` you may see `403 Forbidden … code/v1/preview`. Your key is not allowed to read the generated SQL. Ignore it. The error message above it is what you need.
{: .exercise-note}

### Checkpoint A

- Two staging loads succeeded: `..._fleet_vehicle_...` and `..._fleet_shift_...`, and their PSA loads too
- One load failed: the staging load `..._fleet_movement_...`. Its PSA load did not run
- The error message contains `Invalid datetime string`, a line number, `column_name: "movement_start_ts"`, `column_type: DATETIME`, the file `fleet_movement.csv` and a JobID

Read the message carefully. BigQuery tells you **which value** failed, **on which line** and **in which column**. That is the first bad row it met. It does not tell you how many more there are.

---

## Find the Root Cause

Estimated duration: 15 minutes

1) Paste the error to ADA and let it explain. **Example prompt** (to `ada-developer`):

> The load of `STG_FLEETOPS_FLEET_MOVEMENT` failed in BigQuery with this error: `<paste the error message from the workflow log>`. Explain what caused it: which value, line and column failed, and why. Why did we not see this problem when we inspected the sample file in Exercise 1?

**Under the hood:** ADA reads the error message, the entity YAML, the load SQL and the sample file.

2) The file has over 13,000 rows, and only the first bad row is mentioned in the error. Is it one bad row or a pattern? Query the file in the landing zone directly. Open <span class="ui-action">BigQuery Studio</span>, open a new SQL query and run this as one script. `LOAD DATA INTO TEMP TABLE` reads the file into a table that exists only during the script. It creates nothing in your datasets and nothing in ADE:

```sql
LOAD DATA INTO TEMP TABLE raw_movement (
  movement_id STRING, shift_id STRING, vehicle_id STRING,
  pickup_locationid STRING, dropoff_locationid STRING,
  movement_start_ts STRING, movement_end_ts STRING,
  distance STRING, distance_unit STRING, stop_count STRING,
  package_count STRING, delivery_revenue_usd STRING,
  empty_leg_flag STRING, created STRING, updated STRING
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://ade-training-sourcedata/src/fleetops/package_delivery/fleet_movement.csv'],
  skip_leading_rows = 1
);

SELECT vehicle_id, COUNT(*) AS row_count, MIN(movement_start_ts) AS first_value
FROM raw_movement
WHERE SAFE_CAST(movement_start_ts AS DATETIME) IS NULL
   OR SAFE_CAST(movement_end_ts AS DATETIME) IS NULL
GROUP BY vehicle_id
ORDER BY vehicle_id;
```

All columns are read as text, so the file cannot fail. `SAFE_CAST` returns NULL where a plain `CAST` would fail, so the `WHERE` clause lists the rows that cannot be read as a timestamp.

3) Find out what these vehicles have in common. **Example prompt:**

> The query returned these rows: `<paste the query result>`. Look at the FLEETOPS sample file `fleet_vehicle.csv` and tell me what these vehicles have in common.

### Checkpoint B

You can answer these questions:

| Question | Your answer |
|---|---|
| Which value failed, and on which line? | |
| How many rows have the problem, and which vehicles do they belong to? | |
| What do these vehicles have in common? | |
| From which date does the problem start? | |
| Why did ADA not see this problem in Exercise 1? | |

<details markdown="1">
<summary markdown="span">Check your answers</summary>

- The value `15.02.2017 06:57:00` on line 10446 in column `movement_start_ts`
- 223 rows, from vehicles `VEH-0022`, `VEH-0023` and `VEH-0034`
- All three are `VAN_EU` vehicles: vans imported from Europe
- From 15 February 2017. Before that date, the same vans wrote timestamps in the normal format. Something changed in their telematics on that day
- ADA inferred the column types from the **sample**, which covers only the first week of January. The problem did not exist yet

</details>

---

## Decide and Fix

Estimated duration: 15 minutes

### Decide

There are several ways to handle this. Think about each before you read on:

| Option | What happens |
|---|---|
| A. Ask the source system owner to fix the export | The right long-term fix. But what do you do until then? |
| B. Add `max_bad_records` to the `LOAD DATA` statement | The load succeeds. What happens to the 223 rows? |
| C. Convert both formats inside the `LOAD DATA` statement | The staging table gets real timestamps right away |
| D. Load the two timestamp columns into staging as text, and convert them in the next layer | Staging shows exactly what the source sends |

<details markdown="1">
<summary markdown="span">Discussion</summary>

- **A:** do it anyway, but you still need a fix today.
- **B:** BigQuery skips the bad rows and the job ends green with a warning. 13,648 rows are loaded and 223 movements are silently missing. That is worse than a failed load.
- **C:** not possible. `LOAD DATA` only reads the columns of the file into the types you list. It has no place for a conversion. You would have to load the text first, which is option D.
- **D:** the staging layer stays a faithful copy of the source, and the format rule (*EU vans write DD.MM.YYYY*) becomes an explicit, documented business rule in the next layer. **This is the fix you will implement.**

</details>

### Fix

1) **Example prompt** (to `ada-developer`):

> In the staging entity `STG_FLEETOPS_FLEET_MOVEMENT`, change the data type of `movement_start_ts` and `movement_end_ts` to `VARCHAR(19)`. Do not add any conversion to the load. Find every other file that must change for the load to work with these two columns, and change it too. Then validate and show me the plan.

**Under the hood:** ADA edits `stg_fleetops_fleet_movement.yaml`, the `LOAD DATA` column list in `stg_fleetops_fleet_movement.sql` (`DATETIME` becomes `STRING`) and the PSA entity `psa_fleetops_fleet_movement.yaml`, then runs `ada validate` and `ada plan`.

**Review:** the plan must show **two** modified entities, `STG_FLEETOPS_FLEET_MOVEMENT` and `PSA_FLEETOPS_FLEET_MOVEMENT`, and nothing else. Both show the type change of the two columns. The staging entity also shows a changed `load_data` step.

> **Why three files?** The PSA entity copies the business columns of the staging entity, and its load reads them from staging. If it kept `TIMESTAMP`, it would no longer match the staging columns, and the PSA load would break. The `LOAD DATA` column list is hand-written SQL: a type change in the YAML does not reach it by itself. If your plan shows only one modified entity, ask ADA which other files use these columns.
{: .exercise-note}

> **Note:** The movement tables are empty, because a failed `LOAD DATA` loads nothing. That makes a type change safe here. On a table that holds data, ask ADA about the effect of a type change before you push.
{: .exercise-note}

2) **Example prompt:**

> Push the FLEETOPS staging package. Commit it in ADE with the message "Stage movement timestamps as text", promote it to DEV, deploy it and wait for the deployment. Then trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

**Under the hood:** `ada push -p STG_FLEETOPS_<YOUR_ALIAS>`, `ada deploy commit`, `ada deploy promote --to DEV`, `ada deploy start --to DEV --wait`, then `ada dag trigger --dag-id FLEETOPS_<YOUR_ALIAS> --env dev --wait`

### Checkpoint C

The DAG succeeds. Check the row counts in BigQuery Studio:

```sql
select 'stg vehicle' as entity, count(*) as row_count, 42 as expected_count
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'stg shift', count(*), 1981
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'stg movement', count(*), 13871
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
union all
select 'stg movement, EU format', count(*), 223
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
where movement_start_ts like '__.__.____%'
union all
select 'psa vehicle', count(*), 42
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_VEHICLE
union all
select 'psa shift', count(*), 1981
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_SHIFT
union all
select 'psa movement', count(*), 13871
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_MOVEMENT;
```

The `EU format` row shows that the EU-format movements are there, as text.

> **Why did vehicles and shifts not load twice?** The DAG ran all six loads again. Each staging load first truncates its table and then reads its file, so staging holds one copy of each file. The PSA loads add only rows that are new or have changed, and nothing had changed in vehicle and shift.
{: .exercise-note}

---

## Summary of Exercise 2

You have:

- diagnosed a failed load from the workflow log, and optionally through the `ada-ops` agent
- found the pattern behind a single reported value by querying the file in the landing zone
- weighed four fixes and chosen the one that keeps staging faithful to the source
- followed a type change through the staging entity, the PSA entity and the hand-written load SQL
- fixed and reloaded the data without duplicating anything

All three FLEETOPS files are now in staging and PSA. The timestamps of 223 movements are still in two formats. Remember this rule, because it belongs to the next layers:

> **EU vans (`VAN_EU`) write timestamps as `DD.MM.YYYY HH24:MI:SS` from 15 February 2017.**

---

Something not working? Ask ADA to help you debug it first, then check the [Training Q&A](Training_QA_ADA_Self_Study.md) before you ask your trainer.
