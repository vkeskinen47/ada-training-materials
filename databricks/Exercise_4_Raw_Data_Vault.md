---
layout: exercise
module_label: "Module 2: Exercise 4"
title: "Generate and Load the Raw Data Vault"
duration: "1 hour 15 minutes"
summary: "Generate and load the FLEETOPS Raw Data Vault, extend the shared H_TAXI_ZONE hub, and verify that fleet and taxi data meet on the same zones."
---

Databricks as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## Measure the Starting Point

Estimated duration: 10 minutes

Before you change a shared hub, measure it. Otherwise you cannot tell what your change did.

1) Count the rows in your `H_TAXI_ZONE`. Run in Databricks (SQL editor):

```sql
select count(*) as row_count, 269 as expected_count
from db_datahub_dev.rdv_<your_alias>.H_TAXI_ZONE;
```

In basic training, the check query returned 266. Why does the hub have more rows? Find the zones that have no description:

```sql
select h.taxi_zone_key
from db_datahub_dev.rdv_<your_alias>.H_TAXI_ZONE h
left join db_datahub_dev.rdv_<your_alias>.S_TAXI_ZONE_C s on h.dv_id = s.dv_id
where s.dv_id is null
order by 1;
```

2) Find out where the hub gets its rows from. **Example prompt** (to `ada-developer`):

> How many loads does the entity `H_TAXI_ZONE` in `packages/traffic_dv_<your_alias>/` have, and from which source entities and columns? Do not change anything.

These loads are the pattern you extend in the next section: the FLEETOPS zone columns get loads of the same kind.

### Checkpoint A

| Question | Your answer |
|---|---|
| How many rows does `H_TAXI_ZONE` have? | |
| Which zone keys have no description, and where do they come from? | |
| How many loads does `H_TAXI_ZONE` have? | |

<details markdown="1">
<summary markdown="span">Check your answers</summary>

- 269 rows
- Keys `267`, `268` and `269`. They appear in the taxi trip data but in neither zone lookup file, so the hub has them but the satellite does not. The basic training check joined the hub to the satellite, which hid them
- Five loads: the zone lookup, and pickup and dropoff zones from both yellow and green trips

</details>

> **Note:** Write down 269. You will check it again at the end of this exercise.
{: .exercise-note}

---

## Generate the Entities

Estimated duration: 30 minutes

1) **Example prompt:**

> Generate the Data Vault entities from the approved design `design/dv_design_<name>.md`. For the zones, do not create a new entity: add the FLEETOPS loads to the existing `H_TAXI_ZONE` in `packages/traffic_dv_<your_alias>/h_taxi_zone.yaml`, and name them like the existing loads in that file. Use ADE's standard key transformations, no custom formulas. If an open question in the design blocks the generation, stop and ask me. Then validate and show me the plan.

**Under the hood:** ADA uses its Data Vault generation skill. It reads the design document, the staging YAML, `config/pipeline.yaml` and the ADE defaults in `config/ade_defaults/ade_schema.yaml`. It writes new packages under `packages/`, adds loads to `h_taxi_zone.yaml`, and runs `ada validate` and `ada plan`.

2) Review the plan. In the plan, `+` means an entity is created in ADE, `~` modified and `-` deleted. It must show:

- `+` for every hub, link and satellite in your design, in new packages
- `~` for **one** existing entity: `H_TAXI_ZONE` in `TRAFFIC_DV_<YOUR_ALIAS>`
- **no** `+` for a zone or location hub
- nothing else modified

3) Review the generated entities. **Example prompt:**

> Show me a summary of the generated entities, one row per entity: name, schema, package, business key or parent entities, and source entities. Then list the loads you added to `H_TAXI_ZONE`, one row per load, with the source column and how a NULL value is handled.

Check:

| Check | Why |
|---|---|
| Every schema and package name contains your alias | Otherwise you overwrite someone else's work |
| The entities match the design document | The design is the specification. If ADA deviates, ask why, then fix the YAML or the design |
| `H_TAXI_ZONE` has a new load for every zone column in the design: pickup, dropoff and depot | A missing load means missing zones |
| The movement timestamps are still text in the satellite | The format rule from Exercise 2 belongs to a later layer, not to the Raw Data Vault |
| The NULL zone handling matches the design | See below |

If you accept a deviation from the design, ask ADA to update the design document too. The design must describe what is actually built.

<details markdown="1">
<summary markdown="span">NULL zone IDs: what to look for</summary>

ADE's key transformations turn a NULL zone ID into the key `'-1'`. Filtering the NULL dropoffs out, or keeping `'-1'` as an *unknown zone*, are both fine, as long as the design says which. Do not accept a custom formula such as `'UNKNOWN'`: the taxi loads into the same hub already use `'-1'`.

</details>

### Checkpoint B

- `ada validate` passes
- The plan shows only new entities in new packages, and one modified entity: `H_TAXI_ZONE`
- The NULL zone handling is the one written in the design document

---

## Deploy and Run

Estimated duration: 10 minutes

1) Push, commit and deploy. **Example prompt:**

> Push all new Data Vault packages and `TRAFFIC_DV_<YOUR_ALIAS>` in one batch. Then commit them in ADE with the message "FLEETOPS Raw Data Vault", promote them to DEV, deploy them and wait for the deployment.

**Under the hood:** `ada push` with every package in one batch, because the packages reference each other. Then `ada deploy commit`, `ada deploy promote --to DEV` and `ada deploy start --to DEV --wait`.

2) Open DEV <span class="ui-action">Workflow Orchestration</span> and the DAG `FLEETOPS_<YOUR_ALIAS>` in the <span class="ui-action">Graph view</span>. Check:

- Are the new Data Vault loads there?
- Are the new `H_TAXI_ZONE` loads in **this** DAG, or in `TAXIDATA_<YOUR_ALIAS>`?

If the new loads are not there yet, wait a minute and refresh. ADE updates the DAG after the deployment, and a run that starts before that does not run the new loads.

> **Note:** ADE builds the DAGs from load dependencies. Only the staging loads have a schedule, and every Data Vault load inherits it from the load it depends on. The new `H_TAXI_ZONE` loads read FLEETOPS staging, so they run with FLEETOPS, even though the entity belongs to a taxi package.
{: .exercise-note}

3) Trigger the DAG. **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

### Checkpoint C

- All packages deployed with `SUCCESS`
- The Graph view of `FLEETOPS_<YOUR_ALIAS>` shows the new Data Vault loads and the new `H_TAXI_ZONE` loads
- The DAG succeeds

---

## Verify

Estimated duration: 25 minutes

A green DAG is not a correct result. Check the data.

1) Check your new hubs. Entity names depend on your design, so let ADA write the query. **Example prompt:**

> Write a short Databricks SQL query that returns `entity`, `row_count` and `expected_count` for my new FLEETOPS hubs, one `union all` line per hub. Expected counts: vehicles 42, shifts 1981, movements 13871, drivers 50. Skip the hubs my design does not have. Use the catalog `db_datahub_dev`.

Run the query in Databricks. Every `row_count` should equal its `expected_count`. If one does not, paste the result to ADA and ask it to explain.

2) The headline check, the shared hub:

```sql
select
  count(*)                                       as row_count,
  269                                            as expected_count,
  count_if(taxi_zone_key = '-1') as unknown_zone_rows
from db_datahub_dev.rdv_<your_alias>.H_TAXI_ZONE;
```

The FLEETOPS data uses the same NYC zone IDs as the taxi data. Three new loads write into the hub, but no new real zone should appear. `unknown_zone_rows` counts rows with the key `'-1'`, ADE's replacement for a missing business key.

### Checkpoint D

| Your result | What it means |
|---|---|
| `row_count` 269, `unknown_zone_rows` 0 | The NULL dropoffs were filtered out of the hub load |
| `row_count` 270, `unknown_zone_rows` 1 | The NULL dropoffs became an *unknown zone*. Correct **only if** this is the decision in your design document |
| Anything else | Something went wrong. Paste the result to ADA, and compare the zone keys in FLEETOPS with the hub |

If you got 270 without having decided it, you have found a *silent* data quality issue: the pipeline is green, but the data is not what you assumed. Ask ADA to explain where the extra row comes from and what your options are. Then decide, and record the decision in the design document.

---

## Summary of Exercise 4

You have:

- measured a shared hub before changing it, and found three keys basic training hid
- generated the Raw Data Vault from your own approved design
- extended an existing hub with a new source instead of creating a duplicate
- deployed and loaded the entities, and seen how ADE places the loads in DAGs
- verified the result against staging, and made the NULL-dropoff decision visible

The fleet data and the taxi data now meet in `H_TAXI_ZONE`. Next, you will turn the raw data into the metrics PackageDelivery needs for its decision.

---

Something not working? Ask ADA to help you debug it first, then check the [Training Q&A](Training_QA_ADA_Self_Study.md) before you ask your trainer.
