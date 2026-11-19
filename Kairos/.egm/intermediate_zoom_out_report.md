# Goldfish Review Report

**Target:** `/Users/shubhambansal/Documents/Code/sample-proj and handbook/Kairos/docs/designs/kairos.md` (full file, lines 1-1704)

**Stage:** Stage 8, Intermediate Zoom Out Review

**Inputs reviewed:**

- Earlier stage reports:
  - `zoom_out_report.md` (Stage 5)
  - `comprehension_report.md` (Stage 6)
  - `critic_report.md` (Stage 7)
- Background:
  - `prd.md`
  - `context/design-theme.md`
  - `context/engineering-standards.md`
  - `.egm/decision_options.md`

The design document has not changed since those reports were written, so their line references still apply.

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-zoom-out` | completed |

## Summary

The design still focuses on the root problem. The risk at this stage comes from the earlier reports, not from the design.

**If every earlier recommendation were applied,** the design would gain all of the following:

- a second trigger on `day_totals`
- a reporting view and a second database role
- an export worker process with advisory-lock slots
- a Caddy rate-limit plugin
- a nightly totals-consistency check
- a REPEATABLE READ week read
- a SECURITY DEFINER audit function
- about seven new document tables or sections

Most of these additions do one of two things. Some protect against problems that a single earlier choice created: the `day_totals` table. The rest belong to Stretch scope.

**The recommended set below goes the other way.** It removes:

- one table (`day_totals`)
- one trigger
- one function (`employee_day_hours`)
- one background task
- one SQL event hook

It adds one small table (`employee_credentials`) and three short document tables.

**Severity breakdown:** 1 Blocker, 6 Major, 4 Nit.

## Prioritized Findings (Blockers -> Major -> Nits)

### Credential columns sit on the audited `employees` table: split them out now - Severity: Blocker

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 468-480 (Q8), 508-524 (`employees`), 539, 1297-1302.
* **Earlier findings this concerns:**
  * Stage 7 Blocker "Credential and lock-out columns sit on the audited `employees` table".
  * Stage 6 #6.
  * Stage 5 Major 3, the part about AUTH-2 and AUTH-4 columns that Slice 1 does not use.
* **Discrepancy Description:** The design goal at lines 93-98 means Slice 1 fixes the shape of the `employees` table. That shape causes two problems once the Slice 4 audit triggers arrive:
  * Sign-in writes go through an audited table, which breaks sign-in.
  * `password_hash` gets copied into `audit_log`.
* **Rationale & Recommendation:** Create `employee_credentials (employee_id PK, password_hash)` in Slice 1 and do not audit it.
  * Slice 2 adds `must_change_password`, `failed_sign_in_count` and `locked_until` to that table.
  * This follows the precedent the design already set with `employee_preferences` (line 539).
  * Sign-in writes then never touch an audited table, so the design needs no "sign-in actor" rule.
  * The existing test that every audited table has its trigger already covers this case. Stage 7's extra "audit_log never contains password_hash" test is not needed.
  * **Resist** Stage 7's alternative, in which `audit_row()` drops certain columns and skips updates that only touch them. That puts table-specific exceptions inside the generic PL/pgSQL function.

### Drop `day_totals`: use a per-day lock plus a fresh sum (closes about nine earlier findings) - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:**
  * Lines 401-420 (Q3) and 422-436 (Q4, rows 428-429).
  * Lines 599-605 (`day_totals`) and 632 (functions table).
  * Lines 749-758 (S-1 diagram).
  * Lines 871-878 (Q3-B) and 903-910 (Q4-C).
  * Lines 1430-1431 (`lock_day`), 1441-1445 (`read_week`) and 1462-1466 (`update_work_entry`).
* **Earlier findings this concerns:**
  * **Stage 5:**
    * Major 3 (the `day_totals (entry_date)` index).
    * Major 5 (over-limit edits).
    * Major 6 (the guard ignores holidays and can miss rows).
    * Nit 3 (totals come from two places).
    * Nit 4 (save flow one step shorter).
  * **Stage 6:** #3 (no rule for over-limit days).
  * **Stage 7:**
    * "`day_totals` trigger behaviour is unspecified".
    * "`day_totals` CHECK is described as a last guard".
    * "The 24-hour check blocks fixes".
    * "Week read totals can disagree with its entries".
    * The "`day_totals` drift" and volume-seed bullets of "No Risks and Mitigations section".
* **Discrepancy Description:** The design chose Q3-A over Q3-B because "Nothing in the database catches a path that forgets" (line 875). It also cites "a database `CHECK` as a last guard" (lines 876-878). Stage 5 Major 6 and Stage 7 both showed that this guard does not include holiday minutes, so it is not a 24-hour guard.
  * The earlier reports each add a part to keep the table correct:
    * a BEFORE UPDATE trigger on `day_totals` that calls `employee_holidays` (Stage 5)
    * an upserting trigger that covers both the old and the new date (Stage 7)
    * a nightly consistency comparison (Stage 7)
    * a "disable trigger, then backfill" step for the 20-million-row volume seed (Stage 7)
    * REPEATABLE READ for the week read, or a rule to sum entries in the service (Stage 7)
  * The other reason given for `day_totals` is RPT-2 (the Q4-C rejection, lines 906-910). `decision_options.md` (Q4 option C) estimates 50 to 150 ms for the raw one-week sum. The target is 2 seconds at p95 for about 10 HR users. RPT-2 is in Slice 4, and Slice 5 measures it. That makes the table an optimisation made before any measurement.
* **Rationale & Recommendation:** Choose Q3-B.
  * A save takes `pg_advisory_xact_lock` on (employee, date). When an entry moves to another date, the save locks both days in date order, as the design already does.
  * The save then sums that day's entries through the existing `(employee_id, entry_date)` index (a handful of rows), adds holiday minutes, and checks the limit.
  * This removes `day_totals`, the `time_entries_totals()` trigger, the ensure-row step in `lock_day`, the CHECK wording problem and the drift risk.
  * The week read takes its totals only from the entries it loaded. That fixes Stage 5 Nit 3 and the Stage 7 week-read finding with no change to isolation level.
  * The volume seed needs no trigger handling.
  * **Keep** the concurrency test (lines 1509-1516). It is the real proof for NFR-DATA-1 and works unchanged.
  * **Accept** the non-increasing rule from Stage 5 Major 5, Stage 6 #3 and Stage 7: reject a save only if the new total is over 1,440 minutes and higher than the total before the save. This is one comparison, and it fixes a conflict with the design's own "HR is never blocked" rule (lines 58-62).
  * **Resist** the following, because they exist only to support `day_totals`:
    * the BEFORE UPDATE trigger (Stage 5 Major 6)
    * the nightly check and the seed backfill procedure (Stage 7)
    * REPEATABLE READ (Stage 7)
    * the write-then-check reordering (Stage 5 Nit 4)
  * If the author still wants a guard inside the database, use a single BEFORE INSERT/UPDATE trigger on `time_entries` that takes the advisory lock, sums the day and checks holidays. Do not keep a totals table plus two triggers. The volume seed then pays the cost of that trigger.

### Narrow the shared-hours promise instead of building a reporting view and a second role - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:**
  * Lines 356-375 (Q1), especially 369-372.
  * Lines 633-634 (functions table).
  * Lines 829-832 (section 8, Slice 4) and 1056-1058 (Future Improvements, Slice 4).
  * Lines 1271-1272 (ST-03 migration) and 1441-1445 (`read_week`).
* **Earlier findings this concerns:**
  * Stage 5 Major 1 (both its "narrow" and its "generalize" options).
  * Stage 6 #11.
  * Stage 7 "`employee_day_hours` cannot be the only path for every report".
* **Discrepancy Description:** All three stages agree that the promise on lines 369-372 is wrong. They propose two different fixes, and both are speculative machinery for reports that have no implementation plan yet (Slice 4 and Stretch):
  * Stage 5's "generalize" option would add:
    * a row-level `reportable_hours` view (a UNION ALL of entries and derived holiday rows)
    * a second database role
    * grant-based enforcement
    * an EXPLAIN check at 20 million rows
  * Stage 7 proposes a static scan of repository SQL against an allow-list.
* **Rationale & Recommendation:**
  * In Slice 1, keep one SQL function that returns calendar facts per employee and date: expected minutes and holiday minutes. To do this, extend `employee_holidays`.
  * The week read and the 24-hour check both use that function. Expected minutes are then defined in one place, which fixes Stage 6 #11.
  * Remove `employee_day_hours` from Slice 1. Nothing in Slice 1 calls it or tests it.
  * Restate the rule: "holiday and expected minutes come only from this function; project, task and kind breakdowns sum `time_entries` directly". Drop the claim of a guard test in every slice.
  * The Slice 4 plan states which reports include deactivated employees. RPT-2 does not. RPT-3 does, because of S-7.
  * **Resist** the view plus role plus grants (Stage 5 Major 1, "generalize" option) and the SQL-scan test (Stage 7).

### Exports: label Q5 as Stretch and record the Slice 5 load-test conflict - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 438-447 (Q5), 724 (endpoint table), 912-922 (Q5-B), 1061 (Future Improvements).
* **Earlier findings this concerns:**
  * **Stage 5 Major 7:** a separate export process, ProcessPoolExecutor, advisory-lock slots held in PostgreSQL, and abort on error.
  * **Stage 7:**
    * "CSV export edge cases".
    * "Slice labels for audit, exports and HOL-7 disagree" (the Q5 heading).
    * The exports bullet of "No Risks and Mitigations section".
    * "Process model per instance is not specified" (the export cap).
* **Discrepancy Description:** RPT-5 is an S requirement (prd.md line 206). PRD line 393 places all S requirements in Stretch. PRD section 10 needs a decision on question 5, not an implementation.
  * Stage 5 Major 7 would add the following, all for about 10 HR users of a feature that is not scheduled:
    * a second process type
    * Caddy routing for `*.csv` paths
    * a global slot mechanism in PostgreSQL
  * **A conflict none of the earlier reports raised:** lines 445-447 have the Slice 5 load test run a 100,000-row export. Two PRD statements make that impossible:
    * PRD line 392 assigns NFR-PERF-1 to NFR-PERF-9 to Slice 5, and NFR-PERF-4 includes the CSV export target.
    * RPT-5 is in Stretch, which comes after Slice 5.
  * So Slice 5 would have to load-test a feature that does not exist yet.
* **Rationale & Recommendation:**
  * Label Q5 "(Stretch)". Keep the decision itself: stream the export in the request, with a cap per process.
  * **Accept** these low-cost correctness items for when exports are built:
    * Add tab and carriage return to the formula-prefix characters.
    * Close the connection without sending the final chunk on any error after streaming starts.
    * Enforce an overall deadline in the service, because `statement_timeout` applies to each FETCH separately.
  * Make the CPU-isolation choice (thread or process offload) in the Stretch plan, and gate it on a measurement.
  * Add an Open Question with an owner. One of two options applies:
    * Slice 5 proves NFR-PERF-4 for RPT-1 to RPT-3 only, and exports get their own load run when they are built.
    * Exports move ahead of Slice 5.
  * For 10 HR users, an occasional 429 "try again in a minute" caused by round-robin routing is acceptable.
  * **Resist** the export process, the Caddy routing and the global advisory-lock slots (Stage 5 Major 7).

### State "1 worker per instance": one sentence resolves the process-model, connection-budget and hashing findings - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 227-245, 264-266, 395-399, 441-443, 1135-1139, 1297-1302.
* **Earlier findings this concerns:**
  * Stage 7 "Process model per instance is not specified".
  * Stage 7 "Sign-in lets anyone use up expensive hashing".
  * Stage 5 Major 8.
  * The connection-budget point in Stage 5 Major 7.
* **Discrepancy Description:** The design does not state how many processes run per instance. The earlier reports answer that gap with rate limits, lock-out changes and extra load tests, when stating the process model would settle most of it.
* **Rationale & Recommendation:** State the following in the design:
  * **One uvicorn process per instance.** At 50 requests per second per instance, with a few milliseconds of CPU each, one event loop has room to spare.
  * **A cap of one password hash at a time per process, with a short bounded wait.** The wait is needed because NFR-PERF-3 requires no errors.
    * argon2-cffi calls C code through cffi. The reviewer understands that cffi releases the GIL during the call, but this has not been checked against the library. If it holds, the hash runs on the second core and leaves the event loop's core free.
    * Two instances then handle about 25 to 30 sign-ins per second against a planned 8, the same figure Stage 5 Major 8 reached.
  * **Connection budget:** 2 × (20 + 10), plus the export pool, plus migrate and admin connections, is about 70. That is under the PostgreSQL default of 100.
  * **Caps apply per process.** Slice 5 decides whether a second worker is worth adding. If it is, halve the pool size.
  * **Resist the Stage 7 per-IP rate limit on sign-in**, for two reasons:
    * Standard Caddy has no rate-limit directive. Adding one needs a third-party module and a custom Caddy build.
    * The 5,000 employees probably share a few office egress IPs. A per-IP limit could then refuse the legitimate burst of 500 sign-ins that NFR-PERF-3 requires to complete with no errors.
  * **Also resist:**
    * the abusive-sign-in load test, which is not a PRD requirement
    * per-account plus per-IP lock-out, which changes what AUTH-4 means
  * Record lock-out abuse as one accepted-risk row in the Risks table instead (see the next finding).

### Merge the documentation additions into three short tables, not seven - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 20-26, 120-122, 137-152, 819-837, 999-1025, 1029-1064, and section 6 (lines 642-725).
* **Earlier findings this concerns:**
  * **Stage 7:**
    * "No Risks and Mitigations section" (Blocker).
    * "Slice labels disagree".
    * "Non-Goals list incomplete".
    * "NFR section has coverage gaps".
    * "Additional Resources and Future Improvements are nested".
    * "Line 301 refers to an error-code list".
    * "Rejections are attributed to the author".
    * "Performance estimates stated as facts".
    * "Kairos solves these claims more".
  * **Stage 6:** #2, #4, #7, #9, #12, #15.
  * **Stage 5:** Nit 11.
* **Discrepancy Description:** Applied one by one, these findings would add:
  * a Risks section
  * an NFR mapping table
  * a requirement-to-slice table
  * an error-code table
  * a slice table in the Overview
  * a business-rules list
  * a glossary of PRD IDs
  * a Friday-peak paragraph
* **Rationale & Recommendation:** Add three tables instead.
  1. **One "Scope by slice" table near the Overview.**
     * It replaces the second Non-Goals bullet (lines 120-122), section 8 (lines 819-837) and Future Improvements (lines 1039-1064).
     * It defines M, S, C and Stretch.
     * It lists what PRD section 9 puts in each slice, by ID range. It names only the places where the design departs from the PRD: TIME-17 delivered early, the placement of AUD-2 and AUD-3, and the export conflict from the previous finding.
     * It has one row per NFR group, saying where the group is met or that it is deferred.
     * It closes the slice-label, Non-Goals, NFR-coverage and heading-nesting findings, Stage 6 #7, #9 and #12, and Stage 5 Nit 11.
  2. **An error-code table in section 6.** It is part of the API contract, and NFR-SEC-6 asks for a documented error format.
  3. **A short Risks table of about 8 rows.**
     * Several of Stage 7's risk bullets disappear under the `day_totals`, request-ID and session-cleanup recommendations in this report: drift, the cleanup lock and export blocking.
     * Single database, single proxy, backups (NFR-DATA-4) and NFR-REL-1 each get one row: "accepted; out of scope per Non-Goals (production deployment)".
     * Do not design backups or high availability.
  * **Accept** these low-cost items:
    * a one-line restatement of each PRD question (Stage 6 #2)
    * one clause each for TIME-8, TIME-9, HOL-2 and RPT-6 (Stage 6 #1)
    * one sentence that defines the Friday peak and cites PRD 7.2 (Stage 6 #4)
    * removing the "Rejected by author:" prefix, which also shortens the document
  * **Resist:**
    * the glossary of PRD IDs (Stage 6 #15). Link to `prd.md` instead.
    * a source and a test for every figure (Stage 7). One sentence is enough: "all figures are estimates; Slice 5 confirms them".

### Request ID: one `set_config` statement replaces the SQL comment hook - Severity: Major

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 270-289, 1007-1008, 1218-1227.
* **Earlier findings this concerns:**
  * Stage 5 Major 2.
  * Stage 7 "Request-ID SQL comment breaks the prepared-statement cache and contradicts no SQL is built from strings".
* **Discrepancy Description:** Lines 270-289 add the request ID to each statement as a SQL comment through an event hook. Stage 5 and Stage 7 both found two problems with this:
  * It defeats the prepared-statement cache.
  * It contradicts the design's own rule that no SQL is built from strings.
* **Rationale & Recommendation:** Accept the fix both stages proposed.
  * At the start of each transaction, set `application_name` (or a `kairos.request_id` setting) in the same statement that sets `kairos.actor_id`. Do this for reads as well as writes.
  * Add `%a` to `log_line_prefix`.
  * This removes the event hook, the injection concern and the cache misses, and adds no new component.

### Remove the session-cleanup background task - Severity: Nit

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 392-394, 1319-1321.
* **Earlier findings this concerns:**
  * Stage 5 Major 4.
  * Stage 7 Nit "`audit_log` insert grant and cleanup lock type" (the lock-type half).
* **Discrepancy Description:** The periodic cleanup task is the design's only background process. It also raises the question of whether its advisory lock is session-level or transaction-level.
* **Rationale & Recommendation:** Accept the removal. Cleaning up sessions at sign-in and on deactivation is enough. The lock-type question then goes away.

### Caddy health checks: add the few configuration lines and nothing more - Severity: Nit

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 219-245, 1169-1179.
* **Earlier findings this concerns:**
  * Stage 5 Nit 1.
  * Stage 7 "Caddy load balancing has no health checks". Stage 7 rated this Major; this review rates it Nit.
* **Discrepancy Description:** The Caddy configuration has no upstream health checks.
* **Rationale & Recommendation:**
  * Add `health_uri /api/v1/health/ready`, passive failure detection and a short `lb_try_duration`.
  * Set uvicorn's graceful-shutdown timeout.
  * Add one sentence on where Caddy runs in the reference environment.
  * These are a few configuration lines that support NFR-REL-2. Nothing further is needed.

### TIME-17 in Slice 1: keep the version check, defer the dialog - Severity: Nit

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines 449-459, 1476-1478, 1612-1613.
* **Earlier findings this concerns:**
  * Stage 5 Nit 6.
  * Stage 6 #13.
  * Stage 6 #15 ("Apply my changes again" is undefined).
* **Discrepancy Description:** Slice 1 includes the full two-action conflict dialog for TIME-17. The PRD places TIME-17 in Stretch, and the "Apply my changes again" action is not defined.
* **Rationale & Recommendation:**
  * **Keep in Slice 1:** `version`, `If-Match` on both PUT and DELETE, and the 428 and 409 responses. They cost little on the backend.
  * **In Slice 1, the modal** shows the 409 message and reloads the entry.
  * **Defer** the two-action dialog to Stretch, where the PRD puts TIME-17.
  * This removes the undefined "Apply my changes again" behaviour instead of specifying it.

### Smaller items: accept, resist or defer - Severity: Nit

* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Lines given per item below.
* **Discrepancy Description:** These are smaller items from the earlier reports. Each is sorted into accept, resist, or defer.
* **Rationale & Recommendation:**
  * **Accept:**
    * **Slice 1 fields that no Slice 1 requirement needs** (Stage 5 Major 3; lines 570-571, 588-594, 670, 679-711):
      * Remove these from Slice 1: the LEAVE kind and `leave_type`, `tasks.is_active` with `TASK_INACTIVE`, `flags`, `isFuture`, `editable` and `mustChangePassword`.
      * Restate the design goal as "no destructive change to Slice 1 tables".
      * The credential split above already removes the AUTH columns.
    * **Test database setup** (Stage 6 #8; lines 1233-1246, 1687-1689): mount the existing `db/init/01-roles.sh` in the test container instead of re-creating the roles in `conftest.py`.
    * **Docker Hub images** (Stage 6 #5 and Stage 7): add one line to Open Questions.
    * **Edits to entries on a closed project** (Stage 7 Nit; lines 712-715): check project availability again only when the project or the date changes.
    * **Mermaid diagrams** (Stage 7): replace the `;` characters.
    * **Security headers** (Stage 7 Nit "Security headers and /metrics exposure"):
      * That finding missed line 1178, which already sets `Content-Security-Policy: default-src 'self'`.
      * The real gap is `frame-ancestors`, which does not fall back to `default-src`. Add `frame-ancestors 'none'`.
      * Restricting `/metrics` belongs in the Slice 5 plan.
    * **ETags on `/tasks` and `/holidays`** (Stage 5 Nit 12; lines 461-463, 1397-1398): drop them.
  * **Resist:**
    * **Returning the affected `Day` from each mutation** (Stage 5 Nit 2). It is an optimisation with no measurement behind it. Once `day_totals` is dropped, the reload is one indexed query.
    * **A SECURITY DEFINER `audit_row()` and revoking INSERT** (Stage 7 Nit). AUD-3 only forbids editing and deleting audit records. Defer this to the AUD-3 Stretch plan.
  * **Defer, or leave to the author:**
    * **A holiday added on a day that already has leave** (Stage 7 Nit): add it to the Slice 3 Open Questions. Do not decide it now.
    * **The disabled search box in the top bar** (Stage 5 Nit 9; lines 1555-1557): `design-theme.md` line 7 lists "search" in the top bar. Removing the box departs from the theme and should be stated as a departure. Keeping a disabled box is also acceptable.
    * **`employee_preferences`** (Stage 5 Nit 5): taking the location from the latest entry by `entry_date` does not match "used last" when an employee back-fills older days. Dropping the table saves little. Keep it unless the author accepts the approximation.

## Recommended simplifications at a glance

| Change | Earlier findings it closes or makes unnecessary |
| --- | --- |
| Drop `day_totals`; per-day advisory lock plus a fresh sum (Q3-B) | Stage 5 Major 3 (index), Major 5, Major 6, Nit 3, Nit 4; Stage 6 #3; Stage 7: trigger behaviour, CHECK wording, blocked fixes, week-read totals, drift and seed risk bullets |
| Split `employee_credentials` out of `employees` in Slice 1 | Stage 7 Blocker (credentials on an audited table); Stage 6 #6; the AUTH part of Stage 5 Major 3 |
| State one uvicorn process per instance and a per-process hash cap | Stage 7 process model and hashing findings; Stage 5 Major 8; the connection budget part of Stage 5 Major 7 |
| One calendar-facts function; remove `employee_day_hours` from Slice 1 | Stage 5 Major 1; Stage 6 #11; Stage 7 "`employee_day_hours` cannot be the only path" |
| One "Scope by slice" table, one error-code table, one Risks table | Stage 7 Risks Blocker plus seven Stage 7 structure findings; Stage 6 #7, #9, #12; Stage 5 Nit 11 |
| `set_config` request ID instead of a SQL comment hook | Stage 5 Major 2; Stage 7 request-ID finding |
