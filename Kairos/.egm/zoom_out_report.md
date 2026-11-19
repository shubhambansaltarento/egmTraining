# Goldfish Review Report

**Target:** `Kairos/docs/designs/kairos.md` (lines 1-1704, full file)

**Stage:** Stage 5, Initial Zoom Out Review

**Baseline context given to reviewers:** `Kairos/prd.md`, `Kairos/context/design-theme.md`, `Kairos/context/engineering-standards.md`, `Kairos/.egm/decision_options.md`

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-zoom-out` | completed |
| `egm-inventive-catalyst` | completed |

## Summary

| Severity | Count |
| --- | --- |
| Blocker | 0 |
| Major | 8 |
| Nit | 12 |

- One finding was raised by both lenses and is merged below (the `employee_day_hours` shape, Major 1).
- The synthesizer spot-checked the line references for Q1 (lines 356-375), Q2 (lines 392-399), Q3 (lines 401-420) and the request pipeline (lines 270-275). They match the document.
- The zoom-out handback said "9 Nits" but listed ten (N1-N10). All ten are included here.

**Checks with no findings (egm-zoom-out):**
- Enough context to understand the goal: passes.
- Root problem focus: passes. Each of the PRD's three problems maps to a mechanism.
- Reuse of existing code: nothing to report. The repository has no source files yet, and the chosen libraries are the standard ones for this stack.

**Decisions with no false compromise (egm-inventive-catalyst):** Q2 (database session lookup), Q6 (integer version), Q7 (no server cache) and Q8 (audit triggers with a required actor).

**Where the two lenses pull in different directions:** Some inventive-catalyst resolutions add moving parts: a reporting view with a separate database role (Major 1), a `day_totals` trigger (Major 6), and an export process with advisory-lock slots (Major 7). The zoom-out lens argues for fewer moving parts. The author should weigh each proposed addition against the zoom-out findings on Slice 1 scope (Major 3) and background work (Major 4).

---

## Prioritized Findings (Blockers -> Major -> Nits)

### Major 1. The shared hours function fits only one report and drops deactivated employees - Severity: Major
* **Reviewer Lens:** `egm-zoom-out` (M1) and `egm-inventive-catalyst` (Finding 1)
* **Affected Scope:**
  * Technical Plan section 4, Q1 (lines 356-375)
  * Q4 index table, row `(entry_date, project_id)` (line 427)
  * Functions table (lines 632-634)
  * ST-03 migration (lines 1271-1272)
  * ST-07 `read_week` (lines 1441-1445)
  * "How Slices 2 to 5 build on this", Slice 4 (lines 829-832)
  * Future Improvements, Slice 4 (lines 1056-1058)
* **Discrepancy Description**:
  * Lines 369-372 say the week read, the 24-hour check, copy-week, RPT-1, RPT-2, RPT-3 and the exports "all call these functions". A per-slice test is meant to enforce this. But `employee_day_hours(from_date, to_date)` returns whole-day totals, one row per *active* employee, with no employee filter and no project, task or kind breakdown. So it cannot serve:
    * the single-employee week read, which has no employee parameter to pass;
    * RPT-1, which groups by project and kind;
    * RPT-3 and its export, which group by project, employee and task. The Q4 table already has RPT-3 reading `time_entries` directly, which is the pattern the test is meant to forbid;
    * deactivated employees' history, which S-7 says must still appear in HR reports.
  * `read_week` works out `expectedMinutes` in Python (line 1443). This repeats the rule held in the SQL function (line 365), so the same rule lives in two places.
  * The function is created in Slice 1, but no Slice 1 code calls it and no Slice 1 test covers it.
  * The document does not say how the per-slice guard test works.
* **Rationale & Recommendation**: The design promises one shared function for every hours calculation, but only RPT-2 in Slice 4 can actually use it. The two lenses offer different fixes, and the author should pick one:
  * *Narrow (zoom-out):*
    * Restate the rule as "holiday minutes come only from `employee_holidays`" and keep expected minutes in one place.
    * Remove `employee_day_hours` from the Slice 1 migration. Add it in Slice 4, shaped for RPT-2.
    * Explain how the guard test works, or drop it.
  * *Generalize (inventive-catalyst):*
    * Define one view at the level of single rows, for example `reportable_hours`. It would be `time_entries` UNION ALL one derived holiday row per (employee, date), bounded by date of joining and by deactivation date.
    * Build every total on that view. `employee_day_hours` becomes a GROUP BY over it.
    * Enforce the rule with database grants instead of a code scan.
    * Costs: the view must be checked with EXPLAIN at 20 million rows, and a second database role adds some setup.
  * *Either way:* state which reports include deactivated employees (RPT-2 does not, RPT-3 does).

### Major 2. The per-statement request-ID SQL comment defeats prepared-statement caching - Severity: Major
* **Reviewer Lens:** `egm-zoom-out` (M2)
* **Affected Scope:** Technical Plan section 2, Request pipeline step 1 (lines 270-275); ST-02 `db.py` (lines 1226-1227)
* **Discrepancy Description**:
  * Each statement gets a `/* request_id=<id> */` comment, so the SQL text is different on every request.
  * The SQLAlchemy asyncpg dialect caches prepared statements by SQL text (an LRU of 100 by default). With the comment, every statement misses the cache and pays an extra prepare round trip. The cache also keeps closing evicted statements.
  * This sits on the session lookup, the week read and the save. Those are the paths with the p99 < 100 ms target, which the document itself calls a main risk (line 1013).
* **Rationale & Recommendation**:
  * Set the request ID once per transaction instead, for example `set_config('application_name', 'kairos/<request_id>', true)`. Do this alongside the existing `kairos.actor_id` call, on connection checkout so read-only requests get it too.
  * Add `%a` to PostgreSQL's `log_line_prefix`.
  * This keeps NFR-OPS-3, removes the event hook, and keeps statement text stable.

### Major 3. Slice 1 carries schema and API fields for later slices and stretch items, on a weak reason - Severity: Major
* **Reviewer Lens:** `egm-zoom-out` (M3)
* **Affected Scope:**
  * Design goals (lines 93-98)
  * `employees` (lines 522-524)
  * `tasks.is_active` (line 571)
  * `time_entries.kind` and `leave_type` (lines 588, 592, 594)
  * `day_totals` `(entry_date)` index (lines 429, 604)
  * `TASK_INACTIVE` (lines 670, 1449)
  * `Me.mustChangePassword`, `Day.flags`, `isFuture`, `editable`, `OVER_DAILY_LIMIT` (lines 679-688, 706-711)
* **Discrepancy Description**:
  * The design goal is that Slices 2 to 5 should "not need to rewrite Slice 1's tables". But the document itself says adding a value "is a one-line migration" (line 503).
  * On that reason, Slice 1 ships:
    * AUTH-2 and AUTH-4 columns that nothing uses;
    * the `LEAVE` kind and `leave_type` (Slice 3);
    * a PRJ-13 `TASK_INACTIVE` check (stretch);
    * an RPT-2 index (Slice 4), while the RPT-3 index is deferred, which is inconsistent;
    * API fields that are always empty or constant in Slice 1.
* **Rationale & Recommendation**:
  * Together these widen Slice 1's API, tests and review surface for no Slice 1 requirement. That works against PRD section 9 ("a finished Slice 1 beats a half-finished Slice 4").
  * Keep in Slice 1 only what would break if added later:
    * `version` and `If-Match`;
    * `holidays` and `employee_holidays`, because the 24-hour check needs them;
    * the actor `set_config` call.
  * Let later slices add their own columns, enum values, indexes and response fields.
  * Restate the goal as "no destructive change to Slice 1 tables".

### Major 4. A daily session-cleanup task runs inside the API with no proven need - Severity: Major
* **Reviewer Lens:** `egm-zoom-out` (M4)
* **Affected Scope:** Q2 (lines 392-394); ST-04 "Expired sessions" (lines 1319-1321)
* **Discrepancy Description**:
  * Sign-in already deletes that employee's expired sessions, and deactivation deletes all of them. The only leftover rows belong to active employees who stopped signing in: at most a few thousand rows, looked up by primary key.
  * Even so, the API runs a recurring task under an advisory lock. It is the only background process in a design whose Non-Goals reject job queues.
* **Rationale & Recommendation**:
  * Drop the scheduled task and rely on cleanup at sign-in.
  * If growth shows up in the Slice 5 volume run, add a `make` target or a PostgreSQL-side job then.

### Major 5. The 24-hour rule blocks every edit on a day that a holiday has pushed over the limit - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 2)
* **Affected Scope:**
  * The Problem, first bullet (lines 58-62)
  * Q3 step 3 (lines 411-413)
  * Day `flags`, `OVER_DAILY_LIMIT` (lines 706-709)
  * ST-07 `update_work_entry` (lines 1462-1466)
* **Discrepancy Description**:
  * HR is never blocked, and the limit is checked only when an employee saves. Step 3, however, rejects any save that leaves the day over 1,440 minutes.
  * On a day a holiday has pushed over 24 hours, the employee cannot:
    * fix a typo in Activity;
    * change the location;
    * shorten an entry, if the new total is still over the limit.
  * Only DELETE works. The `OVER_DAILY_LIMIT` flag asks the employee to fix the day, but the rule blocks every partial fix.
* **Rationale & Recommendation**:
  * Change the rule to: a save may not leave the day over 1,440 minutes *and* higher than it was before the save. The check under the existing `FOR UPDATE` lock needs only one extra comparison.
  * Add tests for two cases: an edit that reduces the total but leaves it over the limit, and an edit that does not change the duration.
  * Add an Open Question to confirm that this reading satisfies the PRD's "cannot exceed 24 hours".

### Major 6. The database guard for the 24-hour rule ignores holiday minutes and can silently miss rows - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 3)
* **Affected Scope:**
  * Q3 (lines 417-420)
  * `day_totals` (lines 599-605)
  * Functions table, `time_entries_totals()` (line 632)
  * Alternatives, Q3-B rejection (lines 871-878)
* **Discrepancy Description**:
  * Q3-B is rejected because "nothing in the database catches a path that forgets". The guard used to justify Q3-A is `CHECK (entry_minutes BETWEEN 0 AND 1440)`, which counts entries only. A path that skips the service check can therefore store a 1,920-minute day on a weekday holiday.
  * The trigger's behavior when the `day_totals` row is missing is not stated. This can happen on a path that skipped `lock_day`, or when an edit moves an entry to a new date. A plain UPDATE would affect zero rows, and the totals would drift with no error.
* **Rationale & Recommendation**:
  * Make `time_entries_totals()` an upsert, so the row always exists and the update itself takes the row lock.
  * Add a `BEFORE UPDATE` trigger on `day_totals` that raises an error when the total rises and `NEW.entry_minutes + employee_holidays(...)` is over 1,440. This also applies the non-increasing rule from Major 5.
  * Map the SQLSTATE to `DAILY_LIMIT_EXCEEDED`. Keep the service check to produce the 409 details.
  * This overlaps with zoom-out Nit N2, which reaches the same upsert-in-trigger shape to simplify the save flow.

### Major 7. CSV export runs CPU-bound formatting on the same event loop that serves the interactive p99 target - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 4)
* **Affected Scope:**
  * Q5 (lines 438-447)
  * Alternatives, Q5-B rejection (lines 912-922)
  * Backend structure (lines 264-266)
  * `backend/Containerfile` (lines 1135-1139)
* **Discrepancy Description**:
  * Q5-B is rejected because there is "no spare machine". A separate worker does not need a separate machine.
  * The chosen design formats 100,000 rows (0.5 to 1 s of CPU) in a `StreamingResponse` generator on the async event loop. Each batch blocks every other request on that process while it runs.
  * Other gaps:
    * The number of uvicorn workers per instance is not stated. With 2 instances × 2 workers × (20+10) connections, the total is 120, above PostgreSQL's default `max_connections` of 100.
    * Round-robin routing makes the "1 export per instance" cap unreliable. A user can get 429 while the other instance is idle.
    * Statement timeouts apply to each FETCH, so the 30-second limit on the whole export is not enforced.
    * A failure after the 200 header gives the user a cut-short CSV that looks complete.
* **Rationale & Recommendation**:
  * Run exports in a separate process from the same image, with Caddy routing `*.csv` paths to it. Or keep exports in-request and format each batch in a single-worker `ProcessPoolExecutor`.
  * Replace the per-instance cap with a global cap held in PostgreSQL, for example session-level `pg_try_advisory_lock` slots.
  * On any error after streaming starts, abort the connection without the final chunk, and add a test for it.
  * State the number of workers per instance and the connection budget.
  * Severity is Major even though export is stretch scope, because the Slice 5 load test is meant to prove this decision.

### Major 8. The password-hash concurrency cap is sized for memory, not CPU - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 5)
* **Affected Scope:** Q2, password hashing (lines 395-399); ST-04 `security/passwords.py` (lines 1297-1302); Non-Functional Requirements, Performance (lines 1010-1015)
* **Discrepancy Description**:
  * The cap of 4 concurrent argon2id hashes per instance is set from memory (256 MiB). On a 2-vCPU instance, 4 hashing threads can use both cores while the event loop must still meet p99 < 100 ms.
  * The PRD puts sign-in bursts at the Monday and Friday peaks.
  * The design's own numbers (8 sign-ins a second at 50 to 80 ms each) need under 1 core in total.
* **Rationale & Recommendation**:
  * Cap at 1 hash per instance (or vCPU minus 1). That still allows about 25 to 30 sign-ins a second, about 3 times the planned rate.
  * Add a short bounded wait queue so bursts wait instead of failing (NFR-PERF-3).
  * Run the 500-sign-in burst in the Slice 5 load test at the same time as the interactive load.
  * Tune the queue limit with that test, because a tightly bunched burst pushes sign-in p99 toward 500 ms.

---

### Nit 1. Caddy does not use the readiness endpoint - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 6)
* **Affected Scope:** Technical Plan section 1 (lines 223-245); health endpoints (lines 673-674); ST-01 `proxy/Caddyfile` (lines 1169-1179)
* **Discrepancy Description**:
  * Caddy uses plain round robin with no health checks, so about half the requests fail while one instance restarts. Those failures count against the NFR-PERF-5 error budget.
  * `/health/ready` is defined and tested, but nothing uses it.
* **Rationale & Recommendation**:
  * Configure `health_uri /api/v1/health/ready`, a short `health_interval`, passive `fail_duration`, and an `lb_try_duration` of 1 to 2 s.
  * Have uvicorn shut down gracefully so in-flight requests finish.
  * Limit retries to connection failures, so a POST the upstream already received is not repeated.

### Nit 2. Every save is followed by a full week reload - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst` (Finding 7)
* **Affected Scope:** Frontend structure, Server data (lines 319-322); S-1 sequence diagram (lines 756-759)
* **Discrepancy Description**: Every create, edit or delete is followed by GET `/me/week`. At the Friday peak this roughly doubles write-related load and adds a round trip before the new total appears.
* **Rationale & Recommendation**:
  * Return the affected `Day` (or `Day[]` when an entry moves between dates) in the mutation response, computed in the same transaction. Patch the cache with `setQueryData`.
  * The week read and the mutation should share one service function to build `Day`.
  * See also Nit 3 on where week totals come from.

### Nit 3. The week read gets totals from two places - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N1)
* **Affected Scope:** Q4 table (line 428); S-1 sequence (line 758); `read_week` (lines 1441-1445)
* **Discrepancy Description**: The week read loads the entries it displays and also reads `day_totals`. If the two disagree, the cards and the day total on screen will not match.
* **Rationale & Recommendation**: Compute the week view's day totals from the loaded entries plus `employee_holidays`. Use `day_totals` only for the save-time lock and for RPT-2.

### Nit 4. The Q3 save flow could be one step shorter - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N2)
* **Affected Scope:** Q3 (lines 401-420); `lock_day` (lines 1430-1431); `update_work_entry` (lines 1462-1464)
* **Discrepancy Description**: The save runs three steps before writing (ensure the row exists, lock it, check), and orders the locks by hand for date moves. The trigger's update already takes the row lock.
* **Rationale & Recommendation**:
  * As an option for the author: write the entry, let the trigger upsert `day_totals`, then read the total, add holiday minutes, and check.
  * This removes `lock_day` and the manual lock ordering. The cost is a possible deadlock between opposite moves, which PostgreSQL detects and aborts.
  * This lines up with Major 6.

### Nit 5. `employee_preferences` is a whole table for one value - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N3)
* **Affected Scope:** Data dictionary (lines 534-539); S-1 sequence (line 755); `upsert_last_location` (line 1437)
* **Discrepancy Description**: The table only holds `last_work_location` and adds a write to every save.
* **Rationale & Recommendation**: Read the last location from the employee's most recent work entry through the existing `(employee_id, entry_date)` index. This removes the table, the extra write and a join on `/me`.

### Nit 6. The reason for shipping TIME-17 in Slice 1 does not hold - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N4)
* **Affected Scope:** Q6 (lines 457-459); ST-09 conflict UI (lines 1612-1613)
* **Discrepancy Description**: The stated reason is that adding a required header later "would break existing clients". The only client is the bundled single-page app, which is released together with the API.
* **Rationale & Recommendation**: Keep the `version` column. Either restate the reason (it is cheap to do now), or defer the 428 enforcement and the conflict dialog to stretch, where TIME-17 sits in the PRD.

### Nit 7. The implementation plan names helper functions and constants - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N5)
* **Affected Scope:** ST-07 (lines 1424-1437); `client.ts` (lines 1540-1541); `ProjectSelect` (lines 1604-1606); ST-09 helpers (lines 1621-1625)
* **Discrepancy Description**: The plan names individual helpers and tuning constants. These go stale once code is written, and PRD section 11 requires the document to match the code.
* **Rationale & Recommendation**: Describe responsibilities and contracts for each file instead.

### Nit 8. A shared component depends on a feature module - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N6)
* **Affected Scope:** ST-09 item 4 (lines 1616-1618)
* **Discrepancy Description**: `components/DurationInput.tsx` imports `features/week/duration.ts`. That reverses the dependency direction set in engineering-standards.md.
* **Rationale & Recommendation**: Put both under `features/week/`, or move `duration.ts` to a shared folder.

### Nit 9. A disabled search box in the top bar has nothing to search - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N7)
* **Affected Scope:** ST-08 TopBar (lines 1555-1557)
* **Discrepancy Description**: The top bar shows a disabled "Search, coming soon" box. No slice says what it would search.
* **Rationale & Recommendation**: Leave it out until a requirement defines it.

### Nit 10. The Vite dev proxy points at a port that is not published - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N8)
* **Affected Scope:** `compose.yaml` (lines 1104-1106); `vite.config.ts` (line 1168)
* **Discrepancy Description**:
  * Port 8000 is reachable only inside the container network, but Vite forwards `/api` to `localhost:8000`.
  * The `__Host-` / `Secure` cookie may also fail over plain HTTP in some browsers.
* **Rationale & Recommendation**: Document that this mode runs the API on the host, or drop the Vite proxy and develop against `make up`.

### Nit 11. Slice 4 audit triggers include stretch tables - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N9)
* **Affected Scope:** Q8 (lines 468-474); Future Improvements (lines 1056, 1061-1064)
* **Discrepancy Description**: Q8 (Slice 4) attaches `audit_row()` to `time_entries` (AUD-2) and `period_locks` (RPT-6). Both are stretch items. This adds an audit write to every entry save before Slice 5 measures performance.
* **Rationale & Recommendation**: Limit the Slice 4 triggers to AUD-1's tables. Add the others with their stretch work.

### Nit 12. Server ETags on `/tasks` and `/holidays` save almost nothing - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out` (N10)
* **Affected Scope:** Q7 (lines 461-463); ST-06 tasks (lines 1397-1398)
* **Discrepancy Description**: The server still runs the query and hashes the result. A 304 only saves sending about 12 rows, which TanStack Query already caches in the browser.
* **Rationale & Recommendation**: Drop these ETags until the load test shows a need.
