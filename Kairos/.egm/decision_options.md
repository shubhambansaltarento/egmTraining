# Kairos: Technical Plan decision options (Q1 to Q9)

Each question below has more than one defensible answer. The options are not
ranked. The option marked **[starting answer]** is the one proposed during
brainstorming. That label records where the option came from. It is not a
recommendation.

Some answers depend on each other. Q3 option A creates a per-day totals table
that Q4 option A reuses. Q1 option A changes what that table must contain.

--------------------------------------------------------------------------------

## Q1. Designated holiday entries (PRD 10.1, HOL-4, HOL-5)

### Option A: one stored row per employee

*   **Mechanism:** When HR adds a holiday, one transaction inserts a
    `time_entries` row of kind `holiday` (8 hours) for every active employee
    whose date of joining is on or before the holiday. Moving or deleting the
    holiday updates or deletes those rows. Creating or reactivating an
    employee must also insert rows for that year's later holidays.
*   **Cost when HR adds a holiday:** about 5,000 inserts and 5,000 audit rows
    in one transaction, roughly 0.3 to 1 second. Reading a week costs nothing
    extra, because the holiday is just another entry.
*   **Fails when:** a path that creates or reactivates employees forgets the
    backfill, which leaves that person with no holiday. A holiday that pushes a
    day past 24 hours needs the same rule as in option B. Every holiday change
    rewrites 5,000 rows, which can hold up concurrent saves on that date
    during the Friday peak.

### Option B: worked out when a week is read [starting answer]

*   **Mechanism:** Holidays live only in a `holidays` table (about 15 rows a
    year). Reading a week runs two indexed queries: one for the employee's
    entries in the 7 days, and one for holidays in the 7 days on or after their
    date of joining. The service merges the results. Every reader uses one
    shared database view, `employee_day_hours`: the weekly view, reports,
    exports, the 24-hour check and copy-week (TIME-13).
*   **Cost when HR adds a holiday:** one row. Reading a week adds one lookup
    on a very small table, well under 1 ms.
*   **Fails when:** a new reader adds up `time_entries` directly instead of
    using the shared view, so its totals leave out holidays and disagree with
    the weekly view. Holiday changes in an unlocked past month change past
    completeness without any trace. HOL-5 asks for this, but HR may be
    surprised by it.

--------------------------------------------------------------------------------

## Q2. Session and revocation within 1 minute (PRD 10.2, AUTH-5, AUTH-6, NFR-REL-2)

### Option A: random session token checked in the database on every request [starting answer]

*   **Mechanism:** At sign-in the server creates a random 256-bit token. It
    stores a SHA-256 hash of the token in a `sessions` table with the
    employee, the creation time and an expiry 8 hours later. The browser holds
    the token in a cookie marked `HttpOnly`, `Secure` and `SameSite=Strict`.
    Every request makes one primary-key lookup, joined to `employees.active`.
    Deactivation sets `active = false` and deletes that employee's sessions,
    so their next request is refused. CSRF protection comes from
    `SameSite=Strict` plus a required custom request header.
*   **Trade-offs:** about 0.2 to 0.5 ms of database time per request. At 100
    requests per second this is negligible. Revocation takes effect at once,
    well inside the 1-minute limit.
*   **Fails when:** the database is slow, because then every request is slower
    (but every request already needs the database anyway). Expired session
    rows build up and need a periodic delete.

### Option B: short-lived signed token plus a refresh token

*   **Mechanism:** An access token signed by the server (a JWT) is valid for 60
    seconds and is checked without the database. A refresh token is stored in
    the database and valid for 8 hours. Every 60 seconds the browser refreshes,
    and the refresh step checks `employees.active`. Deactivation deletes the
    refresh tokens.
*   **Trade-offs:** there is no database read for authentication on normal
    requests. In exchange, there are about 8 extra refresh calls per second at
    500 users, logic for rotating refresh tokens, retry handling in the
    frontend, and management of the signing key.
*   **Fails when:** the token lifetime is set above 60 seconds, which breaks
    AUTH-6. Clocks drifting between instances can also break it. Signing out
    does not end the access token until it expires.

--------------------------------------------------------------------------------

## Q3. 24-hour limit under concurrent saves (PRD 10.3, TIME-6, NFR-DATA-1)

Every option includes that day's holiday hours in the check. HR is never
blocked from adding a holiday (see The Problem), so holiday writes do not need
to take the lock.

### Option A: a per-day totals row that the save locks [starting answer]

*   **Mechanism:** A `day_totals` table holds one row per employee per date:
    `(employee_id, entry_date, work_leave_minutes, version)`. A save first
    makes sure the row exists (`INSERT ... ON CONFLICT DO NOTHING`), then
    reads it with `SELECT ... FOR UPDATE`. It checks that the existing total,
    plus holiday minutes, plus the new duration, is at most 1,440 minutes.
    Then it writes the entry and updates the total, all in one transaction. A
    second concurrent save waits for the first and then sees the new total.
    The missing-time report reads this table (see Q4 option A).
*   **Trade-offs:** one extra row write per save. Saves to the same day are
    handled one after another, which only affects the same employee and date.
*   **Fails when:** some code changes `time_entries` without updating
    `day_totals`, so the totals drift away from the real entries. This can be
    guarded against by having a database trigger keep the totals, or by a
    nightly check that compares them.

### Option B: a transaction-level advisory lock and a fresh sum

*   **Mechanism:** A save calls `pg_advisory_xact_lock` with a key built from
    the employee and the date, adds up that day's entries, checks the limit,
    then inserts. The lock is released when the transaction ends. No extra
    table is needed.
*   **Trade-offs:** there is no extra table to keep in step. Each save runs an
    index scan to add up the day, which is usually a handful of rows.
*   **Fails when:** a new code path writes entries without taking the lock.
    The rule then breaks with no error, and nothing in the database catches
    it.

### Option C: SERIALIZABLE transactions with a retry

*   **Mechanism:** Entry writes run at the SERIALIZABLE isolation level. They
    add up the day, check the limit and insert. If PostgreSQL detects a
    conflict, the transaction fails with a serialization error and the
    service retries it up to 3 times.
*   **Trade-offs:** there are no locks to remember. However, PostgreSQL can
    report conflicts that aren't real ones, and those cause extra retries
    during the Friday peak.
*   **Fails when:** a write path does not retry, so users see errors, which
    counts against the 0.1% error budget in NFR-PERF-5. Another failure is if
    one write path runs at a lower isolation level, which quietly turns the
    protection off.

--------------------------------------------------------------------------------

## Q4. Indexes, partitioning and pre-computed totals at 20 million rows (PRD 10.4)

Size estimate: 20 million entries at about 150 to 200 bytes each is 3 to 4 GB
of table plus about 0.6 to 1 GB of indexes. This mostly fits in the 16 GB
database memory.

### Option A: one composite index, per-day totals, no partitioning [starting answer]

*   **Mechanism:** `time_entries` gets a B-tree index on
    `(employee_id, entry_date)`. Reading a week touches about 10 to 20 index
    entries. The missing-time report (RPT-2) reads `day_totals` for one week,
    at most 35,000 rows (5,000 employees × 7 days) through an index on `entry_date`, and adds the holiday
    hours from the shared view. RPT-1 and RPT-3 read `time_entries` through a
    second index on `(entry_date, project_id)`.
*   **Trade-offs:** this depends on Q3 option A for `day_totals`. Without it,
    RPT-2 adds up about 50,000 to 70,000 raw entry rows for a week, which is
    still within the 2-second target.
*   **Fails when:** `day_totals` drifts away from the entries (see Q3 option
    A).

### Option B: range partitioning by month on `entry_date`

*   **Mechanism:** `time_entries` is split into one partition per month, with
    the same indexes in each. Reports covering a range of dates scan only the
    months they need. Old months can be detached later for the 7-year
    retention rule.
*   **Trade-offs:** partitions for future months must be created ahead of
    time, either by a scheduled job or by migrations. Unique constraints must
    include `entry_date`. Foreign keys that point at `time_entries` become
    awkward.
*   **Fails when:** no partition exists for a month yet, so inserts for that
    month fail. At 20 million rows, partitioning mostly helps with archiving,
    not with speed.

### Option C: indexes only, no pre-computed totals

*   **Mechanism:** a `(employee_id, entry_date)` index, plus an
    `(entry_date) INCLUDE (employee_id, duration_minutes, project_id)` index.
    The second index lets reports add up raw entries without reading the
    table. No `day_totals` table.
*   **Trade-offs:** this is the smallest schema. RPT-2 for one week adds up
    about 70,000 rows, roughly 50 to 150 ms. RPT-3 for one month adds up about
    330,000 rows, roughly 0.3 to 1 second.
*   **Fails when:** reports are run over ranges much longer than a month, or
    many HR users run them at the same moment at month end.

--------------------------------------------------------------------------------

## Q5. CSV exports without slowing interactive requests (PRD 10.5, NFR-PERF-4, RPT-5)

### Option A: stream the file in the request, with a cap on concurrent exports [starting answer]

*   **Mechanism:** The export endpoint sends the response as it is produced
    (FastAPI `StreamingResponse`). It reads from a server-side database cursor
    in batches of 2,000 rows. Exports use their own pool of 2 database
    connections, with a 30-second statement timeout. Each instance runs at
    most 1 export at a time and returns 429 with a "try again shortly" message
    when busy. Cells that begin with `=`, `+`, `-` or `@` are prefixed so that
    spreadsheet programs cannot run them as formulas (CSV injection).
*   **Trade-offs:** there is no extra process and no file storage. Turning
    100,000 rows into CSV in Python takes about 0.5 to 1 second of CPU on an
    instance that is also serving interactive requests.
*   **Fails when:** the proxy's timeout is shorter than the export takes.
    Another failure is if month-end exports from several HR users push the
    CPU on both 2-vCPU instances high enough to break the 100 ms p99 target.

### Option B: a background job table and a worker process

*   **Mechanism:** The request inserts a row into an `export_jobs` table. A
    worker process (same image, started with a different command) claims jobs
    with `SELECT ... FOR UPDATE SKIP LOCKED`. It writes the CSV into the
    database as a large object. The UI checks the job's progress and offers a
    download link when it is done.
*   **Trade-offs:** CPU use is completely separate from the API processes. The
    cost is an extra service in the compose file, a place to keep finished
    files, a cleanup job, and a progress screen in the UI.
*   **Fails when:** the worker is down, so exports wait forever without an
    alert. Another failure is if the reference environment, which has only 2
    application instances, gives the worker no room of its own.

### Option C: the database's COPY command streamed through

*   **Mechanism:** The endpoint runs `COPY (SELECT ...) TO STDOUT WITH CSV`
    and passes the bytes straight to the client. PostgreSQL formats the CSV.
*   **Trade-offs:** this uses the least Python CPU and is the fastest.
    Escaping cells against CSV injection has to be written in SQL, and column
    formatting rules live in SQL instead of Python.
*   **Fails when:** the SQL escaping misses a case, which leaves a CSV
    injection hole. Another failure is if the query plan for COPY gets worse
    as data grows, because COPY gives no chance to work in batches.

--------------------------------------------------------------------------------

## Q6. Lost-update protection (PRD 10.6, TIME-17)

### Option A: an integer version number [starting answer]

*   **Mechanism:** Each entry has `version INTEGER`, starting at 1. Reads
    return it (also as an `ETag` header). Edit and delete requests must send
    it. The update runs `UPDATE ... SET ..., version = version + 1 WHERE id =
    :id AND version = :v`. If no row is updated, the API returns 409 with
    `code: "ENTRY_CHANGED"`, and `details` holds the current entry. A request
    with no version is rejected with 428. The UI says: "This entry was changed
    in another tab. Your edit was not saved." It offers "Show latest" and
    "Apply my changes again".
*   **Fails when:** some client or endpoint updates entries without a version
    check, for example a future bulk tool.

### Option B: the last-updated timestamp as the version

*   **Mechanism:** the same flow, but comparing `updated_at` instead of a
    counter.
*   **Trade-offs:** no extra column.
*   **Fails when:** two updates land within the same clock tick, or the
    timestamp loses precision when it is converted to or from JSON, so a
    conflict is missed or reported when there isn't one.

--------------------------------------------------------------------------------

## Q7. Server cache (PRD 10.7, NFR-PERF-2)

### Option A: no server cache [starting answer]

*   **Mechanism:** Rely on indexes, plus HTTP caching headers (`ETag`) for
    read-mostly lists such as holidays and tasks. The frontend uses a query
    library (TanStack Query) that keeps recent results in the browser and
    reloads them when the screen opens. Add a server cache only if the Slice 5
    load test shows a specific endpoint missing its target.
*   **Fails when:** the load test shows the project-selector query, or the
    week-read query, above 100 ms at p99. A cache then has to be added late.

### Option B: an in-memory cache in each instance, cleared by database notifications

*   **Mechanism:** Each instance keeps holidays, tasks and common projects in
    memory. When HR changes them, the write also runs `NOTIFY kairos_cache`.
    Each instance keeps a `LISTEN` connection open and clears its cache when a
    notification arrives. A 5-minute expiry is the fallback.
*   **Trade-offs:** this saves one small query per request for data that
    rarely changes. It adds a long-lived connection per instance and code to
    reconnect it.
*   **Fails when:** the `LISTEN` connection drops without anyone noticing, so
    an instance serves stale holidays for up to 5 minutes. That breaks HOL-5
    ("reflect the change the next time the view is loaded").

(A shared Redis cache is excluded because the brainstorm decided Postgres is
the only shared state.)

--------------------------------------------------------------------------------

## Q8. An audit trail that cannot be skipped (PRD 10.8, AUD-1, AUD-2, AUD-3)

### Option A: database triggers with a required actor [starting answer]

*   **Mechanism:** Each audited table (employees, holidays, projects,
    assignments, period locks, time entries) has a row-level trigger that runs
    after each insert, update or delete. It writes an `audit_log` row with the
    table, the row ID, the action, the actor, the UTC time, and the old and new
    values as JSON (for time entries only: actor and time). The request's
    database session runs `SET LOCAL kairos.actor_id = '<id>'` at the start of
    each transaction. The trigger raises an error if the actor is missing, so
    a write without an actor fails instead of going unaudited. The
    application's database user can only INSERT into `audit_log`, never
    UPDATE or DELETE it. Migrations and seeding set the actor to `system`.
*   **Trade-offs:** no application code path can skip the audit, including raw
    SQL and bulk updates. The cost is that audit logic lives in PL/pgSQL and
    is tested through the database. AUD-2 adds about one audit row per entry
    change, which over 5 years is 20 million or more rows.
*   **Fails when:** someone adds a new audited table without its trigger. A
    migration test that checks every audited table has the trigger closes
    this gap.

### Option B: ORM session events

*   **Mechanism:** SQLAlchemy `before_flush` and `after_flush` hooks inspect
    the objects being created, changed or deleted, and write audit rows in the
    same transaction.
*   **Trade-offs:** all the logic is in Python and easy to unit-test.
*   **Fails when:** code uses bulk `update()` or `delete()` statements, or raw
    SQL. These skip the ORM events, so the audit is silently skipped. That is
    exactly the risk PRD 10.8 names.

### Option C: explicit calls from a base repository class

*   **Mechanism:** Every repository inherits from a base class whose
    `create`, `update` and `delete` methods write the audit row.
*   **Fails when:** a developer writes a repository method that does not go
    through the base class. Nothing stops it.

--------------------------------------------------------------------------------

## Q9. Libraries and tools left open by PRD section 7.1

These have less impact than Q1 to Q8. Each row lists the starting answer and
the main alternative.

| Area | Starting answer | Alternative | What decides it |
|---|---|---|---|
| Database access | SQLAlchemy 2.0, async, with the `asyncpg` driver | SQLAlchemy 2.0, sync, with `psycopg` 3 in FastAPI's thread pool | Async avoids the default 40-thread limit under load. Sync is simpler to debug and test. |
| Migrations | Alembic | (none: Alembic is the standard tool with SQLAlchemy) | NFR-DATA-2 |
| Password hashing | argon2id (`argon2-cffi`, 3 passes, 64 MiB, 1 lane, about 50 to 80 ms each) | bcrypt with cost 12 (about 250 ms each) | NFR-PERF-3: 500 sign-ins in 60 s on 4 vCPU in total. argon2id leaves more room. |
| Frontend server data | TanStack Query | Redux Toolkit Query | TanStack Query has less boilerplate. The app has almost no client-only state. |
| UI components | MUI | shadcn/ui (Radix with Tailwind) | MUI has a ready date picker and data grid. shadcn/ui gives full control over the colours in design-theme.md. |
| Frontend build | Vite | (none: the standard for React 18 with TypeScript) | |
| HTTPS proxy | Caddy (creates local HTTPS certificates automatically) | nginx with a self-signed certificate | Caddy needs less configuration for NFR-SEC-4 locally. |
| Testing | pytest with a real PostgreSQL started by Podman. Vitest and Testing Library for the frontend. Playwright for scenario tests. | SQLite for backend tests | SQLite cannot test row locks, triggers or advisory locks, so it cannot test Q3 or Q8. |
