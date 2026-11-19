# Comprehension Report (Stage 6)

**Target:** `Kairos/docs/designs/kairos.md` (FeatureRange: lines 1-1067; DetailedRange: lines 1068-1704)
**Reviewer:** `egm-comprehension` (completed). No document standard. No Goldfish context: the reviewer read the target document alone.
**Result:** 0 Blockers, 8 Major, 7 Nits. The reviewer could paraphrase every section.

## What this proposes

Kairos is a web app where about 5,000 employees log their working hours each week against projects HR has opened to them, and about 10 HR admins manage employees, projects, assignments and holidays. It replaces spreadsheets and email reminders. The design picks a single-database architecture: a Caddy proxy serves the React app, two stateless FastAPI instances sit behind it, and PostgreSQL does all session lookup, locking and auditing. There is no Redis, queue or cache. Holidays are stored once and turned into hours when a week is read, and a per-day totals row locked during each save enforces the 24-hour daily limit. The detailed plan breaks Slice 1 (sign in, weekly view, add, edit and delete work entries, the project selector, and version-checked edits) into ten sub-tasks, ST-01 to ST-10, each with its files and tests.

## How the existing system works, per this document

Nothing is built yet: Slice 1 starts from an empty repository with no existing data. Today, time is collected in spreadsheets and HR chases missing entries by email. Project names are typed freely, holidays and leave are recorded in different ways, and there is no audit trail or way to close a month. The requirements live in prd.md v1.3, and the files in context/ (design theme, engineering standards) are treated as fixed constraints.

---

## Findings

### 1. Locking and editing rules are referenced but never described - Severity: Major
* **Affected scope:** The Problem, 4th bullet (lines 69-72); also lines 707-711, 826-828, 1052-1055 and 1469-1470.
* **Gap:** The resolution of the "unlocking a month older than 14 days" conflict depends on rules the document never describes: HOL-2, TIME-8, TIME-9, RPT-6, TIME-10 and PRD section 3.3. A new reader cannot tell what locking a month means, what the 14 days are counted from, or what TIME-9 forbids, so they cannot check whether the resolution is right. The same undescribed date rules drive `editable`, the `DATE_NOT_EDITABLE` error and `BELOW_EXPECTED`, which are part of the Slice 1 response shape.
* **Recommendation:** Add one sentence each for TIME-8, TIME-9, HOL-2 and RPT-6/period locks, either inline in The Problem or in a short "business rules this design relies on" list.

### 2. The eight PRD section 10 questions are never stated - Severity: Major
* **Affected scope:** Overview (lines 20-23), Design goals (lines 89-90), Technical Plan section 4 (lines 350-498), Q8-B (lines 955-956).
* **Gap:** The document's main purpose is to answer "the eight questions in PRD section 10", but it never states them. Each Q heading gives only the decision. Q8-B argues from "the 'new code path' risk that PRD question 10.8 asks the design to close", which the reader has never seen.
* **Recommendation:** Put a one-line restatement of each question above its decision.

### 3. No rule for editing a day that a holiday has already pushed over 24 hours - Severity: Major
* **Affected scope:** The Problem, 1st bullet (lines 58-62); Q3 step 3 (lines 411-413); the `OVER_DAILY_LIMIT` flag (lines 708-709); `update_work_entry` (lines 1462-1466).
* **Gap:** A holiday can push a day over 24 hours, and the day is then flagged. But an edit is checked as "existing + holiday + new - old ≤ 1,440". Take a day with 20h of entries and an 8h holiday (28h). If the employee shortens a 5h entry to 4h, the day still totals 27h, so the edit is rejected. The only way out is to delete entries. The document never says whether this is intended or how an employee is expected to clear the flag.
* **Recommendation:** State the rule for days already over the limit, for example "an edit that does not increase the day's total is always allowed". Add a test for it.

### 4. "Friday peak" and the reference environment are never defined - Severity: Major
* **Affected scope:** "Friday peak" (lines 853, 884, 986, 1013-1015); reference environment (lines 84, 229, 434).
* **Gap:** "Friday peak" justifies several rejected alternatives and the riskiest performance targets, but it is never defined (when it happens, how many users, what request rate). The load figures are scattered: 500 concurrent users, about 100 requests a second, 500 sign-ins in 60 seconds. The database instance's size is never given; it can only be guessed from "16 GB of memory" (line 434).
* **Recommendation:** Add one short paragraph defining the Friday peak and the reference environment's sizing (API and database), and point the rationales to it.

### 5. The Docker rule has no source, and the default images may break it - Severity: Major
* **Affected scope:** Non-Goals "Docker tooling" (lines 133-135), "`podman compose`" (lines 975-978), Container images (lines 1020-1025).
* **Gap:** "Docker is not permitted" / "the compliance rule" drives tooling choices but is not linked to any source. The document also admits it does not know whether the rule covers Docker Hub, yet every default image is `docker.io/...`. If the rule does cover Docker Hub, the default `make up` breaks. This question is missing from Open Questions.
* **Recommendation:** Link the source of the rule and add "Does the Docker rule cover Docker Hub images?" to Open Questions, with an owner, before ST-01.

### 6. Sign-in counters write audited `employees` rows, and the audit copy may include `password_hash` - Severity: Major
* **Affected scope:** Data dictionary `employees` (lines 522-524), `employee_preferences` rationale (line 539), Q8 (lines 468-480).
* **Gap:** `last_work_location` is kept out of `employees` "so that saving an entry does not write an audited employee row". The same reasoning is not applied to `failed_sign_in_count` and `locked_until`. Every failed sign-in (and presumably every successful one that resets the count) will write an audited `employees` row. These writes happen during an unauthenticated request, where `audit_row()` raises an error because no actor is set. Q8 also writes whole old and new rows as JSON for master data, which would copy `password_hash` into `audit_log`. These columns are fixed in Slice 1, and the design goal (lines 93-98) says Slice 1's tables must not need rewriting later.
* **Recommendation:** Either move the sign-in counters into a separate table that is not audited, or state the actor used for sign-in writes. Also list the columns `audit_row()` leaves out of the JSON (at least `password_hash`).

### 7. "S and C" and "Stretch" are undefined - Severity: Major
* **Affected scope:** "S and C requirements" (line 120), "Stretch" (lines 724, 1061).
* **Gap:** "S and C" is never defined (probably MoSCoW Should and Could), and "Stretch" is used like a sixth slice without being introduced. Line 120 sets a scope boundary of this design in these terms, so a reader cannot tell which requirements are out of scope.
* **Recommendation:** Define the priority labels in one sentence and say whether "Stretch" is a slice or a priority bucket.

### 8. Test database setup does not say how roles and extensions are created - Severity: Major (Detailed Implementation Plan)
* **Affected scope:** ST-02 `conftest.py` (lines 1233-1246), ST-03 tests (line 1293), Verification Plan (lines 1687-1689).
* **Gap:** `make test-backend` starts a throwaway PostgreSQL on port 55433, but the plan does not say whether that container runs `db/init/01-roles.sh`. That script is what creates `kairos_owner`, `kairos_app` and `btree_gist`. Without them, the migration (which connects as `kairos_owner`) and the exclusion constraint fail. It is also not said which role the app under test connects as. That matters because `test_schema.py` checks that `kairos_app` cannot drop tables.
* **Recommendation:** State that the test container mounts `db/init/`, or that `conftest.py` creates the roles and extension. Name the role used by migrations and the role used by the app in tests.

### 9. Slices 1-5 are not introduced until line 1041 - Severity: Nit
* **Affected scope:** Overview (lines 20-26).
* **Gap:** Slices 1 to 5 are referred to everywhere, but their names and contents first appear at line 1041. Only Slice 1's name appears early (line 24).
* **Recommendation:** Add a five-row slice table in the Overview.

### 10. G-1 counts "key presses" but the design counts "interactions" - Severity: Nit
* **Affected scope:** G-1 (lines 78-80), Open Questions (lines 144-148), Keyboard path (lines 339-345), `Me` shape (line 680).
* **Gap:** G-1 says "6 key presses", but the design counts "interactions", and under a key-press reading the goal cannot be met. The 4-step path also assumes Location has a remembered default. For a first entry, `lastWorkLocation` is null and no fallback is stated.
* **Recommendation:** Reword G-1 to match NFR-UX-5's term, and state the default location when none has been used yet.

### 11. "Expected minutes" is defined in two places - Severity: Nit
* **Affected scope:** Q1 (lines 356-372) vs `read_week` (lines 1441-1445).
* **Gap:** Q1 says the week read goes through `employee_holidays` and `employee_day_hours`. But `read_week` works out `expectedMinutes` in Python ("480 on each weekday on or after the date of joining"), so "expected minutes" is defined in two places.
* **Recommendation:** Either have `read_week` use `employee_day_hours`, or say explicitly that only holiday minutes must go through SQL.

### 12. AUD-2 and AUD-3 are assigned to different slices in different places - Severity: Nit
* **Affected scope:** Line 120, Q8 (lines 468-480), Future Improvements (lines 1056 and 1062).
* **Gap:** Q8 (Slice 4) delivers both. Line 120 lists AUD-2 as a designed S/C item but does not mention AUD-3. Future Improvements puts AUD-2 and AUD-3 under Stretch, and gives Slice 4 only AUD-1.
* **Recommendation:** Pick one slice for each requirement and use it everywhere.

### 13. 428 for DELETE without `If-Match` is not in the router or tests - Severity: Nit
* **Affected scope:** ST-07 router (lines 1476-1478) and tests (lines 1505-1506) vs the API table (line 672).
* **Gap:** The API contract says DELETE without `If-Match` returns 428, but the router bullet mentions 428 only for PUT, and the test does not say which method it covers.
* **Recommendation:** Name the 428 for DELETE in the router bullet and test both methods.

### 14. Some files are missing from the sub-task file lists - Severity: Nit
* **Affected scope:** The claim that the plan lists every file (line 25) vs the ST lists.
* **Gap:** Several files are missing from the lists:
  * `frontend/src/api/openapi.json`, which is committed (lines 1547-1549) but not listed.
  * Package `__init__.py` files for `health`, `security`, `auth`, `projects`, `tasks` and `time_entries`.
  * The file that holds the e2e "test-only SQL helper".
  * How `e2e/` gets Playwright and its Node dependencies, since Playwright is declared in `frontend/package.json` but the config lives in `e2e/`.

  This matters because `implementation-plan.md` is generated from this section (line 1079).
* **Recommendation:** List these files, or say which directory and `package.json` the e2e suite runs from.

### 15. Small terms used without explanation - Severity: Nit
* **Affected scope:** Various.
* **Gap:**
  * "Arjun" (line 767) is never introduced as an HR admin.
  * What "Apply my changes again" does (lines 456-457 and 1612-1613) is not stated (resend with the new version?).
  * AUTH-3 is cited as the reason the seed password is valid (line 1350) but never summarised.
  * RPT-4, HOL-8, AUTH-7, AUTH-8 and PRJ-11 to PRJ-14 appear only as bare IDs.
* **Recommendation:** Add a one-line explanation for each, or a short glossary of PRD IDs used in this document.
