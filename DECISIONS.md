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

---

## 2026-07-09 — Analyst-flag-dismissed tasks resurface with no way to suppress them

**Decision:** Two checks don't respect a prior analyst dismissal the way an analyst would expect:
- `DividendFrequencyChangeCheck` (`app/tasks/dividend_frequency_change_check.rb`) never references `analyst_flag` at all — it flags any consecutive dividends with differing `frequency`, so a genuinely irregular payer trips it on every newly-forecasted future dividend with no possible flag-based suppression.
- `IncompleteFiscalYearCheck` (`app/tasks/incomplete_fiscal_year_check.rb:9`) only excludes `FLAGS_MEANING_NO_FORECAST` (`ACQUIRED`/`CEASED_TRADING`/etc.), not `FLAGS_MEANING_FORECAST_NOT_REQUIRED` (`NO_PATTERN`/`NO_DIVIDENDS`/`INSUFFICIENT_HISTORY`) — so setting `NO_PATTERN`, the natural flag for an irregular payer, doesn't suppress it either.

**Live evidence:** share_id 79369 (SecurityID 211145) — `DIVIDEND_FREQUENCY_CHANGE` (task_type 13) dismissed by two different analysts (user 2 on 2026-02-12, user 137 on 2025-10-15), then 6 fresh instances appeared 2026-07-09, one per newly-rolled-forward forecast dividend (2026 through 2028), each showing the same Triannually↔Irregularly oscillation. `INCOMPLETE_FISCAL_YEAR` (task_type 34) dismissed by user 137 (2025-10-15), 2 fresh instances appeared the same day (fiscal years 2026, 2027).

**Open question, not yet root-caused:** this security's `flag_set_at` is stamped 2026-02-12 10:35:38 (6 seconds after that day's dismissal) but `analyst_flag` is currently empty — a flag was set and later cleared without `flag_set_at` being reset. Worth investigating separately; not required for the two findings above, which hold regardless of this security's specific flag history.

**Source:** `WOODSE_1.DOC` (Mark, "Analyst flag re-surfacing tasks" item).

**Status:** Root cause confirmed for two specific checks, live example identified (share 79369). Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Remove "DividendMax Dividends Missing Pay Date" dashboard widget

**Decision:** Confirmed safe, self-contained removal. Widget lives at `app/views/admin/dashboard/index.haml:8-10`, backed by `dashboard_controller.rb:8` (`Dividend.in_portfolio(TypeId::DIVIDENDMAX_PORTFOLIO_ID).extant.missing_pay_date...`), rendered by `_missing_pay_date.haml` as a plain read-only table (Security, Dividend, Ex-Date) with no action buttons or workflow — matches "never actioned" exactly. Checked for shared dependencies: the `Dividend.missing_pay_date` scope (`dividend.rb:36`) and the dedicated `admin/reports/_dividend.haml` partial it renders are each used in exactly one place — this widget. Nothing else breaks by deleting it. Note: this dashboard already has four other widgets commented out (`@securities_by_paying_clients`, three AGM ones) — this is the next one in an already-started cleanup that was never finished.

**Source:** `WOODSE_1.DOC` (Mark, "DividendMax dividends missing pay date" item).

**Status:** Confirmed removal candidate, no code changes made yet (documentation-only phase).

---

## 2026-07-09 — Remove Symphony Users admin page from Dashboard/nav

**Decision:** Confirmed unused. `Admin::SymphonyUsersController` (index/new/create/edit/update) is a full CRUD admin page, linked from the nav (`shared/_drop_down_menu.haml:25-29`) — not a dead link, but a dead *feature*. Live data: 12 `symphony_users` (all approved), latest created/updated 2022-05-10; 33 `symphony_watchlists` rows, latest 2022-04-01. Zero activity in over 4 years.

**Scope note:** this decision covers only the admin management page. The broader Symphony integration (`Symphony::PushNotification`, fired on every dividend series publish via `PostSecurityUpdateWorker`, plus a separate GraphQL watchlist API) is a distinct, larger question not addressed here — that worker broadcasts to all approved Symphony users for S&P 500 securities (doesn't even query `symphony_watchlists`) and is gated entirely behind `ENV['ENABLE_SYMPHONY_PUSH_NOTIFICATION']`. Whether to remove that too is a separate decision.

**Source:** `WOODSE_1.DOC` (Mark, "Remove Symphony users from Dashboard" item).

**Status:** Confirmed removal candidate for the admin page specifically, no code changes made yet (documentation-only phase).

---

## 2026-07-09 — Maiden Dividend check fires on non-cash first distributions

**Decision:** `MaidenDividendCheck#has_one_dividend?` (`app/tasks/maiden_dividend_check.rb:8-14`) checks dividend count, `extant?`, recency, and `source` — but never checks `disbursement_type`. A security's first-ever "dividend" record can be a non-cash distribution (stock/scrip dividend, capital repayment, etc.) and still trigger a "Maiden Dividend" task, incorrectly treating "first-ever distribution happens to be non-cash" the same as "company just started paying real cash dividends." Fix: exclude non-`CASH` disbursement types (`disbursement_attributes.rb`: `CASH = 1`) from qualifying as a maiden dividend.

**Live evidence:** of 959 total `MAIDEN_DIVIDEND` tasks ever created, 130 (~13.6%) were not cash: 70 `STOCK` (share-based/scrip dividends — the closest literal match to Mark's "share splits" wording), 47 `CAPITAL_REPAYMENT`, 5 `CASH_AND_STOCK`, 5 `NIL_DIVIDEND` (arguably the clearest case — firing on something that wasn't a real distribution at all), 3 `INSTALLMENT`. Three recent concrete examples (task IDs 4903725, 4887079, 4847717; May-July 2026): each has a blank `amount` and a populated `shares_received` (1.0, 3.0, 1.0) — unambiguously non-cash.

**Source:** `WOODSE_1.DOC` (Mark, "Maiden dividend task scope" item).

**Status:** Root cause confirmed, quantified at scale (13.6% of all-time volume), live examples identified. Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — China/India settlement cycle change (T+0 / T+1) — planning notes, not yet actioned

**Request:** Mark wants China shares set to T+0 and India shares set to T+1 settlement. Explicitly flagged by Mark as needing careful planning — this entry documents what that planning needs to cover, no change has been made.

**Mechanism:** `SettlementPeriod` (`app/models/settlement_period.rb`) — effective-dated rows per `stock_exchange_id`. `DateCalculator#ex_date_record_date_diff` (`date_calculator.rb:21-24`) picks the most recent row with `effective_date <= date`, defaulting to `1` day if no row exists at all (`record&.days || 1`) — a silent default, not a logged warning (already flagged in the original date-rules audit).

**Current live state (checked 2026-07-09, zero rows for any of these exchanges):**
- China equity exchanges: Shanghai Stock Exchange (id 59), Shenzhen Stock Exchange (id 62). (Excluded: China Unlisted Bond id 160, Inter-Bank Bond Market id 161 — not equity exchanges, not in scope.)
- India equity exchanges: National Stock Exchange of India (id 2), BSE Ltd. (id 95), Metropolitan Stock Exchange of India (id 28). (Excluded: India Unlisted Bond id 239, Reserve Bank of India id 240.)
- **None of these 5 exchanges have any `settlement_periods` row on file.** Both countries are currently at T+1 purely via the silent fallback default, not through deliberate configuration.

**What this means for the request:**
- **China → T+0** is a genuine behavior change (1 → 0), needs a new row inserted for *both* Shanghai and Shenzhen separately (they're independent exchange IDs, not one "China" setting).
- **India → T+1** already matches the current accidental default — this is really "formalize the existing default into an explicit row" rather than a behavior change, but still worth doing properly (across NSE, BSE, and Metropolitan) rather than continuing to rely on the undocumented fallback.

**Planning considerations (why Mark is right to flag this as needing care):**
1. **`effective_date` choice matters** — these are effective-dated rows; whichever date is chosen is when the new value starts applying going forward. It does not retroactively touch already-settled historical dividends.
2. **Override interaction with learned date rules** — per the original audit (`date_forecaster.rb:279-282,105-115`, `should_override_date_rule?`), once a `settlement_periods` row exists covering the forecast date, it overrides any per-security *learned* ex-date/record-date pattern for that exchange. Adding the China row doesn't just correct the default — it will supersede whatever date-rule pattern the algorithm has independently learned from real historical EDI data for every Shanghai/Shenzhen-listed security. Blast radius is larger than "change one number."
3. Multiple exchanges per country need separate rows — this isn't a single-row change per country.

**Side note, separate from this decision:** confirmed live that `settlement_periods` exists in production Postgres, but is missing from the checked-out `db/schema.rb` (only the legacy singular `stock_exchanges.settlement_period` column shows there, unpopulated for all 5 exchanges above). The local schema file is stale relative to production; worth a `db:schema:dump` refresh independent of this decision.

**Source:** `WOODSE_1.DOC` (Mark, "China / India settlement cycle" item).

**Status:** Planning notes only — mechanism and current state confirmed live, no `settlement_periods` rows have been inserted, no code or data changes made (documentation-only phase, and this one specifically involves a live data write that's out of scope for this repo's current phase regardless).

---

## 2026-07-09 — "Security Has Insufficient Forecasts" automation already exists

**Finding (Priority-2 automation list, working bottom-up):** Mark's ask — "regen fixes in most cases, provided analyst flag = none and there are two years of history; automatically run a regen if both conditions are met" — is **already implemented**. `InsufficientForecastWorker` (`app/workers/insufficient_forecast_worker.rb`) runs daily (`config/clock.rb`: `every(1.day, 'Insufficient Forecast Worker', at: '**:50')`) and force-regenerates (`RegenerateSecurityWorker.perform_later(id, 'force_autopublish')`) any covered security whose forecast horizon has fallen short, for both the standard 2-year case and a 4-year case for index members.

**Live evidence:** zero outstanding `MISSING_FORECAST` (task_type 9) tasks exist anywhere in the system right now, across every `analyst_flag` value — not just the flag=nil case Mark described. The daily worker keeps forecast horizons topped up proactively, so the task type rarely gets a chance to surface a gap at all.

**One real discrepancy worth reconciling with Mark, not a bug:** the worker's `flagged_expecting_forecasts` scope (`share.rb:89`) covers `analyst_flag IS NULL` **or any flag not in `FLAGS_MEANING_NO_FORECAST`** — broader than "flag = none." It also force-regenerates securities flagged `NO_PATTERN`/`NO_DIVIDENDS`/`INSUFFICIENT_HISTORY`/`TAKEOVER_IN_PROGRESS`. Arguably correct (those flags don't mean "stop forecasting"), but Mark should confirm he's aware this already exists and that the broader scope is intentional, rather than treating this as a from-scratch build.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Security Has Insufficient Forecasts" item).

**Status:** No action needed — automation already exists and is confirmed working live. Recommend closing this out with Mark rather than building anything.
