# Exercise 3 — Design the Model with ADA

Snowflake as the target database. Example alias in this document: `test`. Replace `<your_alias>` with your own alias everywhere.

---

## The task

Estimated duration: 1 hour

The FLEETOPS data is in staging. Before ADA generates any Data Vault entities, you will design the model together: ADA proposes, you review, and you decide.

This is the most important review in the course. In basic training, the model was given to you. Here, an agent proposes it — and an agent does not know your business or what already exists in your data warehouse unless you tell it. Every mistake you accept in the design is generated into hubs, links, satellites and loads in the next exercise.

**Result of this exercise:** an approved design document in `design/`, committed to Git. No entities are generated or pushed yet.

---

### Part A — Prepare

Estimated duration: 5 minutes

1) Check that ADA will give your Data Vault entities your alias. **Example prompt:**

> Show me the schema overrides and the package name postfix in `config/pipeline.yaml`. Do not change anything, and do not start any design work yet.

The Data Vault types (`HUB`, `LINK`, `SAT`, `SAT_C`, `S_SAT`) must point to `rdv_<your_alias>`, and the postfix must be `<YOUR_ALIAS>`. If something is missing, ask ADA to add it as in Exercise 1.

2) Write down the business question in your own words — in a note, **not to ADA yet**. You give it to ADA in Part B. For example:

> *PackageDelivery wants to know in which NYC areas its fleet has idle, passenger-capable capacity at times when taxi demand exists, and what that capacity would earn as taxi service.*

ℹ A model designed without a question tends to copy the source tables. A model designed for a question captures the concepts the business needs. If ADA offers to start the design already now, decline.

---

### Part B — Let ADA Propose a Model

Estimated duration: 15 minutes

**Example prompt:**

> I need a Data Vault design for the FLEETOPS staging entities in `packages/stg_fleetops_<your_alias>/`.
>
> The business question: [paste your business question from Part A]
>
> Check the existing Data Vault packages in `packages/` and reuse existing hubs where the business concept is the same. Create a design document in `design/`. Identify hubs, links and satellites with their business keys, and add an entity relationship diagram. Design the Data Vault layer only — the publish layer comes later. Do not generate any entity YAML yet.

**Under the hood:** ADA uses its Data Vault analysis skill. It reads the staging YAML and `config/pipeline.yaml`, and writes `design/dv_design_<name>.md` with a Mermaid diagram.

Open the design document. VS Code shows the diagram when you open the Markdown preview (`Ctrl+Shift+V` / `Cmd+Shift+V`).

ℹ ADA may ask you questions. Answer each in a full sentence that states your choice and the reason: *"A link, because a shift only connects a vehicle, a driver and a depot"*, not *"Link."*

Business definitions — what counts as *idle*, which vehicles can carry passengers, how earnings are calculated — are **not** decided in the Raw Data Vault. PackageDelivery gives them as business rules later in the course. If ADA asks about them, tell it to record them as open questions for the publish layer.

---

### Part C — Review the Design

Estimated duration: 25 minutes

Review the design in four short rounds. Each round follows the same pattern: **ask** ADA, **decide**, and let ADA **change** the document. Do not move on until the decision of the round is written into the document.

ℹ **New to Data Vault?** You do not need to know every rule. Ask ADA to explain any term or choice in plain language, and judge the answer against the business question: *does this help PackageDelivery compare its fleet with taxi demand in the same zones?*

#### Round 1 — Let ADA find its own problems (5 min)

**Example prompt:**

> Review the design document as a critical Data Vault reviewer. Consider the business question and the existing model in `packages/`. List the most important risks or problems, and for each, what you would change. Do not change the document yet.

Read the list. Choose the fixes you agree with and ask ADA to apply them, for example:

> Apply fixes 1 and 3. Add the others to *Open Questions*.

ℹ An agent reviewing its own work finds real problems, but not the ones caused by missing context. The next rounds check what matters most for this business question.

#### Round 2 — Connect to the existing model (5 min)

`fleet_movement.csv` and `fleet_shift.csv` contain NYC taxi zone IDs: `pickup_locationid`, `dropoff_locationid` and `depot_locationid`. Your data warehouse already has a hub for taxi zones from basic training: `rdv_<your_alias>.H_TAXI_ZONE`, with the business key `taxi_zone_key`.

Check: does the design use the existing `H_TAXI_ZONE`, or does it create a new hub such as `H_LOCATION` or `H_ZONE`?

If it creates a new hub, **example prompt:**

> The zone IDs in FLEETOPS are the same NYC taxi zone IDs as in basic training. Do not create a new hub for them. Use the existing hub `rdv_<your_alias>.H_TAXI_ZONE` from `packages/traffic_dv_<your_alias>/h_taxi_zone.yaml`. Update the design and the diagram.

<details>
<summary>Why this matters</summary>

If the design creates a new hub, the fleet data and the taxi data never meet. The comparison PackageDelivery needs — fleet capacity versus taxi demand **in the same zones** — would require joining two hubs that represent the same thing. Data Vault's strength is that a new source attaches to existing business keys.

ADA knows your existing model only if it reads it. Whether it does depends on what you point it to — checking it is your job.

</details>

#### Round 3 — Hubs and links (8 min)

**Example prompt:**

> Explain the hubs and links of the design in plain language, one by one: which business concept each represents, and why it is a hub or a link. Then answer: Is the shift a hub or a link? Is the driver needed for the business question? Does each movement have separate pickup and dropoff references to the zone hub? Is the depot connected to the shift?

What to look for in the answer:

- Every hub is something the business identifies and talks about on its own, and has an explicit business key
- Pickup and dropoff are two separate, named references to `H_TAXI_ZONE`, and the depot is connected to the shift
- **Shift** and **driver**: there is no single correct answer, but the design must say which, and why

Decide, then **example prompt:**

> Record my decisions in a *Design Decisions* section, each with a reason: [your decisions]. Update the tables and the diagram to match.

#### Round 4 — Check against the data (7 min)

A design document records assumptions. Check a few against the staged data. Run in Snowflake:

```sql
select
  count(*)                                                   as movements,
  count_if(dropoff_locationid is null)                        as dropoff_missing,
  count_if(pickup_locationid in (264, 265))                   as pickup_unknown_zone,
  count(distinct distance_unit)                               as distance_units,
  count_if(movement_start_ts like '__.__.____%')              as eu_timestamps
from ADE_TRAINING_DEV_SAAS.STAGING_<your_alias>.STG_FLEETOPS_FLEET_MOVEMENT;
```

**Example prompt:**

> I ran a data check on `STG_FLEETOPS_FLEET_MOVEMENT`. The results: [paste the result]. For each finding, tell me whether it affects the model. Add the findings to the design: under *Assumptions* if the decision is clear, under *Open Questions* if not.

ℹ Pay attention to `dropoff_missing`. Think about what happens when a hub load receives a NULL business key. You will meet this again in the next exercise. Also remember from Exercise 2: the movement timestamps are **text** in staging, and some use the `DD.MM.YYYY` format.

---

### Part D — Approve the Design

Estimated duration: 10 minutes

1) Let ADA verify the design against the checklist in Checkpoint D. Copy the checklist into the prompt. **Example prompt:**

> Check the design document against this checklist. Report pass or fail for each item, and where in the document you found it. Fix inconsistencies between the text, the tables and the diagram, but ask me before you change any decision: [paste the checklist]

2) Read the report. Check the first item yourself in the diagram — it is the one that matters most.

3) Approve the design. **Approved** means: every item in the checklist passes, and every open question is either answered or consciously left open with a reason. You approve, not ADA.

4) Commit the design. **Example prompt:**

> Commit only the design document `design/dv_design_<name>.md` to Git with the message "Approved FLEETOPS Data Vault design".

#### ✅ Checkpoint D

All of these must be true in your design document:

- The existing `rdv_<your_alias>.H_TAXI_ZONE` is used for pickup, dropoff and depot zones. **No new zone or location hub**
- Every hub has an explicit business key
- Pickup and dropoff are two separate, named references to `H_TAXI_ZONE`
- The decisions on the shift and the driver are written down with a reason
- The data findings are recorded: missing dropoff zones, unknown pickup zones (264, 265), two distance units, EU-format timestamps
- Every open question is answered or left open with a reason
- Every schema contains the alias (`rdv_<your_alias>`)
- The text, the tables and the diagram agree with each other

And the document is committed to Git.

---

### Summary of Exercise 3

You have:

- framed the business question before designing
- let ADA propose a Data Vault model and reviewed it critically
- corrected the design where the agent lacked context — above all, connecting a new source to an existing business key
- checked the design's assumptions against the data

The design is your specification for the next exercise, in which ADA generates the Data Vault entities from it.

---

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ada: command not found` | The virtual environment is not active in this terminal. Run `source .venv/bin/activate` |
| ADA creates a design already in Part A | Ask ADA to delete the design document, and start Part B with the business question |
| ADA generates entity YAML files | You did not ask for a design only. Ask ADA to delete the generated files and keep the design document |
| The design names do not contain your alias | The Data Vault overrides are missing from `config/pipeline.yaml` (Exercise 1, *Give ADA your alias*) |
| The diagram does not render | Ask ADA to check the Mermaid syntax in the design document |
| ADA keeps proposing a new zone hub | Point it to the file `packages/traffic_dv_<your_alias>/h_taxi_zone.yaml` explicitly |
