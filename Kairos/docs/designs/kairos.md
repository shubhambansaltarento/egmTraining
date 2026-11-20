# Kairos: Employee Time Tracking

*   **Author:** Shubham Bansal
*   **Status:** Proposed
*   **Date:** 2026-11-19

--------------------------------------------------------------------------------

## Overview

Kairos is a web application in which about 5,000 employees report how they
spent their working hours, and about 10 HR administrators maintain employees,
projects, project assignments and holidays and check that everyone has reported
their time. It replaces the current process of spreadsheets and email
reminders.

This document is the design for the whole application as described in
[prd.md](../../prd.md) version 1.3. It has two levels of detail:

*   The **Technical Plan** covers the architecture for PRD Slices 1 to 5. It
    states a decision and a reason for each of the eight questions in PRD
    section 10, a data dictionary, the API contracts, and sequence diagrams for
    scenarios S-1, S-2 and S-5.
*   The **Detailed Implementation Plan** covers PRD Slice 1 only (sign in and
    log time). It lists every file to create and why. Later slices are listed
    in Scope by slice.

The technology stack is fixed by PRD section 7.1: PostgreSQL 16, Python 3.12
with FastAPI, React 18 with TypeScript, identity built into the application,
OpenAPI 3.1, Podman with Podman Compose, and k6 or Locust for load tests. The
files in [context/](../../context/) (design theme and engineering standards) are
treated as constraints.

### Scope by slice

The PRD gives each requirement a priority: **M** (must have), **S** (should
have) or **C** (could have). PRD section 9 puts the M requirements in Slices 1
to 5 and every S and C requirement in **Stretch**, which is work after Slice 5
with no fixed order.

| Slice | PRD requirements | Where this design departs from PRD section 9 |
|---|---|---|
| 1. Sign in and log time | AUTH-1, AUTH-5, TIME-1 to TIME-3, TIME-5 to TIME-7, TIME-14 to TIME-16, PRJ-4, PRJ-9 | Also delivers the TIME-17 version check (Q6), without the conflict dialog. |
| 2. HR master data | EMP-1 to EMP-5, EMP-7, EMP-8, PRJ-1 to PRJ-3, PRJ-5 to PRJ-8, PRJ-10, AUTH-2 to AUTH-4, AUTH-6 | None. |
| 3. Holidays and leave | HOL-1 to HOL-6, TIME-4, TIME-8, TIME-9, TIME-11, TIME-12 | None. HOL-7 (Stretch) holds with no extra work because of Q1. |
| 4. Reports and audit | RPT-1, RPT-2, RPT-4, AUD-1 | `audit_log` is created with INSERT and SELECT only for the API user, which is part of AUD-3. |
| 5. Performance proof | NFR-PERF-1 to NFR-PERF-9 | The export part of NFR-PERF-4 cannot be tested here, because exports are Stretch. See Open Questions. |
| Stretch | All S and C requirements: AUTH-7, AUTH-8, EMP-6, HOL-7, HOL-8, PRJ-11 to PRJ-14, TIME-10, TIME-13, TIME-17, TIME-18, RPT-3, RPT-5, RPT-6, AUD-2, AUD-3 | Designed here but not planned: TIME-17 (Q6), RPT-5 (Q5), AUD-2 and AUD-3 (Q8), RPT-6 (The Problem). The others have no design. |

Each slice gets its own detailed implementation plan before it is built.
Section 8 of the Technical Plan says how each slice builds on Slice 1.

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

Slices 1 to 4 address these. Employees get one weekly screen in which they can
only choose projects HR has opened to them (Slice 1). Designated holidays are
added to every employee's week automatically (Slice 3). HR sees in one screen
who has an incomplete week (Slice 4), and every change to master data is
recorded (Slice 4). Closing a month (RPT-6) and the audit of time entries
(AUD-2) are Stretch.

Four PRD rules shape the decisions below:

*   TIME-8: no work entries for a future date; leave up to 60 days ahead.
*   TIME-9: an employee cannot add, edit or delete an entry dated more than 14
    days ago.
*   HOL-2: HR can edit or delete a holiday unless its date is in a locked
    month.
*   RPT-6 (Stretch): HR can lock and unlock a calendar month.

TIME-10 (Stretch) adds that only HR, by unlocking, can change entries in a
locked month.

The PRD leaves some business rules open or in conflict. This design settles
them as follows (the author accepted these defaults on 2026-11-19):

*   **A holiday added on a day that is already full.** HR is never blocked from
    adding a holiday. If an employee's day goes over 24 hours as a result, the
    existing entries stay and the day is flagged. The 24-hour limit (TIME-6) is
    checked only when an employee saves an entry. This follows the same
    principle as PRJ-10: HR actions never change existing entries. On such a
    day an employee can still shorten or delete entries: a save is rejected
    only when it would leave the day over 24 hours *and* higher than before
    the save.
*   **Leave on a designated holiday** is rejected, because the holiday already
    covers the day.
*   **Partial weeks.** Expected hours count only weekdays on or after the
    employee's date of joining, and holidays before that date are not shown.
    Without this rule every new joiner appears as short in the missing-time
    report (RPT-2).
*   **Unlocking a month older than 14 days** lets HR change holidays in that
    month (HOL-2). It does not let anyone change time entries, because TIME-9
    still applies to employees and PRD section 3.3 gives HR read-only access
    to other employees' entries.

## Goals

Product goals, taken from PRD section 2.1:

*   **G-1:** An employee can log a full 8-hour day in under 30 seconds, and
    with the keyboard alone in no more than 6 key interactions after the add
    form opens (NFR-UX-5; see Open Questions for how they are counted).
*   **G-2:** HR needs no spreadsheet for any master-data task.
*   **G-3:** HR can list every employee with an incomplete week in one screen.
*   **G-4:** The performance targets in PRD section 7.2 are met at 500
    concurrent users with 20 million time entries loaded, on the reference
    environment of 2 application instances and 1 database instance.

The **Friday peak** in this document is the NFR-PERF-1 load at its busiest:
500 concurrent users sending 50 to 100 requests a second. PRD section 7.2 says
most employees report on Friday afternoon and Monday morning, and again on the
last working day of the month. The **reference environment** (NFR-PERF-7) is 2
application instances with 2 vCPU and 4 GB each, and 1 database instance with
4 vCPU and 16 GB.

Design goals for this document:

*   State a decision and a reason for each of the eight questions in PRD
    section 10.
*   Make every requirement in Slice 1 traceable from this document to a file
    and to at least one test (PRD section 11).
*   Choose Slice 1's tables and API shapes so that Slices 2 to 5 only add
    tables, columns, `CHECK` values and response fields. They never make a
    destructive change to Slice 1's tables or a breaking change to its API. In
    particular, the 24-hour check must already allow for holiday hours
    (Slice 3), entries must already carry a version number for lost-update
    protection (TIME-17), and writes must already pass the actor the audit
    trail needs (Slice 4).
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
*   **S and C requirements without a design.** They are listed under Stretch
    in Scope by slice. Five are designed here because a PRD section 10
    question or a PRD conflict depends on them: TIME-17 (question 6), RPT-5
    (question 5), AUD-2 and AUD-3 (question 8), and RPT-6 (the HOL-2 conflict
    in The Problem). The others get no design.
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
*   **Docker tooling.** Docker is not permitted for compliance reasons (PRD
    section 7.1, Packaging). The design uses
    `podman-compose` (the separate Python tool), because `podman compose` can
    hand work to the `docker-compose` binary.

## Open Questions

The eight PRD section 10 questions and the open library choices (Q1 to Q9)
were decided by the author on 2026-11-19. The options and trade-offs are in
[decision_options.md](../../.egm/decision_options.md), and the decisions are in the
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
*   **Does the Docker rule cover Docker Hub images?** PRD section 7.1 forbids
    Docker but does not say whether images from Docker Hub (`docker.io`) may
    be used. Every default image in this design comes from there. If they may
    not, the image variables must point at an approved registry before
    `make up` can work. Owner: the author, to confirm with compliance before
    ST-01.
*   **Exports and the Slice 5 load test (NFR-PERF-4).** PRD section 9 puts
    NFR-PERF-1 to NFR-PERF-9 in Slice 5 but RPT-5 exports in Stretch, after
    Slice 5. One of two choices applies:
    *   Slice 5 proves NFR-PERF-4 for RPT-1 to RPT-3 only, and exports get
        their own load run when they are built.
    *   Exports move ahead of Slice 5.

    Owner: the author, before Slice 5 is planned.

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
*   **Process model:** each instance runs one uvicorn process with one event
    loop (no `--workers`). Each process has these limits:
    *   a pool of 20 database connections plus up to 10 more when busy;
    *   1 password hash at a time (Q2);
    *   1 export at a time with its own pool of 2 connections (Stretch, Q5).

    Two instances therefore open at most 2 × (30 + 2) = 64 connections.
    Migrations, the seed and a database shell add a few more, which stays
    under PostgreSQL's default `max_connections` of 100. Slice 5 decides
    whether a second process per instance is worth adding. If it is, the pool
    sizes are halved.
*   **db:** PostgreSQL 16. It holds all data and also does the work that
    usually needs extra infrastructure: session lookup (Q2), per-day advisory
    locks for the 24-hour rule (Q3), and the audit triggers (Q8).
*   **Start-up (NFR-OPS-1):** `make up` runs these steps in order. It starts
    `db` and waits until `pg_isready -h 127.0.0.1 -U kairos_owner -d kairos`,
    run with `podman exec`, succeeds over TCP, retrying every second for up to
    60 seconds. A check over the Unix socket is not enough, because the
    PostgreSQL image runs its init scripts on a temporary server that listens
    only on the socket. It runs `alembic upgrade head`
    and then the seed script as one-off containers (`podman-compose run
    --rm`). Then it starts `api` and `proxy`. The order lives in the Makefile
    instead of compose `depends_on` conditions, because `podman-compose`
    handles those conditions inconsistently across versions. Docker is never
    invoked.

### 2. Backend structure

The layers follow [engineering-standards.md](../../context/engineering-standards.md):
the **router** checks input against Pydantic schemas and calls a service. The
**service** holds every business rule. The **repository** holds every SQL
statement. Routers contain no rules and repositories make no decisions.

*   **Database access:** SQLAlchemy 2.0 in async mode with the `asyncpg`
    driver. Each API process has a pool of 20 connections plus up to 10 extra
    when busy. Every request that writes runs in one transaction.
*   **Migrations:** Alembic, with hand-written migrations (NFR-DATA-2).
    Triggers, functions and constraints are part of the migrations, not
    created at start-up.
*   **Request pipeline**, in this order for every API request:
    1.  *Correlation ID:* read `X-Request-ID` from the request, or create a
        UUID if it is missing. Put it on every log line and echo it in the
        response. Every database transaction, including read-only ones,
        starts with one statement that sets `application_name` to
        `kairos:<request id>` (step 5). PostgreSQL's `log_line_prefix`
        includes `%a`, so database log lines can be matched to the request
        (NFR-OPS-3). The SQL text never changes, so prepared statements are
        reused.
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
    5.  *Actor and request ID:* each transaction starts with
        `SELECT set_config('kairos.actor_id', :actor, true),
        set_config('application_name', :app_name, true)` with bound
        parameters. Read-only transactions pass an empty actor. Setting the
        actor happens from Slice 1, so the audit triggers added in Slice 4
        have an actor from the first day.
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
    "details" }`. `code` is a fixed upper-case identifier (listed in the error
    table in section 6).
    `message` is a short sentence that says what went wrong and what to do
    next (NFR-UX-4), and the UI shows it as written. `details` holds
    field-level validation problems or the values behind a rule failure.
    Unexpected errors return `INTERNAL_ERROR` with the correlation ID, and the
    stack trace goes only to the log.
*   **Logs:** JSON lines on standard output. A filter replaces the value of
    any key named `password`, `token`, `cookie`, `set-cookie` or
    `authorization` with `[REDACTED]` before writing. Matching is exact and
    ignores case, at any depth, including request and response header maps
    (NFR-SEC-1).
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
    `If-Match`, parses the error shape, and on a 401 `SESSION_ENDED` from any
    call except sign-in sends the user to `/sign-in`. TypeScript types for requests and responses are generated from
    the backend's OpenAPI document with `openapi-typescript`, so the frontend
    and the API cannot drift apart unnoticed.
*   **Components:** MUI, with a theme built from
    [design-theme.md](../../context/design-theme.md). The top bar is `#111111`,
    the primary action `#1F2A44`, and work, leave, holiday and warning use
    `#00A67E`, `#B45309`, `#5B21B6` and `#B91C1C`. The font is Inter at 14 px,
    with a condensed bold face for hours. Every entry card shows a text label
    and an icon for its kind, so colour is never the only signal (TIME-11,
    NFR-UX-3). The add and edit form opens in a centred modal, and Save is its
    only primary button.
*   **Layout:** seven equal columns from Monday to Sunday at 768 px and above.
    Below 768 px the days stack vertically, down to 360 px (NFR-UX-2).
*   **Keyboard path (NFR-UX-5):** the form opens with Project focused. A Date
    field comes first in the form and defaults to the column the form was
    opened from; changing it moves the entry to another day (TIME-7).
    Location defaults to the one used last, or Office when there is none. The
    path is: (1) type part of a code or name and press
    Enter to pick, which moves focus to Task, (2) type and press Enter, which
    moves focus to Duration, (3) type a duration such as `8` or `7:45`, then
    (4) press Ctrl+Enter or Cmd+Enter to save. That is 4 interactions, within
    the limit of 6 under the counting rule listed in Open Questions.
*   **Duration input (TIME-5):** accepts `8`, `7.5`, `7:45` and `7h45m`.
    Anything that is not a whole multiple of 15 minutes, or is below 15
    minutes, is rejected in the form with the same message the API returns.

### 4. Decisions on PRD section 10 (Q1 to Q8) and open libraries (Q9)

All decided by the author on 2026-11-19. Q3 and Q4 were revised after the
design review on the same day. The options that were turned down are
in [decision_options.md](../../.egm/decision_options.md) and are summarised
under Alternatives Considered.

Timings, sizes and rates in this section are estimates from
`decision_options.md`, not measurements. The Slice 5 load test confirms them.

**Q1. Holiday entries are worked out when a week is read.**
*PRD question 1: stored per employee or derived on read, and what each costs?*

Holidays are stored once in `holidays`. One database function, created in the
Slice 1 migration, is the only place that turns working days and holidays into
minutes:

*   `employee_calendar(employee_ids BIGINT[], from_date DATE, to_date DATE)`
    returns one row per employee and date: `employee_id`, `day`,
    `expected_minutes` (480 on a weekday on or after the employee's date of
    joining, otherwise 0), `holiday_minutes` (480 for a weekday holiday, 0 for
    a weekend holiday (HOL-6) or no holiday), and `holiday_name` (null when
    there is no holiday). Holidays before the date of joining are not
    returned. It is `STABLE` and reads `employees` and `holidays`.

The week read and the 24-hour check both call it, so expected and holiday
minutes are defined in one place. Rule for later slices: holiday and expected
minutes come only from this function, and project, task and kind breakdowns
sum `time_entries` directly. Each report's plan states whether it includes
deactivated employees: RPT-2 does not, and RPT-3 does (S-7). **Cost:** adding
a holiday writes one row. Reading a week adds one indexed lookup on a table of
about 15 rows a year. A weekend holiday returns 0 minutes but is still shown
in the calendar (HOL-6). Work logged on a holiday is added on top of the
holiday hours, which also satisfies HOL-7 (Stretch) at no extra cost.

**Q2. Random session token checked in the database on every request.**
*PRD question 2: how is the session carried, and how is revocation within 1
minute (AUTH-6) met with a stateless tier?*

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
*   Sign-in deletes that employee's expired sessions, and deactivation deletes
    all of an employee's sessions (Slice 2). Nothing else deletes them in
    Slice 1.
*   **Password hashing:** argon2id via `argon2-cffi`, with 3 passes, 64 MiB
    and 1 lane. Hashing runs in a worker thread so that it does not block the
    event loop.
    *   **Cap:** each API process runs at most 1 hash at a time. A sign-in
        waits up to 2 seconds for its turn, then returns 503 `SIGN_IN_BUSY`.
    *   **Capacity:** at about 50 to 80 ms per hash, two instances handle
        about 25 to 40 sign-ins a second. The planned load is about 8 a
        second: 500 sign-ins in 60 seconds (NFR-PERF-3).
    *   **CPU:** the cap of 1 leaves each instance's second vCPU for the event
        loop. This relies on `argon2-cffi` releasing the Python GIL during the
        hash, which ST-04 checks (see Risks).

**Q3. A per-day advisory lock and a fresh sum.**
*PRD question 3: how is the 24-hour limit enforced for concurrent saves
(NFR-DATA-1)?*

Creating, editing or moving an entry runs in one transaction:

1.  `lock_day` runs `SELECT pg_advisory_xact_lock(:employee_id, :entry_date -
    DATE '2000-01-01')`, the two-integer form, which locks that employee's
    day. Employee IDs stay far below 2^31. A concurrent save for the same
    employee and date waits here. PostgreSQL releases the lock at commit or
    rollback.
2.  `day_entry_minutes` sums `duration_minutes` of that employee's entries on
    that date through the `(employee_id, entry_date)` index (a handful of
    rows). The service adds the holiday minutes from `employee_calendar` (Q1).
3.  The service applies the new duration (for an edit on the same date, minus
    the old one). If the result is over 1,440 minutes and higher than the
    day's total before the save, it rolls back with 409
    `DAILY_LIMIT_EXCEEDED`.
4.  It inserts or updates the entry. Commit.

An edit that moves an entry to another date locks both days, earliest date
first, so two such edits cannot deadlock, and checks only the new date. Only
`time_entries/service.py` writes `time_entries`, and every save in it goes
through `lock_day` (see Risks). HR never takes this lock when adding a holiday
(see The Problem).

**Q4. One composite index, no pre-computed totals, no partitioning.**
*PRD question 4: which indexes keep a week read and the missing-time report
fast at 20 million rows, and is partitioning or pre-aggregation needed?*

| Table | Index | Serves |
|---|---|---|
| `time_entries` | `(employee_id, entry_date)` | Reading a week (about 10 to 20 rows), the 24-hour sum, edit and delete by owner, RPT-1 |
| `time_entries` | `(entry_date, project_id)` (added in Slice 4) | RPT-2, RPT-3 and its export |
| `assignments` | `(employee_id, start_date)` and the exclusion constraint below | Project selector (TIME-15), PRJ-9 check |
| `sessions` | primary key `token_hash`, plus `(employee_id)` | Every request, revocation |

RPT-2 (Slice 4) sums about 70,000 entries for one week through the
`(entry_date, project_id)` index. `decision_options.md` estimates this at 50
to 150 ms against a 2-second target. If the Slice 5 load test shows it is too
slow, a summary table is added then.

No partitioning. At 20 million rows the table is about 3 to 4 GB and its
indexes about 1 GB, which mostly fits in 16 GB of memory. Every query reaches
a small range through an index. Partitioning would add maintenance work, and
whether it is needed will be decided by the Slice 5 load test, not now.

**Q5. CSV exports are streamed in the request, with a cap on concurrent
exports (Stretch).**
*PRD question 5: how do CSV exports run without slowing interactive requests
(NFR-PERF-4)?*

Export endpoints return a streamed response that reads a server-side cursor
in batches of 2,000 rows. They use a separate pool of 2 database connections
and a 30-second statement timeout. Each API process allows 1 export at a time
and otherwise returns 429 `EXPORT_BUSY` ("Another export is running. Try
again in a minute."). Cells that start with `=`, `+`, `-` or `@` get a leading
`'` so spreadsheets cannot run them as formulas. Caddy's proxy timeout is set
to 60 seconds.

Exports are built after Slice 5, because RPT-5 is an S requirement. Their plan
adds a load run that holds a 100,000-row export during the NFR-PERF-1 load and
checks NFR-PERF-2 (see Open Questions). That run also decides whether CSV
formatting moves to a worker thread or process. Three rules apply when
exports are built:

*   Tab and carriage return are added to the characters that get a leading
    `'`.
*   An error after streaming has started closes the connection without the
    final chunk, so the browser reports a failed download instead of saving
    a short file.
*   The service enforces a 30-second deadline for the whole export, because
    `statement_timeout` applies to each fetch separately.

**Q6. An integer version number on each entry.**
*PRD question 6: how is lost-update protection (TIME-17) built and shown?*

`time_entries.version`
starts at 1. Responses return it in the body and as `ETag: "<version>"`. PUT
and DELETE require `If-Match: "<version>"` and return 428
`VERSION_REQUIRED` without it. The update runs `... WHERE id = :id AND
employee_id = :me AND version = :v`. If no row changes and the entry still
exists, the response is 409 `ENTRY_CHANGED`, with the current entry in
`details.entry`. The modal then shows the `ENTRY_CHANGED` message and replaces
the form values with `details.entry`, so the user sees the latest version and
can edit again. The version check ships in Slice 1, because adding a required
header to PUT and DELETE later would break existing clients. This delivers
the part of TIME-17 that the PRD requires early. A dialog that offers to
re-apply the user's edit is Stretch.

**Q7. No server cache.**
*PRD question 7: is a cache needed for NFR-PERF-2, and how is it invalidated?*

Every request reads PostgreSQL. `GET /tasks` and
`GET /holidays` return an `ETag` so browsers can reuse them. TanStack Query
keeps the current and adjacent weeks in the browser and reloads them when the
screen opens, which meets HOL-5 ("next time the view is loaded"). The Slice 5
load test decides whether a cache is ever added. Any cache added then must say
how HOL-5 still holds.

**Q8. Database triggers with a required actor (Slice 4 for master data,
Stretch for time entries).**
*PRD question 8: how is the audit trail written so a new code path cannot
skip it?*

One PL/pgSQL function, `audit_row()`, is attached as an `AFTER INSERT OR
UPDATE OR DELETE FOR EACH ROW` trigger in Slice 4 to `employees`, `holidays`,
`projects` and `assignments` (AUD-1). Stretch attaches it to `time_entries`
(AUD-2) and `period_locks` (RPT-6). It reads `current_setting('kairos.actor_id',
true)` and raises an error if it is empty, so a write with no actor fails
instead of going unaudited. It writes old and new values as JSON for master
data (AUD-1), and only actor and time for time entries (AUD-2). The
`kairos_app` database user that the API connects as has INSERT and SELECT on
`audit_log` but no UPDATE or DELETE (AUD-3). Migrations and seeding connect as
`kairos_owner` and set the actor to `system`. A test lists every table in the
audited set and fails if one is missing its trigger. `employee_credentials`,
`employee_preferences` and `sessions` are not audited: they hold secrets or
working state, not records.

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

**`employee_credentials`** (Slice 1. Not audited.)

| Column | Type | Rules |
|---|---|---|
| `employee_id` | BIGINT | Primary key, foreign key to `employees`. |
| `password_hash` | TEXT | argon2id encoded string. Never logged, never returned, never copied into `audit_log`. |

Slice 2 adds `must_change_password` (AUTH-2), `failed_sign_in_count` and
`locked_until` (AUTH-4) to this table. Sign-in reads and writes only this
table and `sessions`, so it never writes an audited row and needs no actor.
This follows the same reason as `employee_preferences`.

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

Stretch adds `is_active` (PRJ-13).

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
| `kind` | TEXT | `WORK` in Slice 1. Slice 3 widens the `CHECK` to add `LEAVE`. Holidays are never stored here (Q1). |
| `project_id` | BIGINT | Foreign key to `projects`. Required when `kind = 'WORK'`, null otherwise. |
| `task_id` | BIGINT | Foreign key to `tasks`. Required when `kind = 'WORK'`, null otherwise. |
| `work_location` | TEXT | `OFFICE`, `CLIENT_SITE` or `WFH`. Required when `kind = 'WORK'`, null otherwise. |
| `activity` | TEXT | Optional, at most 500 characters (TIME-3). |
| `duration_minutes` | SMALLINT | `CHECK (duration_minutes >= 15 AND duration_minutes <= 1440 AND duration_minutes % 15 = 0)` (TIME-5). |
| `version` | INTEGER | Starts at 1, goes up by 1 on each update (Q6). |
| (constraint) | | A `CHECK` ties each `kind` to its required and forbidden columns as listed above. |
| (index) | | `(employee_id, entry_date)`. From Slice 4 also `(entry_date, project_id)`. |

Slice 3 adds `leave_type` (`CASUAL`, `SICK`, `EARNED`, `UNPAID`) and the
240-or-480 rule for leave (TIME-4).

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
| `employee_calendar(employee_ids, from, to)` | 1 | Expected and holiday minutes per employee per date (Q1). |
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
    a valid session: 401 `SESSION_ENDED`.
*   Writes need the header `X-Kairos-CSRF: 1`. Without it: 403 `CSRF_FAILED`.
*   Bad input: 422 `VALIDATION_FAILED`.
*   Every response carries `X-Request-ID`.

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

Later slices add `ACCOUNT_LOCKED` (423, Slice 2), `DATE_NOT_EDITABLE` (403,
Slice 3) and `EXPORT_BUSY` (429, Stretch).

**Slice 1 endpoints**

| Method and path | Request | Success | Failures |
|---|---|---|---|
| `POST /auth/sign-in` | `{ email, password }` | 200 `{ employee: Me }` and `Set-Cookie` | 401 `INVALID_CREDENTIALS` (the same reply for unknown email, wrong password and deactivated account, so it does not reveal which accounts exist). 503 `SIGN_IN_BUSY`. 423 `ACCOUNT_LOCKED` from Slice 2. |
| `POST /auth/sign-out` | none | 204, cookie cleared | 401 |
| `GET /me` | none | 200 `Me` | 401 |
| `GET /me/week?date=YYYY-MM-DD` | any date in the week. Defaults to today in IST. | 200 `Week` | 401, 422 |
| `GET /me/available-projects?date=YYYY-MM-DD&q=text` | date required. `q` matches code prefix or part of the name. | 200 `[ AvailableProject ]`, at most 50, common projects first | 401, 422 |
| `GET /tasks` | none | 200 `[ { id, name } ]` all tasks, with `ETag` | 401 |
| `POST /me/time-entries` | `NewWorkEntry` | 201 `Entry`, with `ETag` | 403 `PROJECT_NOT_AVAILABLE` (S-5). 409 `DAILY_LIMIT_EXCEEDED` (S-4). 422 `VALIDATION_FAILED`. |
| `PUT /me/time-entries/{id}` | `NewWorkEntry` and `If-Match` | 200 `Entry`, new `ETag` | 404 `ENTRY_NOT_FOUND`, 409 `ENTRY_CHANGED`, 409 `DAILY_LIMIT_EXCEEDED`, 403 `PROJECT_NOT_AVAILABLE`, 428 `VERSION_REQUIRED`, 422 (including a malformed `If-Match`) |
| `DELETE /me/time-entries/{id}` | `If-Match` | 204 | 404 `ENTRY_NOT_FOUND`, 409 `ENTRY_CHANGED`, 428 `VERSION_REQUIRED`, 422 (malformed `If-Match`) |
| `GET /health/live` | none, no auth | 200 `{ "status": "ok" }` if the process is running | none |
| `GET /health/ready` | none, no auth | 200 if `SELECT 1` succeeds within 1 s | 503 `NOT_READY` |

**Shapes**

```text
Me               { id, employeeCode, fullName, workEmail, role, dateOfJoining,
                   lastWorkLocation | null }

Week             { weekStart, weekEnd, isoYear, isoWeek,
                   expectedMinutes, totalMinutes, complete,
                   days: [ Day x 7, Monday first ] }

Day              { date, weekday, expectedMinutes, totalMinutes,
                   entries: [ Entry | HolidayEntry ] }

Entry            { id, kind: "WORK", entryDate, durationMinutes,
                   project: { id, code, name, label },
                   task: { id, name },
                   workLocation, activity | null, version }

HolidayEntry     { id: null, kind: "HOLIDAY", entryDate, name,
                   durationMinutes: 480 or 0 }

NewWorkEntry     { entryDate, projectId, taskId, durationMinutes,
                   workLocation, activity? }

AvailableProject { id, code, name, label, type: "CLIENT" | "COMMON",
                   assignmentStart | null, assignmentEnd | null }
```

*   **Derived fields**, all worked out by the server:
    *   `weekday` is the ISO day number, 1 (Monday) to 7 (Sunday).
    *   `isoYear` and `isoWeek` follow ISO 8601. `weekStart` is the Monday and
        `weekEnd` the Sunday.
    *   `Day.expectedMinutes` comes from `employee_calendar`.
    *   `Day.totalMinutes` is the sum of the day's entry durations plus its
        holiday minutes.
    *   `Week.expectedMinutes` and `Week.totalMinutes` are the sums over the
        7 days.
    *   `complete` is `Week.totalMinutes >= Week.expectedMinutes` (PRD
        glossary). Future weekdays count, so the current week stays
        incomplete until it is filled.
    *   Only `Entry` responses (POST 201, PUT 200) carry an `ETag`.
*   **Fields added by later slices**, without changing existing ones:
    `Me.mustChangePassword` (Slice 2); `Day.flags` (`BELOW_EXPECTED` for dates
    up to today, TIME-12; `OVER_DAILY_LIMIT` for a day a holiday pushed past
    24 hours), `Entry.editable`, `Entry.leaveType` and kind `LEAVE`
    (Slice 3). In Slice 1 the UI offers edit and delete on every `WORK` entry
    and on no `HOLIDAY` entry.
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
    API->>DB: BEGIN, set_config(actor, request ID)
    API->>DB: available projects query filtered to this project
    API->>DB: pg_advisory_xact_lock(priya, today)
    API->>DB: SUM(duration_minutes) for priya, today
    API->>DB: employee_calendar([priya], today, today)
    Note over API: 0 + 0 + 480 ≤ 1440, so allowed
    API->>DB: INSERT time_entries (version 1)
    API->>DB: UPSERT employee_preferences (OFFICE), COMMIT
    API-->>UI: 201 Entry, ETag "1"
    UI->>API: GET /me/week?date=today (query reloaded)
    API->>DB: entries for week + employee_calendar
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
    API->>DB: BEGIN, set_config actor, INSERT holidays (unique date)
    Note over DB: audit_row() writes audit_log (from Slice 4)
    API->>DB: COMMIT
    API-->>UI: 201 Holiday
    Note over DB: One row written. No per-employee rows.

    Priya->>UI: Opens the week of that Monday
    UI->>API: GET /me/week?date=that Monday
    API->>DB: session lookup
    API->>DB: time_entries for 7 days (index employee_id, entry_date)
    API->>DB: employee_calendar([priya], Monday, Sunday)
    DB-->>API: Monday, "Founders' Day", 480
    Note over API: Merge as HolidayEntry (id null).<br/>Monday total and week total include 480.
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
    API->>DB: BEGIN, available projects query filtered to Borealis
    DB-->>API: no row
    API->>DB: ROLLBACK (nothing written)
    API-->>Priya: 403 {code: PROJECT_NOT_AVAILABLE,<br/>message: "You cannot log time to this project on this date. Pick a project from the list."}
```

### 8. How Slices 2 to 5 build on this

*   **Slice 2** adds HR routers on the same three layers, the AUTH-2 to AUTH-4
    columns, added to `employee_credentials`, deactivation that deletes
    sessions (Q2), and the EMP-7 date-of-joining check in the entry service.
    Last-HR-admin protection (EMP-8) takes a row lock on the HR admin count,
    so two demotions at once cannot both pass. The department question in
    Open Questions is decided first.
*   **Slice 3** adds holiday endpoints and seed data. It needs no change to the
    week read or the 24-hour check, because both already call
    `employee_calendar`. It also adds leave entries (TIME-4) with the
    `leave_type` column and the `LEAVE` kind, the date rules (TIME-8, TIME-9)
    with the `editable` field and the 403 `DATE_NOT_EDITABLE` error, and the
    day flags.
*   **Slice 4** adds `audit_log` and the `audit_row()` triggers on master data
    (Q8), the report endpoints (expected and holiday minutes from
    `employee_calendar`; each report states whether it includes deactivated
    employees), the HR read-only week
    (the same service as `/me/week`, given an employee ID after an HR role
    check), and the `(entry_date, project_id)` index.
*   **Slice 5** adds the volume seed (20 million entries, NFR-PERF-8), the k6
    scripts, `/metrics`, and tuning. Tuning changes only indexes,
    connection-pool sizes and PostgreSQL settings, unless the load test proves
    a design decision above wrong. If that happens, this document is updated
    first.

## Alternatives Considered

Full mechanisms and costs for each option are in
[decision_options.md](../../.egm/decision_options.md). Each rejection below was decided by
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
*   **Reason for Rejection:** It turns a one-row change
    into a 5,000-row change and adds a backfill step that can be forgotten.
    Reading the holiday table on demand costs under 1 ms per week read.

### Q2-B: Short-lived signed token (JWT) plus a refresh token

*   **Pros:** Normal requests need no database read to check who the user is.
*   **Cons:** It adds refresh-token rotation, frontend retry logic and
    signing-key management, plus about 8 refresh calls a second. Revocation
    takes up to the full token lifetime, so it meets AUTH-6 only if the
    lifetime stays at 60 seconds or less. Signing out does not end the access
    token until it expires.
*   **Reason for Rejection:** Every request already reads
    PostgreSQL, so one primary-key lookup costs under 0.5 ms. The session table
    gives revocation that takes effect at once, with less code.

### Q3-A: A per-day totals row, locked by the save

*   **Pros:** A trigger keeps the total, so a write path that skips the
    service check still cannot push a day's entry minutes past 1,440. RPT-2
    could read the stored totals.
*   **Cons:** The database `CHECK` cannot include holiday minutes, so it does
    not guard TIME-6 as a whole. The trigger must handle a missing row and
    date moves, and the table can drift from the entries. It adds a table, a
    trigger and a second write to every save.
*   **Reason for Rejection:** The extra protection is partial, and the RPT-2
    benefit rests on an estimate that Slice 5 will measure. Chosen first on
    2026-11-19, then replaced by Q3-B after the design review.

### Q3-C: SERIALIZABLE transactions with a retry

*   **Pros:** No locks to remember.
*   **Cons:** PostgreSQL can report conflicts that aren't real, which causes
    retries during the Friday peak. Any write path that doesn't retry shows
    errors to users and counts against the 0.1% error budget (NFR-PERF-5). One
    path at a lower isolation level quietly turns the protection off.
*   **Reason for Rejection:** The protection depends on
    every caller's isolation level and retry code, both of which are easy to
    get wrong without anyone noticing.

### Q4-B: Monthly range partitioning of `time_entries`

*   **Pros:** Range reports scan only the months they need, and old months can
    be detached for archiving.
*   **Cons:** Partitions for future months must be created ahead of time, or
    inserts fail. Unique constraints must include the date. Foreign keys that
    point at entries become awkward.
*   **Reason for Rejection:** At 20 million rows (about
    4 to 5 GB including indexes) every query already reaches a small range
    through an index. The Slice 5 load test will show whether partitioning is
    needed.

### Q4-A: Pre-computed per-day totals (`day_totals`)

*   **Pros:** RPT-2 reads at most 35,000 total rows for a week instead of
    about 70,000 entries.
*   **Cons:** It needs the totals table and trigger that Q3-A was rejected
    for.
*   **Reason for Rejection:** The raw weekly sum is estimated at 50 to 150 ms
    against a 2-second target for about 10 HR users. Slice 5 measures it, and
    a summary is added only if that test fails.

### Q5-B: Background job table and a worker process

*   **Pros:** CSV work runs completely apart from the API processes.
*   **Cons:** It adds an extra service, somewhere to keep finished files, a
    cleanup job and a progress screen. A stopped worker leaves exports
    waiting forever with no alert. The reference environment has no spare
    machine for the worker.
*   **Reason for Rejection:** A 100,000-row export takes
    about 1 second of CPU. A limit of one export per instance and a separate
    connection pool keep it away from interactive requests, without new
    infrastructure. The export load run checks this.

### Q5-C: Stream PostgreSQL `COPY ... TO STDOUT`

*   **Pros:** The least Python CPU and the fastest.
*   **Cons:** Protection against CSV injection and the column formatting rules
    would have to be written in SQL, where a missed case is harder to test.
*   **Reason for Rejection:** Escaping in Python is easier
    to test, and speed is not the limit at 100,000 rows.

### Q6-B: Last-updated timestamp as the version

*   **Pros:** No extra column.
*   **Cons:** Two updates in the same clock tick, or a timestamp that loses
    precision when converted to or from JSON, make conflicts go undetected or
    appear when there isn't one.
*   **Reason for Rejection:** An integer counter cannot be
    misread.

### Q7-B: In-memory cache in each instance, cleared by LISTEN/NOTIFY

*   **Pros:** Saves a small query per request for data that rarely changes.
*   **Cons:** It needs a long-lived connection per instance and code to
    reconnect it. If the connection drops without anyone noticing, an instance
    serves stale holidays, which breaks HOL-5.
*   **Reason for Rejection:** There is no measured need
    yet. The Slice 5 load test decides.

### Q8-B: ORM session events

*   **Pros:** The audit logic is in Python and easy to unit-test.
*   **Cons:** Bulk `update()` and `delete()` statements and raw SQL don't fire
    ORM events, so they skip the audit silently.
*   **Reason for Rejection:** This is exactly the "new
    code path" risk that PRD question 10.8 asks the design to close.

### Q8-C: Explicit audit calls in a base repository class

*   **Pros:** Simple and visible in the code.
*   **Cons:** Nothing stops a repository method that bypasses the base class.
*   **Reason for Rejection:** The protection depends on
    every developer remembering to use the base class.

### Infrastructure and tooling rejected

*   **Redis for sessions or caching.** The reference
    environment (NFR-PERF-7) has no machine for it. PostgreSQL handles session
    lookup at this load (about 100 requests a second), and nothing has shown a
    cache is needed.
*   **Keycloak.** PRD section 7.1 fixes identity as built
    into the application. Keycloak would also need user, role and
    deactivation data kept in step between the two systems (EMP-1, EMP-4,
    AUTH-6).
*   **`podman compose` (the built-in subcommand).** It can
    hand work to the `docker-compose` binary, and PRD section 7.1 forbids
    Docker. `podman-compose` (the separate Python tool) is pinned
    instead.
*   **Compose `depends_on` conditions for start-up order.** `podman-compose` supports them inconsistently across versions. The
    Makefile runs the steps in order instead.
*   **SQLite for backend tests.** It cannot test row
    locks, exclusion constraints or triggers, which Q3, PRJ-8 and Q8 depend
    on.
*   **Synchronous SQLAlchemy with `psycopg` 3.** Under
    the Friday peak, the default limit of 40 worker threads would cap how many
    requests an instance can serve at once. With async, the size of the
    connection pool is the only limit.
*   **bcrypt (cost 12).** At about 250 ms per hash, a
    burst of 500 sign-ins in 60 seconds leaves little spare CPU on 4 vCPU.
    argon2id at about 50 to 80 ms leaves room and is the current OWASP
    recommendation.
*   **shadcn/ui.** MUI ships the autocomplete and modal
    the weekly view needs, and its `TextField` wraps the native date input.
    The colours in design-theme.md can be applied through MUI's theme.

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
*   **Performance:** Section 4 explains the approach for NFR-PERF-2,
    NFR-PERF-3 and NFR-PERF-7; the table below maps every NFR. Slice 1 is not
    load-tested. Slice 5 proves NFR-PERF-1 to NFR-PERF-9 on the reference
    environment with 20 million entries. The riskiest targets are p99 under
    100 ms for the week read and entry save during the Friday peak, because
    they are the most frequent calls and the save takes the day lock, and
    sign-in p99 under 500 ms during a 500-user burst, because each sign-in
    runs a deliberately slow hash.
*   **Rollout and rollback:** Each slice is a migration plus code. Migrations
    are forward-only in deployed environments. Each Alembic revision has a
    `downgrade` for local use only. Slice 1 has no existing data, so nothing
    needs to be migrated.
*   **Container images:** Image names are fully qualified (for example
    `docker.io/library/postgres:16`) because Podman asks which registry to use
    for short names. Each name can be overridden by an environment variable
    (`KAIROS_POSTGRES_IMAGE`, `KAIROS_PYTHON_IMAGE`, `KAIROS_NODE_IMAGE`,
    `KAIROS_CADDY_IMAGE`). This allows switching to an approved registry if
    the compliance rule against Docker also covers Docker Hub (see Open
    Questions).

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

--------------------------------------------------------------------------------

## Additional Resources

*   [prd.md](../../prd.md) version 1.3: requirements, scenarios and slices.
*   [design-theme.md](../../context/design-theme.md): colours, layout, tone of
    messages.
*   [engineering-standards.md](../../context/engineering-standards.md):
    repository layout, layers, error shape, `make` commands, commit rules.
*   [decision_options.md](../../.egm/decision_options.md): full options and trade-offs
    for Q1 to Q9.

--------------------------------------------------------------------------------

## Detailed Implementation Plan (Slice 1 only)

Scope: Slice 1, "Sign in and log time". It covers AUTH-1, AUTH-5, TIME-1 to
TIME-3, TIME-5 to TIME-7, TIME-14 to TIME-16, PRJ-4 and PRJ-9. It also
delivers the TIME-17 version check early (see Q6), plus liveness and
readiness checks (NFR-REL-3) and correlation IDs (NFR-OPS-3), because later
slices build on them.

All paths are relative to the project root `Kairos/`. The files are grouped
into sub-tasks ST-01 to ST-10, each done as one commit whose message starts
with the sub-task ID ([engineering-standards.md](../../context/engineering-standards.md)).
`implementation-plan.md` will be generated from this section.

**Rules for every sub-task**

*   When a sub-task is done, `make up`, `make lint` and `make test` pass. Each
    sub-task ends with a **Done when** line that names any further check.
*   Shared files grow one sub-task at a time. A sub-task that changes a file
    created earlier lists it as Modify and says what it adds.
*   Every path is written in full from `Kairos/`. Each new Python package
    directory gets an empty `__init__.py`, listed in the sub-task that creates
    the directory.

### Target Files to Create/Modify

#### ST-01: Repository skeleton, containers and `make` commands

1.  **`Makefile`** (Create)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** These targets:
        *   `up`: copies `.env.example` to `.env` if `.env` is missing,
            starts `db`, waits until `pg_isready -h 127.0.0.1 -U
            kairos_owner -d kairos` (run with `podman exec`) succeeds over
            TCP, retrying every second for up to 60 seconds, then starts `api`
            and `proxy`. ST-02 adds the `migrate` step and ST-05 the `seed`
            step, both before `api` starts.
        *   `down`
        *   `test-backend`: starts a throwaway PostgreSQL 16 container with
            `podman run --rm -d` (image `KAIROS_POSTGRES_IMAGE`, tmpfs
            storage, host port 55433, the three password variables, and
            `db/init` mounted read-only as the init directory). It waits as
            `up` does, runs `uv run pytest` in `backend/`, then removes the
            container.
        *   `test-frontend`: `npm --prefix frontend run test`.
        *   `test`: runs `test-backend` and `test-frontend`. ST-10 adds
            `test-e2e`.
        *   `lint`: `uv run ruff check .`, `uv run ruff format --check .` and
            `uv run mypy --strict kairos` in `backend/`, then
            `npm --prefix frontend run lint` and
            `npm --prefix frontend run typecheck`. ST-08 adds the
            `openapi-check` step.
        *   `logs`
        *   `psql`: opens a database shell.

        No target calls `docker`. Services and the test database run with
        `podman` or `podman-compose`. Linters and test runners run on the
        host. `up`, `test` and `lint` are required by engineering-standards.md.
2.  **`compose.yaml`** (Create)
    *   **Type / Scope:** podman-compose definition.
    *   **Proposed Changes:** Services:
        *   `db`: PostgreSQL 16, named volume, host port 55432, `db/init`
            mounted read-only, the variables `POSTGRES_PASSWORD`,
            `KAIROS_OWNER_PASSWORD` and `KAIROS_APP_PASSWORD`. It is started
            with `-c log_line_prefix='%m [%p] %a '` so that log lines show the
            request ID (section 2).
        *   `api`: backend image, port 8000 inside the network only, with
            `DATABASE_URL`, `KAIROS_ENV` and `KAIROS_LOG_LEVEL`.
        *   `proxy`: Caddy, host port 8443.

        Images come from the image variables below. Configuration is read
        from `.env`. ST-02 adds the `migrate` service and ST-05 the `seed`
        service.
3.  **`.env.example`** (Create)
    *   **Type / Scope:** Configuration template.
    *   **Proposed Changes:** Every variable below, with its local default.
        `make up` copies the file to `.env` if `.env` is missing (NFR-SEC-8,
        NFR-OPS-1).

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
4.  **`.gitignore`** (Create)
    *   **Type / Scope:** Git configuration.
    *   **Proposed Changes:** Ignore `.env`, `node_modules/`, `dist/`,
        `.venv/`, `__pycache__/`, coverage output and Playwright reports.
5.  **`db/init/01-roles.sh`** (Create)
    *   **Type / Scope:** PostgreSQL first-start script.
    *   **Proposed Changes:** Creates `kairos_owner` and `kairos_app` with the
        passwords from `KAIROS_OWNER_PASSWORD` and `KAIROS_APP_PASSWORD`, and
        creates the `kairos` database owned by `kairos_owner`. Runs only when
        the data volume is empty. The `btree_gist` extension is created by
        migration 0001 (ST-03).
6.  **`backend/pyproject.toml`** and **`backend/uv.lock`** (Create)
    *   **Type / Scope:** Python project.
    *   **Proposed Changes:**
        *   Runtime dependencies: FastAPI, uvicorn, SQLAlchemy 2.0, asyncpg,
            Alembic, pydantic, pydantic-settings, argon2-cffi.
        *   Development dependencies: pytest, pytest-asyncio 0.24 or later,
            pytest-cov, httpx, ruff, mypy.
        *   Configuration: ruff rules, `mypy --strict`, and pytest settings
            with `asyncio_mode = "auto"` and
            `--cov=kairos --cov-fail-under=80` (NFR-OPS-2).
7.  **`backend/Containerfile`** (Create)
    *   **Type / Scope:** Container image.
    *   **Proposed Changes:** Python 3.12 slim base (`KAIROS_PYTHON_IMAGE`).
        Installs runtime dependencies with `uv`, runs as a user that is not
        root, and starts one uvicorn process with
        `--factory kairos.main:create_app` (section 1, process model).
8.  **`backend/kairos/__init__.py`**, **`backend/kairos/main.py`**,
    **`backend/kairos/config.py`** (Create)
    *   **Type / Scope:** FastAPI application entry point and settings.
    *   **Proposed Changes:**
        *   `create_app()` builds the FastAPI app with title "Kairos" and
            OpenAPI at `/api/v1/openapi.json`. Later sub-tasks register their
            middleware and routers here (listed as Modify).
        *   `config.Settings` has the fields `env`, `database_url`,
            `owner_database_url`, `seed_password` and `log_level`, read from
            the variables above. Each entry point checks the fields it needs
            and refuses to start if one is missing. Later sub-tasks do not
            change `config.py`.
9.  **`backend/tests/__init__.py`**, **`backend/tests/test_app.py`** (Create)
    *   **Type / Scope:** API test.
    *   **Proposed Changes:** `create_app()` serves the OpenAPI document at
        `/api/v1/openapi.json` with title "Kairos". The test needs no
        database.
10. **`frontend/package.json`**, **`frontend/package-lock.json`**,
    **`frontend/tsconfig.json`**, **`frontend/vite.config.ts`**,
    **`frontend/eslint.config.js`**, **`frontend/index.html`**,
    **`frontend/src/main.tsx`**, **`frontend/src/vite-env.d.ts`**,
    **`frontend/src/test/setup.ts`**, **`frontend/src/smoke.test.ts`**
    (Create)
    *   **Type / Scope:** Frontend project.
    *   **Proposed Changes:**
        *   Dependencies: `react` and `react-dom` 18, `react-router-dom` 6,
            `@tanstack/react-query` 5, `@mui/material` and
            `@mui/icons-material` 6, `@emotion/react` and `@emotion/styled`
            11, `@fontsource/inter` and `@fontsource/roboto-condensed` 5.
        *   Development tools: `typescript` 5, `vite` 5 with
            `@vitejs/plugin-react`, `vitest` 2 with `jsdom`,
            `@testing-library/react`, `@testing-library/user-event` and
            `@testing-library/jest-dom`, `openapi-typescript` 7, `eslint` 9
            with `typescript-eslint` 8 and `eslint-plugin-react-hooks` 5,
            `@playwright/test`, `pg` and `@types/pg`.
        *   Scripts: `dev`, `build`, `test` (`vitest run`), `lint` and
            `typecheck` (`tsc --noEmit`), both covering `src/` and `e2e/`,
            `gen:api`, and `e2e` (`playwright test -c e2e/playwright.config.ts`).
        *   `vite.config.ts` sets `build.assetsInlineLimit: 0`, so fonts are
            served as files that the Content-Security-Policy allows. It also
            holds the Vitest settings: environment `jsdom`, setup file
            `src/test/setup.ts` (loads the jest-dom matchers), and include
            `src/**/*.test.{ts,tsx}`.
        *   `src/main.tsx` renders a page that says "Kairos". ST-08 replaces
            it. `src/smoke.test.ts` checks that the test setup loads, so
            Vitest has a test from ST-01. ST-08 deletes it.
        *   The date jump in ST-09 uses a native date input inside an MUI
            `TextField`, so no date-picker package is needed.
        *   During development, Vite forwards `/api` to
            `https://localhost:8443` (`secure: false`), the API behind Caddy.
            Use a Chromium-based browser or Firefox for `npm run dev`,
            because Safari refuses `Secure` cookies on `http://localhost`.
11. **`proxy/Caddyfile`** and **`proxy/Containerfile`** (Create)
    *   **Type / Scope:** Reverse proxy.
    *   **Proposed Changes:** The Containerfile builds the frontend in a Node
        22 LTS stage (`KAIROS_NODE_IMAGE`) and copies `dist/` into the Caddy
        image. The Caddyfile does four things:
        *   Serves `dist/`, falling back to `index.html` so client-side
            routes load.
        *   Forwards `/api/*` to `api:8000` with a 60-second timeout.
        *   Uses `tls internal` for local HTTPS.
        *   Sets `Content-Security-Policy: default-src 'self'; style-src
            'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self';
            frame-ancestors 'none'; base-uri 'self'; form-action 'self'`, and
            the `X-Content-Type-Options: nosniff` and
            `Referrer-Policy: same-origin` headers. `style-src` allows inline
            styles because MUI's styling library (Emotion) adds `<style>`
            elements at run time.
12. **`README.md`** (Create)
    *   **Type / Scope:** Documentation (NFR-OPS-6).
    *   **Proposed Changes:**
        *   Prerequisites: Podman 4.9 or later, `podman-compose` 1.1 or
            later, `make`, Python 3.12 with `uv`, Node.js 22 LTS with npm, and
            the Playwright browsers (`npx playwright install chromium webkit`,
            plus `--with-deps` on Linux).
        *   One-command start with `make up`, then open
            `https://localhost:8443` and accept the local certificate once.
        *   `make test`, `make lint` and `make openapi`. `make test` does not
            start the stack; the end-to-end tests need `make up` first.
        *   Every environment variable from `.env.example` with its meaning.
        *   How to reset the database (`make down` and remove the volume).
        *   A placeholder section, "Load test (Slice 5)".

        Written in ST-01, then extended by ST-05 (seed accounts) and ST-10
        (end-to-end tests).

**Done when:** `make up` serves the placeholder page at
`https://localhost:8443` and the OpenAPI document at
`https://localhost:8443/api/v1/openapi.json`, and `make lint` and `make test`
pass.

#### ST-02: Backend foundation (errors, logging, database session, request pipeline, health)

1.  **`backend/kairos/errors.py`** (Create)
    *   **Type / Scope:** Shared module.
    *   **Proposed Changes:**
        *   `AppError(status, code, message, details)` and one subclass per
            code in the section 6 error table.
        *   A handler for `AppError` writes `{ code, message, details }`.
        *   A handler for `RequestValidationError` writes
            `VALIDATION_FAILED`, turning each Pydantic error location into the
            camelCase field name in `details.fields`.
        *   A handler for Starlette's `HTTPException` maps 404 to `NOT_FOUND`
            and 405 to `METHOD_NOT_ALLOWED`.
2.  **`backend/kairos/logging.py`** (Create)
    *   **Type / Scope:** Shared module.
    *   **Proposed Changes:** A JSON log formatter that adds `request_id`,
        `employee_id`, `method`, `path`, `status` and `duration_ms`. A
        redaction filter that applies the rule in Technical Plan section 2
        (NFR-SEC-1, NFR-OPS-3).
3.  **`backend/kairos/middleware.py`** (Create)
    *   **Type / Scope:** ASGI middleware.
    *   **Proposed Changes:** A pure ASGI middleware, not
        `BaseHTTPMiddleware`, and the outermost layer. It takes
        `X-Request-ID` from the request (accepted only if it is a UUID) or
        creates one, stores it in a context variable, echoes it in the
        response, and logs one access line per request. It catches any
        unhandled exception, logs the stack trace, and builds the
        `INTERNAL_ERROR` response itself, with `X-Request-ID` and
        `details.requestId`.
4.  **`backend/kairos/db.py`** (Create)
    *   **Type / Scope:** Database session module.
    *   **Proposed Changes:**
        *   Creates the async engine on first use from `Settings` (pool of 20
            plus up to 10 extra, `statement_timeout` 5 s). When
            `KAIROS_ENV=test` it uses `NullPool`, so no connection outlives
            the event loop of the test that opened it, and each concurrent
            request gets its own connection.
        *   `transaction(actor_id)` begins a transaction and runs the
            statement in Technical Plan section 2, step 5, which sets
            `kairos.actor_id` and `application_name`.
        *   `get_session()` runs the same statement with an empty actor for
            read-only requests.
5.  **`backend/kairos/security/__init__.py`**,
    **`backend/kairos/security/csrf.py`** (Create)
    *   **Type / Scope:** Dependency.
    *   **Proposed Changes:** Refuses POST, PUT, PATCH and DELETE without
        `X-Kairos-CSRF: 1`, with 403 `CSRF_FAILED`. The sign-in route uses it
        too.
6.  **`backend/kairos/health/__init__.py`**,
    **`backend/kairos/health/router.py`** (Create)
    *   **Type / Scope:** Router.
    *   **Proposed Changes:** `GET /api/v1/health/live` and
        `GET /api/v1/health/ready`. Readiness runs `SELECT 1` with a 1-second
        timeout and otherwise returns 503 `NOT_READY` (NFR-REL-3).
7.  **`backend/kairos/main.py`** (Modify)
    *   **Type / Scope:** Application entry point.
    *   **Proposed Changes:** Registers, outermost first: the middleware from
        `middleware.py`; the exception handlers from `errors.py`; the health
        router.
8.  **`backend/alembic.ini`**, **`backend/migrations/env.py`**,
    **`backend/migrations/script.py.mako`**,
    **`backend/migrations/versions/.gitkeep`** (Create)
    *   **Type / Scope:** Migration configuration.
    *   **Proposed Changes:** Async Alembic environment that connects as
        `kairos_owner` through `KAIROS_OWNER_DATABASE_URL`. Revision
        filenames start with a 4-digit number. There are no revisions yet, so
        `alembic upgrade head` does nothing until ST-03.
9.  **`Makefile`** and **`compose.yaml`** (Modify)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** `compose.yaml` adds the `migrate` service
        (backend image, command `alembic upgrade head`,
        `KAIROS_OWNER_DATABASE_URL`). `up` runs it as a one-off container
        (`podman-compose run --rm migrate`) after the database is ready and
        before `api` starts.
10. **`backend/tests/conftest.py`** (Create)
    *   **Type / Scope:** Test fixtures.
    *   **Proposed Changes:**
        *   Runs `uv run alembic upgrade head` once per test session as a
            subprocess, from a synchronous session-scoped fixture, with
            `KAIROS_OWNER_DATABASE_URL` set to
            `KAIROS_TEST_OWNER_DATABASE_URL`.
        *   The app under test connects as `kairos_app`
            (`KAIROS_TEST_DATABASE_URL`, `KAIROS_ENV=test`). Migrations,
            factories and table clearing connect as `kairos_owner`
            (`KAIROS_TEST_OWNER_DATABASE_URL`).
        *   Clears all tables between tests with
            `TRUNCATE ... RESTART IDENTITY CASCADE` (needed because the
            concurrency tests must commit).
        *   An httpx `AsyncClient` with `ASGITransport(app=...)` and
            `base_url="https://test"`, so the `Secure` session cookie is sent.
        *   A test-only router added to the test app, with one route per
            error code that raises it, and a POST route.

        Tests never use the seed (PRD section 8).
11. **`backend/tests/test_health.py`** (Create)
    *   **Type / Scope:** API test.
    *   **Proposed Changes:** Live returns 200. Ready returns 200 with the
        database up, and 503 `NOT_READY` from a second app instance whose
        database URL points at a closed port.
12. **`backend/tests/test_errors_and_logging.py`** (Create)
    *   **Type / Scope:** Unit and API tests, using the test-only router.
    *   **Proposed Changes:**
        *   The error shape is the same for 401, 403, 404 (an unknown path),
            405, 422 and 500.
        *   The 500 response carries `X-Request-ID` and `details.requestId`.
        *   The response echoes `X-Request-ID`.
        *   A POST without the CSRF header gets 403 `CSRF_FAILED`.
        *   The redaction filter replaces `password`, `token`, `cookie`,
            `set-cookie` and `authorization`, in any letter case and at any
            depth.

**Done when:** `make up` runs the empty migration step, and
`https://localhost:8443/api/v1/health/ready` returns 200.

#### ST-03: Slice 1 database schema

1.  **`backend/migrations/versions/0001_slice1_schema.py`** (Create)
    *   **Type / Scope:** Migration (NFR-DATA-2).
    *   **Proposed Changes:**
        *   Creates the `btree_gist` extension. It is trusted since
            PostgreSQL 13, so the database owner can create it.
        *   Creates `employees`, `employee_credentials`, `sessions`,
            `employee_preferences`, `projects` (with `project_code_seq`
            starting at 1001), `assignments` (with the exclusion constraint),
            `tasks`, `holidays` and `time_entries`, with exactly the Slice 1
            columns, `CHECK`s and constraints in the data dictionary.
        *   Creates these indexes: `time_entries (employee_id, entry_date)`,
            `assignments (employee_id, start_date)` and
            `sessions (employee_id)`.
        *   Creates the `employee_calendar()` function (Q1).
        *   Grants SELECT, INSERT, UPDATE and DELETE on all tables, and USAGE
            on the sequence, to `kairos_app`.
        *   Not created: the `(entry_date, project_id)` index,
            `period_locks`, `audit_log`, `leave_type`, `tasks.is_active` and
            the Slice 2 credential columns.

        `downgrade()` drops everything, for local use only.
2.  **`backend/kairos/models.py`** (Create)
    *   **Type / Scope:** SQLAlchemy table definitions.
    *   **Proposed Changes:** Declarative models that match the migration,
        used by repositories. Models are not used to create tables: the
        migration is the only source of the schema.
3.  **`backend/tests/factories.py`** (Create)
    *   **Type / Scope:** Test fixtures.
    *   **Proposed Changes:** Factory functions for employees (each also
        writes an `employee_credentials` row), projects, assignments, tasks,
        holidays and entries. They connect as `kairos_owner`.
4.  **`backend/tests/db/__init__.py`**, **`backend/tests/db/test_schema.py`**
    (Create)
    *   **Type / Scope:** Database tests.
    *   **Proposed Changes:**
        *   The `duration_minutes` `CHECK` rejects 10, 20 and 1455.
        *   The `kind` `CHECK` rejects a WORK entry without a project, and
            any kind other than `WORK`.
        *   The exclusion constraint rejects an overlapping assignment and
            accepts one next to it.
        *   `employee_calendar` returns expected 480 on a weekday and 0 at a
            weekend, 0 before the date of joining, holiday 480 on a weekday
            and 0 at a weekend, and no holiday before the date of joining.
        *   `kairos_app` cannot drop tables.

**Done when:** `make up` applies migration 0001 to a fresh database.

#### ST-04: Sign in, sessions, sign out (AUTH-1, AUTH-5)

1.  **`backend/kairos/security/passwords.py`** (Create)
    *   **Type / Scope:** Module.
    *   **Proposed Changes:** `hash_password()` and `verify_password()` with
        argon2id (3 passes, 64 MiB, 1 lane). They run in a worker thread, at
        most 1 at a time per process. A call waits up to 2 seconds for its
        turn, otherwise sign-in returns 503 `SIGN_IN_BUSY`.
        `needs_rehash()` lets the settings be changed later. Before the cap
        is fixed, this sub-task confirms from the `argon2-cffi` documentation
        or source that hashing releases the Python GIL, and records the
        result in this document (see Risks).
2.  **`backend/kairos/auth/__init__.py`**, **`backend/kairos/auth/schemas.py`**,
    **`backend/kairos/auth/router.py`**, **`backend/kairos/auth/service.py`**,
    **`backend/kairos/auth/repository.py`**,
    **`backend/kairos/auth/dependencies.py`** (Create)
    *   **Type / Scope:** Feature module (router → service → repository).
    *   **Proposed Changes:**
        *   **Sign-in:** the `email` field is a `str` of 3 to 254 characters,
            lowercased. It is not `EmailStr`, so the email-validator package
            is not needed. The password hash is read from
            `employee_credentials`. When the employee is not found, a dummy
            hash is still verified so that response time does not reveal
            whether the account exists. Inactive accounts get the same 401.
            On success the service creates a 32-byte token and stores its
            SHA-256 hash with `expires_at = now() + 8h`. It sets the cookie
            `__Host-kairos_session` with `HttpOnly`, `Secure`,
            `SameSite=Strict`, `Path=/` and `Max-Age=28800`.
        *   **Sign-out:** deletes the session and clears the cookie with the
            same attributes.
        *   **`current_employee` dependency:** runs one query (session joined
            to employee, not expired, active) and otherwise returns 401
            `SESSION_ENDED`.
        *   **`GET /me`.**
        *   **Expired sessions:** deleted at sign-in for that employee.
3.  **`backend/kairos/main.py`** (Modify)
    *   **Type / Scope:** Application entry point.
    *   **Proposed Changes:** Registers the auth router and `GET /me`.
4.  **`backend/tests/conftest.py`** (Modify)
    *   **Type / Scope:** Test fixtures.
    *   **Proposed Changes:** Adds the `signed_in_client(employee)` helper.
5.  **`backend/tests/api/__init__.py`**, **`backend/tests/api/test_auth.py`**
    (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   Sign-in succeeds and sets the cookie with all of its attributes.
        *   Wrong password, unknown email and inactive account all return the
            same 401 body.
        *   Email matching ignores case.
        *   `expires_at` equals the sign-in time plus 8 hours (to within 1
            second), and is unchanged after a later authenticated request
            (AUTH-5).
        *   A session whose `expires_at` is set to the past returns 401
            (AUTH-5).
        *   After sign-out the old cookie returns 401. Sign-out without a
            session gives 401, and without the CSRF header gives 403.
        *   `/me` without a cookie returns 401.
        *   A session belonging to an employee made inactive in the database
            returns 401 on the next request. This is groundwork for AUTH-6.
        *   A log line from a sign-in request contains neither the password
            nor the cookie.

**Done when:** an account created directly in the database can sign in and
out through `https://localhost:8443/api/v1/auth/sign-in`.

#### ST-05: Seed script (PRD section 8, Slice 1 subset)

1.  **`backend/kairos/seed.py`** (Create)
    *   **Type / Scope:** Command-line script, run as
        `python -m kairos.seed`.
    *   **Proposed Changes:** Refuses to run when `KAIROS_ENV` is not `local`.
        Connects as `kairos_owner` and sets the actor to `system`. Entry point
        `run(session, today: date)`. The command line passes today's date in
        `Asia/Kolkata`, and tests pass a fixed date. Every date below is
        relative to `today` (R).
        *   **Employees** (date of joining R - 400 days, password
            `KAIROS_SEED_PASSWORD`, default `Kairos2026x`, which meets
            AUTH-3):

            | Code | Name | Email | Role | Department | Designation |
            |---|---|---|---|---|---|
            | K0001 | Arjun Mehta | arjun@kairos.example.com | HR_ADMIN | Delivery | HR Operations Lead |
            | K0002 | Priya Sharma | priya@kairos.example.com | EMPLOYEE | Engineering | Software Engineer |
            | K0003 | Rahul Verma | rahul@kairos.example.com | EMPLOYEE | Engineering | Senior Software Engineer |
            | K0004 | Meera Iyer | meera@kairos.example.com | EMPLOYEE | Engineering | QA Engineer |
            | K0005 | Vikram Nair | vikram@kairos.example.com | EMPLOYEE | Delivery | Project Manager |
            | K0006 | Ananya Das | ananya@kairos.example.com | EMPLOYEE | Delivery | Business Analyst |

        *   **Projects** (start R - 3650 days, no end date, `ACTIVE`): common
            Internal, Training, Recruitment and Pre-sales (PRJ-4), and client
            "Acme Health – Dev Pod" (Acme Health), "Borealis Data Platform"
            (Borealis Energy) and "Cobalt Mobile App" (Cobalt Retail).
        *   **Assignments** (start R - 400 days, no end date unless stated):
            Priya to Acme; Rahul to Acme and Borealis; Meera to Cobalt;
            Vikram to Borealis; Ananya to Acme, and to Cobalt ending R - 10
            days. Arjun has none.
        *   **Tasks:** the 12 tasks from PRD section 8. No holidays are
            seeded in Slice 1.
        *   **Entries:** for each employee, on each Monday-to-Friday date
            from the Monday of the previous week up to and including R, but
            only on dates where the employee has no entries:
            *   Two work entries that total 8 hours. The first is a random
                multiple of 15 minutes from 4 to 6 hours.
            *   Each entry uses a project available to that employee on that
                date (the PRJ-4/PRJ-9 rule), a random task and a random work
                location.
            *   Randomness comes from `random.Random` seeded with the
                employee code and the date, so the same run date always
                produces the same entries.
            *   Exception: Vikram has no entries on the Friday of the
                previous week, so that week is 32 hours (incomplete).
        *   **Natural keys for reruns:** employees by code, projects by name,
            tasks by name, and assignments by (employee code, project name),
            where any existing assignment for the pair means skip.
            Credentials and preferences are keyed by employee. Running the
            script again, on the same day or later, adds only missing rows
            and never touches entries a developer added.
2.  **`Makefile`** and **`compose.yaml`** (Modify)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** `compose.yaml` adds the `seed` service (backend
        image, command `python -m kairos.seed`, `KAIROS_OWNER_DATABASE_URL`,
        `KAIROS_ENV`, `KAIROS_SEED_PASSWORD`). `up` runs it as a one-off
        container after `migrate` and before `api` starts.
3.  **`backend/tests/test_seed.py`** (Create)
    *   **Type / Scope:** Test.
    *   **Proposed Changes:**
        *   Running the seed twice with the same R gives the same row counts.
        *   `run` with R = a Wednesday, then again with R + 1 day, succeeds
            without an exclusion error.
        *   The current and previous week both contain entries for every
            employee. With R = a Monday, the current week has entries on that
            Monday.
        *   Vikram's previous week is under 40 hours.
        *   No seeded day is over 1,440 minutes, and every seeded entry's
            project is available to its employee on its date.
        *   The script refuses to run with `KAIROS_ENV=production`.
4.  **`README.md`** (Modify)
    *   **Type / Scope:** Documentation.
    *   **Proposed Changes:** Add the table of seed accounts, taken from the
        table above (employee code, name, email, role, department, assigned
        projects), and note that Ananya has the ended assignment and Vikram
        the incomplete week.

**Done when:** after `make up`, every seed account can sign in with
`KAIROS_SEED_PASSWORD`.

#### ST-06: Available projects and tasks (PRJ-4, PRJ-9, TIME-15)

1.  **`backend/kairos/projects/__init__.py`**,
    **`backend/kairos/projects/schemas.py`**,
    **`backend/kairos/projects/router.py`**,
    **`backend/kairos/projects/service.py`**,
    **`backend/kairos/projects/repository.py`** (Create)
    *   **Type / Scope:** Feature module.
    *   **Proposed Changes:** `GET /api/v1/me/available-projects?date=&q=`.
        The repository function `available_projects(employee_id, on_date,
        q=None, project_id=None)` is the single PRJ-4/PRJ-9 query (Technical
        Plan section 6). The service calls it with `project_id` for saves, and
        with `q` for the selector. `q` matches a code prefix or part of the
        name, ignoring case. At most 50 results, ordered with common projects
        first, then by code. `label` is `[code] name`.
2.  **`backend/kairos/tasks/__init__.py`**, **`backend/kairos/tasks/schemas.py`**,
    **`backend/kairos/tasks/router.py`**,
    **`backend/kairos/tasks/repository.py`** (Create)
    *   **Type / Scope:** Feature module. There is no service file, because
        there are no rules.
    *   **Proposed Changes:** `GET /api/v1/tasks` returns all tasks sorted by
        name, with an `ETag` that is a hash of the result.
3.  **`backend/kairos/main.py`** (Modify)
    *   **Type / Scope:** Application entry point.
    *   **Proposed Changes:** Registers the projects and tasks routers.
4.  **`backend/tests/api/test_available_projects.py`** (Create)
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
5.  **`backend/tests/api/test_tasks.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:** Returns all tasks sorted by name with an
        `ETag`. Without a session the request returns 401.

**Done when:** the seed accounts see their projects through
`https://localhost:8443/api/v1/me/available-projects`.

#### ST-07: Weekly view and time entries API (TIME-1 to TIME-3, TIME-5 to TIME-7, TIME-14, TIME-16, TIME-17 early)

1.  **`backend/kairos/time_entries/__init__.py`**,
    **`backend/kairos/time_entries/schemas.py`** (Create)
    *   **Type / Scope:** Pydantic models.
    *   **Proposed Changes:** `NewWorkEntry`:
        *   `durationMinutes` is between 15 and 1440 and a multiple of 15
            (TIME-5).
        *   `activity` is at most 500 characters, with surrounding spaces
            removed (TIME-3).
        *   `workLocation` is one of the enumeration values.

        Also the `Entry`, `HolidayEntry`, `Day` and `Week` models, with the
        derived fields in section 6.
2.  **`backend/kairos/time_entries/repository.py`** (Create)
    *   **Type / Scope:** Repository.
    *   **Proposed Changes:**
        *   `entries_for_week(employee_id, monday)`.
        *   `calendar_for_range(employee_id, from, to)`, which calls
            `employee_calendar`.
        *   `lock_day(employee_id, date)`, which runs `pg_advisory_xact_lock`
            for that employee and date (Q3).
        *   `day_entry_minutes(employee_id, date)`, which returns the fresh
            sum of that day's entry minutes.
        *   `insert_entry`.
        *   `update_entry_if_version(id, employee_id, version, values)`,
            which returns the new row or None.
        *   `delete_entry_if_version`.
        *   `get_entry_for_owner(id, employee_id)`.
        *   `upsert_last_location`.
3.  **`backend/kairos/time_entries/service.py`** (Create)
    *   **Type / Scope:** Service, which holds every rule. It is the only
        module that writes `time_entries`.
    *   **Proposed Changes:**
        *   **`read_week(employee, any_date)`:** finds Monday with ISO
            rules. It merges entries with holiday pseudo-entries, takes
            expected and holiday minutes from `calendar_for_range`, sums day
            and week totals from the entries it loaded plus holiday minutes,
            and works out `complete` (TIME-1, TIME-2, TIME-16).
        *   **`create_work_entry`:** inside `transaction(actor)`:
            1.  Checks `available_projects(..., project_id)`, otherwise 403
                `PROJECT_NOT_AVAILABLE` (PRJ-9).
            2.  Checks the task exists, otherwise 422 `VALIDATION_FAILED` on
                `taskId`.
            3.  Locks the day with `lock_day`.
            4.  Checks that `day_entry_minutes` plus holiday minutes plus the
                new minutes are at most 1,440. Otherwise it returns 409
                `DAILY_LIMIT_EXCEEDED` with the `details` and message in the
                section 6 error table (TIME-6).
            5.  Inserts the entry and updates `last_work_location` (TIME-3).

            Two entries with the same project and task on the same day are
            allowed (TIME-14).
        *   **`update_work_entry(employee, entry_id, version, values)`:**
            inside `transaction(actor)`, in this order. A request with several
            problems gets the first error in the list.
            1.  Reads the entry with `get_entry_for_owner`. No row, including
                another employee's entry, gives 404 `ENTRY_NOT_FOUND`
                (TIME-7, NFR-SEC-3).
            2.  If the entry's `version` differs from `If-Match`, returns 409
                `ENTRY_CHANGED` with the current entry in `details.entry`.
            3.  If the project or the date changes, checks
                `available_projects(..., project_id)` for the new date,
                otherwise 403 `PROJECT_NOT_AVAILABLE`. An edit that keeps
                both the project and the date is not checked again, so an
                employee can still correct an entry on a project that has
                since closed (PRJ-10).
            4.  If the task changes, checks that it exists, otherwise 422
                `VALIDATION_FAILED` on `taskId`.
            5.  Locks the days with `lock_day`: one day when the date is
                unchanged, or both days in ascending date order when it
                changes.
            6.  Sums the target day's entries after the lock
                (`day_entry_minutes`) and adds its holiday minutes. Then:
                *   Date unchanged: new total = day minutes - old minutes +
                    new minutes + holiday minutes.
                *   Date changed: new total = target-day minutes + new
                    minutes + holiday minutes. The old day's total only goes
                    down, so it is not checked.

                If the new total is over 1,440 **and** higher than the target
                day's total before the save, returns 409
                `DAILY_LIMIT_EXCEEDED` (see The Problem).
            7.  Runs `update_entry_if_version`. If it returns None because
                another save changed the entry after step 2, returns 409
                `ENTRY_CHANGED`.
            8.  Updates `last_work_location` (TIME-3).
        *   **`delete_work_entry(employee, entry_id, version)`:** reads the
            entry with `get_entry_for_owner`, otherwise 404
            `ENTRY_NOT_FOUND`. Then runs `delete_entry_if_version`. If no row
            is deleted, returns 409 `ENTRY_CHANGED` with the current entry. A
            delete only lowers a day's total, so it takes no day lock
            (TIME-7).

        Slice 1 does not apply TIME-8 or TIME-9. Slice 3 adds them and the
        `editable` field.
4.  **`backend/kairos/time_entries/router.py`** (Create)
    *   **Type / Scope:** Router.
    *   **Proposed Changes:**
        *   `GET /api/v1/me/week`.
        *   `POST /api/v1/me/time-entries`.
        *   `PUT /api/v1/me/time-entries/{id}` and
            `DELETE /api/v1/me/time-entries/{id}` both require
            `If-Match: "<version>"`. Missing: 428 `VERSION_REQUIRED`. Present
            but not a quoted positive integer, including `*` and `W/"3"`: 422
            `VALIDATION_FAILED` on field `If-Match`. These checks run before
            the service.
        *   POST (201) and PUT (200) set `ETag: "<version>"`. `GET /me/week`
            sets no `ETag`.
5.  **`backend/kairos/main.py`** (Modify)
    *   **Type / Scope:** Application entry point.
    *   **Proposed Changes:** Registers the time-entries router.
6.  **`backend/tests/api/test_week.py`** (Create)
    *   **Type / Scope:** API tests.
    *   **Proposed Changes:**
        *   The ISO week number and date range are correct, including week 1
            of a year that starts mid-week and week 53.
        *   Day and week totals are correct, and `complete` follows the
            section 6 rule.
        *   A holiday inserted directly in the database appears as a
            `HOLIDAY` entry and is counted. This proves the Q1 path before
            Slice 3.
        *   Expected minutes exclude days before the date of joining.
        *   A week with no entries returns 7 empty days.
        *   401 without a session; 422 on field `date` for
            `date=2026-13-01`.
7.  **`backend/tests/api/test_time_entries.py`** (Create)
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
        *   An unknown `taskId` gives 422 on `taskId`.
        *   Edit and delete own entries (TIME-7).
        *   Another employee's entry gives 404 on PUT and DELETE (NFR-SEC-3).
        *   A stale `If-Match` gives 409 `ENTRY_CHANGED` with
            `details.entry`. A missing `If-Match` gives 428 on both PUT and
            DELETE, and a malformed one gives 422.
        *   Moving a 4-hour entry onto a day with 22 hours gives 409
            `DAILY_LIMIT_EXCEEDED`, not 500.
        *   On a day with 20 hours of entries and an 8-hour holiday inserted
            in the database, shortening a 5-hour entry to 4 hours is accepted
            and lengthening it is rejected.
        *   Editing only the duration of an entry whose project has since
            closed is accepted.
        *   A PUT that hits several errors gets the first error in the step
            order.
        *   The last-used work location is returned by `/me`.
8.  **`backend/tests/api/test_daily_limit_concurrency.py`** (Create)
    *   **Type / Scope:** Concurrency test (NFR-DATA-1).
    *   **Proposed Changes:** Two signed-in clients for the same employee
        send requests at the same moment (`asyncio.gather`; each request uses
        its own database connection).
        *   **Case 1:** the day has 16 hours, and both clients POST 5 hours.
            Exactly one gets 201 and the other gets 409
            `DAILY_LIMIT_EXCEEDED`, never 500, and the day's entries sum to
            21 hours.
        *   **Case 2:** the target day has 17 hours, and the employee has a
            4-hour entry on each of two other days. Both clients PUT one of
            those entries onto the target day. Exactly one gets 200 and the
            other 409 `DAILY_LIMIT_EXCEEDED`, with no deadlock error, and the
            target day sums to 21 hours.

        Each case runs 20 times with fresh data.

**Done when:** a seed account can create, edit and delete entries through
the API at `https://localhost:8443`.

#### ST-08: Frontend foundation and sign-in

1.  **`frontend/src/main.tsx`** (Modify), **`frontend/src/App.tsx`** (Create)
    *   **Type / Scope:** App entry and routes.
    *   **Proposed Changes:**
        *   `main.tsx` sets up `QueryClientProvider`, MUI `ThemeProvider` and
            the router, and renders `App`.
        *   `App.tsx` routes `/sign-in`, `/week/:isoWeek`, and `/`, which
            redirects to the current week.
        *   If `GET /me` returns 401, the route guard redirects to
            `/sign-in`.
        *   `App.tsx` registers the redirect handler that `client.ts` calls
            on `SESSION_ENDED`.
2.  **`frontend/src/theme.ts`**, **`frontend/src/theme.test.ts`** (Create)
    *   **Type / Scope:** MUI theme and its test.
    *   **Proposed Changes:**
        *   Colours from design-theme.md as named palette entries (`work`,
            `leave`, `holiday`, `warning`). Each sets `contrastText`:
            `#1A1A1A` on work (`#00A67E`), and `#FFFFFF` on leave, holiday
            and warning.
        *   Inter at 14 px, and a `numeric` text style in condensed bold.
        *   Visible focus outlines on all interactive components (NFR-UX-3).
        *   The MUI type augmentation for the `work`, `leave` and `holiday`
            palette entries and the `numeric` text style.
        *   The test checks that every palette entry's `contrastText` has a
            WCAG contrast ratio of at least 4.5:1 against its main colour.
3.  **`frontend/src/api/client.ts`** (Create)
    *   **Type / Scope:** API client module, the only place `fetch` is used.
    *   **Proposed Changes:**
        *   Typed functions `signIn`, `signOut`, `getMe`, `getWeek`,
            `getAvailableProjects`, `getTasks`, `createEntry`, `updateEntry`
            and `deleteEntry`.
        *   Adds `X-Request-ID` (`crypto.randomUUID()`), `X-Kairos-CSRF: 1`
            and `If-Match`, and uses `credentials: 'same-origin'`.
        *   Throws `ApiError { status, code, message, details }`. On a 401
            `SESSION_ENDED` from any function except `signIn`, it first calls
            the redirect handler that `App.tsx` registers, which goes to
            `/sign-in`.
4.  **`frontend/src/api/schema.ts`**, **`frontend/src/api/openapi.json`**
    (Create, generated) and **`backend/kairos/export_openapi.py`** (Create)
    *   **Type / Scope:** Generated types and the script that exports them.
    *   **Proposed Changes:** `make openapi` runs `python -m
        kairos.export_openapi`, which builds the app without connecting to
        the database and without required settings, and writes
        `frontend/src/api/openapi.json`. It then runs `openapi-typescript` to
        produce `schema.ts`. Both files are committed.
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
7.  **`frontend/src/features/week/isoWeek.ts`**,
    **`frontend/src/features/week/WeekPage.tsx`** (Create)
    *   **Type / Scope:** Pure functions and a placeholder page.
    *   **Proposed Changes:** `isoWeekOf(date)` and `mondayOf(isoWeek)`
        handle week 1 and week 53 edge cases. `WeekPage` shows the ISO week
        from the route; ST-09 builds the full page.
8.  **`frontend/src/features/auth/SignInPage.test.tsx`**,
    **`frontend/src/api/client.test.ts`**,
    **`frontend/src/features/week/isoWeek.test.ts`** (Create)
    *   **Type / Scope:** Unit tests.
    *   **Proposed Changes:**
        *   The client adds the headers and turns an error body into an
            `ApiError`.
        *   A 401 `SESSION_ENDED` from `getWeek` calls the redirect handler.
            A 401 `INVALID_CREDENTIALS` from `signIn` does not.
        *   The sign-in page shows the server's message and moves to the
            week on success.
        *   ISO week edge cases.
9.  **`Makefile`** (Modify), **`frontend/src/smoke.test.ts`** (Delete)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** Adds the `openapi` target, and an
        `openapi-check` step to `lint` that fails if regenerating
        `openapi.json` or `schema.ts` changes either file.

**Done when:** a seed account can sign in at `https://localhost:8443` and
reaches the placeholder week page.

#### ST-09: Weekly view and entry form

1.  **`frontend/src/features/week/WeekPage.tsx`** (Modify)
    *   **Type / Scope:** Feature page.
    *   **Proposed Changes:**
        *   **Header:** date range, "Week 47", week total, and a "Week
            incomplete" or "Week complete" label with an icon.
        *   **Navigation:** previous and next buttons, and a date input that
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
        *   **EntryCard:** a solid card in the kind's colour, with text in
            the palette entry's `contrastText`, the kind's text label and
            icon, task, `[code] name`, duration as `8h 0m` and work location
            (TIME-11, TIME-16). Edit and delete actions are shown on `WORK`
            entries. Holiday entries have none.
        *   **Delete:** asks for confirmation, then sends `If-Match`.
3.  **`frontend/src/features/week/EntryFormModal.tsx`**,
    **`frontend/src/features/week/ProjectSelect.tsx`** (Create)
    *   **Type / Scope:** Components.
    *   **Proposed Changes:**
        *   **The form:** a centred modal with fields in the order Date,
            Project, Task, Duration, Work location, Activity, and Save as the
            only primary button. The form opens with focus on Project.
        *   **ProjectSelect:** an MUI Autocomplete that calls
            `getAvailableProjects(date, q)` with the form's current date as
            the user types, after a pause of 150 ms. When nothing matches, it
            says "No projects match. You can only pick projects HR has
            assigned to you." (TIME-15).
        *   **Keyboard:** picking a value with Enter in Project or Task moves
            focus to the next field. Ctrl+Enter or Cmd+Enter saves
            (NFR-UX-5).
        *   **Defaults:** Date is the column's date. Work location is
            `Me.lastWorkLocation`, or `OFFICE` when that is null (TIME-3).
        *   **On 409 `ENTRY_CHANGED`:** shows the server message in
            `ErrorMessage` and replaces the form values with
            `details.entry`, including its new version.
        *   **On other errors:** when `details.fields` names a form field,
            the problem appears under that field. Otherwise the server
            message appears in `ErrorMessage` at the top of the modal.
4.  **`frontend/src/components/DurationInput.tsx`**,
    **`frontend/src/features/week/duration.ts`** (Create)
    *   **Type / Scope:** Component and pure functions.
    *   **Proposed Changes:**
        *   `parseDuration` accepts `8`, `7.5`, `7:45` and `7h45m`, and
            rejects any value that is not a multiple of 15 or is under 15.
        *   `formatDuration(480)` returns `8h 0m`.
5.  **`frontend/src/features/week/duration.test.ts`**,
    **`frontend/src/features/week/EntryFormModal.test.tsx`**,
    **`frontend/src/features/week/WeekPage.test.tsx`** (Create)
    *   **Type / Scope:** Unit and component tests.
    *   **Proposed Changes:**
        *   Duration parsing table.
        *   The form sends the correct body and shows server errors in the
            right place.
        *   The keyboard-only path saves in 4 interactions. This test is
            written once the NFR-UX-5 counting rule in Open Questions is
            confirmed.
        *   The 409 message appears and the form shows the current entry.
        *   The week page shows totals and labels, and does not rely on
            colour alone.

**Done when:** a seed account can log, edit and delete a day's work at
`https://localhost:8443`, and the NFR-UX-5 counting rule is confirmed.

#### ST-10: End-to-end scenario tests and slice wrap-up

1.  **`frontend/e2e/playwright.config.ts`**, **`frontend/e2e/fixtures.ts`**,
    **`frontend/e2e/db.ts`** (Create)
    *   **Type / Scope:** Playwright setup.
    *   **Proposed Changes:**
        *   Runs against `https://localhost:8443`, ignoring the local
            certificate.
        *   `db.ts` is the test-only SQL helper. It uses `pg` with
            `KAIROS_E2E_DATABASE_URL`, which connects as `kairos_owner`.
        *   A fixture creates its own employees, projects and assignments
            through the helper. It does not use the seed (PRD section 8).
            Fixture employees use one committed argon2id hash of a fixed test
            password. The hash is generated once with `argon2-cffi`, and the
            command is noted in `fixtures.ts`. Every email and code gets a
            per-run suffix, and teardown deletes the rows the run created.
        *   Chromium and WebKit, at 1280 px and 360 px widths.
2.  **`frontend/e2e/tests/cuj01-log-normal-day.spec.ts`**,
    **`frontend/e2e/tests/cuj02-daily-limit.spec.ts`**,
    **`frontend/e2e/tests/cuj03-unassigned-project.spec.ts`**,
    **`frontend/e2e/tests/cuj04-common-projects.spec.ts`**,
    **`frontend/e2e/tests/cuj05-edit-delete-navigate.spec.ts`**,
    **`frontend/e2e/tests/cuj06-sign-in-out.spec.ts`**,
    **`frontend/e2e/tests/keyboard-entry.spec.ts`** (Create)
    *   **Type / Scope:** End-to-end tests.
    *   **Proposed Changes:**
        *   One file per Slice 1 journey, CUJ 1 to CUJ 6. Test names use
            "CUJ-n (S-x)", for example "CUJ-2 (S-4)".
        *   `cuj01` also checks that styles are applied: the top bar's
            background is `#111111`.
        *   `cuj06` simulates the 8-hour expiry by setting `expires_at` in
            the past through the SQL helper.
        *   `keyboard-entry` checks NFR-UX-5 and visible focus. It is written
            once the counting rule in Open Questions is confirmed.
        *   The 360 px run checks that days stack.
3.  **`Makefile`** (Modify)
    *   **Type / Scope:** Build configuration.
    *   **Proposed Changes:** Adds `test-e2e`, which first checks
        `https://localhost:8443/api/v1/health/ready`. If the stack is not
        running, it stops with "Run make up first." Otherwise it runs
        `npm --prefix frontend run e2e`. `test` now also runs `test-e2e`.
4.  **`README.md`** (Modify)
    *   **Type / Scope:** Documentation.
    *   **Proposed Changes:** Add how to run the end-to-end tests, and a
        table that traces each Slice 1 requirement to its tests (taken from
        the traceability table below).

**Done when:** `make up` followed by `make test` passes, including the
end-to-end tests.

### Slice 1 traceability

| Requirement | Main files | Tests |
|---|---|---|
| AUTH-1 | `auth/service.py`, `security/passwords.py`, `SignInPage.tsx` | `test_auth.py`, `cuj06-sign-in-out.spec.ts` |
| AUTH-5 | `auth/service.py`, `auth/dependencies.py` | `test_auth.py` (8-hour expiry, sign-out), `cuj06-sign-in-out.spec.ts` |
| TIME-1 | `time_entries/service.py` (`read_week`), `App.tsx`, `WeekPage.tsx` | `test_week.py`, `cuj01-log-normal-day.spec.ts` |
| TIME-2 | `read_week`, `isoWeek.ts`, `WeekPage.tsx` | `test_week.py`, `isoWeek.test.ts`, `cuj05-edit-delete-navigate.spec.ts` |
| TIME-3 | `create_work_entry`, `EntryFormModal.tsx` | `test_time_entries.py`, `EntryFormModal.test.tsx` |
| TIME-5 | `time_entries/schemas.py`, migration `CHECK`, `duration.ts` | `test_time_entries.py`, `test_schema.py`, `duration.test.ts` |
| TIME-6 | `create_work_entry` / `update_work_entry`, `lock_day`, `day_entry_minutes` | `test_time_entries.py` (S-4), `test_daily_limit_concurrency.py`, `cuj02-daily-limit.spec.ts` |
| TIME-7 | `update_work_entry`, `delete_work_entry`, `EntryCard.tsx` | `test_time_entries.py`, `cuj05-edit-delete-navigate.spec.ts` |
| TIME-14 | no rule that forbids duplicates | `test_time_entries.py`, `cuj05-edit-delete-navigate.spec.ts` |
| TIME-15 | `projects/repository.py`, `ProjectSelect.tsx` | `test_available_projects.py`, `cuj03-unassigned-project.spec.ts` |
| TIME-16 | `EntryCard.tsx`, `Entry` schema | `WeekPage.test.tsx`, `cuj01-log-normal-day.spec.ts` |
| PRJ-4 | `available_projects` | `test_available_projects.py` (S-12), `cuj04-common-projects.spec.ts` |
| PRJ-9 | `available_projects`, `create_work_entry` | `test_available_projects.py`, `test_time_entries.py` (S-5) |
| TIME-17 (early) | `update_entry_if_version`, `EntryFormModal.tsx` | `test_time_entries.py`, `EntryFormModal.test.tsx` |
| NFR-REL-3 | `health/router.py` | `test_health.py` |
| NFR-OPS-3 | `middleware.py`, `logging.py`, `db.py` | `test_errors_and_logging.py` |
| NFR-SEC-1 | `security/passwords.py`, `logging.py` | `test_auth.py`, `test_errors_and_logging.py` |
| NFR-SEC-3 | `auth/dependencies.py`, `time_entries/repository.py` | `test_time_entries.py` |
| NFR-DATA-1 | `lock_day`, the assignments exclusion constraint | `test_daily_limit_concurrency.py`, `test_schema.py` |
| NFR-UX-5 | `EntryFormModal.tsx` | `EntryFormModal.test.tsx`, `keyboard-entry.spec.ts` |

### Verification Plan

*   **Build:** `make up` starts the stack from a fresh clone with a seeded
    database, and `https://localhost:8443` shows the sign-in page
    (NFR-OPS-1).
*   **Unit and API tests:** `make test-backend` runs pytest against a
    throwaway PostgreSQL 16 container (defined in ST-01). `make test-frontend`
    runs Vitest.
*   **End-to-end:** `make test-e2e` runs Playwright against the stack started
    by `make up`. `make test` runs all three.
*   **Static analysis and linting:** `make lint` runs ruff, `mypy --strict`,
    ESLint, `tsc --noEmit`, and the check that the OpenAPI types are up to
    date (NFR-OPS-5).
*   **Test coverage requirement:** backend line coverage of at least 80%,
    enforced by `--cov-fail-under=80` (NFR-OPS-2). Every endpoint except
    `GET /health/live`, which has no failure response, has at least one
    success test and one failure test (engineering standards).
*   **Slice 1 done when:** all of the above pass, every row in the
    traceability table has passing tests, and this document matches the code
    (PRD section 11).

--------------------------------------------------------------------------------

*Drafted with EGM*
