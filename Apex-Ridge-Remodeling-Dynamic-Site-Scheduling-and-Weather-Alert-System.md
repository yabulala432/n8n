# Apex Ridge Remodeling: Dynamic Site Scheduling & Weather Alert System

Prepared for: Apex Ridge Remodeling  
Document type: n8n implementation guide  
Purpose: Step-by-step execution plan for building a Google Sheets to Google Calendar scheduling workflow with weather risk checks for exterior remodeling tasks

## 1. Executive Summary

This automation solves a common coordination problem for remodeling companies: project tasks live in a spreadsheet, field schedules live in a calendar, and weather risk often gets noticed too late for exterior work.

Without automation, office staff or project coordinators must manually review the task list, determine whether work is interior or exterior, check the weather, create calendar events, and then remember to update those events whenever dates change. That process is slow, repetitive, and vulnerable to missed updates and duplicate entries.

For contractors, this workflow creates one reliable operating pattern:

1. Project tasks are entered or updated in Google Sheets.
2. n8n reviews each row.
3. Interior tasks move directly into Google Calendar.
4. Exterior tasks are evaluated against the upcoming forecast before calendar scheduling is finalized.
5. Risky weather conditions are clearly flagged so the office and field teams can react early.

Operationally, this reduces preventable site disruption, gives crews better schedule visibility, improves confidence in exterior job planning, and creates a cleaner audit trail of what was scheduled, when it synced, and whether weather risk was present at the time of scheduling.

---

## 2. Full Architecture Overview

## Core Systems

| System | Role in the solution |
|---|---|
| `n8n` | Orchestration layer that reads sheet rows, validates data, checks weather, creates or updates calendar events, writes results back, and triggers failure alerts |
| `Google Sheets` | Source-of-truth task list for project scheduling inputs |
| `Google Calendar` | Crew-facing or operations-facing scheduling calendar where task events are created and maintained |
| `Weather API` | Forecast source for exterior job risk detection using 5-day weather data |
| `Slack` or `Gmail` | Admin notification channel for errors, validation failures, and external service issues |

## Recommended Workflow Set

Build two workflows:

1. `Apex Ridge - Project Task Sync`
2. `Apex Ridge - Scheduling Error Handler`

The main workflow handles task synchronization and weather logic.  
The error workflow catches unhandled failures and notifies an admin.

## End-to-End Data Flow

`Google Sheets Row Added or Updated`  
`-> n8n Trigger or Scheduled Poll`  
`-> Normalize Row Data`  
`-> Validate Required Fields`  
`-> Determine Task Type`

`Interior` branch:  
`-> Build Calendar Payload`  
`-> Create or Update Calendar Event`  
`-> Write Event ID / Sync Timestamp Back to Sheet`

`Exterior` branch:  
`-> Resolve Weather Lookup Input`  
`-> Request 5-Day Forecast`  
`-> Match Forecast to Task Date`  
`-> Detect Weather Risk`  
`-> Build Calendar Payload with Risk Status`  
`-> Create or Update Calendar Event`  
`-> Write Event ID / Sync Timestamp / Risk Status Back to Sheet`

Failure path:  
`Validation failure or API failure`  
`-> Slack or Gmail Admin Alert`  
`-> Optional row status update in Google Sheets`

## Logical Architecture Diagram

- Source layer
  - Google Sheets stores planned remodeling tasks.
- Decision layer
  - n8n validates data and classifies the task as `Interior` or `Exterior`.
- Enrichment layer
  - Exterior tasks call the weather provider and derive a risk decision for the scheduled day.
- Action layer
  - n8n creates or updates a Google Calendar event.
- Audit layer
  - n8n writes back the calendar event ID, sync timestamp, and risk outcome.
- Exception layer
  - n8n sends an admin alert when validation, weather, or calendar operations fail.

## Recommended Business Rule Summary

| Rule | Recommendation |
|---|---|
| Source of truth | Google Sheet remains the authoritative task list |
| Duplicate prevention | Use `Calendar Event ID` stored in the sheet as the primary idempotency key |
| Weather evaluation | Only run for `Exterior` tasks |
| Forecast horizon | Only process weather risk when task start date falls within the provider's available 5-day forecast window |
| Timezone | Use one business timezone consistently across Google Sheets, n8n, and Google Calendar |
| Failure visibility | Always route service failures to Slack or Gmail |

---

## 3. Suggested Time Breakdown

Use the following estimates for a realistic tracked-hours implementation plan.

| Task | Hours |
|---|---:|
| Project discovery and workflow planning | 1.0 |
| Credential setup in n8n | 1.0 |
| Google Sheet schema design and test data preparation | 0.75 |
| Google Calendar structure and naming conventions | 0.5 |
| Main workflow scaffold and node layout | 1.0 |
| Trigger or scheduled polling implementation | 1.5 |
| Data validation and normalization logic | 1.25 |
| Task classification branch design | 0.75 |
| Weather API integration and forecast parsing | 2.25 |
| Weather risk rule implementation | 1.25 |
| Calendar create/update and duplicate prevention logic | 2.5 |
| Sheet write-back fields and sync audit logic | 1.0 |
| Admin alerts and failure handling | 1.25 |
| Testing, debugging, and edge-case review | 2.0 |
| Documentation, screenshots, and handoff notes | 1.0 |
| **Total Estimated Effort** | **18.0** |

## Recommended Delivery Phases

| Phase | Focus | Hours |
|---|---|---:|
| Phase 0 | Preparation and access | 2.25 |
| Phase 1 | Workflow scaffold and trigger | 2.5 |
| Phase 2 | Validation and branching | 2.0 |
| Phase 3 | Weather intelligence | 3.5 |
| Phase 4 | Calendar upsert and write-back | 3.5 |
| Phase 5 | Error handling and notifications | 1.25 |
| Phase 6 | QA, debugging, and proof of work capture | 3.0 |
| **Total** |  | **18.0** |

---

## 4. Pre-Build Preparation Checklist

Complete this checklist before opening the n8n canvas.

## Access Checklist

- [ ] n8n access with permission to create workflows and credentials
- [ ] Google account access for the scheduling spreadsheet
- [ ] Google account access for the destination calendar
- [ ] Permission to edit or create calendar events
- [ ] Permission to update the source Google Sheet
- [ ] Slack workspace access and incoming webhook details, or Gmail account access for alerts

## Technical Inputs Checklist

- [ ] Weather API key created and tested
- [ ] Confirm whether the weather provider accepts ZIP directly or requires geocoding first
- [ ] Confirm target company timezone for schedule events
- [ ] Confirm which Google Calendar should receive events
- [ ] Confirm expected event color or visual convention for weather-risk events
- [ ] Confirm the polling frequency if using scheduled sync
- [ ] Confirm who receives admin failure alerts

## Data Preparation Checklist

- [ ] Google Sheet created with final column headers
- [ ] At least three test rows prepared
- [ ] Task types standardized to controlled values such as `Interior` and `Exterior`
- [ ] Start dates entered in a consistent date format
- [ ] ZIP codes validated for likely service area correctness
- [ ] Status values agreed for unsynced, synced, validation error, and weather-risk states

## Operational Decisions to Confirm

- [ ] What should happen when the task date is outside the 5-day weather forecast window
- [ ] Whether risky weather should only flag the event or also notify an admin
- [ ] Whether cancelled tasks should delete calendar events or only mark them inactive
- [ ] Whether row edits should always overwrite calendar details
- [ ] Whether the same sheet will include completed and future tasks

---

## 5. Google Sheet Design

Create one source sheet that acts as the scheduling control table.

Recommended spreadsheet name: `Apex Ridge - Project Schedule`  
Recommended tab name: `Project Tasks`

## Required Columns

| Column | Required | Purpose |
|---|---|---|
| `Project ID` | Yes | Stable identifier for the job or project |
| `Project Name` | Yes | Human-readable project label used in calendar titles |
| `Task Name` | Yes | Specific work item, such as roofing, framing, siding, or demo |
| `Task Type` | Yes | Classification used for routing; recommended values: `Interior`, `Exterior` |
| `Start Date` | Yes | Event start date used for weather matching and calendar scheduling |
| `Site Zip Code` | Yes for exterior | Location input for forecast lookup |
| `Duration Days` | Yes | Determines event end date |
| `Calendar Event ID` | Yes for sync logic | Stores the Google Calendar event ID so future edits update the same event |
| `Last Synced At` | No | Timestamp showing the last successful n8n sync |
| `Status` | Yes | Operational state such as `Pending`, `Synced`, `Validation Error`, `Weather Risk` |

## Why `Calendar Event ID` Matters

This column is the most important duplicate-prevention mechanism in the design.

Without `Calendar Event ID`, each row edit can look like a new task to the automation, which creates a risk of duplicate calendar events. By saving the event ID after the first successful calendar creation, n8n can determine whether the next run should:

- create a new event, or
- update the existing event tied to that row

This is the cleanest and most reliable upsert pattern for Google Calendar in an n8n workflow driven by spreadsheet records.

## Recommended Optional Enhancements

If the team is open to two extra columns, consider adding:

| Column | Why it helps |
|---|---|
| `Risk Status` | Separates operational sync state from weather outcome |
| `Last Error` | Makes row-level troubleshooting easier without opening execution logs |

These are optional. The required build can still function without them.

## Sample Rows

| Project ID | Project Name | Task Name | Task Type | Start Date | Site Zip Code | Duration Days | Calendar Event ID | Last Synced At | Status |
|---|---|---|---|---|---|---:|---|---|---|
| `AR-1001` | `Mason Residence` | `Interior Framing` | `Interior` | `2026-05-04` | `60614` | 2 |  |  | `Pending` |
| `AR-1002` | `Lakeview Addition` | `Roofing` | `Exterior` | `2026-05-06` | `60031` | 1 |  |  | `Pending` |
| `AR-1003` | `Hawthorne Renovation` | `Foundation Pour` | `Exterior` | `2026-05-07` | `60120` | 1 | `6k8m2exampleeventid` | `2026-05-01 09:12:22` | `Synced` |

---

## 6. Step-by-Step n8n Build Plan

This section is written as a practical execution sequence so the implementer can build the workflow manually in n8n.

## Recommended Build Strategy

For production reliability, use a scheduled polling workflow instead of relying only on a direct Google Sheets trigger. Polling is easier to audit, more predictable for row edits, and better for controlled re-runs when something fails mid-sync.

If near real-time response is essential, you can still evaluate the Google Sheets Trigger. The guide below explains both options.

## Workflow Names

- Main workflow: `Apex Ridge - Project Task Sync`
- Error workflow: `Apex Ridge - Scheduling Error Handler`

## Phase 1 - Trigger Layer

### Option A: Google Sheets Trigger

Use this when:

- the spreadsheet volume is low
- the n8n environment supports the trigger reliably
- you want faster event processing after edits

Pros:

- Near real-time behavior
- Less manual filtering logic
- Faster feedback during testing

Cons:

- Can be harder to audit for missed edits
- Trigger behavior may vary depending on how rows are edited or inserted
- Less control over bulk recovery if a prior execution fails

### Option B: Scheduled Polling Workflow

Use this when:

- reliability matters more than near real-time execution
- the office can tolerate a short sync delay such as 5, 10, or 15 minutes
- you want a clearer operational pattern for tracked production work

Pros:

- Easier to reason about in production
- Better for retrying and controlled reprocessing
- Stronger fit for row-update workflows

Cons:

- Not instant
- Requires filtering logic to determine which rows need action

### Recommended Production Choice

Use a `Schedule Trigger` every 15 minutes, then query the sheet for rows where:

- `Status` is not `Synced`, or
- `Last Synced At` is blank, or
- the row was edited since the prior sync checkpoint

If the spreadsheet does not track last-edited timestamps, use a practical operational rule:

- Sync all rows with `Status` in `Pending`, `Ready`, `Needs Update`, or `Weather Risk`

### Node 1: Schedule Trigger

Node name: `Schedule Trigger - Task Sync`

Purpose:

- Starts the sync on a fixed interval

Recommended settings:

| Setting | Value |
|---|---|
| Trigger type | `Every X` |
| Interval | `15 minutes` |

### Node 2: Google Sheets - Read Task Rows

Node name: `Read Project Tasks`

Purpose:

- Pull candidate rows from the `Project Tasks` sheet

Implementation notes:

1. Connect the Google Sheets credential.
2. Select the spreadsheet and tab.
3. Read the rows into n8n.
4. Keep row number metadata if available, because you will need it later for sheet updates.

### Node 3: Filter Sync Candidates

Node name: `Filter Rows Needing Sync`

Purpose:

- Limit downstream processing to rows that should create or update calendar events

Recommended filter rules:

- Include rows where `Status` is `Pending`
- Include rows where `Status` is `Needs Update`
- Include rows where `Calendar Event ID` is populated but the row has been changed
- Exclude rows marked `Completed`, `Cancelled`, or blank

If row-level change tracking is not available, use a conservative rule and reprocess only active rows that are not already finalized.

---

## Phase 2 - Data Validation

The validation phase should run before any weather call or calendar action.

### Required Fields to Validate

- `Project Name`
- `Task Type`
- `Start Date`
- `Site Zip Code` for exterior work

### Node 4: Normalize Row Fields

Node name: `Normalize Task Data`

Purpose:

- Standardize incoming field names and formats before decision logic begins

What to normalize:

- Trim whitespace from text values
- Standardize task type to `Interior` or `Exterior`
- Convert the date into one consistent format
- Convert `Duration Days` to a numeric value
- Default blank duration to `1` if the business approves that rule

### Node 5: Validate Required Fields

Node name: `Validate Required Fields`

Purpose:

- Stop bad records before they create downstream failures

Recommended validation logic:

- `Project Name` must not be empty
- `Task Type` must equal `Interior` or `Exterior`
- `Start Date` must be a valid date
- `Site Zip Code` must be present when `Task Type = Exterior`

### Validation Routing

`Valid row`  
`-> continue to task classification`

`Invalid row`  
`-> update sheet status`  
`-> send admin alert`

### Node 6: Flag Invalid Row

Node name: `Flag Validation Error`

Purpose:

- Update the row status to something operationally obvious such as `Validation Error`

Recommended write-back:

- `Status = Validation Error`
- Optional `Last Synced At = current timestamp`
- Optional `Last Error = Missing required field`

### Node 7: Notify Admin - Invalid Input

Node name: `Notify Admin Validation Failure`

Purpose:

- Inform the admin that a row could not be scheduled

Alert content should include:

- project ID
- project name
- task name
- missing or invalid field
- row number if available

---

## Phase 3 - Task Classification

This is the first major decision point in the workflow.

### Business Rule

- If `Task Type = Interior`, skip weather lookup.
- If `Task Type = Exterior`, continue to weather evaluation.

### Node 8: Check Exterior Task

Node name: `Check Exterior Task`

Purpose:

- Branch the workflow based on whether weather matters for the task

Recommended IF logic:

| Condition | Route |
|---|---|
| `Task Type = Exterior` | Weather branch |
| `Task Type = Interior` | Calendar branch without weather |

### Interior Branch Outcome

Interior tasks should still:

- create or update a calendar event
- write back the event ID
- mark the row as synced

Weather fields for interior tasks can use safe defaults such as:

- `weather_alert = false`
- `forecast_summary = Not applicable - interior task`

---

## Phase 4 - Weather API Integration

Use this branch only for exterior work.

## Recommended API Approach

For a real implementation, use one of these patterns:

1. ZIP code -> geocoding endpoint -> latitude/longitude -> forecast endpoint
2. ZIP code -> direct forecast endpoint if supported by the chosen provider

The most robust approach is geocoding first, because it works consistently across providers and avoids ambiguity.

## Node 9: Prepare Weather Lookup Input

Node name: `Prepare Weather Request`

Purpose:

- Convert the row's ZIP code and date fields into a clean API request context

Recommended derived fields:

- normalized ZIP code
- task start date in business timezone
- forecast target date

### Node 10: HTTP Request - Geocode ZIP

Node name: `Resolve Site Coordinates`

Purpose:

- Convert ZIP code into coordinates if required by the weather service

Request intent:

- send the site ZIP code to the weather provider's geocoding endpoint
- return latitude, longitude, city, and region if available

If your provider supports direct ZIP-based forecast calls, this step can be skipped.

### Node 11: HTTP Request - Fetch Forecast

Node name: `Fetch Weather Forecast`

Purpose:

- retrieve the 5-day forecast for the job site

Expected output:

- forecast entries for the next several days
- temperature data
- condition text or code
- precipitation indicators

### Parsing Strategy

Most 5-day APIs return multiple forecast entries per day rather than a single daily object. Because of that, do not just use the first forecast item returned.

Instead:

1. Convert the task `Start Date` into the same timezone used for the forecast.
2. Filter forecast entries to the matching calendar day.
3. If multiple entries exist for that day, choose one of these rules:
   - use the highest precipitation risk entry
   - use the midday forecast entry
   - aggregate the day and keep the worst weather condition

For construction scheduling, the safest rule is usually:  
`choose the worst forecast entry for the target workday`

### Important Edge Case

If the `Start Date` is outside the forecast horizon:

- do not fail the workflow
- set a non-blocking status such as `Forecast Unavailable`
- create or update the event without a weather alert
- optionally notify the admin only if the business wants review of unsafely future exterior tasks

---

## Phase 5 - Risk Detection Logic

This phase converts weather data into a simple scheduling decision.

### Recommended Risk Criteria

| Condition | Suggested rule |
|---|---|
| Heavy rain | Forecast indicates heavy rain or rainfall above team threshold |
| Snow | Any snow forecast on the workday |
| Storm | Thunderstorm or severe storm condition code/text |
| High precipitation probability | Probability exceeds approved threshold such as `50%` or `60%` |

### Node 12: Detect Weather Risk

Node name: `Detect Weather Risk`

Purpose:

- evaluate forecast output and create a binary flag for downstream event formatting

Recommended outputs:

| Field | Meaning |
|---|---|
| `weather_alert` | `true` or `false` |
| `risk_reason` | Human-readable explanation such as `Heavy rain expected` |
| `forecast_summary` | Short summary of the selected forecast |
| `temperature_summary` | Example: `High 58F / Low 49F` |
| `rain_chance` | Forecast precipitation probability or equivalent |

### Recommended Decision Pattern

Set `weather_alert = true` when any one of these is true:

- severe condition contains storm or thunderstorm
- snow is present
- precipitation probability is above threshold
- rainfall amount exceeds field-operations threshold

Otherwise set:

- `weather_alert = false`

### Operational Note

This workflow should flag risk, not automatically reschedule work. Human schedulers should retain control over final changes unless the business explicitly adds auto-rescheduling later.

---

## Phase 6 - Calendar Upsert Logic

This phase is the core scheduling action.

## Upsert Rule

- If `Calendar Event ID` exists, update the existing Google Calendar event.
- If `Calendar Event ID` is blank, create a new event.

This prevents duplicates and keeps Google Calendar aligned with sheet edits.

### Node 13: Decide Create vs Update

Node name: `Check Existing Calendar Event`

Purpose:

- inspect whether the row already has an event ID

Recommended routing:

| Condition | Action |
|---|---|
| `Calendar Event ID` is blank | Create event |
| `Calendar Event ID` is populated | Update event |

### Best-Practice Duplicate Prevention

Primary method:

- Store the event ID returned by Google Calendar back into the sheet.

Secondary safety method:

- Include a unique identifier in the event description such as `Project ID` plus `Task Name`.

Why the secondary method helps:

- If a prior execution created an event but failed before writing back the event ID, the admin can still identify the event manually.

### Node 14A: Create Calendar Event

Node name: `Create Calendar Event`

Purpose:

- create a new Google Calendar event when no existing event ID is stored

### Node 14B: Update Existing Event

Node name: `Update Existing Event`

Purpose:

- update the matching event when the sheet already holds the Google Calendar event ID

### Event Date Logic

Recommended event date handling:

- event start = `Start Date` at the company default start time, or all-day event if preferred
- event end = `Start Date + Duration Days`

If the business schedules all remodeling tasks as all-day blocks, configure the calendar event accordingly. This is usually cleaner for project scheduling than assigning artificial hour-level times.

---

## Phase 7 - Event Formatting

Calendar events should be easily scannable by project managers and field leads.

### Title Rules

Normal title format:

`Project Name - Task Name`

Weather-risk title format:

`⚠️ WEATHER ALERT - Project Name - Task Name`

Examples:

- `Mason Residence - Interior Framing`
- `⚠️ WEATHER ALERT - Lakeview Addition - Roofing`

### Description Content

The event description should include:

- project ID
- task name
- task type
- site ZIP code
- forecast summary
- temperature summary
- rain chance
- last sync time

### Recommended Description Template

```text
Project ID: AR-1002
Project Name: Lakeview Addition
Task Name: Roofing
Task Type: Exterior
Site Zip: 60031
Forecast Summary: Heavy rain expected during scheduled workday
Temperature: High 58F / Low 49F
Rain Chance: 75%
Last Sync Time: 2026-05-02 08:30:00
```

### Visual Formatting Recommendations

| Event type | Recommendation |
|---|---|
| Standard event | Default calendar color |
| Weather-risk event | Distinct color such as red or orange |
| Forecast unavailable | Neutral color with note in description |

---

## Phase 8 - Write Back to Sheet

After each successful calendar action, write the result back to the same row.

### Node 15: Update Sheet After Calendar Sync

Node name: `Write Back Sync Results`

Purpose:

- store identifiers and sync outcomes in Google Sheets

Required write-back fields:

| Field | Value to store |
|---|---|
| `Calendar Event ID` | Event ID returned by Google Calendar |
| `Last Synced At` | Current execution timestamp |
| `Status` | `Synced` or `Weather Risk` depending on result |

Recommended status rules:

| Situation | Status |
|---|---|
| Interior task synced successfully | `Synced` |
| Exterior task synced with no risk | `Synced` |
| Exterior task synced with weather alert | `Weather Risk` |
| Forecast unavailable but event created | `Forecast Unavailable` |
| Validation failed | `Validation Error` |

### Sheet Update Best Practice

Use the row number or a stable row lookup key when writing back. Do not rely only on project name text matching, because similar project names can cause accidental updates.

---

## Phase 9 - Error Handling

Do not leave failures inside the main workflow without notification.

## Failure Types to Plan For

| Failure type | Response |
|---|---|
| Weather API failure | Notify admin, optionally mark row as `Weather API Error` |
| Missing required data | Flag row and notify admin |
| Google Calendar failure | Notify admin and keep row unsynced for retry |
| Google Sheets write-back failure | Notify admin because duplicate prevention may be at risk |

### Weather API Failure Handling

If the weather call fails for an exterior task:

1. Decide whether the task should stop or continue.
2. For most contractors, the safer approach is:
   - do not create a weather-based risk flag
   - notify the admin
   - optionally leave the row in `Pending Review`

If the business wants schedule continuity, a second acceptable rule is:

- create the calendar event
- set status to `Forecast Unavailable`
- notify the admin that weather confirmation did not complete

### Calendar Failure Handling

If Google Calendar create or update fails:

1. Send an admin alert with project and task details.
2. Do not mark the row as synced.
3. Keep the status in a retry-friendly state such as `Pending` or `Calendar Error`.

### Separate Error Workflow

Create a second workflow using an `Error Trigger`.

Workflow shape:

`Error Trigger - Project Task Sync`  
`-> Format Failure Context`  
`-> Notify Admin Failure`

Alert details should include:

- workflow name
- execution ID
- failed node
- error message
- timestamp

---

## 7. Recommended Node Naming Convention

Use descriptive node names so future debugging is fast and obvious.

| Node Type | Recommended Name |
|---|---|
| Schedule Trigger | `Schedule Trigger - Task Sync` |
| Google Sheets read | `Read Project Tasks` |
| Set / Edit Fields | `Normalize Task Data` |
| IF validation | `Validate Required Fields` |
| Google Sheets update | `Flag Validation Error` |
| Slack or Gmail alert | `Notify Admin Validation Failure` |
| IF task type | `Check Exterior Task` |
| Set / Edit Fields | `Prepare Weather Request` |
| HTTP Request | `Resolve Site Coordinates` |
| HTTP Request | `Fetch Weather Forecast` |
| Code or Set logic | `Detect Weather Risk` |
| IF event ID exists | `Check Existing Calendar Event` |
| Google Calendar create | `Create Calendar Event` |
| Google Calendar update | `Update Existing Event` |
| Google Sheets update | `Write Back Sync Results` |
| Error Trigger workflow alert | `Notify Admin Failure` |

## Naming Rules

- Start with the business action, not the vendor name
- Keep names short enough to scan on the canvas
- Use consistent verb patterns such as `Read`, `Validate`, `Check`, `Fetch`, `Create`, `Update`, `Notify`

---

## 8. Testing Strategy

Run the workflow against controlled test rows before activating it in production.

## General Test Setup

- Use a dedicated test calendar first
- Use a copy of the source sheet for QA
- Keep admin alerts pointed to a test Slack channel or test inbox
- Save screenshots during each successful case

## Case A - Interior Framing Task

Test row:

- Task Type = `Interior`
- Valid start date
- ZIP code present or blank depending on your validation rule

Expected result:

- Weather branch is skipped
- Calendar event is created
- Standard title is used
- `Calendar Event ID` is written back
- `Status = Synced`

## Case B - Exterior Roofing with Clear Weather

Test row:

- Task Type = `Exterior`
- Weather forecast shows no significant precipitation

Expected result:

- Weather API is called
- `weather_alert = false`
- Standard event title is used
- Event description includes forecast summary
- `Status = Synced`

## Case C - Exterior Foundation Pour with Heavy Rain

Test row:

- Task Type = `Exterior`
- Forecast day contains heavy rain or high precipitation probability

Expected result:

- Weather API is called
- `weather_alert = true`
- Calendar title begins with `⚠️ WEATHER ALERT`
- Event description includes risk details
- `Status = Weather Risk`

## Case D - Date Changed in Sheet

Test row:

- Existing row already has `Calendar Event ID`
- Change the `Start Date` or `Duration Days`

Expected result:

- Workflow routes to update path
- Existing calendar event is updated, not recreated
- Event ID remains the same
- `Last Synced At` changes to the new execution time

## Case E - Missing ZIP Code

Test row:

- Task Type = `Exterior`
- ZIP code is blank

Expected result:

- Validation branch stops normal processing
- No weather call occurs
- No calendar event is created or updated
- Row is flagged as `Validation Error`
- Admin receives an alert

## Additional Recommended Test Cases

| Scenario | Expected outcome |
|---|---|
| Start date outside 5-day forecast | Event can still be created, status marked `Forecast Unavailable` if using that rule |
| Invalid ZIP code | Admin alert or row error status, depending on provider response |
| Calendar API temporary failure | Row remains unsynced for retry |
| Prior event exists but sheet event ID is blank | Manual review should still be possible via description identifier |

---

## 9. Debugging Checklist

Use this checklist when behavior does not match expectations.

| Issue | Likely cause | What to check |
|---|---|---|
| Wrong calendar date | Timezone mismatch | Compare timezone in Google Sheets, n8n, Google Calendar, and weather parsing logic |
| Duplicate events | Event ID not written back or lost | Verify `Calendar Event ID` write-back executed after calendar create |
| Weather not matching the task day | Forecast parsing logic selected wrong entry | Confirm task date and forecast date are compared in the same timezone |
| Invalid ZIP code errors | Dirty source data | Check ZIP formatting and whether the provider requires country context |
| API rate limits | Too many test calls or aggressive polling | Increase polling interval or reduce unnecessary reprocessing |
| Google Calendar auth errors | Expired or mis-scoped credential | Reconnect the credential and confirm calendar permissions |
| Trigger not catching edits | Trigger limitations or row-update pattern mismatch | Switch to scheduled polling for more predictable sync behavior |
| Write-back updates wrong row | Weak row identification | Use row number or stable key, not loose text matching |

## Practical Debug Sequence

1. Confirm the input row has the expected values.
2. Confirm validation output is correct.
3. Confirm the correct branch was taken for `Interior` vs `Exterior`.
4. Inspect the raw weather API response.
5. Inspect the final event payload before calendar create or update.
6. Confirm the sheet write-back ran after the calendar step.

---

## 10. Deliverables / Proof of Work Plan

Capture the following evidence during build and testing.

## Required Screenshots

| Screenshot | What it should show |
|---|---|
| Workflow canvas overview | Main workflow with visible major branches |
| Trigger and sheet read configuration | Trigger setup and source sheet selection |
| Validation branch | Logic for invalid-row handling |
| Weather API response | Successful forecast response node for an exterior task |
| Weather risk decision | Output showing `weather_alert = true` or `false` |
| Google Calendar event created | New event visible in the test calendar |
| Updated event after sheet change | Same event updated after editing the source row |
| Sheet write-back result | `Calendar Event ID` and `Last Synced At` populated |
| Admin notification test | Slack or Gmail alert from a forced failure or validation error |
| Execution history | Successful run log and one handled error case |

## Suggested Proof-of-Work Sequence

1. Screenshot the blank or initial source sheet.
2. Screenshot the workflow canvas before activation.
3. Run Case A and capture execution plus created event.
4. Run Case C and capture weather-risk output and flagged event title.
5. Edit a previously synced row and capture the update behavior.
6. Force a validation failure and capture the admin alert.

---

## 11. Optimization Ideas

Once the base workflow is stable, consider these upgrades.

| Enhancement | Business value |
|---|---|
| Crew SMS alerts | Gives field leads immediate visibility for weather-risk jobs |
| Material delay notifications | Aligns schedule changes with supply chain disruptions |
| Weekly weather digest | Gives project managers a forecast-based planning overview |
| Map integration | Improves job-site visualization and route planning |
| Resource allocation dashboard | Helps assign crews against upcoming schedule load |
| Auto-reschedule suggestions | Proposes safer dates for weather-sensitive exterior work |
| Risk threshold by task type | Roofing, concrete, and painting can each use different weather rules |
| Cancelled task handling | Automatically update or remove events for cancelled work |

---

## 12. Recommended Build Order Summary

If the implementer wants the cleanest execution path, build in this order:

1. Prepare the Google Sheet and sample rows.
2. Configure credentials for Google Sheets, Google Calendar, weather API, and Slack or Gmail.
3. Build the main workflow scaffold with trigger, sheet read, and candidate filtering.
4. Add normalization and required-field validation.
5. Add the `Interior` vs `Exterior` branch logic.
6. Build the weather branch and verify the raw forecast response.
7. Add weather risk detection logic and test both risky and non-risky outcomes.
8. Build the calendar create/update split using `Calendar Event ID`.
9. Add sheet write-back logic.
10. Build the separate error workflow.
11. Run the test cases and capture proof-of-work screenshots.
12. Activate the workflow only after successful update-path testing.

---

## 13. Final Consultant Recommendation

For a real contractor environment, the strongest first version is:

- scheduled polling every 15 minutes
- Google Sheets as the scheduling source of truth
- Google Calendar as the operational crew calendar
- weather checks only for exterior tasks
- event ID write-back for duplicate prevention
- Slack or Gmail alerts for failures

That combination is practical, maintainable, easy to explain to operations staff, and strong enough for a production pilot without overengineering the first release.
