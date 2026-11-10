---
name: data-wiki-update
description: Update the Data Wiki (apps/data-wiki in the zero-decarb/prototypes repo) after a data-dbt change. Use after a data-dbt PR that touched a model, seed, column or YAML description has merged to main: regenerate the table pages, update the metric, rule and source pages whose meaning changed, open one prototypes PR, merge it when checks pass, and report it in the handoff.
---

# Data Wiki update

Every dbt change that touches a model, seed, column or YAML description updates the Data Wiki, as part of the same task. The table pages (column descriptions and each column's YAML tests; singular tests in `tests/` don't appear) are generated from data-dbt `main`, so the work starts right after the data-dbt PR has merged, and the author of that PR does it (whoever merges it). You need access to zero-decarb/prototypes; without it, say so in the handoff and ask the data owner.

## Which pages are which

| Folder in `apps/data-wiki/site/content/` | Written by |
|---|---|
| `tables/` and `_dbt.json` | **Generated** from data-dbt. Never edit by hand. |
| `metrics/`, `rules/`, `sources/` | By hand |

Each metric and rule page opens with a plain-English definition. That definition belongs to the data owner (named in the `dbt-rules` skill): leave it alone. Until the owner writes it, it is a `[placeholder for ...]`, shown as a dashed box. The rest of a page (Counts, Doesn't count, Details) is facts, and each fact traces to dbt or a recorded decision.

## Steps

1. **Merge the data-dbt PR first**, then bring a data-dbt checkout to `main`: `git checkout main && git pull`, and note `git rev-parse HEAD`.
2. **Branch prototypes.** Clone zero-decarb/prototypes (private) if you haven't, and branch off an up-to-date `main`.
3. **Regenerate the table pages.** Needs Node 24, no warehouse credential:
   ```sh
   cd apps/data-wiki/site
   npm ci --ignore-scripts
   node scripts/generate-tables.mjs <absolute path to the data-dbt checkout> <commit sha>
   npm run check
   ```
   A table page is the relation as tools read it, and its slug is the layer, a dash and the relation name: the L3 model `deals_l3` is `bizops_l3.deals`, page `l3-deals`; `l1-hubspot_deals`, `l3-deal_customer_communications` (underscores kept). Ephemeral models get no page. If `npm run check` fails, a hand-written page names a table or column dbt no longer has (after a rename or drop): fix that page, not the check.
4. **Update metric, rule and source pages when the meaning changed.** Read `site/content/metrics`, `rules` and `sources` for anything the change touches: a count, an exclusion, a date rule, a seed row, a column a page names. Then:
   - Update Counts, Doesn't count and Details; put who decided and when in Decided.
   - Keep the frontmatter `models:` list (table slugs such as `l3-deals`) and links (`../tables/<slug>.md#<column>`) pointing at real tables and columns; the check fails on a broken one. A new table a page relies on goes in `models:`.
   - Don't write a fact you can't trace to dbt or a decision. Ask the data owner.
   - A meaning change that makes a plain-English definition wrong: don't edit it, flag it in the handoff.
5. **Open one PR** in prototypes. It touches only `apps/data-wiki`: never `pnpm-lock.yaml` or the root `package.json` (that would redeploy every app). The PR runs the `Data wiki check` and deploys a preview service `data-wiki-pr<n>`.
6. **Merge it once the checks pass, with no review:** `gh pr merge <n> --squash`. Never bypass a failing check. The merge deploys `data-wiki` only.
7. **Handoff line** in the data-dbt PR or the issue: `Wiki: <PR link>, pages changed: <list>`, or `Wiki: no change` with the reason (for example, the change only touched CI).

A scheduled job may also regenerate and merge the table pages from `main` as a safety net. Don't rely on it: step 3 is still yours, and it never edits metric, rule or source pages, so step 4 is too.
