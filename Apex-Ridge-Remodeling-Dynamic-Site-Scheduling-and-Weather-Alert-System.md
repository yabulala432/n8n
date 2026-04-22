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

This section is written as a true build guide so someone can open n8n and assemble the workflow node by node without guessing the order.

## Workflow Names

- Main workflow: `Apex Ridge - Project Task Sync`
- Error workflow: `Apex Ridge - Scheduling Error Handler`

## Recommended Concrete Stack for This Guide

To keep the build specific, this guide assumes:

- `Schedule Trigger` for the main trigger
- `Google Sheets` for the source table and write-back
- `Google Calendar` for scheduling
- `HTTP Request` for the weather API
- `Slack` or `Gmail` for admin alerts

If you want near real-time response later, you can replace the first node with `Google Sheets Trigger`, but build the scheduled version first because it is easier to test and debug.

## Before You Open the Canvas

Create these credentials in n8n first:

- `Google Sheets OAuth2`
- `Google Calendar OAuth2`
- Weather API key or credential
- `Slack` credential or `Gmail` credential

Confirm these business values before building:

- Spreadsheet: `Apex Ridge - Project Schedule`
- Tab: `Project Tasks`
- Target calendar name
- Company timezone
- Polling frequency, recommended `15 minutes`

---

## Main Workflow Canvas Order

Build the main workflow in this exact order:

`Schedule Trigger - Task Sync`  
`-> Read Project Tasks`  
`-> Filter Rows Needing Sync`  
`-> Loop Through Candidate Rows`  
`-> Normalize Task Data`  
`-> Evaluate Validation Rules`  
`-> Is Task Valid?`

Invalid branch:

`-> Flag Validation Error`  
`-> Notify Admin Validation Failure`

Valid branch:

`-> Check Exterior Task`

Interior branch:

`-> Set Interior Defaults`

Exterior branch:

`-> Prepare Weather Request`  
`-> Resolve Site Coordinates`  
`-> Fetch Weather Forecast`  
`-> Detect Weather Risk`

Then continue to:

`-> Build Calendar Payload`  
`-> Check Existing Calendar Event`

Create path:

`-> Create Calendar Event`  
`-> Write Back Sync Results`

Update path:

`-> Update Existing Event`  
`-> Write Back Sync Results`

If your n8n version prefers an explicit join before `Build Calendar Payload`, insert a `Merge` node named `Merge Prepared Task` before the shared calendar steps.

---

## 6.1 Create the Main Workflow

1. Log in to n8n.
2. Click `Create Workflow`.
3. Rename it to `Apex Ridge - Project Task Sync`.
4. Click `Save`.

---

## 6.2 Build Each Node

### Node 1: Schedule Trigger

### Node Name

`Schedule Trigger - Task Sync`

### Purpose

Starts the sync automatically every 15 minutes.

### How to Add It

1. Click `Add first step`.
2. Search for `Schedule Trigger`.
3. Add the node.
4. Rename it to `Schedule Trigger - Task Sync`.

### Exact Settings

| Setting | Value |
|---|---|
| `Trigger Interval` | `Every X` |
| `Value` | `15` |
| `Unit` | `Minutes` |

### Connect To

`Read Project Tasks`

---

### Node 2: Google Sheets - Read Rows

### Node Name

`Read Project Tasks`

### Purpose

Reads rows from the `Project Tasks` tab.

### How to Add It

1. Click the `+` after `Schedule Trigger - Task Sync`.
2. Search for `Google Sheets`.
3. Choose the read operation that returns many rows.
4. Rename it to `Read Project Tasks`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Sheets credential |
| `Spreadsheet` | `Apex Ridge - Project Schedule` |
| `Sheet` | `Project Tasks` |
| `Operation` | Read rows / Get many rows |
| `Use Header Row` | `On` |

### Important Note

If the Google Sheets node exposes row number metadata, keep it. If it does not, add a helper column such as `Row ID` to the sheet and use that later for write-back.

### Connect To

`Filter Rows Needing Sync`

---

### Node 3: Code

### Node Name

`Filter Rows Needing Sync`

### Purpose

Keeps only rows that still need to be created or updated in Google Calendar.

### How to Add It

1. Click the `+` after `Read Project Tasks`.
2. Search for `Code`.
3. Add the node.
4. Rename it to `Filter Rows Needing Sync`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for All Items` |

Paste this code:

```javascript
const allowedStatuses = [
  'Pending',
  'Ready',
  'Needs Update',
  'Weather Risk',
  'Forecast Unavailable',
];

return items.filter((item) => {
  const status = (item.json.Status ?? '').toString().trim();
  const eventId = (item.json['Calendar Event ID'] ?? '').toString().trim();

  if (allowedStatuses.includes(status)) return true;
  if (!status && !eventId) return true;

  return false;
});
```

### Connect To

`Loop Through Candidate Rows`

---

### Node 4: Loop Over Items

### Node Name

`Loop Through Candidate Rows`

### Purpose

Processes one row at a time so each weather call, calendar update, and sheet write-back stays tied to the correct row.

### How to Add It

1. Click the `+` after `Filter Rows Needing Sync`.
2. Search for `Loop Over Items`.
3. Add the node.
4. Rename it to `Loop Through Candidate Rows`.

### Exact Settings

Default settings are fine for the first build.

### Connect To

`Normalize Task Data`

---

### Node 5: Edit Fields (Set)

### Node Name

`Normalize Task Data`

### Purpose

Creates a consistent field structure for every later node.

### How to Add It

1. Click the `+` after `Loop Through Candidate Rows`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Normalize Task Data`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into `JSON Output`:

```json
{
  "project_id": "{{ ($json['Project ID'] ?? '').toString().trim() }}",
  "project_name": "{{ ($json['Project Name'] ?? '').toString().trim() }}",
  "task_name": "{{ ($json['Task Name'] ?? '').toString().trim() }}",
  "task_type": "{{ (($json['Task Type'] ?? '').toString().trim()).toLowerCase() === 'exterior' ? 'Exterior' : (($json['Task Type'] ?? '').toString().trim()).toLowerCase() === 'interior' ? 'Interior' : ($json['Task Type'] ?? '').toString().trim() }}",
  "start_date": "{{ ($json['Start Date'] ?? '').toString().trim() }}",
  "site_zip": "{{ ($json['Site Zip Code'] ?? '').toString().trim() }}",
  "duration_days": "{{ Number($json['Duration Days'] ?? 1) || 1 }}",
  "calendar_event_id": "{{ ($json['Calendar Event ID'] ?? '').toString().trim() }}",
  "status": "{{ ($json['Status'] ?? '').toString().trim() }}",
  "last_synced_at": "{{ ($json['Last Synced At'] ?? '').toString().trim() }}",
  "source_row_id": "{{ ($json['Row ID'] ?? $json.rowNumber ?? '').toString().trim() }}"
}
```

### What This Node Does

- Trims text values
- Normalizes `Task Type`
- Defaults blank duration to `1`
- Creates a clean JSON shape for the rest of the workflow

### Connect To

`Evaluate Validation Rules`

---

### Node 6: Code

### Node Name

`Evaluate Validation Rules`

### Purpose

Checks whether the row has enough data to continue.

### How to Add It

1. Click the `+` after `Normalize Task Data`.
2. Search for `Code`.
3. Add the node.
4. Rename it to `Evaluate Validation Rules`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for Each Item` |

Paste this code:

```javascript
const item = $input.item;
const data = item.json;

const errors = [];

if (!data.project_name) errors.push('Project Name is required');
if (!data.task_name) errors.push('Task Name is required');
if (!['Interior', 'Exterior'].includes(data.task_type)) {
  errors.push('Task Type must be Interior or Exterior');
}
if (!data.start_date) errors.push('Start Date is required');
if (data.task_type === 'Exterior' && !data.site_zip) {
  errors.push('Site Zip Code is required for exterior tasks');
}

return {
  json: {
    ...data,
    is_valid: errors.length === 0,
    validation_error: errors.join('; '),
  },
};
```

### Connect To

`Is Task Valid?`

---

### Node 7: IF

### Node Name

`Is Task Valid?`

### Purpose

Stops invalid rows before any weather or calendar logic runs.

### How to Add It

1. Click the `+` after `Evaluate Validation Rules`.
2. Search for `IF`.
3. Add the node.
4. Rename it to `Is Task Valid?`.

### Exact Settings

| Value 1 | Operation | Value 2 |
|---|---|---|
| `{{$json.is_valid}}` | `is true` |  |

### Branch Meaning

- `true` = valid row
- `false` = invalid row

### Connect To

- `false` output -> `Flag Validation Error`
- `true` output -> `Check Exterior Task`

---

### Node 8: Google Sheets - Update Row

### Node Name

`Flag Validation Error`

### Purpose

Writes `Validation Error` back to the sheet so office staff can fix the row.

### How to Add It

1. Click the `+` from the `false` output of `Is Task Valid?`.
2. Search for `Google Sheets`.
3. Choose the update-row operation.
4. Rename it to `Flag Validation Error`.

### Exact Settings

| Setting | Value |
|---|---|
| `Spreadsheet` | `Apex Ridge - Project Schedule` |
| `Sheet` | `Project Tasks` |
| `Operation` | Update row |

### Fields to Write Back

- `Status` = `Validation Error`
- `Last Synced At` = `{{$now}}`
- `Last Error` = `{{$json.validation_error}}`

Use row number metadata or `Row ID` as the lookup field.

### Connect To

`Notify Admin Validation Failure`

---

### Node 9: Slack or Gmail

### Node Name

`Notify Admin Validation Failure`

### Purpose

Alerts the admin that a row could not be scheduled.

### How to Add It

1. Click the `+` after `Flag Validation Error`.
2. Add either `Slack` or `Gmail`.
3. Rename it to `Notify Admin Validation Failure`.

### Recommended Message

```text
Validation failure in Apex Ridge scheduling workflow

Project ID: {{$json.project_id}}
Project Name: {{$json.project_name}}
Task Name: {{$json.task_name}}
Error: {{$json.validation_error}}
Row Key: {{$json.source_row_id}}
```

### Branch End

This invalid-row branch ends here.

---

### Node 10: IF

### Node Name

`Check Exterior Task`

### Purpose

Separates interior work from exterior work.

### How to Add It

1. Click the `+` from the `true` output of `Is Task Valid?`.
2. Search for `IF`.
3. Add the node.
4. Rename it to `Check Exterior Task`.

### Exact Settings

| Value 1 | Operation | Value 2 |
|---|---|---|
| `{{$json.task_type}}` | `equals` | `Exterior` |

### Branch Meaning

- `true` = exterior path
- `false` = interior path

### Connect To

- `false` output -> `Set Interior Defaults`
- `true` output -> `Prepare Weather Request`

---

### Node 11A: Edit Fields (Set)

### Node Name

`Set Interior Defaults`

### Purpose

Creates safe weather-related defaults for interior jobs.

### How to Add It

1. Click the `+` from the `false` output of `Check Exterior Task`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Set Interior Defaults`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |

Paste this into `JSON Output`:

```json
{
  "project_id": "{{ $json.project_id }}",
  "project_name": "{{ $json.project_name }}",
  "task_name": "{{ $json.task_name }}",
  "task_type": "{{ $json.task_type }}",
  "start_date": "{{ $json.start_date }}",
  "site_zip": "{{ $json.site_zip }}",
  "duration_days": "{{ $json.duration_days }}",
  "calendar_event_id": "{{ $json.calendar_event_id }}",
  "source_row_id": "{{ $json.source_row_id }}",
  "weather_alert": false,
  "risk_reason": "Not applicable - interior task",
  "forecast_summary": "No weather check required",
  "temperature_summary": "Not applicable",
  "rain_chance": "0",
  "sync_status": "Synced"
}
```

### Connect To

`Build Calendar Payload`

---

### Node 11B: Edit Fields (Set)

### Node Name

`Prepare Weather Request`

### Purpose

Formats the exterior row into the fields needed for the weather lookup.

### How to Add It

1. Click the `+` from the `true` output of `Check Exterior Task`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Prepare Weather Request`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |

Paste this into `JSON Output`:

```json
{
  "project_id": "{{ $json.project_id }}",
  "project_name": "{{ $json.project_name }}",
  "task_name": "{{ $json.task_name }}",
  "task_type": "{{ $json.task_type }}",
  "start_date": "{{ $json.start_date }}",
  "site_zip": "{{ $json.site_zip.replace(/\\s+/g, '') }}",
  "duration_days": "{{ $json.duration_days }}",
  "calendar_event_id": "{{ $json.calendar_event_id }}",
  "source_row_id": "{{ $json.source_row_id }}"
}
```

### Connect To

`Resolve Site Coordinates`

---

### Node 12B: HTTP Request

### Node Name

`Resolve Site Coordinates`

### Purpose

Converts ZIP code to latitude and longitude when the weather service requires coordinates.

### How to Add It

1. Click the `+` after `Prepare Weather Request`.
2. Search for `HTTP Request`.
3. Add the node.
4. Rename it to `Resolve Site Coordinates`.

### Exact Settings

Use your weather provider's geocoding endpoint.

| Setting | Value |
|---|---|
| `Method` | `GET` |
| `Send Query Parameters` | `On` |

Recommended query parameters:

- `zip` = `{{$json.site_zip}}`
- provider API key parameter

### Expected Output

At minimum this node should return:

- `lat`
- `lon`

### Important Note

Make sure your original task fields still exist after this HTTP node runs. If your HTTP node replaces the input JSON completely, add a `Merge` node after the request or use your n8n version's option for including input data with the response.

### Connect To

`Fetch Weather Forecast`

---

### Node 13B: HTTP Request

### Node Name

`Fetch Weather Forecast`

### Purpose

Retrieves the 5-day forecast for the task location.

### How to Add It

1. Click the `+` after `Resolve Site Coordinates`.
2. Search for `HTTP Request`.
3. Add the node.
4. Rename it to `Fetch Weather Forecast`.

### Exact Settings

Use your provider's 5-day forecast endpoint.

| Setting | Value |
|---|---|
| `Method` | `GET` |
| `Send Query Parameters` | `On` |

Recommended query parameters:

- `lat` = latitude from `Resolve Site Coordinates`
- `lon` = longitude from `Resolve Site Coordinates`
- `units` = `imperial`
- provider API key parameter

### Important Note

Most weather APIs return several forecast rows for the same day. The next node will pick the best matching record for the scheduled work date.

Also make sure your original task fields still exist after this HTTP node runs. If the response replaces the input JSON completely, add a `Merge` node before `Detect Weather Risk` or use your n8n version's option for including input data with the response.

### Connect To

`Detect Weather Risk`

---

### Node 14B: Code

### Node Name

`Detect Weather Risk`

### Purpose

Chooses the forecast record for the task date and turns it into a simple weather-risk decision.

### How to Add It

1. Click the `+` after `Fetch Weather Forecast`.
2. Search for `Code`.
3. Add the node.
4. Rename it to `Detect Weather Risk`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for Each Item` |

Paste this code and adjust the field names if your weather provider uses a different response shape:

```javascript
const input = $input.item.json;
const list = input.list ?? input.forecast ?? [];
const targetDate = (input.start_date ?? '').slice(0, 10);

const sameDay = list.filter((entry) => {
  const entryDate = (entry.dt_txt ?? entry.datetime ?? '').slice(0, 10);
  return entryDate === targetDate;
});

if (!sameDay.length) {
  return {
    json: {
      ...input,
      weather_alert: false,
      risk_reason: 'Forecast unavailable for task date',
      forecast_summary: 'Forecast unavailable',
      temperature_summary: 'Unavailable',
      rain_chance: '0',
      sync_status: 'Forecast Unavailable',
    },
  };
}

const scored = sameDay.map((entry) => {
  const weatherText = (
    entry.weather?.[0]?.description ??
    entry.condition ??
    ''
  ).toLowerCase();

  const rainChance = Math.round((entry.pop ?? entry.precipitation_probability ?? 0) * 100);
  const tempMax = entry.main?.temp_max ?? entry.temp_max ?? entry.main?.temp ?? null;
  const tempMin = entry.main?.temp_min ?? entry.temp_min ?? entry.main?.temp ?? null;

  let score = rainChance;
  if (weatherText.includes('thunder')) score += 100;
  if (weatherText.includes('storm')) score += 100;
  if (weatherText.includes('snow')) score += 90;
  if (weatherText.includes('heavy rain')) score += 80;

  return {
    entry,
    weatherText,
    rainChance,
    tempMax,
    tempMin,
    score,
  };
});

scored.sort((a, b) => b.score - a.score);
const chosen = scored[0];

const weatherAlert =
  chosen.weatherText.includes('thunder') ||
  chosen.weatherText.includes('storm') ||
  chosen.weatherText.includes('snow') ||
  chosen.weatherText.includes('heavy rain') ||
  chosen.rainChance >= 60;

return {
  json: {
    ...input,
    weather_alert: weatherAlert,
    risk_reason: weatherAlert ? `Risk detected: ${chosen.weatherText}` : 'No significant weather risk detected',
    forecast_summary: chosen.weatherText || 'Forecast available',
    temperature_summary: chosen.tempMax !== null && chosen.tempMin !== null
      ? `High ${chosen.tempMax}F / Low ${chosen.tempMin}F`
      : 'Temperature unavailable',
    rain_chance: String(chosen.rainChance),
    sync_status: weatherAlert ? 'Weather Risk' : 'Synced',
  },
};
```

### Connect To

`Build Calendar Payload`

---

### Node 15: Edit Fields (Set)

### Node Name

`Build Calendar Payload`

### Purpose

Creates the event title, description, start date, end date, and status fields used by both the create and update nodes.

### How to Add It

1. Connect the output of `Set Interior Defaults` into a new `Edit Fields (Set)` node.
2. Connect the output of `Detect Weather Risk` into the same node.
3. Rename the shared node to `Build Calendar Payload`.

If your n8n version does not like two inbound branches into one node, add a `Merge` node first, then place `Build Calendar Payload` after the merge.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into `JSON Output`:

```json
{
  "project_id": "{{ $json.project_id }}",
  "project_name": "{{ $json.project_name }}",
  "task_name": "{{ $json.task_name }}",
  "task_type": "{{ $json.task_type }}",
  "start_date": "{{ $json.start_date }}",
  "end_date": "{{ DateTime.fromISO($json.start_date).plus({ days: Number($json.duration_days) }).toISODate() }}",
  "site_zip": "{{ $json.site_zip }}",
  "calendar_event_id": "{{ $json.calendar_event_id }}",
  "source_row_id": "{{ $json.source_row_id }}",
  "weather_alert": "{{ $json.weather_alert }}",
  "sync_status": "{{ $json.sync_status }}",
  "calendar_title": "{{ $json.weather_alert === true || $json.weather_alert === 'true' ? '[WEATHER ALERT] ' + $json.project_name + ' - ' + $json.task_name : $json.project_name + ' - ' + $json.task_name }}",
  "calendar_description": "{{ 'Project ID: ' + $json.project_id + '\\nProject Name: ' + $json.project_name + '\\nTask Name: ' + $json.task_name + '\\nTask Type: ' + $json.task_type + '\\nSite Zip: ' + ($json.site_zip || 'N/A') + '\\nForecast Summary: ' + $json.forecast_summary + '\\nTemperature: ' + $json.temperature_summary + '\\nRain Chance: ' + $json.rain_chance + '%' + '\\nRisk Reason: ' + $json.risk_reason + '\\nLast Sync Time: ' + $now }}",
  "calendar_color": "{{ $json.weather_alert === true || $json.weather_alert === 'true' ? '11' : '' }}"
}
```

### Connect To

`Check Existing Calendar Event`

---

### Node 16: IF

### Node Name

`Check Existing Calendar Event`

### Purpose

Chooses whether to create a new event or update an existing one.

### How to Add It

1. Click the `+` after `Build Calendar Payload`.
2. Search for `IF`.
3. Add the node.
4. Rename it to `Check Existing Calendar Event`.

### Exact Settings

| Value 1 | Operation | Value 2 |
|---|---|---|
| `{{$json.calendar_event_id}}` | `is not empty` |  |

### Branch Meaning

- `true` = update existing event
- `false` = create new event

### Connect To

- `false` output -> `Create Calendar Event`
- `true` output -> `Update Existing Event`

---

### Node 17A: Google Calendar - Create Event

### Node Name

`Create Calendar Event`

### Purpose

Creates a new all-day event for the task.

### How to Add It

1. Click the `+` from the `false` output of `Check Existing Calendar Event`.
2. Search for `Google Calendar`.
3. Choose the create-event operation.
4. Rename it to `Create Calendar Event`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Calendar credential |
| `Calendar` | Apex Ridge scheduling calendar |
| `Operation` | Create event |
| `All Day` | `On` |
| `Start` | `{{$json.start_date}}` |
| `End` | `{{$json.end_date}}` |
| `Summary` | `{{$json.calendar_title}}` |
| `Description` | `{{$json.calendar_description}}` |

### Connect To

`Write Back Sync Results`

---

### Node 17B: Google Calendar - Update Event

### Node Name

`Update Existing Event`

### Purpose

Updates the calendar event already tied to the row.

### How to Add It

1. Click the `+` from the `true` output of `Check Existing Calendar Event`.
2. Search for `Google Calendar`.
3. Choose the update-event operation.
4. Rename it to `Update Existing Event`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Calendar credential |
| `Calendar` | Apex Ridge scheduling calendar |
| `Operation` | Update event |
| `Event ID` | `{{$json.calendar_event_id}}` |
| `All Day` | `On` |
| `Start` | `{{$json.start_date}}` |
| `End` | `{{$json.end_date}}` |
| `Summary` | `{{$json.calendar_title}}` |
| `Description` | `{{$json.calendar_description}}` |

### Connect To

`Write Back Sync Results`

---

### Node 18: Google Sheets - Update Row

### Node Name

`Write Back Sync Results`

### Purpose

Writes the scheduling result back to the same source row.

### How to Add It

1. Add a `Google Sheets` update-row node after `Create Calendar Event`.
2. Also connect `Update Existing Event` to the same write-back node if your n8n version supports it.
3. Rename it to `Write Back Sync Results`.

If your version prefers separate nodes, make two identical write-back nodes:

- `Write Back Sync Results - Create`
- `Write Back Sync Results - Update`

### Exact Settings

| Setting | Value |
|---|---|
| `Spreadsheet` | `Apex Ridge - Project Schedule` |
| `Sheet` | `Project Tasks` |
| `Operation` | Update row |

### Fields to Write Back

- `Calendar Event ID` = event ID returned by Google Calendar
- `Last Synced At` = `{{$now}}`
- `Status` = `{{$json.sync_status}}`
- `Risk Status` = `{{$json.weather_alert === true || $json.weather_alert === 'true' ? 'Alert' : 'Clear'}}`
- `Last Error` = blank

### Important Event ID Rule

- For create actions, write the new event ID returned by Google Calendar.
- For update actions, if the node output does not return a new ID, write back the existing `{{$json.calendar_event_id}}`.

### Connect To

After write-back, return to the loop so the next row can be processed.

---

## 6.3 Optional Trigger Variant

If you want to use `Google Sheets Trigger` instead of scheduled polling:

1. Replace `Schedule Trigger - Task Sync` with `Google Sheets Trigger - Project Tasks`.
2. Keep the rest of the workflow the same.
3. Remove `Filter Rows Needing Sync` if the trigger already sends only changed rows.

For production reliability, the scheduled version is still the better first implementation.

---

## 6.4 Create the Separate Error Workflow

Do not skip this workflow. It is what tells the admin when the automation itself fails.

### Error Workflow Canvas Order

`Error Trigger - Project Task Sync`  
`-> Format Failure Context`  
`-> Notify Admin Failure`

### Step 1: Create the Workflow

1. Click `Create Workflow`.
2. Rename it to `Apex Ridge - Scheduling Error Handler`.
3. Save it.

### Step 2: Add Error Trigger

Node name: `Error Trigger - Project Task Sync`

Purpose:

- Starts when the main workflow fails

How to add it:

1. Add `Error Trigger` as the first node.
2. Rename it to `Error Trigger - Project Task Sync`.

### Step 3: Add Set Node

Node name: `Format Failure Context`

Purpose:

- Cleans the failure details before you send them to Slack or email

Recommended fields:

- `workflow_name` = `{{$json.workflow.name}}`
- `execution_id` = `{{$json.execution.id}}`
- `failed_node` = `{{$json.lastNodeExecuted}}`
- `error_message` = `{{$json.error.message}}`
- `failed_at` = `{{$now}}`

### Step 4: Add the Alert Node

Node name: `Notify Admin Failure`

Purpose:

- Sends the actual failure notification

Recommended message:

```text
Apex Ridge scheduling workflow failed

Workflow: {{$json.workflow_name}}
Execution ID: {{$json.execution_id}}
Failed Node: {{$json.failed_node}}
Error Message: {{$json.error_message}}
Time: {{$json.failed_at}}
```

### Step 5: Test It

1. Run the main workflow with a few sample rows.
2. Force one failure on purpose.
3. Confirm the error workflow sends the alert.
4. Activate both workflows only after that test succeeds.

---

## 6.5 Status Rules to Keep Consistent

Use these exact values everywhere:

| Situation | Status |
|---|---|
| Interior task synced successfully | `Synced` |
| Exterior task synced with no risk | `Synced` |
| Exterior task synced with risk | `Weather Risk` |
| Forecast missing for that date | `Forecast Unavailable` |
| Missing or bad input | `Validation Error` |

---

## 6.6 Important Build Notes

### Duplicate Prevention

The single most important field in this design is `Calendar Event ID`.

- Blank `Calendar Event ID` means create a new event
- Existing `Calendar Event ID` means update the existing event

Never skip the write-back node after a successful calendar action.

### Row Identification

When updating Google Sheets:

- use row number metadata if available, or
- use a helper key such as `Row ID`

Do not rely on project-name text matching.

### Forecast Horizon

Most 5-day weather APIs cannot score dates beyond the forecast window.

When that happens:

- still create or update the calendar event
- set the row to `Forecast Unavailable`
- do not mark it as `Weather Risk` unless the API actually returned a risky forecast for that date

### Human Control

This workflow should flag weather risk, not automatically reschedule work. The office team should stay in control of date changes.

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

