---
layout: exercise
module_label: "Module 1: Exercise 1"
title: "ADA Setup and New Source to Staging"
duration: "2 hours 30 minutes"
summary: "Install ADA, connect to ADE, bring the basic-training pipeline into your project, and add FLEETOPS to staging and PSA."
---

BigQuery as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere. Use your alias in lowercase: BigQuery dataset and table names are case-sensitive.

---

## Install ADA and Connect to ADE

Estimated duration: 70 minutes

### Watch: Admin UI

[Watch the Admin UI video](https://youtu.be/kbj3EmWxrwE)

In this training your trainer provides the API keys. In your own organisation, keys are created in the ADE Admin UI. Watch the video so you know where a key comes from, what permissions it needs, and why an installation can fail.

| Who does what | Training | Your own organisation |
|---|---|---|
| Creates the API keys | Trainer | Admin UI user |
| Allows your IP address in Access Rules | Trainer | Admin UI user |
| Installs ADA | You | You |

### Install

1) Download the starter project to your local machine:
https://www.agiledataengine.com/hubfs/Training/ada_starter_project_bigquery.zip

2) Extract the ZIP file and open the `ada_starter_project` folder in VS Code. It contains a `README.md`, the setup script `scripts/setup.py`, and the FLEETOPS sample files in `source_data/fleetops/`.

> **Important:** Do not rename or move the sample files. ADA derives entity names from the file names.
{: .exercise-note}

3) Open a terminal in VS Code and run the setup script:

```bash
python3 scripts/setup.py           # Windows: python scripts\setup.py
```

> **Note:** On Windows, start the script with `python`, as shown. Running `.\scripts\setup.py` directly opens a separate console window that closes when the script ends.
{: .exercise-note}

The script checks your Python version and tells you which interpreter to use if it is not supported (3.10 to 3.14, 3.12 or 3.13 recommended). Then it asks three questions:

| Question | Answer |
|---|---|
| ADE tenant ID | The tenant ID from your registration email |
| Alias | Your alias, in lowercase (BigQuery dataset names are case-sensitive) |
| AI coding tool | The tool you use: GitHub Copilot, Claude Code or both |

It shows what it will do and asks for confirmation. Then it works through five steps:

- creates a virtual environment in `.venv` and installs ADA. The installer asks for your **design** key ID and secret and checks the key before it installs anything
- scaffolds the project with `ada init`. Your sample files are kept
- writes your alias and the target database to `config/pipeline.yaml`
- asks for your **dev** key ID and secret, stores them, and fetches the tenant's configuration (entity types, load templates, schedules) with `ada config pull`
- puts the project under version control with an initial Git commit

| If you see | It means | What to do |
|---|---|---|
| `HTTP 401` / authorization failed | The key is wrong or lacks permissions | Check that you copied the key ID and secret correctly, then re-run the script |
| Unreachable / network error | Your IP address is not allowed | Contact your trainer |
| `Python x.y is not supported` | The Python you started the script with is too old or too new | Run the script with the interpreter the message names |

The script is safe to re-run, for example after a wrong key or a typo in the alias.

> **Note:** ADA uses Git for you from now on. `ada pull` and `ada push` create their own commits, and `ada push` refuses to run while the package has uncommitted changes. Some commands also update files in `config/`, such as the cached schemas. If `git status` shows changes you did not make, that is usually why. If Git is not installed, the script skips it. ADA works without Git too.
{: .exercise-note}

4) Activate the virtual environment in your terminal. The script prints the command at the end:

```bash
source .venv/bin/activate          # Windows: .venv\Scripts\activate
```

### Checkpoint A1

```bash
ada doctor
```

The output must end with both lines:

```
ADE environment reachable: <tenant ID>/<installation>/design
ADE environment reachable: <tenant ID>/<installation>/dev
```

### Optional: connect your AI tool to ADE with MCP

MCP (Model Context Protocol) lets the AI agent query ADE directly, for example which loads failed in a workflow run and why. In Exercise 2, the `ada-ops` agent can use it to diagnose failures. **Everything in this course also works without MCP**, so the choice is yours.

1) Generate the configuration for both environments:

```bash
ada mcp setup --env design --env dev
```

This creates `.vscode/mcp.json` (VS Code) and `.mcp.json` (Claude Code). Both files contain placeholders only, never your keys, and both are excluded from Git.

2) Connect:

- **VS Code:** VS Code finds `.vscode/mcp.json` only when the project folder is the folder you opened in VS Code. Start each server (`ade-mcp-design`, `ade-mcp-dev`), either with the <span class="ui-action">Start</span> link above the server in `mcp.json`, or from the Command Palette: <span class="ui-action">MCP: List Servers</span>, select the server, <span class="ui-action">Start Server</span>. VS Code asks for the key ID and secret of each environment and stores them securely.
- **Claude Code:** set the environment variables `ADE_DESIGN_API_KEY_ID`, `ADE_DESIGN_API_KEY_SECRET`, `ADE_DEV_API_KEY_ID` and `ADE_DEV_API_KEY_SECRET` before you start `claude`.

3) Test it. **Example prompt:**

> Use the ADE MCP server to check that you can reach ADE, and tell me the name of my API key.

### Check your configuration

From now on, you work through ADA. Start ADA in your AI tool:

- **GitHub Copilot in VS Code:** open Copilot Chat in <span class="ui-action">Agent</span> mode and select the `ada-developer` agent
- **Claude Code:** start `claude` in the project folder

Many trainees share the same ADE tenant, so every dataset and package needs your alias. In BigQuery, a schema in ADE is a dataset. The setup script has already set the alias for all entity types at once, so that nothing you create later lands in a shared default dataset.

**Review:** open `config/pipeline.yaml` and check the values yourself. Your alias is `<your_alias>` in the examples below:

```yaml
target_database: BIGQUERY
schema_overrides:
  SOURCE: src_<your_alias>
  STAGE: staging_<your_alias>
  PSA: psa_<your_alias>
  HUB: rdv_<your_alias>
  LINK: rdv_<your_alias>
  SAT: rdv_<your_alias>
  SAT_C: rdv_<your_alias>
  S_SAT: rdv_<your_alias>
  FACT: publish_<your_alias>
  DIM: publish_<your_alias>
package_name_postfix_override: <YOUR_ALIAS>
```

**Under the hood:** ADA reads `schema_overrides` and `package_name_postfix_override` from `config/pipeline.yaml` whenever it generates an entity or a package. `target_database` makes ADA write and render SQL in the BigQuery dialect. Without it, ADA assumes Snowflake.

> **Tip:** If a value is wrong, for example a typo in the alias, re-run `python3 scripts/setup.py` with the right alias, or edit `config/pipeline.yaml` yourself and run `ada doctor` again.
{: .exercise-note}

> **Note:** Datasets are lowercase (`staging_test`), but package names are always **uppercase** (`STG_FLEETOPS_TEST`), whatever case you type the postfix in.
{: .exercise-note}

> **Note:** AI agents do not always produce the same output twice. If your entity or load names differ slightly from this document, that is fine: the checkpoints describe what must be **true**, not what must be identical.
{: .exercise-note}

### Bring the basic-training pipeline into the project

In a later exercise you will connect the fleet data to the `H_TAXI_ZONE` hub from basic training. ADA needs that hub as YAML in your project, and the hub needs data in BigQuery.

The starter project contains a copy of the complete basic-training pipeline in `model_packages/`: `TAXIDATA_STAGING_ZZALIAS` (source, staging and PSA entities), `TRAFFIC_DV_ZZALIAS` and `TRAFFIC_PUBLISH_ZZALIAS`. Wherever an alias belongs, the copy has the placeholder `zzalias` (`ZZALIAS` in uppercase names). You will turn the copy into your own packages, deploy them and load the taxi data.

> **Note:** The copy contains no entity IDs, so pushing it can only create new entities. It cannot change anyone else's packages. Use an alias you have not used in this tenant before. If you did the basic training with the same alias, the plan in step 2 shows `~` (modify) lines instead of only `+` lines.
{: .exercise-note}

1) Create a schedule for the taxi pipeline. **Example prompt:**

> Create a load schedule `TAXIDATA_<YOUR_ALIAS>` with no cron expression and the description `#taxidata #<your_alias>`.

2) Create your packages from the copy. **Example prompt:**

> Copy the three packages in `model_packages/` into `packages/`. In the copies, replace `zzalias` with `<your_alias>` and `ZZALIAS` with `<YOUR_ALIAS>` everywhere: in folder names, file contents and SQL files. Do not change anything in `model_packages/`. Do not touch the `gs://` paths in the `.sql` files.

> **Note:** The placeholder also renames the schedule to `TAXIDATA_<YOUR_ALIAS>`. Until you have replaced it and created that schedule, `ada validate` reports `TAXIDATA_ZZALIAS` as an unknown schedule.
{: .exercise-note}

**Review:** ask ADA to check the result:

> Search `packages/` for `zzalias`, ignoring case, and validate my three new packages. Then show me the plan for them.

- The search finds **nothing**
- Validation passes
- The plan shows **only `+` (create) lines**: 23 entities in three packages, and `0 to modify, 0 to delete`

> **Note:** The plan has no satellite current views (`S_..._C`). ADE creates them automatically for each satellite.
{: .exercise-note}

> **The standard routine.** Whenever you change entities, take them to ADE in this order. ADA knows the routine; your job is to read each result before the next step.
{: .exercise-note}

| Step | Command | What you check |
|---|---|---|
| 1. Validate | `ada validate` | No errors |
| 2. Plan | `ada plan` | Only the changes you expect |
| 3. Dry run | `ada push --dry-run` | ADE accepts the change |
| 4. Push | `ada push` | The upserted count matches the plan |
| 5. Deploy | `ada deploy commit`, `ada deploy promote --to DEV`, `ada deploy start --to DEV --wait` | Every package shows `SUCCESS` |

You use the dry run in this exercise to see what it checks. Later exercises go straight from the plan to the push. A commit alone does not deploy anything: every deployment needs the commit, the promotion to DEV and the deployment itself.

3) Push and deploy. **Example prompt:**

> Push my three taxi packages to ADE in one push. Then commit `CONFIG_LOAD_SCHEDULES`, promote it to DEV, deploy it and wait for the deployment. Then do the same for my three taxi packages in the order staging, DV, publish, one deployment at a time.

**Under the hood:** `ada push -p TAXIDATA_STAGING_<YOUR_ALIAS> -p TRAFFIC_DV_<YOUR_ALIAS> -p TRAFFIC_PUBLISH_<YOUR_ALIAS>`, then for each package `ada deploy commit -p <PACKAGE> -m "..."`, `ada deploy promote --to DEV -p <PACKAGE>` and `ada deploy start --to DEV --wait`

4) The taxi files are already in the landing zone, in a folder shared by all trainees. Open one of your staging load files, for example `stg_taxidata_yellow_tripdata.sql`, and find the path in the `LOAD DATA` statement. You do not need to load or copy any files.

> **Note:** The staging loads **truncate** the table before every load. Running a DAG again therefore never doubles your rows. BigQuery's `LOAD DATA` does not remember which files it has loaded.
{: .exercise-note}

5) Open DEV <span class="ui-action">Workflow Orchestration</span>, find the DAG `TAXIDATA_<YOUR_ALIAS>`, switch it <span class="ui-action">on</span> with the toggle, and check in the <span class="ui-action">Graph view</span> that your loads are there. Then trigger it. **Example prompt:**

> Trigger the DAG `TAXIDATA_<YOUR_ALIAS>` in dev and wait for it to finish.

> **Note:** A run can stay `QUEUED` for several minutes before it starts. This is normal. If it stays queued for more than ten minutes, ask your trainer.
{: .exercise-note}

### Checkpoint A2

- The DAG `TAXIDATA_<YOUR_ALIAS>` succeeds
- `packages/traffic_dv_<your_alias>/` contains `h_taxi_zone.yaml`
- This query returns 266, as in basic training. Run it in BigQuery Studio:

```sql
select count(1) as row_count, 266 as expected_count
from `agiledataengine-training`.rdv_<your_alias>.H_TAXI_ZONE h
join `agiledataengine-training`.rdv_<your_alias>.S_TAXI_ZONE_C s on h.dv_id = s.dv_id;
```

> **Note:** The project name contains hyphens, so it needs backticks. Dataset and table names are case-sensitive: `H_TAXI_ZONE` is uppercase, your dataset is lowercase.
{: .exercise-note}

---

## Add the FLEETOPS Source

Estimated duration: 35 minutes

The `FLEETOPS` source system already exists in ADE (`CONFIG_SYSTEMS`). Your task is to describe the source files to ADA so that it can create the SOURCE, STAGE and PSA entities.

> **Why sample files?** `source_data/fleetops/` contains only a sample: the first week of January. The full files are in the landing zone. ADA infers the file structure and the column types from the sample, and the later loads read the full files.
{: .exercise-note}

1) **Example prompt:**

> Inspect the three files in `source_data/fleetops/` and register them as source system `fleetops`. Also remove the example source `your_source` from `config/sources.yaml`.

**Under the hood:** `ada staging inspect -s fleetops -f <file>` for each file. ADA adds `fleetops` to `config/sources.yaml`.

2) **Example prompt:**

> Generate staging entities for `fleetops`. The data will be loaded with a hand-written `LOAD DATA` statement from cloud storage, so use the TRANSFORM_PERSIST load type.

**Under the hood:** `ada staging generate --source fleetops --load-type TRANSFORM_PERSIST`

ADA creates the package `STG_FLEETOPS_<YOUR_ALIAS>` with three SOURCE entities and three STAGE entities. It does not create a `.sql` file for the loads. You write those in the next section.

> **Note:** In this BigQuery environment the STAGE entities have no metadata attributes such as `stg_batch_id`. BigQuery's `LOAD DATA` cannot add columns that are not in the file. The staging table is an exact copy of the file.
{: .exercise-note}

3) Add the PSA (Persistent Staging Area) layer. **Example prompt:**

> Generate PSA entities for the three FLEETOPS staging entities in `packages/stg_fleetops_<your_alias>/`. Then compare them with the PSA entities of the taxi data in `packages/taxidata_staging_<your_alias>/`.

**Under the hood:** `ada psa generate -i packages/stg_fleetops_<your_alias>`

A PSA entity has the same business columns as its STAGE entity, plus a hash of the row (`stg_hashdiff`), a run ID, a load time and the source name. The PSA load inserts only rows that are new or changed, and later layers read from the PSA, not from staging.

**Review:** the FLEETOPS PSA entities must follow the taxi PSA entities. The load type is `TRANSFORM_PERSIST`, the `stg_hashdiff` mapping uses `HASH_DIFF` over all business attributes, and the load options include `OPT_HISTORIZED_BY_DATAHASH_AS_INSERT`.

4) **Review the data types.** Open `stg_fleetops_fleet_movement.yaml` and check the types ADA inferred from the sample:

| Attribute | Expected type |
|---|---|
| `pickup_locationid`, `dropoff_locationid` | INTEGER8 |
| `movement_start_ts`, `movement_end_ts` | TIMESTAMP |
| `distance`, `delivery_revenue_usd` | DOUBLE |
| `created`, `updated` | DATE |

> **Note:** ADE's `TIMESTAMP` type is BigQuery's `DATETIME` (a timestamp without a time zone). `DOUBLE` is `FLOAT64`, and `INTEGER8` is `INT64`.
{: .exercise-note}

> **Optional:** `DOUBLE` is a floating-point type. It is not ideal for money. If you want to practise, ask ADA to change `delivery_revenue_usd` to a DECIMAL type.
{: .exercise-note}

### Checkpoint B

**Example prompt:**

> Validate the FLEETOPS staging package and show me what pushing it would change in ADE.

**Under the hood:** `ada validate packages/stg_fleetops_<your_alias>/ --recursive`, then `ada plan -p STG_FLEETOPS_<YOUR_ALIAS>`

Both of these must be true:

- Validation reports `All 9 files valid`
- The plan shows **exactly 9 entities to create and nothing else**:

```
STG_FLEETOPS_<YOUR_ALIAS>:
  + src_<your_alias>.FLEET_MOVEMENT (SOURCE)
  + src_<your_alias>.FLEET_SHIFT (SOURCE)
  + src_<your_alias>.FLEET_VEHICLE (SOURCE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT (STAGE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT (STAGE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE (STAGE)
  + psa_<your_alias>.PSA_FLEETOPS_FLEET_MOVEMENT (PSA)
  + psa_<your_alias>.PSA_FLEETOPS_FLEET_SHIFT (PSA)
  + psa_<your_alias>.PSA_FLEETOPS_FLEET_VEHICLE (PSA)

Plan: 9 to create, 0 to modify, 0 to delete.
```

If you see your alias in every dataset and in the package name, your alias configuration works. If the plan shows anything to modify or delete, stop and find out why before you continue.

---

## Load the Files into Staging

Estimated duration: 45 minutes

The production files are in the landing zone in one shared Cloud Storage folder:

```
gs://ade-training-sourcedata/src/fleetops/package_delivery/
    fleet_vehicle.csv
    fleet_shift.csv
    fleet_movement.csv
```

### Create a load schedule

1) **Example prompt:**

> Create a load schedule `FLEETOPS_<YOUR_ALIAS>` with no cron expression and the description `#fleetops #<your_alias>`.

**Under the hood:** `ada config schedule add --name FLEETOPS_<YOUR_ALIAS> --description "#fleetops #<your_alias>"`

> **Note:** A schedule becomes a DAG in Workflow Orchestration. Using your own schedule means that running your fleet loads does not re-run your taxi loads from basic training.
{: .exercise-note}

### Write the staging loads

2) The FLEETOPS STAGE entities have a load from the source entity, but no load logic yet. In this BigQuery environment the logic is a short SQL file next to the entity YAML. Your taxi staging loads already have one. Open `stg_taxidata_yellow_tripdata.sql` in your taxi package:

```sql
-- step: load_staging_<your_alias>_taxidata_yellow_tripdata_from_src_<your_alias>_yellow_tripdata.truncate_table
-- type: PRE
-- language: SQL
TRUNCATE TABLE <target_schema>.<target_entity_name>;

-- step: load_staging_<your_alias>_taxidata_yellow_tripdata_from_src_<your_alias>_yellow_tripdata.load_data
-- type: OVERRIDE
-- language: SQL
LOAD DATA INTO <target_schema>.<target_entity_name> (
  vendorid INT64,
  tpep_pickup_datetime DATETIME,
  ...
  total_amount FLOAT64
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://ade-training-sourcedata/src/taxidata_trainer/tripdata_yellow/*.csv'],
  skip_leading_rows = 1
);
```

A load has two steps: a `PRE` step that empties the table, and an `OVERRIDE` step that replaces the generated load with your `LOAD DATA`. The load in the STAGE YAML file must declare both steps in a `steps:` list (`truncate_table` as `PRE`, `load_data` as `OVERRIDE`). Open `stg_taxidata_yellow_tripdata.yaml` to see how. The `.sql` file only holds the SQL text of the declared steps.

Before you read on, try to find **three things** that would be wrong if you copied this file for `fleet_movement.csv` without changes. Use what you know about the FLEETOPS files and the landing zone above.

<details markdown="1">
<summary markdown="span">The three problems</summary>

| Problem | Why it matters |
|---|---|
| The path ends with `/*.csv` | The taxi folders hold only the files of one entity. The FLEETOPS folder holds **three different files**. With `*.csv`, every load would read all three files into one table, or fail on the first column that does not fit |
| The column list is the list of the taxi table | `LOAD DATA` reads the columns of the file by position into the columns you list. The list must have the columns of the FLEETOPS file, in the order of the file, with BigQuery types that match the entity: `DATETIME` for `TIMESTAMP`, `FLOAT64` for `DOUBLE`, `INT64` for `INTEGER8`, `STRING` for `VARCHAR`. If you declare BigQuery `TIMESTAMP` instead of `DATETIME`, the load fails with `Invalid schema update. Field ... has changed type from DATETIME to TIMESTAMP`. An agent often makes this mistake |
| The load has no schedule | A load without a schedule does not appear in any DAG |

</details>

`skip_leading_rows = 1` is already right: the FLEETOPS files have a header row. The `TRUNCATE` step is also right. Keep it, and keep both steps declared in the YAML: without the `steps:` list, ADE ignores the `.sql` file and generates its own load, which reads the SOURCE entity and fails with `Dataset ... src_<your_alias> was not found`.

3) **Example prompt:**

> Using the taxi staging loads in `packages/taxidata_staging_<your_alias>/` as a pattern, write the load logic for all three FLEETOPS staging entities. In each STAGE YAML file, declare the steps `truncate_table` (`PRE`) and `load_data` (`OVERRIDE`) in the load. Create one `.sql` file per STAGE entity next to its YAML file, with the matching steps: a `PRE` step that truncates the table and an `OVERRIDE` step with a `LOAD DATA` statement:
> - `uris` is the single matching file in `gs://ade-training-sourcedata/src/fleetops/package_delivery/`, for example `fleet_movement.csv`. Do not use a wildcard
> - the column list has the columns of that file in file order, with BigQuery types that match the STAGE entity: declare ADE `TIMESTAMP` columns as `DATETIME` (the staging tables use `DATETIME`, not BigQuery's `TIMESTAMP`)
> - `skip_leading_rows` is 1
> - keep `<target_schema>` and `<target_entity_name>` as variables
> - set the load schedule to `FLEETOPS_<YOUR_ALIAS>` in all three STAGE YAML files

**Under the hood:** ADA adds a `truncate_table` step and a `load_data` step to each load, and writes `stg_fleetops_fleet_movement.sql` and the two others. The PSA loads need no schedule. A load inherits the schedule from the load it depends on.

> **Note:** The column list is hand-written SQL. If you change a column type in the YAML later, the `.sql` file does not change by itself. You will meet this in Exercise 2.
{: .exercise-note}

> **Note:** The landing zone address is written into the SQL file. This tenant has no environment variable for it. That is fine in a training DEV environment.
{: .exercise-note}

**Review:** check all three `.sql` files, and the `steps:` list and the `schedulingName` in all three STAGE YAML files.

### Validate and push the staging package

4) Review the local files, then validate and run a dry run. **Example prompt:**

> Check all three FLEETOPS SQL files and their matching STAGE YAML files. Confirm each `LOAD DATA` reads one single file with `skip_leading_rows = 1`, has a `TRUNCATE` before it, and has the columns in the order of the file with `DATETIME` for the timestamps. Confirm each STAGE load declares the steps `truncate_table` and `load_data` and has the schedule `FLEETOPS_<YOUR_ALIAS>`. Then run `ada validate`, show me the plan, and run a push dry run. Do not push yet.

### Checkpoint C1

- Each local SQL file reads its own single file under `gs://ade-training-sourcedata/src/fleetops/package_delivery/`, has `skip_leading_rows = 1`, and starts with a `TRUNCATE`.
- Each STAGE YAML has `schedulingName: FLEETOPS_<YOUR_ALIAS>` and declares the steps `truncate_table` and `load_data`.
- `ada validate` passes, the plan shows only the nine expected entities to create, and the push dry run succeeds.

5) Push the package only after the review passes. **Example prompt:**

> Push the FLEETOPS staging package to ADE.

**Under the hood:** `ada push -p STG_FLEETOPS_<YOUR_ALIAS>`

### Deploy and run

6) Your new schedule lives in the shared `CONFIG_LOAD_SCHEDULES` package. It must be deployed before your package, just like in basic training.

**Example prompt:**

> Commit `CONFIG_LOAD_SCHEDULES` with the message "Add `FLEETOPS_<YOUR_ALIAS>` schedule", promote it to DEV, deploy it and wait for the deployment. Then do the same for my FLEETOPS staging package.

**Under the hood:** `ada deploy commit -p CONFIG_LOAD_SCHEDULES -m "..."`, `ada deploy promote --to DEV -p CONFIG_LOAD_SCHEDULES`, `ada deploy start --to DEV --wait`, then the same for `STG_FLEETOPS_<YOUR_ALIAS>`

> **Note:** `ada deploy start --wait` may print `Deployment RUNNING` even though every package below it shows `SUCCESS`. This only means the overall deployment had not closed when ADA stopped waiting. Trust the package lines.
{: .exercise-note}

7) Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`. Switch it <span class="ui-action">on</span> with the toggle, then open the <span class="ui-action">Graph view</span> and check that all three staging loads **and the three PSA loads** are there. A DAG that is switched off does not run.

8) **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

**Under the hood:** `ada dag trigger --dag-id FLEETOPS_<YOUR_ALIAS> --env dev --wait`

> **Note:** The run can stay `QUEUED` for several minutes before it starts. Wait for it.
{: .exercise-note}

### Checkpoint C2: the DAG fails

**The DAG fails. This is expected.**

Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`:

| Load | Expected state |
|---|---|
| staging `..._fleet_vehicle_...` | Success |
| staging `..._fleet_shift_...` | Success |
| staging `..._fleet_movement_...` | Failed |
| PSA `..._fleet_vehicle_...` | Success |
| PSA `..._fleet_shift_...` | Success |
| PSA `..._fleet_movement_...` | Not run (upstream failed) |

Check the row counts in BigQuery Studio:

```sql
select 'stg vehicle' as entity, count(*) as row_count, 42 as expected_count
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'stg shift', count(*), 1981
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'stg movement', count(*), 0
from `agiledataengine-training`.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT
union all
select 'psa vehicle', count(*), 42
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_VEHICLE
union all
select 'psa shift', count(*), 1981
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_SHIFT
union all
select 'psa movement', count(*), 0
from `agiledataengine-training`.psa_<your_alias>.PSA_FLEETOPS_FLEET_MOVEMENT;
```

**Do not fix the movement load yet.** Finding out why it failed is the starting point of Exercise 2.

---

## Summary of Exercise 1

You have:

- installed ADA with the setup script and connected it to the design and dev environments
- brought your basic-training work into the project as YAML
- described a new source to ADA and reviewed the staging and PSA entities it generated
- found and fixed three problems in a load before it reached ADE
- deployed and run a pipeline without opening Designer

Two of three files are in staging and PSA. The third, `fleet_movement.csv`, failed. It holds the data that the expansion decision depends on.

---

Something not working? Ask ADA to help you debug it first, then check the [Training Q&A](Training_QA_ADA_Self_Study.md) before you ask your trainer.
