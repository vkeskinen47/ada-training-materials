# Exercise 1 — ADA Setup and New Source to Staging

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## How these exercises work

These exercises are different from basic training. There are no click-by-click instructions. You give instructions to the AI agent **ADA**, and ADA writes the YAML, runs the `ada` commands and talks to Agile Data Engine for you.

Your job is to:

- **Specify** — tell ADA clearly what you want
- **Review** — read what ADA proposes before you accept it, and reject it when it is wrong
- **Verify** — confirm the result against the checkpoints in this document

Each step gives you an **example prompt**. You do not have to use it word for word. Each step also lists what ADA should do **under the hood**, so you can tell when it goes off track. ADA asks for permission before running terminal commands. Read the command before you approve it.

ADA often suggests what to do next at the end of its answer. You may follow its suggestions when they match the exercise. When they do not, tell ADA what you want instead.

ℹ **The standard routine.** Whenever you change entities, take them to ADE in this order. ADA knows the routine; your job is to read each result before the next step.

| Step | Command | What you check |
|---|---|---|
| 1. Validate | `ada validate` | No errors |
| 2. Plan | `ada plan` | Only the changes you expect |
| 3. Dry run | `ada push --dry-run` | ADE accepts the change |
| 4. Push | `ada push` | The upserted count matches the plan |
| 5. Deploy | `ada deploy commit --wait` | Every package shows `SUCCESS` |

ℹ AI agents do not always produce the same output twice. If your entity or load names differ slightly from this document, that is fine — the checkpoints describe what must be **true**, not what must be identical.

### Choose your AI tool

This document uses **GitHub Copilot in VS Code** as the reference. ADA also supports Claude Code.

| Tool | How to start ADA | Notes |
|---|---|---|
| GitHub Copilot in VS Code | Open Copilot Chat in **Agent** mode and select the `ada-developer` agent | Reference tool for these exercises |
| Claude Code | Run `ada init --tool claude` instead of `ada init` in Part A. Then start `claude` in the project folder | Supported. Example prompts work as written |
| Other agents that read `AGENTS.md` (e.g. Cursor, Codex) | Open the project folder in the tool | May work, but not tested in this training. Your trainer cannot support them |

ℹ **Costs.** ADA itself is free to use in this training, but the AI tool is not. You need your own paid subscription (for example GitHub Copilot Pro or Claude Pro), and the model usage of these exercises is charged to that subscription. See the course prerequisites for details.

---

## The task

Estimated duration: 2 hours

PackageDelivery has not yet decided whether to enter taxi operations. To prepare the decision, the company wants to compare taxi demand with its own fleet. The fleet management system **FLEETOPS** exports three files:

| File | Content |
|---|---|
| `fleet_vehicle.csv` | One row per vehicle: type, passenger seats, cost per mile, home borough |
| `fleet_shift.csv` | One row per driver shift: vehicle, depot, start and end time |
| `fleet_movement.csv` | One row per vehicle movement between two NYC taxi zones |

In this module you will install ADA, connect it to your ADE tenant and bring the FLEETOPS files into the staging layer.

**Connection details:**

| | |
|---|---|
| Tenant ID | `s7922169` |
| Installation name | `datahub` |
| API keys | A **design** key and a **dev** key (key ID and secret for each). Your trainer sends them to you by email |

**Requirements:** Python 3.10 or newer, Git, and an AI tool from the table above.

ℹ **Did you do the basic training in this tenant?** If you did, you will pull your own packages in Part A. If you did not, or your packages are gone, you will clone the training model packages instead. Both options are described in Part A, under *Bring your basic-training work into the project*. Option 2 takes about 30 minutes more.

---

### Part A — Install ADA and Connect to ADE

Estimated duration: 40 minutes

#### Watch: Admin UI

[TBD: link to the Admin UI video]

In this training your trainer provides the API keys. In your own organisation, keys are created in the ADE Admin UI. Watch the video so you know where a key comes from, what permissions it needs, and why an installation can fail.

| Who does what | Training | Your own organisation |
|---|---|---|
| Creates the API keys | Trainer | Admin UI user |
| Allows your IP address in Access Rules | Trainer | Admin UI user |
| Installs ADA | You | You |

#### Install

1) Download the starter project to your local machine:
https://www.agiledataengine.com/hubfs/Training/ada_starter_project.zip

2) Extract the ZIP file and open the `ada_starter_project` folder in VS Code. It contains a `README.md` and the FLEETOPS sample files in `source_data/fleetops/`.

NOTE! Do not rename or move the sample files. ADA derives entity names from the file names.

3) Open a terminal in VS Code and create and activate a virtual environment. The installer refuses to run without one.

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
```

4) Run the installer:

```bash
curl -fsSL https://artifacts.saas.agiledataengine.com/install.sh | bash
```

Windows (PowerShell): `irm https://artifacts.saas.agiledataengine.com/install.ps1 | iex`

5) When prompted, enter tenant ID `s7922169`, installation name `datahub`, environment **`design`**, and your **design** key ID and secret. The installer checks the key before it installs anything.

| If you see | It means | What to do |
|---|---|---|
| `HTTP 401` / authorization failed | The key is wrong or lacks permissions | Check that you copied the key ID and secret correctly |
| Unreachable / network error | Your IP address is not allowed | Contact your trainer |

6) When the installer prints `✓ ada-cli <version> installed`, scaffold the project:

```bash
ada init                           # Claude Code users: ada init --tool claude
```

Your sample files in `source_data/` are kept.

7) Add the dev key. Answer the tenant and installation prompts, then paste the **dev** key ID and secret.

```bash
ada config credentials --env dev
```

This also registers `dev` as a runtime environment in `config/pipeline.yaml`.

8) Fetch the tenant's configuration, which includes entity types, load templates and schedules:

```bash
ada config pull
```

9) Put the project under version control. `ada push` refuses to run without Git.

```bash
git init
git add -A
git commit -m "Initial ADA project"
```

ℹ From now on, ADA uses Git for you. `ada pull` and `ada push` create their own commits, and `ada push` refuses to run while the package has uncommitted changes. Some commands also update files in `config/`, such as the cached schemas. If `git status` shows changes you did not make, that is usually why.

#### ✅ Checkpoint A1

```bash
ada doctor
```

The output must end with both lines:

```
✓ ADE environment reachable: s7922169/datahub/design
✓ ADE environment reachable: s7922169/datahub/dev
```

#### Optional: connect your AI tool to ADE with MCP

MCP (Model Context Protocol) lets the AI agent query ADE directly — for example, which loads failed in a workflow run and why. In Exercise 2, the `ada-ops` agent can use it to diagnose failures. **Everything in this course also works without MCP**, so the choice is yours.

1) Generate the configuration for both environments:

```bash
ada mcp setup --env design --env dev
```

This creates `.vscode/mcp.json` (VS Code) and `.mcp.json` (Claude Code). Both files contain placeholders only, never your keys, and both are excluded from Git.

2) Connect:

- **VS Code:** open `.vscode/mcp.json` and click **Start** above each server (`ade-mcp-design`, `ade-mcp-dev`). VS Code asks for the key ID and secret of each environment and stores them securely.
- **Claude Code:** set the environment variables `ADE_DESIGN_API_KEY_ID`, `ADE_DESIGN_API_KEY_SECRET`, `ADE_DEV_API_KEY_ID` and `ADE_DEV_API_KEY_SECRET` before you start `claude`.

3) Test it. **Example prompt:**

> Use the ADE MCP server to check that you can reach ADE, and tell me the name of my API key.

#### Give ADA your alias

From now on, you work through ADA. Start ADA in your AI tool (see *Choose your AI tool*).

Many trainees share the same ADE tenant, so every schema and package needs your alias. Set it for all entity types at once, so that nothing you create later lands in a shared default schema.

**Example prompt:**

> My alias is `<your_alias>`. Configure the project so that:
> - SOURCE entities go to schema `src_<your_alias>`
> - STAGE entities go to schema `staging_<your_alias>`
> - HUB, LINK, SAT, SAT_C and S_SAT entities go to schema `rdv_<your_alias>`
> - DIM and FACT entities go to schema `publish_<your_alias>`
> - all package names get the postfix `<YOUR_ALIAS>`

**Under the hood:** ADA adds `schema_overrides` and `package_name_postfix_override` to `config/pipeline.yaml`.

**Review:** open `config/pipeline.yaml` and check the values yourself.

ℹ Schemas are lowercase (`staging_test`), but package names are always **uppercase** (`STG_FLEETOPS_TEST`), whatever case you type the postfix in.

#### Bring your basic-training work into the project

In a later exercise you will connect the fleet data to the `H_TAXI_ZONE` hub from basic training. ADA needs that hub as YAML in your project, and the hub needs data in Snowflake.

Choose **one** option:

- **Option 1** — you completed the basic training in this tenant and your packages still exist
- **Option 2** — you have no basic-training packages in this tenant

##### Option 1 — Pull your own packages

**Example prompt:**

> Pull all my packages from ADE. They are tagged `#<your_alias>`.

**Under the hood:** `ada pull --tag <your_alias>`

##### ✅ Checkpoint A2 (Option 1)

- `packages/` contains your basic-training packages, for example `taxidata_staging_<your_alias>/`, `traffic_dv_<your_alias>/` and `traffic_publish_<your_alias>/`
- `packages/traffic_dv_<your_alias>/` contains `h_taxi_zone.yaml`

Open `h_taxi_zone.yaml`. This is the hub you built by clicking in Designer. It is now a text file that you can version, review and change through an agent.

##### Option 2 — Clone the training model packages

The starter project contains a copy of the complete basic-training pipeline in `model_packages/`: `TAXIDATA_STAGING_ZZALIAS`, `TRAFFIC_DV_ZZALIAS` and `TRAFFIC_PUBLISH_ZZALIAS`. Wherever an alias belongs, the copy has the placeholder `zzalias` (`ZZALIAS` in uppercase names). You will turn the copy into your own packages, deploy them and load the taxi data.

ℹ The copy contains no entity IDs, so pushing it can only create new entities. It cannot change anyone else's packages.

1) Create a schedule for the taxi pipeline. **Example prompt:**

> Create a load schedule `TAXIDATA_<YOUR_ALIAS>` with no cron expression and the description `#taxidata #<your_alias>`.

2) Create your packages from the copy. **Example prompt:**

> Copy the three packages in `model_packages/` into `packages/`. In the copies, replace `zzalias` with `<your_alias>` and `ZZALIAS` with `<YOUR_ALIAS>` everywhere — in folder names, file contents and SQL files. Do not change anything in `model_packages/`.

ℹ The placeholder also renames the schedule to `TAXIDATA_<YOUR_ALIAS>`. Until you have replaced it and created that schedule, `ada validate` reports `TAXIDATA_ZZALIAS` as an unknown schedule.

**Review:** ask ADA to check the result:

> Search `packages/` for `zzalias`, ignoring case, and validate my three new packages. Then show me the plan for them.

- The search finds **nothing**
- Validation passes
- The plan shows **only `+` (create) lines**: 19 entities in three packages, and `0 to modify, 0 to delete`

ℹ The plan has no satellite current views (`S_..._C`). ADE creates them automatically for each satellite.

3) Push and deploy. **Example prompt:**

> Push my three taxi packages to ADE in one push. Then commit `CONFIG_LOAD_SCHEDULES` and wait for the deployment, and then commit my three taxi packages in the order staging, DV, publish and wait for each deployment.

**Under the hood:** `ada push -p TAXIDATA_STAGING_<YOUR_ALIAS> -p TRAFFIC_DV_<YOUR_ALIAS> -p TRAFFIC_PUBLISH_<YOUR_ALIAS>`, then `ada deploy commit ... --wait` for each package

4) Load the taxi files into the landing zone. Run in Snowflake, as in basic training:

```sql
CALL ade_training_dev_saas.staging.load_files_to_blob ('<your_alias>', 1);
CALL ade_training_dev_saas.staging.load_files_to_blob ('<your_alias>', 2);
```

NOTE! The alias is case-sensitive.

5) Open DEV Workflow Orchestration, find the DAG `TAXIDATA_<YOUR_ALIAS>`, switch it **on** with the toggle, and check in the Graph view that your loads are there. Then trigger it. **Example prompt:**

> Trigger the DAG `TAXIDATA_<YOUR_ALIAS>` in dev and wait for it to finish.

##### ✅ Checkpoint A2 (Option 2)

- The DAG `TAXIDATA_<YOUR_ALIAS>` succeeds
- `packages/traffic_dv_<your_alias>/` contains `h_taxi_zone.yaml`
- This query returns 266, as in basic training:

```sql
select count(1) as row_count, 266 as expected_count
from ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.H_TAXI_ZONE h
join ADE_TRAINING_DEV_SAAS.RDV_<your_alias>.S_TAXI_ZONE_C s on h.dv_id = s.dv_id;
```

---

### Part B — Add the FLEETOPS Source

Estimated duration: 25 minutes

The `FLEETOPS` source system already exists in ADE (`CONFIG_SYSTEMS`). Your task is to describe the source files to ADA so that it can create the SOURCE and STAGE entities.

ℹ **Why sample files?** `source_data/fleetops/` contains a **sample extract** — the first week of January — not the full production files. This is common in real projects: the source owner gives you a sample, and the production data arrives later in the landing zone. ADA infers column types from the sample. Keep this in mind. It will matter later.

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

ℹ DOUBLE is a floating-point type. It is not ideal for money. If you want to practise, ask ADA to change `delivery_revenue_usd` to a DECIMAL type. This step is optional.

#### ✅ Checkpoint B

**Example prompt:**

> Validate the FLEETOPS staging package and show me what pushing it would change in ADE.

**Under the hood:** `ada validate packages/stg_fleetops_<your_alias>/ --recursive`, then `ada plan -p STG_FLEETOPS_<YOUR_ALIAS>`

Both of these must be true:

- Validation reports `✓ All 6 files valid`
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

If you see your alias in every schema and in the package name, your alias configuration from Part A works. If the plan shows anything to modify or delete, stop and find out why before you continue.

---

### Part C — Load the Files into Staging

Estimated duration: 45 minutes

The production files are in the landing zone in one shared folder:

```
<snowflake_azure_stage>/FLEETOPS/PACKAGE_DELIVERY/
    fleet_vehicle.csv
    fleet_shift.csv
    fleet_movement.csv
```

`<snowflake_azure_stage>` is an environment variable. ADE replaces it with the DEV or PROD stage when it generates the load.

#### Create a load schedule

1) **Example prompt:**

> Create a load schedule `FLEETOPS_<YOUR_ALIAS>` with no cron expression and the description `#fleetops #<your_alias>`.

**Under the hood:** `ada config schedule add --name FLEETOPS_<YOUR_ALIAS> --description "#fleetops #<your_alias>"`

ℹ A schedule becomes a DAG in Workflow Orchestration. Using your own schedule means that running your fleet loads does not re-run your taxi loads from basic training.

#### Review and fix the generated loads

2) Open `stg_fleetops_fleet_movement.sql`. ADA generated the COPY INTO statement from the tenant's default load template:

```sql
COPY INTO <target_schema>.<target_entity_name>
FROM
  (
    SELECT
      <target_entity_attribute_list_with_transform_cast_and_positions>
    FROM
      '<snowflake_azure_stage>/<source_system_name>/<target_schema>/<target_entity_logical_name>/'
  ) FILE_FORMAT=(type='csv' skip_header=0 field_delimiter=',' FIELD_OPTIONALLY_ENCLOSED_BY='"' COMPRESSION='AUTO');
```

The default template fits the basic-training folder layout, not this source. Before you read on, try to find **three problems** yourself, using what you know from Part B and the folder layout above.

<details markdown="1">
<summary markdown="span">The three problems</summary>

| Problem | Why it matters |
|---|---|
| The path points to `<target_schema>/<target_entity_logical_name>/` | That folder does not exist. If you only shorten the path to `PACKAGE_DELIVERY/`, every load reads **all three files**, because COPY INTO loads every file in a folder |
| `skip_header=0` | The files have a header row. It would be loaded as data and would fail on the first numeric column |
| The loads have no schedule | A load without a schedule does not appear in any DAG |

</details>

3) **Example prompt:**

> In all three FLEETOPS staging loads:
> - change the COPY INTO path to `'<snowflake_azure_stage>/<source_system_name>/PACKAGE_DELIVERY/<file name>'`, using the matching file name for each entity, for example `fleet_movement.csv`
> - change `skip_header` to 1
> - set the load schedule to `FLEETOPS_<YOUR_ALIAS>`

ℹ Keep `<snowflake_azure_stage>` and `<source_system_name>` as variables. Hard-coding the DEV stage would break the load in PROD.

**Review:** check all three `.sql` files and the `schedulingName` in all three YAML files.

#### Push and check the generated SQL

4) **Example prompt:**

> Validate the FLEETOPS package, show me the plan, and push it to ADE.

**Under the hood:** `ada validate`, `ada plan`, `ada push -p STG_FLEETOPS_<YOUR_ALIAS>`

5) **Example prompt:**

> Show me the SQL that ADE generates for the FLEETOPS staging loads.

**Under the hood:** `ada code preview -p STG_FLEETOPS_<YOUR_ALIAS>`

#### ✅ Checkpoint C1

In the generated SQL:

- The path of the movement load reads `'<snowflake_azure_stage>/FLEETOPS/PACKAGE_DELIVERY/fleet_movement.csv'`. ADE has already replaced `<source_system_name>` with `FLEETOPS`. `<snowflake_azure_stage>` stays unresolved in the preview. That is expected: it is an environment variable, and ADE resolves it only when the load runs in DEV or PROD.
- The file format contains `skip_header=1`.
- The SELECT list maps file columns by position (`$1 AS movement_id`, `$2 AS shift_id`, ...), and the technical attributes get generated values (`'FLEETOPS' AS stg_source_system`).

#### Deploy and run

6) Your new schedule lives in the shared `CONFIG_LOAD_SCHEDULES` package. It must be deployed before your package, just like in basic training.

**Example prompt:**

> Commit `CONFIG_LOAD_SCHEDULES` with the message "Add `FLEETOPS_<YOUR_ALIAS>` schedule" and wait for the deployment. Then commit my FLEETOPS staging package and wait for that deployment too.

**Under the hood:** `ada deploy commit -p CONFIG_LOAD_SCHEDULES -m "..." --wait`, then the same for `STG_FLEETOPS_<YOUR_ALIAS>`

ℹ `ada deploy commit --wait` may print `✗ Deployment RUNNING` even though every package below it shows `SUCCESS`. This only means the overall deployment had not closed when ADA stopped waiting. Trust the package lines.

7) Open DEV Workflow Orchestration and find the DAG `FLEETOPS_<YOUR_ALIAS>`. Switch it **on** with the toggle, then open the Graph view and check that all three staging loads are there. A DAG that is switched off does not run.

8) **Example prompt:**

> Trigger the DAG `FLEETOPS_<YOUR_ALIAS>` in dev and wait for it to finish.

**Under the hood:** `ada dag trigger --dag-id FLEETOPS_<YOUR_ALIAS> --env dev --wait`

#### ✅ Checkpoint C2 — the DAG fails

**The DAG fails. This is expected.**

Open DEV Workflow Orchestration and find the DAG `FLEETOPS_<YOUR_ALIAS>`:

| Load | Expected state |
|---|---|
| `..._fleet_vehicle_...` | ✅ Success |
| `..._fleet_shift_...` | ✅ Success |
| `..._fleet_movement_...` | ❌ Failed |

Check the row counts in Snowflake:

```sql
select 'vehicle' as entity, count(*) as row_count, 42 as expected_count
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_VEHICLE
union all
select 'shift', count(*), 1981
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_SHIFT
union all
select 'movement', count(*), 0
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT;
```

**Do not fix the movement load yet.** Finding out why it failed is the starting point of Exercise 2.

---

### Summary of Exercise 1

You have:

- installed ADA with the official installer and connected it to the design and dev environments
- brought your basic-training work into a Git repository as YAML
- described a new source to ADA and reviewed the entities it generated
- found and fixed three problems in a generated load before they reached ADE
- deployed and run a pipeline without opening Designer

Two of three files are in staging. The third, `fleet_movement.csv`, failed. It holds the data that the expansion decision depends on.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| Installer: `no active virtual environment detected` | Run step 3 of Part A first |
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` in the project folder. Every new terminal needs it |
| Installer: `requires Python >=3.10` | Recreate the virtual environment with a newer Python |
| `ada staging generate`: `ADE schema not found` | `ada config pull` was not run (Part A, step 8) |
| `ada push` refuses because of uncommitted changes | Commit your changes first, or ask ADA to do it |
| `ada doctor` does not list the dev environment | Run `ada config credentials --env dev` again |
| The plan shows names without your alias | Check `schema_overrides` and `package_name_postfix_override` in `config/pipeline.yaml` |
| PROD deployment fails: schedule not recognized | Deploy `CONFIG_LOAD_SCHEDULES` to PROD first |
| The DAG stays `QUEUED` and never runs | The DAG is switched off. Switch it on in Workflow Orchestration |
| The plan for your new taxi packages shows `~ modified` lines | A package with the same name already exists in ADE. Check that every name contains your own alias |
| `ada validate` reports `TAXIDATA_ZZALIAS` as unknown | The placeholder was not replaced everywhere, or the schedule `TAXIDATA_<YOUR_ALIAS>` was not created yet |
