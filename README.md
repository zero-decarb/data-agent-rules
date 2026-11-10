# data-agent-rules

Rules a coding agent follows when it works on the Zero Homes warehouse. A Claude Code plugin with two skills:

| Skill | When it loads |
|---|---|
| `dbt-rules` | Working in a data-dbt checkout: adding or changing a model, seed, column, test or description; PR text; SQL a tool will run against the warehouse |
| `data-wiki-update` | After a data-dbt PR merges: update the Data Wiki |

## Install

In Claude Code:

```
/plugin marketplace add zero-decarb/data-agent-rules
/plugin install data-rules@zero-data-rules
```

Then work in a data-dbt clone as usual. To pick up a change to these rules: `/plugin marketplace update zero-data-rules`.

## Changing the rules

By PR to this repo. The skills are the only copy: no other doc or file repeats them.
