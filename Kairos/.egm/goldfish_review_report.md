# Goldfish Annotated Review Report

**Target:** `Kairos/docs/designs/kairos.md` (`full file`, lines 1-1704)
**Findings in scope:** the 5 Blockers (A1-A5) and 23 Majors (A6-A28) in [consolidated_review_report.md](consolidated_review_report.md). Nits A29-A36 are out of scope. Two of them (A29, A32) are folded into A3 and A4, because those fixes remove the text the Nits are about.

## Summary

| Category | Count | Items |
| --- | --- | --- |
| PROPOSE FIX | 28 | A1-A28 |
| CONTEST | 0 | |
| ESCALATE | 0 | |
| DECLINE | 0 | |

I checked each finding against the document and the PRD. None was inaccurate:
* **Checked against the PRD:**
  * TIME-6 includes holiday hours.
  * PRD section 9 puts all S requirements in Stretch, including RPT-5, AUD-2, AUD-3 and HOL-7.
  * PRD section 7.1 is the only source of the Docker rule.
  * MUI core has no date picker.
* **Checked against the document:**
  * The `day_totals` CHECK ignores holidays.
  * `main.py` is never listed as Modify after ST-01.

**Three fixes change decisions you recorded on 2026-11-19. Check them before you accept:**
1. **A6** replaces the Q3 decision (`day_totals` table) with Q3-B (a per-day advisory lock and a fresh sum), and makes Q4 "indexes only". A fallback edit that keeps `day_totals` is given under A6.
2. **A13** removes from Slice 1 several fields you added so that later slices would not need schema changes:
   * the LEAVE kind and `leave_type`
   * `tasks.is_active` and `TASK_INACTIVE`
   * `flags`, `isFuture`, `editable` and `mustChangePassword`

   Later slices add them as new columns, CHECK values and response fields.
3. **A26** reduces the TIME-17 conflict dialog in Slice 1 to one action. The two-action dialog moves to Stretch.

**Two new business-rule choices are in the edits.** They are the reviewers' recommendations, not PRD text:
* **Non-increasing rule (A4, A6):** a save is rejected only if the day would be over 24 hours *and* higher than before the save.
* **Busy sign-in (A10):** sign-in returns 503 `SIGN_IN_BUSY` after waiting 2 seconds for a hashing slot.

**One open fact is not settled by the edits:** whether `argon2-cffi` releases the Python GIL while hashing (A10). The edit records it as a risk and as a check in ST-04.

---

## Prioritized Findings & AI Suggestions

### [A1] [`egm-design`, `egm-zoom-out`, `egm-comprehension`] - Severity: Blocker
*   **Affected Scope:** [Lines 468-480](../docs/designs/kairos.md#L468-L480) (Q8), [Lines 508-524](../docs/designs/kairos.md#L508-L524) (`employees`), [Lines 821-823](../docs/designs/kairos.md#L821-L823) (section 8), [Lines 1266-1270](../docs/designs/kairos.md#L1266-L1270) (ST-03), [Lines 1297-1321](../docs/designs/kairos.md#L1297-L1321) (ST-04)
*   **Discrepancy Description:** `password_hash`, `failed_sign_in_count` and `locked_until` sit on `employees`, which Slice 4 audits. Sign-in counter writes would then happen with no actor, so `audit_row()` raises an error. Q8 also copies whole rows as JSON, so `password_hash` would land in `audit_log`.
*   **Rationale:** Create `employee_credentials` in Slice 1 and do not audit it. Slice 2 adds the AUTH-2 and AUTH-4 columns there.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed against the document:
    * Q8 attaches `audit_row()` to `employees` (line 470) and raises an error when there is no actor (lines 471-473).
    * Sign-in is unauthenticated, so it has no actor.
    * Q8 writes old and new values as JSON for master data (line 473).
    * The design already moved `last_work_location` out of `employees` for the same reason (line 539), so this follows the document's own precedent.
*   ***Proposed Edit:***
    *   Location: `employees` table, lines 521-524
        *   Change: delete the rows `password_hash`, `must_change_password`, `failed_sign_in_count` and `locked_until`.
    *   Location: after the `employees` table (after line 524)
        *   Change: insert:
            ```markdown
            **`employee_credentials`** (Slice 1. Not audited.)

            | Column | Type | Rules |
            |---|---|---|
            | `employee_id` | BIGINT | Primary key, foreign key to `employees`. |
            | `password_hash` | TEXT | argon2id encoded string. Never logged, never returned, never copied into `audit_log`. |

            Slice 2 adds `must_change_password` (AUTH-2), `failed_sign_in_count` and `locked_until` (AUTH-4) to this table. Sign-in reads and writes only this table and `sessions`, so it never writes an audited row and needs no actor. This follows the same reason as `employee_preferences`.
            ```
    *   Location: Q8, lines 478-480
        *   Change: "`employee_preferences`, `sessions` and `day_totals` are not audited: they hold working state, not records." → "`employee_credentials`, `employee_preferences` and `sessions` are not audited: they hold secrets or working state, not records."
    *   Location: section 8 Slice 2 bullet, line 821-822
        *   Change: "the AUTH-2 to AUTH-4 columns already in `employees`" → "the AUTH-2 to AUTH-4 columns, added to `employee_credentials`"
    *   Location: ST-03 migration list
        *   Change: add `employee_credentials` after `employees`.
    *   Location: ST-04 sign-in bullet
        *   Change: add "The password hash is read from `employee_credentials`."
    *   Location: ST-03 `factories.py` (see A2)
        *   Change: the employee factory also writes an `employee_credentials` row.

### [A2] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Blocker
*   **Affected Scope:** [Lines 1076-1260](../docs/designs/kairos.md#L1076-L1260) (ST-01 to ST-03), [Lines 1638-1661](../docs/designs/kairos.md#L1638-L1661) (ST-10), [Lines 1684-1694](../docs/designs/kairos.md#L1684-L1694) (Verification Plan)
*   **Discrepancy Description:** Sub-tasks depend on files from later sub-tasks:
    * `make up` needs Alembic (ST-03), `seed.py` (ST-05) and `src/main.tsx` (ST-08).
    * `make lint` needs `export_openapi` (ST-08).
    * `make test` calls targets that are not defined.
    * The ST-01 health tests need ST-02 files.
    * The ST-02 fixtures need ST-03 and ST-04.

    No sub-task before ST-10 leaves the build passing.
*   **Rationale:** Either reorder the sub-tasks or grow the shared files step by step, and give each sub-task a checkpoint.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed:
    * Line 1088 has `up` run `migrate` and `seed` in ST-01.
    * Lines 1147-1156 put a readiness check that needs `db.py` and `errors.py` in ST-01.
    * Lines 1237-1244 have ST-02 fixtures run the ST-03 migration and use ST-04 sign-in.

    `engineering-standards.md` requires `make up`, `make test` and `make lint` to exist after ST-01. The project rule "touch only the files it lists" makes the problem a hard stop for the implementer. The fix combines a small reordering with growing the shared files one step at a time.
*   ***Proposed Edit:***
    *   Location: after line 1079 ("`implementation-plan.md` will be generated from this section.")
        *   Change: insert:
            ```markdown
            **Rules for every sub-task**

            *   When a sub-task is done, `make up`, `make lint` and `make test` pass. Each sub-task ends with a **Done when** line that names any further check.
            *   Shared files grow one sub-task at a time. A sub-task that changes a file created earlier lists it as Modify and says what it adds.
            *   Every path is written in full from `Kairos/`. Each new Python package directory gets an empty `__init__.py`, listed in the sub-task that creates the directory.
            ```
    *   Location: ST-01 `Makefile` item (lines 1085-1100)
        *   Change: ST-01 defines only targets whose inputs exist. The targets then grow:

            | Target | ST-01 | Added later |
            | --- | --- | --- |
            | `up` | Copies `.env` if missing, starts `db`, waits (A17), then starts `api` and `proxy` | `migrate` step (ST-02), `seed` step (ST-05) |
            | `down`, `logs`, `psql` | As now | |
            | `test-backend` | Throwaway database details moved here from the Verification Plan, with A16's `db/init` mount | |
            | `test-frontend` | `npm --prefix frontend run test` | |
            | `test` | Runs `test-backend` and `test-frontend` | `test-e2e` (ST-10) |
            | `lint` | ruff check, ruff format --check, mypy --strict, ESLint, `tsc --noEmit` | `openapi` target and an `openapi-check` step inside `lint` (ST-08) |
    *   Location: ST-01 `compose.yaml`
        *   Change: services `db`, `api` and `proxy` only. The `migrate` service is added in ST-02 and the `seed` service in ST-05, each listed as Modify there.
    *   Location: ST-01 items 9-10 (`health/router.py`, `test_health.py`)
        *   Change: move both to ST-02, together with `backend/kairos/health/__init__.py`.
        *   Change: add to ST-01 `backend/tests/__init__.py` and `backend/tests/test_app.py` (Create). The test checks that `create_app()` serves the OpenAPI document at `/api/v1/openapi.json` with title "Kairos". It needs no database.
    *   Location: ST-01 frontend item 11
        *   Change: add `frontend/src/main.tsx` (Create, a minimal page that renders "Kairos", replaced in ST-08), `frontend/src/vite-env.d.ts` and `frontend/src/test/setup.ts`.
        *   Change: add `frontend/src/smoke.test.ts` (Create, checks that the test setup loads), so that Vitest has a test from ST-01. It is removed in ST-08 (Delete).
    *   Location: ST-03 item 1 (Alembic configuration)
        *   Change: move `backend/alembic.ini`, `backend/migrations/env.py`, `backend/migrations/script.py.mako` and `backend/migrations/versions/.gitkeep` to ST-02. `alembic upgrade head` is then a no-op until ST-03.
    *   Location: ST-02 `conftest.py` and `factories.py` (lines 1233-1246)
        *   Change: ST-02 creates `conftest.py` only. Its contents are given in A16 and A18, and it adds a test-only router to the test app with routes that raise each error code and accept a POST.
        *   Change: `factories.py` moves to ST-03 (Create).
        *   Change: `signed_in_client(employee)` moves to ST-04 (`conftest.py`, Modify).
    *   Location: ST-02 `test_errors_and_logging.py` (lines 1247-1254)
        *   Change: tests use the test-only router:
            * The error shape is the same for 401, 403, 404 (an unknown path), 405, 422 and 500.
            * The 500 response carries `X-Request-ID` and `details.requestId`.
            * A POST without the CSRF header gets 403.
            * The redaction filter replaces the five keys of A27 at any depth.
        *   Change: the sign-in log-line test moves to `test_auth.py` (ST-04).
    *   Location: ST-08
        *   Change: add `Makefile` (Modify: `openapi` target and `openapi-check` step in `lint`), `frontend/src/main.tsx` (Modify) and `frontend/src/smoke.test.ts` (Delete).
    *   Location: ST-10
        *   Change: add `Makefile` (Modify: `test-e2e`, and `test` also runs it).
    *   Location: end of each sub-task
        *   Change: add a **Done when** line. For example ST-01: "Done when: `make up` serves the placeholder page at `https://localhost:8443` and `/api/v1/openapi.json`, and `make lint` and `make test` pass."
    *   Location: Verification Plan, lines 1687-1691
        *   Change: point to the ST-01 target definitions instead of defining `test-backend` there.

### [A3] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`, `egm-comprehension`] - Severity: Blocker
*   **Affected Scope:** [Lines 1140-1146](../docs/designs/kairos.md#L1140-L1146) (ST-01 `main.py`, `config.py`), all ST file lists, [Lines 1520-1527](../docs/designs/kairos.md#L1520-L1527) (ST-08 `App.tsx`), [Lines 1319-1321](../docs/designs/kairos.md#L1319-L1321) (cleanup task)
*   **Discrepancy Description:** Some files are changed by sub-tasks that do not list them:
    * `main.py` gains registrations in ST-02, ST-04, ST-06 and ST-07.
    * `config.py` gains settings later.
    * `App.tsx` routes to ST-09 files.

    Some entries use bare filenames, and supporting files are missing: `__init__.py` files, `openapi.json`, `vite-env.d.ts`, the Vitest setup file and the MUI type augmentation.
*   **Rationale:** List shared files as Modify where they change, write full paths, and add the missing files.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed:
    * ST-01 says `create_app()` "registers middleware and routers" (lines 1143-1146), but no later sub-task lists `main.py`.
    * Lines 1303-1304, 1383-1384, 1393-1394, 1626-1628 and 1649-1652 use bare names.
    * `openapi.json` is described at line 1548 but not listed.
*   ***Proposed Edit:***
    *   Location: ST-01 `main.py`, `config.py`
        *   Change: "`create_app()` builds the FastAPI app with title "Kairos" and OpenAPI at `/api/v1/openapi.json`. Later sub-tasks register their middleware and routers here (listed as Modify). `config.Settings` defines every setting Slice 1 uses (A16), so later sub-tasks do not change `config.py`."
    *   Location: ST-02
        *   Change: add `backend/kairos/main.py` (Modify). It registers, outermost first:
            1. the request-ID and access-log middleware, which also builds the `INTERNAL_ERROR` response (A12);
            2. exception handlers for `AppError`, `RequestValidationError` and Starlette's `HTTPException`;
            3. the health router.

            Add `backend/kairos/security/__init__.py`.
    *   Location: ST-04
        *   Change: add `backend/kairos/main.py` (Modify: auth router and `GET /me`), `backend/kairos/auth/__init__.py`, `backend/tests/api/__init__.py` and `backend/tests/conftest.py` (Modify: `signed_in_client`).
        *   Change: delete the "start-up task ... every 24 hours" sentence (lines 1319-1321). Change Q2 lines 392-394 to "Sign-in deletes that employee's expired sessions, and deactivation deletes all of an employee's sessions (Slice 2). Nothing else deletes them in Slice 1." This takes in Nit A29 and removes the lifespan hook B2 asked to specify.
    *   Location: ST-06
        *   Change: add `backend/kairos/main.py` (Modify: projects and tasks routers), `backend/kairos/projects/__init__.py` and `backend/kairos/tasks/__init__.py`.
    *   Location: ST-07
        *   Change: add `backend/kairos/main.py` (Modify: time-entries router) and `backend/kairos/time_entries/__init__.py`.
    *   Location: ST-03
        *   Change: add `backend/tests/db/__init__.py`.
    *   Location: ST-08 / ST-09 (frontend routes)
        *   Change: move `frontend/src/features/week/isoWeek.ts` and `isoWeek.test.ts` to ST-08.
        *   Change: ST-08 creates `frontend/src/features/week/WeekPage.tsx` as a placeholder that shows the ISO week, so `App.tsx` routes are complete in ST-08. ST-09 lists `WeekPage.tsx` as Modify and does not change `App.tsx`.
    *   Location: ST-08 item 4
        *   Change: add `frontend/src/api/openapi.json` (Create, generated). Add "`export_openapi` builds the app without connecting to the database and without required settings."
    *   Location: ST-08 `theme.ts`
        *   Change: add "It also declares the MUI type augmentation for the `work`, `leave`, `holiday` palette entries and the `numeric` text style."
    *   Location: bare filenames
        *   Change: write full paths. `backend/kairos/auth/schemas.py`, `backend/kairos/auth/router.py` and so on, and the same for `projects/`, `tasks/`, `frontend/src/features/week/*.test.ts(x)` and the e2e specs (A20).

### [A4] [`egm-style`, `egm-implementation-readiness`, `egm-comprehension`] - Severity: Blocker
*   **Affected Scope:** [Lines 1462-1467](../docs/designs/kairos.md#L1462-L1467) (`update_work_entry`, `delete_work_entry`), [Lines 1476-1479](../docs/designs/kairos.md#L1476-L1479) (router), [Lines 1505-1507](../docs/designs/kairos.md#L1505-L1507) (tests), [Line 671](../docs/designs/kairos.md#L671) (PUT row)
*   **Discrepancy Description:**
    * "When the date changes ... the entry's old minutes are subtracted" reads as subtracting the old minutes on a date move.
    * The order of the 404, 409, 403 and 422 checks and the lock is not given.
    * `If-Match` is required on PUT only in the router.
    * Edge cases have no defined result: a missing task, a malformed `If-Match`, and the location upsert.
*   **Rationale:** Write numbered steps for update and delete covering both date cases, fix the order of checks, and require `If-Match` on both PUT and DELETE.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 1462-1464 and 1476-1478. Read literally, the date-move sentence lets 22h + 4h - 4h pass. The edit uses `lock_day` and a fresh day sum, which works with A6 accepted or not. It also adds the non-increasing rule (Stage 6 #3, Stage 7, Stage 8). That rule is the only way an employee can shorten an entry on a day a holiday pushed over 24 hours, and it fits the existing "HR is never blocked" rule (lines 58-62). It folds in Nit A32: availability is re-checked only when the project or the date changes (PRJ-10).
*   ***Proposed Edit:***
    *   Location: replace lines 1462-1467
        *   Change: replace with:
            ```markdown
            *   **`update_work_entry(employee, entry_id, version, values)`:** inside `transaction(actor)`, in this order. A request with several problems gets the first error in the list.
                1.  Reads the entry with `get_entry_for_owner`. No row, including another employee's entry, gives 404 `ENTRY_NOT_FOUND` (TIME-7, NFR-SEC-3).
                2.  If the entry's `version` differs from `If-Match`, returns 409 `ENTRY_CHANGED` with the current entry in `details.entry`.
                3.  If the project or the date changes, checks `available_projects(..., project_id)` for the new date, otherwise 403 `PROJECT_NOT_AVAILABLE`. An edit that keeps both the project and the date is not checked again, so an employee can still correct an entry on a project that has since closed (PRJ-10).
                4.  If the task changes, checks that it exists, otherwise 422 `VALIDATION_FAILED` on `taskId`.
                5.  Locks the days with `lock_day`: one day when the date is unchanged, or both days in ascending date order when it changes.
                6.  Sums the target day's entries after the lock (`day_entry_minutes`) and adds its holiday minutes. Then:
                    *   Date unchanged: new total = day minutes - old minutes + new minutes + holiday minutes.
                    *   Date changed: new total = target-day minutes + new minutes + holiday minutes. The old day's total only goes down, so it is not checked.
                    If the new total is over 1,440 **and** higher than the target day's total before the save, returns 409 `DAILY_LIMIT_EXCEEDED`.
                7.  Runs `update_entry_if_version`. If it returns None because another save changed the entry after step 2, returns 409 `ENTRY_CHANGED`.
                8.  Updates `last_work_location` (TIME-3).
            *   **`delete_work_entry(employee, entry_id, version)`:** reads the entry with `get_entry_for_owner`, otherwise 404 `ENTRY_NOT_FOUND`. Then runs `delete_entry_if_version`. If no row is deleted, returns 409 `ENTRY_CHANGED` with the current entry. A delete only lowers a day's total, so it takes no day lock (TIME-7).
            ```
    *   Location: `create_work_entry` step 2 (line 1449)
        *   Change: "Checks the task exists, otherwise 422 `VALIDATION_FAILED` on `taskId`." (see A13)
    *   Location: router lines 1476-1479
        *   Change: "`PUT` and `DELETE /api/v1/me/time-entries/{id}` both require `If-Match: "<version>"`. Missing: 428 `VERSION_REQUIRED`. Present but not a quoted positive integer, including `*` and `W/"3"`: 422 `VALIDATION_FAILED` on field `If-Match`. These checks run before the service. POST (201) and PUT (200) set `ETag: "<version>"`. `GET /me/week` sets no `ETag`."
    *   Location: tests lines 1505-1507
        *   Change: replace with:
            * A stale `If-Match` gives 409 `ENTRY_CHANGED` with `details.entry`.
            * A missing `If-Match` gives 428 on both PUT and DELETE, and a malformed one gives 422.
            * Moving a 4-hour entry onto a day with 22 hours gives 409 `DAILY_LIMIT_EXCEEDED`, not 500.
            * On a day with 20 hours of entries and an 8-hour holiday inserted in the database, shortening a 5-hour entry to 4 hours is accepted and lengthening it is rejected.
            * Editing only the duration of an entry whose project has since closed is accepted.
            * A PUT that hits several errors gets the first error in the step order.
    *   Location: The Problem, first bullet (after line 62)
        *   Change: add "On such a day an employee can still shorten or delete entries: a save is rejected only when it would leave the day over 24 hours *and* higher than before the save."
    *   Location: section 6 PUT row (line 671)
        *   Change: add "422 (malformed `If-Match`)". Section 6 DELETE row: "404 `ENTRY_NOT_FOUND`, 409 `ENTRY_CHANGED`, 428 `VERSION_REQUIRED`, 422".

### [A5] [`egm-structural`, `egm-design`] - Severity: Blocker
*   **Affected Scope:** No Risks section ([Lines 999-1025](../docs/designs/kairos.md#L999-L1025) is the nearest section)
*   **Discrepancy Description:** The feature-spec standard says the spec must state its constraints and risks. The document has no Risks section.
*   **Rationale:** Add a short Risks table of about 8 rows. Accept the single-instance and backup risks as out of scope. Do not design high availability.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed: no section names risks. Several risks are already acknowledged in passing (lines 128-132, 1024-1025, 144-148) but are scattered.
*   ***Proposed Edit:***
    *   Location: new `## Risks` section after `## Non-Functional Requirements` (before Additional Resources)
        *   Change: insert:
            ```markdown
            ## Risks

            | Risk | Impact | Decision or mitigation |
            |---|---|---|
            | One PostgreSQL instance holds all state. | Kairos is down while it is down. | Accepted. Failover and the 99.5% availability target (NFR-REL-1) belong to a production deployment, which is out of scope (Non-Goals). |
            | Backups for 7-year retention (NFR-DATA-4) are not designed. | Data loss if the database volume is lost. | Accepted for this design. The schema never deletes employees or projects. Backups belong to the deployment design. |
            | One Caddy proxy fronts both API instances. | Kairos is down while it is down. | Accepted, for the same reason as the database. |
            | Timings, sizes and rates in this document are estimates. | A decision may not hold at 20 million entries. | The Slice 5 load test measures them. If one fails, this document is updated before the code (section 8). |
            | Password hashing during a sign-in burst competes with interactive requests for CPU. | Slower week reads during the Monday-morning burst. | One hash at a time per process (Q2). This relies on `argon2-cffi` releasing the Python GIL while hashing; ST-04 checks this before the cap is fixed. Slice 5 measures NFR-PERF-3. |
            | Anyone who knows a work email can lock that account for 15 minutes with 5 wrong passwords (AUTH-4, Slice 2). | One employee cannot sign in for 15 minutes. | Accepted: AUTH-4 defines lock-out per account. A per-IP limit is not used, because most employees share a few office network addresses and NFR-PERF-3 requires 500 sign-ins in 60 seconds to succeed. |
            | A future write path to `time_entries` skips the day lock. | Two concurrent saves could pass 24 hours. | Only `time_entries/service.py` writes `time_entries`, and every save goes through `lock_day`. The concurrency test covers create and move. |
            | The Docker rule may also cover Docker Hub images. | `make up` cannot pull its default images. | Open Question. Every image name can be overridden (Non-Functional Requirements). |
            | The NFR-UX-5 counting rule is not confirmed. | The keyboard tests may check the wrong limit. | Open Question. ST-09 and ST-10 are not done until it is answered. |
            ```

### [A6] [`egm-zoom-out`, `egm-inventive-catalyst`, `egm-design`, `egm-comprehension`] - Severity: Major
*   **Affected Scope:** [Lines 246-248](../docs/designs/kairos.md#L246-L248), [Lines 401-436](../docs/designs/kairos.md#L401-L436) (Q3, Q4), [Lines 599-605](../docs/designs/kairos.md#L599-L605) (`day_totals`), [Line 632](../docs/designs/kairos.md#L632), [Lines 749-758](../docs/designs/kairos.md#L749-L758) (S-1 diagram), [Lines 871-910](../docs/designs/kairos.md#L871-L910) (Q3-B, Q4-C), [Lines 1269-1271](../docs/designs/kairos.md#L1269-L1271), [Lines 1289-1290](../docs/designs/kairos.md#L1289-L1290), [Lines 1430-1431](../docs/designs/kairos.md#L1430-L1431), [Line 1673](../docs/designs/kairos.md#L1673)
*   **Discrepancy Description:** The Q3 choice rests on two claims that do not hold:
    * **"A database CHECK as a last guard" (lines 876-878).** The CHECK ignores holiday minutes, so it does not guard the 24-hour rule.
    * **The RPT-2 benefit (lines 906-910).** It rests on an unmeasured estimate (50-150 ms against a 2 s target).

    The table also causes problems of its own:
    * Its trigger's handling of a missing row and of date moves is unspecified.
    * It can drift from the entries.
    * Earlier stages proposed four extra parts to keep it correct: a second trigger, a nightly check, a seed backfill and REPEATABLE READ.
*   **Rationale:** Choose Q3-B: a per-day advisory lock and a fresh sum. Add the non-increasing rule in either case.
*   ***AI Suggestion:*** PROPOSE FIX (**reverses the Q3 and Q4 decisions of 2026-11-19; needs your explicit choice**)
*   ***Justification:*** Both factual points are confirmed:
    * `CHECK (entry_minutes BETWEEN 0 AND 1440)` (line 605) counts entry minutes only, while TIME-6 includes holiday hours.
    * Q4-C's rejection (lines 908-910) is circular: it keeps `day_totals` because Q3 chose it.

    The table's protection only partly makes up for a write path that forgets the lock, which was the Q3-B objection: it still does not enforce TIME-6 with holidays. Removing it takes away a table, a trigger, a write on every save and the drift risk, and closes about nine findings. The remaining risk (a path that skips the lock) gets a Risks row (A5). This proposal goes against your recorded decision, so **if you want to keep `day_totals`, use the fallback edit below instead**.
*   ***Proposed Edit (recommended, Q3-B):***
    *   Location: section 1 `db` bullet, line 247-248
        *   Change: "row locks for the 24-hour rule (Q3)" → "per-day advisory locks for the 24-hour rule (Q3)".
    *   Location: Q3, replace lines 401-420
        *   Change: replace with:
            ```markdown
            **Q3. A per-day advisory lock and a fresh sum.** Creating, editing or moving an entry runs in one transaction:

            1.  `lock_day` runs `SELECT pg_advisory_xact_lock(:employee_id, :entry_date - DATE '2000-01-01')`, the two-integer form, which locks that employee's day. Employee IDs stay far below 2^31. A concurrent save for the same employee and date waits here. PostgreSQL releases the lock at commit or rollback.
            2.  `day_entry_minutes` sums `duration_minutes` of that employee's entries on that date through the `(employee_id, entry_date)` index (a handful of rows). The service adds the holiday minutes from `employee_calendar` (Q1).
            3.  The service applies the new duration (for an edit on the same date, minus the old one). If the result is over 1,440 minutes and higher than the day's total before the save, it rolls back with 409 `DAILY_LIMIT_EXCEEDED`.
            4.  It inserts or updates the entry. Commit.

            An edit that moves an entry to another date locks both days, earliest date first, so two such edits cannot deadlock, and checks only the new date. Only `time_entries/service.py` writes `time_entries`, and every save in it goes through `lock_day` (see Risks). HR never takes this lock when adding a holiday (see The Problem).
            ```
    *   Location: Q4 heading and table, lines 422-436
        *   Change: the heading becomes "**Q4. One composite index, no pre-computed totals, no partitioning.**" Delete both `day_totals` rows.
        *   Change: after the table, add "RPT-2 (Slice 4) sums about 70,000 entries for one week through the `(entry_date, project_id)` index. `decision_options.md` estimates this at 50 to 150 ms against a 2-second target. If the Slice 5 load test shows it is too slow, a summary table is added then."
    *   Location: data dictionary `day_totals` (lines 599-605)
        *   Change: delete the table.
    *   Location: functions table, line 632
        *   Change: delete the `time_entries_totals()` row.
    *   Location: S-1 diagram, lines 749-754 and 758
        *   Change: replace the two `day_totals` lines with `API->>DB: pg_advisory_xact_lock(priya, today)` and `API->>DB: SUM(duration_minutes) for priya, today`. Use `employee_calendar` in place of `employee_holidays`. Delete `Note over DB: trigger adds 480 to day_totals`. Line 758 becomes `API->>DB: entries for week + employee_calendar`.
    *   Location: Alternatives, "Q3-B" section (lines 871-878)
        *   Change: replace with:
            ```markdown
            ### Q3-A: A per-day totals row, locked by the save

            *   **Pros:** A trigger keeps the total, so a write path that skips the service check still cannot push a day's entry minutes past 1,440. RPT-2 could read the stored totals.
            *   **Cons:** The database `CHECK` cannot include holiday minutes, so it does not guard TIME-6 as a whole. The trigger must handle a missing row and date moves, and the table can drift from the entries. It adds a table, a trigger and a second write to every save.
            *   **Reason for Rejection:** The extra protection is partial, and the RPT-2 benefit rests on an estimate that Slice 5 will measure. Chosen first on 2026-11-19, then replaced by Q3-B after the design review.
            ```
    *   Location: Alternatives, "Q4-C" section (lines 903-910)
        *   Change: replace with:
            ```markdown
            ### Q4-A: Pre-computed per-day totals (`day_totals`)

            *   **Pros:** RPT-2 reads at most 35,000 total rows for a week instead of about 70,000 entries.
            *   **Cons:** It needs the totals table and trigger that Q3-A was rejected for.
            *   **Reason for Rejection:** The raw weekly sum is estimated at 50 to 150 ms against a 2-second target for about 10 HR users. Slice 5 measures it, and a summary is added only if that test fails.
            ```
    *   Location: ST-03, lines 1269-1271 and 1289-1290
        *   Change: remove `day_totals` and the `time_entries_totals()` trigger from the migration. Delete the trigger test.
    *   Location: ST-07 repository, lines 1430-1431
        *   Change: "`lock_day(employee_id, date)`, which runs `pg_advisory_xact_lock` for that employee and date." Add "`day_entry_minutes(employee_id, date)`, which returns the fresh sum of that day's entry minutes."
    *   Location: ST-07 `create_work_entry` step 4
        *   Change: "Checks that `day_entry_minutes` plus holiday minutes plus the new minutes are at most 1,440."
    *   Location: ST-07 `read_week`
        *   Change: "Day and week totals are summed from the entries it loaded plus holiday minutes."
    *   Location: traceability row TIME-6, line 1673
        *   Change: "`lock_day`, `day_entry_minutes`" in place of "`day_totals` trigger".
*   ***Fallback edit (if you keep `day_totals`):***
    *   Change: in Q3, change "is a last guard if the service check is ever skipped" to "guards entry minutes only. It does not include holiday minutes, so the service check is the only guard for TIME-6."
    *   Change: specify the trigger as `INSERT ... ON CONFLICT (employee_id, entry_date) DO UPDATE` for the old and the new date of every insert, update and delete, and give `entry_minutes` `NOT NULL DEFAULT 0`.
    *   Change: `read_week` sums totals from the entries it loaded.
    *   Change: add the non-increasing rule (A4) and a Risks row for drift.
    *   Change: rewrite Q4-C's rejection to cite the RPT-2 estimate instead of Q3.

### [A7] [`egm-zoom-out`, `egm-inventive-catalyst`, `egm-design`, `egm-comprehension`, `egm-codebase-details`] - Severity: Major
*   **Affected Scope:** [Lines 356-375](../docs/designs/kairos.md#L356-L375) (Q1), [Lines 633-634](../docs/designs/kairos.md#L633-L634), [Lines 824-832](../docs/designs/kairos.md#L824-L832), [Lines 1271-1272](../docs/designs/kairos.md#L1271-L1272), [Lines 1291-1292](../docs/designs/kairos.md#L1291-L1292), [Lines 1428-1429](../docs/designs/kairos.md#L1428-L1429), [Lines 1441-1445](../docs/designs/kairos.md#L1441-L1445), [Lines 763-791](../docs/designs/kairos.md#L763-L791) (S-2 diagram)
*   **Discrepancy Description:** The document says every report and the week read use `employee_day_hours`. That function returns whole-day totals for active employees only, with no employee filter, so it cannot serve RPT-1, RPT-3, the week read or deactivated employees (S-7). Expected minutes are also worked out in Python in `read_week`. `employee_day_hours` has no caller or test in Slice 1.
*   **Rationale:** Use one calendar-facts function for expected and holiday minutes, remove `employee_day_hours` from Slice 1, and restate the rule.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 364-371 and 1443-1444. To avoid a second function for reports across many employees later, the proposed function takes an array of employee IDs. The week read passes one ID, and Slice 4 passes many.
*   ***Proposed Edit:***
    *   Location: Q1, replace lines 356-375
        *   Change: replace with:
            ```markdown
            **Q1. Holiday entries are worked out when a week is read.** Holidays are stored once in `holidays`. One database function, created in the Slice 1 migration, is the only place that turns working days and holidays into minutes:

            *   `employee_calendar(employee_ids BIGINT[], from_date DATE, to_date DATE)` returns one row per employee and date: `employee_id`, `day`, `expected_minutes` (480 on a weekday on or after the employee's date of joining, otherwise 0), `holiday_minutes` (480 for a weekday holiday, 0 for a weekend holiday (HOL-6) or no holiday), and `holiday_name` (null when there is no holiday). Holidays before the date of joining are not returned. It is `STABLE` and reads `employees` and `holidays`.

            The week read and the 24-hour check both call it, so expected and holiday minutes are defined in one place. Rule for later slices: holiday and expected minutes come only from this function, and project, task and kind breakdowns sum `time_entries` directly. Each report's plan states whether it includes deactivated employees: RPT-2 does not, and RPT-3 does (S-7). **Cost:** adding a holiday writes one row. Reading a week adds one indexed lookup on a table of about 15 rows a year. A weekend holiday returns 0 minutes but is still shown in the calendar (HOL-6). Work logged on a holiday is added on top of the holiday hours, which also satisfies HOL-7 (Stretch) at no extra cost.
            ```
    *   Location: functions table, lines 633-634
        *   Change: replace both rows with `| `employee_calendar(employee_ids, from, to)` | 1 | Expected and holiday minutes per employee per date (Q1). |`.
    *   Location: section 8, lines 825-826 and 829-830
        *   Change: "both already call `employee_calendar`". Slice 4: "the report endpoints (expected and holiday minutes from `employee_calendar`; each report states whether it includes deactivated employees)".
    *   Location: ST-03 lines 1271-1272 and test lines 1291-1292
        *   Change: the migration "Creates the `employee_calendar()` function." The test becomes "`employee_calendar` returns expected 480 on a weekday and 0 at a weekend, 0 before the date of joining, holiday 480 on a weekday and 0 at a weekend, and no holiday before the date of joining."
    *   Location: ST-07 repository, lines 1428-1429
        *   Change: "`calendar_for_range(employee_id, from, to)`, which calls `employee_calendar`."
    *   Location: ST-07 `read_week`, lines 1441-1445
        *   Change: "takes expected and holiday minutes from `calendar_for_range`" in place of "(480 on each weekday on or after the date of joining)".
    *   Location: S-2 diagram, line 786
        *   Change: `API->>DB: employee_calendar([priya], Monday, Sunday)`.

### [A8] [`egm-structural`, `egm-design`, `egm-style`, `egm-comprehension`, `egm-zoom-out`] - Severity: Major
*   **Affected Scope:** [Lines 17-32](../docs/designs/kairos.md#L17-L32) (Overview), [Lines 50-53](../docs/designs/kairos.md#L50-L53), [Lines 120-122](../docs/designs/kairos.md#L120-L122), [Lines 468-474](../docs/designs/kairos.md#L468-L474) (Q8), [Lines 819-837](../docs/designs/kairos.md#L819-L837), [Lines 999-1064](../docs/designs/kairos.md#L999-L1064)
*   **Discrepancy Description:**
    * AUD-2, AUD-3, exports and HOL-7 get different slices in different sections.
    * "S and C" and "Stretch" are undefined.
    * Slices are described late.
    * The Non-Goals list is incomplete and its "one line each" promise is not kept.
    * Some NFRs have no home, and line 1010's "each NFR-PERF target" claim is false.
    * Additional Resources and Future Improvements sit under the NFR heading.
    * "Kairos solves these" claims more than the core slices deliver.
*   **Rationale:** Add one "Scope by slice" table that defines the labels and records the departures from the PRD. Add an NFR mapping. Promote the headings.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed against the PRD:
    * PRD section 5 defines M, S and C.
    * PRD section 9 puts all S and C requirements in Stretch.
    * AUD-2, AUD-3, RPT-5, HOL-7 and TIME-17 are all S.
    * In the document, Q8 is labelled "Slice 4" and covers AUD-2 and AUD-3, while Future Improvements puts both in Stretch. Q5 says "Slice 4 and stretch".
*   ***Proposed Edit:***
    *   Location: end of Overview (after line 32)
        *   Change: insert:
            ```markdown
            ### Scope by slice

            The PRD gives each requirement a priority: **M** (must have), **S** (should have) or **C** (could have). PRD section 9 puts the M requirements in Slices 1 to 5 and every S and C requirement in **Stretch**, which is work after Slice 5 with no fixed order.

            | Slice | PRD requirements | Where this design departs from PRD section 9 |
            |---|---|---|
            | 1. Sign in and log time | AUTH-1, AUTH-5, TIME-1 to TIME-3, TIME-5 to TIME-7, TIME-14 to TIME-16, PRJ-4, PRJ-9 | Also delivers the TIME-17 version check (Q6), without the conflict dialog. |
            | 2. HR master data | EMP-1 to EMP-5, EMP-7, EMP-8, PRJ-1 to PRJ-3, PRJ-5 to PRJ-8, PRJ-10, AUTH-2 to AUTH-4, AUTH-6 | None. |
            | 3. Holidays and leave | HOL-1 to HOL-6, TIME-4, TIME-8, TIME-9, TIME-11, TIME-12 | None. HOL-7 (Stretch) holds with no extra work because of Q1. |
            | 4. Reports and audit | RPT-1, RPT-2, RPT-4, AUD-1 | `audit_log` is created with INSERT and SELECT only for the API user, which is part of AUD-3. |
            | 5. Performance proof | NFR-PERF-1 to NFR-PERF-9 | The export part of NFR-PERF-4 cannot be tested here, because exports are Stretch. See Open Questions. |
            | Stretch | All S and C requirements: AUTH-7, AUTH-8, EMP-6, HOL-7, HOL-8, PRJ-11 to PRJ-14, TIME-10, TIME-13, TIME-17, TIME-18, RPT-3, RPT-5, RPT-6, AUD-2, AUD-3 | Designed here but not planned: TIME-17 (Q6), RPT-5 (Q5), AUD-2 and AUD-3 (Q8), RPT-6 (The Problem). The others have no design. |

            Each slice gets its own detailed implementation plan before it is built. Section 8 of the Technical Plan says how each slice builds on Slice 1.
            ```
    *   Location: line 25-26
        *   Change: "Slices 2 to 5 are listed as future work and will get their own implementation plans." → "Later slices are listed in Scope by slice."
    *   Location: The Problem, lines 50-53
        *   Change: replace with "Slices 1 to 4 address these. Employees get one weekly screen in which they can only choose projects HR has opened to them (Slice 1). Designated holidays are added to every employee's week automatically (Slice 3). HR sees in one screen who has an incomplete week (Slice 4), and every change to master data is recorded (Slice 4). Closing a month (RPT-6) and the audit of time entries (AUD-2) are Stretch."
    *   Location: Non-Goals, lines 120-122
        *   Change: replace with "**S and C requirements without a design.** They are listed under Stretch in Scope by slice. Five are designed here because a PRD section 10 question or a PRD conflict depends on them: TIME-17 (question 6), RPT-5 (question 5), AUD-2 and AUD-3 (question 8), and RPT-6 (the HOL-2 conflict in The Problem). The others get no design."
    *   Location: Q5 heading, line 439
        *   Change: "(Slice 4 and stretch)" → "(Stretch)" (with A9).
    *   Location: Q8, lines 468-476
        *   Change: the heading becomes "**Q8. Database triggers with a required actor (Slice 4 for master data, Stretch for time entries).**" The trigger list becomes "attached in Slice 4 to `employees`, `holidays`, `projects` and `assignments` (AUD-1). Stretch attaches it to `time_entries` (AUD-2) and `period_locks` (RPT-6)."
    *   Location: section 8, Slice 2 bullet
        *   Change: append "Last-HR-admin protection (EMP-8) takes a row lock on the HR admin count, so two demotions at once cannot both pass. The department question in Open Questions is decided first." (moved from Future Improvements)
    *   Location: Future Improvements, lines 1039-1064
        *   Change: delete the section. Its content is now in Scope by slice and section 8.
    *   Location: line 1029
        *   Change: "### Additional Resources" → "## Additional Resources".
    *   Location: NFR Performance bullet, lines 1010-1015
        *   Change: "Section 4 explains the approach for NFR-PERF-2, NFR-PERF-3 and NFR-PERF-7; the table below maps every NFR. ... The riskiest targets are p99 under 100 ms for the week read and entry save during the Friday peak, because they are the most frequent calls and the save takes the day lock, and sign-in p99 under 500 ms during a 500-user burst, because each sign-in runs a deliberately slow hash."
    *   Location: end of the NFR section
        *   Change: add an NFR mapping table:
            ```markdown
            | NFR | Where it is met |
            |---|---|
            | NFR-PERF-1 to NFR-PERF-3, NFR-PERF-5, NFR-PERF-7 to NFR-PERF-9 | Approach in section 1 (process model) and section 4 (Q2, Q3, Q4, Q7). Proven in Slice 5. |
            | NFR-PERF-4 | Reports: Slice 4 (Q1, Q4), proven in Slice 5. Exports: Stretch (Q5), see Open Questions. |
            | NFR-PERF-6 | The weekly view is the static app plus one week request. Measured in Slice 5. |
            | NFR-SEC-1, NFR-SEC-3 to NFR-SEC-6, NFR-SEC-8 | Slice 1: argon2id (Q2), `/me` paths (section 2), HTTPS through Caddy, CSRF header, bound parameters, the error table (section 6), `.env`. |
            | NFR-SEC-2 | Authentication from Slice 1. Role checks from Slice 2, the first slice with HR-only routes. |
            | NFR-SEC-7 | Slice 1 returns personal data only through `/me`. HR routes from Slice 2 check the HR role. |
            | NFR-REL-1 | Out of scope (Non-Goals, production deployment). See Risks. |
            | NFR-REL-2, NFR-REL-3 | Slice 1: stateless API (section 1), sessions in PostgreSQL (Q2), liveness and readiness endpoints. |
            | NFR-DATA-1 | Day lock (Q3) and the assignments exclusion constraint in Slice 1. Unique holiday date in Slice 3. |
            | NFR-DATA-2, NFR-DATA-3 | Slice 1: Alembic migrations, `DATE` and `TIMESTAMPTZ` in UTC (section 2). |
            | NFR-DATA-4 | The application never deletes employees or projects. Backups belong to the deployment (Risks). |
            | NFR-UX-1 | Playwright runs Chromium and WebKit from Slice 1. Firefox is not tested. |
            | NFR-UX-2 to NFR-UX-5 | Slice 1: layout, theme, focus and contrast (section 3), messages (section 6), keyboard path (section 3, Open Questions). |
            | NFR-OPS-1, NFR-OPS-2, NFR-OPS-5, NFR-OPS-6 | ST-01: `make up`, `make lint`, `make test`, coverage gate, README. |
            | NFR-OPS-3 | Request ID from browser to database (section 2), Slice 1. |
            | NFR-OPS-4 | `/metrics`, Slice 5. |
            ```

### [A9] [`egm-zoom-out`, `egm-inventive-catalyst`, `egm-design`] - Severity: Major
*   **Affected Scope:** [Lines 438-447](../docs/designs/kairos.md#L438-L447) (Q5), [Lines 912-922](../docs/designs/kairos.md#L912-L922) (Q5-B), [Lines 137-152](../docs/designs/kairos.md#L137-L152) (Open Questions)
*   **Discrepancy Description:**
    * RPT-5 is an S requirement, so exports are Stretch, after Slice 5. Yet the Slice 5 load test plans a 100,000-row export.
    * CSV edge cases are missing: tab and CR prefixes, behaviour on a mid-stream error, and an overall deadline.
*   **Rationale:** Label Q5 Stretch, keep the decision, and add an Open Question with an owner. Record the correctness rules.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Verified against the PRD:
    * PRD line 206 makes RPT-5 an S requirement.
    * Line 393 puts all S requirements in Stretch.
    * Line 392 assigns NFR-PERF-1 to NFR-PERF-9 to Slice 5.
    * NFR-PERF-4 includes the export target.

    The design's line 445-447 cannot be met in that order.
*   ***Proposed Edit:***
    *   Location: Q5, lines 438-447
        *   Change: the heading becomes "(Stretch)". "Each instance allows 1 export" → "Each API process allows 1 export". Replace the last sentence ("The Slice 5 load test runs...") with:
            > "Exports are built after Slice 5, because RPT-5 is an S requirement. Their plan adds a load run that holds a 100,000-row export during the NFR-PERF-1 load and checks NFR-PERF-2 (see Open Questions). That run also decides whether CSV formatting moves to a worker thread or process. Three rules apply when exports are built:
            > - tab and carriage return are added to the characters that get a leading `'`;
            > - an error after streaming has started closes the connection without the final chunk, so the browser reports a failed download instead of saving a short file;
            > - the service enforces a 30-second deadline for the whole export, because `statement_timeout` applies to each fetch separately."
    *   Location: Q5-B rejection, line 922
        *   Change: "The Slice 5 load test checks this." → "The export load run checks this."
    *   Location: Open Questions (after line 152)
        *   Change: add:
            > "**Exports and the Slice 5 load test (NFR-PERF-4).** PRD section 9 puts NFR-PERF-1 to NFR-PERF-9 in Slice 5 but RPT-5 exports in Stretch, after Slice 5. One of two choices applies:
            > - Slice 5 proves NFR-PERF-4 for RPT-1 to RPT-3 only, and exports get their own load run when they are built;
            > - exports move ahead of Slice 5.
            >
            > Owner: the author, before Slice 5 is planned."

### [A10] [`egm-design`, `egm-zoom-out`, `egm-inventive-catalyst`] - Severity: Major
*   **Affected Scope:** [Lines 238-245](../docs/designs/kairos.md#L238-L245) (section 1), [Lines 264-266](../docs/designs/kairos.md#L264-L266), [Lines 395-399](../docs/designs/kairos.md#L395-L399) (Q2 hashing), [Lines 1135-1139](../docs/designs/kairos.md#L1135-L1139) (Containerfile), [Lines 1297-1302](../docs/designs/kairos.md#L1297-L1302) (`passwords.py`)
*   **Discrepancy Description:** The number of worker processes per instance is never stated. The connection budget, the hash cap of 4 (sized for memory, not CPU) and the export cap all depend on it.
*   **Rationale:** State one uvicorn process per instance, cap hashing at one per process with a bounded wait, and state the connection budget.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed:
    * No process count appears anywhere.
    * With 4 concurrent hashes on 2 vCPU per instance, an argon2id burst can occupy every core, including the one running the event loop.
    * The reference environment is 2 vCPU per instance (NFR-PERF-7).

    The GIL-release claim is unverified, so the edit makes ST-04 check it and lists it in Risks (A5). Returning 503 after a 2-second wait is a new choice. At about 8 sign-ins a second against a capacity of about 25, the wait should not occur under NFR-PERF-3's load.
*   ***Proposed Edit:***
    *   Location: section 1, after the `api` bullet (line 245)
        *   Change: add:
            > "**Process model:** each instance runs one uvicorn process with one event loop (no `--workers`). Each process has these limits:
            > - a pool of 20 database connections plus up to 10 more when busy;
            > - 1 password hash at a time (Q2);
            > - 1 export at a time with its own pool of 2 connections (Stretch, Q5).
            >
            > Two instances therefore open at most 2 × (30 + 2) = 64 connections. Migrations, the seed and a database shell add a few more, which stays under PostgreSQL's default `max_connections` of 100. Slice 5 decides whether a second process per instance is worth adding. If it is, the pool sizes are halved."
    *   Location: section 2 database access, line 265
        *   Change: "Each API instance has a pool" → "Each API process has a pool".
    *   Location: Q2 hashing bullet, lines 395-399
        *   Change: replace with:
            > "**Password hashing:** argon2id via `argon2-cffi`, with 3 passes, 64 MiB and 1 lane. Hashing runs in a worker thread so that it does not block the event loop.
            > - **Cap:** each API process runs at most 1 hash at a time. A sign-in waits up to 2 seconds for its turn, then returns 503 `SIGN_IN_BUSY`.
            > - **Capacity:** at about 50 to 80 ms per hash, two instances handle about 25 to 40 sign-ins a second. The planned load is about 8 a second: 500 sign-ins in 60 seconds (NFR-PERF-3).
            > - **CPU:** the cap of 1 leaves each instance's second vCPU for the event loop. This relies on `argon2-cffi` releasing the Python GIL during the hash, which ST-04 checks (see Risks)."
    *   Location: ST-01 Containerfile, line 1138-1139
        *   Change: "starts uvicorn with `kairos.main:app`" → "starts one uvicorn process with `--factory kairos.main:create_app`".
    *   Location: ST-04 `passwords.py`, lines 1300-1302
        *   Change: "They run in a worker thread, at most 1 at a time per process, waiting up to 2 seconds for a slot, otherwise 503 `SIGN_IN_BUSY`. Before the cap is fixed, ST-04 confirms from the `argon2-cffi` documentation or source that hashing releases the GIL, and records the result in this document."
    *   Location: section 6 error table (A12)
        *   Change: add `SIGN_IN_BUSY`.

### [A11] [`egm-zoom-out`, `egm-inventive-catalyst`, `egm-design`, `egm-codebase-details`] - Severity: Major
*   **Affected Scope:** [Lines 270-289](../docs/designs/kairos.md#L270-L289) (request pipeline), [Lines 1007-1008](../docs/designs/kairos.md#L1007-L1008), [Lines 1218-1227](../docs/designs/kairos.md#L1218-L1227) (`db.py`)
*   **Discrepancy Description:** Adding the request ID to every statement as a SQL comment through an event hook has two costs:
    * It defeats the prepared-statement cache on the paths that carry the p99 target.
    * It contradicts "no SQL is built from strings".
*   **Rationale:** Set the request ID once per transaction with `set_config`, and log it with `%a`.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 273-275 and 1226-1227:
    * A per-request comment makes each statement's text unique, so the asyncpg statement cache misses every time.
    * The comment contains request data, which conflicts with the rule at lines 1007-1008.

    `application_name` is limited to 63 bytes, and `kairos:` plus a UUID is 43.
*   ***Proposed Edit:***
    *   Location: pipeline step 1, lines 271-275
        *   Change: replace "The repository layer adds it as a SQL comment (`/* request_id=... */`), so PostgreSQL logs can be matched to the request (NFR-OPS-3)." with "Every database transaction, including read-only ones, starts with one statement that sets `application_name` to `kairos:<request id>` (step 5). PostgreSQL's `log_line_prefix` includes `%a`, so database log lines can be matched to the request (NFR-OPS-3). The SQL text never changes, so prepared statements are reused."
    *   Location: pipeline step 5, lines 286-289
        *   Change: "*Actor and request ID:* each transaction starts with `SELECT set_config('kairos.actor_id', :actor, true), set_config('application_name', :app_name, true)` with bound parameters. Read-only transactions pass an empty actor. Setting the actor happens from Slice 1, so the audit triggers added in Slice 4 have an actor from the first day."
    *   Location: ST-01 `compose.yaml` `db` service
        *   Change: add "started with `-c log_line_prefix='%m [%p] %a '`".
    *   Location: ST-02 `db.py`, lines 1224-1227
        *   Change: replace the `transaction` and event-hook bullets with "`transaction(actor_id)` begins a transaction and runs the statement in section 2, step 5. `get_session()` runs the same statement with an empty actor for read-only requests." Delete the event-hook bullet.

### [A12] [`egm-structural`, `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Major
*   **Affected Scope:** [Line 301](../docs/designs/kairos.md#L301), [Lines 649-658](../docs/designs/kairos.md#L649-L658) (section 6 common), [Lines 1198-1217](../docs/designs/kairos.md#L1198-L1217) (ST-02 `errors.py`, `middleware.py`)
*   **Discrepancy Description:**
    * Line 301 refers to an error-code list that does not exist.
    * Codes are scattered and mixed with codes from later slices.
    * Message text is missing for most codes.
    * The `details` formats are undefined.
    * Starlette's own 404 and 405 responses do not use the error shape.
    * A handler for `Exception` runs outside the app's middleware, so the 500 response has no `X-Request-ID`.
*   **Rationale:** Add one error table, framework-error handlers and a pure ASGI middleware.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed that no list exists. The Starlette behaviour is accurate: unknown routes raise Starlette's `HTTPException`, and the `Exception` handler runs in `ServerErrorMiddleware`, outside the middleware the app adds. The message text follows the tone in design-theme.md ("Say what went wrong and what to do next").
*   ***Proposed Edit:***
    *   Location: section 6, after "Common to all endpoints" (after line 658)
        *   Change: insert:
            ```markdown
            **Error codes (Slice 1)**

            | Code | Status | Message | `details` |
            |---|---|---|---|
            | `VALIDATION_FAILED` | 422 | "Some fields need correcting. Fix the highlighted fields and try again." | `fields: [{ field, problem }]`. `field` is the camelCase body field, query parameter or header name (for example `durationMinutes`, `date`, `If-Match`). `problem` is a sentence for that field, for example "Enter at least 15 minutes, in steps of 15 minutes (for example 7:45)." |
            | `INVALID_CREDENTIALS` | 401 | "Email or password is incorrect. Check both and try again." | none |
            | `SESSION_ENDED` | 401 | "Your session has ended. Sign in again." | none |
            | `CSRF_FAILED` | 403 | "This request was blocked. Reload the page and try again." | none |
            | `PROJECT_NOT_AVAILABLE` | 403 | "You cannot log time to this project on this date. Pick a project from the list." | `{ projectId, date }` |
            | `ENTRY_NOT_FOUND` | 404 | "This entry no longer exists. Reload the week to see your entries." | none |
            | `NOT_FOUND` | 404 | "This address does not exist. Go back to your week." | none |
            | `METHOD_NOT_ALLOWED` | 405 | "This action is not supported here. Reload the page and try again." | none |
            | `ENTRY_CHANGED` | 409 | "This entry was changed in another tab. Your edit was not saved. Check the latest version and edit again." | `{ entry: Entry }` |
            | `DAILY_LIMIT_EXCEEDED` | 409 | "This day would have more than 24 hours. Reduce the duration or edit another entry." | `{ date, existingMinutes, holidayMinutes, requestedMinutes, limitMinutes }` |
            | `VERSION_REQUIRED` | 428 | "This change is missing the entry version. Reload the page and try again." | none |
            | `INTERNAL_ERROR` | 500 | "Something went wrong on our side. Try again. If it keeps happening, report request ID <id>." | `{ requestId }` |
            | `NOT_READY` | 503 | "The service is starting or cannot reach the database. Try again shortly." | none |
            | `SIGN_IN_BUSY` | 503 | "Many people are signing in right now. Try again in a few seconds." | none |

            Later slices add `ACCOUNT_LOCKED` (423, Slice 2), `DATE_NOT_EDITABLE` (403, Slice 3) and `EXPORT_BUSY` (429, Stretch).
            ```
    *   Location: section 6 "Common to all endpoints"
        *   Change: shorten the three message bullets to point at the table.
    *   Location: ST-02 `errors.py`, lines 1200-1205
        *   Change: replace with bullets:
            * "`AppError(status, code, message, details)` and one subclass per code in the section 6 error table."
            * "A handler for `AppError` writes `{ code, message, details }`."
            * "A handler for `RequestValidationError` writes `VALIDATION_FAILED`, turning each Pydantic error location into the camelCase field name."
            * "A handler for Starlette's `HTTPException` maps 404 to `NOT_FOUND` and 405 to `METHOD_NOT_ALLOWED`."
    *   Location: ST-02 `middleware.py`, lines 1212-1217
        *   Change: add "It is a pure ASGI middleware, not `BaseHTTPMiddleware`, and it is the outermost layer. It catches any unhandled exception, logs the stack trace, and builds the `INTERNAL_ERROR` response itself, with `X-Request-ID` and `details.requestId`."

### [A13] [`egm-zoom-out`, `egm-style`, `egm-codebase-details`] - Severity: Major
*   **Affected Scope:** [Lines 93-98](../docs/designs/kairos.md#L93-L98) (design goal), [Lines 565-597](../docs/designs/kairos.md#L565-L597) (`tasks`, `time_entries`), [Lines 669-711](../docs/designs/kairos.md#L669-L711) (endpoints and shapes), [Lines 1263-1276](../docs/designs/kairos.md#L1263-L1276) (ST-03 migration)
*   **Discrepancy Description:** Slice 1 carries items for later slices:
    * LEAVE and `leave_type`
    * `tasks.is_active` and `TASK_INACTIVE`
    * `flags`, `isFuture`, `editable` and `mustChangePassword`

    The reason given is weak, because the document says adding a value is a one-line migration (line 504). ST-03 says "exactly as in the data dictionary", but the dictionary marks some items for later slices.
*   **Rationale:** Remove these items or keep each with a Slice 1 reason. Restate the goal as "no destructive change". List exactly what migration 0001 creates.
*   ***AI Suggestion:*** PROPOSE FIX (**changes your design goal; check before accepting**)
*   ***Justification:*** Confirmed:
    * Line 504 says "adding a value is a one-line migration".
    * Lines 571, 592 and 594 mark items as stretch or Slice 3.
    * Line 710 says `flags` is always empty and `editable` always true in Slice 1.

    None of these items is read or written by a Slice 1 requirement. Adding a column, a CHECK value or a response field later does not break Slice 1 data or clients. That still meets the purpose of the design goal (no rewrite of Slice 1). The version number, the holiday-aware check and the actor stay in Slice 1, because adding those later *would* change behaviour.
*   ***Proposed Edit:***
    *   Location: design goal, lines 93-98
        *   Change: "Choose Slice 1's tables and API shapes so that Slices 2 to 5 only add tables, columns, `CHECK` values and response fields. They never make a destructive change to Slice 1's tables or a breaking change to its API. In particular, the 24-hour check must already allow for holiday hours (Slice 3), entries must already carry a version number for lost-update protection (TIME-17), and writes must already pass the actor the audit trail needs (Slice 4)."
    *   Location: `tasks` table, line 571
        *   Change: delete the `is_active` row. Add below the table: "Stretch adds `is_active` (PRJ-13)."
    *   Location: `time_entries` table, lines 588-597
        *   Change: `kind`: "`WORK` in Slice 1. Slice 3 widens the `CHECK` to add `LEAVE`. Holidays are never stored here (Q1)." Delete the `leave_type` row. Remove "Leave entries also require 240 or 480 (TIME-4)." from `duration_minutes`. Add below the table: "Slice 3 adds `leave_type` (`CASUAL`, `SICK`, `EARNED`, `UNPAID`) and the 240-or-480 rule for leave (TIME-4)."
    *   Location: endpoints table, lines 669-670
        *   Change: `GET /tasks` returns "all tasks". `POST /me/time-entries` failures: remove `TASK_INACTIVE`.
    *   Location: shapes, lines 679-697
        *   Change: remove `mustChangePassword` from `Me`, `isFuture` and `flags` from `Day`, `leaveType` and `editable` from `Entry`, and `editable` from `HolidayEntry`. `Entry.kind` is `"WORK"`.
        *   Change: replace the bullet at lines 706-711 with "Later slices add response fields without changing existing ones: `Me.mustChangePassword` (Slice 2); `Day.flags` (`BELOW_EXPECTED`, `OVER_DAILY_LIMIT`), `Entry.editable`, `Entry.leaveType` and kind `LEAVE` (Slice 3). In Slice 1 the UI offers edit and delete on every `WORK` entry and on no `HOLIDAY` entry."
    *   Location: ST-03 migration, lines 1266-1270
        *   Change: replace "Columns, `CHECK`s and indexes are exactly as in the data dictionary." with "Creates the `btree_gist` extension (trusted since PostgreSQL 13, so the database owner can create it). Creates exactly the Slice 1 columns, `CHECK`s and constraints in the data dictionary, and these indexes: `time_entries (employee_id, entry_date)`, `assignments (employee_id, start_date)`, `sessions (employee_id)`. Not created: the `(entry_date, project_id)` index, `period_locks`, `audit_log`, `leave_type`, `tasks.is_active` and the Slice 2 credential columns."
    *   Location: ST-06 tasks, line 1397
        *   Change: "returns all tasks sorted by name".
    *   Location: ST-07 `read_week` and lines 1469-1470
        *   Change: remove `isFuture`. "Slice 1 does not apply TIME-8 or TIME-9. Slice 3 adds them and the `editable` field."
    *   Location: ST-09 `EntryCard`, lines 1594-1595
        *   Change: "Edit and delete actions are shown on `WORK` entries. Holiday entries have none."

### [A14] [`egm-comprehension`, `egm-structural`, `egm-style`, `egm-design`] - Severity: Major
*   **Affected Scope:** [Lines 55-72](../docs/designs/kairos.md#L55-L72) (The Problem), [Lines 83-85](../docs/designs/kairos.md#L83-L85) (G-4), [Lines 350-480](../docs/designs/kairos.md#L350-L480) (Q1-Q8), [Lines 839-995](../docs/designs/kairos.md#L839-L995) (Alternatives)
*   **Discrepancy Description:** Parts of the document can only be checked with the PRD open:
    * The eight PRD section 10 questions are never stated.
    * TIME-8, TIME-9, HOL-2 and RPT-6 drive decisions but are not described.
    * "Friday peak" and the database size are undefined.
    * Estimates read as measurements.
    * Rejections say "Rejected by author".
*   **Rationale:** Add a one-line restatement of each question, one clause per rule, one Friday-peak sentence and one "estimates" sentence, and remove the "Rejected by author:" prefix. Link to `prd.md` rather than adding a glossary.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed: a reader without the PRD cannot check the month-lock resolution (lines 69-72) or the Q8-B argument (lines 955-956). The PRD supplies all the facts: section 10, TIME-8, TIME-9, HOL-2, RPT-6, TIME-10, NFR-PERF-1, NFR-PERF-7 and the "uneven load" paragraph in 7.2. Not fixed here: `decision_options.md` lives in the EGM artifact folder `.egm/`, which git does not track. If you want the decision record to last, move it next to the design (for example `docs/designs/kairos-decision-options.md`) and update the 4 links. That touches a file outside the target, so it is not part of the edit unless you ask.
*   ***Proposed Edit:***
    *   Location: The Problem, before line 55
        *   Change: insert:
            > "Four PRD rules shape the decisions below:
            > - TIME-8: no work entries for a future date; leave up to 60 days ahead.
            > - TIME-9: an employee cannot add, edit or delete an entry dated more than 14 days ago.
            > - HOL-2: HR can edit or delete a holiday unless its date is in a locked month.
            > - RPT-6 (Stretch): HR can lock and unlock a calendar month.
            >
            > TIME-10 (Stretch) adds that only HR, by unlocking, can change entries in a locked month."
    *   Location: line 56
        *   Change: "(the author accepted these as defaults during brainstorming)" → "(the author accepted these defaults on 2026-11-19)".
    *   Location: line 71
        *   Change: "section 3.3" → "PRD section 3.3".
    *   Location: after G-4 (line 85)
        *   Change: add "The **Friday peak** in this document is the NFR-PERF-1 load at its busiest: 500 concurrent users sending 50 to 100 requests a second. PRD section 7.2 says most employees report on Friday afternoon and Monday morning, and again on the last working day of the month. The **reference environment** (NFR-PERF-7) is 2 application instances with 2 vCPU and 4 GB each, and 1 database instance with 4 vCPU and 16 GB."
    *   Location: section 4 intro, after line 354
        *   Change: add "Timings, sizes and rates in this section are estimates from `decision_options.md`, not measurements. The Slice 5 load test confirms them."
    *   Location: under each Q heading
        *   Change: add one italic line restating the PRD question:
            * Q1 "*PRD question 1: stored per employee or derived on read, and what each costs?*"
            * Q2 "*PRD question 2: how is the session carried, and how is revocation within 1 minute (AUTH-6) met with a stateless tier?*"
            * Q3 "*PRD question 3: how is the 24-hour limit enforced for concurrent saves (NFR-DATA-1)?*"
            * Q4 "*PRD question 4: which indexes keep a week read and the missing-time report fast at 20 million rows, and is partitioning or pre-aggregation needed?*"
            * Q5 "*PRD question 5: how do CSV exports run without slowing interactive requests (NFR-PERF-4)?*"
            * Q6 "*PRD question 6: how is lost-update protection (TIME-17) built and shown?*"
            * Q7 "*PRD question 7: is a cache needed for NFR-PERF-2, and how is it invalidated?*"
            * Q8 "*PRD question 8: how is the audit trail written so a new code path cannot skip it?*"
    *   Location: Alternatives, every "**Reason for Rejection:** Rejected by author: " and "Rejected by author: " in the infrastructure list
        *   Change: remove the prefix and keep the reason. The section intro (lines 841-843) already records who decided and when.
    *   Location: Alternatives infrastructure heading, line 965
        *   Change: "rejected during brainstorming" → "rejected".

### [A15] [`egm-comprehension`, `egm-design`, `egm-structural`] - Severity: Major
*   **Affected Scope:** [Lines 133-135](../docs/designs/kairos.md#L133-L135), [Lines 137-152](../docs/designs/kairos.md#L137-L152), [Lines 975-978](../docs/designs/kairos.md#L975-L978), [Lines 1020-1025](../docs/designs/kairos.md#L1020-L1025)
*   **Discrepancy Description:** "Docker is not permitted" cites no source. Whether the rule covers Docker Hub is unknown, yet every default image is `docker.io/...`, and the question is not in Open Questions.
*   **Rationale:** Link the source and add an Open Question with an owner.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** The source exists: PRD section 7.1, Packaging row, "Docker is not permitted for compliance reasons". The document just does not cite it. The Docker Hub question is real and unanswered in any file. If you already know the answer, tell me and I will write it in instead of the open question.
*   ***Proposed Edit:***
    *   Location: Non-Goals, line 133
        *   Change: "Docker is not permitted." → "Docker is not permitted for compliance reasons (PRD section 7.1, Packaging)."
    *   Location: line 976-977
        *   Change: "the compliance rule forbids Docker" → "PRD section 7.1 forbids Docker".
    *   Location: Open Questions
        *   Change: add "**Does the Docker rule cover Docker Hub images?** PRD section 7.1 forbids Docker but does not say whether images from Docker Hub (`docker.io`) may be used. Every default image in this design comes from there. If they may not, the image variables must point at an approved registry before `make up` can work. Owner: the author, to confirm with compliance before ST-01."

### [A16] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`, `egm-comprehension`] - Severity: Major
*   **Affected Scope:** [Lines 1101-1125](../docs/designs/kairos.md#L1101-L1125) (ST-01 compose, `.env.example`, `01-roles.sh`), [Lines 1140-1146](../docs/designs/kairos.md#L1140-L1146), [Lines 1233-1246](../docs/designs/kairos.md#L1233-L1246), [Lines 1687-1689](../docs/designs/kairos.md#L1687-L1689)
*   **Discrepancy Description:**
    * Database roles and URLs have no named variables, and `POSTGRES_PASSWORD` is missing.
    * The values of `KAIROS_ENV` are not given.
    * The test database does not mount `db/init`, so it has no roles and no `btree_gist`.
    * The roles used by tests are not named.
*   **Rationale:** List every variable and which file reads it. Mount `db/init` in the test database. Name the test roles.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 1112-1114: "the owner and app database users and passwords" with no names. The official postgres image does not start without `POSTGRES_PASSWORD`.
*   ***Proposed Edit:***
    *   Location: ST-01 `.env.example`, lines 1111-1115
        *   Change: replace with "Every variable below, with its local default. `make up` copies the file to `.env` if `.env` is missing (NFR-SEC-8, NFR-OPS-1)." followed by:
            ```markdown
            | Variable | Local default | Read by | Required |
            |---|---|---|---|
            | `POSTGRES_PASSWORD` | `kairos-local-postgres` | `db` container (superuser) | yes |
            | `KAIROS_OWNER_PASSWORD` | `kairos-local-owner` | `db/init/01-roles.sh` | yes |
            | `KAIROS_APP_PASSWORD` | `kairos-local-app` | `db/init/01-roles.sh` | yes |
            | `DATABASE_URL` | `postgresql+asyncpg://kairos_app:kairos-local-app@db:5432/kairos` | `config.Settings` (API) | yes, for the API |
            | `KAIROS_OWNER_DATABASE_URL` | `postgresql+asyncpg://kairos_owner:kairos-local-owner@db:5432/kairos` | `migrations/env.py`, `seed.py` | yes, for migrate and seed |
            | `KAIROS_TEST_DATABASE_URL` | `postgresql+asyncpg://kairos_app:kairos-local-app@localhost:55433/kairos` | `tests/conftest.py` (app under test) | for tests |
            | `KAIROS_TEST_OWNER_DATABASE_URL` | `postgresql+asyncpg://kairos_owner:kairos-local-owner@localhost:55433/kairos` | `tests/conftest.py` (migrations, fixtures, table clearing) | for tests |
            | `KAIROS_E2E_DATABASE_URL` | `postgresql://kairos_owner:kairos-local-owner@localhost:55432/kairos` | `frontend/e2e/db.ts` | for e2e |
            | `KAIROS_ENV` | `local` | `config.Settings`, `seed.py` | yes: `local`, `test` or `production` |
            | `KAIROS_SEED_PASSWORD` | `Kairos2026x` | `seed.py` | no |
            | `KAIROS_LOG_LEVEL` | `INFO` | `config.Settings` | no |
            | `KAIROS_POSTGRES_IMAGE` | `docker.io/library/postgres:16` | `compose.yaml`, `Makefile` | no |
            | `KAIROS_PYTHON_IMAGE` | `docker.io/library/python:3.12-slim` | `backend/Containerfile` | no |
            | `KAIROS_NODE_IMAGE` | `docker.io/library/node:22-slim` | `proxy/Containerfile` | no |
            | `KAIROS_CADDY_IMAGE` | `docker.io/library/caddy:2` | `proxy/Containerfile` | no |
            ```
    *   Location: ST-01 `config.py` (with A3)
        *   Change: "`config.Settings` has the fields `env`, `database_url`, `owner_database_url`, `seed_password` and `log_level`. Each entry point checks the fields it needs and refuses to start if one is missing."
    *   Location: ST-01 `compose.yaml`
        *   Change: "`db` gets `POSTGRES_PASSWORD`, `KAIROS_OWNER_PASSWORD` and `KAIROS_APP_PASSWORD` and mounts `db/init` read-only."
    *   Location: ST-01 `01-roles.sh`, lines 1122-1125
        *   Change: drop "and enables `btree_gist`" (the migration creates it, A13).
    *   Location: ST-01 `test-backend` (with A2)
        *   Change: "starts `podman run --rm -d` with `KAIROS_POSTGRES_IMAGE`, tmpfs storage, host port 55433, the three password variables, and `db/init` mounted as the init directory. Waits as in A17, runs `uv run pytest` in `backend/`, then removes the container."
    *   Location: ST-02 `conftest.py`
        *   Change: "The app under test connects as `kairos_app` (`KAIROS_TEST_DATABASE_URL`). Migrations, factories and table clearing (`TRUNCATE ... RESTART IDENTITY CASCADE`) connect as `kairos_owner` (`KAIROS_TEST_OWNER_DATABASE_URL`)."

### [A17] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Major
*   **Affected Scope:** [Lines 1085-1100](../docs/designs/kairos.md#L1085-L1100) (Makefile), [Lines 1180-1188](../docs/designs/kairos.md#L1180-L1188) (README), [Lines 249-255](../docs/designs/kairos.md#L249-L255) (start-up)
*   **Discrepancy Description:**
    * The plan does not say whether linters and tests run on the host or in containers, and the README lists only Podman, podman-compose and make.
    * `make test` needs the running stack for e2e and does not say whether it starts it.
    * `pg_isready` over the Unix socket can pass while the init scripts are still running.
*   **Rationale:** Decide host or container for each target and give its command. Use a TCP `pg_isready` in a retry loop.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 1099 and 1183-1184. The official postgres image runs its init scripts on a temporary server that listens only on the Unix socket, so a socket check passes early. **Choice made in the edit:** linters and test runners run on the host. The PRD forbids Docker only, and running Playwright, Vitest and pytest in containers would need extra images and networking. If you prefer everything in containers, say so and I will change this item.
*   ***Proposed Edit:***
    *   Location: Makefile, line 1099
        *   Change: "Every target calls `podman` or `podman-compose`, never `docker`." → "No target calls `docker`. Services and the test database run with `podman` or `podman-compose`. Linters and test runners run on the host: `uv run ruff ...`, `uv run mypy --strict kairos` and `uv run pytest` in `backend/`, and `npm --prefix frontend run lint|typecheck|test|e2e`."
    *   Location: Makefile `up`, and section 1 start-up (line 250)
        *   Change: "waits until `pg_isready -h 127.0.0.1 -U kairos_owner -d kairos`, run with `podman exec`, succeeds over TCP, retrying every second for up to 60 seconds, then fails with a message".
    *   Location: Makefile `test-e2e` (added in ST-10)
        *   Change: "first checks `https://localhost:8443/api/v1/health/ready`. If the stack is not running, it stops with 'Run make up first.' `make test` does not start the stack."
    *   Location: README prerequisites, lines 1183-1184
        *   Change: "Podman 4.9 or later, `podman-compose` 1.1 or later, `make`, Python 3.12 with `uv`, Node.js 22 LTS with npm, and the Playwright browsers (`npx playwright install chromium webkit`, plus `--with-deps` on Linux)."

### [A18] [`egm-codebase-details`, `egm-implementation-readiness`] - Severity: Major
*   **Affected Scope:** [Lines 1131-1134](../docs/designs/kairos.md#L1131-L1134), [Lines 1218-1223](../docs/designs/kairos.md#L1218-L1223), [Lines 1233-1246](../docs/designs/kairos.md#L1233-L1246)
*   **Discrepancy Description:**
    * asyncpg pooled connections break across pytest-asyncio's per-test event loops.
    * Async Alembic's `asyncio.run()` fails inside an async fixture.
    * httpx does not send `Secure` cookies to an `http://` base URL.
*   **Rationale:** Specify the pytest-asyncio settings or `NullPool`, run migrations from a synchronous fixture or a subprocess, and use `base_url="https://test"`.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** All three are accurate library behaviours, and the plan is silent on each. `NullPool` in tests avoids depending on pytest-asyncio's loop-scope options, which have changed between versions.
*   ***Proposed Edit:***
    *   Location: ST-01 `pyproject.toml` pytest settings, line 1133
        *   Change: add "`asyncio_mode = "auto"`".
    *   Location: ST-02 `db.py`
        *   Change: "The engine is created on first use from `Settings`. When `KAIROS_ENV=test`, it uses `NullPool`, so no connection outlives the event loop of the test that opened it, and each concurrent request gets its own connection."
    *   Location: ST-02 `conftest.py`
        *   Change: "Runs `uv run alembic upgrade head` once per test session as a subprocess, from a synchronous session-scoped fixture." "An httpx `AsyncClient` with `ASGITransport(app=...)` and `base_url="https://test"`, so the `Secure` session cookie is sent."

### [A19] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Major
*   **Affected Scope:** [Lines 1509-1516](../docs/designs/kairos.md#L1509-L1516) (concurrency test), [Lines 1322-1334](../docs/designs/kairos.md#L1322-L1334), [Lines 1393-1410](../docs/designs/kairos.md#L1393-L1410), [Lines 1480-1490](../docs/designs/kairos.md#L1480-L1490), [Lines 1649-1680](../docs/designs/kairos.md#L1649-L1680), [Lines 1695-1697](../docs/designs/kairos.md#L1695-L1697)
*   **Discrepancy Description:**
    * The concurrency test can pass with locking broken, and its second case cannot fail.
    * `GET /tasks`, the week failure paths and the sign-out failure paths have no tests.
    * No test checks the 8-hour expiry value.
    * CUJ 5 has no e2e test.
    * NFRs delivered in Slice 1 are missing from the traceability table.
    * "Every endpoint" overstates the coverage.
*   **Rationale:** Strengthen the assertions, add the missing tests and rows, and narrow the claim.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed: 16h + 4h + 4h = 24h exactly, which is allowed, so case 2 at lines 1514-1516 passes with or without locks. Lines 1663-1680 have no NFR rows. Line 1654 skips CUJ 5.
*   ***Proposed Edit:***
    *   Location: concurrency test, lines 1511-1516
        *   Change: replace with "Two signed-in clients for the same employee send requests at the same moment (`asyncio.gather`; each request uses its own database connection). **Case 1:** the day has 16 hours, and both clients POST 5 hours. Exactly one gets 201 and the other gets 409 `DAILY_LIMIT_EXCEEDED`, never 500, and the day's entries sum to 21 hours. **Case 2:** the target day has 17 hours, and the employee has a 4-hour entry on each of two other days. Both clients PUT one of those entries onto the target day. Exactly one gets 200 and the other 409 `DAILY_LIMIT_EXCEEDED`, with no deadlock error, and the target day sums to 21 hours. Each case runs 20 times with fresh data."
    *   Location: ST-06
        *   Change: add `backend/tests/api/test_tasks.py` (Create): returns all tasks sorted by name with an `ETag`, and 401 without a session.
    *   Location: `test_week.py`
        *   Change: add "401 without a session; 422 on field `date` for `date=2026-13-01`."
    *   Location: `test_auth.py`
        *   Change: add:
            * "Sign-out without a session gives 401, and without the CSRF header gives 403."
            * "`expires_at` equals the sign-in time plus 8 hours (to within 1 second), and is unchanged after a later authenticated request (AUTH-5)."
            * "A log line from a sign-in request contains neither the password nor the cookie" (moved from ST-02).
    *   Location: `test_time_entries.py`
        *   Change: add "An unknown `taskId` gives 422 on `taskId`."
    *   Location: ST-10 e2e specs (with A20)
        *   Change: rename to `frontend/e2e/tests/cuj01-log-normal-day.spec.ts`, `cuj02-daily-limit.spec.ts`, `cuj03-unassigned-project.spec.ts`, `cuj04-common-projects.spec.ts`, `cuj06-sign-in-out.spec.ts` and `keyboard-entry.spec.ts`. Add `cuj05-edit-delete-navigate.spec.ts`. Test names use "CUJ-n (S-x)". CUJ 6 simulates the 8-hour expiry by setting `expires_at` in the past through the SQL helper.
    *   Location: traceability table, lines 1663-1680
        *   Change: update the e2e file names.
        *   Change: add `cuj05-edit-delete-navigate.spec.ts` to TIME-2, TIME-7 and TIME-14.
        *   Change: add rows:
            * NFR-REL-3 (`health/router.py` | `test_health.py`)
            * NFR-OPS-3 (`middleware.py`, `logging.py`, `db.py` | `test_errors_and_logging.py`)
            * NFR-SEC-1 (`security/passwords.py`, `logging.py` | `test_auth.py`, `test_errors_and_logging.py`)
            * NFR-SEC-3 (`auth/dependencies.py`, `time_entries/repository.py` | `test_time_entries.py`)
            * NFR-DATA-1 (`lock_day`, the assignments exclusion constraint | `test_daily_limit_concurrency.py`, `test_schema.py`)
            * NFR-UX-5 (`EntryFormModal.tsx` | `EntryFormModal.test.tsx`, `keyboard-entry.spec.ts`)
    *   Location: Verification Plan, lines 1696-1697
        *   Change: "Every endpoint except `GET /health/live`, which has no failure response, has at least one success test and one failure test."

### [A20] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Major
*   **Affected Scope:** [Lines 1157-1167](../docs/designs/kairos.md#L1157-L1167), [Lines 1638-1661](../docs/designs/kairos.md#L1638-L1661) (ST-10)
*   **Discrepancy Description:**
    * `@playwright/test` is in `frontend/package.json`, but the specs are in root `e2e/`, so Node cannot resolve the package from there.
    * The SQL helper has no Postgres client or connection string.
    * Fixture employees need an argon2id hash.
    * Fixture data has no uniqueness or cleanup rule.
    * `e2e/` is not in the repository layout in `engineering-standards.md`.
*   **Rationale:** Add a package and tsconfig for e2e, or move the suite into `frontend/`. Commit a precomputed hash. Use run-ID suffixes and teardown.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed: `engineering-standards.md` lists only `backend/`, `frontend/` and `docs/`. Moving the suite to `frontend/e2e/` fixes package resolution and the layout at once, with no extra `package.json`.
*   ***Proposed Edit:***
    *   Location: ST-10 items 1-2
        *   Change: move all paths from `e2e/` to `frontend/e2e/`. Add `frontend/e2e/db.ts` (Create): the test-only SQL helper, using `pg` with `KAIROS_E2E_DATABASE_URL` (A16).
    *   Location: ST-10 fixtures bullet
        *   Change: add:
            * "Fixture employees use one committed argon2id hash of a fixed test password. The hash is generated once with `argon2-cffi` and the command is noted in `fixtures.ts`."
            * "Every email and code gets a per-run suffix, and teardown deletes the rows the run created."
    *   Location: ST-01 frontend dev dependencies
        *   Change: add `pg` and `@types/pg` (see A23). `tsconfig.json` and ESLint cover `e2e/`. Vitest's `include` is limited to `src/**/*.test.{ts,tsx}`.

### [A21] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`] - Severity: Major
*   **Affected Scope:** [Lines 1338-1379](../docs/designs/kairos.md#L1338-L1379) (ST-05), [Lines 1303-1313](../docs/designs/kairos.md#L1303-L1313) (sign-in schema)
*   **Discrepancy Description:**
    * The seed values are not given.
    * Assignment dates are relative to the run date and the assignment key is undefined, so a run on a later day breaks on the exclusion constraint.
    * "Up to today" is ambiguous, and there is no way to inject the run date.
    * `@kairos.local` fails Pydantic `EmailStr`.
*   **Rationale:** Add a literal seed table with relative dates, define natural keys and the date range, add `run(session, today)`, and fix the email field or domain.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed at lines 1343-1355: the assignment key is "by its natural key", which is never defined. The PRD (section 8) says the HR administrator is one of the 6 people, and section 3.3 says HR administrators log their own time, so the seed includes entries for the HR administrator. The names below are new and you can change them.
*   ***Proposed Edit:***
    *   Location: ST-05 `seed.py`, lines 1341-1365
        *   Change: replace the bullets with:
            ```markdown
            Refuses to run when `KAIROS_ENV` is not `local`. Connects as `kairos_owner` and sets the actor to `system`. Entry point `run(session, today: date)`. The command line passes today's date in `Asia/Kolkata`, and tests pass a fixed date. Every date below is relative to `today` (R).

            *   **Employees** (date of joining R - 400 days, password `KAIROS_SEED_PASSWORD`, default `Kairos2026x`, which meets AUTH-3):

                | Code | Name | Email | Role | Department | Designation |
                |---|---|---|---|---|---|
                | K0001 | Arjun Mehta | arjun@kairos.example.com | HR_ADMIN | Delivery | HR Operations Lead |
                | K0002 | Priya Sharma | priya@kairos.example.com | EMPLOYEE | Engineering | Software Engineer |
                | K0003 | Rahul Verma | rahul@kairos.example.com | EMPLOYEE | Engineering | Senior Software Engineer |
                | K0004 | Meera Iyer | meera@kairos.example.com | EMPLOYEE | Engineering | QA Engineer |
                | K0005 | Vikram Nair | vikram@kairos.example.com | EMPLOYEE | Delivery | Project Manager |
                | K0006 | Ananya Das | ananya@kairos.example.com | EMPLOYEE | Delivery | Business Analyst |

            *   **Projects** (start R - 3650 days, no end date, `ACTIVE`): common Internal, Training, Recruitment and Pre-sales (PRJ-4), and client "Acme Health – Dev Pod" (Acme Health), "Borealis Data Platform" (Borealis Energy) and "Cobalt Mobile App" (Cobalt Retail).
            *   **Assignments** (start R - 400 days, no end date unless stated): Priya to Acme; Rahul to Acme and Borealis; Meera to Cobalt; Vikram to Borealis; Ananya to Acme, and to Cobalt ending R - 10 days. Arjun has none.
            *   **Tasks:** the 12 tasks from PRD section 8. No holidays in Slice 1.
            *   **Entries:** for each employee, on each Monday-to-Friday date from the Monday of the previous week up to and including R, but only on dates where the employee has no entries:
                *   Two work entries that total 8 hours. The first is a random multiple of 15 minutes from 4 to 6 hours.
                *   Each entry uses a project available to that employee on that date (the PRJ-4/PRJ-9 rule), a random task and a random work location.
                *   Randomness comes from `random.Random` seeded with the employee code and the date.
                *   Exception: Vikram has no entries on the Friday of the previous week, so that week is 32 hours (incomplete).
            *   **Natural keys for reruns:** employees by code, projects by name, tasks by name, and assignments by (employee code, project name), where any existing assignment for the pair means skip. Credentials and preferences are keyed by employee.
            ```
    *   Location: ST-05 tests, lines 1369-1374
        *   Change: add:
            * "`run` with R = a Wednesday, then again with R + 1 day, succeeds without an exclusion error."
            * "With R = a Monday, the current week has entries on that Monday."
            * "Every seeded entry's project is available to its employee on its date."
    *   Location: ST-04 `auth/schemas.py`
        *   Change: add "The sign-in `email` is a `str` of 3 to 254 characters, lowercased. It is not `EmailStr`, so the email-validator package is not needed."
    *   Location: README seed table, ST-05 item 3
        *   Change: "taken from the table above".

### [A22] [`egm-implementation-readiness`] - Severity: Major
*   **Affected Scope:** [Lines 682-697](../docs/designs/kairos.md#L682-L697) (shapes), [Lines 1441-1445](../docs/designs/kairos.md#L1441-L1445) (`read_week`)
*   **Discrepancy Description:**
    * The format of `Day.weekday` is undefined.
    * The `complete` rule is not stated.
    * It is unclear whether totals include holiday minutes, and whether `GET /me/week` sets `ETag`.
*   **Rationale:** Give the type and exact rule for each derived field.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed. The PRD glossary defines a complete week as one where "total reported hours (work + leave + holidays) are at least the expected hours", at week level.
*   ***Proposed Edit:***
    *   Location: after the shapes block (after line 704)
        *   Change: add:
            > "Derived fields:
            > - `weekday` is the ISO day number, 1 (Monday) to 7 (Sunday).
            > - `isoYear` and `isoWeek` follow ISO 8601. `weekStart` is the Monday and `weekEnd` the Sunday.
            > - `Day.expectedMinutes` comes from `employee_calendar`.
            > - `Day.totalMinutes` is the sum of the day's entry durations plus its holiday minutes.
            > - `Week.expectedMinutes` and `Week.totalMinutes` are the sums over the 7 days.
            > - `complete` is `Week.totalMinutes >= Week.expectedMinutes` (PRD glossary). Future weekdays count, so the current week stays incomplete until it is filled.
            > - Only `Entry` responses (POST 201, PUT 200) carry an `ETag`."

### [A23] [`egm-implementation-readiness`, `egm-codebase-details`, `egm-style`, `egm-zoom-out`, `egm-design`] - Severity: Major
*   **Affected Scope:** [Lines 1157-1179](../docs/designs/kairos.md#L1157-L1179) (ST-01 frontend, proxy), [Lines 993-995](../docs/designs/kairos.md#L993-L995) (shadcn/ui rejection)
*   **Discrepancy Description:**
    * Packages are missing: Emotion, icons, a date picker, fonts and test add-ons, and no React Router major version is given.
    * Node 20 is end-of-life.
    * The CSP has no `style-src` exception for Emotion and blocks inlined fonts. `frame-ancestors` is missing.
    * The Vite dev proxy targets an unpublished port.
*   **Rationale:** List every package with its major version, use Node 22 or 24, state the exact CSP and `assetsInlineLimit: 0`, add a styles check, and fix the proxy.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed:
    * MUI v6 requires `@emotion/react` and `@emotion/styled`.
    * MUI core has no date picker, so line 994's "MUI ships the ... date picker" is also wrong.
    * Node 20 reached end-of-life on 2026-04-30.
    * Emotion injects `<style>` elements, which `default-src 'self'` blocks.
    * `frame-ancestors` does not fall back to `default-src`.

    For the dev proxy, the edit points Vite at Caddy. Chromium and Firefox accept `Secure` cookies on `http://localhost`; Safari does not.
*   ***Proposed Edit:***
    *   Location: ST-01 item 11, lines 1161-1168
        *   Change: replace the dependency bullets with:
            > - "Dependencies: `react` and `react-dom` 18, `react-router-dom` 6, `@tanstack/react-query` 5, `@mui/material` and `@mui/icons-material` 6, `@emotion/react` and `@emotion/styled` 11, `@fontsource/inter` and `@fontsource/roboto-condensed` 5."
            > - "Development tools: `typescript` 5, `vite` 5 with `@vitejs/plugin-react`, `vitest` 2 with `jsdom`, `@testing-library/react`, `@testing-library/user-event` and `@testing-library/jest-dom`, `openapi-typescript` 7, `eslint` 9 with `typescript-eslint` 8 and `eslint-plugin-react-hooks` 5, `@playwright/test`, `pg` and `@types/pg`."
            > - "`vite.config.ts` sets `build.assetsInlineLimit: 0` and the Vitest settings: `jsdom`, setup file `src/test/setup.ts`, include `src/**/*.test.{ts,tsx}`."
            > - "The date jump uses a native date input inside an MUI `TextField`, so no date-picker package is needed."
            > - "During development, Vite forwards `/api` to `https://localhost:8443` (`secure: false`), the API behind Caddy. Use a Chromium-based browser or Firefox for `npm run dev`, because Safari refuses `Secure` cookies on `http://localhost`."
    *   Location: ST-01 item 12, lines 1171-1179
        *   Change: "Node 20 stage" → "Node 22 LTS stage (`KAIROS_NODE_IMAGE`)". The CSP line becomes "Sets `Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'`, and the `X-Content-Type-Options: nosniff` and `Referrer-Policy: same-origin` headers. `style-src` allows inline styles because MUI's styling library (Emotion) adds `<style>` elements at run time."
    *   Location: ST-10 `cuj01` spec
        *   Change: add "and checks that styles are applied: the top bar's background is `#111111`."
    *   Location: shadcn/ui rejection, lines 993-995
        *   Change: "MUI ships the autocomplete and modal the weekly view needs, and its `TextField` wraps the native date input. The colours in design-theme.md can be applied through MUI's theme."

### [A24] [`egm-implementation-readiness`, `egm-codebase-details`] - Severity: Major
*   **Affected Scope:** [Lines 1528-1533](../docs/designs/kairos.md#L1528-L1533) (`theme.ts`), [Lines 1592-1595](../docs/designs/kairos.md#L1592-L1595) (`EntryCard`)
*   **Discrepancy Description:** White on `#00A67E` is about 3.1:1, below the 4.5:1 that NFR-UX-3 requires. Card text colours are not specified.
*   **Rationale:** Set `contrastText` per palette entry and add a contrast check.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** I recomputed the WCAG ratios:
    * White on `#00A67E`: 3.1:1.
    * `#1A1A1A` (the theme's text colour) on `#00A67E`: 5.6:1.
    * White on `#B45309`: 5.0:1.
    * White on `#5B21B6`: 9.2:1.
    * White on `#B91C1C`: 6.5:1.

    A unit test is cheaper than adding an axe dependency.
*   ***Proposed Edit:***
    *   Location: ST-08 `theme.ts`, lines 1530-1533
        *   Change: add "Each palette entry sets `contrastText`: `#1A1A1A` on work (`#00A67E`), and `#FFFFFF` on leave, holiday and warning."
    *   Location: ST-08
        *   Change: add `frontend/src/theme.test.ts` (Create), which checks that every palette entry's `contrastText` has a WCAG contrast ratio of at least 4.5:1 against its main colour.
    *   Location: ST-09 `EntryCard`
        *   Change: "card text uses the palette entry's `contrastText`".

### [A25] [`egm-style`, `egm-implementation-readiness`, `egm-codebase-details`, `egm-comprehension`, `egm-design`] - Severity: Major
*   **Affected Scope:** [Lines 78-80](../docs/designs/kairos.md#L78-L80) (G-1), [Lines 339-345](../docs/designs/kairos.md#L339-L345) (keyboard path), [Lines 1600-1611](../docs/designs/kairos.md#L1600-L1611) (form), [Line 1633](../docs/designs/kairos.md#L1633), [Lines 1654-1656](../docs/designs/kairos.md#L1654-L1656)
*   **Discrepancy Description:**
    * The form has no Date field, though section 3 says the date defaults and the API allows a date change.
    * G-1 says "key presses" while the design counts "interactions".
    * The keyboard tests ignore the open counting question.
    * Initial focus and the default location for a new employee are not stated.
*   **Rationale:** Add the Date field or state that moves are API-only, reword G-1, state the focus and default, and gate the tests.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed:
    * Line 79 says "key presses", while NFR-UX-5 says "key interactions".
    * Line 339 says "Date defaults to the column it was opened from", but line 1601 lists no Date field.
    * `Me.lastWorkLocation` can be null (line 680).

    Placing Date first while focus starts on Project keeps the 4-step path.
*   ***Proposed Edit:***
    *   Location: G-1, lines 78-80
        *   Change: "in no more than 6 key presses" → "in no more than 6 key interactions (NFR-UX-5; see Open Questions for how they are counted)".
    *   Location: section 3 keyboard path, lines 339-341
        *   Change: "The form opens with Project focused. A Date field comes first in the form and defaults to the column the form was opened from; changing it moves the entry to another day (TIME-7). Location defaults to the one used last, or Office when there is none."
    *   Location: ST-09 form, lines 1601-1603 and 1611
        *   Change: "fields in the order Date, Project, Task, Duration, Work location, Activity. The form opens with focus on Project." Defaults: "Date is the column's date. Work location is `Me.lastWorkLocation`, or `OFFICE` when that is null (TIME-3)." `ProjectSelect` uses the form's current date.
    *   Location: ST-09 test, line 1633, and ST-10 line 1655
        *   Change: add "Written once the NFR-UX-5 counting rule in Open Questions is confirmed. ST-09 and ST-10 are not done until then."

### [A26] [`egm-style`, `egm-codebase-details`, `egm-implementation-readiness`, `egm-zoom-out`] - Severity: Major
*   **Affected Scope:** [Lines 323-328](../docs/designs/kairos.md#L323-L328) (section 3 client), [Lines 449-459](../docs/designs/kairos.md#L449-L459) (Q6), [Lines 1520-1542](../docs/designs/kairos.md#L1520-L1542) (ST-08), [Lines 1612-1615](../docs/designs/kairos.md#L1612-L1615), [Line 1634](../docs/designs/kairos.md#L1634)
*   **Discrepancy Description:**
    * "Show latest" and "Apply my changes again" have no defined behaviour.
    * Errors without `details.fields` have no place on screen.
    * Section 3 says `client.ts` redirects on a 401, but ST-08 says it only throws. "Sends a 401 to `/sign-in`" is garbled.
    * The two-action dialog is a Stretch feature (TIME-17).
*   **Rationale:** In Slice 1, show the 409 message and reload the entry, and defer the dialog. Place fieldless errors. Assign the redirect and test it.
*   ***AI Suggestion:*** PROPOSE FIX (**reduces the Slice 1 conflict dialog; check before accepting**)
*   ***Justification:*** Confirmed at lines 455-457, 1527, 1542 and 325-326. TIME-17 requires only that "the user is told the entry has changed". Reloading the entry meets that and removes the undefined "Apply my changes again".
*   ***Proposed Edit:***
    *   Location: Q6, lines 455-459
        *   Change: "The modal then shows the `ENTRY_CHANGED` message and replaces the form values with `details.entry`, so the user sees the latest version and can edit again. The version check ships in Slice 1, because adding a required header to PUT and DELETE later would break existing clients. This delivers the part of TIME-17 that the PRD requires early. A dialog that offers to re-apply the user's edit is Stretch."
    *   Location: section 3 API access, lines 325-326
        *   Change: "on a 401 `SESSION_ENDED` from any call except sign-in, sends the user to `/sign-in`".
    *   Location: ST-08 `App.tsx`, line 1527
        *   Change: "If `GET /me` returns 401, the route guard redirects to `/sign-in`."
    *   Location: ST-08 `client.ts`, line 1542
        *   Change: "Throws `ApiError { status, code, message, details }`. On a 401 `SESSION_ENDED` from any function except `signIn`, it first calls the redirect handler that `App.tsx` registers, which goes to `/sign-in`."
    *   Location: ST-08 `client.test.ts`
        *   Change: add "A 401 `SESSION_ENDED` from `getWeek` calls the redirect handler. A 401 `INVALID_CREDENTIALS` from `signIn` does not."
    *   Location: ST-09 form, lines 1612-1615
        *   Change: replace with:
            > - "**On 409 `ENTRY_CHANGED`:** shows the server message in `ErrorMessage` and replaces the form values with `details.entry`, including its new version."
            > - "**On other errors:** when `details.fields` names a form field, the problem appears under that field. Otherwise the server message appears in `ErrorMessage` at the top of the modal."
    *   Location: ST-09 test, line 1634
        *   Change: "The 409 message appears and the form shows the current entry."

### [A27] [`egm-style`] - Severity: Major
*   **Affected Scope:** [Lines 307-309](../docs/designs/kairos.md#L307-L309) (section 2 logs), [Lines 1209-1211](../docs/designs/kairos.md#L1209-L1211) (`logging.py`)
*   **Discrepancy Description:** The redaction rule leaves three things open: case sensitivity, exact or substring matching, and whether `Set-Cookie` is covered.
*   **Rationale:** State the rule exactly.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Confirmed. Section 2 lists `Authorization` and `Cookie` with capitals, while ST-02 lists lower-case keys.
*   ***Proposed Edit:***
    *   Location: section 2 logs, lines 307-309
        *   Change: "A filter replaces the value of any key named `password`, `token`, `cookie`, `set-cookie` or `authorization` with `[REDACTED]` before writing. Matching is exact and ignores case, at any depth, including request and response header maps (NFR-SEC-1)."
    *   Location: ST-02 `logging.py`, lines 1209-1211
        *   Change: "A redaction filter that applies the rule in Technical Plan section 2 (NFR-SEC-1, NFR-OPS-3)."

### [A28] [`egm-structural`] - Severity: Major
*   **Affected Scope:** [Lines 747](../docs/designs/kairos.md#L747), [755](../docs/designs/kairos.md#L755), [776](../docs/designs/kairos.md#L776), [813](../docs/designs/kairos.md#L813) (sequence diagrams)
*   **Discrepancy Description:** Semicolons in sequence-diagram message text will probably break rendering. No reviewer rendered the diagrams.
*   **Rationale:** Replace each `;` and render once to confirm.
*   ***AI Suggestion:*** PROPOSE FIX
*   ***Justification:*** Mermaid's sequence-diagram syntax treats `;` as a statement separator, so the text after it would parse as a new statement. The fix costs four line edits whether or not rendering actually fails.
*   ***Proposed Edit:***
    *   Location: line 747
        *   Change: `API->>DB: BEGIN, set_config(actor, request ID)`
    *   Location: line 755
        *   Change: `API->>DB: UPSERT employee_preferences (OFFICE), COMMIT`
    *   Location: line 776
        *   Change: `API->>DB: BEGIN, set_config actor, INSERT holidays (unique date)`
    *   Location: line 813
        *   Change: `API->>DB: BEGIN, available projects query filtered to Borealis`
