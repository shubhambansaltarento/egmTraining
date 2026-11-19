# Kairos: Employee Time Tracking

*   **Author:** Shubham Bansal
*   **Status:** Draft
*   **Date:** 2026-11-19

--------------------------------------------------------------------------------

## Overview

Kairos is a web application in which about 5,000 employees report how they
spent their working hours, and about 10 HR administrators maintain employees,
projects, project assignments and holidays and check that everyone has reported
their time. It replaces the current process of spreadsheets and email
reminders.

This document is the design for the whole application as described in
[prd.md](../prd.md) version 1.3. It has two levels of detail:

*   The **Technical Plan** covers the architecture for PRD Slices 1 to 5. It
    states a decision and a reason for each of the eight questions in PRD
    section 10, a data dictionary, the API contracts, and sequence diagrams for
    scenarios S-1, S-2 and S-5.
*   The **Detailed Implementation Plan** covers PRD Slice 1 only (sign in and
    log time). It lists every file to create and why. Slices 2 to 5 are listed
    as future work and will get their own implementation plans.

The technology stack is fixed by PRD section 7.1: PostgreSQL 16, Python 3.12
with FastAPI, React 18 with TypeScript, identity built into the application,
OpenAPI 3.1, Podman with Podman Compose, and k6 or Locust for load tests. The
files in [context/](../context/) (design theme and engineering standards) are
treated as constraints.

## The Problem

Every employee must report their working hours each day. The data feeds client
billing, project costing, utilisation reports and statutory compliance. Today
it is collected with spreadsheets and email reminders, which causes three
problems:

1.  **Data arrives late and incomplete.** Nobody can see who has not reported
    until HR checks by hand. HR spends several days a month chasing missing
    entries every Monday and at month end.
2.  **Data is inconsistent.** Employees type project names freely, report
    against projects they do not work on, and record holidays and leave in
    different ways. HR corrects holiday and leave data by hand.
3.  **Data cannot be audited.** There is no record of who changed what, and no
    way to close a month so that its figures stop changing.

Kairos solves these by giving employees one weekly screen in which they can
only choose projects HR has opened to them, by adding designated holidays to
every employee's week automatically, by showing HR in one screen who has an
incomplete week, and by recording every change to master data.

The PRD leaves some business rules open or in conflict. This design settles
them as follows (the author accepted these as defaults during brainstorming):

*   **A holiday added on a day that is already full.** HR is never blocked from
    adding a holiday. If an employee's day goes over 24 hours as a result, the
    existing entries stay and the day is flagged. The 24-hour limit (TIME-6) is
    checked only when an employee saves an entry. This follows the same
    principle as PRJ-10: HR actions never change existing entries.
*   **Leave on a designated holiday** is rejected, because the holiday already
    covers the day.
*   **Partial weeks.** Expected hours count only weekdays on or after the
    employee's date of joining, and holidays before that date are not shown.
    Without this rule every new joiner appears as short in the missing-time
    report (RPT-2).
*   **Unlocking a month older than 14 days** lets HR change holidays in that
    month (HOL-2). It does not let anyone change time entries, because TIME-9
    still applies to employees and section 3.3 gives HR read-only access to
    other employees' entries.

## Goals

Product goals, taken from PRD section 2.1:

*   **G-1:** An employee can log a full 8-hour day in under 30 seconds, and
    with the keyboard alone in no more than 6 key presses after the add form
    opens (NFR-UX-5).
*   **G-2:** HR needs no spreadsheet for any master-data task.
*   **G-3:** HR can list every employee with an incomplete week in one screen.
*   **G-4:** The performance targets in PRD section 7.2 are met at 500
    concurrent users with 20 million time entries loaded, on the reference
    environment of 2 application instances and 1 database instance.

Design goals for this document:

*   State a decision and a reason for each of the eight questions in PRD
    section 10.
*   Make every requirement in Slice 1 traceable from this document to a file
    and to at least one test (PRD section 11).
*   Choose Slice 1's tables and API shapes so that Slices 2 to 5 add tables
    and endpoints but do not need to rewrite Slice 1's tables. In particular,
    the 24-hour check must already allow for holiday hours (Slice 3), entries
    must already carry a version number for lost-update protection (TIME-17),
    and writes must already pass the information the audit trail needs
    (Slice 4).
*   Keep the application tier stateless (NFR-REL-2) with PostgreSQL as the only
    place that holds shared state.

## Non-Goals

Out of scope for the product (PRD section 2.2):

*   Manager approval workflows.
*   Leave balances, accrual and approval. Leave is only recorded as an entry.
*   Payroll, invoicing, billing rates and expenses.
*   Single sign-on, social login and multi-factor authentication.
*   Native mobile apps. The web app must work in a mobile browser.
*   Multi-tenancy, multiple time zones and multiple languages. Everything is in
    IST (UTC+05:30) and English.
*   Integrations with Jira, HRMS or calendar systems.

Out of scope for this design:

*   **Detailed implementation of Slices 2 to 5.** Their architecture is
    covered in the Technical Plan, but their files are listed only as future
    work.
*   **S and C requirements other than TIME-17, AUD-2, RPT-5 and RPT-6.** Those
    four are designed because PRD section 10 questions 5, 6 and 8 depend on
    them. Each of the others gets one line saying where it would fit.
*   **Extra infrastructure.** No Redis, message broker, job queue service or
    read replica. The reference environment in NFR-PERF-7 has none, and nothing
    at about 100 requests per second requires them yet.
*   **Keycloak.** Identity is built into the application, as PRD section 7.1
    states.
*   **A production deployment.** The design covers local start with Podman and
    a run on the reference environment for the load test. HTTPS is provided by
    a reverse proxy in the compose file so that NFR-SEC-4 can be shown, but
    there is no cloud deployment or work toward the 99.5% availability target
    beyond liveness and readiness checks.
*   **Docker tooling.** Docker is not permitted. The design uses
    `podman-compose` (the separate Python tool), because `podman compose` can
    hand work to the `docker-compose` binary.

## Open Questions

The eight PRD section 10 questions and the open library choices (Q1 to Q9)
were decided by the author on 2026-11-19. The options and trade-offs are in
[decision_options.md](decision_options.md), and the decisions are in the
Technical Plan below. These questions remain open:

*   **How to count "6 key interactions" (NFR-UX-5).** This design counts
    typing a search string and pressing Enter to pick a result as one
    interaction. If each character counts separately, the target cannot be met
    with any project selector. This needs confirmation before the Slice 1 test
    for NFR-UX-5 is written.
*   **Department as free text or a managed list (EMP-1, EMP-5, RPT-2).** Free
    text lets "Engineering" and "engineering " become two departments in the
    filters. This needs a decision before Slice 2. Slice 1 stores it as text
    from the seed.

## Critical User Journeys (CUJs)

The PRD's scenarios S-1 to S-12 are the acceptance tests. The journeys below
are the ones Slice 1 must pass. Their tests create their own data and do not
depend on the seed (PRD section 8).

### CUJ 1: Log a normal day (S-1: TIME-1, TIME-3, TIME-5, TIME-16)

*   **Given**: Priya is signed in and assigned to "Acme Health – Dev Pod".
*   **When**: She adds an entry for today with that project, task
    "Development (Coding)", work location Office and duration 8h 0m.
*   **Then**: The entry appears in today's column showing its task, project,
    duration and work location. Today's total shows 8h 0m and the week total
    goes up by 8 hours.

### CUJ 2: Daily limit (S-4: TIME-6)

*   **Given**: Priya has 20 hours of entries on one day.
*   **When**: She tries to add a 5-hour entry on that day.
*   **Then**: The entry is rejected with a message stating the 24-hour daily
    limit, and nothing is saved. The same holds when two saves for that day
    arrive at the same moment (NFR-DATA-1).

### CUJ 3: Unassigned project (S-5: PRJ-9, TIME-15)

*   **Given**: Priya is not assigned to project "Borealis".
*   **When**: She opens the project selector, and separately a direct API
    request tries to create an entry against "Borealis".
*   **Then**: "Borealis" is not offered, and the API request is rejected as
    forbidden.

### CUJ 4: Common projects only (S-12: PRJ-4)

*   **Given**: An employee with no assignments.
*   **When**: They open the project selector.
*   **Then**: Internal, Training, Recruitment and Pre-sales are offered and no
    client project is.

### CUJ 5: Edit, delete and move between weeks (TIME-2, TIME-7, TIME-14)

*   **Given**: Priya has two entries today with the same project and task.
*   **When**: She edits one, deletes the other, moves to the previous week and
    back, and jumps to a date.
*   **Then**: Both entries were accepted when created, the edit and delete are
    reflected in the day and week totals, and the header shows the correct date
    range and ISO week number for each week shown.

### CUJ 6: Sign in and sign out (AUTH-1, AUTH-5)

*   **Given**: A seeded employee with known credentials.
*   **When**: They sign in with work email and password, then sign out, and
    separately a session is used more than 8 hours after sign-in.
*   **Then**: Sign-in opens the weekly view for the current week. After sign-out
    or after 8 hours, requests with that session are refused and the user is
    sent to the sign-in screen.

Scenarios belonging to later slices: S-2, S-3 and S-8 to S-9 (Slices 3 and 4),
S-6, S-7, S-10 and S-11 (Slice 2). S-11's rule that an ended assignment stops
new entries is enforced in Slice 1 through PRJ-9, but the HR action that ends
an assignment arrives in Slice 2.

## Technical Plan (Architecture)

### 1. Runtime components

```text
 Browser (React SPA)
     │  HTTPS, cookie __Host-kairos_session, headers X-Request-ID, X-Kairos-CSRF
     ▼
 ┌──────────────┐   /            static files (built React app)
 │ proxy (Caddy) │───────────────────────────────────────────────┐
 └──────┬───────┘   /api/*       reverse proxy, round robin       │
        ▼                                                         │
 ┌──────────────┐  ┌──────────────┐                               │
 │ api #1       │  │ api #2       │  FastAPI + uvicorn, stateless │
 │ (2 vCPU 4GB) │  │ (2 vCPU 4GB) │  (one instance locally)       │
 └──────┬───────┘  └──────┬───────┘                               │
        └────────┬────────┘                                       │
                 ▼                                                │
        ┌──────────────────┐   the only shared state:             │
        │ db (PostgreSQL 16)│  sessions, data, locks, audit        │
        └──────────────────┘                                      │
```

*   **proxy:** Caddy serves the built frontend and forwards `/api/*` to the
    API instances. Locally it issues its own HTTPS certificate, so the
    `__Host-` cookie and NFR-SEC-4 work the same way locally as when deployed.
    Its image is built in two stages: Node builds the frontend, then the
    output is copied into the Caddy image.
*   **api:** a FastAPI application with no state in memory between requests.
    It keeps no sessions, caches or files. Any instance can serve any request,
    and instances can be restarted without signing anyone out (NFR-REL-2).
*   **db:** PostgreSQL 16. It holds all data and also does the work that
    usually needs extra infrastructure: session lookup (Q2), row locks for the
    24-hour rule (Q3), and the audit triggers (Q8).
*   **Start-up (NFR-OPS-1):** `make up` runs these steps in order. It starts
    `db` and waits until `pg_isready` succeeds. It runs `alembic upgrade head`
    and then the seed script as one-off containers (`podman-compose run
    --rm`). Then it starts `api` and `proxy`. The order lives in the Makefile
    instead of compose `depends_on` conditions, because `podman-compose`
    handles those conditions inconsistently across versions. Docker is never
    invoked.

### 2. Backend structure

The layers follow [engineering-standards.md](../context/engineering-standards.md):
the **router** checks input against Pydantic schemas and calls a service. The
**service** holds every business rule. The **repository** holds every SQL
statement. Routers contain no rules and repositories make no decisions.

*   **Database access:** SQLAlchemy 2.0 in async mode with the `asyncpg`
    driver. Each API instance has a pool of 20 connections plus up to 10 extra
    when busy. Every request that writes runs in one transaction.
*   **Migrations:** Alembic, with hand-written migrations (NFR-DATA-2).
    Triggers, functions and constraints are part of the migrations, not
    created at start-up.
*   **Request pipeline**, in this order for every API request:
    1.  *Correlation ID:* read `X-Request-ID` from the request, or create a
        UUID if it is missing. Put it on every log line and echo it in the
        response. The repository layer adds it as a SQL comment
        (`/* request_id=... */`), so PostgreSQL logs can be matched to the
        request (NFR-OPS-3).
    2.  *Authentication:* read the session cookie and look up the session
        (see Q2). Public endpoints (sign-in and health) skip this step.
    3.  *CSRF check:* for POST, PUT, PATCH and DELETE, require the header
        `X-Kairos-CSRF: 1`. Browsers do not let another site add custom
        headers without a CORS preflight, and the API allows no cross-origin
        requests. Together with `SameSite=Strict`, this blocks cross-site
        request forgery (NFR-SEC-5).
    4.  *Role check:* each router declares the role it needs. HR-only routers
        (Slice 2 onward) refuse an Employee with 403 before any service code
        runs (NFR-SEC-2, S-6).
    5.  *Actor:* each write transaction runs
        `SELECT set_config('kairos.actor_id', '<employee id>', true)`. This
        happens from Slice 1, so the audit triggers added in Slice 4 have an
        actor from the first day.
*   **"Own data" paths:** an employee's own data is reached only through
    `/api/v1/me/...` paths. The employee ID comes from the session, never from
    the request. A time entry ID in a `/me` path is always looked up together
    with the session's employee ID, so another employee's entry returns 404,
    the same as an entry that doesn't exist (NFR-SEC-3). HR's read-only access
    to other employees uses separate `/api/v1/employees/{id}/...` paths from
    Slice 4.
*   **"Today" and dates:** the service works out "today" in `Asia/Kolkata`.
    All dates are PostgreSQL `DATE` values. Timestamps are `TIMESTAMPTZ`
    stored in UTC (NFR-DATA-3).
*   **Errors:** every error response has the body `{ "code", "message",
    "details" }`. `code` is a fixed upper-case identifier (listed in section 6).
    `message` is a short sentence that says what went wrong and what to do
    next (NFR-UX-4), and the UI shows it as written. `details` holds
    field-level validation problems or the values behind a rule failure.
    Unexpected errors return `INTERNAL_ERROR` with the correlation ID, and the
    stack trace goes only to the log.
*   **Logs:** JSON lines on standard output. A filter removes the `password`,
    `token` and `cookie` fields and the `Authorization` and `Cookie` headers
    before writing (NFR-SEC-1).
*   **Configuration:** pydantic-settings reads environment variables. Local
    values come from a `.env` file that is created from `.env.example` and
    listed in `.gitignore` (NFR-SEC-8).

### 3. Frontend structure

*   React 18 and TypeScript, built with Vite. React Router handles the
    `/sign-in` and `/week/:isoWeek` routes (for example `/week/2026-W47`).
    `/` redirects to the current week (TIME-1).
*   **Server data:** TanStack Query. The weekly view is one query keyed by the
    ISO week. Saving, editing or deleting an entry reloads that week's query,
    so the day and week totals always come from the server. The browser never
    adds up totals on its own.
*   **API access:** components never call `fetch`. They call functions in
    `src/api/client.ts`. That module adds `X-Request-ID`, `X-Kairos-CSRF` and
    `If-Match`, parses the error shape, and on a 401 sends the user to
    `/sign-in`. TypeScript types for requests and responses are generated from
    the backend's OpenAPI document with `openapi-typescript`, so the frontend
    and the API cannot drift apart unnoticed.
*   **Components:** MUI, with a theme built from
    [design-theme.md](../context/design-theme.md). The top bar is `#111111`,
    the primary action `#1F2A44`, and work, leave, holiday and warning use
    `#00A67E`, `#B45309`, `#5B21B6` and `#B91C1C`. The font is Inter at 14 px,
    with a condensed bold face for hours. Every entry card shows a text label
    and an icon for its kind, so colour is never the only signal (TIME-11,
    NFR-UX-3). The add and edit form opens in a centred modal, and Save is its
    only primary button.
*   **Layout:** seven equal columns from Monday to Sunday at 768 px and above.
    Below 768 px the days stack vertically, down to 360 px (NFR-UX-2).
*   **Keyboard path (NFR-UX-5):** the form opens with Project focused. Date
    defaults to the column it was opened from, and Location defaults to the
    one used last. The path is: (1) type part of a code or name and press
    Enter to pick, which moves focus to Task, (2) type and press Enter, which
    moves focus to Duration, (3) type a duration such as `8` or `7:45`, then
    (4) press Ctrl+Enter or Cmd+Enter to save. That is 4 interactions, within
    the limit of 6 under the counting rule listed in Open Questions.
*   **Duration input (TIME-5):** accepts `8`, `7.5`, `7:45` and `7h45m`.
    Anything that is not a whole multiple of 15 minutes, or is below 15
    minutes, is rejected in the form with the same message the API returns.

### 4. Decisions on PRD section 10 (Q1 to Q8) and open libraries (Q9)

All decided by the author on 2026-11-19. The options that were turned down are
in [decision_options.md](decision_options.md) and are summarised
under Alternatives Considered.

**Q1. Holiday entries are worked out when a week is read.** Holidays are
stored once in `holidays`. Two database functions, created in the Slice 1
migration, are the only places that turn holidays into hours:

*   `employee_holidays(employee_id, from_date, to_date)` returns each holiday
    in the range that falls on or after the employee's date of joining. It
    gives 480 minutes for a weekday holiday and 0 for a Saturday or Sunday
    holiday (HOL-6).
*   `employee_day_hours(from_date, to_date)` returns one row per active
    employee per date: expected minutes (480 on a weekday on or after the date
    of joining, otherwise 0), entry minutes (from `day_totals`), and holiday
    minutes.

The week read, the 24-hour check, copy-week, RPT-1, RPT-2, RPT-3 and the
exports all call these functions. A test in each slice checks that no
repository adds up `time_entries` hours for a report without going through
them. **Cost:** adding a holiday writes one row. Reading a week adds one
indexed lookup on a table of about 15 rows a year. A weekend holiday returns
0 minutes but is still shown in the calendar (HOL-6). Work logged on a
holiday is added on top of the holiday hours (HOL-7).

**Q2. Random session token checked in the database on every request.**

*   Sign-in creates 32 random bytes, stores their SHA-256 hash in `sessions`
    with `expires_at = created_at + 8 hours`, and sends the token in the
    cookie `__Host-kairos_session` (`HttpOnly; Secure; SameSite=Strict;
    Path=/`). The expiry never moves, so a session ends 8 hours after sign-in
    whatever the activity (AUTH-5).
*   Every authenticated request runs one query: the session joined to its
    employee, where the token hash matches, `expires_at > now()` and the
    employee is active. If no row comes back, the response is 401
    `SESSION_ENDED`.
*   Sign-out deletes the session row. Deactivation (Slice 2) sets
    `is_active = false` and deletes all of that employee's sessions in the
    same transaction, so their very next request fails. That is well within
    the 1-minute limit of AUTH-6.
*   Sign-in deletes that employee's expired sessions. A daily cleanup that
    runs from the API at start-up and every 24 hours, under an advisory lock
    so only one instance does it, deletes any others.
*   **Password hashing:** argon2id via `argon2-cffi`, with 3 passes, 64 MiB
    and 1 lane. Hashing runs in a worker thread so it doesn't block other
    requests. At most 4 hashes run at once per instance, which caps memory at
    256 MiB. The planned load of 500 sign-ins in 60 seconds is about 8 a
    second across 4 vCPU (NFR-PERF-3).

**Q3. A per-day totals row, locked by the save.** `day_totals` holds one row
per employee and date with that day's total work and leave minutes. A
database trigger on `time_entries` keeps the total up to date, so no code path
can change entries without changing the total. Creating or editing an entry
runs in one transaction:

1.  `INSERT INTO day_totals (employee_id, entry_date) VALUES (...) ON CONFLICT
    DO NOTHING` makes sure the row exists.
2.  `SELECT entry_minutes FROM day_totals WHERE ... FOR UPDATE` locks it. A
    concurrent save for the same employee and date waits here.
3.  The service adds the holiday minutes from `employee_holidays` and the new
    duration, subtracting the old duration if this is an edit. If the result
    is over 1,440 minutes, it rolls back with 409 `DAILY_LIMIT_EXCEEDED`.
4.  It inserts or updates the entry. The trigger adds the change to
    `day_totals`. Commit.

An edit that moves an entry to another date locks both days' rows, earliest
date first, so two such edits cannot deadlock. A `CHECK (entry_minutes BETWEEN
0 AND 1440)` on `day_totals` is a last guard if the service check is ever
skipped. HR never takes this lock when adding a holiday (see The Problem).

**Q4. One composite index, per-day totals, no partitioning.**

| Table | Index | Serves |
|---|---|---|
| `time_entries` | `(employee_id, entry_date)` | Reading a week (about 10 to 20 rows), edit and delete by owner, RPT-1 |
| `time_entries` | `(entry_date, project_id)` (added in Slice 4) | RPT-3 and its export |
| `day_totals` | primary key `(employee_id, entry_date)` | Q3 lock, week totals |
| `day_totals` | `(entry_date)` | RPT-2: at most 35,000 rows for a week |
| `assignments` | `(employee_id, start_date)` and the exclusion constraint below | Project selector (TIME-15), PRJ-9 check |
| `sessions` | primary key `token_hash`, plus `(employee_id)` | Every request, revocation |

No partitioning. At 20 million rows the table is about 3 to 4 GB and its
indexes about 1 GB, which mostly fits in 16 GB of memory. Every query reaches
a small range through an index. Partitioning would add maintenance work, and
whether it is needed will be decided by the Slice 5 load test, not now.

**Q5. CSV exports are streamed in the request, with a cap on concurrent
exports (Slice 4 and stretch).** Export endpoints return a streamed response
that reads a server-side cursor in batches of 2,000 rows. They use a separate
pool of 2 database connections and a 30-second statement timeout. Each
instance allows 1 export at a time and otherwise returns 429 `EXPORT_BUSY`
("Another export is running. Try again in a minute."). Cells that start with
`=`, `+`, `-` or `@` get a leading `'` so spreadsheets cannot run them as
formulas. Caddy's proxy timeout is set to 60 seconds. The Slice 5 load test
runs a 100,000-row export during the interactive load to show that the
NFR-PERF-2 targets still hold.

**Q6. An integer version number on each entry.** `time_entries.version`
starts at 1. Responses return it in the body and as `ETag: "<version>"`. PUT
and DELETE require `If-Match: "<version>"` and return 428
`VERSION_REQUIRED` without it. The update runs `... WHERE id = :id AND
employee_id = :me AND version = :v`. If no row changes and the entry still
exists, the response is 409 `ENTRY_CHANGED`, with the current entry in
`details`. The modal then shows "This entry was changed in another tab. Your
edit was not saved." with two actions, "Show latest" and "Apply my changes
again". The version check ships in Slice 1, because adding a required header
to PUT and DELETE later would break existing clients. This delivers TIME-17
early.

**Q7. No server cache.** Every request reads PostgreSQL. `GET /tasks` and
`GET /holidays` return an `ETag` so browsers can reuse them. TanStack Query
keeps the current and adjacent weeks in the browser and reloads them when the
screen opens, which meets HOL-5 ("next time the view is loaded"). The Slice 5
load test decides whether a cache is ever added. Any cache added then must say
how HOL-5 still holds.

**Q8. Database triggers with a required actor (Slice 4).** One PL/pgSQL
function, `audit_row()`, is attached as an `AFTER INSERT OR UPDATE OR DELETE
FOR EACH ROW` trigger to `employees`, `holidays`, `projects`, `assignments`,
`period_locks` and `time_entries`. It reads `current_setting('kairos.actor_id',
true)` and raises an error if it is empty, so a write with no actor fails
instead of going unaudited. It writes old and new values as JSON for master
data (AUD-1), and only actor and time for time entries (AUD-2). The
`kairos_app` database user that the API connects as has INSERT and SELECT on
`audit_log` but no UPDATE or DELETE (AUD-3). Migrations and seeding connect as
`kairos_owner` and set the actor to `system`. A test lists every table in the
audited set and fails if one is missing its trigger. `employee_preferences`,
`sessions` and `day_totals` are not audited: they hold working state, not
records.

**Q9. Libraries and tools.**

| Area | Choice |
|---|---|
| Database access | SQLAlchemy 2.0 async with `asyncpg` |
| Migrations | Alembic |
| Password hashing | argon2id (`argon2-cffi`), 3 passes, 64 MiB, 1 lane |
| Validation and settings | Pydantic v2, pydantic-settings |
| Python tooling | `uv` for dependencies, `ruff` for lint and format, `mypy --strict` for type checks, `pytest` with `pytest-asyncio` and `pytest-cov` |
| Frontend server data | TanStack Query v5 |
| UI components | MUI v6 |
| Frontend build and tests | Vite, Vitest with Testing Library, ESLint, `tsc --noEmit` |
| End-to-end tests | Playwright, run against the stack started by `make up` |
| HTTPS proxy | Caddy 2 |
| Load test (Slice 5) | k6 |
| Backend tests | A real PostgreSQL 16 started with Podman. Never SQLite, because it cannot test row locks or triggers. |

### 5. Data dictionary

Conventions: `id` columns are `BIGINT GENERATED ALWAYS AS IDENTITY` unless
stated. Every table has `created_at` and `updated_at` (`TIMESTAMPTZ`, UTC,
default `now()`). Enumerations are `TEXT` with a `CHECK` constraint, so adding
a value is a one-line migration. The "Slice" column says which slice's
migration creates the table. Slice 1 creates every table the 24-hour rule and
the week read depend on, even where it adds no screen for them.

**`employees`** (Slice 1. HR edits it from Slice 2.)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. Internal only, never shown. |
| `employee_code` | TEXT | The "employee ID" from EMP-1. Unique, never changes (EMP-2, EMP-3). 1 to 20 characters. |
| `full_name` | TEXT | Required, 1 to 200 characters. |
| `work_email` | TEXT | Required, stored in lower case, unique (EMP-2). `CHECK (work_email = lower(work_email))`. |
| `designation` | TEXT | Required, 1 to 100 characters. |
| `department` | TEXT | Required, 1 to 100 characters. See Open Questions. |
| `date_of_joining` | DATE | Required. Earliest date the employee can log time (EMP-7). Start of expected hours. |
| `role` | TEXT | `EMPLOYEE` or `HR_ADMIN`. |
| `is_active` | BOOLEAN | Default true. Deactivated employees are never deleted (EMP-4). |
| `password_hash` | TEXT | argon2id encoded string. Never logged or returned. |
| `must_change_password` | BOOLEAN | Default false. Used from Slice 2 (AUTH-2). |
| `failed_sign_in_count` | SMALLINT | Default 0. Used from Slice 2 (AUTH-4). |
| `locked_until` | TIMESTAMPTZ | Null unless locked. Used from Slice 2 (AUTH-4). |

**`sessions`** (Slice 1)

| Column | Type | Rules |
|---|---|---|
| `token_hash` | BYTEA | Primary key. SHA-256 of the cookie value. The cookie value itself is never stored. |
| `employee_id` | BIGINT | Foreign key to `employees`. Indexed. |
| `expires_at` | TIMESTAMPTZ | `created_at + 8 hours`. Never extended. |

**`employee_preferences`** (Slice 1)

| Column | Type | Rules |
|---|---|---|
| `employee_id` | BIGINT | Primary key, foreign key to `employees`. |
| `last_work_location` | TEXT | `OFFICE`, `CLIENT_SITE` or `WFH`. Updated on each work entry save. Default for the next form (TIME-3). Kept out of `employees` so that saving an entry does not write an audited employee row. |

**`projects`** (Slice 1, created by the seed. HR edits it from Slice 2.)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `code` | INTEGER | Unique. Set from the sequence `project_code_seq` (starting at 1001) and never changed (PRJ-1). Shown as `[code] name`. |
| `name` | TEXT | Required, 1 to 200 characters. |
| `client_name` | TEXT | Required for `CLIENT` projects, null for `COMMON` projects. |
| `type` | TEXT | `CLIENT` or `COMMON`. |
| `start_date` | DATE | Required. |
| `end_date` | DATE | Optional. `CHECK (end_date IS NULL OR end_date >= start_date)`. |
| `status` | TEXT | `ACTIVE` or `CLOSED` (PRJ-2). Projects are never deleted. |

**`assignments`** (Slice 1, created by the seed. HR edits it from Slice 2.)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `employee_id` | BIGINT | Foreign key to `employees`. |
| `project_id` | BIGINT | Foreign key to `projects`. Must be a `CLIENT` project (checked by the service). |
| `start_date` | DATE | Required. |
| `end_date` | DATE | Optional. `CHECK (end_date IS NULL OR end_date >= start_date)`. |
| (constraint) | | `EXCLUDE USING gist (employee_id WITH =, project_id WITH =, daterange(start_date, end_date, '[]') WITH &&)`. This stops overlapping assignments of the same employee to the same project even when two requests arrive at once (PRJ-8, NFR-DATA-1). Needs the `btree_gist` extension. |

**`tasks`** (Slice 1)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `name` | TEXT | Unique, 1 to 100 characters. |
| `is_active` | BOOLEAN | Default true. Inactive tasks stay on old entries but are not offered (PRJ-13, stretch). |

**`holidays`** (Slice 1 creates it empty. Seeded and edited from Slice 3.)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `holiday_date` | DATE | Unique: one holiday per date (HOL-1, NFR-DATA-1). |
| `name` | TEXT | Required, 1 to 100 characters. |

**`time_entries`** (Slice 1)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `employee_id` | BIGINT | Foreign key to `employees`. Always the session's employee. |
| `entry_date` | DATE | Required. Calendar date in IST. |
| `kind` | TEXT | `WORK` or `LEAVE`. Holidays are never stored here (Q1). |
| `project_id` | BIGINT | Foreign key to `projects`. Required when `kind = 'WORK'`, null otherwise. |
| `task_id` | BIGINT | Foreign key to `tasks`. Required when `kind = 'WORK'`, null otherwise. |
| `work_location` | TEXT | `OFFICE`, `CLIENT_SITE` or `WFH`. Required when `kind = 'WORK'`, null otherwise. |
| `leave_type` | TEXT | `CASUAL`, `SICK`, `EARNED` or `UNPAID`. Required when `kind = 'LEAVE'`, null otherwise (Slice 3). |
| `activity` | TEXT | Optional, at most 500 characters (TIME-3). |
| `duration_minutes` | SMALLINT | `CHECK (duration_minutes >= 15 AND duration_minutes <= 1440 AND duration_minutes % 15 = 0)` (TIME-5). Leave entries also require 240 or 480 (TIME-4). |
| `version` | INTEGER | Starts at 1, goes up by 1 on each update (Q6). |
| (constraint) | | A `CHECK` ties each `kind` to its required and forbidden columns as listed above. |
| (index) | | `(employee_id, entry_date)`. From Slice 4 also `(entry_date, project_id)`. |

**`day_totals`** (Slice 1)

| Column | Type | Rules |
|---|---|---|
| `employee_id` | BIGINT | Part of the primary key. |
| `entry_date` | DATE | Part of the primary key. Indexed alone for RPT-2. |
| `entry_minutes` | INTEGER | Sum of `time_entries.duration_minutes` for that employee and date. Kept up to date only by the `time_entries_totals` trigger. `CHECK (entry_minutes BETWEEN 0 AND 1440)`. |

**`period_locks`** (Slice 4, stretch)

| Column | Type | Rules |
|---|---|---|
| `month_start` | DATE | Primary key. Always the first day of a month (RPT-6). |
| `locked_by` | BIGINT | Foreign key to `employees`. |

**`audit_log`** (Slice 4)

| Column | Type | Rules |
|---|---|---|
| `id` | BIGINT | Primary key. |
| `occurred_at` | TIMESTAMPTZ | Default `now()`. Indexed. |
| `actor_id` | BIGINT | Employee who made the change. Null when the actor is `system`. |
| `actor_label` | TEXT | `system` or the actor's employee code at the time. |
| `record_type` | TEXT | Table name. Indexed with `occurred_at`. |
| `record_id` | TEXT | Primary key of the changed row. |
| `action` | TEXT | `CREATE`, `UPDATE` or `DELETE`. |
| `old_values` | JSONB | Row before the change. Master data only. |
| `new_values` | JSONB | Row after the change. Master data only. |

**Database functions and triggers**

| Name | Slice | Purpose |
|---|---|---|
| `time_entries_totals()` trigger | 1 | After any insert, update or delete on `time_entries`, adds or subtracts the change in `day_totals`. Handles an edit that moves an entry to another date. |
| `employee_holidays(emp, from, to)` | 1 | Holiday hours for one employee (Q1). |
| `employee_day_hours(from, to)` | 1 | Expected, entry and holiday minutes per active employee per date (Q1, RPT-2). |
| `audit_row()` trigger | 4 | Writes `audit_log` (Q8). |

**Database users:** `kairos_owner` owns the schema and runs migrations and
the seed. `kairos_app` is the user the API connects as. It can SELECT,
INSERT, UPDATE and DELETE on application tables, and only INSERT and SELECT on
`audit_log`. Both passwords come from environment variables.

### 6. API contracts

The base path is `/api/v1`. Bodies are JSON with camelCase field names. Dates
are `YYYY-MM-DD`. Durations are whole minutes. FastAPI generates the OpenAPI
3.1 document at `/api/v1/openapi.json` (PRD section 7.1), and that document is
the full contract. This section is its summary.

**Common to all endpoints**

*   Authenticated endpoints need the `__Host-kairos_session` cookie. Without
    a valid session: 401 `SESSION_ENDED`, "Your session has ended. Sign in
    again."
*   Writes need the header `X-Kairos-CSRF: 1`. Without it: 403 `CSRF_FAILED`,
    "This request was blocked. Reload the page and try again."
*   Bad input: 422 `VALIDATION_FAILED`, with `details.fields` as a list of
    `{ "field", "problem" }`.
*   Every response carries `X-Request-ID`.

**Slice 1 endpoints**

| Method and path | Request | Success | Failures |
|---|---|---|---|
| `POST /auth/sign-in` | `{ email, password }` | 200 `{ employee: Me }` and `Set-Cookie` | 401 `INVALID_CREDENTIALS` "Email or password is incorrect. Check both and try again." (the same reply for unknown email, wrong password and deactivated account, so it does not reveal which accounts exist). 423 `ACCOUNT_LOCKED` from Slice 2. |
| `POST /auth/sign-out` | none | 204, cookie cleared | 401 |
| `GET /me` | none | 200 `Me` | 401 |
| `GET /me/week?date=YYYY-MM-DD` | any date in the week. Defaults to today in IST. | 200 `Week` | 401, 422 |
| `GET /me/available-projects?date=YYYY-MM-DD&q=text` | date required. `q` matches code prefix or part of the name. | 200 `[ AvailableProject ]`, at most 50, common projects first | 401, 422 |
| `GET /tasks` | none | 200 `[ { id, name } ]` active tasks, with `ETag` | 401 |
| `POST /me/time-entries` | `NewWorkEntry` | 201 `Entry`, with `ETag` | 403 `PROJECT_NOT_AVAILABLE` (S-5). 409 `DAILY_LIMIT_EXCEEDED` (S-4). 422 `VALIDATION_FAILED`, `TASK_INACTIVE`. |
| `PUT /me/time-entries/{id}` | `NewWorkEntry` and `If-Match` | 200 `Entry`, new `ETag` | 404 `ENTRY_NOT_FOUND`, 409 `ENTRY_CHANGED`, 409 `DAILY_LIMIT_EXCEEDED`, 403 `PROJECT_NOT_AVAILABLE`, 428 `VERSION_REQUIRED`, 422 |
| `DELETE /me/time-entries/{id}` | `If-Match` | 204 | 404, 409 `ENTRY_CHANGED`, 428 |
| `GET /health/live` | none, no auth | 200 `{ "status": "ok" }` if the process is running | none |
| `GET /health/ready` | none, no auth | 200 if `SELECT 1` succeeds within 1 s | 503 `NOT_READY` |

**Shapes**

```text
Me               { id, employeeCode, fullName, workEmail, role, dateOfJoining,
                   lastWorkLocation | null, mustChangePassword }

Week             { weekStart, weekEnd, isoYear, isoWeek,
                   expectedMinutes, totalMinutes, complete,
                   days: [ Day x 7, Monday first ] }

Day              { date, weekday, isFuture, expectedMinutes, totalMinutes,
                   flags: [ "BELOW_EXPECTED" | "OVER_DAILY_LIMIT" ],
                   entries: [ Entry | HolidayEntry ] }

Entry            { id, kind: "WORK" | "LEAVE", entryDate, durationMinutes,
                   project: { id, code, name, label } | null,
                   task: { id, name } | null,
                   workLocation | null, leaveType | null, activity | null,
                   version, editable }

HolidayEntry     { id: null, kind: "HOLIDAY", entryDate, name,
                   durationMinutes: 480 or 0, editable: false }

NewWorkEntry     { entryDate, projectId, taskId, durationMinutes,
                   workLocation, activity? }

AvailableProject { id, code, name, label, type: "CLIENT" | "COMMON",
                   assignmentStart | null, assignmentEnd | null }
```

*   `flags` and `complete` are worked out by the server. `BELOW_EXPECTED` is
    set only for dates up to today, so future days of the current week are not
    shown as missing (TIME-12, Slice 3). `OVER_DAILY_LIMIT` marks a day that a
    holiday pushed past 24 hours (see The Problem). In Slice 1, `flags` is
    always empty and `editable` is always true for the employee's own entries.
    Slice 3 adds the date rules that set both.
*   The project rule (PRJ-4, PRJ-9) is one repository query used by both the
    selector and the save, so they cannot disagree. A project is available on
    date `d` when it is `ACTIVE`, `d` is within its start and end dates, and
    it is either `COMMON` or the employee has an assignment covering `d`.

**Slice 2 to 5 endpoints (contracts to be detailed in each slice's plan)**

| Slice | Endpoints |
|---|---|
| 2 | `GET/POST /employees`, `GET/PATCH /employees/{id}`, `POST /employees/{id}/deactivate` and `/reactivate`, `GET/POST /projects`, `GET/PATCH /projects/{id}`, `POST /projects/{id}/close` and `/reopen`, `GET/POST /projects/{id}/assignments`, `GET /employees/{id}/assignments`, `PATCH/DELETE /assignments/{id}`, `POST /me/password` (first sign-in change, AUTH-2) |
| 3 | `GET /holidays?year=`, `POST /holidays`, `PATCH/DELETE /holidays/{id}`, leave entries via `POST /me/time-entries` with `kind: "LEAVE"` |
| 4 | `GET /me/reports/monthly?month=`, `GET /reports/missing-time?week=&department=`, `GET /employees/{id}/week?date=`, `GET /audit-events` |
| Stretch | `GET /reports/project-hours?from=&to=`, `.../missing-time.csv`, `.../project-hours.csv`, `POST/DELETE /period-locks/{month}`, `POST /me/time-entries/copy-previous-week`, `POST /employees/import` (CSV) |
| 5 | `GET /metrics` (Prometheus text, NFR-OPS-4) |

### 7. Sequence diagrams

**S-1: Log a normal day** (TIME-1, TIME-3, TIME-5, TIME-16; Slice 1)

```mermaid
sequenceDiagram
    actor Priya
    participant UI as React app
    participant API as api (router → service → repository)
    participant DB as PostgreSQL

    Priya->>UI: Opens add form on today's column
    UI->>API: GET /me/available-projects?date=today&q=acme
    API->>DB: session lookup (sessions ⨝ employees, active, not expired)
    API->>DB: available projects query (COMMON or assignment covers today)
    DB-->>API: [2827] Acme Health – Dev Pod, Internal, Training, ...
    API-->>UI: 200 [AvailableProject]
    Priya->>UI: Picks project, task "Development (Coding)", 8h, Office, Save
    UI->>API: POST /me/time-entries {today, project, task, 480, OFFICE}
    API->>DB: session lookup
    API->>DB: BEGIN; set_config('kairos.actor_id', priya)
    API->>DB: available projects query filtered to this project
    API->>DB: INSERT day_totals ON CONFLICT DO NOTHING
    API->>DB: SELECT entry_minutes FROM day_totals ... FOR UPDATE
    API->>DB: employee_holidays(priya, today, today)
    Note over API: 0 + 0 + 480 ≤ 1440, so allowed
    API->>DB: INSERT time_entries (version 1)
    Note over DB: trigger adds 480 to day_totals
    API->>DB: UPSERT employee_preferences (OFFICE); COMMIT
    API-->>UI: 201 Entry, ETag "1"
    UI->>API: GET /me/week?date=today (query reloaded)
    API->>DB: entries for week + day_totals + employee_holidays
    API-->>UI: 200 Week (today 480, week total +480)
    UI-->>Priya: Card shows task, [2827] project, 8h 0m, Office. Totals updated.
```

**S-2: Holiday appears automatically** (HOL-1, HOL-4, HOL-5; Slice 3)

```mermaid
sequenceDiagram
    actor Arjun
    actor Priya
    participant UI as React app
    participant API as api
    participant DB as PostgreSQL

    Arjun->>UI: Adds "Founders' Day" on a Monday next month
    UI->>API: POST /holidays {date, name}
    API->>DB: session lookup → role HR_ADMIN, allowed
    API->>DB: BEGIN; set_config actor; INSERT holidays (unique date)
    Note over DB: audit_row() writes audit_log (from Slice 4)
    API->>DB: COMMIT
    API-->>UI: 201 Holiday
    Note over DB: One row written. No per-employee rows.

    Priya->>UI: Opens the week of that Monday
    UI->>API: GET /me/week?date=that Monday
    API->>DB: session lookup
    API->>DB: time_entries for 7 days (index employee_id, entry_date)
    API->>DB: employee_holidays(priya, Monday, Sunday)
    DB-->>API: Monday, "Founders' Day", 480
    Note over API: Merge as HolidayEntry (id null, editable false).<br/>Monday total and week total include 480.
    API-->>UI: 200 Week
    UI-->>Priya: Purple card labelled "Designated Holiday" with a lock icon,<br/>8h 0m, no edit or delete controls
```

**S-5: Unassigned project** (PRJ-9, TIME-15; Slice 1)

```mermaid
sequenceDiagram
    actor Priya
    participant UI as React app
    participant API as api
    participant DB as PostgreSQL

    Priya->>UI: Types "bor" in the project selector
    UI->>API: GET /me/available-projects?date=today&q=bor
    API->>DB: session lookup
    API->>DB: available projects query (no assignment to Borealis)
    DB-->>API: no rows match "bor"
    API-->>UI: 200 []
    UI-->>Priya: "No projects match. You can only pick projects HR has assigned to you."

    Note over Priya,API: Direct API call, bypassing the UI
    Priya->>API: POST /me/time-entries {projectId: Borealis, ...}
    API->>DB: session lookup
    API->>DB: BEGIN; available projects query filtered to Borealis
    DB-->>API: no row
    API->>DB: ROLLBACK (nothing written)
    API-->>Priya: 403 {code: PROJECT_NOT_AVAILABLE,<br/>message: "You cannot log time to this project on this date. Pick a project from the list."}
```

### 8. How Slices 2 to 5 build on this

*   **Slice 2** adds HR routers on the same three layers, the AUTH-2 to AUTH-4
    columns already in `employees`, deactivation that deletes sessions (Q2),
    and the EMP-7 date-of-joining check in the entry service.
*   **Slice 3** adds holiday endpoints and seed data. It needs no change to the
    week read or the 24-hour check, because both already call
    `employee_holidays`. It also adds leave entries (TIME-4), the date rules
    (TIME-8, TIME-9) that set `editable` and the 403 `DATE_NOT_EDITABLE`
    error, and the day and week flags.
*   **Slice 4** adds `audit_log` and the `audit_row()` triggers (Q8), the
    report endpoints (built on `employee_day_hours`), the HR read-only week
    (the same service as `/me/week`, given an employee ID after an HR role
    check), and the `(entry_date, project_id)` index.
*   **Slice 5** adds the volume seed (20 million entries, NFR-PERF-8), the k6
    scripts, `/metrics`, and tuning. Tuning changes only indexes,
    connection-pool sizes and PostgreSQL settings, unless the load test proves
    a design decision above wrong. If that happens, this document is updated
    first.

## Alternatives Considered

Full mechanisms and costs for each option are in
[decision_options.md](decision_options.md). Each rejection below was decided by
the author on 2026-11-19.

### Q1-A: One stored holiday row per employee

*   **Pros:** Reading a week needs no merge step, and every report adds up one
    table. The holiday row exists as a plain fact that can be audited per
    employee.
*   **Cons:** Adding, moving or deleting one holiday writes about 5,000 rows
    and 5,000 audit rows in one transaction. Every path that creates or
    reactivates an employee must also add that year's later holidays. A
    holiday change locks rows that employees' saves for that date may be
    waiting on during the Friday peak.
*   **Reason for Rejection:** Rejected by author: it turns a one-row change
    into a 5,000-row change and adds a backfill step that can be forgotten.
    Reading the holiday table on demand costs under 1 ms per week read.

### Q2-B: Short-lived signed token (JWT) plus a refresh token

*   **Pros:** Normal requests need no database read to check who the user is.
*   **Cons:** It adds refresh-token rotation, frontend retry logic and
    signing-key management, plus about 8 refresh calls a second. Revocation
    takes up to the full token lifetime, so it meets AUTH-6 only if the
    lifetime stays at 60 seconds or less. Signing out does not end the access
    token until it expires.
*   **Reason for Rejection:** Rejected by author: every request already reads
    PostgreSQL, so one primary-key lookup costs under 0.5 ms. The session table
    gives revocation that takes effect at once, with less code.

### Q3-B: Transaction-level advisory lock and a fresh sum

*   **Pros:** No extra table to keep in step with the entries.
*   **Cons:** The rule holds only if every write path remembers to take the
    lock. Nothing in the database catches a path that forgets.
*   **Reason for Rejection:** Rejected by author: the `day_totals` row gives
    the same one-at-a-time behaviour. It also adds a database `CHECK` as a last
    guard, and the missing-time report can reuse it.

### Q3-C: SERIALIZABLE transactions with a retry

*   **Pros:** No locks to remember.
*   **Cons:** PostgreSQL can report conflicts that aren't real, which causes
    retries during the Friday peak. Any write path that doesn't retry shows
    errors to users and counts against the 0.1% error budget (NFR-PERF-5). One
    path at a lower isolation level quietly turns the protection off.
*   **Reason for Rejection:** Rejected by author: the protection depends on
    every caller's isolation level and retry code, both of which are easy to
    get wrong without anyone noticing.

### Q4-B: Monthly range partitioning of `time_entries`

*   **Pros:** Range reports scan only the months they need, and old months can
    be detached for archiving.
*   **Cons:** Partitions for future months must be created ahead of time, or
    inserts fail. Unique constraints must include the date. Foreign keys that
    point at entries become awkward.
*   **Reason for Rejection:** Rejected by author: at 20 million rows (about
    4 to 5 GB including indexes) every query already reaches a small range
    through an index. The Slice 5 load test will show whether partitioning is
    needed.

### Q4-C: Indexes only, no pre-computed totals

*   **Pros:** The smallest schema.
*   **Cons:** RPT-2 adds up about 70,000 raw rows per week, and month-end
    reports by many HR users at once compete for the same pages.
*   **Reason for Rejection:** Rejected by author: `day_totals` already exists
    for the 24-hour lock (Q3), so the report can read it at no extra schema
    cost.

### Q5-B: Background job table and a worker process

*   **Pros:** CSV work runs completely apart from the API processes.
*   **Cons:** It adds an extra service, somewhere to keep finished files, a
    cleanup job and a progress screen. A stopped worker leaves exports
    waiting forever with no alert. The reference environment has no spare
    machine for the worker.
*   **Reason for Rejection:** Rejected by author: a 100,000-row export takes
    about 1 second of CPU. A limit of one export per instance and a separate
    connection pool keep it away from interactive requests, without new
    infrastructure. The Slice 5 load test checks this.

### Q5-C: Stream PostgreSQL `COPY ... TO STDOUT`

*   **Pros:** The least Python CPU and the fastest.
*   **Cons:** Protection against CSV injection and the column formatting rules
    would have to be written in SQL, where a missed case is harder to test.
*   **Reason for Rejection:** Rejected by author: escaping in Python is easier
    to test, and speed is not the limit at 100,000 rows.

### Q6-B: Last-updated timestamp as the version

*   **Pros:** No extra column.
*   **Cons:** Two updates in the same clock tick, or a timestamp that loses
    precision when converted to or from JSON, make conflicts go undetected or
    appear when there isn't one.
*   **Reason for Rejection:** Rejected by author: an integer counter cannot be
    misread.

### Q7-B: In-memory cache in each instance, cleared by LISTEN/NOTIFY

*   **Pros:** Saves a small query per request for data that rarely changes.
*   **Cons:** It needs a long-lived connection per instance and code to
    reconnect it. If the connection drops without anyone noticing, an instance
    serves stale holidays, which breaks HOL-5.
*   **Reason for Rejection:** Rejected by author: there is no measured need
    yet. The Slice 5 load test decides.

### Q8-B: ORM session events

*   **Pros:** The audit logic is in Python and easy to unit-test.
*   **Cons:** Bulk `update()` and `delete()` statements and raw SQL don't fire
    ORM events, so they skip the audit silently.
*   **Reason for Rejection:** Rejected by author: this is exactly the "new
    code path" risk that PRD question 10.8 asks the design to close.

### Q8-C: Explicit audit calls in a base repository class

*   **Pros:** Simple and visible in the code.
*   **Cons:** Nothing stops a repository method that bypasses the base class.
*   **Reason for Rejection:** Rejected by author: the protection depends on
    every developer remembering to use the base class.

### Infrastructure and tooling rejected during brainstorming

*   **Redis for sessions or caching.** Rejected by author: the reference
    environment (NFR-PERF-7) has no machine for it. PostgreSQL handles session
    lookup at this load (about 100 requests a second), and nothing has shown a
    cache is needed.
*   **Keycloak.** Rejected by author: PRD section 7.1 fixes identity as built
    into the application. Keycloak would also need user, role and
    deactivation data kept in step between the two systems (EMP-1, EMP-4,
    AUTH-6).
*   **`podman compose` (the built-in subcommand).** Rejected by author: it can
    hand work to the `docker-compose` binary, and the compliance rule
    forbids Docker. `podman-compose` (the separate Python tool) is pinned
    instead.
*   **Compose `depends_on` conditions for start-up order.** Rejected by
    author: `podman-compose` supports them inconsistently across versions. The
    Makefile runs the steps in order instead.
*   **SQLite for backend tests.** Rejected by author: it cannot test row
    locks, exclusion constraints or triggers, which Q3, PRJ-8 and Q8 depend
    on.
*   **Synchronous SQLAlchemy with `psycopg` 3.** Rejected by author: under
    the Friday peak, the default limit of 40 worker threads would cap how many
    requests an instance can serve at once. With async, the size of the
    connection pool is the only limit.
*   **bcrypt (cost 12).** Rejected by author: at about 250 ms per hash, a
    burst of 500 sign-ins in 60 seconds leaves little spare CPU on 4 vCPU.
    argon2id at about 50 to 80 ms leaves room and is the current OWASP
    recommendation.
*   **shadcn/ui.** Rejected by author: MUI ships the autocomplete, date
    picker and modal the weekly view needs. The colours in design-theme.md can
    be applied through MUI's theme.

--------------------------------------------------------------------------------

## Non-Functional Requirements

*   **Security and privacy:** Server-side checks for authentication, CSRF and
    role on every route (NFR-SEC-2, NFR-SEC-5). `/me` paths take the employee
    from the session and never from the request (NFR-SEC-3). argon2id hashes
    (NFR-SEC-1). Log redaction of passwords, tokens and cookies (NFR-SEC-1).
    HTTPS through Caddy in every environment, including local (NFR-SEC-4).
    Secrets only in environment variables, with `.env` git-ignored
    (NFR-SEC-8). Every SQL statement uses bound parameters through
    SQLAlchemy, and no SQL is built from strings (NFR-SEC-5). React escapes
    output by default, and the code does not use `dangerouslySetInnerHTML`.
*   **Performance:** Section 4 explains how each NFR-PERF target is
    approached. Slice 1 is not load-tested. Slice 5 proves NFR-PERF-1 to
    NFR-PERF-9 on the reference environment with 20 million entries. The
    riskiest targets are p99 under 100 ms for the week read and entry save
    during the Friday peak, and sign-in p99 under 500 ms during a 500-user
    burst.
*   **Rollout and rollback:** Each slice is a migration plus code. Migrations
    are forward-only in deployed environments. Each Alembic revision has a
    `downgrade` for local use only. Slice 1 has no existing data, so nothing
    needs to be migrated.
*   **Container images:** Image names are fully qualified (for example
    `docker.io/library/postgres:16`) because Podman asks which registry to use
    for short names. Each name can be overridden by an environment variable
    (`KAIROS_POSTGRES_IMAGE`, `KAIROS_PYTHON_IMAGE`, `KAIROS_NODE_IMAGE`,
    `KAIROS_CADDY_IMAGE`). This allows switching to an approved registry if
    the compliance rule against Docker also covers Docker Hub.

--------------------------------------------------------------------------------

### Additional Resources

*   [prd.md](../prd.md) version 1.3: requirements, scenarios and slices.
*   [design-theme.md](../context/design-theme.md): colours, layout, tone of
    messages.
*   [engineering-standards.md](../context/engineering-standards.md):
    repository layout, layers, error shape, `make` commands, commit rules.
*   [decision_options.md](decision_options.md): full options and trade-offs
    for Q1 to Q9.

### Future Improvements

Slices 2 to 5 are future work. Each one gets its own detailed implementation
plan, added to this document or a follow-up design, before it is built. The
Technical Plan already fixes their architecture.

*   **Slice 2, HR master data** (EMP-1 to EMP-5, EMP-7, EMP-8, PRJ-1 to PRJ-3,
    PRJ-5 to PRJ-8, PRJ-10, AUTH-2 to AUTH-4, AUTH-6). HR routers and screens
    for employees, projects and assignments. Temporary passwords and
    first-sign-in change. Lock-out after 5 failures. Deactivation that deletes
    sessions. Last-HR-admin protection (EMP-8), which needs a row lock on the
    HR admin count so that two demotions at once cannot both pass. Decide on
    department as free text or a managed list first.
*   **Slice 3, Holidays and leave** (HOL-1 to HOL-6, TIME-4, TIME-8, TIME-9,
    TIME-11, TIME-12). Holiday endpoints and screen. Holiday seed data. Leave
    entries. Date rules that set `editable`. The `BELOW_EXPECTED` and
    `OVER_DAILY_LIMIT` flags.
*   **Slice 4, Reports and audit** (RPT-1, RPT-2, RPT-4, AUD-1). `audit_log`
    and the triggers (Q8). Monthly and missing-time reports on
    `employee_day_hours`. HR read-only week view.
*   **Slice 5, Performance proof** (NFR-PERF-1 to NFR-PERF-9). Volume seed,
    k6 scripts, `/metrics`, tuning.
*   **Stretch** (remaining S and C items). RPT-3 and RPT-5 exports (Q5). RPT-6
    locks and TIME-10. AUD-2 and AUD-3. TIME-13 copy-week. TIME-18 shortcut.
    EMP-6 and PRJ-12 CSV import. PRJ-11, PRJ-13, PRJ-14, HOL-7, HOL-8, AUTH-7,
    AUTH-8.

--------------------------------------------------------------------------------

## Detailed Implementation Plan (Slice 1 only)

Scope: Slice 1, "Sign in and log time". It covers AUTH-1, AUTH-5, TIME-1 to
TIME-3, TIME-5 to TIME-7, TIME-14 to TIME-16, PRJ-4 and PRJ-9. It also
delivers the TIME-17 version check early (see Q6), plus liveness and
readiness checks (NFR-REL-3) and correlation IDs (NFR-OPS-3), because later
slices build on them.

All paths are relative to the project root `Kairos/`. The files are grouped
into sub-tasks ST-01 to ST-10, each done as one commit whose message starts
with the sub-task ID ([engineering-standards.md](../context/engineering-standards.md)).
`implementation-plan.md` will be generated from this section.

### Target Files to Create/Modify

#### ST-01: Repository skeleton, containers and `make` commands

1.  **`Makefile`** (Create)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** These targets:
        *   `up`: starts `db`, waits for `pg_isready`, runs `migrate` then
            `seed` as one-off containers, then starts `api` and `proxy`.
        *   `down`
        *   `test`: `test-backend`, `test-frontend` and `test-e2e`.
        *   `lint`: `ruff check`, `ruff format --check`, `mypy --strict`,
            `eslint`, `tsc --noEmit`, and `openapi-check`, which fails if the
            generated frontend API types are out of date.
        *   `openapi`: regenerates the API types.
        *   `logs`
        *   `psql`: opens a database shell.

        Every target calls `podman` or `podman-compose`, never `docker`.
        `up`, `test` and `lint` are required by engineering-standards.md.
2.  **`compose.yaml`** (Create)
    *   **Type / Scope:** podman-compose definition.
    *   **Proposed Changes:** Services `db` (PostgreSQL 16, named volume,
        init scripts mounted, host port 55432), `api` (backend image, port
        8000 inside the network only), `proxy` (Caddy, host port 8443), and
        `migrate` and `seed` (backend image with a different command). Images
        come from `KAIROS_*_IMAGE` variables with fully-qualified defaults.
        Configuration is read from `.env`.
3.  **`.env.example`** (Create)
    *   **Type / Scope:** Configuration template.
    *   **Proposed Changes:** Every variable with a safe local value:
        `KAIROS_ENV=local`, the owner and app database users and passwords,
        `DATABASE_URL`, `KAIROS_SEED_PASSWORD`, the image variables and
        `KAIROS_LOG_LEVEL`. `make up` copies it to `.env` if `.env` is
        missing (NFR-SEC-8, NFR-OPS-1).
4.  **`.gitignore`** (Create)
    *   **Type / Scope:** Git configuration.
    *   **Proposed Changes:** Ignore `.env`, `node_modules/`, `dist/`,
        `.venv/`, `__pycache__/`, coverage output and Playwright reports.
5.  **`db/init/01-roles.sh`** (Create)
    *   **Type / Scope:** PostgreSQL first-start script.
    *   **Proposed Changes:** Creates `kairos_owner` and `kairos_app` with
        passwords from environment variables, creates the `kairos` database
        owned by `kairos_owner`, and enables `btree_gist`. Runs only when the
        data volume is empty.
6.  **`backend/pyproject.toml`** and **`backend/uv.lock`** (Create)
    *   **Type / Scope:** Python project.
    *   **Proposed Changes:**
        *   Runtime dependencies: FastAPI, uvicorn, SQLAlchemy 2.0, asyncpg,
            Alembic, pydantic, pydantic-settings, argon2-cffi.
        *   Development dependencies: pytest, pytest-asyncio, pytest-cov,
            httpx, ruff, mypy.
        *   Configuration: ruff rules, `mypy --strict`, and pytest settings
            with `--cov=kairos --cov-fail-under=80` (NFR-OPS-2).
7.  **`backend/Containerfile`** (Create)
    *   **Type / Scope:** Container image.
    *   **Proposed Changes:** Python 3.12 slim base. Installs dependencies
        with `uv`, runs as a user that is not root, and starts uvicorn with
        `kairos.main:app`.
8.  **`backend/kairos/__init__.py`** (Create), **`backend/kairos/main.py`**
    (Create), **`backend/kairos/config.py`** (Create)
    *   **Type / Scope:** FastAPI application entry point and settings.
    *   **Proposed Changes:** `create_app()` builds the FastAPI app with title
        "Kairos" and OpenAPI at `/api/v1/openapi.json`, then registers
        middleware and routers. `config.Settings` reads environment variables
        and refuses to start if a required one is missing.
9.  **`backend/kairos/health/router.py`** (Create)
    *   **Type / Scope:** Router.
    *   **Proposed Changes:** `GET /api/v1/health/live` and
        `GET /api/v1/health/ready`. Readiness runs `SELECT 1` with a 1-second
        timeout (NFR-REL-3).
10. **`backend/tests/test_health.py`** (Create)
    *   **Type / Scope:** API test.
    *   **Proposed Changes:** Live returns 200. Ready returns 200 with the
        database up, and 503 `NOT_READY` when the engine points at a closed
        port.
11. **`frontend/package.json`**, **`frontend/package-lock.json`**,
    **`frontend/tsconfig.json`**, **`frontend/vite.config.ts`**,
    **`frontend/eslint.config.js`**, **`frontend/index.html`** (Create)
    *   **Type / Scope:** Frontend project.
    *   **Proposed Changes:**
        *   Dependencies: React 18, React Router, TanStack Query v5, MUI v6,
            and the Inter and Roboto Condensed fonts installed locally.
        *   Development tools: Vite, Vitest, Testing Library,
            openapi-typescript, ESLint and Playwright.
        *   Scripts: `dev`, `build`, `test`, `lint`, `typecheck`, `gen:api`
            and `e2e`.
        *   During development, Vite forwards `/api` to `localhost:8000`.
12. **`proxy/Caddyfile`** and **`proxy/Containerfile`** (Create)
    *   **Type / Scope:** Reverse proxy.
    *   **Proposed Changes:** The Containerfile builds the frontend in a Node
        20 stage and copies `dist/` into the Caddy image. The Caddyfile does
        four things:
        *   Serves `dist/`, falling back to `index.html` so client-side
            routes load.
        *   Forwards `/api/*` to `api:8000` with a 60-second timeout.
        *   Uses `tls internal` for local HTTPS.
        *   Sets the `Content-Security-Policy: default-src 'self'`,
            `X-Content-Type-Options` and `Referrer-Policy` headers.
13. **`README.md`** (Create)
    *   **Type / Scope:** Documentation (NFR-OPS-6).
    *   **Proposed Changes:**
        *   Prerequisites: Podman 4.9 or later, `podman-compose` 1.1 or later,
            and `make`.
        *   One-command start with `make up`, then open
            `https://localhost:8443` and accept the local certificate once.
        *   The seed accounts and their password, as a table.
        *   `make test`, `make lint` and `make openapi`.
        *   Every environment variable from `.env.example` with its meaning.
        *   How to reset the database (`make down` and remove the volume).
        *   A placeholder section, "Load test (Slice 5)".

        Written in ST-01, then extended by ST-05 (seed accounts) and ST-10
        (end-to-end tests).

#### ST-02: Backend foundation (errors, logging, database session, request pipeline)

1.  **`backend/kairos/errors.py`** (Create)
    *   **Type / Scope:** Shared module.
    *   **Proposed Changes:** `AppError(status, code, message, details)` and a
        subclass for each code in Technical Plan section 6. Exception handlers
        turn `AppError`, FastAPI validation errors (into `VALIDATION_FAILED`
        with `details.fields`) and unexpected exceptions (into
        `INTERNAL_ERROR`, with the stack trace sent only to the log) into the
        `{ code, message, details }` body.
2.  **`backend/kairos/logging.py`** (Create)
    *   **Type / Scope:** Shared module.
    *   **Proposed Changes:** A JSON log formatter that adds `request_id`,
        `employee_id`, `method`, `path`, `status` and `duration_ms`. A
        redaction filter for the `password`, `token`, `cookie` and
        `authorization` keys at any depth (NFR-SEC-1, NFR-OPS-3).
3.  **`backend/kairos/middleware.py`** (Create)
    *   **Type / Scope:** ASGI middleware.
    *   **Proposed Changes:** Takes `X-Request-ID` from the request (accepted
        only if it is a UUID) or creates one. Stores it in a context
        variable, echoes it in the response, and logs one access line per
        request.
4.  **`backend/kairos/db.py`** (Create)
    *   **Type / Scope:** Database session module.
    *   **Proposed Changes:**
        *   Creates the async engine (pool of 20 plus up to 10 extra,
            `statement_timeout` 5 s for the interactive pool).
        *   `get_session()` dependency.
        *   `transaction(actor_id)` context manager. It begins a transaction
            and runs `set_config('kairos.actor_id', …, true)`.
        *   An event hook that adds `/* request_id=<id> */` to every
            statement.
5.  **`backend/kairos/security/csrf.py`** (Create)
    *   **Type / Scope:** Dependency.
    *   **Proposed Changes:** Refuses POST, PUT, PATCH and DELETE without
        `X-Kairos-CSRF: 1`, with 403 `CSRF_FAILED`. The sign-in route uses it
        too.
6.  **`backend/tests/conftest.py`** and **`backend/tests/factories.py`**
    (Create)
    *   **Type / Scope:** Test fixtures.
    *   **Proposed Changes:**
        *   Points at the test database started by `make test-backend`.
        *   Runs `alembic upgrade head` once per test session.
        *   Clears all tables between tests (needed because the concurrency
            tests must commit).
        *   An httpx `AsyncClient` against the app.
        *   Factory functions for employees, projects, assignments, tasks,
            holidays and entries.
        *   A `signed_in_client(employee)` helper.

        Tests never use the seed (PRD section 8).
7.  **`backend/tests/test_errors_and_logging.py`** (Create)
    *   **Type / Scope:** Unit and API tests.
    *   **Proposed Changes:**
        *   The error shape is the same for 401, 403, 404, 422 and 500.
        *   A log line from a sign-in request contains neither the password
            nor the cookie.
        *   The response echoes `X-Request-ID`.
        *   A request without the CSRF header gets 403.

#### ST-03: Slice 1 database schema

1.  **`backend/alembic.ini`**, **`backend/migrations/env.py`**,
    **`backend/migrations/script.py.mako`** (Create)
    *   **Type / Scope:** Migration configuration.
    *   **Proposed Changes:** Async Alembic environment that connects as
        `kairos_owner`. Revision filenames start with a 4-digit number.
2.  **`backend/migrations/versions/0001_slice1_schema.py`** (Create)
    *   **Type / Scope:** Migration (NFR-DATA-2).
    *   **Proposed Changes:**
        *   Creates `employees`, `sessions`, `employee_preferences`,
            `projects` (with `project_code_seq` starting at 1001),
            `assignments` (with the exclusion constraint), `tasks`,
            `holidays`, `time_entries` and `day_totals`. Columns, `CHECK`s
            and indexes are exactly as in the data dictionary.
        *   Creates the `time_entries_totals()` trigger, and the
            `employee_holidays()` and `employee_day_hours()` functions.
        *   Grants SELECT, INSERT, UPDATE and DELETE on all tables, and USAGE
            on the sequence, to `kairos_app`.

        `downgrade()` drops everything, for local use only.
3.  **`backend/kairos/models.py`** (Create)
    *   **Type / Scope:** SQLAlchemy table definitions.
    *   **Proposed Changes:** Declarative models that match the migration,
        used by repositories. Models are not used to create tables: the
        migration is the only source of the schema.
4.  **`backend/tests/db/test_schema.py`** (Create)
    *   **Type / Scope:** Database tests.
    *   **Proposed Changes:**
        *   The `duration_minutes` `CHECK` rejects 10, 20 and 1455.
        *   The `kind` `CHECK` rejects a WORK entry without a project.
        *   The exclusion constraint rejects an overlapping assignment and
            accepts one next to it.
        *   The trigger keeps `day_totals` correct across insert, update,
            change of date and delete.
        *   `employee_holidays` returns 480 for a weekday, 0 for a weekend,
            and nothing before the date of joining.
        *   `kairos_app` cannot drop tables.

#### ST-04: Sign in, sessions, sign out (AUTH-1, AUTH-5)

1.  **`backend/kairos/security/passwords.py`** (Create)
    *   **Type / Scope:** Module.
    *   **Proposed Changes:** `hash_password()` and `verify_password()` with
        argon2id (3 passes, 64 MiB, 1 lane). They run in a worker thread, at
        most 4 at a time per process. `needs_rehash()` lets the settings be
        changed later.
2.  **`backend/kairos/auth/schemas.py`**, **`router.py`**, **`service.py`**,
    **`repository.py`**, **`dependencies.py`** (Create)
    *   **Type / Scope:** Feature module (router → service → repository).
    *   **Proposed Changes:**
        *   **Sign-in:** the email is lowercased. When the employee is not
            found, a dummy hash is still verified so that response time does
            not reveal whether the account exists. Inactive accounts get the
            same 401. On success the service creates a 32-byte token and
            stores its SHA-256 hash with `expires_at = now() + 8h`. It sets
            the cookie `__Host-kairos_session` with `HttpOnly`, `Secure`,
            `SameSite=Strict`, `Path=/` and `Max-Age=28800`.
        *   **Sign-out:** deletes the session and clears the cookie.
        *   **`current_employee` dependency:** runs one query (session joined
            to employee, not expired, active) and otherwise returns 401
            `SESSION_ENDED`.
        *   **`GET /me`.**
        *   **Expired sessions:** deleted at sign-in for that employee. A
            start-up task, under an advisory lock, deletes expired sessions
            every 24 hours.
3.  **`backend/tests/api/test_auth.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   Sign-in succeeds and sets the cookie with all of its attributes.
        *   Wrong password, unknown email and inactive account all return the
            same 401 body.
        *   Email matching ignores case.
        *   A session whose `expires_at` is set to the past returns 401
            (AUTH-5).
        *   After sign-out the old cookie returns 401.
        *   `/me` without a cookie returns 401.
        *   A session belonging to an employee made inactive in the database
            returns 401 on the next request. This is groundwork for AUTH-6.

#### ST-05: Seed script (PRD section 8, Slice 1 subset)

1.  **`backend/kairos/seed.py`** (Create)
    *   **Type / Scope:** Command-line script, run as
        `python -m kairos.seed`.
    *   **Proposed Changes:** Refuses to run when `KAIROS_ENV` is not `local`.
        Connects as `kairos_owner` and sets the actor to `system`. Every
        insert first checks for an existing row by its natural key, so
        running the script twice gives the same result. Fixed data in Slice 1:
        *   **Employees:** 1 HR administrator and 5 employees across the
            Engineering and Delivery departments. Fixed employee codes
            `K0001` to `K0006`, fixed emails `<first-name>@kairos.local`, and
            dates of joining 400 days before the run date. All use the
            password from `KAIROS_SEED_PASSWORD` (default `Kairos2026x`,
            which meets AUTH-3). No holidays are seeded in Slice 1.
        *   **Projects:** the 4 common projects from PRJ-4 and 3 client
            projects, found again by name on later runs.
        *   **Assignments:** each employee has at least one client
            assignment, and one employee has an assignment that ended 10
            days before the run date.
        *   **Tasks:** the 12 tasks from PRD section 8.
        *   **Entries:** for each employee, for the previous week (Monday to
            Friday) and the current week up to today, the script inserts
            entries only on dates where that employee has none. That keeps
            reruns from duplicating entries or touching entries a developer
            added. One employee's previous week is left at 32 hours, so at
            least one week is incomplete. The script uses a fixed random
            seed, so the same run date always produces the same entries.
            Durations are multiples of 15 minutes and stay within 24 hours a
            day.
2.  **`backend/tests/test_seed.py`** (Create)
    *   **Type / Scope:** Test.
    *   **Proposed Changes:**
        *   Running the seed twice gives the same row counts.
        *   The current and previous week both contain entries for every
            employee.
        *   At least one employee's previous week is under 40 hours.
        *   No seeded day is over 1,440 minutes.
        *   The script refuses to run with `KAIROS_ENV=production`.
3.  **`README.md`** (Modify)
    *   **Type / Scope:** Documentation.
    *   **Proposed Changes:** Add the table of seed accounts (employee code,
        name, email, role, department, assigned projects), and note which
        employee has the ended assignment and which has the incomplete week.

#### ST-06: Available projects and tasks (PRJ-4, PRJ-9, TIME-15)

1.  **`backend/kairos/projects/schemas.py`**, **`router.py`**,
    **`service.py`**, **`repository.py`** (Create)
    *   **Type / Scope:** Feature module.
    *   **Proposed Changes:** `GET /api/v1/me/available-projects?date=&q=`.
        The repository function `available_projects(employee_id, on_date,
        q=None, project_id=None)` is the single PRJ-4/PRJ-9 query (Technical
        Plan section 6). The service calls it with `project_id` for saves, and
        with `q` for the selector. `q` matches a code prefix or part of the
        name, ignoring case. At most 50 results, ordered with common projects
        first, then by code. `label` is `[code] name`.
2.  **`backend/kairos/tasks/schemas.py`**, **`router.py`**,
    **`repository.py`** (Create)
    *   **Type / Scope:** Feature module. There is no service file, because
        there are no rules.
    *   **Proposed Changes:** `GET /api/v1/tasks` returns active tasks sorted
        by name, with an `ETag` that is a hash of the result.
3.  **`backend/tests/api/test_available_projects.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   S-12: a new employee sees exactly the 4 common projects.
        *   S-5 (selector): an unassigned client project is absent.
        *   S-11 date rule (assignments created directly in the database):
            the project is offered for the assignment's last day and not the
            day after.
        *   A closed project and a project outside its own dates are not
            offered.
        *   Search by code prefix and by part of the name works.
        *   Without a session the request returns 401.

#### ST-07: Weekly view and time entries API (TIME-1 to TIME-3, TIME-5 to TIME-7, TIME-14, TIME-16, TIME-17 early)

1.  **`backend/kairos/time_entries/schemas.py`** (Create)
    *   **Type / Scope:** Pydantic models.
    *   **Proposed Changes:** `NewWorkEntry`:
        *   `durationMinutes` is between 15 and 1440 and a multiple of 15
            (TIME-5).
        *   `activity` is at most 500 characters, with surrounding spaces
            removed (TIME-3).
        *   `workLocation` is one of the enumeration values.

        Also the `Entry`, `HolidayEntry`, `Day` and `Week` models.
2.  **`backend/kairos/time_entries/repository.py`** (Create)
    *   **Type / Scope:** Repository.
    *   **Proposed Changes:**
        *   `entries_for_week(employee_id, monday)`.
        *   `holidays_for_range(employee_id, from, to)`, which calls
            `employee_holidays`.
        *   `lock_day(employee_id, date)`, which runs the insert-on-conflict
            and then `SELECT … FOR UPDATE`, returning the minutes.
        *   `insert_entry`.
        *   `update_entry_if_version(id, employee_id, version, values)`,
            which returns the new row or None.
        *   `delete_entry_if_version`.
        *   `get_entry_for_owner(id, employee_id)`.
        *   `upsert_last_location`.
3.  **`backend/kairos/time_entries/service.py`** (Create)
    *   **Type / Scope:** Service, which holds every rule.
    *   **Proposed Changes:**
        *   **`read_week(employee, any_date)`:** finds Monday with ISO
            rules. It merges entries with holiday pseudo-entries, then works
            out day and week totals, `expectedMinutes` (480 on each weekday
            on or after the date of joining), `complete` and `isFuture`
            (TIME-1, TIME-2, TIME-16).
        *   **`create_work_entry`:** inside `transaction(actor)`:
            1.  Checks `available_projects(..., project_id)`, otherwise 403
                `PROJECT_NOT_AVAILABLE` (PRJ-9).
            2.  Checks the task is active, otherwise 422 `TASK_INACTIVE`.
            3.  Locks the day with `lock_day`.
            4.  Checks that existing minutes plus holiday minutes plus the
                new minutes are at most 1,440. Otherwise it returns 409
                `DAILY_LIMIT_EXCEEDED` with
                `details { date, existingMinutes, holidayMinutes,
                requestedMinutes, limitMinutes }`, and the message "This day
                would have more than 24 hours. Reduce the duration or edit
                another entry." (TIME-6).
            5.  Inserts the entry and updates `last_work_location` (TIME-3).

            Two entries with the same project and task on the same day are
            allowed (TIME-14).
        *   **`update_work_entry`:** the same checks. When the date changes,
            both days are locked in date order, and the entry's old minutes
            are subtracted. The version must match, otherwise 409
            `ENTRY_CHANGED` with the current entry. Another employee's entry
            gives 404 (TIME-7, NFR-SEC-3).
        *   **`delete_work_entry`:** checks the version and the owner (TIME-7).

        Slice 1 does not apply TIME-8 or TIME-9, so `editable` is always true
        for the employee's own entries.
4.  **`backend/kairos/time_entries/router.py`** (Create)
    *   **Type / Scope:** Router.
    *   **Proposed Changes:**
        *   `GET /api/v1/me/week`.
        *   `POST /api/v1/me/time-entries`.
        *   `PUT /api/v1/me/time-entries/{id}`, which requires `If-Match`
            and otherwise returns 428 `VERSION_REQUIRED`.
        *   `DELETE /api/v1/me/time-entries/{id}`.
        *   Sets the `ETag` response header.
5.  **`backend/tests/api/test_week.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   The ISO week number and date range are correct, including week 1
            of a year that starts mid-week and week 53.
        *   Day and week totals are correct.
        *   A holiday inserted directly in the database appears as a
            non-editable `HOLIDAY` entry and is counted. This proves the Q1
            path before Slice 3.
        *   Expected minutes exclude days before the date of joining.
        *   A week with no entries returns 7 empty days.
6.  **`backend/tests/api/test_time_entries.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   S-1 at the API level: 8h for today shows in the day and week
            totals.
        *   S-4: 20h existing plus 5h is rejected with
            `DAILY_LIMIT_EXCEEDED`, and nothing is saved.
        *   S-5 (direct API): returns 403 `PROJECT_NOT_AVAILABLE`, and
            nothing is saved.
        *   TIME-5: 10 minutes and 50 minutes are rejected, 15 minutes is
            accepted.
        *   TIME-14: two identical entries on one day are accepted.
        *   Edit and delete own entries (TIME-7).
        *   Another employee's entry gives 404 on PUT and DELETE (NFR-SEC-3).
        *   A stale `If-Match` gives 409 `ENTRY_CHANGED` with the current
            entry. A missing `If-Match` gives 428.
        *   Moving an entry to a full day is rejected.
        *   The last-used work location is returned by `/me`.
7.  **`backend/tests/api/test_daily_limit_concurrency.py`** (Create)
    *   **Type / Scope:** Concurrency test (NFR-DATA-1).
    *   **Proposed Changes:** An employee has 16 hours on one day. Two
        sessions each send a 5-hour POST at the same time (`asyncio.gather`
        over two separate database connections), and exactly one succeeds.
        The test runs 20 times. In a second case, two 4-hour edits that move
        entries onto the same day run at the same time, and the day never
        ends over 1,440 minutes.

#### ST-08: Frontend foundation and sign-in

1.  **`frontend/src/main.tsx`**, **`frontend/src/App.tsx`** (Create)
    *   **Type / Scope:** App entry and routes.
    *   **Proposed Changes:**
        *   Sets up `QueryClientProvider`, MUI `ThemeProvider` and the
            router.
        *   Routes `/sign-in`, `/week/:isoWeek`, and `/`, which redirects to
            the current week.
        *   A route guard calls `GET /me` and sends a 401 to `/sign-in`.
2.  **`frontend/src/theme.ts`** (Create)
    *   **Type / Scope:** MUI theme.
    *   **Proposed Changes:** Colours from design-theme.md as named palette
        entries (`work`, `leave`, `holiday`, `warning`). Inter at 14 px, and
        a `numeric` text style in condensed bold. Visible focus outlines on
        all interactive components (NFR-UX-3).
3.  **`frontend/src/api/client.ts`** (Create)
    *   **Type / Scope:** API client module, the only place `fetch` is used.
    *   **Proposed Changes:**
        *   Typed functions `signIn`, `signOut`, `getMe`, `getWeek`,
            `getAvailableProjects`, `getTasks`, `createEntry`, `updateEntry`
            and `deleteEntry`.
        *   Adds `X-Request-ID` (`crypto.randomUUID()`), `X-Kairos-CSRF: 1`
            and `If-Match`, and uses `credentials: 'same-origin'`.
        *   Throws `ApiError { status, code, message, details }`.
4.  **`frontend/src/api/schema.ts`** (Create, generated) and
    **`backend/kairos/export_openapi.py`** (Create)
    *   **Type / Scope:** Generated types and the script that exports them.
    *   **Proposed Changes:** `make openapi` runs `python -m
        kairos.export_openapi`, which writes
        `frontend/src/api/openapi.json`, then runs `openapi-typescript` to
        produce `schema.ts`. Both files are committed. `make lint` fails if
        regenerating them changes either file.
5.  **`frontend/src/components/TopBar.tsx`**,
    **`frontend/src/components/ErrorMessage.tsx`** (Create)
    *   **Type / Scope:** Shared components.
    *   **Proposed Changes:**
        *   **TopBar:** a dark bar with "Kairos", a search box (disabled in
            Slice 1 and labelled "Search, coming soon"), and a user menu
            with the user's name and "Sign out".
        *   **ErrorMessage:** shows `ApiError.message` as written, with an
            icon and `role="alert"`.
6.  **`frontend/src/features/auth/SignInPage.tsx`** (Create)
    *   **Type / Scope:** Feature page.
    *   **Proposed Changes:** Email and password fields with labels. Shows
        the server's message on failure. On success, goes to the current
        week.
7.  **`frontend/src/features/auth/SignInPage.test.tsx`**,
    **`frontend/src/api/client.test.ts`** (Create)
    *   **Type / Scope:** Unit tests.
    *   **Proposed Changes:**
        *   The client adds the headers and turns an error body into an
            `ApiError`.
        *   The sign-in page shows the server's message and moves to the
            week on success.

#### ST-09: Weekly view and entry form

1.  **`frontend/src/features/week/WeekPage.tsx`** (Create)
    *   **Type / Scope:** Feature page.
    *   **Proposed Changes:**
        *   **Header:** date range, "Week 47", week total, and a "Week
            incomplete" or "Week complete" label with an icon.
        *   **Navigation:** previous and next buttons, and a date picker that
            jumps to any date (TIME-2).
        *   **Layout:** seven `DayColumn`s, stacking below 768 px.
        *   **Loading and errors:** a skeleton while loading, `ErrorMessage`
            on failure.
2.  **`frontend/src/features/week/DayColumn.tsx`**,
    **`frontend/src/features/week/EntryCard.tsx`** (Create)
    *   **Type / Scope:** Components.
    *   **Proposed Changes:**
        *   **DayColumn:** the date, the day total in the numeric style, an
            "Add" button, and entry cards stacked under it.
        *   **EntryCard:** a solid card in the kind's colour, with the kind's
            text label and icon, task, `[code] name`, duration as `8h 0m` and
            work location (TIME-11, TIME-16). Edit and delete actions are
            shown only when `editable` is true.
        *   **Delete:** asks for confirmation, then sends `If-Match`.
3.  **`frontend/src/features/week/EntryFormModal.tsx`**,
    **`frontend/src/features/week/ProjectSelect.tsx`** (Create)
    *   **Type / Scope:** Components.
    *   **Proposed Changes:**
        *   **The form:** a centred modal with fields in the order Project,
            Task, Duration, Work location, Activity, and Save as the only
            primary button.
        *   **ProjectSelect:** an MUI Autocomplete that calls
            `getAvailableProjects(date, q)` as the user types, after a pause
            of 150 ms. When nothing matches, it says "No projects match. You
            can only pick projects HR has assigned to you." (TIME-15).
        *   **Keyboard:** picking a value with Enter in Project or Task moves
            focus to the next field. Ctrl+Enter or Cmd+Enter saves
            (NFR-UX-5).
        *   **Defaults:** work location is the last one used (TIME-3).
        *   **On 409 `ENTRY_CHANGED`:** shows the message, with "Show latest"
            and "Apply my changes again".
        *   **On other errors:** shows the server message next to the
            relevant field when `details.fields` names one.
4.  **`frontend/src/components/DurationInput.tsx`**,
    **`frontend/src/features/week/duration.ts`**,
    **`frontend/src/features/week/isoWeek.ts`** (Create)
    *   **Type / Scope:** Component and pure functions.
    *   **Proposed Changes:**
        *   `parseDuration` accepts `8`, `7.5`, `7:45` and `7h45m`, and
            rejects any value that is not a multiple of 15 or is under 15.
        *   `formatDuration(480)` returns `8h 0m`.
        *   `isoWeekOf(date)` and `mondayOf(isoWeek)` handle week 1 and week
            53 edge cases.
5.  **`frontend/src/features/week/duration.test.ts`**,
    **`isoWeek.test.ts`**, **`EntryFormModal.test.tsx`**,
    **`WeekPage.test.tsx`** (Create)
    *   **Type / Scope:** Unit and component tests.
    *   **Proposed Changes:**
        *   Duration parsing table, and ISO week edge cases.
        *   The form sends the correct body and shows server errors.
        *   The keyboard-only path saves in 4 interactions.
        *   The 409 conflict message and its two actions.
        *   The week page shows totals and labels, and does not rely on
            colour alone.

#### ST-10: End-to-end scenario tests and slice wrap-up

1.  **`e2e/playwright.config.ts`**, **`e2e/fixtures.ts`** (Create)
    *   **Type / Scope:** Playwright setup.
    *   **Proposed Changes:**
        *   Runs against `https://localhost:8443`, ignoring the local
            certificate.
        *   A fixture creates its own employees, projects and assignments
            through a test-only SQL helper that connects as `kairos_owner`.
            It does not use the seed (PRD section 8).
        *   Chromium and WebKit, at 1280 px and 360 px widths.
2.  **`e2e/tests/s01-log-normal-day.spec.ts`**,
    **`s04-daily-limit.spec.ts`**, **`s05-unassigned-project.spec.ts`**,
    **`s12-common-projects.spec.ts`**, **`sign-in-out.spec.ts`**,
    **`keyboard-entry.spec.ts`** (Create)
    *   **Type / Scope:** End-to-end tests.
    *   **Proposed Changes:** One file per Slice 1 scenario (CUJs 1 to 4 and
        6), with the scenario ID in the test name. `keyboard-entry` checks
        NFR-UX-5 and visible focus. The 360 px run checks that days stack.
3.  **`README.md`** (Modify)
    *   **Type / Scope:** Documentation.
    *   **Proposed Changes:** Add how to run the end-to-end tests, and a
        table that traces each Slice 1 requirement to its tests (taken from
        the traceability table below).

### Slice 1 traceability

| Requirement | Main files | Tests |
|---|---|---|
| AUTH-1 | `auth/service.py`, `security/passwords.py`, `SignInPage.tsx` | `test_auth.py`, `sign-in-out.spec.ts` |
| AUTH-5 | `auth/service.py`, `auth/dependencies.py` | `test_auth.py` (8-hour expiry, sign-out) |
| TIME-1 | `time_entries/service.py` (`read_week`), `App.tsx`, `WeekPage.tsx` | `test_week.py`, `s01-log-normal-day.spec.ts` |
| TIME-2 | `read_week`, `isoWeek.ts`, `WeekPage.tsx` | `test_week.py`, `isoWeek.test.ts` |
| TIME-3 | `create_work_entry`, `EntryFormModal.tsx` | `test_time_entries.py`, `EntryFormModal.test.tsx` |
| TIME-5 | `time_entries/schemas.py`, migration `CHECK`, `duration.ts` | `test_time_entries.py`, `test_schema.py`, `duration.test.ts` |
| TIME-6 | `create_work_entry` / `update_work_entry`, `lock_day`, `day_totals` trigger | `test_time_entries.py` (S-4), `test_daily_limit_concurrency.py`, `s04-daily-limit.spec.ts` |
| TIME-7 | `update_work_entry`, `delete_work_entry`, `EntryCard.tsx` | `test_time_entries.py` |
| TIME-14 | no rule that forbids duplicates | `test_time_entries.py` |
| TIME-15 | `projects/repository.py`, `ProjectSelect.tsx` | `test_available_projects.py`, `s05-unassigned-project.spec.ts` |
| TIME-16 | `EntryCard.tsx`, `Entry` schema | `WeekPage.test.tsx`, `s01-log-normal-day.spec.ts` |
| PRJ-4 | `available_projects` | `test_available_projects.py` (S-12), `s12-common-projects.spec.ts` |
| PRJ-9 | `available_projects`, `create_work_entry` | `test_available_projects.py`, `test_time_entries.py` (S-5) |
| TIME-17 (early) | `update_entry_if_version`, `EntryFormModal.tsx` | `test_time_entries.py`, `EntryFormModal.test.tsx` |

### Verification Plan

*   **Build:** `make up` starts the stack from a fresh clone with a seeded
    database, and `https://localhost:8443` shows the sign-in page
    (NFR-OPS-1).
*   **Unit and API tests:** `make test-backend` starts a throwaway PostgreSQL
    16 container (`podman run --rm` with tmpfs storage on port 55433), runs
    `pytest`, then removes the container. `make test-frontend` runs Vitest.
*   **End-to-end:** `make test-e2e` runs Playwright against the stack started
    by `make up`. `make test` runs all three.
*   **Static analysis and linting:** `make lint` runs ruff, `mypy --strict`,
    ESLint, `tsc --noEmit`, and the check that the OpenAPI types are up to
    date (NFR-OPS-5).
*   **Test coverage requirement:** backend line coverage of at least 80%,
    enforced by `--cov-fail-under=80` (NFR-OPS-2). Every endpoint has at
    least one success test and one failure test (engineering standards).
*   **Slice 1 done when:** all of the above pass, every row in the
    traceability table has passing tests, and this document matches the code
    (PRD section 11).

--------------------------------------------------------------------------------

*Drafted with EGM*
