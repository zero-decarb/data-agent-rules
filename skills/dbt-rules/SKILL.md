---
name: dbt-rules
description: House rules for the data-dbt repo (zero-decarb/data-dbt, the MotherDuck warehouse layers bizops_l1 / l2 / l3). Use whenever working in a data-dbt checkout, before adding or changing a model, seed, column, test or YAML description, before retiring a model, before writing a PR description for a dbt change, and before writing SQL that a dive, dashboard, board or Flight will run against the warehouse (a business rule it needs becomes a dbt model here, not code in the tool).
---

# dbt rules

Scope: the data-dbt repo and anything that reads the warehouse (dives, dashboards, boards, Flights). Run from a data-dbt checkout; paths below are in it, and its `README.md` has setup, CI and the layers table.

Data owner: [placeholder for who owns rule conflicts]. Wherever this skill says "the data owner", that is who.

Never push to `main` and never commit tokens, `.env` or `profiles.yml`. The company's own web apps keep and compute their own operational data in their own store; don't build models or seeds for an app's own needs.

Each rule says why, so you can judge cases it doesn't cover.

## The layers

| Layer | Folder | Prod schema | What it is | Who reads it |
|---|---|---|---|---|
| L0 | `models/l0/` (sources only) | data-engine's (`hubspot`, `podium`, ...) | Raw loads. Never edited. | dbt only |
| L1 | `models/l1/<source>/`, `seeds/` | `bizops_l1` | One cleaned table per source object; seeds | dbt only |
| L2 | `models/l2/` | `bizops_l2` | One row per real thing | Tools, after L3, when the question is about the thing itself |
| L3 | `models/l3/` | `bizops_l3` | Rules and measures | Tools, first choice |

Stage builds add `_stage` (and `_pr<n>` for a PR) to each schema name.

## Rules

1. **Pick the layer by what the model is.** Renaming, typing or deduping one source table is L1. Combining sources into one row per real thing is L2. Applying a business rule, measuring or aggregating is L3. A model that reads only L0 is L1, whatever it's called. Folder = schema (`models/l1`, `l2`, `l3`), set by `+schema` in `dbt_project.yml`. *Why: a table's layer tells a reader what it can be trusted for. Names come from the file name (or `alias`), so check a new name isn't taken in another layer.*
2. **One definition per business rule, and it lives in dbt.** A rule (qualified, program vs Direct, installed, business hours, SLA windows, week and quarter boundaries) is an L3 or L2 model that every tool reads. A step only one model uses can be `materialized: ephemeral` (no table). *Why: a rule retyped in a tool drifts from the original, and then two numbers for the same thing both look right.*
3. **Lists, targets and thresholds are rows, not code.**
   - Small, slow-changing lists you want reviewed in git are seeds: CSV in `seeds/`, entry in `seeds/seeds.yml` with `column_types` and tests. Where a seed is a rule, carry `decided_by`, `decided_on` and `source` columns.
   - Single values (hours, minutes, days, limits, dates, tags) are rows in the `business_rules` seed. Read one in SQL with `{{ business_rule('key', 'type') }}`. Tools read them through `bizops_l3.business_rule_values`.
   - Seeds are L1, so tools never read a seed: they read the L2 or L3 model in front of it.
   - A value whose past matters gets new `effective_from` rows, never an overwrite: git keeps who and when, not a queryable past value.
   - Data an app or Flight writes, or someone edits often, is not a seed. It lives in `bizops_external_data`, outside dbt.
   *Why: a constant in code is invisible to everyone who needs to question it.*
4. **Fewest tables.** Before proposing a table or column, read the columns of the existing L3 and L2 tables and test the question against real data (`dbt show` or a stage build; with no warehouse access, say which checks you couldn't run). Order of preference:
   - **The query alone**, for a derivation: a null in a milestone, a subtraction of two dates, a stage on the deals table. Don't add a column only because one tool re-derives something once.
   - **A column on an existing table.** Passing an L2 column through to the L3 table of the same row grain is normal: L3 is what tools read first, so they shouldn't have to join back to L2 for a fact about the same thing. Test that the copy equals its source.
   - **A new table**, only for a new grain or a new source, never one table per tool. A plan that adds tables lists each with a "why not an existing table" line, naming the tables checked. This rule wins over the README's "extracts shaped for one dive" wording.
   *Why: every table is something to keep correct, document and explain.*
5. **Don't build around missing architecture, and don't touch what isn't ours.**
   - Never write to `data_warehouse` directly: dbt builds it.
   - L0 belongs to data-engine. A missing raw table or column is a request to its owner; once loaded, declare it in `models/l0/sources.yml`.
   - No new schemas, databases or shares without the data owner's OK.
   - Check whether a model exists or is planned before building it.
   *Why: these are shared and hard to undo.*
6. **Every model and seed has tests.** A key column gets `unique` and `not_null`; a joined id gets `relationships`; a fixed set of values gets `accepted_values`. A rule that must hold across tables gets a singular test: a SQL file in `tests/` named `<model>_<what>.sql` that selects the rows breaking the rule (no rows = pass). Test the rule, not only the shape: when a column is meant to equal another table's, test that. In dbt 1.10 the keys are `data_tests:` and `arguments:` (copy a neighbouring YAML). *Why: a quiet wrong number in a warehouse is found by whoever is reading it.*
7. **Descriptions in the house style.** The `.yml` sits beside the model (L1: one per source folder; L3 may share a file for related models). Describe with a `>` folded block, plain English, no filler. First sentence is the grain ("One row per ..."). Then, where the text states a rule, the rule with its numbers and who decided it and when (`<who>, <date>`); and what a reader should use instead, if the table is a step toward another. Describe a column when its name isn't enough: units, time zone, what null means, which date. A rule too long for YAML goes in a `{# #}` comment at the top of the SQL. *Why: the Data Wiki's table pages are generated from these descriptions.*
8. **Change or retire with the readers in mind.** Before renaming, retyping or dropping a column or model, find every reader: other models and `tests/` (search data-dbt), then dives, boards and Flights, which live outside data-dbt (ask the data owner who reads the table). Don't edit a tool you weren't asked to. A rename-only change must give identical results; a change to a rule gets a before/after diff with the rule named. To retire a model or seed, add its name to `retired_relations` in `dbt_project.yml`, with a comment line (what, when, who OK'd, where it moved), so `drop_retired` removes the leftover tables; a schema-qualified entry (`bizops_l2.callbacks`) drops one layer's copy only. *Why: readers outside data-dbt fail silently.*
9. **Rule conflicts are the data owner's call.** If two tools define a rule differently, don't pick one. Ask. If a request itself goes against a rule (a new table where a column would do, a rule typed into a tool), do the part that fits and say in the PR what you left out and why; ask the data owner only if that leaves the request unmet.

## Tools that read the warehouse

Dives, boards and Flights read L3 first, then L2 (when the question is about the thing itself and L3 doesn't carry it), and never L0, L1, a seed or the retired `bizops.<name>` views. If the rule, join or measure a tool needs isn't in L3 or L2, the answer is a dbt model in data-dbt (rules 1 to 4), then the tool reads it; it is not SQL or JS in the tool. If it's too big to build now, say so and say what the tool does in the meantime. A tool you touch ends up reading a higher layer than you found it, with numbers checked: identical for a rename, a before/after for a changed rule. A reporting rule typed into a tool is a defect.

## Build, stage and PR

- Setup and CI are in `README.md`. A PR runs `dbt build (stage)` into schemas named for that PR (`bizops_l3_stage_pr<n>`), tests included. Stage output is scratch: no tool reads it. A merge to `main` runs the prod build, and prod also builds hourly at :30.
- A local build: pass your own suffix so you don't collide with CI, for example `dbt build --target stage --select <model>+ --vars '{hubspot_source_schema: hubspot_stage, hubspot_source_table: deals, bizops_schema_suffix: _stage_<you>}'`.
- Branch, commit, open a PR against `main`; never push to `main`. Wait for `dbt build (stage)` to pass. Don't merge on a red check.
- Terraform deploys (`deploy/`, `infra/`) are not part of model work.

## PR description and handoff

Lead with the layers verdict in plain words, before anything else: which layer the change is in, which tables it reads by layer, and whether a table was added. For example: "Layers: adds `x` to L3 `deals` from L2 `deals`. No new table." A change to a YAML description is a dbt change (the wiki shows it), so it gets the verdict too. A change with no model SQL (tests or descriptions only) says that in the verdict. A change with no model, seed or YAML change (CI, README) says "Layers: no model change" and stops there.

Then:
- **What the target tables don't cover.** When you point a reader at a table, list the columns the reader uses that the table lacks (owner, amount, name ...); dives aren't in data-dbt, so ask the requester which columns the reader uses. If a missing column keeps the reader on L1 until something is built, that is the headline.
- **Rule changes:** the rule, its before/after numbers, who decided it and when.
- **New tables:** each with its "why not an existing table" line.
- **Tests added**; the stage build result (a link, or "running" until it finishes); readers affected; anything still reading L0 or L1.
- **Wiki:** `data-wiki-update` runs after the merge.
- When you repointed a reader, end with: "Layers: reads L3 x, L2 y; still on L0/L1: z."


## Before you write a model, check

- [ ] Which layer is it, by what the model does (rule 1)?
- [ ] Does the rule or measure already exist in an L3 or L2 table? Did I test the question against existing columns (rule 4)?
- [ ] Is a list, target or threshold typed into SQL instead of a seed or `business_rules` row (rule 3)?
- [ ] Does a new table have a new grain or source, with its "why not an existing table" line?
- [ ] Key tests, relationships, and a singular test for the rule itself (rule 6)?
- [ ] Description with grain, rule, who decided and when (rule 7)?
- [ ] Readers found; before/after for a changed rule; retired names in `retired_relations` (rule 8)?
- [ ] After the merge: `data-wiki-update`. If the change moves a number the Weekly Lookback shows, that is updated in the same piece of work: [placeholder for who owns the Lookback update].
