# ds-pipeline-review as a Microsoft Copilot agent

A Copilot agent version of the Claude skill in `skills/ds-pipeline-review/`. It has the same review rules, split into one main text and six knowledge files to fit Copilot's agent builder.

## Setting it up

1. **Name:** `DS Pipeline Review` (or anything you like).
2. **Description:** `Reviews Python, pandas, Polars, SQL, Snowflake, dbt, ML and forecasting code for bugs that silently produce wrong data, leakage, and pipeline failures.`
3. **Instructions:** paste the whole content of `instructions.md`. It is about 6,600 characters (about 1,500 tokens), so it fits whether the 8,000 limit counts tokens or characters.
4. **Knowledge:** upload all six files in `knowledge/`. The instructions refer to them by these exact file names, so don't rename them:
   - `review-method.md`
   - `checklist-pandas-polars-python.md`
   - `checklist-sql-snowflake-dbt.md`
   - `checklist-analytics-ml-forecasting.md`
   - `checklist-orchestration.md`
   - `checklist-library-numerics.md`
5. **Optional:** if your builder offers a code interpreter capability, turn it on. The agent then runs small checks on a few rows of data and can label findings "confirmed" from those checks. Without it, the agent traces the code by hand.
6. **Optional:** if the builder lets you turn off general web knowledge or "only use specified sources", leave web knowledge on. The checklists are a guide, not the only source of library knowledge.

## Using it

Paste or attach the code to review: a diff, the old and new versions of a file, whole files, a notebook or SQL. Add a sentence on what the change is meant to do, and mention the pandas or Snowflake connector version if you know it. The agent can't open git or GitHub itself, so give it everything it should see.

## Differences from the Claude skill

- No git or `gh` commands. The agent reviews only what you paste or attach.
- Verification is done with Copilot's code interpreter when it's available, and only on self-contained snippets. Otherwise the agent traces the code by hand.
- Copilot searches knowledge files in chunks rather than reading them whole. That's why the main workflow, severity rules and output format are in the instructions, and each checklist item is written to stand on its own.
- `SOURCES.md` (research notes and the revision log) isn't included, because the agent doesn't need it.

## Keeping it in sync

The checklists in `knowledge/` are copies of the skill's checklists, with only the cross-references renamed and the differential-testing step adapted. When you change a checklist in the skill, copy the change here too.
