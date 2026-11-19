# Consolidated Review Report: Kairos Design

**Target:** `Kairos/docs/designs/kairos.md` (1,704 lines)
**Split:** FeatureRange lines 1-1067; DetailedRange lines 1068-1704 (`## Detailed Implementation Plan (Slice 1 only)`)
**Mode:** Consolidated report, all five stages, no checkpoints
**Goldfish context:** `prd.md`, `context/design-theme.md`, `context/engineering-standards.md`, `.egm/decision_options.md` (Stage 6 read the document alone)
**Date:** 2026-11-19

## Verdict

**Not ready to approve.** The feature design holds up: no reviewer found a section it could not paraphrase, and the core architecture (one PostgreSQL database, two stateless API instances, no cache or queue) drew no objection. The problems are in three places:

1. **One data-model choice will break sign-in later.** Password and lock-out columns sit on the `employees` table, which Slice 4 audits. Sign-in writes would then fail (no actor is set) and password hashes would be copied into `audit_log` (A1).
2. **The `day_totals` table creates most of the remaining design findings.** Its CHECK does not cover holiday minutes, its trigger is unspecified, it can drift, and it blocks fixes on days a holiday pushed over 24 hours. Stage 8 recommends removing it rather than adding four parts to protect it (A6). This is the largest decision in this report and the author should make it first.
3. **The Slice 1 plan cannot be followed as written.** Sub-tasks depend on files that later sub-tasks create, and shared files (`main.py`, `config.py`, `App.tsx`) change in sub-tasks that do not list them. Under the project rule "touch only the files it lists", ST-01 to ST-08 cannot be finished in order, and no sub-task before ST-10 leaves `make up`, `make lint` and `make test` passing (A2, A3).

## Reviewer Status

| Stage | Report | Reviewers | Status | Raw findings |
| --- | --- | --- | --- | --- |
| 5 Initial Zoom Out | [zoom_out_report.md](zoom_out_report.md) | `egm-zoom-out`, `egm-inventive-catalyst` | completed | 0 B, 8 M, 12 N |
| 6 Comprehension | [comprehension_report.md](comprehension_report.md) | `egm-comprehension` | completed | 0 B, 8 M, 7 N |
| 7 Critic | [critic_report.md](critic_report.md) | `egm-design`, `egm-structural`, `egm-style` | completed | 2 B, 19 M, 24 N |
| 8 Intermediate Zoom Out | [intermediate_zoom_out_report.md](intermediate_zoom_out_report.md) | `egm-zoom-out` | completed | 1 B, 6 M, 4 N |
| 9 Implementation Readiness | [implementation_readiness_report.md](implementation_readiness_report.md) | `egm-implementation-readiness`, `egm-codebase-details`, `egm-style` | completed | 3 B, 27 M, 27 N |
| **Total** | | | | **6 B, 68 M, 74 N = 148** |

After deduplication: **36 action items: 🔴 5 Blocker, 🟡 23 Major, 🟢 8 Nit.**

## How conflicts between stages were settled

* **Stage 8 was the arbiter for FeatureRange fixes.** It read Stages 5-7 and marked each proposed addition as accept, resist or defer. Where Stage 8 resisted an addition, this report lists it under "Recommendations not taken" instead of as an action item.
* **Stage 9 reviewed the document as it stands, with `day_totals` in it.** If the author accepts A6, parts of Stage 9 M9 (trigger contract) and M13 (the CHECK hiding a missing lock) no longer apply. The date-move wording in Stage 9 B3 still applies, because the check arithmetic is the same with or without the table.
* **Severity disagreements.** The higher rating was kept unless Stage 8 gave a reason to lower it. Two items were lowered on Stage 8's reasoning: Caddy health checks (Stage 7 Major, now Nit A30) and the session-cleanup task (Stage 5 Major, now Nit A29).
* **Claims no reviewer verified.** These are marked where they appear:
  * argon2-cffi releasing the GIL while hashing (A10)
  * Mermaid failing on semicolons in message text (A28)
  * Emotion styles being blocked by the CSP (A23)

## Suggested order of work

1. **Decide the design questions first,** because the Slice 1 plan changes depend on them: A1, A6, A7, A9, A10, A13.
2. **Then add the three documentation tables:** Scope by slice (A8), error codes (A12) and Risks (A5). Also add the PRD context lines (A14).
3. **Then rewrite the Slice 1 plan:** A2, A3, A4, then the remaining DetailedRange Majors.

---

## 🔴 Blockers

### A1. Move password and lock-out columns off the audited `employees` table
* **Scope:** Lines 468-480 (Q8), 508-524 (`employees`), 539, 1297-1302 (ST-04).
* **Problem:** `password_hash`, `failed_sign_in_count` and `locked_until` sit on `employees`. Slice 4 adds audit triggers to that table, and they cause two failures:
  * Every sign-in that updates a counter writes an audited row during an unauthenticated request, where `audit_row()` raises an error because no actor is set.
  * Q8 copies whole rows as JSON, so `password_hash` lands in `audit_log`.
  * The design goal at lines 93-98 says Slice 1 tables must not need rewriting, so this has to be fixed now, not in Slice 4.
* **Action:** Create `employee_credentials (employee_id PK, password_hash)` in Slice 1 and do not audit it. Slice 2 adds `must_change_password`, `failed_sign_in_count` and `locked_until` to it. This follows the `employee_preferences` precedent (line 539).
* **Sources:** S7 Blocker 1; S8 Blocker; S6 #6; S5 Major 3 (AUTH column part).

### A2. Make every sub-task leave the build passing
* **Scope:** Lines 1083-1100, 1147-1159, 1169-1179, 1233-1254, 1684-1694.
* **Problem:** Several commands and tests fail because they need files from later sub-tasks:
  * **`make up` (ST-01):** it needs Alembic (ST-03), `seed.py` (ST-05) and `src/main.tsx` (ST-08).
  * **`make lint`:** it needs `export_openapi` and `schema.ts` (ST-08), and `tsc` fails when there are no inputs.
  * **`make test`:** it calls targets that ST-01 does not define.
  * **ST-01 health tests:** they need `db.py`, `errors.py` and `conftest.py`, which come in ST-02.
  * **ST-02 fixtures and tests:** they need the migration (ST-03), sign-in (ST-04) and a route that returns 404 (ST-07).
* **Action:** Either reorder the sub-tasks or grow the shared files one step at a time. Either way, also:
  * Add a checkpoint line to each sub-task naming the `make` targets that must pass when it is done.
  * Define `test-backend`, `test-frontend` and `test-e2e` in ST-01.
  * Use pass-with-no-tests flags until the tests exist.
  * Give ST-02 a test-only router.
* **Sources:** S9 B1.

### A3. List every file a sub-task changes, with full paths
* **Scope:** ST-01 to ST-10 file lists; lines 1076-1077.
* **Problem:** Some files change in a sub-task that does not list them, and some files are missing from every list:
  * **`main.py`:** ST-02, ST-04, ST-06 and ST-07 register handlers, routers and middleware, but none lists `main.py` as Modify.
  * **`config.py`:** later sub-tasks add settings to it but do not list it.
  * **`App.tsx`:** it routes to `WeekPage.tsx` and `isoWeek.ts`, which arrive in ST-09, but ST-09 does not list `App.tsx`.
  * **Bare names:** several entries are bare filenames (`router.py`, `service.py`).
  * **Missing files:** `__init__.py` files, `openapi.json`, `vite-env.d.ts`, the Vitest setup file, the MUI palette type augmentation and `package.json` changes.
  * Under the project rule "touch only the files it lists", every gap blocks the implementer.
* **Action:**
  * List shared files as Modify wherever they change, and say what each sub-task registers and in what order.
  * Write every path in full.
  * Add each missing file to the first sub-task that needs it.
* **Sources:** S9 B2, S9 M27, S6 #14, S9 M19 (`package.json` part).

### A4. Rewrite the update and delete rules as numbered steps with a fixed order of checks
* **Scope:** Lines 1462-1467, 1476-1478, 1506; lines 411-412 (Q3 step 3), 451, 672.
* **Problem:**
  * **Date-move wording:** "When the date changes, both days are locked ... and the entry's old minutes are subtracted" reads as subtracting the old minutes on a date move. That is the one case where they must not be subtracted. Read that way, a 4-hour entry could move onto a day that already has 22 hours.
  * **Order of checks:** the order of the 404, 409, 403 and 422 checks and the lock is not given, so a request with several faults could get any one of those errors.
  * **`If-Match`:** the router requires it on PUT only. Section 6 requires it on PUT and DELETE.
  * **Edge cases:** a missing task, a malformed `If-Match` and the `last_work_location` upsert have no defined result.
* **Action:**
  * **Numbered steps:** write numbered steps for update and delete, covering both date cases:
    * Date unchanged: existing - old + new + holiday ≤ 1,440.
    * Date changed: lock both days in ascending date order; check the new day only: existing + holiday + new ≤ 1,440.
  * **Order:** fix the order of checks.
  * **Edge cases:** give each one its error code.
  * **`If-Match`:** require it on both PUT and DELETE, with 428 when it is missing, and test both methods.
  * Apply the non-increasing rule from A6.
* **Sources:** S9 B3, S6 #13.

### A5. Add a short Risks table
* **Scope:** New section (none exists).
* **Problem:** The feature-spec standard says the spec must state its constraints and risks. The document has no Risks section. Stage 7 rated this a Blocker; Stage 8 agreed it is needed but asked for a short table, not a long section.
* **Action:** Add a table of about 8 rows: risk, likelihood or impact, mitigation or "accepted".
  * Give one "accepted; out of scope (production deployment)" row each to the single database, the single proxy, backups (NFR-DATA-4) and NFR-REL-1.
  * Add one row for lock-out abuse (see A10).
  * Do not design backups or high availability.
  * Several risks Stage 7 listed go away if A6, A11 and A29 are accepted: `day_totals` drift, the cleanup lock, and exports blocking requests.
* **Sources:** S7 Blocker 2; S8 Major "three tables".

---

## 🟡 Major

### A6. Decide on `day_totals`: Stage 8 recommends a per-day lock plus a fresh sum (Q3-B)
* **Scope:** Lines 401-436 (Q3, Q4), 599-605, 632, 749-758, 871-910, 1430-1431, 1441-1445, 1462-1466.
* **Problem:** The design chose Q3-A over Q3-B partly because `day_totals` has "a database CHECK as a last guard" (lines 876-878). That CHECK ignores holiday minutes, so it does not guard the 24-hour rule. Its trigger's handling of a missing row is unspecified, and it can lose minutes without any error. The earlier stages proposed four extra parts to keep the table correct:
  * a second trigger
  * a nightly check
  * a seed backfill
  * REPEATABLE READ for the week read

  The other reason given, RPT-2 speed, rests on an estimate of 50-150 ms (`decision_options.md`) against a 2 s target, for a Slice 4 report that has not been measured.
* **Action (author decision):**
  * **Recommended:** remove `day_totals` and its trigger. Each save then does three things:
    * takes `pg_advisory_xact_lock` on (employee, date), locking both days in date order on a move
    * sums that day's entries through the existing index
    * adds holiday minutes and checks the limit

    The week read sums the entries it already loaded. Keep the concurrency test.
  * **In either case,** add the non-increasing rule: reject a save only if the new day total is over 1,440 **and** higher than before the save. Without it, an employee whose day a holiday pushed to 28h cannot shorten an entry. This is the conflict with the design's own "HR is never blocked" rule (lines 58-62).
  * **If the table is kept,** Stage 9 M9 applies in full: specify the trigger as an upsert covering the old and new dates, and give `entry_minutes` `NOT NULL DEFAULT 0`. Stage 5 Major 6 then also needs a fix.
* **Sources:** S8 Major "Drop day_totals"; S5 Major 5, Major 6, Nit 3, Nit 4, Major 3 (index); S6 #3; S7 "CHECK last guard", "24-hour check blocks fixes", "trigger behaviour unspecified", "week read totals can disagree"; S9 M9 (trigger part), M13 (CHECK part).

### A7. Narrow the shared-hours promise to one calendar-facts function
* **Scope:** Lines 356-375 (Q1, especially 369-372), 633-634, 829-832, 1056-1058, 1271-1272, 1441-1445.
* **Problem:** The document says every report and the week read use `employee_day_hours`. That function returns whole-day totals for active employees only, with no employee filter. So it cannot serve RPT-1, RPT-3, the week read, or the history of deactivated employees (S-7). Expected minutes are also computed in Python in `read_week`, so the rule lives in two places. In Slice 1, `employee_day_hours` has no caller and no test.
* **Action:**
  * Keep one SQL function that returns expected minutes and holiday minutes per employee and date (extend `employee_holidays`).
  * Use it for both the week read and the 24-hour check, and give its full `RETURNS TABLE(...)` signature.
  * Remove `employee_day_hours` from Slice 1.
  * Restate the rule: "holiday and expected minutes come only from this function; project, task and kind breakdowns sum `time_entries` directly".
  * The Slice 4 plan says which reports include deactivated employees.
* **Sources:** S8 Major "Narrow the shared-hours promise"; S5 Major 1; S6 #11; S7 "`employee_day_hours` cannot be the only path"; S9 M9 (function part).

### A8. Add one "Scope by slice" table and fix slice labels
* **Scope:** Lines 20-26, 120-122, 819-837, 999-1064.
* **Problem:** The slice placement of the same items differs between sections:
  * AUD-2 and AUD-3 are in Slice 4 in Q8, partly in Non-Goals, and in Stretch in Future Improvements.
  * Exports and HOL-7 also have conflicting labels.

  Other gaps:
  * "S and C" and "Stretch" are never defined.
  * Slices 1-5 are first described at line 1041.
  * The Non-Goals list of designed S and C items is incomplete.
  * Some NFRs have no stated home.
  * Additional Resources and Future Improvements sit under the NFR heading.
  * "Kairos solves these" claims more than the core slices deliver.
* **Action:** Add one table near the Overview:
  * define M, S, C and Stretch
  * list each slice's PRD ID ranges
  * name only where the design departs from the PRD: TIME-17 early, AUD-2 and AUD-3 placement, and the export conflict (A9)
  * add one row per NFR group

  The table replaces the second Non-Goals bullet, section 8 and Future Improvements. Promote Additional Resources and Future Improvements (if kept) to `##`.
* **Sources:** S8 Major "three tables"; S7 "Slice labels disagree", "Non-Goals incomplete", "NFR coverage gaps", "nesting", "Kairos solves these"; S6 #7, #9, #12; S5 Nit 11.

### A9. Label exports (Q5) as Stretch and record the Slice 5 load-test conflict
* **Scope:** Lines 438-447, 724, 912-922, 1061.
* **Problem:** RPT-5 is an S requirement (PRD line 206), and PRD line 393 puts all S requirements in Stretch. Lines 445-447 plan a 100,000-row export in the Slice 5 load test, but exports are not built until after Slice 5. No earlier stage caught this (found in Stage 8 and checked against the PRD).
* **Action:**
  * Mark Q5 "(Stretch)" and keep the decision: stream in the request, with a cap per process.
  * Add an Open Question with an owner. Either Slice 5 proves NFR-PERF-4 for RPT-1 to RPT-3 only, or exports move ahead of Slice 5.
  * Record three correctness rules for when exports are built:
    * add tab and CR to the formula-prefix characters
    * close the connection without the final chunk on any error after streaming starts
    * enforce an overall deadline in the service, because `statement_timeout` applies to each FETCH separately
* **Sources:** S8 Major "Exports"; S5 Major 7; S7 "CSV export edge cases", "Q5 does not cite NFR-PERF-4".

### A10. State the process model: one uvicorn process per instance, one hash at a time
* **Scope:** Lines 227-245, 264-266, 395-399, 441-443, 1135-1139, 1297-1302.
* **Problem:** The document never says how many worker processes run per instance. The connection budget, the password-hash cap of 4 and the export cap all depend on that number.
* **Action:** State the following:
  * One uvicorn process per instance.
  * At most one password hash at a time per process, with a short bounded wait. NFR-PERF-3 requires no errors during a burst of 500 sign-ins.
  * The connection budget: 2 × (20 + 10), plus migrate and admin connections, is about 70, under the PostgreSQL default of 100.
  * Slice 5 decides whether to add a second worker.
  * Add a Risks row (A5) for lock-out abuse.
* **Unverified:** the plan relies on argon2-cffi releasing the GIL during hashing. Check this before writing the cap.
* **Sources:** S8 Major "1 worker per instance"; S5 Major 8, Major 7 (connection part); S7 "Process model not specified", "Sign-in lets anyone use up expensive hashing".

### A11. Carry the request ID with `set_config`, not a SQL comment
* **Scope:** Lines 270-289, 1007-1008, 1218-1227.
* **Problem:** Adding the request ID to every statement as a SQL comment through an event hook has two costs:
  * Each statement's text is unique, so the prepared-statement cache misses on the hot paths that carry the p99 < 100 ms target.
  * It contradicts the design's own rule that no SQL is built from strings.
* **Action:**
  * At the start of each transaction, set `kairos.request_id` (or `application_name`) in the same statement that sets `kairos.actor_id`. Do this for reads as well as writes.
  * Add `%a` to `log_line_prefix`.
  * Remove the event hook.
* **Sources:** S8 Major "Request ID"; S5 Major 2; S7 "Request-ID SQL comment"; S9 N9.

### A12. Add an error-code table to the API contract
* **Scope:** Section 6 (lines 642-725); line 301; ST-02 (lines 1198-1217, 1250); lines 1612-1615.
* **Problem:** Line 301 refers to an error-code list that does not exist. Codes are scattered across sections and mixed with codes from later slices. Several things are undefined:
  * the message text (NFR-UX-4 needs messages that say what to do next)
  * the format of `details.fields[].field`
  * the key that holds the current entry in `ENTRY_CHANGED`
  * the place of the request ID in an `INTERNAL_ERROR` body

  Two framework behaviours also break the stated error shape:
  * Starlette's own 404 and 405 responses return `{"detail": ...}`, so the "same shape for 404" test fails.
  * A handler registered for `Exception` runs outside the app's middleware, so the 500 response has no `X-Request-ID`.
* **Action:**
  * Add one table with code, HTTP status, exact message and `details` schema for every Slice 1 code.
  * Add `NOT_FOUND` and `METHOD_NOT_ALLOWED`, and register a handler for Starlette's `HTTPException`.
  * Fix `details.entry` for `ENTRY_CHANGED` and `details.requestId` for `INTERNAL_ERROR`.
  * Write `middleware.py` as a pure ASGI middleware that builds the 500 response itself.
* **Sources:** S8 Major "three tables"; S7 "Line 301 refers to an error-code list"; S9 M4.

### A13. Remove Slice 1 fields that no Slice 1 requirement needs, and list exactly what migration 0001 creates
* **Scope:** Lines 93-98, 570-571, 588-594, 670, 679-711; ST-03 (lines 1266-1272).
* **Problem:** Slice 1 includes items for later slices and Stretch:
  * the LEAVE kind and `leave_type`
  * `tasks.is_active` and `TASK_INACTIVE`
  * `flags`, `isFuture`, `editable` and `mustChangePassword`

  The reason given is weak, because the document itself says adding a column later is a one-line migration. ST-03 says to create tables "exactly as in the data dictionary", but the dictionary marks some items "Slice 3" or "From Slice 4". The `assignments (employee_id, start_date)` index appears only in the Q4 table.
* **Action:**
  * Remove the listed items from Slice 1, or keep each one with a stated Slice 1 reason.
  * Restate the design goal as "no destructive change to Slice 1 tables".
  * In ST-03, list each table, column, constraint and index that is in or out of 0001.
* **Sources:** S5 Major 3; S8 Nit "smaller items" (accept); S9 M8.

### A14. State the PRD context the document depends on, and mark estimates as estimates
* **Scope:** Lines 20-26, 58-72, 84, 89-90, 229, 350-498, 434, 853-1015, 955-956.
* **Problem:** A reader without the PRD cannot check the document's main arguments:
  * The eight PRD section 10 questions are never stated, although the design exists to answer them.
  * The rules TIME-8, TIME-9, HOL-2 and RPT-6 drive `editable`, `DATE_NOT_EDITABLE` and the month-lock resolution, but are never described.
  * "Friday peak" justifies several rejections and is never defined, and the database size is never given.
  * Performance and size figures read as measured facts.
  * Rejections say "Rejected by author" with no record of the decision.
* **Action:**
  * Add a one-line restatement above each Q decision.
  * Add one clause each for TIME-8, TIME-9, HOL-2 and RPT-6.
  * Add one sentence defining the Friday peak, citing PRD 7.2, and the sizing of the reference environment.
  * Add one sentence: "All figures are estimates; Slice 5 confirms them."
  * Replace the "Rejected by author:" prefix with the reason, or with a link to `decision_options.md`.
* **Sources:** S6 #1, #2, #4; S7 "Performance estimates stated as facts", "Rejections attributed to the author", "Q4 capacity sizing"; S8 Major "three tables" (accept list).

### A15. Put the Docker Hub question in Open Questions, and link the source of the Docker rule
* **Scope:** Lines 133-135, 975-978, 1020-1025, 137-152.
* **Problem:** "Docker is not permitted" drives the tooling choices but cites no source. The document admits it does not know whether the rule covers Docker Hub, yet every default image is `docker.io/...`. If the rule covers Docker Hub, `make up` breaks. The question sits in an NFR bullet instead of Open Questions.
* **Action:** Link the source of the rule. Add "Does the Docker rule cover Docker Hub images?" to Open Questions, with an owner and a deadline before ST-01.
* **Sources:** S6 #5; S7 "Docker Hub compliance question buried"; S8 Nit "smaller items".

### A16. Name every environment variable, database role and test-database setup step
* **Scope:** Lines 1101-1125, 1140-1146, 1233-1238, 1258-1262, 1293, 1338-1343, 1687-1689.
* **Problem:**
  * **Roles and URLs:** the owner and app roles have no named variables. One `DATABASE_URL` serves two roles and three hosts.
  * **Missing variables:** `POSTGRES_PASSWORD`, which the postgres image needs, is not listed, and the allowed values of `KAIROS_ENV` are not given.
  * **Test database:** the container on port 55433 does not mount `db/init/01-roles.sh`. So it has no `kairos_owner`, no `kairos_app` and no `btree_gist`, and the migration fails.
  * **Test roles:** the role used by tests and by the app under test is not named. TRUNCATE needs owner rights.
* **Action:**
  * List every variable with its name, default and whether it is required.
  * Say which file reads which variable.
  * Mount `db/init` in the test container.
  * State that the app under test uses `kairos_app`, and that fixtures and table clearing use `kairos_owner`.
* **Sources:** S9 M1; S6 #8; S8 Nit "smaller items" (accept: mount `db/init`).

### A17. Decide where tools run, and fix the database readiness wait
* **Scope:** Lines 1085-1100, 1135-1139, 1157-1188, 1640-1648, 1687-1694.
* **Problem:**
  * **Where tools run:** line 1099 says every target calls podman, and the README prerequisites are only Podman, podman-compose and make. But ruff, mypy, pytest, ESLint, tsc, Vitest and Playwright need Python or Node somewhere.
  * **Stack for e2e:** `make test` needs the running stack for e2e, and the plan does not say whether it starts it.
  * **Readiness wait:** `podman exec db pg_isready` uses the Unix socket. On first start, it can report ready while the init scripts are still running and before TCP is open.
* **Action:**
  * Decide host or container for each target and write its exact command.
  * Add host tools, with versions, to the README.
  * Run `pg_isready -h 127.0.0.1 -U <owner> -d kairos` in a retry loop with a timeout.
* **Sources:** S9 M2, M3.

### A18. Specify the async test setup
* **Scope:** Lines 1131-1134, 1218-1223, 1237-1244, 1310-1313, 1325.
* **Problem:**
  * **Event loops:** asyncpg pooled connections belong to one event loop. pytest-asyncio's default loop per test breaks them.
  * **Migrations:** async Alembic calls `asyncio.run()`, which fails inside an async fixture.
  * **Secure cookie:** httpx sends a `Secure` cookie only on `https` requests, so a client on `http://test` gets 401 on every signed-in call.
* **Action:**
  * Name the pytest-asyncio version and loop-scope settings, or use `NullPool` in tests.
  * Run migrations from a synchronous fixture or a subprocess.
  * Use `ASGITransport` with `base_url="https://test"`.
* **Sources:** S9 M6, M7.

### A19. Make the tests prove what the traceability table claims
* **Scope:** Lines 1322-1334, 1393-1410, 1480-1516, 1649-1680, 1695-1699.
* **Problem:**
  * **Concurrency test:** it can pass with locking broken. The CHECK turns a missing lock into a 500. Also, two 4-hour moves onto a 16-hour day reach exactly 1,440, which is allowed, so both succeed with or without a lock.
  * **Missing endpoint tests:** `GET /tasks` has no tests. There are no failure tests for `GET /me/week` or sign-out, and no `TASK_INACTIVE` test.
  * **AUTH-5:** no test checks the 8-hour expiry value.
  * **CUJ 5:** it has no end-to-end test.
  * **NFR rows:** NFRs delivered in Slice 1 have no rows in the traceability table.
* **Action:**
  * **Concurrency:** assert 409 (never 500) for the losing request, and that the final total equals the sum of entries. Give the second case its own setup that can fail: 17h on the target day, two 4-hour moves.
  * **Endpoint tests:** add `test_tasks.py` and the missing failure cases.
  * **AUTH-5:** add the 8-hour expiry test.
  * **CUJ 5:** add `cuj05-edit-delete-navigate.spec.ts`, and use one naming scheme for e2e files.
  * **Traceability table:** add rows for NFR-REL-3, NFR-OPS-3, NFR-SEC-1, NFR-SEC-3, NFR-DATA-1 and NFR-UX-5.
  * **Coverage claim:** narrow "every endpoint" to exclude `GET /health/live`.
* **Sources:** S9 M13, M14, M15, M16, M17.

### A20. Specify the end-to-end harness
* **Scope:** Lines 1166-1167, 1640-1656.
* **Problem:**
  * **Packages:** `@playwright/test` is in `frontend/package.json`, but the specs are in root `e2e/`, so Node cannot resolve the package from there.
  * **SQL helper:** it needs a Postgres client and a connection string, and neither is listed.
  * **Fixture data:** fixture employees need an argon2id hash. Fixture data goes into the development database, with no rule for unique values or cleanup.
* **Action:**
  * Add `e2e/package.json` and `e2e/tsconfig.json` (with `@playwright/test` and `pg`), or move the suite to `frontend/e2e/`.
  * Commit a precomputed hash for a fixed test password.
  * Add a run-ID suffix to fixture values and delete the fixture rows in teardown.
* **Sources:** S9 M18.

### A21. Specify the seed data and make it safe to re-run
* **Scope:** Lines 1303-1313, 1338-1379.
* **Problem:**
  * **Missing values:** employees, projects, assignments, dates and the entry-generation rule are not given.
  * **Re-runs:** assignment dates are relative to the run date. A run on a later day therefore inserts an overlapping assignment, which the `EXCLUDE` constraint rejects, so `make up` fails from the second day on.
  * **Email domain:** `@kairos.local` emails fail Pydantic `EmailStr`, because email-validator rejects special-use domains.
* **Action:**
  * Add a literal seed table with relative dates.
  * Define the natural key of an assignment as (employee code, project name).
  * Define `run(session, today)` so tests can set the run date.
  * Define the date range as "up to and including today (IST), Monday to Friday".
  * Use `str` for the email field, or switch the seed domain to `kairos.example.com`.
* **Sources:** S9 M10, M11.

### A22. Define the Week and Day response fields
* **Scope:** Lines 1423, 1441-1445, 1479, 1487; refs 682-697.
* **Problem:**
  * The format of `Day.weekday` is undefined.
  * The `complete` rule is not stated.
  * It is unclear whether totals include holiday minutes, and whether `GET /me/week` sets `ETag`.
* **Action:** Give the Pydantic type and the exact rule for each derived field, and list which endpoints set `ETag`.
* **Sources:** S9 M12.

### A23. Fix the frontend dependencies, the Content-Security-Policy and the development proxy
* **Scope:** Lines 1104-1105, 1157-1179, 1581-1595, 1626-1636.
* **Problem:**
  * **Missing packages:** Emotion (which MUI v6 needs), icons, a date picker, fonts, jsdom and the testing-library add-ons, and the React Router major version.
  * **Node version:** the proxy build uses Node 20, which reached end-of-life on 2026-04-30.
  * **CSP:** `default-src 'self'` with no `style-src` exception will probably block Emotion's runtime styles. Vite also inlines small fonts as `data:` URIs, which the policy blocks. (No reviewer tested this in a browser.)
  * **Clickjacking:** `frame-ancestors` does not fall back to `default-src`, so framing is not blocked.
  * **Dev proxy:** the Vite proxy points at `localhost:8000`, which is not published to the host.
* **Action:**
  * List every npm package with its major version, and use a Node 22 or 24 LTS image.
  * State the exact policy, for example `default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; frame-ancestors 'none'`.
  * Set `build.assetsInlineLimit: 0`.
  * Add an e2e check that styles are applied.
  * Point the dev proxy at `https://localhost:8443`, publish `127.0.0.1:8000`, or remove the proxy.
* **Sources:** S9 M19, M20, M25; S5 Nit 10; S7 "Security headers" (header part); S8 Nit "smaller items" (`frame-ancestors`).

### A24. Fix text contrast on the work-entry colour
* **Scope:** Lines 1528-1533, 1592-1595.
* **Problem:** White on `#00A67E` is about 3.1:1. NFR-UX-3 requires 4.5:1.
* **Action:** Set `contrastText` in `theme.ts`: `#1A1A1A` on work entries (about 5.6:1), and white on leave and holiday. Add a contrast or axe check.
* **Sources:** S9 M26.

### A25. Define the entry form and the keyboard path
* **Scope:** Lines 78-80, 144-148, 339-345, 680, 1601-1611, 1633, 1655.
* **Problem:**
  * **Date field:** the form has no Date field, though date moves are in scope.
  * **Counting unit:** G-1 counts "key presses" while the design counts "interactions".
  * **Keyboard tests:** they assert "4 interactions" even though Open Questions says the counting rule is unconfirmed.
  * **Defaults:** initial focus and the default location for a new employee (`lastWorkLocation` null) are not stated, so the keyboard path cannot save for that employee.
* **Action:**
  * Add a Date field, with its position and its default, or state that moves are API-only in Slice 1.
  * Reword G-1 to use NFR-UX-5's term.
  * Add: "The form opens with focus on Project. Work location defaults to `OFFICE` when there is no last-used location."
  * Record the confirmed counting rule, or mark the keyboard tests as blocked until it is confirmed.
* **Sources:** S9 M21, M22; S6 #10; S7 "G-1 counts key presses", "Keyboard path when there is no saved location".

### A26. Define where errors appear, simplify the conflict dialog, and assign the 401 redirect
* **Scope:** Lines 325-326, 449-459, 1527-1542, 1612-1615.
* **Problem:**
  * **Conflict actions:** "Show latest" and "Apply my changes again" have no defined behaviour.
  * **Errors without fields:** errors such as `DAILY_LIMIT_EXCEEDED` have no stated place on screen.
  * **401 redirect:** section 3 says `client.ts` redirects on any 401, but ST-08 says it only throws. So a save after the 8-hour expiry would show an error instead of the redirect that CUJ 6 needs.
  * **Scope of TIME-17:** the PRD puts TIME-17 in Stretch.
* **Action:**
  * **Slice 1:** keep `version`, `If-Match`, 428 and 409. The modal shows the 409 message and reloads the entry.
  * **Defer** the two-action dialog to Stretch, which removes the undefined "Apply my changes again".
  * **Error placement:** show errors without `details.fields` in `ErrorMessage` at the top of the modal.
  * **Redirect:** decide which component redirects on a 401 `SESSION_ENDED` (never on the sign-in call), and test it.
* **Sources:** S9 M23, M24; S5 Nit 6; S8 Nit "TIME-17"; S6 #15 ("Apply my changes again").

### A27. State the log redaction rule exactly
* **Scope:** Lines 1209-1211, 1251-1252.
* **Problem:** The rule leaves three questions open: case sensitivity, exact or substring matching (which decides whether `password_hash` is redacted), and whether `Set-Cookie` is covered. The test depends on the answers.
* **Action:** Write the rule. For example: "Case-insensitive exact match on `password`, `token`, `cookie`, `set-cookie` and `authorization`. Values are replaced with `[REDACTED]`."
* **Sources:** S9 M5.

### A28. Remove semicolons from Mermaid message text
* **Scope:** Section 7 (lines 727-818).
* **Problem:** Semicolons in sequence-diagram message text will probably break rendering. **Unverified:** no reviewer rendered the diagrams.
* **Action:** Replace each `;` in message text with a comma or "and", or escape it as `#59;`. Render the diagrams once to confirm.
* **Sources:** S7 "Mermaid sequence diagrams use semicolons"; S8 Nit "smaller items".

---

## 🟢 Nits

### A29. Remove the session-cleanup background task
Sign-in and deactivation already clear expired sessions. Removing the task removes:
* the only background process
* the lock-type question
* the ST-04 lifespan registration that Stage 9 B2 asks to specify

*Sources:* S5 Major 4 (lowered per S8); S8 Nit; S7 "audit_log insert grant and cleanup lock type" (lock half); S9 B2 (cleanup-task part).

### A30. Add Caddy health checks and a graceful shutdown
Add the following, and nothing more:
* `health_uri /api/v1/health/ready`
* passive failure detection
* a short `lb_try_duration`
* uvicorn's graceful-shutdown timeout
* one sentence on where Caddy runs

*Sources:* S7 "Caddy load balancing has no health checks" (Major, lowered per S8); S5 Nit 1; S8 Nit; S9 N6.

### A31. Remove server ETags from `/tasks` and `/holidays`
They save almost nothing, and the `/tasks` handling is incomplete. *Sources:* S5 Nit 12; S9 N10; S8 Nit "smaller items".

### A32. Re-check project availability only when the project or the date changes
Today, editing only the duration of an entry on a project that is no longer available returns 403 (PRJ-10). *Sources:* S7 "Edits are blocked on entries whose project is no longer available"; S9 B3 (edge-case part); S8 Nit.

### A33. State module boundaries and departures from the theme
* **Shared component:** a shared component imports from a feature folder. Move it or invert the dependency.
* **Helper names:** the plan names helper functions and constants. Keep only the names that other sub-tasks depend on.
* **Search box:** the disabled top-bar search box departs from `design-theme.md` line 7 if it is removed. State whichever choice is made.
* **`employee_preferences`:** keep it unless the author accepts "latest entry by date" as an approximation of "used last".

*Sources:* S5 Nit 5, 7, 8, 9; S9 N13; S8 Nit "smaller items" (defer).

### A34. Clean up wording and structure
* **Structure:**
  * chosen decisions are bold paragraphs while rejected options have H3 headings
  * the scenario-to-slice note sits under CUJ 6
  * the failure-column format varies in the endpoint table
  * design goals have no IDs
  * the infrastructure list lacks alternatives and has a weak rationale
* **Overstated claims:**
  * "every route" (security checks)
  * "SQLite cannot test triggers"
  * "cannot drift apart unnoticed"
  * "cannot be met with any project selector"
* **Misplaced text:** Q1 behaviour rules are filed under "Cost:".
* **Wording:**
  * the same statements repeat across sections, and there are filler phrases and loose wording in Alternatives
  * "brainstorming" is process jargon
  * project-availability wording is inconsistent
  * small terms are unexplained: "Arjun", AUTH-3, bare PRD IDs. Link to `prd.md` rather than adding a glossary.
* **Contradiction:** the S-5 message contradicts PRJ-4. This is a content error, not only wording.
* **Implementation plan:** the traceability table format is inconsistent, some sentences are wordy or vague, and some terms drift.

*Sources:* S7 Nits (bold vs H3; infrastructure list; scenario note; failure column; design goal IDs; "every route"; SQLite; "cannot drift"; "cannot be met"; "Cost:"; repeated statements; filler; loose wording; availability wording and S-5; "brainstorming"); S6 #15 (terms); S9 N23, N24, N25.

### A35. Settle the smaller implementation details in the Slice 1 plan
These are open details from Stage 9. Each is small; together they are the remaining guesswork for the implementer:
* function signatures (N1)
* the uvicorn target vs `main.py` (N2)
* README contradictions (N3)
* image build details (N4)
* Makefile target details (N5)
* Caddy setup (N6, with A30)
* password-handling parameters (N7)
* clearing the cookie with the same attributes (N8)
* the `q` search parameter (N11)
* the readiness test engine swap (N12)
* Firefox not tested (N14)
* the time zone for "current week" (N15)
* display strings and enum values (N16)
* `activity` and duration validation (N17)
* the call path for `available_projects` (N18)
* the scope of database grants (N19)
* CUJ 6 wording vs the e2e fixture (N20)
* small missing tests (N21)
* theme values (N22)
* pool and test parameters (N26)
* "fonts installed locally" (N27)

*Sources:* S9 N1-N8, N11, N12, N14-N22, N26, N27.

### A36. Move deferred questions to the slice that owns them
* **Holiday on a leave day:** add it to the Slice 3 Open Questions.
* **`audit_log` permissions:** a SECURITY DEFINER `audit_row()` and revoking INSERT go to the AUD-3 Stretch plan.
* **`/metrics`:** restricting access goes to the Slice 5 plan.

*Sources:* S7 "A holiday added on a day that already has leave", "audit_log insert grant" (grant half), "Security headers and /metrics exposure" (metrics half); S8 Nit "smaller items".

---

## Recommendations not taken

Stage 8 resisted these proposals from earlier stages because they add parts to the design without a matching need. They are recorded so the author can overrule the call, not as action items.

| Proposal | Raised by | Why not taken |
| --- | --- | --- |
| Second (BEFORE UPDATE) trigger on `day_totals` that calls `employee_holidays` | S5 Major 6 | Only needed if `day_totals` stays (see A6). |
| Nightly totals-consistency check; seed backfill procedure; REPEATABLE READ week read | S7 | Same: they exist only to support `day_totals`. |
| Write-then-check reordering of the save flow | S5 Nit 4 | Same. |
| Row-level `reportable_hours` view, second database role, grant enforcement, EXPLAIN check at 20M rows | S5 Major 1 ("generalize" option) | Speculative machinery for Slice 4 and Stretch reports that have no plan yet. A7 covers the real gap. |
| Static scan of repository SQL against an allow-list | S7 | Same. |
| Separate export process, Caddy routing for `*.csv`, global advisory-lock slots | S5 Major 7 | About 10 HR users of a Stretch feature. A9 records the decision and defers the CPU-isolation choice to a measurement. |
| Per-IP sign-in rate limit; abusive sign-in load test; per-account plus per-IP lock-out | S7 | Standard Caddy has no rate-limit directive. Employees likely share a few egress IPs, so a per-IP limit could refuse the burst NFR-PERF-3 requires. It also changes what AUTH-4 means. Recorded as a Risks row (A5, A10). |
| `audit_row()` dropping credential columns and skipping counter-only updates | S7 | Puts table-specific exceptions in a generic function. A1 removes the need. |
| "audit_log never contains password_hash" test | S7 | A1 makes it unnecessary; the existing "every audited table has its trigger" test covers it. |
| Return the affected `Day` from each mutation instead of reloading the week | S5 Nit 2 | Optimisation without a measurement. Once A6 is applied, the reload is one indexed query. |
| A source and a test for every figure | S7 | One sentence marking figures as estimates is enough (A14). |
| Glossary of PRD IDs | S6 #15 | Link to `prd.md` instead (A34). |
| Separate Risks section, NFR mapping table, requirement-to-slice table, slice table, business-rules list, glossary and Friday-peak paragraph as seven additions | S6, S7 | Merged into three tables (A5, A8, A12) plus a few lines (A14). |

---

## Traceability Matrix

Every raw finding from the five stage reports, mapped to the action item that covers it. "Not taken" refers to the table above.

### Stage 5: Initial Zoom Out ([zoom_out_report.md](zoom_out_report.md))

| Finding | Severity | Action |
| --- | --- | --- |
| Major 1. Shared hours function fits one report, drops deactivated employees | Major | A7 (generalize option: Not taken) |
| Major 2. Request-ID SQL comment defeats prepared-statement caching | Major | A11 |
| Major 3. Slice 1 carries schema and API fields for later slices | Major | A13; AUTH part A1; `day_totals` index A6 |
| Major 4. Daily session-cleanup task with no proven need | Major | A29 |
| Major 5. 24-hour rule blocks every edit on a holiday-pushed day | Major | A6 |
| Major 6. Database guard ignores holiday minutes, can miss rows | Major | A6 (second trigger: Not taken) |
| Major 7. CSV export CPU-bound on the event loop | Major | A9; connection budget A10 (export process: Not taken) |
| Major 8. Password-hash cap sized for memory, not CPU | Major | A10 |
| Nit 1. Caddy does not use the readiness endpoint | Nit | A30 |
| Nit 2. Every save followed by a full week reload | Nit | Not taken |
| Nit 3. Week read gets totals from two places | Nit | A6, A7 |
| Nit 4. Q3 save flow could be one step shorter | Nit | A6 (reordering: Not taken) |
| Nit 5. `employee_preferences` is a table for one value | Nit | A33 |
| Nit 6. Reason for TIME-17 in Slice 1 does not hold | Nit | A26 |
| Nit 7. Plan names helper functions and constants | Nit | A33 |
| Nit 8. Shared component depends on a feature module | Nit | A33 |
| Nit 9. Disabled search box has nothing to search | Nit | A33 |
| Nit 10. Vite dev proxy points at an unpublished port | Nit | A23 |
| Nit 11. Slice 4 audit triggers include stretch tables | Nit | A8 |
| Nit 12. ETags on `/tasks` and `/holidays` save almost nothing | Nit | A31 |

### Stage 6: Comprehension ([comprehension_report.md](comprehension_report.md))

| Finding | Severity | Action |
| --- | --- | --- |
| #1 Locking and editing rules referenced but not described | Major | A14 |
| #2 PRD section 10 questions never stated | Major | A14 |
| #3 No rule for editing a day already over 24 hours | Major | A6, A4 |
| #4 "Friday peak" and reference environment undefined | Major | A14 |
| #5 Docker rule has no source; default images may break it | Major | A15 |
| #6 Sign-in counters write audited rows; `password_hash` in audit | Major | A1 |
| #7 "S and C" and "Stretch" undefined | Major | A8 |
| #8 Test database roles and extensions not set up | Major | A16 |
| #9 Slices 1-5 introduced late | Nit | A8 |
| #10 G-1 key presses vs interactions; no default location | Nit | A25 |
| #11 Expected minutes defined in two places | Nit | A7 |
| #12 AUD-2 and AUD-3 slice placement inconsistent | Nit | A8 |
| #13 428 for DELETE missing from router and tests | Nit | A4 |
| #14 Files missing from sub-task lists | Nit | A3 |
| #15 Small terms unexplained | Nit | A34; "Apply my changes again" A26; glossary: Not taken |

### Stage 7: Critic ([critic_report.md](critic_report.md))

| Finding | Severity | Action |
| --- | --- | --- |
| Credential and lock-out columns on the audited `employees` table | Blocker | A1 (column-dropping `audit_row()`: Not taken) |
| No Risks and Mitigations section | Blocker | A5 |
| Slice labels for audit, exports and HOL-7 disagree | Major | A8 |
| Non-Goals list of S and C items incomplete | Major | A8 |
| Rejections attributed to "the author" | Major | A14 |
| NFR section coverage gaps and false cross-reference | Major | A8 |
| Additional Resources and Future Improvements nested under NFR | Major | A8 |
| Mermaid diagrams use semicolons | Major | A28 |
| Line 301 refers to a missing error-code list | Major | A12 |
| G-1 counts key presses, design counts interactions | Major | A25 |
| "Kairos solves these" claims more than core slices deliver | Major | A8 |
| `day_totals` CHECK called a last guard | Major | A6 |
| Request-ID SQL comment | Major | A11 |
| Performance and size estimates stated as facts | Major | A14 (a source and a test per figure: Not taken) |
| Docker Hub question buried in an NFR bullet | Major | A15 |
| 24-hour check blocks fixes on holiday-pushed days | Major | A6 |
| `employee_day_hours` cannot be the only path | Major | A7 (SQL scan: Not taken) |
| `day_totals` trigger behaviour unspecified | Major | A6 (nightly check, backfill, REPEATABLE READ: Not taken) |
| Process model per instance not specified | Major | A10 |
| Sign-in lets anyone use up hashing; overload undefined | Major | A10 (per-IP limit: Not taken) |
| Caddy load balancing has no health checks | Major → Nit | A30 |
| Chosen decisions bold, rejected options H3 | Nit | A34 |
| Missing alternatives, weak rationale in infrastructure list | Nit | A34 |
| Scenario-to-slice note unclear, under CUJ 6 | Nit | A34 |
| Failure column format varies | Nit | A34 |
| Design goals have no IDs | Nit | A34 |
| "Every route" overstates security checks | Nit | A34 |
| SQLite "cannot test triggers" inaccurate | Nit | A34 |
| "Cannot drift apart unnoticed" omits the check | Nit | A34 |
| "Cannot be met with any project selector" too absolute | Nit | A34 |
| Behaviour rules filed under "Cost:" in Q1 | Nit | A34 |
| Same statements repeated across sections | Nit | A34 |
| Filler and redundant phrases | Nit | A34 |
| Loose or subjective wording in Alternatives | Nit | A34 |
| Q4 capacity sizing uses 5 years and "mostly fits" | Nit | A14 |
| Availability wording inconsistent; S-5 message contradicts PRJ-4 | Nit | A34 |
| "Brainstorming" is process jargon | Nit | A34 |
| Q5 does not cite NFR-PERF-4; overstates the plan | Nit | A9 |
| Edits blocked on entries whose project is no longer available | Nit | A32 |
| Holiday added on a day that already has leave | Nit | A36 |
| Week read totals can disagree with its entries | Nit | A6 |
| CSV export edge cases | Nit | A9 |
| Keyboard path with no saved location | Nit | A25 |
| Security headers and `/metrics` exposure | Nit | A23 (headers), A36 (metrics) |
| `audit_log` insert grant and cleanup lock type | Nit | A36 (grant), A29 (lock) |

### Stage 8: Intermediate Zoom Out ([intermediate_zoom_out_report.md](intermediate_zoom_out_report.md))

| Finding | Severity | Action |
| --- | --- | --- |
| Credential columns on audited `employees`: split them out | Blocker | A1 |
| Drop `day_totals`: per-day lock plus fresh sum | Major | A6 |
| Narrow the shared-hours promise | Major | A7 |
| Exports: label Q5 Stretch, record Slice 5 conflict | Major | A9 |
| State "1 worker per instance" | Major | A10 |
| Merge documentation additions into three tables | Major | A5, A8, A12, A14 |
| Request ID via `set_config` | Major | A11 |
| Remove the session-cleanup task | Nit | A29 |
| Caddy health checks, nothing more | Nit | A30 |
| TIME-17: keep version check, defer dialog | Nit | A26 |
| Smaller items: accept, resist or defer | Nit | A13, A15, A16, A23, A28, A31, A32, A33, A36 |

### Stage 9: Implementation Readiness ([implementation_readiness_report.md](implementation_readiness_report.md))

| Finding | Severity | Action |
| --- | --- | --- |
| B1. Sub-tasks depend on later files; build fails until ST-10 | Blocker | A2 |
| B2. `main.py`, `config.py`, `App.tsx` not listed as Modify | Blocker | A3; cleanup-task part A29 |
| B3. `update_work_entry` date-move wording, order of checks | Blocker | A4; PRJ-10 edge case A32 |
| M1. Env vars and DB connection users not specified | Major | A16 |
| M2. Host or container for linters and tests | Major | A17 |
| M3. `pg_isready` can pass early | Major | A17 |
| M4. Error catalogue, messages, `details`, framework errors | Major | A12 |
| M5. Log redaction rule ambiguous | Major | A27 |
| M6. Async test setup not specified | Major | A18 |
| M7. Test clients will not send the `Secure` cookie | Major | A18 |
| M8. Migration "exactly as in the data dictionary" | Major | A13 |
| M9. Function and trigger contracts incomplete | Major | A7 (functions), A6 (trigger) |
| M10. Seed data unspecified; re-run fails | Major | A21 |
| M11. `.local` seed emails fail `EmailStr` | Major | A21 |
| M12. Week and Day fields under-defined | Major | A22 |
| M13. Concurrency test can pass with broken locking | Major | A19 (CHECK part also A6) |
| M14. Endpoint tests missing | Major | A19 |
| M15. Traceability table leaves out NFRs | Major | A19 |
| M16. No test for 8-hour expiry | Major | A19 |
| M17. CUJ 5 has no e2e test; mixed naming | Major | A19 |
| M18. E2E harness under-specified | Major | A20 |
| M19. Frontend dependencies incomplete; Node 20 EOL | Major | A23; `package.json` lists A3 |
| M20. CSP will probably break MUI styling | Major | A23 |
| M21. Entry form has no Date field | Major | A25 |
| M22. Keyboard tests rest on an open question | Major | A25 |
| M23. Conflict actions and fieldless errors unspecified | Major | A26 |
| M24. Which component redirects on 401 | Major | A26 |
| M25. Vite proxy points at an unpublished port | Major | A23 |
| M26. White on work colour fails WCAG AA | Major | A24 |
| M27. Bare filenames; supporting files missing | Major | A3 |
| N1-N8 | Nit | A35 (N6 also A30) |
| N9. Request-ID SQL comment inconsistent and costly | Nit | A11 |
| N10. `GET /tasks` `ETag` handling incomplete | Nit | A31 |
| N11, N12 | Nit | A35 |
| N13. Shared component imports from a feature folder | Nit | A33 |
| N14-N22 | Nit | A35 |
| N23-N25 | Nit | A34 |
| N26, N27 | Nit | A35 |
