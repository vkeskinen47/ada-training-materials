# Exercise 2 — Diagnose and Fix a Failed Load

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The task

Estimated duration: 45 minutes

At the end of Exercise 1, the DAG `FLEETOPS_<YOUR_ALIAS>` failed. Vehicles and shifts are in staging, but `STG_FLEETOPS_FLEET_MOVEMENT` is empty. Without movements, PackageDelivery cannot see where its fleet drives or where it stands idle — the decision cannot be made.

In this exercise you will find out what failed and why, decide how to handle it, and fix the load.

ℹ In real projects, a failed overnight load is a normal part of the job. The skill is not avoiding failures but diagnosing them quickly and fixing them in the right place.

---

### Part A — Find What Failed

Estimated duration: 15 minutes

Use either path, or both and compare.

#### Path 1: Workflow Orchestration

1) Open DEV Workflow Orchestration and find the DAG `FLEETOPS_<YOUR_ALIAS>`.

2) Open the latest run. Click the failed load (the red box) and press **Logs**.

3) Find the error message and the **QueryId**.

#### Path 2: The ada-ops agent

`ada-ops` is ADA's operations agent. It reads workflow runs, load statistics and errors from ADE. Select the `ada-ops` agent instead of `ada-developer`.

**Example prompt:**

> Why did the latest run of the DAG `FLEETOPS_<YOUR_ALIAS>` in dev fail? Show me which loads succeeded, which failed, and the full error message.

**Under the hood:** `ada workflow diagnose --env dev --dag-id FLEETOPS_<YOUR_ALIAS>`. It finds the failed run, lists the failed loads with their error messages, and fetches the generated SQL of the failing load. If you configured MCP in Exercise 1, ADA may query ADE through the MCP server instead.

#### ✅ Checkpoint A

- Two loads succeeded: `..._fleet_vehicle_...` and `..._fleet_shift_...`
- One load failed: `..._fleet_movement_...`
- The error message contains `Timestamp  is not recognized` and a QueryId

Notice the **two spaces** in `Timestamp  is not recognized`. ADE hides the data value from the workflow log, so you know *what kind* of value failed, but not *which* value. To see it, you need the target database.

---

### Part B — Find the Root Cause

Estimated duration: 15 minutes

1) In Snowflake, open **Monitoring → Query History**, search for the QueryId from Part A, and copy the full error message.

2) Paste the error to ADA and let it explain. **Example prompt** (to `ada-developer`):

> The load of `STG_FLEETOPS_FLEET_MOVEMENT` failed in Snowflake with this error: `<paste the error message>`. Explain what caused it: which value, line and column failed, and why. Why did we not see this problem when we inspected the sample file in Exercise 1?

**Under the hood:** ADA reads the error message, the entity YAML and the sample file.

3) The file has over 13,000 rows, and only one row is mentioned in the error. Is it one bad row or a pattern? Query the file in the landing zone directly. Snowflake can read staged files without loading them:

```sql
select $3 as vehicle_id, count(*) as row_count, min($6) as first_value
from '@staging.source_data_azure_stage_dev/FLEETOPS/PACKAGE_DELIVERY/fleet_movement.csv'
where $6 like '__.__.____%'
group by 1
order by 1;
```

`$3` and `$6` are the third and sixth columns of the file: `vehicle_id` and `movement_start_ts`.

4) Find out what these vehicles have in common. **Example prompt:**

> The query returned these rows: `<paste the query result>`. Look at the FLEETOPS sample file `fleet_vehicle.csv` and tell me what these vehicles have in common.

#### ✅ Checkpoint B

You can answer these questions:

| Question | Your answer |
|---|---|
| Which value failed, and on which line? | |
| How many rows have the problem, and which vehicles do they belong to? | |
| What do these vehicles have in common? | |
| From which date does the problem start? | |
| Why did ADA not see this problem in Exercise 1? | |

<details>
<summary>Check your answers</summary>

- The value `15.02.2017 06:57:00` on line 10446 in column `movement_start_ts`
- 223 rows, from vehicles `VEH-0022`, `VEH-0023` and `VEH-0034`
- All three are `VAN_EU` vehicles — vans imported from Europe
- From 15 February 2017. Before that date, the same vans wrote timestamps in the normal format. Something changed in their telematics on that day
- ADA inferred the column types from the **sample**, which covers only the first week of January. The problem did not exist yet

</details>

---

### Part C — Decide and Fix

Estimated duration: 15 minutes

#### Decide

There are several ways to handle this. Think about each before you read on:

| Option | What happens |
|---|---|
| A. Ask the source system owner to fix the export | The right long-term fix. But what do you do until then? |
| B. Set `ON_ERROR = CONTINUE` in the COPY INTO statement | The load succeeds. What happens to the 223 rows? |
| C. Convert both formats inside the COPY INTO statement | The staging table gets real timestamps right away |
| D. Load the two timestamp columns into staging as text, and convert them in the next layer | Staging shows exactly what the source sent |

<details>
<summary>Discussion</summary>

- **A** — do it anyway, but you still need a fix today.
- **B** — Snowflake skips the bad rows. The load is green and 223 movements are silently missing. That is worse than a failed load.
- **C** — tempting, but Snowflake does not handle conversion errors inside COPY INTO the way it does in a normal query. Expect a new error.
- **D** — the staging layer stays a faithful copy of the source, and the format rule (*EU vans write DD.MM.YYYY*) becomes an explicit, documented business rule in the next layer. **This is the fix you will implement.**

</details>

#### Fix

1) **Example prompt** (to `ada-developer`):

> In the staging entity `STG_FLEETOPS_FLEET_MOVEMENT`, change the data type of `movement_start_ts` and `movement_end_ts` to `VARCHAR(19)`. Do not add any conversion to the load. Then validate and show me the plan.

**Under the hood:** ADA edits `stg_fleetops_fleet_movement.yaml`, then runs `ada validate` and `ada plan`.

**Review:** the plan must show **one** modified entity, `STG_FLEETOPS_FLEET_MOVEMENT`, and nothing else.

2) **Example prompt:**

> Push the FLEETOPS staging package, commit it with the message "Stage movement timestamps as text" and wait for the deployment. Then trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

#### ✅ Checkpoint C

The DAG succeeds. Check the row counts:

```sql
select 'vehicle' as entity, count(*) as row_count, 42 as expected_count
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'shift', count(*), 1981
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'movement', count(*), 13871
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
union all
select 'movement, EU format', count(*), 223
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
where movement_start_ts like '__.__.____%';
```

The last row shows that the EU-format movements are there, as text.

ℹ **Why did vehicles and shifts not load twice?** The DAG ran all three loads again, but Snowflake remembers which files a table has already loaded and skips them. Only `fleet_movement.csv` was loaded this time, because its earlier load failed.

---

### Summary of Exercise 2

You have:

- diagnosed a failed load from the workflow log, and optionally through the `ada-ops` agent
- found the real root cause in the target database, and the pattern behind it
- weighed four fixes and chosen the one that keeps staging faithful to the source
- fixed and reloaded the data without duplicating anything

All three FLEETOPS files are now in staging. The timestamps of 223 movements are still in two formats. Remember this rule — it belongs to the next layers:

> **EU vans (`VAN_EU`) write timestamps as `DD.MM.YYYY HH24:MI:SS` from 15 February 2017.**

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` |
| `ada-ops` cannot reach ADE through MCP | The MCP servers are not running. In VS Code, open `.vscode/mcp.json` and start them. Or ask ADA to use `ada workflow diagnose` instead |
| The stage query in Part B fails with a permission error | Ask your trainer. You can skip the query and continue from Checkpoint B |
| The plan shows changes in other packages | Something else changed locally. Ask ADA to show the diff before you push |
| The DAG fails again with `Timestamp ... is not recognized` | The type change did not reach DEV. Check that the deployment succeeded, and that the plan showed the change |
