# Multi-Stage Consulting Session Lifecycle & Resource Tracking

Prepared for: Pivot Strategy Group  
Document type: n8n implementation guide / delivery-ready build plan  
Purpose: Manual build guidance for a production-quality consulting session lifecycle workflow inside n8n

## 1. Executive Summary

This automation is designed to standardize how Pivot Strategy Group handles billable consulting sessions from calendar monitoring through post-session follow-up. The workflow watches Google Calendar for client strategy meetings, filters out non-billable or internal activity, checks the client's remaining retainer balance in Airtable, alerts Slack based on credit level, creates a session notes record, waits until two hours after the session ends, and then sends a professional Gmail follow-up.

Operationally, this matters because it reduces missed billing signals, prevents overlooked low-balance accounts, creates a consistent audit trail for each session, and ensures client follow-up happens on time without relying on manual reminders. It also adds a safe exception path for missing client records so operations teams are informed instead of discovering problems later.

Main systems involved:

| System | Role in Solution | Key Data Used |
|------|------|------|
| n8n | Orchestration layer | Triggering, branching, waiting, routing, logging |
| Google Calendar | Source of session events | Event title, start time, end time, event URL |
| Airtable | Client retainer and session tracking | Remaining hours, contact email, notes log |
| Slack | Internal operational notifications | Balance alerts, unresolved client alerts |
| Gmail | Client-facing follow-up delivery | Session follow-up email |

Core working assumptions for this design:

| Assumption | Reason |
|------|------|
| Strategy sessions are consistently named using the format `[STRAT] Client Name` | Makes filtering and name extraction reliable |
| Airtable has one active retainer record per client | Prevents ambiguous credit lookups |
| Gmail follow-up is sent to the primary client email stored in Airtable | Avoids sending to the wrong attendee |
| Session duration is derived from the Google Calendar start and end time | Keeps logging consistent with the scheduled session |

## 2. Full Architecture Overview

### System Relationship Overview

| Step | Source System | Action | Destination System |
|------|------|------|------|
| 1 | Google Calendar | Detect strategy session event | n8n |
| 2 | n8n | Validate title format and ignore non-strategy events | n8n |
| 3 | n8n | Extract client name and normalize match key | n8n |
| 4 | n8n | Search retainer record | Airtable |
| 5 | n8n | Evaluate balance threshold | Slack |
| 6 | n8n | Create notes entry | Airtable |
| 7 | n8n | Pause until end time plus two hours | n8n |
| 8 | n8n | Send follow-up email | Gmail |
| 9 | n8n | Mark follow-up as sent or escalate unresolved path | Airtable / Slack |

### End-to-End Data Flow

- Google Calendar Trigger fires when a client strategy session begins.
- The workflow checks whether the event title starts with `[STRAT]`.
- Non-matching events exit immediately without creating records or sending alerts.
- Matching events continue to a client-name extraction step.
- The extracted client name is normalized for reliable Airtable matching.
- Airtable is queried for the corresponding retainer record.
- If a match is found, remaining hours are evaluated.
- If remaining hours are `>= 2`, Slack receives a standard confirmation message.
- If remaining hours are `< 2`, Slack receives an urgent low-credit alert.
- Airtable receives a new Session Notes row for the session.
- The workflow waits until the event end time plus two hours.
- Gmail sends a follow-up email to the client contact on file.
- Airtable is updated to mark the follow-up as sent.
- If no Airtable match is found, Slack notifies operations and the workflow records the session as unresolved instead of failing hard.

### Recommended Workflow Shape Inside n8n

| Sequence | Recommended Node Name | Purpose |
|------|------|------|
| 1 | Calendar Strategy Trigger | Detect qualifying calendar events |
| 2 | Validate Strategy Prefix | Allow only `[STRAT]` events |
| 3 | Extract Client Name | Remove prefix and trim title |
| 4 | Normalize Client Key | Standardize name for lookup |
| 5 | Lookup Retainer Balance | Search Airtable Retainer Log |
| 6 | Retainer Record Found? | Handle missing client records |
| 7 | Evaluate Credit Threshold | Split healthy vs urgent balance paths |
| 8 | Slack Session Confirmed | Standard internal notification |
| 9 | Slack Low Credit Alert | Urgent internal notification |
| 10 | Create Session Notes | Insert Airtable notes row |
| 11 | Wait for Follow-Up Window | Pause until end time plus two hours |
| 12 | Send Client Follow-Up | Send Gmail message |
| 13 | Mark Follow-Up Sent | Update Airtable status |
| 14 | Notify Ops Unresolved Client | Slack alert for missing match |

## 3. Suggested Time Breakdown

The following estimate is intentionally realistic for consultant-grade work, including setup, validation, debugging, documentation, and handoff preparation.

| Task | Estimated Hours |
|------|-----------------|
| Requirements review and workflow design confirmation | 1.0 |
| Environment and access validation | 1.0 |
| Credential setup and connection testing | 1.0 |
| Airtable schema preparation and sample data staging | 1.5 |
| Trigger workflow build and calendar event testing | 1.5 |
| Session filtering and client name extraction logic | 1.0 |
| Airtable retainer lookup and normalization logic | 1.5 |
| Slack branching and message formatting | 1.0 |
| Session notes creation and field mapping | 1.0 |
| Wait node timing logic and resume validation | 2.0 |
| Gmail follow-up branch and status update | 1.5 |
| Exception handling and unresolved-client path | 1.0 |
| End-to-end testing, debugging, and reruns | 2.0 |
| Documentation, screenshots, and final handoff packaging | 1.0 |
| **Total** | **16.0** |

### Suggested Delivery Timeline

| Phase | Focus | Estimated Hours |
|------|------|------|
| Phase A | Setup, credentials, Airtable readiness | 3.5 |
| Phase B | Core workflow build through Slack and Airtable | 5.0 |
| Phase C | Wait and Gmail completion flow | 3.5 |
| Phase D | Testing, debugging, client-ready documentation | 4.0 |

## 4. Pre-Build Preparation Checklist

Complete the following before opening the n8n canvas for build work.

### Accounts and Platform Access

- [ ] n8n instance access with permission to create credentials and activate workflows
- [ ] Google account access for the target calendar
- [ ] Airtable workspace access with permission to read and create records
- [ ] Slack workspace access with permission to post to the required channel or use an approved Slack app
- [ ] Gmail account or shared mailbox access for sending client follow-up emails

### Credential Requirements

- [ ] Google OAuth credential in n8n with calendar access enabled
- [ ] Gmail credential in n8n for the sending mailbox
- [ ] Airtable credential in n8n using an approved Personal Access Token
- [ ] Slack credential or webhook credential depending on the chosen notification method

### Business Inputs to Confirm

- [ ] Exact Google Calendar to monitor
- [ ] Standard event title naming convention
- [ ] Confirmed threshold rule for low balance alert: `< 2 hours`
- [ ] Slack channel for normal notifications
- [ ] Slack channel for unresolved or operational issues
- [ ] Sender name and sender mailbox for Gmail
- [ ] Approved email wording for follow-up
- [ ] Whether unresolved client sessions should suppress automatic email send

### Airtable Readiness

- [ ] `Retainer Log` table created
- [ ] `Session Notes` table created
- [ ] Client names standardized
- [ ] Remaining hours populated for active clients
- [ ] Primary email field populated for follow-up recipients

### Recommended Operational Inputs

| Item | Example |
|------|------|
| Strategy notification channel | `#ops-consulting-sessions` |
| Low balance alert channel | `#ops-retainer-alerts` |
| Unresolved client alert channel | `#automation-exceptions` |
| Gmail sender | `strategy@pivotstrategygroup.com` |
| Calendar name | `Client Strategy Sessions` |

## 5. Airtable Database Design

The requested tables are sufficient for the core workflow. A few optional helper fields are also recommended to improve reliability and reporting.

### A. Retainer Log

#### Required Fields

| Field Name | Type | Purpose | Example |
|------|------|------|------|
| Client Name | Single line text | Canonical client name used for matching | `Acme Corp` |
| Remaining Hours | Number | Current hours left on retainer | `5.0` |
| Contract Status | Single select | Indicates whether the client should continue through the workflow | `Active` |
| Email | Email | Primary recipient for follow-up | `ops@acmecorp.com` |
| Slack Contact | Single line text | Internal contact or handle for alerts | `@acme-owner` |
| Notes | Long text | Operational notes about the account | `Monthly strategy retainer` |

#### Strongly Recommended Helper Fields

| Field Name | Type | Purpose |
|------|------|------|
| Normalized Client Key | Formula or text | Lowercase trimmed version of client name for reliable matching |
| Last Session Date | Date | Helpful for reporting and troubleshooting |
| Low Balance Flag | Formula or checkbox | Makes low-balance reporting easier |

#### Sample Records

| Client Name | Remaining Hours | Contract Status | Email | Slack Contact | Notes |
|------|------|------|------|------|------|
| Acme Corp | 5.0 | Active | ops@acmecorp.com | @acme-owner | Healthy balance |
| BrightPath LLC | 1.0 | Active | team@brightpath.com | @brightpath-lead | Low balance testing case |
| Northwind Labs | 0.0 | Paused | hello@northwindlabs.com | @northwind-pm | Awaiting renewal |

### B. Session Notes

#### Required Fields

| Field Name | Type | Purpose | Example |
|------|------|------|------|
| Client Name | Single line text | Client tied to the session | `Acme Corp` |
| Session Date | Date with time | Scheduled session start timestamp | `2026-04-22 14:00` |
| Duration | Number | Session duration stored as hours or minutes | `1.0` |
| Calendar Event URL | URL | Direct link back to the source event | `https://calendar.google.com/...` |
| Follow-up Sent | Checkbox or single select | Indicates whether Gmail follow-up completed | `No` |
| Notes | Long text | Audit details and resolution notes | `Auto-created from n8n at session start` |

#### Strongly Recommended Helper Fields

| Field Name | Type | Purpose |
|------|------|------|
| Workflow Status | Single select | `Created`, `Waiting`, `Follow-up Sent`, `Unresolved` |
| Match Status | Single select | `Matched`, `No Client Match` |
| Follow-up Sent At | Date with time | Useful for proof and troubleshooting |

#### Sample Records

| Client Name | Session Date | Duration | Calendar Event URL | Follow-up Sent | Notes |
|------|------|------|------|------|------|
| Acme Corp | 2026-04-22 14:00 | 1.0 | `https://calendar.google.com/...` | No | Auto-created at session start |
| BrightPath LLC | 2026-04-22 16:00 | 1.5 | `https://calendar.google.com/...` | No | Low balance alert issued to Slack |
| Unknown Client | 2026-04-23 10:00 | 1.0 | `https://calendar.google.com/...` | No | No Airtable retainer match found |

### Naming and Matching Guidance

- Keep `Client Name` identical between the calendar title and Airtable whenever possible.
- Avoid punctuation variants such as `LLC` in one system and `L.L.C.` in another.
- Avoid extra title suffixes unless a standard delimiter is adopted.
- If the business wants titles such as `[STRAT] Acme Corp | QBR`, define a single delimiter rule and parse only the portion before the delimiter.

## 6. Step-by-Step n8n Build Plan

This section is the main execution plan for the manual build.

### Phase 1 - Trigger Layer

**Objective:** Detect relevant strategy sessions from Google Calendar.

**Recommended node:** `Google Calendar Trigger`

**Recommended event choice:** `Event Started`

**Why this is the best default:** Triggering at session start reduces false positives from events that were created but later moved, canceled, or never held. It also aligns the logging event with the beginning of a billable session.

**Configuration steps:**

1. Create a new workflow in n8n and save it with a clear name such as `PSG - Strategy Session Lifecycle`.
2. Add a Google Calendar Trigger node.
3. Connect the approved Google credential.
4. Select the specific business calendar to monitor rather than a personal calendar whenever possible.
5. Set the trigger event to `Event Started`.
6. Test the trigger using a short sample event on the target calendar.
7. Confirm the trigger output includes the event summary, start time, end time, and event link fields needed downstream.

**Events to monitor:**

- Client-facing strategy sessions only
- Sessions whose title format begins with `[STRAT]`
- Events on the approved consulting calendar

**Polling recommendation:**

- Use the node's default trigger behavior first.
- If the environment exposes a configurable polling interval, use a moderate interval during build and test work.
- For production, aim for a balance between responsiveness and API efficiency. A practical benchmark is every 1 to 5 minutes for a low-to-moderate event volume environment.

**Implementation note:** If the business later wants pre-session alerts instead of at-session-start alerts, build a second workflow using a created or updated event pattern with duplicate protection.

### Phase 2 - Session Filtering

**Objective:** Allow only strategy sessions to continue.

**Recommended node:** `IF` or `Filter`

**Primary rule:** Event title starts with `[STRAT]`

**Recommended logic:**

- Condition 1: Summary exists
- Condition 2: Summary starts with `[STRAT]`

**Optional defensive exclusions:**

- Exclude titles containing `[INTERNAL]`
- Exclude titles containing `PTO`
- Exclude titles containing `Admin`
- Exclude titles containing `1:1`

**Pass examples:**

- `[STRAT] Acme Corp`
- `[STRAT] BrightPath LLC`

**Fail examples:**

- `Internal Ops Sync`
- `[INTERNAL] Team Planning`
- `Dentist Appointment`
- `Acme Corp Strategy Session`

**Best practice:** Use inclusion logic, not a long list of exclusions. The `[STRAT]` prefix should be the authoritative gate.

### Phase 3 - Client Name Extraction

**Objective:** Convert the event title into a clean Airtable lookup key.

**Recommended node:** `Set`, `Edit Fields`, or a lightweight transformation node

**Target output fields:**

| Output Field | Purpose |
|------|------|
| `client_name_raw` | Title after removing `[STRAT]` |
| `client_name_clean` | Trimmed display value used for notes and email |
| `client_key_normalized` | Lowercase trimmed comparison key |

**Extraction method:**

1. Remove the literal `[STRAT]` prefix.
2. Trim leading and trailing whitespace.
3. If a delimiter is used, keep only the client portion before the delimiter.
4. Create a normalized version for matching.

**Examples:**

| Event Title | Extracted Client Name |
|------|------|
| `[STRAT] Acme Corp` | `Acme Corp` |
| `[STRAT] BrightPath LLC` | `BrightPath LLC` |
| `[STRAT]   Acme Corp   ` | `Acme Corp` |
| `[STRAT] Acme Corp | Q2 Planning` | `Acme Corp` |

**Normalization guidance:**

- Convert to lowercase for matching
- Trim extra spaces
- Collapse repeated internal spaces if possible
- Avoid fuzzy matching unless absolutely necessary

**Consulting recommendation:** The easiest long-term solution is to enforce strict calendar naming rather than relying on increasingly complex parsing logic.

### Phase 4 - Airtable Credit Lookup

**Objective:** Find the client's retainer record and retrieve remaining hours.

**Recommended node:** `Airtable`

**Target table:** `Retainer Log`

**Lookup strategy:**

- Preferred approach: match against a helper field such as `Normalized Client Key`
- Acceptable fallback: match against `Client Name` if naming is perfectly standardized

**Configuration steps:**

1. Add the Airtable node after client normalization.
2. Connect the Airtable credential.
3. Select the correct base and the `Retainer Log` table.
4. Configure the operation to search or list records with a filter condition.
5. Return the first exact active match only.
6. Validate that the output includes `Remaining Hours`, `Contract Status`, `Email`, and `Slack Contact`.

**How to avoid spacing and case issues:**

- Normalize the event-derived key in n8n.
- Store a normalized helper field in Airtable.
- Match normalized-to-normalized.
- Do not rely on partial text matching for billing logic.

**Additional business rule to apply:**

- If `Contract Status` is not `Active`, treat the record as an exception even if a name match exists.

**Recommended outcome fields to carry forward:**

| Output Field | Source |
|------|------|
| `client_name` | Airtable `Client Name` |
| `remaining_hours` | Airtable `Remaining Hours` |
| `contract_status` | Airtable `Contract Status` |
| `client_email` | Airtable `Email` |
| `slack_contact` | Airtable `Slack Contact` |

### Phase 5 - Credit Decision Branching

**Objective:** Route notifications based on retainer balance level.

**Recommended node:** `IF` or `Switch`

**Exact branching logic requested:**

| Condition | Outcome |
|------|------|
| `remaining_hours >= 2` | Send standard session confirmed Slack message |
| `remaining_hours < 2` | Send urgent low-credit Slack alert |

**Operational enhancement strongly recommended:**

| Condition | Suggested Handling |
|------|------|
| `remaining_hours = 0` or negative | Use the urgent branch and include stronger wording |
| `contract_status != Active` | Route to operations exception path |

**Slack message guidance for healthy balance:**

- Keep it informational
- Include client name
- Include session start time
- Include current remaining hours
- Include session notes record intent

**Slack message guidance for low balance:**

- Mark it as urgent
- Include client name
- Include session date/time
- Include remaining hours
- Mention that the session is proceeding but the account needs review

**Suggested Slack message structure:**

| Message Type | Example Content |
|------|------|
| Standard | `Strategy session started for Acme Corp. Remaining retainer balance: 5.0 hours.` |
| Urgent | `Urgent: BrightPath LLC strategy session started with only 1.0 retainer hour remaining. Please review coverage.` |

### Phase 6 - Session Notes Record

**Objective:** Create a durable session log in Airtable before the workflow enters the wait state.

**Recommended node:** `Airtable`

**Target table:** `Session Notes`

**Fields to map:**

| Session Notes Field | Mapping Guidance |
|------|------|
| Client Name | Use the matched Airtable client name when available, otherwise the parsed calendar name |
| Session Date | Use the calendar event start time |
| Duration | Calculate from end time minus start time |
| Calendar Event URL | Use the event URL from the trigger payload |
| Follow-up Sent | Default to `No` |
| Notes | Record whether the client was matched, balance level, and workflow entry timestamp |

**Recommended notes format:**

- `Auto-created by n8n at session start`
- `Retainer match: Yes/No`
- `Remaining hours at session start: X`
- `Alert level: Standard/Urgent/Unresolved`

**Important build choice:** Create the session notes row before the Wait node. This ensures there is already a record for operational review even if the workflow later fails during the follow-up stage.

### Phase 7 - Post Session Wait Logic

**Objective:** Pause the workflow until two hours after the session ends.

**Recommended node:** `Wait`

**Wait target:** `Event End Time + 2 Hours`

**Configuration steps:**

1. Add a Wait node after the Session Notes creation step.
2. Choose a date/time-based wait mode rather than a fixed duration.
3. Build the wait timestamp from the calendar event end time plus two hours.
4. Test with a short mock event so the wait behavior can be validated quickly.
5. Confirm the workflow resumes successfully and continues to the Gmail step.

**Why this design is preferable:**

- The follow-up timing stays tied to the actual meeting schedule.
- Rescheduled events still use the event's real end time.
- The follow-up delay is transparent and auditable.

**Critical timing note:** n8n's Wait behavior should be validated against the server timezone and the workflow's expected operating timezone. Use test events with known start and end timestamps and confirm the resume time exactly.

**Practical QA test:**

- Create a 15-minute test event
- Trigger the workflow at start
- Verify the Wait resumes 2 hours after the event end, not 2 hours after workflow start

### Phase 8 - Gmail Follow-Up

**Objective:** Send a professional post-session follow-up email after the waiting period.

**Recommended node:** `Gmail`

**Pre-send conditions:**

- A valid Airtable client match exists
- `client_email` is populated
- Contract status is acceptable for outbound communication

**Recommended subject line:**

`Thank you for today's strategy session - {{Client Name}}`

**Professional email template:**

```text
Hi {{Client Name}},

Thank you for today's strategy session with Pivot Strategy Group.

This is a quick follow-up to keep momentum moving. Please use the link below to share any additional feedback, priorities, or follow-up items that should be addressed after the session:

{{Feedback Form Link}}

If helpful, we can also consolidate next steps and outstanding action items into the next engagement checkpoint.

Best,
Pivot Strategy Group
```

**Dynamic merge tags to map:**

| Placeholder | Data Source |
|------|------|
| `{{Client Name}}` | Airtable client name |
| `{{Feedback Form Link}}` | Static company-approved URL or configurable field |
| Subject date reference if desired | Calendar session date |

**Post-send step:**

- Update the Session Notes record to mark `Follow-up Sent` as `Yes`
- If helper fields exist, update `Workflow Status` to `Follow-up Sent`
- If helper fields exist, stamp `Follow-up Sent At`

**Recommended safeguard:** If the client match is unresolved, do not send the email automatically. Notify operations instead and require manual review.

### Phase 9 - Error Handling

**Objective:** Prevent silent failures and preserve operational visibility.

#### A. No Airtable Match Found

**Required behavior:**

- Send a Slack message to the operations or exceptions channel
- Flag the session as unresolved
- Do not let the workflow crash without notice

**Recommended handling flow:**

1. Branch immediately after the Airtable lookup.
2. If no record is returned, send a Slack alert containing:
   - Event title
   - Parsed client name
   - Session date/time
   - Calendar event URL
3. Create a Session Notes record anyway using the parsed client name.
4. Mark the notes record as unresolved in the `Notes` field or a helper `Match Status` field.
5. Stop automatic Gmail delivery unless the business explicitly approves a fallback email source.

**Suggested Slack alert wording:**

`Unresolved session: no Airtable retainer record found for parsed client "Acme Advisory" from calendar event "[STRAT] Acme Advisory". Manual review required.`

#### B. External API Failures

**Recommended retry strategy:**

- Enable retries on Airtable, Slack, and Gmail nodes where appropriate
- Keep retry counts modest to avoid duplicate client emails or duplicate notifications
- Use idempotent update logic where possible

**Practical failure handling rules:**

| Failure Type | Suggested Action |
|------|------|
| Airtable temporary failure | Retry, then notify ops if still failing |
| Slack send failure | Retry, then continue if record creation is more critical |
| Gmail send failure | Retry, then notify ops and leave `Follow-up Sent` as `No` |
| Wait resume issue | Validate timezone and workflow activation state immediately |

**Recommended operational hardening:**

- Add a separate error-monitoring workflow later using n8n's error handling pattern
- Log enough context in Slack alerts so support staff can act without opening n8n first

## 7. Recommended Node Naming Convention

Use names that read like process steps rather than generic node defaults.

| Node Purpose | Recommended Name |
|------|------|
| Google calendar trigger | `Calendar Strategy Trigger` |
| Strategy prefix gate | `Validate Strategy Prefix` |
| Remove prefix and trim | `Extract Client Name` |
| Normalize comparison key | `Normalize Client Key` |
| Airtable search | `Lookup Retainer Balance` |
| Missing client check | `Retainer Record Found?` |
| Hours threshold branch | `Evaluate Credit Threshold` |
| Standard Slack path | `Slack Session Confirmed` |
| Urgent Slack path | `Slack Low Credit Alert` |
| Session notes insert | `Create Session Notes` |
| Wait for delayed email | `Wait for Follow-Up Window` |
| Gmail send | `Send Client Follow-Up` |
| Airtable update after send | `Mark Follow-Up Sent` |
| Missing client alert | `Notify Ops Unresolved Client` |
| Contract exception branch | `Validate Contract Status` |

Naming standard guidance:

- Start names with the business action, not the software brand
- Keep names readable in execution logs
- Avoid generic names such as `IF1`, `Airtable2`, or `Set3`
- Keep branching node names phrased as questions when they evaluate conditions

## 8. Testing Strategy

Run controlled test cases with sample data before activation.

### Test Case Matrix

| Case | Scenario | Expected Result |
|------|------|------|
| Case A | Client has 5 remaining hours | Strategy Slack confirmation sent, Session Notes created, Wait resumes correctly, Gmail follow-up sent, Airtable updated to `Follow-up Sent = Yes` |
| Case B | Client has 1 remaining hour | Urgent Slack low-credit alert sent, Session Notes created, workflow still continues unless policy blocks it, Gmail follow-up sent after wait |
| Case C | Client not found in Airtable | Operations Slack alert sent, Session Notes created with unresolved note, no automatic Gmail by default |
| Case D | Internal or personal meeting | Workflow stops after filter, no Slack message, no Airtable record, no Gmail |

### Recommended Test Execution Steps

1. Prepare one Airtable record for a healthy-balance client.
2. Prepare one Airtable record for a low-balance client.
3. Leave one test client intentionally missing from Airtable.
4. Create four calendar events covering the four scenarios.
5. Execute tests in a controlled order so logs are easy to review.
6. Confirm each expected branch path in the n8n execution history.
7. Capture screenshots of Slack, Airtable, and Gmail outcomes.

### Additional Edge Cases Worth Testing

| Edge Case | Why It Matters |
|------|------|
| Event title includes extra spaces | Validates normalization logic |
| Event is rescheduled shortly before start | Confirms trigger behavior is acceptable |
| Session has no event URL in the payload | Ensures notes record still gets created |
| Gmail recipient missing | Confirms safe suppression of automatic email |
| Contract status is paused | Confirms exception handling works correctly |

## 9. Debugging Checklist

Use this list during build validation and post-deployment support.

| Issue | Symptom | Likely Cause | Resolution |
|------|------|------|------|
| Calendar timezone mismatch | Wait resumes at the wrong local time | Server timezone and event timezone not aligned | Re-test with known timestamps and confirm timezone handling |
| Airtable field mismatch | Lookup returns no record even though the client exists | Wrong field name, wrong table, or inconsistent client naming | Validate base, table, field names, and normalized key logic |
| Gmail auth expired | Follow-up step fails at send | OAuth token no longer valid | Reconnect Gmail credential and retest |
| Slack message goes to wrong place | Notifications appear in an unexpected channel | Wrong credential, channel mapping, or Slack app scope | Reconfirm channel target and credential setup |
| Wait node resume problems | Workflow stays paused or resumes unexpectedly | Miscalculated timestamp or activation issue | Validate wait expression and confirm workflow remains active |
| Duplicate records created | Multiple Session Notes rows for one event | Event update behavior or repeated testing without cleanup | Add duplicate protection later if necessary |
| Missing event URL | Notes record has blank URL | Trigger payload variation | Allow blank value and keep the run moving |
| Wrong client matched | Slack and email reference the wrong account | Weak or partial name matching | Use exact normalized matching only |

## 10. Final Delivery Checklist

This is the client-facing completion checklist for a consulting engagement.

### Screenshots to Capture

- [ ] Workflow canvas showing the final node structure
- [ ] Credential list with sensitive values hidden
- [ ] Successful execution for a healthy-balance client
- [ ] Successful execution for a low-balance client
- [ ] Unresolved client alert example
- [ ] Airtable `Retainer Log` sample records
- [ ] Airtable `Session Notes` record created by the workflow
- [ ] Slack notification example
- [ ] Gmail follow-up example

### Proof to Provide Client or Internal Stakeholders

- [ ] One end-to-end successful run with a matched client
- [ ] One low-balance run proving the urgent Slack branch
- [ ] One unresolved run proving graceful exception handling
- [ ] Confirmation that non-`[STRAT]` meetings are ignored
- [ ] Confirmation that `Follow-up Sent` is updated after email delivery

### Recommended Handoff Summary

Present the completed solution using this simple structure:

1. Show how the workflow is triggered from Google Calendar.
2. Show how client retainer status is checked in Airtable.
3. Show Slack branching for healthy and low-balance outcomes.
4. Show the Session Notes record being created.
5. Show the delayed Gmail follow-up path and final Airtable update.

### Operational Handoff Notes

- Document the expected naming convention for calendar titles
- Document which Slack channels receive which message type
- Document who owns Airtable data quality
- Document who owns Gmail mailbox access and template approval

## 11. Optimization Ideas

The core workflow above is sufficient for delivery. The following upgrades would make it stronger over time.

| Enhancement | Value |
|------|------|
| Auto deduct used hours from retainer balance | Converts alerting into true retainer consumption management |
| PDF summary generation | Creates a polished session artifact for clients or account teams |
| CRM sync | Keeps client engagement history aligned across systems |
| Invoice trigger | Launches billing workflows when balances fall below a threshold |
| Dashboard reporting | Gives leadership visibility into session volume, follow-up completion, and low-balance accounts |
| Duplicate event protection | Prevents repeated logs if calendar behavior changes |
| Client-specific email templates | Improves personalization and account experience |
| Fallback contact routing | Uses a secondary contact if the primary email is missing |

## Reference Sources

The recommendations in this guide were aligned to the current official product documentation available on April 22, 2026.

- [n8n Google Calendar Trigger documentation](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/)
- [n8n Airtable node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/)
- [n8n Airtable credential documentation](https://docs.n8n.io/integrations/builtin/credentials/airtable/)
- [n8n Gmail node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)
- [n8n Slack node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/)
- [n8n Wait node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/)
- [n8n Google OAuth credential documentation](https://docs.n8n.io/integrations/builtin/credentials/google/)
- [Google Calendar Events resource reference](https://developers.google.com/workspace/calendar/api/v3/reference/events)
