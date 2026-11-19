# Goldfish Review Report

**Target:** `Kairos/docs/designs/kairos.md` (lines 1068-1704, "## Detailed Implementation Plan (Slice 1 only)" to end)

**Stage:** Stage 9, Implementation Readiness Test

**Verdict:** Not ready. Two reviewers returned "not ready". The main problem is the sub-task boundaries. Several sub-tasks depend on files that a later sub-task creates, or need to edit files they do not list. Because the project rule is "touch only the files it lists", an implementer cannot finish ST-01, ST-02, ST-04, ST-06, ST-07 or ST-08 as written. No sub-task before ST-10 leaves `make up`, `make lint` and `make test` passing. Most of the remaining findings are contracts, defaults and test assertions that the plan leaves for the implementer to decide.

**Severity breakdown:** 3 Blocker, 27 Major, 27 Nit. These counts are after deduplication. The reviewers raised 87 items between them: 1 + 20 + 9 groups from style, 2 + 15 + 19 from implementation-readiness, and 1 + 17 + 20 from codebase-details.

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-implementation-readiness` | completed |
| `egm-codebase-details` | completed |
| `egm-style` | completed |

Greenfield note: codebase-details reported no "contradicts the code" or "non-existent reference" findings. Every file in the plan is marked Create or Modify. Table, column, endpoint, status code, CHECK value, cookie attribute, argon2 parameter and pool size names in the range match sections 5 and 6 and Q2, Q3 and Q6, except where a finding below says otherwise.

## Prioritized Findings (Blockers -> Major -> Nits)

---

### B1. Sub-tasks depend on files from later sub-tasks, so no sub-task before ST-10 leaves the build passing - Severity: Blocker
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1083-1100, 1147-1159, 1169-1179, 1233-1254, 1684-1694
* **Discrepancy Description:**
  * **`make up` (ST-01, line 1088):** it runs `migrate`, but the Alembic configuration arrives in ST-03. It runs `seed`, but `kairos/seed.py` arrives in ST-05. It builds `proxy`, which runs `vite build`, but `frontend/index.html` points at `/src/main.tsx`, which arrives in ST-08. So `make up` fails from ST-01 to ST-07.
  * **`make lint` (ST-01, lines 1092-1094):** it runs `openapi-check`, which needs `kairos.export_openapi` and `schema.ts` (ST-08). `tsc --noEmit` with no `src/` files fails with TS18003 ("No inputs were found"). The text writes `openapi-check` like a target name but does not list it as one.
  * **`make test` (line 1091):** it calls `test-backend`, `test-frontend` and `test-e2e`, which are not in the ST-01 target list. The behaviour of `test-backend` (a throwaway PostgreSQL 16 container on port 55433) appears only in the Verification Plan (lines 1687-1689). Vitest exits 1 when it finds no test files (true until ST-08), and `test-e2e` needs `e2e/` (ST-10).
  * **ST-01 `health/router.py` and `test_health.py` (lines 1147-1156):** the readiness check needs the async engine (`db.py`, ST-02), the `NOT_READY` error shape (`errors.py`, ST-02) and the test client (`conftest.py`, ST-02).
  * **ST-02 `conftest.py` and `factories.py` (lines 1233-1246):** they run `alembic upgrade head` (ST-03), build factories for tables that ST-03 creates, and provide `signed_in_client`, which needs sign-in (ST-04).
  * **ST-02 `test_errors_and_logging.py` (lines 1247-1254):** it tests a log line from a sign-in request (ST-04), a 401 shape (`current_employee`, ST-04), and a CSRF 403 on a write route. ST-02 has only GET health routes. The 404 case needs a route that can return 404, and the only defined one, `ENTRY_NOT_FOUND`, comes in ST-07.
* **Rationale & Recommendation:** The implementation-readiness standard requires every phase to build and pass its own tests. Do one of the following:
  * **Reorder the sub-tasks:** put `db.py` and `errors.py` before the health check, and put the migration and auth before the fixtures that use them.
  * **Or grow the shared files step by step:** give each sub-task the exact new content of each shared file, and list the file as Modify wherever it grows. The shared files are the Makefile targets, the `compose.yaml` services, `conftest.py` and `factories.py`, `index.html` with a stub `src/main.tsx`, and `package.json`.

  Either way, also do the following:
  * In ST-01, list `test-backend`, `test-frontend` and `test-e2e` as targets, and move the throwaway-database details into ST-01.
  * State whether `openapi-check` is its own target or a step inside `lint`.
  * Use `--passWithNoTests` for Vitest and `--pass-with-no-tests` for Playwright until ST-08 and ST-10, or skip those steps until their inputs exist.
  * For ST-02, say that its tests use a test-only router mounted in `conftest.py`, with routes that raise each error and accept a POST. Alternatively, move the sign-in redaction test to `test_auth.py` in ST-04.
  * Add a checkpoint line to each sub-task that names the `make` targets that must pass when it is done.

### B2. Shared files that later sub-tasks must change are not listed as Modify (`main.py`, `config.py`, `App.tsx`) - Severity: Blocker
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1140-1146 (ST-01 `main.py`). Also 1196-1232 (ST-02), 1303-1321 (ST-04), 1383-1398 (ST-06), 1471-1479 (ST-07), 1520-1527 and 1560-1564 (ST-08), 1576 and 1618-1625 (ST-09).
* **Discrepancy Description:**
  * **`main.py` registrations:** ST-01 says `create_app()` "registers middleware and routers". The things it must register are created later:
    * ST-02: the exception handlers, the request-ID and access-log middleware, the JSON log formatter and the CSRF dependency.
    * ST-04: the auth router, and the 24-hour expired-session cleanup task, which needs a lifespan hook.
    * ST-06: the projects and tasks routers.
    * ST-07: the time-entries router.

    None of ST-02, ST-04, ST-06 or ST-07 lists `main.py` as Modify. An implementer who follows the file lists cannot make the new code reachable, so the API tests in those sub-tasks cannot pass.
  * **`config.py` settings:** the same applies to settings that later sub-tasks add, such as `KAIROS_SEED_PASSWORD` and `KAIROS_ENV` for ST-05.
  * **Frontend routes:** in ST-08, `App.tsx` routes `/week/:isoWeek` and redirects `/` to the current week, and `SignInPage` sends the user to the current week. Both need `WeekPage.tsx` and `isoWeek.ts`, which are ST-09 files. ST-09 does not list `App.tsx` as Modify.
* **Rationale & Recommendation:**
  * **`main.py`:** list it as Modify in ST-02, ST-04, ST-06 and ST-07, and say exactly what each sub-task registers and in what order. The correlation-ID middleware is outermost, then the exception handlers.
  * **ST-04 cleanup task:** name the function that holds the cleanup task and how it is registered in the lifespan hook. Also name the advisory-lock call and its key, for example `pg_try_advisory_lock(<constant>)`, with the run skipped when the lock is not acquired.
  * **`config.py`:** either list every `Settings` field in ST-01, or list `config.py` as Modify wherever a field is added.
  * **Frontend:** either move `isoWeek.ts` into ST-08, add a placeholder week route there, and list `App.tsx` as Modify in ST-09, or create `App.tsx` in ST-09.

### B3. The `update_work_entry` rule for edits and date moves can be read the wrong way, and its order of checks is not given - Severity: Blocker
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1462-1467, 1476-1478 (refs lines 411-412, 451, 672)
* **Discrepancy Description:**
  * **Date-move wording:** "When the date changes, both days are locked in date order, and the entry's old minutes are subtracted." Read literally, this subtracts the old minutes from the check when the date changes. That is the one case where they must not be subtracted, because the target day never held them. An implementer who follows the text would let a 4-hour entry move onto a day that already has 22 hours (22 + 4 - 4 <= 24). The `day_totals` CHECK would then fail inside the trigger and return a 500 instead of 409 `DAILY_LIMIT_EXCEEDED`. Section 4 Q3 step 3 (lines 411-412) has the same ambiguity.
  * **Order of checks:** "The same checks" does not fix the order of the owner lookup (404), version compare (409), project availability (403), task check (422), day locks and `update_entry_if_version`. The order decides which error a request gets when several apply. The plan also does not say whether the old minutes are read before or after the day lock.
  * **Undefined edge cases:**
    * A `taskId` that does not exist could get either `TASK_INACTIVE` or `VALIDATION_FAILED`.
    * A malformed `If-Match` (no quotes, `W/`, `*` or a non-integer) has no defined response.
    * The plan does not say whether an update also upserts `last_work_location`.
    * Editing only the duration of an entry whose project is no longer available returns 403. The plan does not say whether that is intended (PRJ-10).
  * **`If-Match` on DELETE:** the router item requires `If-Match` only on PUT. Section 6 (line 672) and Q6 (line 451) require it on both PUT and DELETE, with 428 when it is missing. The test "A missing `If-Match` gives 428" (line 1506) does not say which method. The plan also does not say which responses set `ETag`.
* **Rationale & Recommendation:**
  * **Numbered steps:** write numbered steps for update and delete, like the create steps. State both date cases:
    * Date unchanged: lock the day and require existing - old + new + holiday <= 1,440.
    * Date changed: lock both days in ascending date order, and check only the new day: existing + holiday + new <= 1,440.
  * **Precedence:** fix the order of checks, including whether `ENTRY_CHANGED` is checked before or after the daily limit.
  * **Edge cases:** give each edge case above its error code.
  * **Router:** write "PUT and DELETE require `If-Match: "<version>"` and return 428 `VERSION_REQUIRED` without it", and test both methods. State that POST 201, PUT 200 and (see M12) `GET /me/week` set `ETag`.

---

### M1. Environment variables and database connection users are not specified - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1101-1115, 1120-1125, 1140-1146, 1233-1238, 1258-1262, 1293, 1338-1343, 1687-1689
* **Discrepancy Description:**
  * **Unnamed users:** the plan never names "the owner and app database users and passwords", yet `compose.yaml`, `.env.example`, `01-roles.sh` and `config.py` must agree on them.
  * **One URL for two roles:** there is a single `DATABASE_URL`. The API connects as `kairos_app`, while migrations, the seed, the e2e helper and the "cannot drop tables" test connect as `kairos_owner`. The host also differs: `db:5432` inside the network, and `localhost:55432` or `localhost:55433` from the host.
  * **Missing superuser password:** the official postgres image does not initialise without `POSTGRES_PASSWORD`, and that variable is not listed.
  * **Settings fields:** the `Settings` fields, which of them are required, and the allowed values of `KAIROS_ENV` (the seed test uses `production`) are not listed. "A safe local value" is vague.
  * **Test database:** the test container on port 55433 does not mount `db/init/01-roles.sh`. It therefore has no `kairos_owner`, no `kairos_app` and no `btree_gist`. `CREATE EXTENSION btree_gist` must also run while connected to the `kairos` database.
  * **Test roles:** the plan does not say which role pytest and the app under test use. Clearing tables with TRUNCATE needs owner rights, because `kairos_app` has DELETE only.
* **Rationale & Recommendation:**
  * **Variable list:** list every variable with its exact name and default, for example `POSTGRES_PASSWORD`, `KAIROS_OWNER_PASSWORD`, `KAIROS_APP_PASSWORD`, `DATABASE_URL` (app role), `KAIROS_OWNER_DATABASE_URL`, `KAIROS_TEST_DATABASE_URL` and `KAIROS_ENV` in {local, test, production}. Mark which are required.
  * **Image variables:** name the image variables with full default references instead of `KAIROS_*_IMAGE`.
  * **Readers:** state which variable `Settings`, `migrations/env.py` and `seed.py` each read.
  * **Test database:** mount `db/init` in the test container. Optionally, move `CREATE EXTENSION IF NOT EXISTS btree_gist` into migration 0001. It is a trusted extension since PostgreSQL 13, so the database owner can create it.
  * **Test roles:** state that the app under test uses `kairos_app`, and that fixtures and table clearing use `kairos_owner`.

### M2. The plan does not say whether linters and tests run on the host or in containers - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1085-1100, 1135-1139, 1157-1168, 1180-1188, 1640-1648, 1687-1694
* **Discrepancy Description:**
  * **Conflicting claims:** line 1099 says "every target calls podman or podman-compose", and the README prerequisites (lines 1183-1184) are only Podman, podman-compose and make. But ruff, mypy, pytest, ESLint, tsc, Vitest and Playwright (Chromium and WebKit) all need Python 3.12 with uv or Node.
  * **Missing details:** if these tools run in containers, no image, dev-dependency stage or source mount is given. The backend Containerfile does not say whether dev dependencies are installed. No image variable exists for Playwright.
  * **WebKit libraries:** on Linux, Playwright's WebKit needs extra system libraries.
  * **Stack for e2e:** `make test` includes `test-e2e`, which needs the stack from `make up`. The plan does not say whether `make test` starts the stack or expects it to be running.
* **Rationale & Recommendation:** For each target, decide host or container and write its exact command. If a tool runs on the host, add it with its version to the README prerequisites. If it runs in a container, name the image and the dev or test build stage, and say how it reaches the test database (port 55433) and the stack (port 8443). Say whether `make test` runs `make up` first, or fails with a clear message when the stack is down.

### M3. The `pg_isready` wait can pass before the database is ready - Severity: Major
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1088-1089
* **Discrepancy Description:** On first start, the postgres image runs a temporary server that listens only on a Unix socket while the init scripts run. `podman exec db pg_isready` uses that socket by default. So it can report ready before `01-roles.sh` finishes and before the real server listens on TCP, and `migrate` then fails.
* **Rationale & Recommendation:** Run `pg_isready -h 127.0.0.1 -U <owner> -d kairos` in a retry loop with a timeout.

### M4. The error catalogue, message text, `details` formats and framework errors are left to the implementer - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1198-1205, 1213-1217, 1250, 1446-1467, 1612-1615 (refs lines 305, 348)
* **Discrepancy Description:**
  * **No single list of codes:** "a subclass for each code in Technical Plan section 6" is not a usable list. Section 6 mixes Slice 1 codes with `ACCOUNT_LOCKED` (Slice 2) and later codes. `INTERNAL_ERROR` is defined in section 2, `NOT_READY` sits in an endpoint table row, and `EXPORT_BUSY` and `DATE_NOT_EDITABLE` are elsewhere.
  * **Missing message text:** NFR-UX-4 requires messages that say what to do next, and the UI shows them as written. No message text is given for `VALIDATION_FAILED`, `TASK_INACTIVE`, `ENTRY_NOT_FOUND`, `VERSION_REQUIRED`, `NOT_READY`, `INTERNAL_ERROR`, the duration rule (line 348) or the API body of `ENTRY_CHANGED`.
  * **Undefined `details` formats:**
    * The format of `details.fields[].field` (`durationMinutes`, or Pydantic's `["body","durationMinutes"]`) and the `problem` wording are not defined. The frontend mapping at lines 1614-1615 depends on both.
    * The key that holds the current entry in the `ENTRY_CHANGED` details is not defined, and the "Show latest" action depends on it.
    * Section 2 (line 305) requires the correlation ID on `INTERNAL_ERROR`, but the plan does not say where it goes in the body.
  * **Starlette's own 404 and 405:** an unknown route or a wrong method raises Starlette's `HTTPException`, which returns `{"detail": ...}`. So the "same shape for 404" test (line 1250) fails.
  * **500 response without the request ID:** Starlette runs a handler registered for `Exception` from its outermost error middleware, outside the app's own middleware. So the 500 response would not carry `X-Request-ID`.
  * **Hard-to-read sentence:** the handler sentence at lines 1200-1205 has two nested parentheticals.
* **Rationale & Recommendation:**
  * **Code table:** add a table to ST-02 with the code, HTTP status, exact message and `details` schema for every Slice 1 code. Add `NOT_FOUND` and `METHOD_NOT_ALLOWED` for framework errors, and register a handler for Starlette's `HTTPException`.
  * **Field names:** define `details.fields[].field` as the camelCase body field name, and give the `problem` text for each Pydantic rule used in Slice 1.
  * **Fixed keys:** fix `details.entry` (an `Entry`) for `ENTRY_CHANGED` and `details.requestId` for `INTERNAL_ERROR`.
  * **Middleware:** say that `middleware.py` is a pure ASGI middleware, not `BaseHTTPMiddleware`. It catches unhandled exceptions and builds the `INTERNAL_ERROR` response itself, including the header.
  * **Handler sentence:** split it into one bullet per exception type.

### M5. The log redaction matching rule is ambiguous - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1209-1211, 1251-1252
* **Discrepancy Description:** "The `password`, `token`, `cookie` and `authorization` keys at any depth" leaves three questions open:
  * Is matching case-insensitive? Section 2 writes `Authorization` and `Cookie`.
  * Is it an exact match or a substring match? This decides whether `token_hash` and `password_hash` are redacted.
  * Is the `set-cookie` response header covered? The test at lines 1251-1252 depends on this, because the sign-in response carries `Set-Cookie`.
* **Rationale & Recommendation:** State the rule. For example: "Case-insensitive exact match on the keys `password`, `token`, `cookie`, `set-cookie` and `authorization`. Values are replaced with `[REDACTED]`."

### M6. The async test setup is not specified, and the common defaults fail - Severity: Major
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1131-1134, 1218-1223, 1237-1238
* **Discrepancy Description:**
  * **Event loops:** `db.py` creates one engine at import time, and asyncpg connections belong to one event loop. By default, pytest-asyncio gives each test a new loop. A pooled connection reused in the next test then fails with an "attached to a different loop" error.
  * **Migrations:** the async Alembic `env.py` calls `asyncio.run()`, which cannot be called from inside an async session fixture.
* **Rationale & Recommendation:**
  * **pytest-asyncio settings:** name the pytest-asyncio version and settings, for example `asyncio_mode = "auto"` with session-scoped default fixture and test loop scopes. Alternatively, use `NullPool` in tests.
  * **Migrations:** run `alembic upgrade head` from a synchronous session fixture or as a subprocess.

### M7. Test clients will not send the `Secure` session cookie - Severity: Major
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1241-1244, 1310-1313, 1325
* **Discrepancy Description:** The session cookie is `Secure` with the `__Host-` prefix. httpx stores cookies with `http.cookiejar`, which sends `Secure` cookies only on `https` requests. With the common `base_url="http://test"`, every request from `signed_in_client` gets 401.
* **Rationale & Recommendation:** State in `conftest.py` that `AsyncClient` uses `ASGITransport(app=...)` with `base_url="https://test"`.

### M8. The ST-03 migration says "exactly as in the data dictionary", but some dictionary items belong to later slices - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-codebase-details`
* **Affected Scope:** Lines 1266-1272
* **Discrepancy Description:** The data dictionary marks some items "Slice 3" or "From Slice 4": the `leave_type` column, the 240/480 leave rule and the `(entry_date, project_id)` index. "Exactly as in" does not say whether the Slice 1 migration includes them. Also, the `assignments (employee_id, start_date)` index appears only in the Q4 table, not in the dictionary.
* **Rationale & Recommendation:** List each table, column, constraint and index that is in or out of migration 0001. For example: includes `leave_type` and the `kind` CHECK for LEAVE; includes `assignments (employee_id, start_date)`; excludes the `(entry_date, project_id)` index, `period_locks` and `audit_log`.

### M9. The database function and trigger contracts are incomplete - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1263-1276, 1282-1293, 1428-1431 (refs lines 360-367, 599-635)
* **Discrepancy Description:**
  * **Function signatures:** `employee_holidays(...)` and `employee_day_hours(...)` have no return column names or types and no volatility. ST-07's `holidays_for_range` depends on them, and they are written in a different sub-task.
  * **Trigger with no `day_totals` row:** `time_entries_totals()` says only "adds or subtracts". It does not say that it upserts the `day_totals` row when none exists. The seed and the trigger test insert entries without calling `lock_day`, so a plain `UPDATE` would lose minutes without any error.
  * **Column default:** `day_totals.entry_minutes` needs `NOT NULL DEFAULT 0` for `lock_day`'s `INSERT ... ON CONFLICT DO NOTHING`.
  * **Unstated details:** the plan does not say whether rows that drop to zero are kept, whether `day_totals` has a foreign key to `employees` and the usual `created_at` and `updated_at` columns, or how `updated_at` is kept current on any table.
  * **Untested function:** `employee_day_hours` has no test and no caller in Slice 1.
* **Rationale & Recommendation:**
  * **Functions:** give each function's full SQL signature with `RETURNS TABLE(...)` and its body rules.
  * **Trigger:** specify the trigger as `INSERT ... ON CONFLICT (employee_id, entry_date) DO UPDATE` for each operation, and spell out the date-move case.
  * **`updated_at`:** state how it is kept current.
  * **Tests:** add an `employee_day_hours` test to `test_schema.py`, or drop the function from Slice 1.

### M10. The seed data is called fixed but is not specified, and a re-run on a later day fails - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1338-1379
* **Discrepancy Description:**
  * **Missing seed values:** the plan does not give:
    * the employee names, designations, departments and roles, or which person is the HR administrator;
    * the client project and client names;
    * the project start and end dates (PRJ-9 needs a project's start date before every seeded entry and every date of joining);
    * which employee is assigned to which project, or which employee has the ended assignment.
  * **"For each employee":** it is unclear whether this includes the HR administrator.
  * **Entries:** the plan does not say how entries are spread (per day, projects, tasks, locations, activity text) or give the random seed value.
  * **PRJ-9 conflict:** one assignment ends 10 days before the run date, which falls inside the previous week. The seed bypasses the service, so it can create entries that PRJ-9 would reject.
  * **"Up to today":** it is unclear whether today and weekends are included, and in which time zone. The test at lines 1370-1371 fails on a Monday if today is excluded. No entry point lets a test inject the run date.
  * **Re-runs:** assignment dates are relative to the run date, and rows are "found again by their natural key", which is never defined for assignments. If the key includes the start date, a run on the next day inserts an overlapping assignment. The `EXCLUDE` constraint then rejects it, and `make up` fails from the second day on.
* **Rationale & Recommendation:**
  * **Seed table:** add a literal table of employees, projects and assignments with relative dates. For example, common projects start on `run_date - 3650 days`.
  * **Entries:** state the entry-generation algorithm and the seed value. Add: "Each seeded entry uses a project that is available to that employee on that date."
  * **Date range:** define the range as "up to and including today (IST), Monday to Friday".
  * **Re-runs:** define the assignment key as (employee code, project name), and skip the insert when an assignment for that pair already exists.
  * **Test entry point:** define `run(session, today: date)` so tests can set the run date.

### M11. Seed emails on `.local` fail Pydantic `EmailStr` - Severity: Major
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1303-1313, 1347-1348
* **Discrepancy Description:** `<first-name>@kairos.local` uses a special-use domain. `email-validator`, which `EmailStr` uses, rejects such domains even when deliverability checks are off. If the sign-in schema uses `EmailStr`, every seeded account gets 422 at sign-in. The plan does not say whether the field is `EmailStr` or `str`, and `email-validator` is not in the dependency list.
* **Rationale & Recommendation:** Either use `str` with lowercasing and a length limit, or use `EmailStr`, add `email-validator`, and change the seed domain (for example to `kairos.example.com`).

### M12. Week and Day response fields are under-defined - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1423, 1441-1445, 1479 (refs lines 682-697)
* **Discrepancy Description:**
  * **`Day.weekday`:** its format is undefined. It could be ISO 1-7, `"MON"` or `"Monday"`.
  * **`complete`:** the rule is not stated. The PRD glossary rule is that the total of work, leave and holiday is at least the expected minutes, applied to the whole week, including future days.
  * **Totals:** it is unclear whether `Week.totalMinutes` and `Day.totalMinutes` include holiday minutes. Line 1487 implies they do.
  * **`ETag`:** it is unclear whether `GET /me/week` sets `ETag`.
* **Rationale & Recommendation:** Give the Pydantic field types and the exact rule for each derived field. List which endpoints return `ETag`.

### M13. The concurrency test can pass when locking is broken - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1509-1516
* **Discrepancy Description:**
  * **The CHECK hides a missing lock:** `CHECK (entry_minutes BETWEEN 0 AND 1440)` on `day_totals` turns a missing lock into a 500 for the losing request. So "exactly one succeeds" and "never ends over 1,440" both still pass without `lock_day`.
  * **The second case cannot fail:** with 16 hours on the target day, two 4-hour moves reach exactly 1,440, which is allowed, so both succeed with or without locking. The second case also has no starting state of its own.
  * **"Two sessions":** it is unclear whether this means sign-in sessions or database sessions.
* **Rationale & Recommendation:**
  * **Assertions:** assert that the losing request gets 409 `DAILY_LIMIT_EXCEEDED` and never 500, and that the final `day_totals` equals the sum of `time_entries`.
  * **Second case:** give it its own setup. For example, put 17 hours on the target day and move two 4-hour entries from two other days onto it. Expect exactly one 200 and one 409, with no deadlock.
  * **Wording:** replace "sessions" with "two signed-in clients for the same employee".

### M14. Endpoint tests are missing, and the "every endpoint" claim overstates coverage - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1322-1334, 1393-1410, 1480-1508, 1695-1697
* **Discrepancy Description:**
  * **Missing tests:** `GET /api/v1/tasks` has no test file. `GET /me/week` has no failure test (401, or 422 for a bad `date`). `POST /auth/sign-out` has no failure test. `TASK_INACTIVE` has no test.
  * **Overstated claim:** lines 1695-1697 claim every endpoint has a success test and a failure test. `GET /health/live` has no failure response by design.
* **Rationale & Recommendation:**
  * **New tests:** add `backend/tests/api/test_tasks.py` to ST-06, covering active tasks only, sorted by name, `ETag` present, and 401 without a session. Add 401 and 422 cases to `test_week.py`, a sign-out failure case (401 without a session, 403 without CSRF), and a `TASK_INACTIVE` case.
  * **Claim:** narrow it to "...except `GET /health/live`, which has no failure response".

### M15. The traceability table leaves out NFRs that Slice 1 delivers and tests - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1070-1074, 1663-1680, 1698-1699
* **Discrepancy Description:** The Scope delivers NFR-REL-3 and NFR-OPS-3, and the plan also tests NFR-SEC-1, NFR-SEC-3, NFR-DATA-1 and NFR-UX-5. None of these has a row. So "done when every row in the traceability table has passing tests" does not cover them.
* **Rationale & Recommendation:** Add a row for each of these NFRs, with its files and tests.

### M16. The AUTH-5 row claims "8-hour expiry", but no test checks the 8-hour value - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1325-1331, 1668
* **Discrepancy Description:** The listed test sets `expires_at` to the past. No test checks that `expires_at` equals the sign-in time plus 8 hours, or that it does not move when the employee is active ("regardless of activity").
* **Rationale & Recommendation:** Add a test to `test_auth.py`: `expires_at` equals the sign-in time plus 8 hours and is unchanged after a later authenticated request. Otherwise, relabel the row.

### M17. CUJ 5 has no end-to-end test, and the e2e files mix identifier schemes - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1649-1656, 1663-1680 (ref lines 192-203)
* **Discrepancy Description:**
  * **CUJ 5 gap:** ST-10 covers "CUJs 1 to 4 and 6". CUJ 5 (edit, delete, move between weeks, jump to a date, ISO week in the header) has no end-to-end test, and no reason is given.
  * **Mixed identifiers:** file names use PRD S-IDs (`s01`, `s04`), while the sentence uses CUJ numbers. CUJ 6 has no S-ID, so "with the scenario ID in the test name" cannot apply to `sign-in-out.spec.ts`. `keyboard-entry` is not a CUJ.
  * **Expiry:** the plan does not say how the e2e test simulates the 8-hour expiry in CUJ 6.
* **Rationale & Recommendation:**
  * **CUJ 5:** add `e2e/tests/cuj05-edit-delete-navigate.spec.ts` and reference it in the TIME-2, TIME-7 and TIME-14 rows.
  * **Naming:** use one naming scheme, such as "CUJ-n (S-x)".
  * **Expiry:** state how the test simulates expiry, for example by setting `expires_at` through the SQL helper.

### M18. The end-to-end harness is under-specified - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1166-1167, 1640-1656
* **Discrepancy Description:**
  * **Package resolution:** `@playwright/test` is a dev dependency of `frontend/package.json`, but the specs live in the root `e2e/` folder. Node resolves packages upward from `e2e/`, so it never finds `frontend/node_modules`. No `package.json`, `tsconfig` or lint scope covers `e2e/`.
  * **SQL helper:** the test-only SQL helper needs a Node Postgres client such as `pg`, and a connection string. Neither is listed.
  * **Password hashes:** fixture employees need an argon2id `password_hash`. The plan does not say how one is produced.
  * **Shared database:** fixture data is written into the `make up` development database. The plan does not say how emails and codes stay unique across runs, or whether the data is cleaned up.
  * **Layout:** `e2e/` is not in the repository layout in engineering-standards.md.
* **Rationale & Recommendation:**
  * **Packages:** add `e2e/package.json` and `e2e/tsconfig.json` with `@playwright/test` and `pg`, or move the suite to `frontend/e2e/`.
  * **Hashes:** commit a precomputed argon2id hash for a fixed test password.
  * **Test data:** add a run-ID suffix to each email and code, and delete the fixture rows in teardown.
  * **Layout:** record the layout exception in the plan.

### M19. Frontend dependencies are incomplete, and the Node version is end-of-life - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1157-1168, 1171-1172, 1581-1582, 1592-1595, 1626-1636
* **Discrepancy Description:** These packages are missing:
  * `@emotion/react` and `@emotion/styled`, which MUI v6 requires.
  * `@mui/icons-material`, for the kind icons.
  * A date picker for "jump to any date". MUI core has none, so the plan needs `@mui/x-date-pickers` with an adapter, or a native `<input type="date">`.
  * The font packages, for example `@fontsource/inter` and `@fontsource/roboto-condensed`.
  * `jsdom`, `@testing-library/user-event` and `@testing-library/jest-dom`.
  * A React Router major version. v6 uses `react-router-dom`, v7 uses `react-router`.
  * `typescript-eslint` and `eslint-plugin-react-hooks`.

  The Vitest configuration and setup file are not named. Neither ST-08 nor ST-09 lists `package.json` as Modify. The proxy build uses Node 20, which reached end-of-life on 2026-04-30.
* **Rationale & Recommendation:** List every npm package with its major version in ST-01, or list `package.json` as Modify where packages are added. Add the Vitest environment and setup file to the file lists. Use a Node 22 or Node 24 LTS image.

### M20. The Content-Security-Policy will probably break MUI styling - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1177-1179
* **Discrepancy Description:**
  * **Styles:** `default-src 'self'` has no `style-src` exception. Emotion, which MUI uses, injects `<style>` elements at runtime. MUI's guidance for a strict policy calls for a nonce or `'unsafe-inline'`, and a static file served by Caddy cannot easily get a nonce per request.
  * **Fonts:** Vite inlines assets under 4 KiB as `data:` URIs, which `font-src` (falling back to `default-src 'self'`) blocks.
  * **Result:** the page could render unstyled, which would also break the 360 px layout test.
* **Rationale & Recommendation:**
  * **Policy:** state the exact policy, for example `default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'`.
  * **Fonts:** set `build.assetsInlineLimit: 0`.
  * **Check:** add an e2e check that styles are applied.

### M21. The entry form has no Date field, but date moves are in scope - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1601-1603 (refs lines 339, 1462-1464, 1507, 1515)
* **Discrepancy Description:** The field order is Project, Task, Duration, Work location and Activity. Section 3 says the date "defaults to the column it was opened from", the API supports changing `entryDate`, and CUJ 5 moves entries between days. The plan does not say whether the form has a Date field or where it sits in the tab order.
* **Rationale & Recommendation:** Either add a Date field, with its position and its default (the column's date), and confirm that the keyboard path is still 4 interactions, or state that moving an entry is API-only in Slice 1.

### M22. The keyboard-path tests rest on an open question and on unstated form behaviour - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1608-1611, 1633, 1655 (refs lines 144-148, 339)
* **Discrepancy Description:**
  * **Open question:** Open Questions (lines 144-148) says the NFR-UX-5 counting rule "needs confirmation before the Slice 1 test for NFR-UX-5 is written". The plan still specifies "saves in 4 interactions" with no condition.
  * **Initial focus:** the 4-step path depends on the form opening with focus on Project. ST-09 does not say so.
  * **No last location:** there is no Work location default for when `Me.lastWorkLocation` is null (a new employee). Without one, the keyboard path cannot save.
* **Rationale & Recommendation:** Add to ST-09: "The form opens with focus on Project. Work location defaults to `OFFICE` when there is no last-used location." Either record the confirmed counting rule here, or mark both keyboard tests as blocked until it is confirmed.

### M23. The behaviour of the conflict actions and of errors without field details is not specified - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1612-1615 (ref lines 455-456)
* **Discrepancy Description:**
  * **Conflict actions:** "Show latest" and "Apply my changes again" are named, but their behaviour is not described.
  * **Message source:** "Shows the message" does not say whether this is the server `message` or the fixed text in Q6.
  * **Errors without fields:** errors that have no `details.fields`, such as `DAILY_LIMIT_EXCEEDED` and `PROJECT_NOT_AVAILABLE`, have no stated place on screen.
* **Rationale & Recommendation:** Define the behaviour:
  * **"Show latest":** replaces the form values with `details.entry` and discards the user's edits.
  * **"Apply my changes again":** keeps the user's values and resubmits with `If-Match` set to the current version.
  * **Errors without `details.fields`:** appear in `ErrorMessage` at the top of the modal.

### M24. It is unclear which component redirects on a 401, and the wording is garbled - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-codebase-details`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1527, 1534-1542 (ref lines 325-326)
* **Discrepancy Description:**
  * **Wording:** "A route guard calls `GET /me` and sends a 401 to `/sign-in`" reads as sending a status code to a route.
  * **Conflict with section 3:** section 3 says `client.ts` redirects on any 401, but ST-08's `client.ts` only "throws ApiError". So a save after the 8-hour expiry would show an error instead of the redirect that CUJ 6 requires.
  * **Sign-in page:** its own 401 (`INVALID_CREDENTIALS`) must not redirect.
* **Rationale & Recommendation:** Write "If `GET /me` returns 401, the route guard redirects to `/sign-in`." Then decide one of the following, and add a unit test for it:
  * `client.ts` redirects on a 401 `SESSION_ENDED` from any call except `signIn`.
  * Only the guard redirects.

### M25. The Vite development proxy points at a port that is not published - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-codebase-details`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1104-1105, 1168
* **Discrepancy Description:** `api` listens on port 8000 "inside the network only", but Vite forwards `/api` to `localhost:8000`. Nothing listens on that port on the host. Safari also refuses `Secure` and `__Host-` cookies on `http://localhost`.
* **Rationale & Recommendation:** Do one of the following:
  * Publish `127.0.0.1:8000` for development.
  * Point the Vite proxy at `https://localhost:8443` with `secure: false`.
  * Remove the dev proxy.

### M26. White text on the work-entry colour fails WCAG AA, and card text colours are not specified - Severity: Major
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1528-1533, 1592-1595
* **Discrepancy Description:** White on `#00A67E` has a contrast ratio of about 3.1:1. NFR-UX-3 requires 4.5:1 for 14 px text. The plan says "a solid card in the kind's colour" but gives no text colour. White passes on `#B45309` and `#5B21B6`, and `#1A1A1A` on `#00A67E` gives about 5.6:1.
* **Rationale & Recommendation:** Set `contrastText` for each palette entry in `theme.ts`: `#1A1A1A` for work, and `#FFFFFF` for leave and holiday. Add a contrast or axe check.

### M27. File lists use bare filenames and leave out supporting files - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-implementation-readiness`, `egm-codebase-details`
* **Affected Scope:** Lines 1076-1077, 1140-1159, 1303-1304, 1383-1384, 1393-1394, 1543-1550, 1626-1628, 1649-1652
* **Discrepancy Description:**
  * **Bare filenames:** line 1076 says all paths are relative to `Kairos/`, but several items use bare names such as `router.py`, `service.py`, `isoWeek.test.ts` and `s04-daily-limit.spec.ts`.
  * **Missing files:** these files are not listed:
    * `__init__.py` for `health/`, `security/`, `auth/`, `projects/`, `tasks/`, `time_entries/`, `tests/api/` and `tests/db/`;
    * `frontend/src/api/openapi.json`, which the description mentions;
    * `frontend/src/vite-env.d.ts`;
    * the Vitest setup file;
    * the MUI type augmentation for the custom palette keys (`work`, `leave`, `holiday`) and the `numeric` text style.
  * **Why it matters:** under the rule "touch only the files it lists", every one of these gaps blocks the implementer.
* **Rationale & Recommendation:** Write every path in full. Add each missing file to the sub-task that first needs it. For `__init__.py`, either list the files or state that namespace packages are intended and configure mypy for them. Also state that `export_openapi` runs without a database or required settings.

---

### N1. Function signature problems - Severity: Nit
* **Reviewer Lens:** `egm-style`, `egm-codebase-details`
* **Affected Scope:** Lines 1224-1225, 1342, 1428, 1432-1437
* **Discrepancy Description:**
  * **Reserved word:** `holidays_for_range(employee_id, from, to)` uses `from`, which is a reserved word in Python.
  * **Missing parameters:** `insert_entry`, `delete_entry_if_version` and `upsert_last_location` have no parameters listed.
  * **Seed actor:** `transaction(actor_id)` takes an employee ID, but the seed "sets the actor to `system`". The plan does not say how `system` is passed.
  * **Type:** asyncpg needs the actor ID as a `str` for `set_config`.
* **Rationale & Recommendation:** Use `from_date` and `to_date`, list every parameter, and say how the `system` actor is passed and that the ID is converted to `str`.

### N2. The uvicorn target does not match `main.py` - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1138-1145
* **Discrepancy Description:** uvicorn starts `kairos.main:app`, but `main.py` only describes `create_app()`.
* **Rationale & Recommendation:** Add a module-level `app = create_app()`, or use `--factory kairos.main:create_app`.

### N3. The README contents contradict themselves - Severity: Nit
* **Reviewer Lens:** `egm-style`, `egm-codebase-details`
* **Affected Scope:** Lines 1187, 1193
* **Discrepancy Description:** The ST-01 README bullet includes the seed-account table, but line 1193 says ST-05 adds it.
* **Rationale & Recommendation:** Remove the seed-account table from the ST-01 list.

### N4. Image build details are not specified - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1106-1107, 1129, 1137-1138
* **Discrepancy Description:**
  * **Image variables:** they can only reach the Containerfiles through `ARG`s and compose `build.args`, which the plan does not mention.
  * **uv:** how uv is installed in the image is not stated.
  * **Test database image:** `test-backend` should use `KAIROS_POSTGRES_IMAGE`.
  * **SQLAlchemy:** it should be pinned as `sqlalchemy[asyncio]>=2.0,<2.1` so that `greenlet` is installed.
* **Rationale & Recommendation:** State each of these.

### N5. Makefile target details are incomplete - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-style`
* **Affected Scope:** Lines 1088-1099, 1106, 1114, 1687-1692
* **Discrepancy Description:**
  * **Undescribed targets:** `down` and `logs` are not described. It is unclear whether `down` keeps the volume.
  * **Build step:** `up` should build the images before `podman-compose run`.
  * **Cleanup:** `test-backend` should stop the test container even when pytest fails.
  * **psql user:** `psql` does not name the user it connects as.
  * **Unnamed commands:** "backend image with a different command" should name `alembic upgrade head` and `python -m kairos.seed`.
  * **Misplaced rule:** the rule that `make up` copies `.env.example` belongs under the `up` target.
  * **Lint commands:** the Verification Plan lists `ruff`, while line 1092 also has `ruff format --check`.
* **Rationale & Recommendation:** Define each target fully and keep the Verification Plan identical to the target list.

### N6. Caddy setup details are missing - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1169-1179, 1186
* **Discrepancy Description:**
  * **Site address:** the site address and port mapping (`localhost:8443` to container 443) are not stated.
  * **Certificate:** `tls internal` issues short-lived certificates, and its root certificate changes on every rebuild unless `/data` is on a named volume. So "accept the local certificate once" is not accurate.
  * **Header values:** the values of `X-Content-Type-Options` and `Referrer-Policy` are not given.
* **Rationale & Recommendation:** State the site address, keep `/data` on a named volume, document how to trust `root.crt`, and give each header's value.

### N7. Password handling details are open - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`
* **Affected Scope:** Lines 1297-1302, 1307
* **Discrepancy Description:** `needs_rehash()` is never called. Tests may need lower argon2 parameters to run fast. The sign-in password field has no maximum length, so very long inputs are costly to hash.
* **Rationale & Recommendation:** Say whether sign-in rehashes when the parameters change, and which parameters tests use. Add a maximum password length.

### N8. Clearing the session cookie needs the same attributes - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Line 1314
* **Discrepancy Description:** A `__Host-` cookie is cleared only when the clearing cookie also has `Secure` and `Path=/`.
* **Rationale & Recommendation:** State these attributes for sign-out.

### N9. The request-ID SQL comment is inconsistent and costly - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1222, 1226-1227 (ref line 273)
* **Discrepancy Description:**
  * **Cache misses:** a different comment on every statement defeats asyncpg's prepared-statement cache. This is relevant to the Slice 5 p99 target.
  * **Owner:** the plan says an "event hook" adds the comment, but section 2 says the repository layer does.
  * **Undefined term:** "the interactive pool" is not defined.
* **Rationale & Recommendation:** Align on one mechanism, and say what is used when there is no request (seed, migrations). Consider `application_name` instead.

### N10. `GET /tasks` `ETag` handling is incomplete - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1397-1398, 1476-1477
* **Discrepancy Description:**
  * **No 304:** nothing handles `If-None-Match` with a 304 response, so the `ETag` does nothing for Q7.
  * **Hash input:** the input to the `ETag` hash is not stated.
  * **`If-Match` format:** the accepted formats (`"1"`, `W/"1"`, `*`) are not stated.
* **Rationale & Recommendation:** Specify the 304 handling and the hash input, and list the accepted `If-Match` formats.

### N11. The project search parameter `q` is under-specified - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1386-1392, 1409
* **Discrepancy Description:**
  * **Escaping:** `%` and `_` are not escaped in `ILIKE`.
  * **Code prefix:** matching a code prefix needs `code::text LIKE :q || '%'`.
  * **Limits:** the maximum length and the behaviour for an empty `q` are not stated.
  * **Vague test:** the test "Search ... works" gives no inputs or expected results.
* **Rationale & Recommendation:** Specify each of these.

### N12. The readiness test cannot swap the engine - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1154-1156
* **Discrepancy Description:** The test points the engine at a closed port, but the engine has no override hook.
* **Rationale & Recommendation:** Add a `get_engine` dependency.

### N13. A shared component imports from a feature folder - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1616-1618
* **Discrepancy Description:** `components/DurationInput.tsx` depends on `features/week/duration.ts`. This goes against the convention "shared components in `components/`".
* **Rationale & Recommendation:** Move `duration.ts` to a shared location.

### N14. Firefox is not tested - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`, `egm-style`
* **Affected Scope:** Line 1648
* **Discrepancy Description:** The Playwright projects are Chromium and WebKit. NFR-UX-1 lists Firefox.
* **Rationale & Recommendation:** Add Firefox, or record why it is left out.

### N15. The time zone for "current week" is not stated - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1525-1526
* **Discrepancy Description:** The frontend could use IST or the browser's local time zone. Everything else in the plan uses IST.
* **Rationale & Recommendation:** State IST.

### N16. Display strings and enum values are not given - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1421, 1579-1596
* **Discrepancy Description:**
  * **Display strings:** these are not given: the header date-range format, the kind labels and icons, the labels for `OFFICE`, `CLIENT_SITE` and `WFH`, and the delete-confirmation wording.
  * **Enum values:** the work-location enum values are not listed at line 1421.
* **Rationale & Recommendation:** List the exact strings and values.

### N17. Validation of `activity` and duration is incomplete - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`, `egm-style`
* **Affected Scope:** Lines 1419-1420, 1621-1622
* **Discrepancy Description:**
  * **Trimming:** it is not stated whether `activity` is trimmed before the length check.
  * **Empty values:** it is not stated whether an empty `activity` is stored as null.
  * **Duration bound:** `parseDuration` has no stated upper bound of 1,440.
* **Rationale & Recommendation:** State each rule.

### N18. The call path for `available_projects` is not stated - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1387-1390
* **Discrepancy Description:** It is unclear whether `time_entries/service.py` calls `projects.service` or `projects.repository`.
* **Rationale & Recommendation:** Name the module it calls.

### N19. The scope of the database grants is not stated - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1273-1274
* **Discrepancy Description:** It is unclear whether "all tables" includes `alembic_version`, and whether default privileges are set for tables that later slices add.
* **Rationale & Recommendation:** State both.

### N20. CUJ 6 wording conflicts with the e2e fixture - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1645-1647 (ref line 203)
* **Discrepancy Description:** CUJ 6 says "a seeded employee", but ST-10 says the e2e tests do not use the seed.
* **Rationale & Recommendation:** Change the CUJ wording to "an employee created by the test fixture".

### N21. Some small tests are missing - Severity: Nit
* **Reviewer Lens:** `egm-implementation-readiness`
* **Affected Scope:** Lines 1250-1254, 1565-1572, 1626-1636
* **Discrepancy Description:** No test covers these cases:
  * a non-UUID `X-Request-ID` is replaced;
  * redaction works on nested keys;
  * the SQL comment is present;
  * ProjectSelect shows a message when there are no results;
  * `EntryCard` hides its actions when `editable` is false;
  * TopBar sign-out works.
* **Rationale & Recommendation:** Add these tests.

### N22. Some theme values are not specified - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1163, 1528-1533
* **Discrepancy Description:**
  * **Name collision:** the palette name `warning` collides with MUI's built-in `palette.warning`.
  * **Unnamed values:** the top bar and primary colours, the "condensed bold" font and the focus outline specification are not given.
  * **Font link:** Roboto Condensed appears without a link to the "condensed face" in section 3.
* **Rationale & Recommendation:** Name each value, and say whether `warning` overrides MUI's palette entry.

### N23. The traceability table format is inconsistent - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1663-1680
* **Discrepancy Description:**
  * **Paths:** paths such as `auth/service.py` conflict with line 1076.
  * **Mixed column:** the "Main files" column mixes paths, function names and prose. For example, the TIME-14 cell says "no rule that forbids duplicates".
  * **Unsupported citation:** the TIME-16 row cites `WeekPage.test.tsx`, which is not described as checking those fields.
* **Rationale & Recommendation:** Use full paths and list only files in that column.

### N24. Some sentences are wordy or vague - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1087, 1138, 1172-1173, 1214, 1231-1232, 1287, 1318-1321, 1351-1352, 1361, 1443-1444, 1483-1484, 1487, 1635-1636
* **Discrepancy Description:** Examples:
  * **Line 1087:** "These targets:" is a fragment.
  * **Line 1214:** the rule for a missing or invalid `X-Request-ID` should be explicit.
  * **Line 1318:** the bare bullet "`GET /me`." does not say what the endpoint returns.
  * **Lines 1319-1321:** "A start-up task ... every 24 hours" is ambiguous, and the file that holds the task is not named.
  * **Lines 1483-1484:** week 53 needs concrete dates.
  * **Line 1487:** "proves the Q1 path" overstates what the test does.
  * **Lines 1635-1636:** "does not rely on colour alone" should name the text label and icon.
* **Rationale & Recommendation:** Apply the rewrites from the style review. For example, use "runs at start-up and then every 24 hours", "exercises the Q1 path", and "2026-W53 = 2026-12-28 to 2027-01-03".

### N25. Some terms drift - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Lines 1301, 1442, 1466
* **Discrepancy Description:**
  * **Holiday entries:** "holiday pseudo-entries" should be `HolidayEntry`, as in section 6.
  * **Scope wording:** "per process" here conflicts with "per instance" at line 397.
  * **Error code:** the 404 at line 1466 is not named `ENTRY_NOT_FOUND`.
* **Rationale & Recommendation:** Use one term for each concept.

### N26. Pool and test parameter details - Severity: Nit
* **Reviewer Lens:** `egm-codebase-details`
* **Affected Scope:** Lines 1112, 1341
* **Discrepancy Description:** The tests do not say which `KAIROS_ENV` value they run with. This is listed separately from M1 because the seed's production guard depends on it.
* **Rationale & Recommendation:** State the value the tests use.

### N27. "Fonts installed locally" is ambiguous - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** Line 1163
* **Discrepancy Description:** "Installed locally" could mean installed on the host or bundled with the app.
* **Rationale & Recommendation:** Say "bundled from npm packages (for example `@fontsource/*`), not loaded from a CDN".
