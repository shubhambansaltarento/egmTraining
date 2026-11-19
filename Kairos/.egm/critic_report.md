# Goldfish Review Report

**Target:** `/Users/shubhambansal/Documents/Code/sample-proj and handbook/Kairos/docs/designs/kairos.md` (`FeatureRange, lines 1-1067`)

Stage 7, Critic Review. Standard for `egm-design` and `egm-structural`: `/Users/shubhambansal/.claude/plugins/cache/egm/egm/0.1.1/resources/document_standards/feature-spec-standard.md`. `egm-style` applied its persona's REVIEW instructions with no document standard. Background reading: `prd.md`, `context/design-theme.md`, `context/engineering-standards.md`, `.egm/decision_options.md`.

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-design` | completed |
| `egm-structural` | completed |
| `egm-style` | completed |

**Severity breakdown (after deduplication):** 2 Blocker, 19 Major, 24 Nit (45 findings).

Triage notes:
- Overlapping findings from more than one lens are merged into one entry, and every contributing lens is listed.
- Two findings were raised above what one reviewer proposed:
  - Credential columns audited: `egm-design` proposed Major and said it could be a Blocker. The synthesizer checked lines 468-480 and 508-524 and confirmed the problem.
  - Missing Risks section: `egm-design` proposed Major and `egm-structural` proposed Blocker. The standard says the spec "must explicitly call out its own constraints".
- Rejection attribution: `egm-structural` proposed Major, while `egm-design` and `egm-style` proposed Nit. The finding is kept at Major.

## Prioritized Findings (Blockers -> Major -> Nits)

### Credential and lock-out columns sit on the audited `employees` table - Severity: Blocker
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 508-524` (`employees` data dictionary), `Lines 468-480` (Q8), `Line 539`
* **Discrepancy Description**: These columns live on `employees`:
  * `password_hash`
  * `must_change_password`
  * `failed_sign_in_count`
  * `locked_until`

  Q8 attaches `audit_row()` to every insert, update and delete on `employees`. The function raises an error when `kairos.actor_id` is empty, and it writes the old and new row as JSON for master data. That has three effects:
  * **Sign-in stops working once Slice 4 adds the triggers.** A failed sign-in (AUTH-4) updates the counter on a request with no signed-in user, so no actor is set and the trigger raises an error. A successful sign-in that resets the counter fails the same way.
  * **Password hashes are copied into `audit_log`.** That table cannot be deleted from and is readable by HR through `GET /audit-events` (AUD-3). This conflicts with NFR-SEC-1 and NFR-SEC-7.
  * **Audit volume grows.** Every sign-in attempt writes an audit row, which is about 500 rows in a sign-in burst.
* **Rationale & Recommendation**: The design already moved `last_work_location` out of `employees` for this reason (line 539) but did not do the same for credential state. Move the credential and lock-out columns into a separate `employee_credentials` table that is not audited. Alternatively, have `audit_row()` drop these columns and skip updates that only touch them. Either way, say how sign-in writes that have no actor work. Add a test that `audit_log` never contains `password_hash`.

### No Risks and Mitigations section - Severity: Blocker
* **Reviewer Lens:** `egm-structural`, `egm-design`
* **Affected Scope:** Whole range. The new section belongs between "Alternatives Considered" (line 839) and "Non-Functional Requirements" (line 999).
* **Discrepancy Description**: The standard requires the spec to name its constraints, failure modes, security implications, scalability limits and their mitigations. No such section exists. Some risk material is spread across Q2-Q8 and the NFR bullets, and line 1013 names the "riskiest targets" without saying why. Most "Fails when" entries for the chosen options in `decision_options.md` were dropped. These risks are not stated anywhere:
  * **Single points of failure.** There is one PostgreSQL instance and one Caddy proxy, measured against the 99.5% availability target (NFR-REL-1).
  * **Backups.** Nothing covers backups for the 7-year retention rule (NFR-DATA-4).
  * **The Q8 audit trigger.** It raises an error when no actor is set, so a missed `set_config` call becomes a write outage.
  * **`day_totals` drift.** The totals can drift from `time_entries` if the trigger is disabled or bypassed, for example by the Slice 5 volume seed.
  * **Sign-in hashing.** Sign-ins queue behind the cap of 4 concurrent hashes per instance.
  * **Exports.** Only 1 export runs per instance, so HR users block each other, and export CPU competes with interactive requests on 2 vCPU.
  * **`audit_log` growth.** The table passes 20 million rows under AUD-2.
  * **Account lock-out.** AUTH-4 lets anyone lock any account whose email they know.
  * **Holiday changes.** Changing a holiday silently changes the completeness of past weeks (decision_options Q1-B).
  * **Pool exhaustion.**
* **Rationale & Recommendation**: This section is required by the standard. Add an H2 "Risks and Mitigations" after Alternatives Considered. Use a table with these columns: risk, kind (failure, security, scalability, complexity), trigger, impact, mitigation or accepted residual risk, and the slice that verifies it. Move the scattered items into it and add the ones above.

### Slice labels for audit, exports and HOL-7 disagree across sections - Severity: Major
* **Reviewer Lens:** `egm-structural`, `egm-style`
* **Affected Scope:**
  * `Lines 120-122` (Non-Goals)
  * `Lines 438-439` (Q5 heading)
  * `Lines 468-480` (Q8)
  * `Lines 719-725` (Slice 2-5 endpoint table)
  * `Lines 819-837` (section 8)
  * `Lines 1039-1064` (Future Improvements)
* **Discrepancy Description**: Section 8 and Future Improvements are two separate slice roadmaps, and they do not agree:
  * **AUD-2.** Non-Goals calls it designed, and Q8 and section 8 put the `time_entries` trigger in Slice 4. Future Improvements lists AUD-2 as Stretch and Slice 4 as AUD-1 only.
  * **AUD-3.** Q8 designs the INSERT/SELECT-only grant, and the endpoint table puts `GET /audit-events` in Slice 4. Future Improvements lists AUD-3 as Stretch.
  * **RPT-5 exports.** Q5 is headed "(Slice 4 and stretch)". The PRD's Slice 4 has no exports, and both the endpoint table and Future Improvements put exports under Stretch.
  * **HOL-7.** Its behaviour is defined at line 375, but it is listed as Stretch.

  A reader cannot tell what ships in Slice 4.
* **Rationale & Recommendation**: Keep one requirement-to-slice table with one row per requirement ID. Either merge it into section 8 or have it replace Future Improvements. Have Non-Goals, Q5, Q8 and the endpoint table refer to it. Label Q5 "(Stretch)". In Q8, either separate the Slice 4 work (AUD-1) from the stretch work (AUD-2, AUD-3), or move AUD-2 and AUD-3 into Slice 4 and say this departs from the PRD.

### Non-Goals list of designed S and C items is incomplete, and its "one line each" promise is not kept - Severity: Major
* **Reviewer Lens:** `egm-style`, `egm-structural`
* **Affected Scope:** `Lines 120-122` (Non-Goals, second bullet), `Lines 1061-1064`
* **Discrepancy Description**: The bullet says only TIME-17, AUD-2, RPT-5 and RPT-6 are designed, "because PRD section 10 questions 5, 6 and 8 depend on them". RPT-6 (month locking) is not tied to any of those questions. The range also designs other S and C items:
  * AUD-3: the grant at line 476
  * HOL-7: line 375
  * PRJ-13: `tasks.is_active`, line 571
  * TIME-13 copy-week: line 369
  * the `period_locks` table: line 607

  The bullet also promises that every other S and C item "gets one line saying where it would fit". In practice they share one combined line of IDs, most with no location (lines 1061-1064).
* **Rationale & Recommendation**: Either list every S and C item the range designs, with the question that pulls each one in, or remove the design detail for items that are out of scope. Replace the "one line each" promise with what the document provides, or provide it in the requirement-to-slice table above.

### Rejections are attributed to "the author" without a durable decision record - Severity: Major
* **Reviewer Lens:** `egm-structural`, `egm-design`, `egm-style`
* **Affected Scope:**
  * `Lines 842-843`
  * every "Reason for Rejection" bullet in `Lines 855-995` (20 occurrences)
  * `Lines 139-141`
  * `Line 352`
* **Discrepancy Description**: The standard does not allow a rejection to be attributed to a person without a linked issue, discussion or PR. Every rejection starts with "Rejected by author:". The only link is to `../../.egm/decision_options.md`, which has three problems:
  * Its header says the options are not ranked and that "[starting answer]" is not a recommendation, so it records no decision.
  * It sits in the untracked `.egm/` working directory, so the link may not survive publication.
  * The prefix repeats line 842-843 twenty times.
* **Rationale & Recommendation**: The rationales after the colon state concrete trade-offs, so removing the prefix loses nothing. Drop "Rejected by author:" and the sentence "Each rejection below was decided by the author on 2026-11-19". If the decision record matters, link a committed decision log, issue or PR. Otherwise stop citing `decision_options.md` as the source.

### Non-Functional Requirements section has coverage gaps and a false cross-reference - Severity: Major
* **Reviewer Lens:** `egm-structural`, `egm-style`
* **Affected Scope:** `Lines 999-1025`, specifically `Lines 1010-1013`
* **Discrepancy Description**: The section covers only Security, Performance, Rollout and Container images. It has no entries for Reliability (NFR-REL), Data (NFR-DATA), UX/Accessibility (NFR-UX) or Operability (NFR-OPS). These requirements are not traced anywhere in lines 1-1067:
  * NFR-DATA-4: keep data for 7 years
  * NFR-SEC-7: personal data is returned only to the employee and HR
  * NFR-UX-1: supported browsers
  * NFR-PERF-6: weekly view usable within 2.5 s

  Line 1010 says "Section 4 explains how each NFR-PERF target is approached", but section 4 covers only NFR-PERF-2, -3 and -7. NFR-PERF-5 is mentioned only under Q3-C, and NFR-PERF-4 and NFR-PERF-6 are not addressed.
* **Rationale & Recommendation**: Add a table that maps each NFR ID to the section that meets it, or to "deferred to Slice N". Correct the claim on line 1010, and give one clause on why the targets on line 1013 are the riskiest.

### Additional Resources and Future Improvements are nested under Non-Functional Requirements - Severity: Major
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Line 1029` ("### Additional Resources"), `Line 1039` ("### Future Improvements")
* **Discrepancy Description**: Both headings follow the horizontal rule at line 1027, but because they are H3 they sit under "## Non-Functional Requirements" in the outline. Future Improvements carries the slice roadmap, and readers will not look for it under NFRs.
* **Rationale & Recommendation**: Promote both headings to H2.

### Mermaid sequence diagrams use semicolons in message text - Severity: Major
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Lines 747, 755` (S-1), `Line 776` (S-2), `Line 813` (S-5)
* **Discrepancy Description**: Mermaid treats `;` in sequence-diagram text as the end of a statement. Message text such as `BEGIN; set_config(...)` is therefore likely to split into invalid statements, and the three diagrams that section 7 promises would fail to render. The reviewer did not confirm this in a renderer.
* **Rationale & Recommendation**: Replace `;` with a comma or "then", or use the `#59;` escape. Render each diagram once, for example in a GitHub preview or with mermaid-cli, to confirm.

### Line 301 refers to an error-code list that does not exist - Severity: Major
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Line 301` (section 2, "Errors"), section 6 (`Lines 645-725`)
* **Discrepancy Description**: Line 301 says error codes are "listed in section 6". Section 6 has no list of codes, only codes scattered through table cells. These codes appear elsewhere but nowhere in section 6:
  * `INTERNAL_ERROR` (line 305)
  * `EXPORT_BUSY` (line 442)
  * `DATE_NOT_EDITABLE` (line 827)
* **Rationale & Recommendation**: Add an "Error codes" table to section 6 with code, HTTP status, message text and slice. Have the endpoint tables cite it.

### G-1 counts "key presses" while the design counts "interactions" - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 78-80` (G-1), `Lines 144-148` (Open Questions), `Lines 339-345` (keyboard path), G-4
* **Discrepancy Description**: G-1 says "no more than 6 key presses after the add form opens (NFR-UX-5)", but NFR-UX-5 says "6 key interactions". The keyboard path reaches 4 only by counting "type part of a name and press Enter" as one interaction. Read literally, the design does not meet its own goal. G-1 is labelled "taken from PRD section 2.1", but the PRD's G-1 says nothing about the keyboard. G-4 has the same problem with NFR-PERF-7 and NFR-PERF-8.
* **Rationale & Recommendation**: In G-1, write "key interactions (counting rule in Open Questions)", or move the keyboard target to its own line that cites NFR-UX-5. Correct the "from PRD section 2.1" attribution on G-1 and G-4.

### "Kairos solves these" claims more than the core slices deliver - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 47-53` (The Problem)
* **Discrepancy Description**: Problem 3 includes having no way to close a month, but month locking (RPT-6) is a stretch item. Master-data audit arrives in Slice 4, and time-entry audit (AUD-2) is stretch. "Solves" implies all three problems are fully fixed.
* **Rationale & Recommendation**: Use "addresses", and say which parts are core and which are stretch.

### `day_totals` CHECK is described as a "last guard" but does not cover the 24-hour rule - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 418-420` (Q3), `Lines 876-878` (Q3-B)
* **Discrepancy Description**: `CHECK (entry_minutes BETWEEN 0 AND 1440)` limits entry minutes only. The 24-hour rule adds holiday minutes to the entries. On a day with a 480-minute holiday, entries can reach 1,440 minutes, a total of 1,920, and the CHECK still passes. "Last guard" is used twice and implies a backstop the CHECK does not provide.
* **Rationale & Recommendation**: State what the CHECK guards: "it stops entry minutes alone from going over 1,440; it does not include holiday minutes."

### Request-ID SQL comment breaks the prepared-statement cache and contradicts "no SQL is built from strings" - Severity: Major
* **Reviewer Lens:** `egm-design`, `egm-style`
* **Affected Scope:** `Lines 271-275` (request pipeline step 1), `Lines 1007-1008` (Security)
* **Discrepancy Description**: The client-supplied `X-Request-ID` is added to every statement as `/* request_id=... */`. That causes two problems:
  * **Performance.** Every SQL text becomes unique, so the asyncpg prepared-statement cache misses on every query. Each query gets an extra prepare round trip, and useful cache entries keep getting evicted. This hurts p99 on the paths NFR-PERF-2 measures.
  * **Security.** A header value is inserted into SQL text, where bound parameters do not protect it. Only the out-of-range implementation plan (line 1215) restricts the value to a UUID. The Technical Plan does not mention that check, and line 1008 states "no SQL is built from strings".
* **Rationale & Recommendation**: Carry the ID with `set_config('application_name', <id>, true)` or a custom setting at the start of each transaction, and log it through `log_line_prefix`. That keeps statement text stable. If the comment stays, state the UUID-only check in step 1, narrow the claim on line 1008, and measure the cache effect in Slice 5.

### Performance and size estimates are stated as measured facts - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:**
  * `Lines 397-399` (Q2 hashing)
  * `Lines 433-434` (Q4)
  * `Lines 856-857` (Q2-A)
  * `Lines 867-869` (Q2-B)
  * `Lines 919-920` (Q5-B)
  * `Lines 969-970` (Redis)
  * `Lines 989-992` (bcrypt)
* **Discrepancy Description**: Several estimates are stated as facts and used to reject alternatives:
  * "under 1 ms per week read"
  * "under 0.5 ms" per primary-key lookup
  * "about 1 second of CPU" for a 100,000-row export, where decision_options says 0.5 to 1 s
  * "PostgreSQL handles session lookup at this load"
  * "argon2id at about 50 to 80 ms leaves room"

  The Q2-B sentence "Every request already reads PostgreSQL, so one primary-key lookup costs under 0.5 ms" does not follow: the first part does not imply the second. The document itself says the targets are proven only in Slice 5 (lines 1010-1012).
* **Rationale & Recommendation**: Mark each figure as an estimate, name its source, and name the test that will confirm it. Rewrite Q2-B so the cost is a separate claim. In Q2, give the per-hash estimate next to the load (8 sign-ins a second on 4 vCPU) and state the conclusion against the 500 ms p99 target.

### Docker Hub compliance question is buried in an NFR bullet - Severity: Major
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 1020-1025` (Container images), `Lines 137-152` (Open Questions)
* **Discrepancy Description**: The default images come from `docker.io`, followed by "if the compliance rule against Docker also covers Docker Hub". That condition is an open compliance question, and its answer decides where images come from. It is not listed in Open Questions. PRD section 7.1 says only that "Docker is not permitted".
* **Rationale & Recommendation**: Move it to Open Questions, with who decides and by when (before Slice 1's compose file is fixed). Leave a pointer in the NFR bullet.

### The 24-hour check blocks fixes on a day a holiday pushed over the limit - Severity: Major
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 59-62` (The Problem), `Lines 411-413` (Q3 step 3), `Lines 706-709` (`OVER_DAILY_LIMIT`)
* **Discrepancy Description**: A holiday is allowed to push a day past 24 hours, and the day is then flagged. But Q3 rejects any create or edit whose result is over 1,440 minutes. On a flagged day, shortening an entry or changing only its activity text still gives a total over 1,440, so the edit gets `DAILY_LIMIT_EXCEEDED`. The employee can delete entries but cannot correct them. Q3 also does not say whether the "old duration" is read after the `FOR UPDATE` lock.
* **Rationale & Recommendation**: Reject an edit only if the new total is over 1,440 and higher than the current total. Then any edit that lowers the total is accepted. State that the old duration is read after the lock, in the same transaction.

### `employee_day_hours` cannot be the only path for every report, and it drops deactivated employees - Severity: Major
* **Reviewer Lens:** `egm-design`, `egm-style`
* **Affected Scope:** `Lines 364-372` (Q1), `Lines 426-427` (Q4 indexes), `Lines 829-830` (Slice 4)
* **Discrepancy Description**: The function has three problems:
  * **No project or task breakdown.** Line 369 routes RPT-1, RPT-3 and the exports through these functions, but `employee_day_hours` returns day-level totals only. RPT-1 needs totals by project and kind, and RPT-3 by project, employee and task. Q4 says RPT-1 and RPT-3 read `time_entries` through their own indexes, so the document contradicts itself.
  * **Deactivated employees disappear.** The function returns active employees only, but S-7 requires a deactivated employee's past entries to stay in HR reports.
  * **No employee parameter.** `employee_day_hours(from, to)` covers every employee. A per-employee read is cheap only if the function is an inlinable `LANGUAGE SQL STABLE` function, and the design does not say which kind it is.

  The guard test on lines 370-372 ("no repository adds up `time_entries` hours without going through them") has no stated mechanism, so it cannot be checked.
* **Rationale & Recommendation**:
  * Narrow the rule: the functions are the only source of holiday and expected minutes. Project and task breakdowns sum `time_entries` directly.
  * Make the active-only filter a parameter, or limit it to RPT-2.
  * Add an employee filter, or state that the function is inlinable.
  * Say how the guard test works, for example a static scan of repository SQL with a list of allowed exceptions.

### `day_totals` trigger behaviour is unspecified, so "no code path can bypass it" depends on it - Severity: Major
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 401-415` (Q3), `Line 632` (functions table)
* **Discrepancy Description**: The service creates the `day_totals` row (step 1), and the trigger only "adds the change". These writers do not go through the service:
  * the seed script
  * the Slice 5 volume seed
  * copy-week (TIME-13)
  * any future bulk tool

  If the trigger only runs `UPDATE`, their inserts affect zero rows and the totals drift without anyone noticing. The same happens when an edit moves an entry to a date with no row. Loading 20 million rows through a row-level trigger also has a seeding cost that is not mentioned.
* **Rationale & Recommendation**: Specify that the trigger upserts `day_totals` (`INSERT ... ON CONFLICT DO UPDATE`) for both the old and the new date. Add a consistency check as a guard, such as the nightly comparison from decision_options Q3-A. State how the volume seed fills `day_totals`, for example by disabling the trigger and running one aggregate insert afterwards.

### Process model per instance is not specified - Severity: Major
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 227-245` (runtime components), `Lines 264-266` (database access), `Lines 395-399` (Q2), `Lines 441-443` (Q5)
* **Discrepancy Description**: Each instance gets 2 vCPU, but the design never says how many uvicorn workers run on it. ST-01 (line 1138) starts one process, which uses about one core, so half the CPU is idle at the Friday peak. That puts the NFR-PERF-2 p99 target at risk. If workers are added later, several numbers silently multiply:
  * the connection pool (20 + 10)
  * the 4-hash cap, which line 1301 already calls "per process"
  * the 1-export cap
  * the `/metrics` counters

  Total connections are never compared with PostgreSQL `max_connections`.
* **Rationale & Recommendation**: State the worker count per instance. Restate each cap as per process, with instance totals. Add a connection budget: workers × instances × (pool + overflow + export pool), plus migration and admin connections, compared with `max_connections`.

### Sign-in lets anyone use up expensive hashing, and overload behaviour is undefined - Severity: Major
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 395-399` (Q2 hashing), `Line 664` (sign-in row)
* **Discrepancy Description**: Each sign-in costs 64 MiB and about 65 ms of CPU, and anyone can trigger one without an account. Unknown emails also run a dummy verification. Four concurrent hashes on 2 vCPU use all the CPU, so junk sign-ins starve interactive requests and queue legitimate sign-ins. The design does not say what happens when the semaphore is full. It has no per-IP or global rate limit, and the 500-sign-in burst covers legitimate load only. AUTH-4 lock-out lets anyone lock any account whose email they know.
* **Rationale & Recommendation**:
  * Set the hash limit to the number of cores or fewer.
  * Bound the wait with a timeout that returns 503 or 429.
  * Add a per-IP rate limit on `/api/v1/auth/sign-in` in Caddy.
  * Record lock-out abuse as an accepted risk, or mitigate it, for example by locking per account and IP.
  * Include abusive sign-ins in the Slice 5 load test.

### Caddy load balancing has no health checks or graceful restart - Severity: Major
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 219-255` (runtime components), `Lines 128-132` (Non-Goals, production deployment)
* **Discrepancy Description**: Caddy uses round robin with no health checks or retries. While one API instance restarts, which NFR-REL-2 is meant to allow, about half of requests fail. That uses up the 0.1% error budget in NFR-PERF-5. The design also does not say where Caddy runs in the reference environment (2 app VMs, 1 DB VM). If it runs on an app VM, TLS and static-file serving share the 2 vCPU the p99 target is measured on.
* **Rationale & Recommendation**: Configure `health_uri /api/v1/health/ready`, passive health checks and `lb_try_duration`. Give uvicorn a graceful shutdown timeout. State where Caddy runs during the Slice 5 load test.

### Chosen decisions are bold paragraphs while rejected options have H3 headings - Severity: Nit
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Lines 356-497` (section 4), `Lines 508-628` (section 5), `Lines 649, 660, 676, 717` (section 6)
* **Discrepancy Description**: Rejected options have anchors, but chosen options do not, so a decision cannot be linked to. Section 4 decisions also never give their option ID, so a reader cannot tell what "Q1-A" is being compared against.
* **Rationale & Recommendation**: Use H4 headings with option IDs, for example "#### Q1-B (chosen): worked out when a week is read". Do the same for the data-dictionary tables and API sub-blocks.

### Missing alternatives and a weak rationale in the infrastructure list - Severity: Nit
* **Reviewer Lens:** `egm-structural`, `egm-design`
* **Affected Scope:** `Lines 352-354`, `Lines 965-995`
* **Discrepancy Description**: Line 352 says the turned-down options are summarised under Alternatives Considered, but some are missing:
  * Redux Toolkit Query and nginx from decision_options Q9
  * k6 over Locust, both of which the PRD allows

  The subsection uses single bullets instead of the Pros / Cons / Reason for Rejection format used elsewhere. The reason for rejecting sync SQLAlchemy ("40 worker threads would cap" concurrency) is weak, because the 20 + 10 pool caps concurrency in both designs.
* **Rationale & Recommendation**: Add the missing entries, use the same format or say plainly that minor tooling uses a shorter one, and restate the async rationale.

### Scenario-to-slice note is unclear and sits under CUJ 6 - Severity: Nit
* **Reviewer Lens:** `egm-structural`, `egm-style`
* **Affected Scope:** `Lines 210-213`
* **Discrepancy Description**: The paragraph that lists later-slice scenarios sits under "### CUJ 6". "S-2, S-3 and S-8 to S-9 (Slices 3 and 4)" does not say which scenario belongs to which slice, and "S-8 to S-9" names only two items.
* **Rationale & Recommendation**: Move the paragraph above CUJ 1 or into its own H3. Give the slice for each scenario, and write "S-8 and S-9".

### Failure column format varies in the Slice 1 endpoint table - Severity: Nit
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Lines 662-674`
* **Discrepancy Description**: Some rows give a status and a code, such as `409 ENTRY_CHANGED`, while others give only a status, such as `404` or `428`. The PUT row also omits `TASK_INACTIVE`, which POST lists for the same body.
* **Rationale & Recommendation**: Give a status and a code for every failure, and cite the new error-code table.

### Design goals have no IDs - Severity: Nit
* **Reviewer Lens:** `egm-structural`
* **Affected Scope:** `Lines 76-100`
* **Discrepancy Description**: G-1 to G-4 have IDs, but the four design goals do not, so they cannot be cited.
* **Rationale & Recommendation**: Give them IDs, for example DG-1 to DG-4.

### "Every route" overstates the security checks - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 1001-1002`, compared with `Lines 277-278`
* **Discrepancy Description**: Sign-in and health are exempt from authentication, and CSRF checks apply only to writes.
* **Rationale & Recommendation**: Write "on every route except sign-in and health; CSRF on writes".

### SQLite "cannot test triggers" is inaccurate - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Line 497`, `Lines 982-984`
* **Discrepancy Description**: SQLite has triggers. What it lacks are PostgreSQL row locks, exclusion constraints and PL/pgSQL.
* **Rationale & Recommendation**: Name those features instead.

### "Cannot drift apart unnoticed" leaves out the check that makes it true - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 326-328`
* **Discrepancy Description**: Generated types prevent drift only if they are regenerated and compared. The check that does this (`make lint`) is described only at line 1549, outside the range.
* **Rationale & Recommendation**: Name the check in this sentence.

### "Cannot be met with any project selector" is too absolute - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 146-147`
* **Discrepancy Description**: A selector that defaults to the last-used project could meet the target.
* **Rationale & Recommendation**: Write "with a type-to-search selector".

### Behaviour rules are filed under "Cost:" in Q1 - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 372-375`
* **Discrepancy Description**: The HOL-6 and HOL-7 sentences are behaviour rules, not costs.
* **Rationale & Recommendation**: Move them to the `employee_holidays` bullet (lines 360-363).

### The same statements are repeated across sections - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 133-135, 254-255, 975-978`; `Lines 24-26, 117-119, 1041-1043`
* **Discrepancy Description**: Two statements are repeated:
  * that Docker is never invoked, three times
  * that Slices 2 to 5 get their own implementation plans, three times
* **Rationale & Recommendation**: State each once and refer to it elsewhere.

### Filler and redundant phrases - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 321-322, 390, 436, 955`
* **Discrepancy Description**: These phrases add nothing:
  * "The browser never adds up totals on its own" repeats the sentence before it.
  * "very next request"
  * "not now"
  * "exactly the"
* **Rationale & Recommendation**: Remove them.

### Loose or subjective wording in Alternatives - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 883, 888-889, 938-939, 952, 960, 987-988, 991-992`
* **Discrepancy Description**: These phrases do not state a concrete property:
  * "conflicts that aren't real"
  * "easy to get wrong without anyone noticing"
  * "cannot be misread"
  * "easy to unit-test"
  * "Simple and visible"
  * "the only limit": CPU and the event loop also limit throughput
  * "leaves room"
  * an OWASP recommendation with no citation
* **Rationale & Recommendation**: Replace each with a concrete property or remove it. For example, "conflicts that aren't real" are serialization failures between transactions that did not change the same data.

### Q4 capacity sizing uses 5 years and "mostly fits" - Severity: Nit
* **Reviewer Lens:** `egm-design`, `egm-style`
* **Affected Scope:** `Lines 433-436`
* **Discrepancy Description**: The sizing uses 20 million rows (5 years, NFR-PERF-8), but NFR-DATA-4 requires 7 years. That is about 28 million rows, plus `day_totals` and `audit_log`. "Mostly fits in 16 GB" is a hedge, because the real question is how much PostgreSQL keeps cached.
* **Rationale & Recommendation**: State the 7-year size and the assumed cache size, and remove "mostly".

### Project availability wording is inconsistent and the S-5 message contradicts PRJ-4 - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Line 51`, `Line 808`
* **Discrepancy Description**: The document uses "opened to them", "available" and "assigned" for the same idea. The message "You can only pick projects HR has assigned to you" ignores common projects, which need no assignment.
* **Rationale & Recommendation**: Use one term throughout, and reword the message so it covers common projects.

### "Brainstorming" is process jargon - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Line 56`, `Line 965`
* **Discrepancy Description**: A reader new to the project will not know what this step refers to.
* **Rationale & Recommendation**: Replace it with a plain reference to the decision.

### Q5 does not cite NFR-PERF-4, and "already fixes their architecture" overstates the plan - Severity: Nit
* **Reviewer Lens:** `egm-style`
* **Affected Scope:** `Lines 445-447`, `Line 1043`
* **Discrepancy Description**: NFR-PERF-4 holds the 30-second, 100,000-row target, but Q5 does not cite it. Line 1043 claims the Technical Plan fixes the architecture of Slices 2 to 5, but their contracts are "to be detailed" (line 717) and the department question is still open.
* **Rationale & Recommendation**: Cite NFR-PERF-4 in Q5 and soften line 1043.

### Edits are blocked on entries whose project is no longer available - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 712-715`, `Line 671`
* **Discrepancy Description**: PUT re-runs the project check, so an entry whose project has closed (PRJ-10, S-11) cannot have its activity or duration corrected. DELETE has no such check.
* **Rationale & Recommendation**: State whether this is intended. If not, check availability only when `projectId` or `entryDate` changes.

### A holiday added on a day that already has leave is undefined - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 63-64`
* **Discrepancy Description**: Leave cannot be recorded on an existing holiday, but the reverse is not covered. If HR adds a holiday after leave was recorded, the day counts both, which inflates completeness and RPT-1.
* **Rationale & Recommendation**: Define the behaviour, for example flag the day or exclude the holiday hours.

### Week read totals can disagree with its entries - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Line 758`, `Lines 319-322`
* **Discrepancy Description**: Entries, `day_totals` and holidays are read in separate queries. A save from another tab in between can make the totals disagree with the entries shown.
* **Rationale & Recommendation**: Read the week in one `REPEATABLE READ` read-only transaction, or sum the returned entries in the service.

### CSV export edge cases - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 438-447`
* **Discrepancy Description**: Three details are missing:
  * Tab and carriage return are not prefixed, although they are formula triggers.
  * With a cursor, `statement_timeout` applies to each FETCH, so it does not limit the whole export to 30 seconds.
  * An export that fails partway returns 200 with a truncated file.
* **Rationale & Recommendation**: Add the two characters to the prefix rule. Enforce an overall deadline in the service. Say how a failure partway through is signalled.

### Keyboard path when there is no saved location - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 339-345`
* **Discrepancy Description**: A new employee has `lastWorkLocation = null`, so the 4-interaction path needs one more choice.
* **Rationale & Recommendation**: Define a default location, or count the extra step.

### Security headers and `/metrics` exposure - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 1001-1009`, `Line 725`
* **Discrepancy Description**: NFR-SEC-5 is addressed without a Content Security Policy, `frame-ancestors` or `X-Content-Type-Options`. `/api/v1/metrics` would be public through Caddy's `/api/*` route.
* **Rationale & Recommendation**: Set the headers in Caddy, and restrict `/metrics` to an internal network or require authentication.

### `audit_log` insert grant and cleanup lock type - Severity: Nit
* **Reviewer Lens:** `egm-design`
* **Affected Scope:** `Lines 475-476`, `Lines 637-640`, `Lines 392-394`
* **Discrepancy Description**: `kairos_app` can insert fake audit rows directly. The design does not say whether the cleanup advisory lock is session-level or transaction-level. A session-level lock taken on a pooled connection can stay held after the connection returns to the pool.
* **Rationale & Recommendation**: Make `audit_row()` `SECURITY DEFINER`, owned by `kairos_owner`, and revoke INSERT from `kairos_app`. Use `pg_try_advisory_xact_lock` for the cleanup.
