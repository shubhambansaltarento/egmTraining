# PRD: Kairos — Employee Time Tracking

| | |
|---|---|
| **Document type** | Product Requirements Document (workshop edition) |
| **Version** | 1.3 |
| **Status** | Approved for design |
| **Audience** | Workshop attendees practising spec-driven development with EGM |

> **Before you start (workshop attendees)**
>
> 1. Read the whole document once. It describes *what* to build and *why*, not *how*.
> 2. Edit **section 7.1 (Technology stack)** to match the stack you prefer. Everything else stays as is.
> 3. Use this file as the input to your design: `/egm:egm-brainstorm` → `/egm:egm-draft` → `/egm:egm-review` (Claude Code command names). Do not write code until your design is reviewed and approved by you.
> 4. Build in the order given in **section 9 (Delivery slices)**. A finished Slice 1 beats a half-finished Slice 4.

---

## 1. Background

Our organisation has about 5,000 employees working across client projects and internal initiatives. Every employee is expected to report how they spent their working hours each day. This data drives client billing, project costing, utilisation reporting and statutory compliance.

Today the data is collected through spreadsheets and email reminders. It is late, inconsistent and hard to audit. HR spends several days each month chasing missing entries and correcting holiday and leave data by hand.

**Kairos** is a web application that replaces this process. Employees report time against projects in a weekly view. HR maintains the master data (employees, projects, holidays) and monitors compliance.

## 2. Goals and non-goals

### 2.1 Goals

| ID | Goal | How we will measure it |
|---|---|---|
| G-1 | Make daily time reporting fast for employees | An employee can log a full 8-hour day in under 30 seconds |
| G-2 | Give HR a single place to manage employees, projects and holidays | No spreadsheet is needed for any master-data task |
| G-3 | Make missing time visible | HR can list every employee with incomplete time for any week in one screen |
| G-4 | Stay responsive at peak | Meets the performance targets in section 7.2 at 500 concurrent users |

### 2.2 Non-goals (out of scope for this version)

- Manager approval workflows. There is no manager persona.
- Leave balances, leave accrual and leave approval. Leave is only *recorded* as a time entry.
- Payroll, invoicing, billing rates and expense reporting.
- Single sign-on with other applications, social login and multi-factor authentication.
- Native mobile apps. The web app must work on a mobile browser, nothing more.
- Multi-tenancy. One deployment serves one organisation.
- Multiple time zones and multiple languages. The application uses IST (UTC+05:30) and English.
- Integrations with Jira, HRMS or calendar systems.

## 3. Personas

### 3.1 Employee — "Priya, Software Engineer"

- Works on one to three projects at a time.
- Wants to log time in seconds and get back to work. Often logs the whole week on Friday evening.
- Needs to see at a glance whether her week is complete.
- Uses a laptop most of the time, sometimes her phone.

### 3.2 HR Administrator — "Arjun, HR Operations"

- One of about 10 HR administrators.
- Onboards and offboards employees, maintains the holiday calendar and the project list.
- Chases employees with incomplete timesheets every Monday and at month end.
- Exports data for finance.
- Is also an employee: an HR administrator logs their own time and has every employee capability.

### 3.3 Role summary

| Capability | Employee | HR Administrator |
|---|---|---|
| Log, edit and delete own time entries | Yes | Yes |
| View own reports | Yes | Yes |
| View holiday calendar | Yes | Yes |
| View other employees' time entries | No | Yes (read-only) |
| Manage employees | No | Yes |
| Manage holidays | No | Yes |
| Manage projects, project assignments and tasks | No | Yes |
| View organisation reports and export data | No | Yes |
| Lock a reporting period | No | Yes |

## 4. Glossary

| Term | Meaning |
|---|---|
| **Time entry** | A record that an employee spent a duration on a date, against a project and a task (or as leave). |
| **Project** | A unit of work that time is reported against. Shown everywhere as `[code] name`, for example "[2827] Acme Health – Dev Pod". |
| **Client project** | A project that an employee can report time against only while HR has assigned them to it. |
| **Common project** | A project open to every employee without assignment, for example "Training" or "Recruitment". |
| **Assignment** | The link between an employee and a client project, with a start date and an optional end date. |
| **Task** | The kind of work done, chosen from a list maintained by HR, for example "Development (Coding)", "Review", "Meetings". |
| **Activity** | Free text in which the employee describes what they did, for example "Fixed login timeout bug". |
| **Work location** | Where the work was done: Office, Client Site or Work From Home. |
| **Leave entry** | A time entry that records absence instead of work, for example "Sick Leave". |
| **Designated holiday** | An organisation-wide holiday defined by HR. |
| **Week** | Monday to Sunday, identified by ISO week number. |
| **Expected hours** | 8 hours for each weekday (Monday to Friday), 0 for Saturday and Sunday. 40 hours for a normal week. |
| **Complete week** | A week in which an employee's total reported hours (work + leave + holidays) are at least the expected hours. |
| **Locked period** | A calendar month that HR has closed. Entries in it cannot be changed by employees. |

## 5. Functional requirements

Priorities: **M** = must have, **S** = should have, **C** = could have.

### 5.1 Authentication and account (AUTH)

| ID | Requirement | Priority |
|---|---|---|
| AUTH-1 | A user signs in with their work email and a password. | M |
| AUTH-2 | A new employee receives a temporary password and must set a new password at first sign-in. | M |
| AUTH-3 | Passwords are at least 10 characters and contain at least one letter and one digit. | M |
| AUTH-4 | After 5 consecutive failed sign-in attempts, the account is locked for 15 minutes. | M |
| AUTH-5 | A user can sign out. A session ends after 8 hours regardless of activity. | M |
| AUTH-6 | A deactivated employee cannot sign in, and any active session of theirs stops working within 1 minute of deactivation. | M |
| AUTH-7 | A user can change their own password after re-entering the current one. | S |
| AUTH-8 | An HR administrator can reset any employee's password, which issues a new temporary password. | S |

### 5.2 Employee management — HR (EMP)

| ID | Requirement | Priority |
|---|---|---|
| EMP-1 | HR can create an employee with: employee ID, full name, work email, designation, department, date of joining, and role (Employee or HR Administrator). | M |
| EMP-2 | Employee ID and work email are each unique across all employees, including deactivated ones. | M |
| EMP-3 | HR can edit every employee field except employee ID. | M |
| EMP-4 | HR can deactivate and reactivate an employee. Deactivation keeps all of the employee's history. Employees are never deleted. | M |
| EMP-5 | HR can search employees by name, employee ID or email and filter by department and status. The list is paginated. | M |
| EMP-6 | HR can bulk-create employees by uploading a CSV file. The upload reports each rejected row with a reason and creates the valid rows. | S |
| EMP-7 | An employee cannot report time for a date before their date of joining. | M |
| EMP-8 | The system prevents removing the HR Administrator role from, or deactivating, the last active HR administrator. | M |

### 5.3 Holiday management — HR (HOL)

| ID | Requirement | Priority |
|---|---|---|
| HOL-1 | HR can add a designated holiday with a date and a name. Only one holiday can exist on a date. | M |
| HOL-2 | HR can edit and delete a holiday, unless its date falls in a locked period. | M |
| HOL-3 | Every user can view the holiday list for a calendar year. | M |
| HOL-4 | A designated holiday on a weekday appears in every active employee's weekly view as an 8-hour "Designated Holiday" entry. The employee did nothing to create it and cannot edit or delete it. | M |
| HOL-5 | When HR adds, moves or deletes a holiday, every employee's weekly view and totals reflect the change the next time the view is loaded. | M |
| HOL-6 | A holiday that falls on a Saturday or Sunday is shown in the calendar but adds no hours. | M |
| HOL-7 | An employee can still log work on a designated holiday. The work hours are added to the holiday hours for that day. | S |
| HOL-8 | HR can copy all holidays from one year to the next as a starting point, then adjust the dates. | C |

### 5.4 Project and assignment management — HR (PRJ)

HR owns the project list and decides who can report time against each project. An employee never creates a project and never assigns themselves to one.

**Projects**

| ID | Requirement | Priority |
|---|---|---|
| PRJ-1 | HR can create a project with: name, client name, type (Client or Common), start date and optional end date. The system gives each project a unique numeric code that never changes. | M |
| PRJ-2 | HR can edit a project's details and can close or reopen it. Projects are never deleted. | M |
| PRJ-3 | HR can search projects by code, name or client name and filter by type and status. The list is paginated. | M |
| PRJ-4 | A common project is available to every active employee without assignment. The system starts with four: Internal, Training, Recruitment and Pre-sales. | M |

**Assignments**

| ID | Requirement | Priority |
|---|---|---|
| PRJ-5 | HR can assign one or more employees to a client project in a single action. Each assignment has a start date (default: today) and an optional end date. | M |
| PRJ-6 | HR can change the dates of an assignment, end it, or remove it. | M |
| PRJ-7 | HR can manage assignments from either side: open a project to see and change its assigned employees, or open an employee to see and change their projects. | M |
| PRJ-8 | An employee cannot be assigned to the same project twice for overlapping dates. A deactivated employee and a closed project cannot receive new assignments. | M |
| PRJ-9 | An employee can log time against a client project only when the project is active and the entry date falls within both the project's dates and the employee's assignment dates. | M |
| PRJ-10 | Closing a project, or ending or removing an assignment, never changes existing time entries. | M |
| PRJ-11 | An employee can see the list of projects currently available to them, with assignment dates. | S |
| PRJ-12 | HR can assign employees to a project by uploading a CSV of employee IDs. Rejected rows are reported with a reason. | C |

**Tasks**

| ID | Requirement | Priority |
|---|---|---|
| PRJ-13 | HR can maintain the list of tasks (add, rename, deactivate). A deactivated task remains on existing entries but cannot be chosen for new ones. | S |
| PRJ-14 | HR can limit which tasks are offered for a project. By default a project offers every active task. | C |

### 5.5 Time reporting — Employee (TIME)

| ID | Requirement | Priority |
|---|---|---|
| TIME-1 | The default screen after sign-in is the weekly view for the current week: seven day columns (Monday to Sunday), each showing its date, its total hours and its entries. | M |
| TIME-2 | The weekly view shows the week's total hours, the date range and the ISO week number. The user can move to the previous or next week and jump to any date. | M |
| TIME-3 | An employee can add a work entry to a day by entering: activity (free text, optional, up to 500 characters), project, task, duration and work location (Office, Client Site or Work From Home). Work location defaults to the one the employee used last. | M |
| TIME-4 | An employee can add a leave entry to a day by choosing a leave type (Casual Leave, Sick Leave, Earned Leave, Unpaid Leave) and a duration of 4 or 8 hours. | M |
| TIME-5 | Duration is entered in hours and minutes, in multiples of 15 minutes, with a minimum of 15 minutes. | M |
| TIME-6 | The total of all entries on one day, including holiday hours, cannot exceed 24 hours. | M |
| TIME-7 | An employee can edit and delete their own entries. | M |
| TIME-8 | Work entries cannot be created for a future date. Leave entries can be created up to 60 days ahead. | M |
| TIME-9 | An employee cannot add, edit or delete an entry dated more than 14 days in the past. | M |
| TIME-10 | No one except HR (through unlocking, see RPT-6) can change entries in a locked period. | S |
| TIME-11 | Entries are visually distinct by kind: work, leave and designated holiday. | M |
| TIME-12 | A day with fewer than the expected hours and a week that is not complete are visibly flagged. | M |
| TIME-13 | An employee can copy all work entries from the previous week into the current week. Entries that would break a rule (closed project, ended assignment, future date, holiday already filling the day) are skipped and listed. | S |
| TIME-14 | Two entries on the same day with the same project and task are allowed. | M |
| TIME-15 | The project selector offers only the projects the employee can use on the entry's date (PRJ-4, PRJ-9). The employee can find a project by typing part of its code or name. | M |
| TIME-16 | Each entry in the weekly view shows its task, project, duration and work location. | M |
| TIME-17 | If the same entry is edited from two browser tabs, the second save must not silently overwrite the first. The user is told the entry has changed. | S |
| TIME-18 | A keyboard shortcut opens the add-entry form for today from anywhere in the weekly view. | C |

### 5.6 Reports (RPT)

| ID | Requirement | Priority |
|---|---|---|
| RPT-1 | An employee can view their own hours for a chosen month, grouped by project and by kind (work, leave, holiday). | M |
| RPT-2 | HR can view a "missing time" report for a chosen week: every active employee whose week is not complete, with expected hours, reported hours and shortfall. Filter by department. | M |
| RPT-3 | HR can view total hours by project for a date range, with a breakdown by employee and by task. | S |
| RPT-4 | HR can view any employee's weekly view in read-only mode. | M |
| RPT-5 | HR can export RPT-2 and RPT-3 as CSV. | S |
| RPT-6 | HR can lock and unlock a calendar month. Locking requires confirmation. | S |

### 5.7 Audit (AUD)

| ID | Requirement | Priority |
|---|---|---|
| AUD-1 | Every create, update and delete of an employee, holiday, project, assignment or period lock is recorded with who did it, when, and the values before and after. | M |
| AUD-2 | Every create, update and delete of a time entry is recorded with who did it and when. | S |
| AUD-3 | HR can view the audit trail, filtered by date range, actor and record type. Audit records cannot be edited or deleted through the application. | S |

## 6. Key scenarios and acceptance criteria

These scenarios are the minimum acceptance tests. Your design should make each one traceable to a requirement ID.

**S-1: Log a normal day** (TIME-1, TIME-3, TIME-5, TIME-16)
- Given Priya is signed in and assigned to "Acme Health – Dev Pod"
- When she adds an entry for today with that project, task "Development (Coding)", work location Office and duration 8h 0m
- Then the entry appears in today's column, today's total shows 8h 0m and the week total increases by 8 hours.

**S-2: Holiday appears automatically** (HOL-1, HOL-4, HOL-5)
- Given Arjun adds a designated holiday "Founders' Day" on a Monday next month
- When Priya opens the weekly view for that week
- Then Monday shows an 8h 0m "Designated Holiday" entry that she cannot edit or delete, and the week total includes it.

**S-3: Holiday removed** (HOL-2, HOL-5)
- Given the holiday from S-2 exists
- When Arjun deletes it
- Then Priya's Monday no longer shows the holiday entry and Monday is flagged as below expected hours.

**S-4: Daily limit** (TIME-6)
- Given Priya has 20 hours of entries on one day
- When she tries to add a 5-hour entry on that day
- Then the entry is rejected with a message stating the 24-hour daily limit and nothing is saved.

**S-5: Unassigned project** (PRJ-9, TIME-15)
- Given Priya is not assigned to project "Borealis"
- When she opens the project selector, "Borealis" is not offered
- And a direct API request to create an entry against "Borealis" is rejected as forbidden.

**S-6: Role enforcement** (section 3.3)
- Given Priya has the Employee role
- When she requests any HR-only screen or API
- Then the request is refused, and no HR-only data is returned.

**S-7: Deactivation** (EMP-4, AUTH-6)
- Given Priya is signed in
- When Arjun deactivates her account
- Then within 1 minute her requests are refused, she cannot sign in again, and her past entries still appear in HR reports.

**S-8: Missing time** (RPT-2)
- Given a week with no holidays in which Priya has reported 32 hours
- When Arjun opens the missing time report for that week
- Then Priya is listed with expected 40h, reported 32h, shortfall 8h.

**S-9: Late edit** (TIME-9)
- Given an entry dated 15 days ago
- When Priya tries to edit it
- Then the edit is rejected with a message explaining the 14-day limit.

**S-10: HR creates a project and assigns employees** (PRJ-1, PRJ-5, PRJ-9)
- Given Arjun creates a client project "Borealis" starting today, and the system gives it a code
- When he assigns Priya and two other employees to it in one action, starting today
- Then "Borealis" appears with its code in Priya's project selector for today, and she can save an entry against it.

**S-11: Assignment ends** (PRJ-6, PRJ-9, PRJ-10)
- Given Priya has entries against "Borealis" last week
- When Arjun ends her assignment as of yesterday
- Then "Borealis" is not offered for an entry dated today, it is still offered for an entry dated yesterday, and last week's entries are unchanged.

**S-12: Common project** (PRJ-4)
- Given a newly created employee with no assignments
- When they open the project selector
- Then Internal, Training, Recruitment and Pre-sales are offered and no client project is.

## 7. Non-functional requirements

### 7.1 Technology stack — EDIT THIS SECTION

> **Attendees: replace the preferred choices below with your own before starting your design.** Keep the table structure so the design agent can find each decision. If you change the database or backend, re-read section 7.2 and confirm the targets are still realistic for your choice.

| Layer | Preferred choice | Notes |
|---|---|---|
| Database | PostgreSQL 16 | Relational database is required (NFR-DATA-1). |
| Backend | Python 3.12 + FastAPI | REST API with JSON. |
| Frontend | ReactJS 18 (TypeScript) | Single-page application. |
| Identity and access | Built into the application | Accepted alternative: Keycloak (OpenID Connect). Recommended only if you have used it before. See the note below the table. |
| API contract | OpenAPI 3.1 | Generated from the backend and published with the app. |
| Packaging | Podman + Podman Compose | Docker is not permitted for compliance reasons. See NFR-OPS-1. |
| Load testing | k6 or Locust | Used to verify section 7.2. |

**If you choose Keycloak:** it may satisfy AUTH-2 to AUTH-4, AUTH-7 and AUTH-8 through configuration instead of code. Your design must then state how creating, deactivating and changing the role of an employee in Kairos stays in sync with Keycloak (EMP-1, EMP-4, AUTH-6), and how Keycloak is included in the one-command local start (NFR-OPS-1). Role checks on Kairos data (section 3.3, NFR-SEC-2, NFR-SEC-3) are always enforced by the Kairos backend, whichever option you choose.

Anything not listed (ORM, migration tool, state management, UI component library, cache, session or token mechanism) is a design decision. Record each choice and its reason in your design document.

### 7.2 Performance and capacity (NFR-PERF)

| ID | Requirement |
|---|---|
| NFR-PERF-1 | The application supports **500 concurrent signed-in users** actively using it. "Active" means each user sends one request every 5 to 10 seconds, giving a sustained load of roughly 50 to 100 requests per second. |
| NFR-PERF-2 | At the load in NFR-PERF-1, **p99 server-side latency is below 100 ms** for every API used by the weekly view and by master-data screens: read a week, create, edit and delete an entry, list holidays, list projects, search employees. Latency is measured at the application server, from request received to response sent. |
| NFR-PERF-3 | Sign-in is exempt from NFR-PERF-2 because secure password hashing is deliberately slow. Sign-in p99 is below 500 ms. The system accepts a burst of 500 sign-ins within 60 seconds with no errors. |
| NFR-PERF-4 | Reports (RPT-1 to RPT-3) return within 2 seconds at p95 for a one-month range. CSV exports of up to 100,000 rows complete within 30 seconds and must not degrade NFR-PERF-2 for other users while running. |
| NFR-PERF-5 | Error rate at the load in NFR-PERF-1 is below 0.1%. |
| NFR-PERF-6 | The weekly view is usable within 2.5 seconds of navigation on a typical office laptop and connection. |
| NFR-PERF-7 | All targets hold with the data volume in NFR-PERF-8 loaded, on the reference environment: 2 application instances (2 vCPU, 4 GB each) and 1 database instance (4 vCPU, 16 GB). |
| NFR-PERF-8 | Design for 5,000 employees, 500 projects, 15,000 assignments, and 5 years of history: about 20 million time entries. |
| NFR-PERF-9 | A repeatable load test script and a data seeding script are part of the deliverable. They demonstrate NFR-PERF-1 to NFR-PERF-5. |

Expect load to be uneven. Most employees report time on Friday afternoon and Monday morning, and usage peaks again on the last working day of the month.

### 7.3 Security and privacy (NFR-SEC)

| ID | Requirement |
|---|---|
| NFR-SEC-1 | Passwords are stored only as salted hashes using a current, purpose-built password hashing algorithm. Passwords and tokens never appear in logs. |
| NFR-SEC-2 | Every API enforces authentication and role checks on the server. Hiding a button in the UI is not access control. |
| NFR-SEC-3 | An employee can never read or change another employee's time entries by altering an identifier in a request. |
| NFR-SEC-4 | All traffic uses HTTPS in any deployed environment. |
| NFR-SEC-5 | The application is protected against the OWASP Top 10, including injection, cross-site scripting and cross-site request forgery. |
| NFR-SEC-6 | All input is validated on the server. Validation errors return a consistent, documented error format. |
| NFR-SEC-7 | Employee personal data (name, email, employee ID) is returned only to the employee themselves and to HR administrators. |
| NFR-SEC-8 | Secrets and credentials are supplied through configuration, never committed to the repository. |

### 7.4 Reliability and data integrity (NFR-REL, NFR-DATA)

| ID | Requirement |
|---|---|
| NFR-REL-1 | Availability target is 99.5% between 08:00 and 22:00 IST, Monday to Saturday. |
| NFR-REL-2 | The application tier is stateless so that instances can be added or restarted without signing users out. |
| NFR-REL-3 | The application exposes liveness and readiness checks. |
| NFR-DATA-1 | Business rules that protect data (uniqueness, daily 24-hour limit, one holiday per date) hold even when requests arrive concurrently. |
| NFR-DATA-2 | Schema changes are applied through versioned, repeatable migrations. |
| NFR-DATA-3 | Dates are calendar dates in IST. Timestamps are stored in UTC. |
| NFR-DATA-4 | Time entries, employees and projects are retained for at least 7 years. |

### 7.5 Usability and accessibility (NFR-UX)

| ID | Requirement |
|---|---|
| NFR-UX-1 | Supported browsers: the latest two versions of Chrome, Edge, Firefox and Safari. |
| NFR-UX-2 | The weekly view is usable from 360 px width. On narrow screens the days stack vertically. |
| NFR-UX-3 | The application meets WCAG 2.1 level AA: full keyboard operation, visible focus, sufficient colour contrast, and no information conveyed by colour alone. |
| NFR-UX-4 | Every failed action tells the user what went wrong and what to do next. |
| NFR-UX-5 | An entry can be added with the keyboard alone in no more than 6 key interactions after opening the add form. |

### 7.6 Maintainability and operations (NFR-OPS)

| ID | Requirement |
|---|---|
| NFR-OPS-1 | A new developer can start the full application locally with one command, with seed data loaded. |
| NFR-OPS-2 | Automated tests cover every scenario in section 6. Backend line coverage is at least 80%. |
| NFR-OPS-3 | Logs are structured (JSON) and every request carries a correlation ID from the browser through to the database call. |
| NFR-OPS-4 | The application exposes request count, error count and latency metrics per endpoint. |
| NFR-OPS-5 | Linting, type checks and tests run with one command and pass before any slice is called done. |
| NFR-OPS-6 | The repository contains a README covering setup, running tests, running the load test and the environment variables used. |

## 8. Seed data

The local environment starts with:

- 1 HR administrator and 5 employees across 2 departments, with known credentials listed in the README.
- The 4 common projects (PRJ-4) and 3 client projects, with each employee assigned to at least one client project and one employee whose assignment has ended.
- 12 tasks: Analysis, Design, Development (Coding), Testing, Review, Deployment, Defect Fixing, Refactor, Documentation, Meetings, Knowledge Transfer, Project Management.
- Designated holidays for the current year, including 26 January, 15 August, 2 October and 25 December.
- Two weeks of time entries for each employee, with at least one incomplete week.

Seed data rules:

- Slice 1 has no HR screens, so in Slice 1 every employee, project, assignment and task comes from the seed script.
- The seed script grows with the application. Each slice seeds the items above that its tables support. For example, holidays are seeded from Slice 3.
- The seed script is idempotent and deterministic, and uses dates relative to the day it runs, so the current and previous week always contain entries.
- Automated tests create their own data and never depend on the seed.

The separate seeding script for load testing (NFR-PERF-9) generates the volume in NFR-PERF-8.

## 9. Delivery slices

Build and verify one slice at a time. Each slice is a working, tested, demonstrable application.

| Slice | Contents | Requirements |
|---|---|---|
| **1. Sign in and log time** | Sign-in, seed data, weekly view, add/edit/delete work entries, daily and weekly totals | AUTH-1, AUTH-5, TIME-1 to TIME-3, TIME-5 to TIME-7, TIME-14 to TIME-16, PRJ-4, PRJ-9 |
| **2. HR master data** | Employee management, project creation, assigning employees to projects, role enforcement | EMP-1 to EMP-5, EMP-7, EMP-8, PRJ-1 to PRJ-3, PRJ-5 to PRJ-8, PRJ-10, AUTH-2 to AUTH-4, AUTH-6 |
| **3. Holidays and leave** | Holiday management, automatic holiday entries, leave entries, date rules, flags | HOL-1 to HOL-6, TIME-4, TIME-8, TIME-9, TIME-11, TIME-12 |
| **4. Reports and audit** | Own monthly report, missing time report, HR read-only view, master-data audit | RPT-1, RPT-2, RPT-4, AUD-1 |
| **5. Performance proof** | Volume seeding, load test, tuning until section 7.2 is met | NFR-PERF-1 to NFR-PERF-9 |
| **Stretch** | All S and C requirements | Remaining IDs |

## 10. Questions the design must answer

The PRD deliberately does not decide these. Your design document should state a decision and a reason for each.

1. How are designated holiday entries represented: stored as one row per employee, or derived when a week is read? What does each option cost when HR adds a holiday for 5,000 employees, and when a week is read at p99 < 100 ms?
2. How is the signed-in session carried and how is AUTH-6 (revocation within 1 minute) achieved without breaking NFR-REL-2 (stateless application tier)?
3. How is the daily 24-hour limit enforced when two requests for the same day arrive at the same time (NFR-DATA-1)?
4. How does the 20-million-row time entry data stay fast for "read one employee's week" and for the organisation-wide missing time report? Which indexes, and is partitioning or pre-aggregation needed?
5. How do CSV exports run without affecting interactive latency (NFR-PERF-4)?
6. How is lost-update protection (TIME-17) implemented and surfaced to the user?
7. Is a cache needed to meet NFR-PERF-2? If so, what is cached and how is it invalidated when HR changes master data?
8. How is the audit trail written so that it cannot be skipped by a new code path?

## 11. Definition of done

A slice is done when:

- Every requirement in the slice is implemented and traceable from the design document to code and to at least one test.
- All section 6 scenarios that belong to the slice pass as automated tests.
- NFR-OPS-5 passes.
- The design document matches the code. If they differ, one of them has been updated.
