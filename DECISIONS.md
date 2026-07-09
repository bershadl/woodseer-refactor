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

**Checked for a hidden-manual-labor pattern (per the Dividend Cut correction above) — ruled out.** Full history of task_type 9: only **15 tasks ever existed**, all created in a 3-week window in May 2018, all processed within that same window (13 by the same user who does the Dividend Cut manual clearing, 2 by another user) — then genuine silence for 8+ years, not a continuous drip. Git history confirms the timeline isn't coincidental cover: `MissingForecastCheck` was introduced 2017-11-27, but `InsufficientForecastWorker` (the daily auto-regen fix) wasn't added until **2019-11-20** — 18 months after the May 2018 incident, with zero new tasks firing in that gap even before the dedicated worker existed. This is a one-time historical cleanup, not an ongoing hidden backlog — unlike the Dividend Cut case, this automation's "zero outstanding" state reflects ~6.5 years of the worker actually doing the job.

**One real discrepancy worth reconciling with Mark, not a bug:** the worker's `flagged_expecting_forecasts` scope (`share.rb:89`) covers `analyst_flag IS NULL` **or any flag not in `FLAGS_MEANING_NO_FORECAST`** — broader than "flag = none." It also force-regenerates securities flagged `NO_PATTERN`/`NO_DIVIDENDS`/`INSUFFICIENT_HISTORY`/`TAKEOVER_IN_PROGRESS`. Arguably correct (those flags don't mean "stop forecasting"), but Mark should confirm he's aware this already exists and that the broader scope is intentional, rather than treating this as a from-scratch build.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Security Has Insufficient Forecasts" item).

**Status:** No action needed — automation already exists and is confirmed working live. Recommend closing this out with Mark rather than building anything.

---

## 2026-07-09 — Dividend frequency should be inferred from date spacing, not trusted from the EDI code

**Decision:** `DividendFrequencyIdentifier.identify` (`app/calculators/dividend_frequency_identifier.rb:38-50`) maps the raw EDI `field2` code directly to a frequency label via a `case` statement covering only 7 of the ~25 codes documented in its own comment block (`WKL`/`MNT`/`BIM`/`QTR`/`SMA`/`TRM`/`ANL`) — everything else (`REG`, `VAR`, `ARR`, `ONE`, `11M`, `35D`, `DLY`, etc.) silently falls into `else return Dividend::IRREGULARLY`.

**But the deeper problem, confirmed live, is worse than incomplete code coverage:** on share 79369 (SecurityID 211145, the same security from the item-4 finding), the *actual* ex-date spacing since 2023 has been consistently ~90-108 days (quarterly-ish) or ~166-184 days (the corresponding gap), genuinely regular — but the EDI `field2` codes across those same events are `SMA, SMA, SMA, IRG, TRM, IRG, QTR, TRM, IRG`. Most of these are *already-mapped, valid* codes (SMA/TRM/QTR), not just unmapped catchalls — meaning the vendor itself codes a consistent real-world cadence inconsistently, event to event. Expanding the `case` statement to cover more EDI codes would not fix this specific oscillation, since the instability isn't primarily about missing codes.

**Mark's proposed fix is the right one:** compute frequency from the actual spacing between historical dividend dates and pick the best-fit frequency, rather than trusting the EDI-provided code per event. This would produce a stable classification for genuinely regular payers regardless of how inconsistently the vendor codes each individual event, and would directly resolve the `DividendFrequencyChangeCheck` resurfacing problem from the earlier finding at its source rather than just suppressing the symptom.

**Unconfirmed side note:** the class's own comment documents the weekly EDI code as `WKY` (line 37) while the `case` checks for `'WKL'` (line 41) — possible typo, not verified live (this security's data has no weekly-coded events to test against).

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Dividend Frequency Does Not Match Previous" item).

**Status:** Root cause confirmed and quantified with a concrete live example; Mark's proposed approach (spacing-based inference) is well-founded, not just a nice-to-have. Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — "Delisted Security with Forecasts" automation: not viable

**Decision (Jose's call):** Flagged impossible, not pursued further. `DelistedSecurityForecastCheck` (`app/tasks/delisted_security_forecast_check.rb`) fires on `primary_listing.delisted? && has_forecast?`. Mark's proposal was to automate covered-status removal once delisting is confirmed against a reliable source, contingent on that source actually being trustworthy.

**Partial rationale found before this was called off:** only two code paths in the entire app ever assign `Listing#list_status` — `shares_creator.rb:28` (sets `ACTIVE_LISTING` on creation) and `listing_status_change_processor.rb:36` (sets `DELISTED`, triggered by an EDI `LSTAT`/`field3: 'D'` corporate action). There is no code path anywhere that reverses a delisting back to active — so if that signal is ever wrong, there's currently no automated recovery, only a manual fix. Live verification of whether this has actually happened (checking `Listing`'s PaperTrail history for delisted→active reversals) was not completed before Jose ruled this out.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Delisted Security with Forecasts" item).

**Status:** Not viable, per Jose. No further investigation planned.

---

## 2026-07-09 — EDI/Analyst amount mismatch auto-resolution for ADRs — threshold not yet decided

**Request:** Mark proposes auto-deleting and regenerating the forecast for ADR (`DR` security_type) amount mismatches below a threshold, rather than always creating an `EDI_DATA_DOES_NOT_MATCH_MANUAL` task. Threshold value is an open decision, not yet made — this entry documents the data gathered to inform it.

**Current mechanism:** `EdiDividendComparator#different_amount?` (`edi_dividend_comparator.rb:39-44`) is an exact-equality check today — any difference at all, including rounding artifacts, triggers the task. No tolerance exists.

**Live distribution (105 total ADR amount-mismatch tasks, all-time):**
- 23.8% within 1%, 39.0% within 2%, 61.0% within 5%, 74.3% within 10%
- Median difference: 3.2%. Average: 29.0% — heavily right-skewed by outliers, not representative of the typical case.

**The outliers are not just "bigger" — they're two distinct, identifiable failure modes, not noise:**
1. **Systematic/recurring ratio error:** share_id 68882 shows a ~406% mismatch on three separate occasions (2023-12, 2024-12, 2025-07), each time EDI ≈ 5.06x the analyst amount. Not FX drift — a persistent conversion/ratio problem specific to this security. Auto-regenerating would reproduce the same wrong value again, not fix anything.
2. **Decimal/unit error:** share_id 68348 — EDI=0.3049 vs Analyst=30.49, a clean 100x factor. Looks like a cents-vs-dollars or stray `*100`/`/100` bug, not a real business discrepancy.

**Recommendation to inform the threshold decision (not a final call):**
- A pure percentage cutoff around 5% would auto-resolve a meaningful majority (61%) of historical cases, in a range consistent with ordinary FX-timing noise.
- But large-and-recurring mismatches on the same security, and suspiciously clean-ratio mismatches (100x, possibly 5x/10x patterns), should be excluded from auto-resolution regardless of magnitude — these indicate a structural data/config problem that regen would mask, not fix. Worth a repeat-offender guard (e.g., skip auto-resolve if this security had 2+ mismatches in the trailing N months) in addition to whatever percentage threshold is chosen.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "EDI Data Does Not Match Analyst Data" item).

**Status:** Threshold decision still open — data gathered above is meant to inform it. No code or automation built yet (documentation-only phase).

---

## 2026-07-09 — Stale Analyst Flag (Dividend Cut) resumption automation — high-value: Mark has done this manually 4,828 times

**Request:** Mark proposes that once a `DIVIDEND_CUT`-flagged security's resumption is confirmed via new EDI data, a script should clear the flag, insert zero-dividend entries for the years nothing was paid (to satisfy the 2-year-history requirement), and trigger a regen — replacing the current `StaleAnalystFlagCheck` (`app/tasks/stale_analyst_flag_check.rb`) task with an automated fix.

**Existing building block:** `AnalystDividendInstanceCreator` (`app/dividends/analyst_dividend_instance_creator.rb`) already supports creating a "nil dividend" instance (amount=nil, `disbursement_type: NIL_DIVIDEND`) — the existing manual tool for this kind of backfill. It requires an existing draft series and explicit per-year params though; it doesn't itself compute which years are missing, so the automation needs new logic for that part.

**Correction to initial assessment:** first pass checked only the `STALE_ANALYST_FLAG` analyst-task detection path and found just 2 tasks ever, neither for `DIVIDEND_CUT` — that looked like a zero-occurrence scenario. It wasn't. Checking `Share`'s PaperTrail history directly for `analyst_flag` transitions away from `DIVIDEND_CUT` (4) instead of the task table: **4,828 occurrences since 2021-02-09**, every single one attributed to `whodunnit = 2` — confirmed against the `users` table to be **Mark himself**, the same Mark who wrote this backlog item. Timestamps run 1-4 minutes apart in clusters (e.g. 15 of them across 2026-07-07 through 2026-07-09 alone) — the pace of one person manually working through a queue, not a batch script. `flag_set_at` ages at time of clearing range from ~2 months to ~2.5 years.

**Why the automated check misses nearly all of these:** `StaleAnalystFlagCheck`'s gating (`skip_processing?`) requires both a `current_series` and `previous_series` to exist, and a genuine EDI/LIMEBURNER-sourced `DECLARED` dividend to appear in the current series — conditions that apparently almost never align for this scenario in practice. Mark is finding and resolving these some other way entirely (most likely watching the EDI feed directly), completely bypassing the app's own task system. If he stopped doing this, the backlog would likely just accumulate silently.

**Side finding, confirmed at scale:** every one of these updates also bumps `flag_set_at` to the current timestamp even though the flag is being *cleared*, not set — the systemic version of the dangling-`flag_set_at` anomaly first noticed on share 79369 in the item-4 finding. Worth fixing as part of whatever touches this code path.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Stale Analyst Flag (Dividend Cut)" item).

**Status:** Root cause and real-world scale confirmed — this is one of the stronger automation candidates on the list (real, high-volume, currently 100% manual). Fix/automation not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Dividend Currency Mismatch auto-approval — real ongoing burden, but Mark's named scope may be too narrow

**Mechanism:** `DividendCurrencyMatchCheck#create_task?` (`app/tasks/dividend_currency_match_check.rb:12-14`) is an exact-match check — any dividend whose `currency_code` differs from `@security.declaration_currency` fires a task, no exceptions.

**Live evidence — same hidden-ongoing-labor pattern as the Dividend Cut item, at even larger scale:** 3,865 total tasks all-time, **100% processed** (0 currently outstanding), spanning 2017-11-27 through just 3 days ago (2026-07-06) — continuous activity for nearly 9 years, not a one-time cleanup. The same person as the Dividend Cut item accounts for 2,220 of 3,865 (57%), active across the entire span; another individual contributes 909 more. This is a large, real, sustained manual burden.

**Scoping wrinkle — Mark's named pairs are real but not the majority:** counting both directions, China/Hong Kong (HKD↔CNY) is 736 occurrences and US/Canada (CAD↔USD) is 407 — combined ~29.6% of all-time volume. The rest is spread across other recurring pairs, several individually larger than either named pair's single direction: GBP↔USD alone is 655 (bigger than the entire US/Canada total), plus EUR↔USD (324), EUR↔GBP (163), various Nordic pairs, HKD/USD, MXN/USD, CNY/USD, CLP/USD, BMD/USD. A "currency blank" bucket of 356 tasks didn't match the parsing pattern used here — unexamined, worth a look before implementation.

**Recommendation:** if the rationale for auto-approving is "known, structurally-expected currency-switching pairs," several of these other pairs (especially GBP/USD) likely qualify under the same logic and represent comparable or greater volume. Worth deciding the actual scope with Mark rather than narrowly building only the two pairs he named — the data suggests the real opportunity is broader.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Dividend Currency Does Not Match Declared Currency" item).

**Status:** Root cause and real-world scale confirmed; scope (which currency pairs qualify for auto-approval) is an open question for Mark to weigh in on, not yet decided. Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Significant Dividend Amount Change auto-approval — largest, cleanest automation candidate found

**Mechanism:** `BigDividendAmountChangeCheck` (`app/tasks/big_dividend_amount_change_check.rb`) fires whenever a dividend's adjusted amount more than doubles or less than halves versus the matching prior-year dividend (same position/frequency/currency) — with no check on the triggering dividend's `status`/`source`. An algorithm making a risky projection and EDI confirming a real declared change are treated identically today.

**Live evidence — largest volume and most sustained burden found in this review:** 26,355 total tasks all-time, **100% processed** (0 outstanding), spanning 2017-11-28 through **literally this morning** (2026-07-09 09:51) — continuous activity for ~9 years with no sign of slowing. The same person as the Dividend Cut / Currency Mismatch items accounts for 18,320 of 26,355 (69.5%); another individual contributes 6,172 more.

**Status/source breakdown of the triggering dividend confirms Mark's exact proposed rule cleanly, with almost no caveats needed:**
- **DECLARED + EDI: 20,962 (79.5%)** — exactly "EDI-confirmed declared change," Mark's proposed auto-approve condition.
- DECLARED + ANALYST: 916 — a manual (non-EDI) declaration, correctly excluded by the "EDI-confirmed" wording.
- FORECAST + ALGORITHM/ANALYST: 228 combined — the actual risk case this check exists to catch (an unconfirmed projection making a big jump on its own), correctly excluded.
- REMOVED_FORECAST: 2,394 combined, and DELETED_BY_PROVIDER/CANCELLED_BY_ISSUER: 481 combined — dividends being cleared or cancelled outright, a different scenario from "amount changed," reasonably left as manual review.

**Recommendation:** this is the strongest, cleanest automation candidate identified across the whole Priority-2 list — highest all-time volume, most sustained ongoing burden, and the proposed rule (`status == DECLARED && source == EDI`) maps almost exactly onto the 79.5% that should be safe, while the remaining ~20.5% naturally falls into categories that clearly should stay manual under the same logic Mark already articulated. Worth prioritizing first among the amount/currency-type automations.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Significant Dividend Amount Change" item).

**Status:** Root cause and real-world scale confirmed, proposed rule validated against live data at 79.5% coverage. Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Analyst Estimate Follows a Recent Ratio auto-delete-and-regen — clean mechanism, real ongoing volume

**Mechanism:** `RatioWithManualEstimatesCheck#find_ratio` (`app/tasks/ratio_with_manual_estimates_check.rb:9-16`) already detects the exact condition Mark describes: a real `ShareRatio` record with a significant multiplier (`inverse_multiplier < 0.8 || > 1.2`) whose `effective_date` falls between the security's last declared dividend and its last analyst-sourced forecast — meaning the analyst's manual estimate predates a split/consolidation and is now stale. Unlike the ADR-mismatch or currency-pair items, there's no threshold or scope ambiguity here — the check's existing trigger condition already *is* the precise condition for Mark's proposed fix (delete the stale estimate, regen). The only new work is wiring the fix instead of just flagging it.

**Live evidence — same ongoing-burden pattern, smaller scale:** 1,824 total tasks all-time, 100% processed, spanning 2018-05-16 through 2026-06-25 (about 2 weeks ago) — roughly 228/year, lower frequency than the Currency Mismatch or Amount Change items but still a real, continuous burden. Same two people as those items account for 67.9% and 24% of resolutions respectively.

**Cross-reference for whoever implements this:** the "regen" step this automation would trigger runs through the same `DividendGenerator`/`RegenerateSecurityWorker` pipeline already implicated in the item-2 (partial-declaration duplicate) and item-3 (draft-series churn) findings earlier in this doc — worth being aware of those existing quirks rather than treating regen as a black box when designing this.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Analyst Estimate Follows a Recent Ratio" item).

**Status:** Root cause confirmed, mechanism is architecturally clean and ready to build (no open threshold/scope decision, unlike several other Priority-2 items). Fix not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Invalid KR/JP Record Date: two distinct findings (scope gap + false-positive pattern)

**Scope gap, separate from the auto-approval question:** despite the task type name (`INVALID_ASIAN_RECORD_DATE`) and Mark's own wording ("KR or JP"), `AsianRecordDateCheck#skip_security?` (`asian_record_date_check.rb:17-19`) only references `StockExchange::JAPAN_STOCK_EXCHANGE_IDS` — `KOREA_STOCK_EXCHANGE_IDS` (a separate, distinct constant) is never checked. Korea-listed securities never get this validation at all, contradicting the apparent intent of both the task type's name and Mark's description.

**False-positive pattern, live-confirmed:** the check assumes every Japan dividend's record date must be the exact calendar end-of-month (`d.record_date != d.record_date.end_of_month`) — but this doesn't hold for every Japan-listed security. Share 88769's full dividend history back to 2022, every occurrence **`status=DECLARED, source=EDI`** (real, vendor-confirmed, not estimated), shows record date on the **20th of the month** every single time, alternating Final/Interim every 6 months — a genuine, 6+ year established convention, not end-of-month at all. The two "invalid" tasks flagged for this security (2028-11-20, 2029-05-20) are the algorithm's own forecasts *correctly continuing that exact real pattern* — the check flags correct behavior as an error. At least 7 other securities in the live data show the same recurring day-of-month signature (10th, 15th, 20th, 30th, each consistently repeating in 6-month pairs), suggesting this is a systemic pattern, not one outlier security.

**Refined framing of Mark's proposed fix:** rather than "auto-approve dates that fall under the acceptable pattern of end-of-month" (which reads as adding one more hardcoded exception), the data suggests the better fix is comparing a flagged forecast's day-of-month against **that specific security's own historical EDI-confirmed record-date pattern**, and only flagging genuine deviations from *that* security's established convention — not deviations from a blanket end-of-month assumption that several real securities simply don't follow.

**Live volume:** 2,061 total tasks all-time, 100% processed, spanning 2018-08-01 through literally this morning (2026-07-09 10:48) — same continuous ongoing-burden pattern as the other Priority-2 items.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Invalid KR or JP Record Date" item).

**Status:** Both findings confirmed live. Fix not yet designed — needs to address the Korea scope gap and replace the blanket end-of-month assumption with a per-security historical-pattern comparison (documentation-only phase).

---

## 2026-07-09 — Missing Ticker/ISIN: infrastructure constraint changes the design, and a systemic invocation gap explains zero historical tasks

**Design correction (Jose):** Mark's proposed fix — query IvyDB directly by known identifiers to auto-populate the missing field — isn't viable. Woodseer's Heroku infrastructure has no network path to OptionMetrics' internal IvyDB (PSQL10x) systems; they're not in the same ecosystem. The right approach instead is a rule-based AI agent that does web research (known reliable sources — exchange listing pages, ISIN registries, etc.) using whatever identifiers are already known (company name, exchange, the other non-missing identifier), then updates the record.

**Real scope of the gap:** 298,822 active primary listings are currently missing ticker and/or ISIN, but 289,947 (97%) are `coverage_type=9` (not covered) — genuinely outside this check's universe and not relevant to Woodseer's product. The real, in-scope number is **8,875** (`coverage_type=3`: 8,874, `coverage_type=4`: 1).

**Why `MISSING_TICKER_AND_ISIN_CHECK` has zero tasks in its entire history despite 8,875 real in-scope gaps — not a bug, an invocation-model gap:** `MissingTickerAndIsinCheck` is correctly wired into `AnalystTaskRunner` (`analyst_task_runner.rb:92`), and `Share#covered?` (`share.rb:130-132`) correctly includes tier 3. But mapping every caller of `AnalystTaskRunner` shows it is **entirely event-driven, per-security** — triggered by a regen completing, a corporate action landing, an admin editing that specific security/listing, a portfolio addition, a dividend series publishing. The *only* thing that runs on an actual daily schedule (`config/clock.rb:14`) is `DailyAnalystTaskRunner`, which only calls `PassedEstimateCheck` and `MissingForecastCheck` — a much narrower list that excludes `MissingTickerAndIsinCheck` entirely. So a security whose ticker/ISIN has been missing since it was first created, with nothing happening to it since (no corporate action, no regen, no manual touch), may simply never have had `AnalystTaskRunner` invoked for it at all — not a defect in the check's own logic, a structural blind spot in how it gets triggered.

**Broader implication, worth flagging beyond this one item:** the same blind spot likely applies to every other check in the full `AnalystTaskRunner` list that isn't *also* in `DailyAnalystTaskRunner`'s narrow daily set — dormant securities with data gaps in any of those ~50 other checks could similarly go unexamined indefinitely. Worth a systemic look, not just a ticker/ISIN-specific fix.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Missing Ticker/ISIN" item).

**Status:** Design redirected to a web-research AI agent per infrastructure constraints; real in-scope gap quantified at 8,875; root cause of zero historical tasks identified as an invocation-model gap, not a check-logic bug. Fix/agent not yet designed or implemented (documentation-only phase).

---

## 2026-07-09 — Security Is Being Taken Over auto-approval — safest, cleanest candidate on the whole Priority-2 list

**Mechanism:** `TakeoverCheck` (`app/tasks/takeover_check.rb`) fires when a real EDI corporate action of type `TAKEOVER` was created for the security within the last 30 days — a pure factual notification (close date, compulsory acquisition date, target %, status) derived directly from confirmed vendor data. No real judgment call exists in the check's own logic.

**Live evidence — the cleanest validation found in this entire review:** 2,912 total tasks all-time, **100% approved (`state = CHECKED`), zero ever rejected**, spanning 2018-07-25 through yesterday (2026-07-08) — the same continuous ongoing-burden pattern as every other Priority-2 item (same two people account for 61.8% and 33.6% of resolutions). Unlike every other item on this list, there is no historical counter-evidence at all: not a single rejection across 2,912 instances over 8 years.

**Recommendation:** this is the safest and simplest automation candidate identified across the whole Priority-2 review — no threshold decision (unlike the ADR/currency items), no scope ambiguity, and a perfect historical track record with zero exceptions. Straightforward to auto-approve on detection exactly as Mark proposed.

**Source:** `WOODSE_1.DOC` (Mark, Priority-2 list, "Security Is Being Taken Over" item).

**Status:** Root cause and real-world scale confirmed, live data validates "always approved" with zero exceptions. Fix not yet implemented (documentation-only phase). This closes out the full Priority-2 automation list — all 12 items now reviewed (Duplicate Dividend was covered earlier under the Highest-Priority item-1 finding).

---

## 2026-07-09 — Rarely Seen / Outdated Analyst Tasks: last-occurrence check reveals the label is only accurate for 2 of 5

**Request:** the doc lists 5 task types as "Rarely seen / outdated": Limeburner Discrepancy, Whack Date, Date Breaks a Rule, Dividends Appear Too Close, Security Associated with ETF type is not ETF. Checked actual last-occurrence data for each before treating any as safe to remove.

**Findings:**

| Task | Total ever | Last occurred | Verdict |
|---|---|---|---|
| Limeburner Discrepancy (22) | 173 | 2024-01-25 (~2.5 yrs ago) | Genuinely dormant — matches the label |
| Whack Date (27) | 50 | 2026-06-26 (~2 weeks ago) | **Not rare** — mislabeled |
| Date Breaks a Rule (30) | 5,649 | 2026-07-08 (yesterday) | **Not rare at all** — high-volume, active, badly mislabeled |
| Dividends Appear Too Close / `CLOSE_DIVIDENDS` (38) | 0 | never | Investigated further below |
| ETF Security Type Not ETF (43) | 0 | never | Investigated further below |

**`CLOSE_DIVIDENDS`/`CloseDividendsCheck` — genuinely dead code, safe to consider removing (or wiring in).** Confirmed zero callers anywhere in the app (`grep` across `app/`, `lib/`, `config/` finds only the class's own definition) — never invoked from `AnalystTaskRunner` or anywhere else. Has a spec (`spec/tasks/close_dividends_check_spec.rb`), matching the exact "built, tested, never wired in" orphan pattern the original audit already documented for a different check (`SecurityPrimaryExchangeCheck`). This is now a second confirmed instance of that same pattern.

**`ETF_SECURITY_TYPE_NOT_ETF`/`SecurityWithEtfTypeCheck` — NOT safe to remove; same invocation-gap bug as Missing Ticker/ISIN.** This check IS properly wired into `AnalystTaskRunner` (`analyst_task_runner.rb:85`). Live-checked the real data directly rather than trusting "zero tasks means zero cases": **49 securities right now** are associated with an ETF (`etfs.security_id`) while their own `security_type != 'ETF'` — exactly `create_task?`'s condition (`security_with_etf_type_check.rb:8-10`). Zero tasks have ever been created for any of them. Same root cause established for Missing Ticker/ISIN: `AnalystTaskRunner` is entirely event-driven per-security, with no daily full-sweep, so dormant securities with a long-standing mismatch may simply never have triggered a (re-)check. Recommendation: fix the invocation gap (or add this check to a periodic sweep), don't remove it — there's real, live, uncaught data behind it.

**Source:** `WOODSE_1.DOC` ("Rarely Seen / Outdated Analyst Tasks" reference list).

**Status:** Confirmed and quantified for all 5. `CLOSE_DIVIDENDS` is a genuine removal/rewire candidate; `WHACK_DATE` and `DATE_BREAKS_RULE` should be removed from the "rarely seen" label entirely; `ETF_SECURITY_TYPE_NOT_ETF` needs the invocation-gap fix, not removal; `LIMEBURNER_DISCREPANCY` is the one item that actually matches its label as-is.
