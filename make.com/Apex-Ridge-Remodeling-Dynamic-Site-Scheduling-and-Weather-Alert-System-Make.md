# Apex Ridge Remodeling: Dynamic Site Scheduling & Weather Alert System in Make.com

Prepared for: Apex Ridge Remodeling  
Document type: Make.com implementation guide  
Purpose: Step-by-step build plan for syncing remodeling tasks from Google Sheets into Google Calendar with weather-aware scheduling for exterior work

## 1. Executive Summary

This automation solves a scheduling problem that is common in remodeling operations: project tasks are planned in a spreadsheet, field visibility happens in a calendar, and exterior work often carries weather risk that gets checked too late.

Without automation, office staff must manually:

1. review new or updated task rows in Google Sheets
2. determine whether the work is interior or exterior
3. check the weather for exterior jobs
4. create or update the related Google Calendar event
5. remember to avoid duplicate events
6. notify someone when something breaks

In a contractor environment, that manual process creates real operational waste:

- duplicate calendar entries
- missed schedule updates
- crews assigned to weather-risk work without warning
- preventable admin time spent reconciling sheet data versus calendar data

This Make.com scenario gives Apex Ridge a cleaner operating model:

- Google Sheets remains the source of truth
- Make.com handles validation, branching, weather checks, and sync logic
- Google Calendar becomes the live scheduling view
- Slack or Gmail handles exception visibility

### Operational Impact

- Field crews get a more reliable calendar
- Exterior tasks are visibly flagged when forecast conditions are risky
- Office staff can update one row instead of managing multiple systems
- Admins can see failures quickly and correct them before crews are affected

---

## 2. Full Architecture Overview

## Core Systems

| System | Role |
|---|---|
| `Make.com` | Orchestration layer that watches sheet data, validates fields, branches by task type, calls the weather API, creates or updates calendar events, writes results back, and handles exceptions |
| `Google Sheets` | Source task register for remodeling jobs |
| `Google Calendar` | Crew-facing schedule destination |
| `Weather API` | Forecast source for exterior task risk evaluation |
| `Slack` or `Gmail` | Admin notification channel for failures and validation issues |

## Recommended Scenario Set

Build two scenarios:

1. `Apex Ridge - Task Sync`
2. `Apex Ridge - Error Notifications`

The main scenario handles row processing, weather checks, and calendar sync.  
The second scenario can be used if you want a dedicated notification flow for escalations or logged errors.

## End-to-End Data Flow

`Google Sheets Row Added or Updated`  
`-> Make.com Scenario Trigger`  
`-> Normalize and Validate Row Data`  
`-> Router: Interior vs Exterior`

Interior path:  
`-> Build Calendar Payload`  
`-> Search Existing Calendar Event`  
`-> Create or Update Event`  
`-> Update Sheet with Event ID and Sync Timestamp`

Exterior path:  
`-> Call Weather API`  
`-> Parse Forecast for Scheduled Date`  
`-> Detect Weather Risk`  
`-> Build Calendar Payload with Alert Data`  
`-> Search Existing Calendar Event`  
`-> Create or Update Event`  
`-> Update Sheet with Event ID, Sync Timestamp, and Risk Status`

Failure path:  
`Validation Error or API Failure`  
`-> Update Sheet Status`  
`-> Send Slack or Gmail Alert`

## Architecture Diagram in Text Bullets

- Source layer
  - Google Sheets stores project scheduling rows
- Trigger layer
  - Make.com watches for row changes or runs on a schedule
- Validation layer
  - Required fields are checked before downstream modules run
- Decision layer
  - Router splits Interior and Exterior work
- Enrichment layer
  - Exterior tasks call the weather provider
- Risk layer
  - Weather conditions are converted into a business-friendly alert flag
- Action layer
  - Google Calendar event is created or updated
- Audit layer
  - Google Sheet row is updated with IDs, timestamps, and statuses
- Exception layer
  - Slack or Gmail sends admin notifications

---

## 3. Suggested Time Breakdown

This estimate is appropriate for a real tracked-hours build in Make.com.

| Task | Hours |
|---|---:|
| Project review and business rule confirmation | 1.0 |
| Make.com connection setup | 1.0 |
| Google Sheet schema prep and test data | 0.75 |
| Google Calendar destination prep | 0.5 |
| Scenario scaffold and routing design | 1.0 |
| Trigger selection and initial module wiring | 1.25 |
| Data validation and field normalization | 1.25 |
| Interior vs Exterior routing logic | 0.75 |
| Weather API integration | 1.75 |
| Forecast parsing and date matching | 1.75 |
| Weather risk rules | 1.0 |
| Calendar duplicate prevention logic | 2.0 |
| Calendar create/update modules | 1.5 |
| Google Sheets write-back modules | 1.0 |
| Slack or Gmail failure notifications | 1.0 |
| Error handling and retries review | 0.75 |
| Testing and debugging | 2.0 |
| Proof-of-work screenshots and documentation | 1.0 |
| **Total Estimated Effort** | **18.25** |

## Suggested Delivery Phases

| Phase | Focus | Hours |
|---|---|---:|
| Phase 0 | Preparation and access | 2.25 |
| Phase 1 | Trigger and row processing setup | 2.25 |
| Phase 2 | Validation and routing | 2.0 |
| Phase 3 | Weather enrichment and risk logic | 3.5 |
| Phase 4 | Calendar upsert design | 3.5 |
| Phase 5 | Sheet write-back and notifications | 2.0 |
| Phase 6 | Testing, QA, and handoff capture | 3.0 |
| **Total** |  | **18.5** |

---

## 4. Pre-Build Preparation Checklist

Complete this before opening Make.com.

## Access Checklist

- [ ] Make.com workspace access with permission to create scenarios and connections
- [ ] Google account access for the source spreadsheet
- [ ] Permission to edit the spreadsheet
- [ ] Google Calendar access with event create/update permission
- [ ] Weather API account and active API key
- [ ] Slack webhook or Slack app access, or Gmail account access

## Data Preparation Checklist

- [ ] Spreadsheet created and named
- [ ] `Project Tasks` tab created
- [ ] Required columns added
- [ ] At least three realistic test rows entered
- [ ] Task Type values standardized to `Interior` and `Exterior`
- [ ] Dates confirmed in a consistent format
- [ ] ZIP codes checked for correctness
- [ ] Duration values entered as numbers

## Configuration Checklist

- [ ] Business timezone confirmed
- [ ] Target Google Calendar confirmed
- [ ] Polling interval or watch strategy confirmed
- [ ] Admin notification recipient confirmed
- [ ] Weather risk threshold approved
- [ ] Handling for forecast dates outside the 5-day horizon agreed

---

## 5. Google Sheet Design

Recommended spreadsheet name: `Apex Ridge - Project Schedule`  
Recommended tab: `Project Tasks`

## Required Columns

| Column | Required | Purpose |
|---|---|---|
| `Project ID` | Yes | Stable project identifier |
| `Project Name` | Yes | Used in event titles |
| `Task Name` | Yes | Specific work activity |
| `Task Type` | Yes | Determines whether weather logic is needed |
| `Start Date` | Yes | Scheduled date for the task |
| `Site Zip Code` | Yes for exterior tasks | Weather lookup input |
| `Duration Days` | Yes | Used to calculate event duration |
| `Calendar Event ID` | Yes for update logic | Stores linked Google Calendar event ID |
| `Last Synced At` | No | Timestamp of the latest successful sync |
| `Status` | Yes | Operational state of the row |

## Why `Calendar Event ID` Is Required

This is the primary duplicate-prevention key.

When the scenario creates a Google Calendar event for the first time, Make.com should write the resulting event ID back into the sheet. On future runs, the scenario can check that field:

- if it is blank, create a new event
- if it has a value, update the existing event

Without this field, the scenario would need unreliable matching logic based on titles or dates, which is not appropriate for production scheduling.

## Sample Rows

| Project ID | Project Name | Task Name | Task Type | Start Date | Site Zip Code | Duration Days | Calendar Event ID | Last Synced At | Status |
|---|---|---|---|---|---|---:|---|---|---|
| `AR-1001` | `Mason Residence` | `Interior Framing` | `Interior` | `2026-05-04` | `60614` | 2 |  |  | `Pending` |
| `AR-1002` | `Lakeview Addition` | `Roofing` | `Exterior` | `2026-05-06` | `60031` | 1 |  |  | `Pending` |
| `AR-1003` | `Hawthorne Renovation` | `Foundation Pour` | `Exterior` | `2026-05-07` | `60120` | 1 | `6k8m2exampleeventid` | `2026-05-01 09:12:22` | `Synced` |

---

## 6. Step-by-Step Make.com Build Plan

This build plan is written for someone manually creating modules, routers, filters, and update paths in Make.com.

## Recommended Main Scenario Name

`Apex Ridge - Task Sync`

## Recommended Main Scenario Structure

`Trigger`  
`-> Get/Watch Project Rows`  
`-> Validate Required Fields`  
`-> Router: Valid vs Invalid`

Invalid path:  
`-> Update Google Sheet Status`  
`-> Notify Admin`

Valid path:  
`-> Router: Interior vs Exterior`

Interior path:  
`-> Set Default Weather Values`  
`-> Search Existing Calendar Event`  
`-> Create or Update Event`  
`-> Update Sheet Results`

Exterior path:  
`-> HTTP Request to Weather API`  
`-> Parse Forecast for Task Date`  
`-> Detect Weather Risk`  
`-> Search Existing Calendar Event`  
`-> Create or Update Event`  
`-> Update Sheet Results`

---

### Phase 1 - Trigger Layer

## Goal

Detect rows that need to be processed.

## Trigger Options

| Option | How It Works | Pros | Cons |
|---|---|---|---|
| `Watch New Rows / Updated Rows` | Reacts to sheet changes | More immediate | Can be less predictable to debug during build |
| `Scheduled Scenario` | Runs every X minutes and reads the sheet | Easier to test, more controlled | Not instant |

## Consultant Recommendation

Start with a scheduled scenario that runs every 15 minutes.

This is usually the safest first version in Make.com because it gives you clean execution cycles, clearer logs, and easier retesting while the scenario is still being developed.

## Build Steps

1. Create a new scenario in Make.com.
2. Add the Google Sheets connection.
3. Use a Google Sheets module that can read the source rows from the `Project Tasks` tab.
4. Configure the scenario schedule to run every 15 minutes.
5. If needed, add filtering logic so only rows with relevant statuses are processed.

Recommended statuses to process:

- `Pending`
- `Needs Update`
- `Weather Risk`
- `Forecast Unavailable`

## Practical Note

If your chosen Google Sheets module does not give you reliable row references for updating later, add a helper column or ensure the row number is captured early in the scenario.

---

### Phase 2 - Data Validation

## Goal

Stop incomplete or invalid rows before they create bad calendar events.

## Required Validation Fields

- `Project Name`
- `Task Type`
- `Start Date`
- `Duration Days`
- `Site Zip Code` for exterior tasks

## Build Steps

1. Add a module or tools step to normalize incoming values.
2. Trim blank strings where needed.
3. Standardize Task Type values to expected forms.
4. Add filters or a router for valid versus invalid rows.
5. On the invalid path:
   - update the row `Status` to `Validation Error`
   - optionally populate a `Last Error` field if you add one
   - send a Slack or Gmail notification
6. Stop processing for invalid rows after notification.

## Recommended Validation Rules

| Field | Rule |
|---|---|
| `Project Name` | Must not be blank |
| `Task Type` | Must equal `Interior` or `Exterior` |
| `Start Date` | Must be parseable as a valid date |
| `Duration Days` | Must be numeric and greater than zero |
| `Site Zip Code` | Required when task type is `Exterior` |

---

### Phase 3 - Task Classification

## Goal

Split processing so only exterior work goes through weather evaluation.

## Router Logic

| Condition | Route |
|---|---|
| `Task Type = Interior` | Skip weather modules |
| `Task Type = Exterior` | Continue to weather modules |

## Build Steps

1. Add a Router after the validation step.
2. Create one route for interior tasks.
3. Create one route for exterior tasks.
4. On the interior route, set default weather values such as:
   - `weather_alert = false`
   - `forecast_summary = Weather check not required`
5. Pass both paths into the shared calendar upsert section later in the scenario.

---

### Phase 4 - Weather API Integration

## Goal

Retrieve a 5-day forecast for exterior tasks and find the forecast entries relevant to the scheduled work date.

## Recommended Module

`HTTP` module in Make.com

## Build Steps

1. Add an HTTP request module on the exterior route.
2. Configure the request to call your weather provider.
3. Pass the ZIP code dynamically from the sheet row.
4. Include the API key securely using the connection or request settings.
5. Request the forecast in JSON format.
6. Inspect the returned forecast structure carefully.
7. Add parsing logic to isolate the entries matching the task date.

## Parsing Guidance

The implementer should:

1. convert the Google Sheet date into the business timezone
2. compare that date against the forecast timestamps
3. select the forecast entries that align with the scheduled date
4. summarize the relevant conditions for the event description and risk logic

## Handling Dates Outside the Forecast Window

If the task is scheduled beyond the provider's 5-day range:

- do not guess weather conditions
- set a clear status such as `Forecast Unavailable`
- optionally still create the calendar event without the alert prefix

---

### Phase 5 - Risk Detection Logic

## Goal

Turn raw weather data into a clear operational decision.

## Required Output

Create a Boolean-style decision:

- `weather_alert = true`
- `weather_alert = false`

## Suggested Risk Rules

Mark the task as risky if forecast data shows:

- heavy rain
- snow
- thunderstorm or storm conditions
- high precipitation probability

## Suggested Threshold Table

| Condition | Suggested Rule |
|---|---|
| Rain probability | 50% or greater |
| Snow | Any snow for exterior work |
| Thunderstorm | Any thunderstorm condition |
| Severe storm wording | Treat as risky |

## Build Steps

1. Add a tools or data transformation step after the HTTP module.
2. Extract the most relevant forecast details for the task date.
3. Build the following values:
   - `weather_alert`
   - `risk_status`
   - `forecast_summary`
   - `temperature_summary`
   - `rain_chance`
4. Ensure the logic is readable enough that another builder can maintain it later.

---

### Phase 6 - Calendar Upsert Logic

## Goal

Create a calendar event when no linked event exists, and update it when the row already has one.

## Best Make.com Approach

Use `Calendar Event ID` as the upsert key.

## Upsert Logic Table

| Condition | Action |
|---|---|
| `Calendar Event ID` is blank | Create new event |
| `Calendar Event ID` is populated | Update existing event |

## Build Steps

1. Add a router or filter after payload preparation.
2. Create one path for rows with blank `Calendar Event ID`.
3. Create one path for rows with a populated `Calendar Event ID`.
4. Use the Google Calendar `Create an Event` action on the create path.
5. Use the Google Calendar `Update an Event` action on the update path.
6. Store the returned event ID back into Google Sheets after success.

## Why This Prevents Duplicates

This keeps the relationship one row to one event.

The sheet row remains the source record, and the calendar event ID acts as the persistent link between systems. That is much safer than searching by title or date alone.

---

### Phase 7 - Event Formatting

## Goal

Make the calendar event useful for dispatch and scheduling decisions.

## Title Rules

| Scenario | Title |
|---|---|
| Standard task | `Project Name – Task Name` |
| Risky exterior task | `⚠️ WEATHER ALERT – Project Name – Task Name` |

## Description Requirements

Each event description should include:

- task type
- address or ZIP code
- forecast summary
- temperature
- rain chance
- last sync time

## Build Steps

1. Add a mapping step before the calendar module.
2. Create the event title using `weather_alert`.
3. Build a structured event description using sheet and weather values.
4. Keep description formatting identical in both create and update paths.

## Recommended Description Content

| Field | Reason |
|---|---|
| Project ID | Easy office lookup |
| Task Type | Clarifies job classification |
| ZIP or site location | Gives field context |
| Forecast summary | Supports exterior planning |
| Temperature | Helpful for site readiness |
| Rain chance | Supports go/no-go review |
| Last synced at | Confirms the data is fresh |

---

### Phase 8 - Write Back to Sheet

## Goal

Update the source row so the spreadsheet remains an operational control panel.

## Required Write-Back Fields

- `Calendar Event ID`
- `Last Synced At`
- `Status`

## Recommended Additional Fields

- `Risk Status`
- `Last Error`

## Status Recommendations

| Scenario | Suggested Status |
|---|---|
| Successful interior sync | `Synced` |
| Successful exterior sync with no risk | `Synced` |
| Exterior sync with weather risk | `Weather Risk` |
| Outside forecast horizon | `Forecast Unavailable` |
| Missing required data | `Validation Error` |
| API or calendar failure | `Sync Failed` |

## Build Steps

1. Add a Google Sheets update module after successful create.
2. Add a Google Sheets update module after successful update.
3. Write the event ID into `Calendar Event ID`.
4. Write the current timestamp into `Last Synced At`.
5. Write the business-friendly status value.
6. If using `Last Error`, clear it on successful sync.

---

### Phase 9 - Error Handling

## Goal

Ensure failures are visible and actionable.

## Failure Types to Plan For

| Failure Type | Response |
|---|---|
| Missing required row data | Mark row and notify admin |
| Weather API failure | Notify admin and preserve row for retry |
| Google Calendar create/update failure | Notify admin and keep row unsynced |
| Google Sheets write-back failure | Notify admin because the audit trail is incomplete |

## Build Steps

1. Add Make.com error handlers where appropriate on critical modules.
2. For expected data issues, route through normal validation paths rather than relying only on technical error handlers.
3. Send Slack or Gmail alerts with the row context and failure message.
4. Keep the row in a status that allows reprocessing after correction.

## Alert Content Recommendations

Include:

- scenario name
- project name
- task name
- row reference
- failed module
- timestamp
- error summary
- recommended next action

---

## 7. Recommended Module Naming Convention

Use business-readable names rather than default module labels.

| Purpose | Recommended Name |
|---|---|
| Trigger | `Watch Project Tasks` or `Scheduled Task Sync Trigger` |
| Read rows | `Read Project Task Rows` |
| Validation | `Validate Required Fields` |
| Invalid update | `Flag Validation Error` |
| Notification | `Notify Admin - Validation Failure` |
| Router | `Route by Task Type` |
| Weather call | `Fetch Weather Forecast` |
| Forecast parsing | `Match Forecast to Task Date` |
| Risk logic | `Detect Weather Risk` |
| Event mapper | `Build Calendar Payload` |
| Search existing event | `Check Existing Calendar Event` |
| Create event | `Create Calendar Event` |
| Update event | `Update Existing Calendar Event` |
| Sheet write-back | `Write Sync Results to Sheet` |
| Failure alert | `Notify Admin - Sync Failure` |

---

## 8. Testing Strategy

Use real test rows and document expected outcomes.

### Case A - Interior Framing Task

| Test Input | Expected Result |
|---|---|
| Interior task with valid date and duration | Weather branch is skipped |
| Calendar result | Event is created normally |
| Sheet result | Event ID and sync timestamp written back |
| Status | `Synced` |

### Case B - Exterior Roofing With Clear Weather

| Test Input | Expected Result |
|---|---|
| Exterior task with valid ZIP and clear forecast | Weather API is called |
| Risk result | `weather_alert = false` |
| Calendar result | Standard event title |
| Sheet result | Event ID written back |

### Case C - Exterior Foundation Pour With Heavy Rain

| Test Input | Expected Result |
|---|---|
| Exterior task with rainy forecast | Risk is detected |
| Calendar result | Title includes warning prefix |
| Sheet result | Status becomes `Weather Risk` |
| Description | Includes forecast summary and rain chance |

### Case D - Date Changed in Sheet

| Test Input | Expected Result |
|---|---|
| Existing row already has event ID, then Start Date changes | Existing event is updated |
| Duplicate prevention | No new event should be created |
| Sheet result | Same event ID remains linked |

### Case E - Missing ZIP Code

| Test Input | Expected Result |
|---|---|
| Exterior row missing ZIP code | Validation path triggers |
| Calendar result | No event created |
| Notification | Admin alert sent |
| Sheet result | Status becomes `Validation Error` |

## Additional Recommended Tests

- exterior task scheduled outside 5-day forecast window
- invalid date format in Google Sheets
- weather API temporary outage
- Google Calendar permission failure
- row correction after a previous validation failure

---

## 9. Debugging Checklist

- [ ] Confirm the scenario timezone matches the intended business timezone
- [ ] Confirm Google Sheets dates are being interpreted correctly
- [ ] Check whether `Calendar Event ID` is blank when duplicates occur
- [ ] Confirm ZIP code format is accepted by the weather provider
- [ ] Check API key validity and usage limits
- [ ] Confirm Google Calendar module is targeting the correct calendar
- [ ] Verify Google Sheets update module is writing to the correct row
- [ ] Confirm filters are not accidentally skipping valid rows
- [ ] Confirm alert titles and descriptions are mapped from the correct values

## Common Issues Table

| Issue | Likely Cause |
|---|---|
| Duplicate calendar events | Missing or overwritten `Calendar Event ID` |
| Wrong event date | Timezone or date parsing mismatch |
| Weather alerts not appearing | Forecast date matching logic is incorrect |
| Calendar update fails | Stale event ID or permission issue |
| Row never syncs | Filter logic excludes it or status is not eligible |
| Admin not notified | Slack/Gmail module misconfigured or on wrong route |

---

## 10. Deliverables / Proof of Work Plan

Capture the following screenshots for handoff and documentation.

| Screenshot | Purpose |
|---|---|
| Full Make.com scenario canvas | Shows overall structure and routing |
| Trigger settings | Shows schedule or watch configuration |
| Validation route | Proves missing data handling exists |
| Weather API module output | Proves exterior forecast retrieval |
| Risk detection logic output | Proves alert rules are functioning |
| Calendar event created | Proves successful event creation |
| Updated calendar event after row change | Proves update logic instead of duplicate creation |
| Sheet row after sync | Shows event ID, sync time, and status written back |
| Failure notification example | Shows Slack or Gmail alert formatting |

## Recommended Final Handoff Package

- final scenario screenshot
- test case screenshots
- sample successful interior run
- sample successful exterior clear-weather run
- sample successful exterior weather-risk run
- sample validation failure run
- short note explaining statuses and retry behavior

---

## 11. Optimization Ideas

After the base scenario is stable, consider:

| Upgrade | Value |
|---|---|
| Crew SMS alerts | Faster field communication |
| Material delay notifications | Coordinate schedule shifts with supply issues |
| Weekly weather digest | Gives PMs a forward view of exterior risk |
| Map links in event descriptions | Easier navigation for crews |
| Crew assignment logic | Supports resource planning |
| Auto-reschedule suggestions | Higher-value decision support for weather-risk jobs |
| Separate calendars by trade | Improves operational clarity |
| Data store for audit history | Better reporting and recovery options |

---

## Recommended Build Order Summary

1. Prepare sheet structure, calendar target, and API credentials
2. Build the main scenario trigger and row retrieval
3. Add validation logic and invalid-row handling
4. Add the interior vs exterior router
5. Build the weather API call and forecast parsing
6. Add risk detection rules
7. Build calendar create/update logic using `Calendar Event ID`
8. Add sheet write-back modules
9. Add Slack or Gmail notifications
10. Test each scenario path and capture screenshots

---

## Final Notes

The most important design choices in this Make.com build are:

- Google Sheets remains the source of truth
- `Calendar Event ID` is used as the duplicate-prevention key
- weather checks only apply to exterior work
- statuses are written back into the sheet for visibility
- failures are surfaced immediately to an admin

If built in this order, the scenario will be easier to test, easier to maintain, and much safer to operate in a real remodeling scheduling environment.
