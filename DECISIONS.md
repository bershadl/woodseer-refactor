# Decisions Log

Finalized refactor/improvement decisions for Woodseer-app. Each entry is something Jose (and/or Lev) has actually agreed to, not just a raised idea. New proposals get checked against this list before being added; anything that contradicts an existing entry goes to a PR instead of a direct commit.

Format per entry: date, decision, source (which report/discussion it came from), status.

## 2026-07-09 — Duplicate Dividend check needs frequency-awareness

**Decision:** `DuplicateDividendCheck` (`app/tasks/duplicate_dividend_check.rb:20`) must check dividend frequency before applying its `< 11 day` proximity threshold, not instead of it. As written, any two dividends within 11 days sharing an amount or flag get flagged — so every weekly-paying ETF trips this on every run, since consecutive weekly dividends are always ≤7-9 days apart. Not a rare false positive; guaranteed for that entire class of security.

**Source:** `WOODSE_1.DOC` (Mark, "Duplicate Dividend" item) + `reports/analyst-tasks-report.html`. Live-confirmed 2026-07-09: 5 weekly-ETF duplicate tasks fired in one 5-minute batch (task IDs 4904548-4904552), all "Week N vs Week N+1" collisions.

**Status:** Root cause confirmed, fix direction agreed. Not yet implemented (this repo is documentation-only for now).

---

## 2026-07-09 — Regen can create a second partial declaration for the same dividend event

**Decision:** Confirmed as a real bug, not just a theoretical concern. When EDI issues two corporate-action rows for the same real dividend (e.g. original notice + amendment) before either is fully declared, `DividendGenerator#create_forecasts_from_partial_corporate_actions` (`app/forecasting/dividend_generator.rb:69-77`) processes them in sequence: the first row correctly matches and updates the existing partial-declaration dividend, but immediately marks that dividend's ID as "assigned" (`instance_assigned`, `app/versioning/dividend_instance_manager.rb:14-16`), which excludes it from matching for the rest of the run. The second row for the same event then fails to find a match and creates a brand-new dividend instance instead of updating the first.

**Live evidence:** share_id 289684, dividends 9446266 (Interim, position 1) and 9446267 (Final, position 2), both frequency=BIANNUALLY, both still undeclared (no `declaration_date`). Two signals confirm same-event duplication rather than legitimate distinct dividends: identical amount to 5 decimal places (9.12004 on both), and adjacent EDI event IDs (remote_id `DIV-4679239` / `DIV-4679241`, gap of 2) — contrast with a same-security same-frequency pair on share 277849 that turned out legitimate (different amounts, event IDs 86 apart, normal quarterly progression).

**Source:** `WOODSE_1.DOC` (Mark, "Partial declaration handling" item).

**Status:** Root cause confirmed, live example identified. Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Draft series created and discarded daily for securities with no real dividend changes

**Decision:** `CorporateActionImporter#security_ids` (`app/algo/corporate_action_importer.rb:84-87`) triggers a full `RegenerateSecurityWorker.perform_later(id, 'autopublish')` for any covered security touched by a new corporate action, filtering out only `eventcd: 'AGM'` — every other corporate action type (ticker changes, ISIN updates, listing changes, etc., regardless of dividend relevance) forces a complete regen cycle. When the regenerated forecast matches what's already published, `RegenerateSecurityWorker#publish_or_discard_series` (`regenerate_security_worker.rb:87-95`) destroys the just-created draft. Needs a filter so only corporate-action types that can actually affect a dividend forecast trigger regen.

**Live evidence:** security_id 122504 — `DividendSeries` PaperTrail history shows 4899 automated `destroy` events (system-triggered, `whodunnit` blank) across 1645 distinct calendar days from 2021-08-02 to 2026-05-06, vs. only 13 events total attributed to real users (whodunnit 2 and 137) over the same 4.5-year span. Essentially pure automation churn with almost no genuine analyst interaction.

**Source:** `WOODSE_1.DOC` (Mark, "Draft series management" item).

**Status:** Root cause confirmed, live example identified (security 122504). Fix not yet designed or implemented (documentation-only phase).
