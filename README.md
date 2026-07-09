# woodseer-refactor

Working repo for the Woodseer-app refactor/improvement list. This is currently a **documentation-only** effort: no code changes are made here yet, we're building the list of what should change and why, with AI assistance (Eddard).

Confluence: https://optionmetrics.atlassian.net/wiki/spaces/DATAQA/pages/5320245252/Woodseer+refactoring

## Contents

- `reports/` — audit reports of the current `Woodseer-app` codebase (date rules engine, analyst-task checks), cited to file:line, compiled 2026-07-07.
- `DECISIONS.md` — running log of finalized refactor decisions. This is the source of truth for what's already been agreed, and what gets pushed to Confluence.

## Workflow

- Every Claude Code session started in this repo pulls the latest changes first (`.claude/settings.json` SessionStart hook).
- File or decision changes made during a session are committed back to this repo.
- If a change contradicts something already in `DECISIONS.md`, it goes to a branch + PR for Jose or Lev to review, instead of a direct commit.
- Finalized decisions get pushed to the Confluence page above, with confirmation from Jose first each time.
