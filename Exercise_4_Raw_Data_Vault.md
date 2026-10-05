# Exercise 4 — Generate and Load the Raw Data Vault

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The task

Estimated duration: 1 hour 30 minutes

Your Data Vault design is approved. Now ADA generates the entities from it, you take them to ADE, and you load the FLEETOPS data into the Raw Data Vault.

The most important entity in this exercise is one you do not create: the existing `H_TAXI_ZONE`. The fleet data connects to the taxi data through it. When this exercise is done, the same hub holds the zones of both businesses — and you will check that it still holds the right number of rows.

**Result of this exercise:** the FLEETOPS Raw Data Vault entities are deployed to DEV and loaded, and `H_TAXI_ZONE` is loaded from both the taxi data and the fleet data.

---

### Part A — Measure the Starting Point

Estimated duration: 10 minutes

Before you change a shared hub, measure it. Otherwise you cannot tell what your change did.

1) Count the rows in your `H_TAXI_ZONE`. Run in Snowflake:

```sql
select count(*) as row_count, 269 as expected_count
from ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.H_TAXI_ZONE;
```

In basic training, the check query returned 266. Why does the hub have more rows? Find the zones that have no description:

```sql
select h.taxi_zone_key
from ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.H_TAXI_ZONE h
left join ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.S_TAXI_ZONE_C s on h.dv_id = s.dv_id
where s.dv_id is null
order by 1;
```

2) Find out where the hub gets its rows from. **Example prompt** (to `ada-developer`):

> How many loads does the entity `H_TAXI_ZONE` in `packages/traffic_dv_<your_alias>/` have, and from which source entities and columns? Do not change anything.

These loads are the pattern you extend in Part B: the FLEETOPS zone columns get loads of the same kind.

#### ✅ Checkpoint A

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

ℹ Write down 269. You will check it again in Part D.

---

### Part B — Generate the Entities

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
| The entities match the design document | The design is the specification. If ADA deviates, ask why — then fix the YAML or the design |
| `H_TAXI_ZONE` has a new load for every zone column in the design: pickup, dropoff and depot | A missing load means missing zones |
| The movement timestamps are still text in the satellite | The format rule from Exercise 2 belongs to a later layer, not to the Raw Data Vault |
| The NULL zone handling matches the design | See below |

If you accept a deviation from the design, ask ADA to update the design document too. The design must describe what is actually built.

<details markdown="1">
<summary markdown="span">NULL zone IDs — what to look for</summary>

ADE's hash key and business key transformations turn a NULL into `'-1'`. Unless the load filters the NULL rows out, the 40 movements without a dropoff zone create a new hub row with the key `'-1'`.

Both are valid choices — filtering the rows out, or keeping `'-1'` as an explicit *unknown zone* — as long as it is the choice written in your design. If the design is silent, decide now and ask ADA to record it.

Avoid a custom formula that maps NULL to something else, such as `'UNKNOWN'`. The five taxi loads into the same hub already use ADE's `'-1'`, so a custom value gives one hub two conventions for *unknown*, and a formula that differs from ADE's standard can break the hash keys that connect fleet zones to taxi zones.

</details>

#### ✅ Checkpoint B

- `ada validate` passes
- The plan shows only new entities in new packages, and one modified entity: `H_TAXI_ZONE`
- The NULL zone handling is the one written in the design document

---

### Part C — Deploy and Run

Estimated duration: 25 minutes

1) Push the new packages and the modified `TRAFFIC_DV_<YOUR_ALIAS>`. **Example prompt:**

> Push all new Data Vault packages and `TRAFFIC_DV_<YOUR_ALIAS>` in one batch, with a dry run first. Do not commit them in ADE yet.

**Under the hood:** `ada push --dry-run`, then `ada push` with every package. The packages reference each other, so they are pushed in one batch. `ada push` refuses uncommitted changes, so ADA commits the YAML to Git first.

2) Now that the entities are in ADE, look at the SQL ADE will run for the dropoff zones. **Example prompt:**

> Show me the code preview of the `H_TAXI_ZONE` load from the FLEETOPS dropoff zone. What happens to the rows where `dropoff_locationid` is NULL?

**Under the hood:** `ada code preview` for the package `TRAFFIC_DV_<YOUR_ALIAS>`, against the design environment (no `--env`). The preview shows what is in ADE, so it only works after the push.

3) Commit the packages. **Example prompt:**

> Commit all new Data Vault packages and `TRAFFIC_DV_<YOUR_ALIAS>` in ADE with the message "FLEETOPS Raw Data Vault" and wait for the deployments.

4) Check which loads belong to the DAG. **Example prompt:**

> List the loads of the DAG `FLEETOPS_<YOUR_ALIAS>` in dev.

**Under the hood:** `ada workflow describe --env dev --dag-id FLEETOPS_<YOUR_ALIAS>`.

Then open DEV Workflow Orchestration and the DAG `FLEETOPS_<YOUR_ALIAS>` in the Graph view, which shows the same loads and their order.

- Are the new Data Vault loads there?
- Are the new `H_TAXI_ZONE` loads in **this** DAG, or in `TAXIDATA_<YOUR_ALIAS>`?

ℹ ADE builds the DAGs from load dependencies. Only the staging loads have a schedule; every Data Vault load inherits it from the load it depends on. The new `H_TAXI_ZONE` loads read FLEETOPS staging, so they run with FLEETOPS, even though the entity belongs to a taxi package. You do not set a schedule on Data Vault loads.

5) Trigger the DAG. **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

#### ✅ Checkpoint C

- The code preview shows the NULL zone handling written in your design
- All packages deployed with `SUCCESS`
- The DAG `FLEETOPS_<YOUR_ALIAS>` contains the new Data Vault loads and the new `H_TAXI_ZONE` loads
- The DAG succeeds

---

### Part D — Verify

Estimated duration: 25 minutes

A green DAG is not a correct result. Check the data.

1) Check your new hubs. Entity names depend on your design, so let ADA write the query. **Example prompt:**

> Write a short Snowflake query that returns `entity`, `row_count` and `expected_count` for my new FLEETOPS hubs, one `union all` line per hub. Expected counts: vehicles 42, shifts 1981, movements 13871, drivers 50. Skip the hubs my design does not have. Use the database `ADE_TRAINING_DEV_SAAS`.

Run the query in Snowflake. Every `row_count` should equal its `expected_count`. If one does not, paste the result to ADA and ask it to explain.

2) The headline check — the shared hub:

```sql
select
  count(*)                                       as row_count,
  269                                            as expected_count,
  count_if(try_to_number(taxi_zone_key) is null) as unknown_zone_rows
from ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.H_TAXI_ZONE;
```

The FLEETOPS data uses the same NYC zone IDs as the taxi data. Three new loads write into the hub, but no new real zone should appear. `unknown_zone_rows` counts keys that are not zone numbers, such as `'-1'`.

#### ✅ Checkpoint D

| Your result | What it means |
|---|---|
| `row_count` 269, `unknown_zone_rows` 0 | The NULL dropoffs were filtered out of the hub load |
| `row_count` 270, `unknown_zone_rows` 1 | The NULL dropoffs became an *unknown zone*. Correct **only if** this is the decision in your design document |
| Anything else | Something went wrong. Paste the result to ADA, and compare the zone keys in FLEETOPS with the hub |

If you got 270 without having decided it, you have found a *silent* data quality issue: the pipeline is green, but the data is not what you assumed. Ask ADA to explain where the extra row comes from and what your options are. Then decide, and record the decision in the design document.

3) Commit your work. **Example prompt:**

> Commit the remaining changes, such as the updated design document, to Git with the message "FLEETOPS Raw Data Vault loaded".

---

### Summary of Exercise 4

You have:

- measured a shared hub before changing it, and found three keys basic training hid
- generated the Raw Data Vault from your own approved design
- extended an existing hub with a new source instead of creating a duplicate
- deployed and loaded the entities, and seen how ADE places the loads in DAGs
- verified the result against staging, and made the NULL-dropoff decision visible

The fleet data and the taxi data now meet in `H_TAXI_ZONE`. Next, you will turn the raw data into the metrics PackageDelivery needs for its decision.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` |
| The plan shows `+` for a zone or location hub | ADA did not reuse `H_TAXI_ZONE`. Check the design document, and point ADA to `packages/traffic_dv_<your_alias>/h_taxi_zone.yaml` |
| The plan does not show `~ H_TAXI_ZONE` | ADA created the zone loads somewhere else, or not at all. Ask it to add them to the existing entity |
| `ada push` fails with an unknown reference | The packages reference each other. Push them in one batch |
| `H_TAXI_ZONE` does not exist in Snowflake | Your basic-training model is not deployed. Go back to Exercise 1, Part A |
| The DAG succeeds but the new tables are empty | The staging data was loaded before the new loads existed. Ask ADA to check the run IDs of the FLEETOPS staging entities |
