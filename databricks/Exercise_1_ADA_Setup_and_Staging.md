---
layout: exercise
module_label: "Module 1: Exercise 1"
title: "ADA Setup and New Source to Staging"
duration: "2 hours 20 minutes"
summary: "Install ADA, connect to ADE, bring the basic-training pipeline into your project, and add FLEETOPS to staging."
---

Databricks as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

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
https://www.agiledataengine.com/hubfs/Training/ada_starter_project_databricks.zip

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
| Alias | Your alias, in lowercase |
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

### Check your configuration

From now on, you work through ADA. Start ADA in your AI tool:

- **GitHub Copilot in VS Code:** open Copilot Chat in <span class="ui-action">Agent</span> mode and select the `ada-developer` agent
- **Claude Code:** start `claude` in the project folder

Many trainees share the same ADE tenant, so every schema and package needs your alias. The setup script has already set the alias for all entity types at once, so that nothing you create later lands in a shared default schema.

**Review:** open `config/pipeline.yaml` and check the values yourself. Your alias is `<your_alias>` in the examples below:

```yaml
target_database: DATABRICKS
schema_overrides:
  SOURCE: src_<your_alias>
  STAGE: staging_<your_alias>
  HUB: rdv_<your_alias>
  LINK: rdv_<your_alias>
  SAT: rdv_<your_alias>
  SAT_C: rdv_<your_alias>
  S_SAT: rdv_<your_alias>
  FACT: publish_<your_alias>
  DIM: publish_<your_alias>
package_name_postfix_override: <YOUR_ALIAS>
```

**Under the hood:** ADA reads `schema_overrides` and `package_name_postfix_override` from `config/pipeline.yaml` whenever it generates an entity or a package. `target_database` makes ADA write and render SQL in the Databricks dialect. Without it, ADA assumes Snowflake.

> **Tip:** If a value is wrong, for example a typo in the alias, re-run `python3 scripts/setup.py` with the right alias, or edit `config/pipeline.yaml` yourself and run `ada doctor` again.
{: .exercise-note}

> **Note:** Schemas are lowercase (`staging_test`), but package names are always **uppercase** (`STG_FLEETOPS_TEST`), whatever case you type the postfix in.
{: .exercise-note}

> **Note:** AI agents do not always produce the same output twice. If your entity or load names differ slightly from this document, that is fine: the checkpoints describe what must be **true**, not what must be identical.
{: .exercise-note}

### Bring the basic-training pipeline into the project

In a later exercise you will connect the fleet data to the `H_TAXI_ZONE` hub from basic training. ADA needs that hub as YAML in your project, and the hub needs data in Databricks.

The starter project contains a copy of the complete basic-training pipeline in `model_packages/`: `TAXIDATA_STAGING_ZZALIAS`, `TRAFFIC_DV_ZZALIAS` and `TRAFFIC_PUBLISH_ZZALIAS`. Wherever an alias belongs, the copy has the placeholder `zzalias` (`ZZALIAS` in uppercase names). You will turn the copy into your own packages, deploy them and load the taxi data.

> **Note:** The copy contains no entity IDs, so pushing it can only create new entities. It cannot change anyone else's packages. Use an alias you have not used in this tenant before. If you did the basic training with the same alias, the plan in step 2 shows `~` (modify) lines instead of only `+` lines.
{: .exercise-note}

1) Create a schedule for the taxi pipeline. **Example prompt:**

> Create a load schedule `TAXIDATA_<YOUR_ALIAS>` with no cron expression and the description `#taxidata #<your_alias>`.

2) Create your packages from the copy. **Example prompt:**

> Copy the three packages in `model_packages/` into `packages/`. In the copies, replace `zzalias` with `<your_alias>` and `ZZALIAS` with `<YOUR_ALIAS>` everywhere: in folder names, file contents and SQL files. Do not change anything in `model_packages/`.

> **Note:** The placeholder also renames the schedule to `TAXIDATA_<YOUR_ALIAS>`. Until you have replaced it and created that schedule, `ada validate` reports `TAXIDATA_ZZALIAS` as an unknown schedule.
{: .exercise-note}

**Review:** ask ADA to check the result:

> Search `packages/` for `zzalias`, ignoring case, and validate my three new packages. Then show me the plan for them.

- The search finds **nothing**
- Validation passes
- The plan shows **only `+` (create) lines**: 19 entities in three packages, and `0 to modify, 0 to delete`

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

4) Load the taxi files into the landing zone, as in basic training. The taxi loads read their files from a folder named after your alias, for example `<unity_catalog_raw>/TAXIDATA/staging_<your_alias>/RATECODE_DESCRIPTION/`. The **Load Training Data** job in Databricks copies the files there. It loads the taxi data only: the FLEETOPS files are already in the landing zone (see below):

   - Open the [Load Training Data job](https://adb-1302321888178973.13.azuredatabricks.net/jobs/586698680803194?o=1302321888178973) in Databricks
   - Click the arrow next to **Run now** and choose **Run now with different parameters**. Set `alias` to `<your_alias>` and `batch_number` to `1`, then click **Run**
   - Wait until the job run has succeeded
   - Run the job again with `batch_number` set to `2`

> **Important:** The alias is case-sensitive.
{: .exercise-note}

5) Open DEV <span class="ui-action">Workflow Orchestration</span>, find the DAG `TAXIDATA_<YOUR_ALIAS>`, switch it <span class="ui-action">on</span> with the toggle, and check in the <span class="ui-action">Graph view</span> that your loads are there. Then trigger it. **Example prompt:**

> Trigger the DAG `TAXIDATA_<YOUR_ALIAS>` in dev and wait for it to finish.

### Checkpoint A2

- The DAG `TAXIDATA_<YOUR_ALIAS>` succeeds
- `packages/traffic_dv_<your_alias>/` contains `h_taxi_zone.yaml`
- This query returns 266, as in basic training:

```sql
select count(1) as row_count, 266 as expected_count
from db_datahub_dev.rdv_<your_alias>.H_TAXI_ZONE h
join db_datahub_dev.rdv_<your_alias>.S_TAXI_ZONE_C s on h.dv_id = s.dv_id;
```

---

## Add the FLEETOPS Source

Estimated duration: 25 minutes

The `FLEETOPS` source system already exists in ADE (`CONFIG_SYSTEMS`). Your task is to describe the source files to ADA so that it can create the SOURCE and STAGE entities.

> **Why sample files?** `source_data/fleetops/` contains a **sample extract**: the first week of January, not the full production files. This is common in real projects: the source owner gives you a sample, and the production data arrives later in the landing zone. ADA infers column types from the sample. Keep this in mind. It will matter later.
{: .exercise-note}

1) **Example prompt:**

> Inspect the three files in `source_data/fleetops/` and register them as source system `fleetops`. Also remove the example source `your_source` from `config/sources.yaml`.

**Under the hood:** `ada staging inspect -s fleetops -f <file>` for each file. ADA adds `fleetops` to `config/sources.yaml`.

2) **Example prompt:**

> Generate staging entities for `fleetops`. The data will be loaded with a COPY INTO statement from cloud storage, so use the TRANSFORM_PERSIST load type.

**Under the hood:** `ada staging generate --source fleetops --load-type TRANSFORM_PERSIST`

ADA creates the package `STG_FLEETOPS_<YOUR_ALIAS>` with three SOURCE entities, three STAGE entities, and a `.sql` file for each staging load.

3) **Review the data types.** Open `stg_fleetops_fleet_movement.yaml` and check the types ADA inferred from the sample:

| Attribute | Expected type |
|---|---|
| `pickup_locationid`, `dropoff_locationid` | INTEGER8 |
| `movement_start_ts`, `movement_end_ts` | TIMESTAMP |
| `distance`, `delivery_revenue_usd` | DOUBLE |
| `created`, `updated` | DATE |

> **Optional:** `DOUBLE` is a floating-point type. It is not ideal for money. If you want to practise, ask ADA to change `delivery_revenue_usd` to a DECIMAL type.
{: .exercise-note}

### Checkpoint B

**Example prompt:**

> Validate the FLEETOPS staging package and show me what pushing it would change in ADE.

**Under the hood:** `ada validate packages/stg_fleetops_<your_alias>/ --recursive`, then `ada plan -p STG_FLEETOPS_<YOUR_ALIAS>`

Both of these must be true:

- Validation reports `All 6 files valid`
- The plan shows **exactly 6 entities to create and nothing else**:

```
STG_FLEETOPS_<YOUR_ALIAS>:
  + src_<your_alias>.FLEET_MOVEMENT (SOURCE)
  + src_<your_alias>.FLEET_SHIFT (SOURCE)
  + src_<your_alias>.FLEET_VEHICLE (SOURCE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT (STAGE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT (STAGE)
  + staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE (STAGE)

Plan: 6 to create, 0 to modify, 0 to delete.
```

If you see your alias in every schema and in the package name, your alias configuration works. If the plan shows anything to modify or delete, stop and find out why before you continue.

---

## Load the Files into Staging

Estimated duration: 45 minutes

The production files are in the landing zone in one shared folder:

```
<unity_catalog_raw>/FLEETOPS/PACKAGE_DELIVERY/
    fleet_vehicle.csv
    fleet_shift.csv
    fleet_movement.csv
```

`<unity_catalog_raw>` is an environment variable. ADE replaces it with the cloud storage address of the DEV or PROD landing zone when it generates the load.

### Create a load schedule

1) **Example prompt:**

> Create a load schedule `FLEETOPS_<YOUR_ALIAS>` with no cron expression and the description `#fleetops #<your_alias>`.

**Under the hood:** `ada config schedule add --name FLEETOPS_<YOUR_ALIAS> --description "#fleetops #<your_alias>"`

> **Note:** A schedule becomes a DAG in Workflow Orchestration. Using your own schedule means that running your fleet loads does not re-run your taxi loads from basic training.
{: .exercise-note}

### Review and fix the generated loads

2) Open `stg_fleetops_fleet_movement.sql`. ADA generated the COPY INTO statement from the tenant's default load template:

```sql
COPY INTO <target_schema>.<target_entity_name>
FROM
  (
    SELECT
      <target_entity_attribute_list_with_transform_cast_and_positions>
    FROM
      '<unity_catalog_raw>/<source_system_name>/<target_schema>/<target_entity_logical_name>/'
  ) FILEFORMAT = CSV FORMAT_OPTIONS(
    'header' = 'false',
    'skipRows' = '0',
    'inferSchema' = 'false',
    'delimiter' = ','
  ) COPY_OPTIONS('force' = 'false');
```

The default template fits the basic-training folder layout, not this source. Before you read on, try to find **three problems** yourself, using what you know from the earlier steps and the folder layout above.

<details markdown="1">
<summary markdown="span">The three problems</summary>

| Problem | Why it matters |
|---|---|
| The path points to `<target_schema>/<target_entity_logical_name>/` | That folder does not exist. If you only shorten the path to `PACKAGE_DELIVERY/`, every load reads **all three files**, because COPY INTO loads every file in a folder |
| `'skipRows' = '0'` | The files have a header row. It would be loaded as data and would fail on the first numeric column |
| The loads have no schedule | A load without a schedule does not appear in any DAG |

</details>

3) **Example prompt:**

> In all three FLEETOPS staging loads:
> - change the COPY INTO path to `'<unity_catalog_raw>/<source_system_name>/PACKAGE_DELIVERY/'`, and add `FILES = ('<file name>')` right after `FILEFORMAT = CSV`, using the matching file name for each entity, for example `fleet_movement.csv`
> - change `skipRows` to 1
> - set the load schedule to `FLEETOPS_<YOUR_ALIAS>`

> **Important:** Keep `<unity_catalog_raw>` and `<source_system_name>` as variables. Hard-coding the DEV storage address would break the load in PROD.
{: .exercise-note}

**Review:** check all three `.sql` files and the `schedulingName` in all three YAML files.

### Validate and push the staging package

4) Review the local files, then validate and run a dry run. **Example prompt:**

> Check all three FLEETOPS SQL sidecars and their matching STAGE YAML files. Confirm each COPY INTO path points to the folder `PACKAGE_DELIVERY/` and selects the matching single file with `FILES`, each load has `'skipRows' = '1'`, and each load has the schedule `FLEETOPS_<YOUR_ALIAS>`. Then run `ada validate`, show me the plan, and run a push dry run. Do not push yet.

### Checkpoint C1

- Each local SQL sidecar reads from `'<unity_catalog_raw>/FLEETOPS/PACKAGE_DELIVERY/'`, selects its matching single file with `FILES = (...)` and has `'skipRows' = '1'`.
- Each STAGE YAML has `schedulingName: FLEETOPS_<YOUR_ALIAS>`.
- `ada validate` passes, the plan shows only the three expected STAGE entity changes, and the push dry run succeeds.

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

7) Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`. Switch it <span class="ui-action">on</span> with the toggle, then open the <span class="ui-action">Graph view</span> and check that all three staging loads are there. A DAG that is switched off does not run.

8) **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

**Under the hood:** `ada dag trigger --dag-id FLEETOPS_<YOUR_ALIAS> --env dev --wait`

### Checkpoint C2: the DAG fails

**The DAG fails. This is expected.**

Open DEV <span class="ui-action">Workflow Orchestration</span> and find the DAG `FLEETOPS_<YOUR_ALIAS>`:

| Load | Expected state |
|---|---|
| `..._fleet_vehicle_...` | Success |
| `..._fleet_shift_...` | Success |
| `..._fleet_movement_...` | Failed |

Check the row counts in Databricks:

```sql
select 'vehicle' as entity, count(*) as row_count, 42 as expected_count
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'shift', count(*), 1981
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'movement', count(*), 0
from db_datahub_dev.staging_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT;
```

**Do not fix the movement load yet.** Finding out why it failed is the starting point of Exercise 2.

---

## Summary of Exercise 1

You have:

- installed ADA with the setup script and connected it to the design and dev environments
- brought your basic-training work into the project as YAML
- described a new source to ADA and reviewed the entities it generated
- found and fixed three problems in a generated load before they reached ADE
- deployed and run a pipeline without opening Designer

Two of three files are in staging. The third, `fleet_movement.csv`, failed. It holds the data that the expansion decision depends on.

---

Something not working? Ask ADA to help you debug it first, then ask your trainer.
