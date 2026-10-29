---
layout: exercise
module_label: "Training Q&A"
title: "ADA Self Study"
summary: "The most common questions and problems of this training, with answers. Debug with ADA first, then look here."
---

We have collected the most common questions and errors of this training in this document. When you have a problem, find your symptom here and see if the answer helps. You can skim through it now and return to it when needed.

**Start by debugging with ADA.** Describe what you see, paste the error message and ask ADA what could be done. Debugging with ADA is part of what this course teaches, and it often finds the cause quickly. Until ADA is installed, start with the setup questions below. Never paste API keys, passwords or personal data into the chat.

If ADA cannot help, look for your symptom here. Can't solve it? **Ask your trainer.**

---

## Setup and connection

**1.1 `ada: command not found`**

The virtual environment is not active in this terminal. In the project folder, run `source .venv/bin/activate` (Windows: `.venv\Scripts\activate`). Every new terminal needs this.

**1.2 The installer says `no active virtual environment detected`**

The installer was started outside a virtual environment. Create and activate one first, as in the installation steps of Exercise 1, then run the installer again.

**1.3 The installer says `requires Python >=3.10`**

The virtual environment uses an older Python, for example 3.9. Create it again with a newer one:

1. Check which Python you have: `python3 --version`. If you have no Python 3.10 or newer, install one first, within your organisation's rules.
2. Leave the old environment: `deactivate`.
3. Delete the `.venv` folder.
4. Create a new environment with the newer interpreter: `python3.12 -m venv .venv` (macOS and Linux) or `py -3.12 -m venv .venv` (Windows). Use the version you installed.
5. Activate it and check `python --version`. It must show 3.10 or newer.
6. Run the installer again.

**1.4 The installation is blocked by a firewall or security software**

ADA is installed with a script that downloads files, and a firewall or security software can block it. Try the installation again in a project folder on your computer's local hard disk, not on a network drive or in a synced folder. If that does not work, contact your trainer or your own company's IT support.

**1.5 `HTTP 401` or "authorization failed"**

The API key ID or secret is wrong, or the key lacks the required permission. Check that you copied both values from your registration email. Enter them in the terminal when ADA asks for them, not into the chat.

**1.6 "Unreachable" or a network error**

Your IP address is probably not allowed in the tenant's access rules. Contact your trainer. The Admin UI video in Exercise 1 explains why this can block an installation.

**1.7 `ada doctor` does not list the dev environment**

The dev credentials were not saved. Run `ada config credentials --env dev` again and type the key values in the terminal.

**1.8 `ada doctor` lists the dev environment twice**

The doctor reads the environments from `config/pipeline.yaml`. Ask ADA to read the file and explain which environments it contains and which one is the design environment. Do not let ADA change credentials. If the list is still odd, ask your trainer.

**1.9 `ada doctor` says that skills are not found or the skills manifest is missing**

Run `ada skills update`, as the message suggests, and run `ada doctor` again. If you have not run `ada init` in this project yet, run that first.

**1.10 `ada doctor` says `Missing config/sources.yaml`**

Look at where you are in Exercise 1. The source configuration is created when you register the FLEETOPS source, so the message can be expected before that step. If you have already described the source to ADA and the message remains, ask your trainer.

**1.11 `ada doctor` warns that `require_ada_cli_version` is not set**

This is a warning, not an error. You can continue. Do not run `ada upgrade` unless your trainer tells you to.

**1.12 `ada staging generate` says `ADE schema not found`**

`ada config pull` was not run. Run it and try again.

**1.13 A command that uses the dev environment returns `403`**

The dev key does not have the permission for that command. Ask your trainer.

**1.14 ADA previews or writes SQL that looks like Snowflake SQL**

`target_database: DATABRICKS` is missing in `config/pipeline.yaml`. Add it, as in Exercise 1, and ask ADA to preview or generate the SQL again.

---

## Packages, plan and push

**2.1 `ada push` refuses because of uncommitted changes**

The package has local changes that are not committed in Git. Commit them first, or ask ADA to do it. A Git commit in your project is not the same as a commit in Agile Data Engine; you make the latter later, when you deploy.

**2.2 The plan shows names without your alias**

`schema_overrides` or `package_name_postfix_override` is missing in `config/pipeline.yaml`. Ask ADA to set them, as in Exercise 1.

**2.3 The plan shows `~` (modify) for packages you just created**

A package with the same name already exists in Agile Data Engine. Check that every name contains your own alias, and that nobody has used the alias before in this tenant.

**2.4 `ada validate` reports `TAXIDATA_ZZALIAS` as an unknown schedule**

The placeholder `zzalias` was not replaced everywhere, or the schedule `TAXIDATA_<YOUR_ALIAS>` was not created yet. Search `packages/` for `zzalias`, replace it, and create the schedule.

**2.5 ADA cannot create or find the load schedule**

Schedules live in the shared `CONFIG_LOAD_SCHEDULES` package in the design environment, and are deployed to the runtime environment later. Ask ADA to check that `config/pipeline.yaml` has the design environment and the dev environment registered. If it still fails, ask your trainer.

**2.6 Schedule names appear in upper case**

ADA upper-cases schedule names. Use the upper-case name in your prompts.

**2.7 `ada push` fails with an unknown reference**

The packages reference each other. Push them in one batch.

**2.8 The plan shows changes in other packages**

Something else changed locally. Ask ADA to show the diff before you push.

**2.9 `git status` shows changes you did not make**

ADA's commands update files in `config/` and make their own commits. This is normal. Check what changed before you commit.

---

## Deployment and workflows

**3.1 The DAG stays `QUEUED` or does not run**

The DAG is switched off. Open DEV Workflow Orchestration, find the DAG and switch it on with the toggle.

**3.2 A deployment fails with "schedule not recognized"**

The load schedule is not deployed to that environment yet. Deploy `CONFIG_LOAD_SCHEDULES` there first, then deploy your package again.

**3.3 The deployment of a shared configuration package fails**

Another participant's change may be deploying at the same time. Wait a minute and try again. If it continues, ask your trainer.

**3.4 The DAG does not contain your new loads**

The loads were not deployed, or they depend on a different workflow. Ask ADA to list the loads of the DAG (`ada workflow describe`), or open the Graph view of the DAG in Workflow Orchestration.

**3.5 The load succeeds but a table has no new rows**

The files were probably loaded already. The target database and Agile Data Engine remember which files have been loaded. This is expected for files that loaded earlier. If rows are missing, ask ADA to check the run IDs of the staging entities. If ADA reports a permission error, ask your trainer.

**3.6 My row counts do not match**

Investigate in this order:

1. Compare the row counts of the entities that feed the one in question. If they match, continue.
2. Open the workflow log in Workflow Orchestration and check whether the load statements ran, and whether the log says that no loadable run IDs were found.
3. Review the entity and its source entities in Designer or in the YAML for missing configuration, such as the run ID load option.
4. After a fix, deploy the change and trigger the load again.

You can also paste your result to ADA, tell it which figure differs, and ask which rule could explain it.

**3.7 The DAG fails again with `CAST_INVALID_INPUT` and `TIMESTAMP` (Exercise 2)**

The type change did not reach DEV. Check that the deployment succeeded and that the plan showed the change.

**3.8 `ada-ops` cannot reach Agile Data Engine through MCP**

The MCP servers are not running. In VS Code, open `.vscode/mcp.json` and start them. Or ask ADA to use `ada workflow diagnose` instead. The `ada-ops` agent is optional in Exercise 2.

**3.9 A query in Exercise 2 fails with a permission error**

Ask your trainer. You can skip the query and continue from the next checkpoint.

**3.10 `ada deploy commit` creates a commit, but nothing is deployed**

`allow_automatic_deployment: true` is missing in `config/pipeline.yaml`. Add it, as in Exercise 1. For a commit you have already made, ask ADA to promote it to dev and start the deployment (`ada deploy promote --to DEV`, then `ada deploy start --to DEV --wait`).

**3.11 The Load Training Data notebook does not show `Copied` for all four entities, or the DAG finds no files**

Check that the alias in the first cell is exactly your alias, in the same case as in your schema names (`staging_<your_alias>`). Run the notebook again with the right alias and batch number.

**3.12 A DAG run stays `QUEUED` for several minutes**

The first run after a pause can take a few minutes to start. If it stays queued for more than ten minutes, check that the DAG is switched on in DEV Workflow Orchestration, and then ask your trainer.

---

## Data Vault

**4.1 ADA creates a design too early, or generates entity YAML when you asked for a design only**

Ask ADA to delete what it generated and keep the design document. Then repeat your prompt with the limit stated: "Design only. Do not generate any YAML yet."

**4.2 The design names do not contain your alias**

The Data Vault overrides are missing from `config/pipeline.yaml`. Ask ADA to add them, as in Exercise 1.

**4.3 The diagram does not render**

Ask ADA to check the Mermaid syntax in the design document.

**4.4 ADA keeps proposing a new zone hub, or the plan shows `+` for one**

ADA did not reuse `H_TAXI_ZONE`. Point it explicitly to the file `packages/traffic_dv_<your_alias>/h_taxi_zone.yaml` and ask it to update the design and the diagram.

**4.5 The plan does not show `~ H_TAXI_ZONE`**

ADA created the zone loads somewhere else, or not at all. Ask it to add them to the existing entity.

**4.6 `H_TAXI_ZONE` does not exist in the target database**

Your basic-training model is not deployed. Go back to Exercise 1 and bring the model packages into your project and deploy them.

**4.7 The DAG succeeds but the new tables are empty**

The staging data was loaded before the new loads existed. Ask ADA to check the run IDs of the FLEETOPS staging entities.

**4.8 The row count of `H_TAXI_ZONE` is not what the checkpoint says**

Count the rows with the replacement key `'-1'`, which is the standard replacement for a missing business key. If that count does not explain the difference, list the keys that are not in the zone lookup and find the unexpected one. Do not decide that the hub is right because the workflow run was green.

---

## Publish and analysis

**5.1 The design creates a new zone dimension**

Point ADA to `packages/traffic_publish_<your_alias>/d_taxi_zone.yaml` and ask it to reuse that dimension.

**5.2 The plan shows changes to `F_TRIP` or `D_TAXI_ZONE` that your design did not decide**

ADA modified the existing taxi entities. Ask it to revert the changes and only reference them.

**5.3 The publish tables are empty after the DAG**

The publish loads did not run in this DAG. Ask ADA to list the loads of `FLEETOPS_<YOUR_ALIAS>` (`ada workflow describe`), or check the Graph view.

**5.4 Fleet movements disappear when joined to `D_TAXI_ZONE`**

Some zone keys are not in the dimension, for example an unknown zone from Exercise 4. Decide whether they should be, and record the decision in the design.

**5.5 Taxi revenue per hour is far too high for one borough**

Trips of 0 minutes or with negative duration are probably included. Check the duration filter of rule D2.

**5.6 The query shows a borough called `Unknown` or `EWR`**

These come from the taxi zones for unknown locations and Newark Airport. They are not PackageDelivery boroughs. Leave them out of the recommendation. If you want to keep them visible, report them separately as excluded.

**5.7 Movements per borough differ from the reference**

The movements are probably counted by home borough instead of the borough of the pickup zone (rule D5).

**5.8 My revenue figure differs a little from the reference**

A difference of a few hundredths can come from rounding, or from how trips at the filter boundaries are handled. Rule D2 includes trips of exactly 1 and exactly 120 minutes. If the difference is larger than rounding, ask ADA to count the trips at the boundaries and compare them with your filter.

**5.9 A borough fails the confidence rule but the query still shows a monthly potential**

The reference shows no monthly potential for a borough that fails rule D5. A calculated figure can stay in the query as a working value, but it is not reportable. Show it as not reportable in the memo.

**5.10 The memo contains a figure you cannot find in your results**

Ask ADA where it comes from. If there is no source, remove it.

---

## Working with ADA and the AI tool

**6.1 ADA's names differ from the exercise text**

An agent does not produce the same output twice. This is fine if the checkpoint is true. Checkpoints describe what must be true, not what must be identical.

**6.2 ADA starts a task you did not ask for**

The agent often suggests a next step at the end of its answer. Follow it only if it matches the exercise. Otherwise tell ADA what you want.

**6.3 ADA's answer is based on something it cannot see**

The agent only knows what it reads. Name the files and entities in your prompt.

**6.4 ADA tries to use code preview**

Code preview is not available in this training. If ADA calls it or suggests it, stop it and ask ADA to review the local SQL files in the package instead.

**6.5 I cannot get ADA to run a database query**

ADA has no access to the target database. Ask ADA to write the query, run it yourself in the database, and paste the result back.

**6.6 Pasted query results arrive as one line of text**

The results were copied from a rendered page. Copy them from the result grid of the database, so the columns stay separate.

**6.7 Where do I see the commands ADA runs?**

In the terminal of your AI tool. Read a command before you approve it. If the terminal is not visible, open the terminal panel of VS Code or the command view of your AI tool.

---

## Missing something?

If your problem is not here, tell your trainer. We add the common ones to this document.
