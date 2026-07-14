# woodseer-refactor

Working repo for the Woodseer-app refactor/improvement list. This is currently a **documentation-only** effort: no code changes are made here yet, we're building the list of what should change and why, with AI assistance (Eddard).

Confluence: https://optionmetrics.atlassian.net/wiki/spaces/DATAQA/pages/5320245252/Woodseer+refactoring

## Contents

- `reports/` — audit reports of the current `Woodseer-app` codebase (date rules engine, analyst-task checks), cited to file:line, compiled 2026-07-07.
- `reports/analyst-tasks-live-report.html` — live data report: real analyst-task volume by type, filterable by date range (defaults to 2020-05-01 to current), with automation-opportunity findings. Data snapshot as of the date in its own footer, not auto-refreshing — ask Eddard to regenerate it against current production Woodseer data when it goes stale.
  - Also published as a claude.ai artifact for easy viewing: https://claude.ai/code/artifact/d8ff14c3-808d-408a-8115-29c3c91ad797 (published under Jose's account). If you're editing this file in your own Claude session and want to update that same link rather than mint a new one, try passing this URL as the artifact `url` param when republishing — unconfirmed whether editor-level share access actually grants that, so if it fails/mints a new URL for you, that's expected: fall back to publishing your own link and note it here, and/or hand the change to Jose to republish to the URL above. The committed HTML file in this repo is the real source of truth regardless of which artifact URL(s) end up live.
- `DECISIONS.md` — running log of finalized refactor decisions. This is the source of truth for what's already been agreed, and what gets pushed to Confluence.

## Workflow

- Every Claude Code session started in this repo pulls the latest changes first (`.claude/settings.json` SessionStart hook).
- File or decision changes made during a session are committed back to this repo.
- If a change contradicts something already in `DECISIONS.md`, it goes to a branch + PR for Jose or Lev to review, instead of a direct commit.
- Finalized decisions get pushed to the Confluence page above, with confirmation from Jose first each time.
