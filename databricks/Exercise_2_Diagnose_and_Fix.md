---
layout: exercise
module_label: "Module 1: Exercise 2"
title: "Diagnose and Fix a Failed Load"
duration: "45 minutes"
summary: "Find why the FLEETOPS movement load failed, identify the timestamp pattern, and fix staging without dropping records."
---

Databricks as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias.

> **Note:** In real projects, a failed overnight load is a normal part of the job. The skill is not avoiding failures but diagnosing them quickly and fixing them in the right place.
{: .exercise-note}

---

## Part A — Find What Failed

Estimated duration: 15 minutes

Use either path, or both and compare.

### Path 1: Workflow Orchestration

1) Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`.

2) Open the latest run. Click the failed load and press <span class="ui-action">Logs</span>.

3) Find the error message.

### Path 2: The ada-ops agent

`ada-ops` is ADA's operations agent. It reads workflow runs, load statistics and errors from ADE. Select the `ada-ops` agent instead of `ada-developer`.

**Example prompt:**

> Why did the latest run of the DAG `FLEETOPS_<YOUR_ALIAS>` in dev fail? Show me which loads succeeded, which failed, and the full error message.

**Under the hood:** `ada workflow diagnose --env dev --dag-id FLEETOPS_<YOUR_ALIAS>`. It finds the failed run, lists the failed loads with their error messages, and fetches the generated SQL of the failing load. If you configured MCP in Exercise 1, ADA may query ADE through the MCP server instead.

### Checkpoint A

- Two loads succeeded: `..._fleet_vehicle_...` and `..._fleet_shift_...`
- One load failed: `..._fleet_movement_...`
- The error message contains `[CAST_INVALID_INPUT]` and `cannot be cast to "TIMESTAMP"`

Notice the **two spaces** in `The value  of the type "STRING"`. ADE hides the data value from the workflow log, so you know *what kind* of value failed, but not *which* value. ADE runs the load with its own service account, so you cannot find the value in your own Databricks <span class="ui-action">Query History</span> either. To see the values, you read the source file itself.

---

## Part B — Find the Root Cause

Estimated duration: 15 minutes

1) Paste the error to ADA and let it explain. **Example prompt** (to `ada-developer`):

> The load of `STG_FLEETOPS_FLEET_MOVEMENT` failed in Databricks with this error: `<paste the error message from the workflow log>`. Explain what caused it: which kind of value failed, which columns of the load could be affected, and why. Why did we not see this problem when we inspected the sample file in Exercise 1?

**Under the hood:** ADA reads the error message, the entity YAML and the sample file.

2) The error does not tell which value failed, and the file has over 13,000 rows. Is it one bad row or a pattern? Query the file in the landing zone directly. Databricks can read files in cloud storage without loading them. This query lists the rows whose start or end time cannot be read as a timestamp, per vehicle:

```sql
select _c2 as vehicle_id, count(*) as row_count, min(_c5) as first_value
from read_files(
  'abfss://src@dlssrcdatadatahubdev.dfs.core.windows.net/FLEETOPS/PACKAGE_DELIVERY/fleet_movement.csv',
  format => 'csv', header => false)
where _c0 <> 'movement_id'
  and (try_cast(_c5 as timestamp) is null or try_cast(_c6 as timestamp) is null)
group by 1
order by 1;
```

`_c2`, `_c5` and `_c6` are the third, sixth and seventh columns of the file (Databricks counts from zero): `vehicle_id`, `movement_start_ts` and `movement_end_ts`. `try_cast` returns NULL where a plain `CAST` would fail, and the first condition skips the header row.

3) Find out what these vehicles have in common. **Example prompt:**

> The query returned these rows: `<paste the query result>`. Look at the FLEETOPS sample file `fleet_vehicle.csv` and tell me what these vehicles have in common.

### Checkpoint B

You can answer these questions:

| Question | Your answer |
|---|---|
| Which kind of value failed, and in which column? | |
| How many rows have the problem, and which vehicles do they belong to? | |
| What do these vehicles have in common? | |
| From which date does the problem start? | |
| Why did ADA not see this problem in Exercise 1? | |

<details markdown="1">
<summary markdown="span">Check your answers</summary>

- Timestamps in the format `DD.MM.YYYY HH24:MI:SS`, for example `15.02.2017 06:57:00`, in column `movement_start_ts` (`_c5` in the load statement). The error message only says that a value could not be cast to `TIMESTAMP`
- 223 rows, from vehicles `VEH-0022`, `VEH-0023` and `VEH-0034`
- All three are `VAN_EU` vehicles — vans imported from Europe
- From 15 February 2017. Before that date, the same vans wrote timestamps in the normal format. Something changed in their telematics on that day
- ADA inferred the column types from the **sample**, which covers only the first week of January. The problem did not exist yet

</details>

---

## Part C — Decide and Fix

Estimated duration: 15 minutes

### Decide

There are several ways to handle this. Think about each before you read on:

| Option | What happens |
|---|---|
| A. Ask the source system owner to fix the export | The right long-term fix. But what do you do until then? |
| B. Wrap the timestamp columns in `try_cast` in the COPY INTO statement | The load succeeds. What happens to the 223 rows? |
| C. Convert both formats inside the COPY INTO statement | The staging table gets real timestamps right away |
| D. Load the two timestamp columns into staging as text, and convert them in the next layer | Staging shows exactly what the source sent |

<details markdown="1">
<summary markdown="span">Discussion</summary>

- **A** — do it anyway, but you still need a fix today.
- **B** — `try_cast` returns NULL for a value it cannot convert. The load is green and 223 movements have no start time. That is worse than a failed load.
- **C** — works: `try_to_timestamp` with each of the two formats and `coalesce` parses all rows. But you must replace the generated column list of the load with hand-written SQL, and the format rule ends up in a staging load, where nobody looks for business rules. A source that sends a third format would silently produce NULLs.
- **D** — the staging layer stays a faithful copy of the source, and the format rule (*EU vans write DD.MM.YYYY*) becomes an explicit, documented business rule in the next layer. **This is the fix you will implement.**

</details>

### Fix

1) **Example prompt** (to `ada-developer`):

> In the staging entity `STG_FLEETOPS_FLEET_MOVEMENT`, change the data type of `movement_start_ts` and `movement_end_ts` to `VARCHAR(19)`. Do not add any conversion to the load. Then validate and show me the plan.

**Under the hood:** ADA edits `stg_fleetops_fleet_movement.yaml`, then runs `ada validate` and `ada plan`.

**Review:** the plan must show **one** modified entity, `STG_FLEETOPS_FLEET_MOVEMENT`, and nothing else.

2) **Example prompt:**

> Push the FLEETOPS staging package, commit it with the message "Stage movement timestamps as text" and wait for the deployment. Then trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

### Checkpoint C

The DAG succeeds. Check the row counts:

```sql
select 'vehicle' as entity, count(*) as row_count, 42 as expected_count
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'shift', count(*), 1981
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'movement', count(*), 13871
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
union all
select 'movement, EU format', count(*), 223
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
where movement_start_ts like '__.__.____%';
```

The last row shows that the EU-format movements are there, as text.

> **Why did vehicles and shifts not load twice?** The DAG ran all three loads again, but Databricks remembers which files a table has already loaded and skips them. Only `fleet_movement.csv` was loaded this time, because its earlier load failed.
{: .exercise-note}

---

## Summary of Exercise 2

You have:

- diagnosed a failed load from the workflow log, and optionally through the `ada-ops` agent
- found the real root cause in the target database, and the pattern behind it
- weighed four fixes and chosen the one that keeps staging faithful to the source
- fixed and reloaded the data without duplicating anything

All three FLEETOPS files are now in staging. The timestamps of 223 movements are still in two formats. Remember this rule — it belongs to the next layers:

> **EU vans (`VAN_EU`) write timestamps as `DD.MM.YYYY HH24:MI:SS` from 15 February 2017.**

---

Something not working? Ask ADA to help you debug it first, then ask your trainer.
