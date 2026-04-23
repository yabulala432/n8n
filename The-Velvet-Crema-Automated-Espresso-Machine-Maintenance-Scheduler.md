# The Velvet Crema: Automated Espresso Machine Maintenance Scheduler

Prepared for: The Velvet Crema  
Document type: n8n implementation guide  
Purpose: Step-by-step execution plan for building a Google Sheets to Google Calendar preventive-maintenance workflow based on espresso machine shot-count thresholds

## 1. Executive Summary

The Velvet Crema depends on espresso machines that run at high volume every day. When preventive maintenance is scheduled too late, the business risks machine downtime, slower service, inconsistent drink quality, and expensive repairs. When maintenance is scheduled too early, the shop loses productive machine hours and technician time.

This automation solves that problem by scheduling service based on actual machine usage instead of arbitrary calendar intervals.

Operationally, the workflow works like this:

1. A daily shot-count row is added to Google Sheets.
2. n8n reads the machine's current running total since the last maintenance.
3. n8n adds the new shot count to that total.
4. If the machine is still below the threshold, the sheet is updated with the remaining shots until service is needed.
5. If the machine reaches or exceeds 1,000 shots, n8n finds the next available maintenance window on Google Calendar.
6. n8n books a `Deep Clean & Calibration` appointment and notifies the Team Lead in Slack.

For a coffee bar environment, this creates one clean operating pattern:

- usage is logged once in Google Sheets
- service thresholds are checked automatically
- maintenance is scheduled consistently
- the team gets notified without manual follow-up
- duplicate bookings are prevented by tracking service state in the sheet

---

## 2. Full Architecture Overview

## Core Systems

| System | Role in the solution |
|---|---|
| `n8n` | Orchestration layer that reads shot logs, calculates threshold progress, finds a maintenance window, schedules service, updates the sheet, and sends notifications |
| `Google Sheets` | Source system for daily shot logs and current machine service state |
| `Google Calendar` | Stores available maintenance windows and receives the scheduled maintenance appointment |
| `Slack` | Sends a confirmation message to the Team Lead when maintenance is scheduled |

## Recommended Workflow Set

Build two workflows:

1. `The Velvet Crema - PM Scheduler`
2. `The Velvet Crema - PM Error Handler`

The first workflow handles the business logic.  
The second workflow catches failures such as no available calendar slot or a failed Google Calendar action.

## End-to-End Data Flow

`Google Sheets Row Added`  
`-> n8n Trigger`  
`-> Normalize New Shot Log Row`  
`-> Read Machine Service State`  
`-> Calculate Updated Cumulative Total`

If threshold is not reached:

`-> Update Machine Status in Google Sheets with Remaining Shots`

If threshold is reached:

`-> Search Google Calendar for Maintenance Window`  
`-> Book Deep Clean & Calibration Appointment`  
`-> Mark Service Scheduled in Google Sheets`  
`-> Send Slack Confirmation`

Failure path:

`Calendar slot not found or event creation fails`  
`-> Error Trigger Workflow`  
`-> Slack or Gmail Failure Notification`

## Logical Architecture Diagram

- Source layer
  - Google Sheets stores the daily `Shot Count` log.
- State layer
  - A second Google Sheets tab stores current per-machine service state.
- Decision layer
  - n8n calculates the new running total and decides whether service is due.
- Scheduling layer
  - n8n searches the Google Calendar for the next usable maintenance window.
- Action layer
  - n8n creates or updates the maintenance appointment.
- Notification layer
  - n8n sends the Team Lead a Slack message with the machine ID and scheduled date.
- Exception layer
  - A separate error workflow alerts the admin when scheduling fails.

## Recommended Business Rule Summary

| Rule | Recommendation |
|---|---|
| Service threshold | Schedule maintenance at `1,000` cumulative shots since the last maintenance |
| Source of truth for logs | `Daily Shot Log` tab in Google Sheets |
| Source of truth for service state | `Machine Status` tab in Google Sheets |
| Duplicate prevention | Use `Service Scheduled` plus `Calendar Event ID` in the status tab |
| Reset logic | Do not reset the tally when the appointment is merely scheduled; reset only when maintenance is actually completed |
| Calendar search method | Search for the next placeholder event or approved maintenance block named `Maintenance Window` |
| Failure visibility | Route Google Calendar failures to a separate error workflow |

---

## 3. Suggested Time Breakdown

Use the following estimates for a realistic tracked-hours implementation plan.

| Task | Hours |
|---|---:|
| Discovery and workflow planning | 0.75 |
| Google Sheets design and sample data setup | 0.75 |
| Credential setup in n8n | 0.5 |
| Main workflow scaffold and node layout | 0.75 |
| Trigger and row normalization | 0.75 |
| Machine status lookup and threshold logic | 1.5 |
| Calendar search and appointment booking | 1.5 |
| Slack notification | 0.5 |
| Sheet write-back and duplicate prevention | 1.0 |
| Error workflow and failure alerting | 0.75 |
| Testing and proof-of-work capture | 1.25 |
| Documentation and handoff notes | 0.5 |
| **Total Estimated Effort** | **9.0** |

## Recommended Delivery Phases

| Phase | Focus | Hours |
|---|---|---:|
| Phase 0 | Sheet design and credentials | 2.0 |
| Phase 1 | Main workflow scaffold | 1.5 |
| Phase 2 | Threshold calculation logic | 2.0 |
| Phase 3 | Calendar scheduling and Slack alerting | 2.0 |
| Phase 4 | Error handling and QA | 1.5 |
| **Total** |  | **9.0** |

---

## 4. Pre-Build Preparation Checklist

Complete this checklist before building the workflow.

## Access Checklist

- [ ] n8n access with permission to create workflows and credentials
- [ ] Google account access for the shot log spreadsheet
- [ ] Google account access for the maintenance calendar
- [ ] Permission to create or update calendar events
- [ ] Slack workspace access for the Team Lead notification channel

## Technical Inputs Checklist

- [ ] Confirm the exact Google Sheet name
- [ ] Confirm the exact Google Calendar used for maintenance windows
- [ ] Confirm the naming pattern for placeholder events such as `Maintenance Window`
- [ ] Confirm the Team Lead's Slack channel or user destination
- [ ] Confirm whether the machine tally resets immediately after scheduling or only after maintenance completion

## Data Preparation Checklist

- [ ] Create a `Daily Shot Log` tab
- [ ] Create a `Machine Status` tab
- [ ] Add at least two machine IDs for testing, such as `Machine A` and `Machine B`
- [ ] Add at least one machine already near the 1,000-shot threshold for threshold testing
- [ ] Add at least one future placeholder event in Google Calendar named `Maintenance Window`

## Operational Decisions to Confirm

- [ ] Should the workflow schedule the earliest available window or the earliest window at least one business day ahead
- [ ] Should the workflow stop when no window is found, or should it notify the Team Lead and leave the row unscheduled
- [ ] Who is responsible for marking maintenance as completed and resetting the tally
- [ ] Should the calendar title include both the machine ID and the maintenance type

---

## 5. Google Sheets Design

Use one spreadsheet with two tabs. This gives you both the raw usage log and a stable service-state record.

Recommended spreadsheet name: `The Velvet Crema - Machine Maintenance`

## Tab 1: `Daily Shot Log`

This tab records the incoming daily usage rows that trigger the workflow.

### Required Columns

| Column | Required | Purpose |
|---|---|---|
| `Log Date` | Yes | Date of the shot-count entry |
| `Machine ID` | Yes | Identifies which espresso machine the log belongs to |
| `Shot Count` | Yes | Number of shots pulled since the last log row |
| `Operator` | No | Optional staff or shift label |
| `Notes` | No | Optional comments from the shift |

### Sample Rows

| Log Date | Machine ID | Shot Count | Operator | Notes |
|---|---|---:|---|---|
| `2026-04-23` | `Machine A` | 180 | `Morning Shift` |  |
| `2026-04-23` | `Machine B` | 220 | `Evening Shift` | Heavy rush |

## Tab 2: `Machine Status`

This tab stores the current maintenance state for each machine and is the key to preventing duplicate bookings.

### Required Columns

| Column | Required | Purpose |
|---|---|---|
| `Machine ID` | Yes | Unique machine identifier |
| `Shots Since Last Service` | Yes | Running total since last completed maintenance |
| `Threshold` | Yes | Service threshold; recommended default `1000` |
| `Shots Remaining` | Yes | Remaining shots before maintenance is due |
| `Current Status` | Yes | Human-readable current state |
| `Service Scheduled` | Yes | Prevents duplicate bookings |
| `Scheduled Service Date` | No | Date or datetime of the scheduled maintenance |
| `Calendar Event ID` | No | ID of the scheduled calendar event |
| `Last Maintenance Date` | No | Date of the last completed service |
| `Last Updated At` | No | Latest sync timestamp |

### Why the Status Tab Matters

The daily log tab tells you what happened today.  
The status tab tells you what the machine's current maintenance state is right now.

Without the status tab, every new row would force n8n to re-sum historical logs or risk duplicate bookings. With the status tab:

- the workflow reads one machine row
- adds the new shot count
- decides whether maintenance is due
- updates the same row again

That is the cleanest approach for a threshold-based maintenance workflow.

### Sample Rows

| Machine ID | Shots Since Last Service | Threshold | Shots Remaining | Current Status | Service Scheduled | Scheduled Service Date | Calendar Event ID | Last Maintenance Date | Last Updated At |
|---|---:|---:|---:|---|---|---|---|---|---|
| `Machine A` | 640 | 1000 | 360 | `360 shots remaining until service` | `No` |  |  | `2026-04-16` | `2026-04-23 08:00:00` |
| `Machine B` | 800 | 1000 | 200 | `200 shots remaining until service` | `No` |  |  | `2026-04-16` | `2026-04-23 08:00:00` |

---

## 6. Step-by-Step n8n Build Plan

This section is written as a true build guide so someone can open n8n and assemble the workflow node by node without guessing the sequence.

## Workflow Names

- Main workflow: `The Velvet Crema - PM Scheduler`
- Error workflow: `The Velvet Crema - PM Error Handler`

## Recommended Build Strategy

Use a Google Sheets row-added trigger for the main workflow because the business requirement is to react when a new daily shot-count row is logged.

For maintainability, keep the threshold state in the `Machine Status` tab rather than recalculating the full machine history on every run.

## Main Workflow Canvas Order

Build the main workflow in this order:

`Google Sheets Trigger - New Shot Log Row`  
`-> Normalize Shot Log Row`  
`-> Read Machine Status`  
`-> Calculate Maintenance Need`  
`-> Should Schedule Maintenance?`

If threshold is not reached:

`-> Update Remaining Shots`

If threshold is reached:

`-> Find Maintenance Window`  
`-> Create Deep Clean Appointment`  
`-> Mark Service Scheduled`  
`-> Notify Team Lead Scheduled`

If you want the daily log row to be marked as processed as well, add one more Google Sheets update node after the status update. The leaner version below keeps the service state in `Machine Status` and uses that as the duplicate-prevention source of truth.

---

## 6.1 Create the Main Workflow

1. Log in to n8n.
2. Click `Create Workflow`.
3. Rename it to `The Velvet Crema - PM Scheduler`.
4. Click `Save`.

---

## 6.2 Build Each Node

### Node 1: Google Sheets Trigger

### Node Name

`Google Sheets Trigger - New Shot Log Row`

### Purpose

Starts the workflow when a new daily shot-count row is added.

### How to Add It

1. Click `Add first step`.
2. Search for `Google Sheets Trigger`.
3. Add the node.
4. Rename it to `Google Sheets Trigger - New Shot Log Row`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Sheets credential |
| `Spreadsheet` | `The Velvet Crema - Machine Maintenance` |
| `Sheet` | `Daily Shot Log` |
| `Trigger Event` | New row added |

### Connect To

`Normalize Shot Log Row`

---

### Node 2: Edit Fields (Set)

### Node Name

`Normalize Shot Log Row`

### Purpose

Cleans the incoming row and gives every later node a stable field structure.

### How to Add It

1. Click the `+` after `Google Sheets Trigger - New Shot Log Row`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Normalize Shot Log Row`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into `JSON Output`:

```json
{
  "log_date": "{{ ($json['Log Date'] ?? $now.toISODate()).toString().trim() }}",
  "machine_id": "{{ ($json['Machine ID'] ?? '').toString().trim() }}",
  "shots_pulled": "{{ Number($json['Shot Count'] ?? 0) || 0 }}",
  "operator": "{{ ($json['Operator'] ?? '').toString().trim() }}",
  "notes": "{{ ($json['Notes'] ?? '').toString().trim() }}"
}
```

### What This Node Does

- normalizes the machine ID
- converts `Shot Count` into a numeric value
- keeps the row compact and predictable for the rest of the workflow

### Connect To

`Read Machine Status`

---

### Node 3: Google Sheets - Lookup Row

### Node Name

`Read Machine Status`

### Purpose

Fetches the current service-state row for the machine that appears in the new shot log.

### How to Add It

1. Click the `+` after `Normalize Shot Log Row`.
2. Search for `Google Sheets`.
3. Choose the row lookup or search-row operation.
4. Rename it to `Read Machine Status`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Sheets credential |
| `Spreadsheet` | `The Velvet Crema - Machine Maintenance` |
| `Sheet` | `Machine Status` |
| `Operation` | Lookup row / Search rows |
| `Lookup Column` | `Machine ID` |
| `Lookup Value` | `{{$json.machine_id}}` |

### Important Note

Make sure the machine-status row can still be combined with the trigger row data. If your Google Sheets node replaces the input JSON completely, add a `Merge` node before the calculation step or use your n8n version's option for preserving input fields.

### Connect To

`Calculate Maintenance Need`

---

### Node 4: Code

### Node Name

`Calculate Maintenance Need`

### Purpose

Adds the new shot count to the current running total and decides whether maintenance should be scheduled.

### How to Add It

1. Click the `+` after `Read Machine Status`.
2. Search for `Code`.
3. Add the node.
4. Rename it to `Calculate Maintenance Need`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for Each Item` |

Paste this code:

```javascript
const input = $input.item.json;

const priorShots = Number(input['Shots Since Last Service'] ?? input.shots_since_last_service ?? 0);
const threshold = Number(input['Threshold'] ?? input.threshold ?? 1000);
const dailyShots = Number(input.shots_pulled ?? 0);
const serviceScheduledRaw = (input['Service Scheduled'] ?? input.service_scheduled ?? 'No').toString().trim().toLowerCase();
const existingEventId = (input['Calendar Event ID'] ?? input.calendar_event_id ?? '').toString().trim();

const newTotalShots = priorShots + dailyShots;
const shotsRemaining = Math.max(0, threshold - newTotalShots);
const serviceAlreadyScheduled = serviceScheduledRaw === 'yes' || serviceScheduledRaw === 'true';
const shouldSchedule = newTotalShots >= threshold && !serviceAlreadyScheduled && !existingEventId;

return {
  json: {
    ...input,
    prior_shots: priorShots,
    threshold,
    daily_shots: dailyShots,
    new_total_shots: newTotalShots,
    shots_remaining: shotsRemaining,
    service_already_scheduled: serviceAlreadyScheduled,
    should_schedule: shouldSchedule,
    current_status_text: shouldSchedule
      ? `Threshold reached at ${newTotalShots} shots - scheduling maintenance`
      : `${shotsRemaining} shots remaining until service`,
  },
};
```

### Output

This node creates the fields the workflow actually cares about:

- `new_total_shots`
- `shots_remaining`
- `should_schedule`
- `current_status_text`

### Connect To

`Should Schedule Maintenance?`

---

### Node 5: IF

### Node Name

`Should Schedule Maintenance?`

### Purpose

Branches the workflow based on whether the machine has crossed the threshold and is not already scheduled.

### How to Add It

1. Click the `+` after `Calculate Maintenance Need`.
2. Search for `IF`.
3. Add the node.
4. Rename it to `Should Schedule Maintenance?`.

### Exact Settings

| Value 1 | Operation | Value 2 |
|---|---|---|
| `{{$json.should_schedule}}` | `is true` |  |

### Branch Meaning

- `true` = threshold reached, scheduling path
- `false` = threshold not reached or already scheduled

### Connect To

- `false` output -> `Update Remaining Shots`
- `true` output -> `Find Maintenance Window`

---

### Node 6A: Google Sheets - Update Row

### Node Name

`Update Remaining Shots`

### Purpose

Updates the `Machine Status` row when the machine has not yet reached the threshold.

### How to Add It

1. Click the `+` from the `false` output of `Should Schedule Maintenance?`.
2. Search for `Google Sheets`.
3. Choose the update-row operation.
4. Rename it to `Update Remaining Shots`.

### Exact Settings

| Setting | Value |
|---|---|
| `Spreadsheet` | `The Velvet Crema - Machine Maintenance` |
| `Sheet` | `Machine Status` |
| `Operation` | Update row |
| `Lookup Column` | `Machine ID` |
| `Lookup Value` | `{{$json.machine_id}}` |

### Fields to Write Back

- `Shots Since Last Service` = `{{$json.new_total_shots}}`
- `Shots Remaining` = `{{$json.shots_remaining}}`
- `Current Status` = `{{$json.current_status_text}}`
- `Last Updated At` = `{{$now}}`

### Result

This is the branch that satisfies the requirement:

- if threshold is not reached
- update the sheet with the remaining shots until maintenance is needed

---

### Node 6B: Google Calendar - Search Events

### Node Name

`Find Maintenance Window`

### Purpose

Finds the next available placeholder maintenance slot in Google Calendar.

### How to Add It

1. Click the `+` from the `true` output of `Should Schedule Maintenance?`.
2. Search for `Google Calendar`.
3. Choose the event-search or get-many-events operation.
4. Rename it to `Find Maintenance Window`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Calendar credential |
| `Calendar` | Maintenance calendar |
| `Operation` | Search events / Get many events |
| `Query` | `Maintenance Window` |
| `Start After` | `{{$now}}` |

### Important Rule

This node should return the earliest future placeholder event that the business treats as a valid service slot.

If no such slot is found, let the workflow fail so the error workflow can notify the admin.

### Connect To

`Create Deep Clean Appointment`

---

### Node 7B: Google Calendar - Create Event

### Node Name

`Create Deep Clean Appointment`

### Purpose

Creates the actual maintenance appointment on the selected maintenance window.

### How to Add It

1. Click the `+` after `Find Maintenance Window`.
2. Search for `Google Calendar`.
3. Choose the create-event operation.
4. Rename it to `Create Deep Clean Appointment`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential` | Your Google Calendar credential |
| `Calendar` | Maintenance calendar |
| `Operation` | Create event |
| `Title / Summary` | `Machine {{$json.machine_id}} - Deep Clean & Calibration` |
| `Start` | Start time from the maintenance window event |
| `End` | End time from the maintenance window event |

### Recommended Description

Use a description like this:

```text
Machine ID: {{$json.machine_id}}
Maintenance Type: Deep Clean & Calibration
Trigger Count: {{$json.new_total_shots}} shots
Daily Increment: {{$json.daily_shots}} shots
Logged Date: {{$json.log_date}}
Scheduled By: n8n
```

### Important Note

The create node must use the actual slot returned by `Find Maintenance Window`. If your calendar-search node returns several events, sort or limit the search so the earliest valid slot is the one passed forward.

### Connect To

`Mark Service Scheduled`

---

### Node 8B: Google Sheets - Update Row

### Node Name

`Mark Service Scheduled`

### Purpose

Writes the scheduled state back to `Machine Status` so the machine is not booked again on the next shot-log row.

### How to Add It

1. Click the `+` after `Create Deep Clean Appointment`.
2. Search for `Google Sheets`.
3. Choose the update-row operation.
4. Rename it to `Mark Service Scheduled`.

### Exact Settings

| Setting | Value |
|---|---|
| `Spreadsheet` | `The Velvet Crema - Machine Maintenance` |
| `Sheet` | `Machine Status` |
| `Operation` | Update row |
| `Lookup Column` | `Machine ID` |
| `Lookup Value` | `{{$json.machine_id}}` |

### Fields to Write Back

- `Shots Since Last Service` = `{{$json.new_total_shots}}`
- `Shots Remaining` = `0`
- `Current Status` = `Maintenance scheduled`
- `Service Scheduled` = `Yes`
- `Scheduled Service Date` = scheduled date from the created calendar event
- `Calendar Event ID` = event ID returned by Google Calendar
- `Last Updated At` = `{{$now}}`

### Duplicate-Prevention Rule

This node is what stops duplicate bookings.

Future daily rows for the same machine can still update usage, but the workflow should not create another appointment while:

- `Service Scheduled = Yes`, or
- `Calendar Event ID` is already present

### Connect To

`Notify Team Lead Scheduled`

---

### Node 9B: Slack

### Node Name

`Notify Team Lead Scheduled`

### Purpose

Sends the Team Lead a human-readable Slack confirmation with the machine ID and scheduled date.

### How to Add It

1. Click the `+` after `Mark Service Scheduled`.
2. Search for `Slack`.
3. Choose the send-message operation.
4. Rename it to `Notify Team Lead Scheduled`.

### Recommended Slack Message

```text
Machine {{$json.machine_id}} has reached {{$json.new_total_shots}} shots; Deep Clean & Calibration has been scheduled for {{$json.start || $json['Scheduled Service Date']}}.
```

### Better User-Facing Version

If your Slack expression formatting supports it, prefer a message like:

```text
Machine B has reached 1,020 shots; service scheduled for Friday at 2:00 PM.
```

That wording directly matches the acceptance criteria and is easier for operations staff to read.

---

## 6.3 Error Workflow

Do not leave calendar failures silent. Build a second workflow for failures.

### Error Workflow Canvas Order

`Error Trigger - PM Scheduler Failures`  
`-> Notify Admin PM Failure`

### Step 1: Create the Workflow

1. Click `Create Workflow`.
2. Rename it to `The Velvet Crema - PM Error Handler`.
3. Save it.

### Step 2: Add Error Trigger

Node name: `Error Trigger - PM Scheduler Failures`

Purpose:

- starts when the main workflow fails

How to add it:

1. Add `Error Trigger` as the first node.
2. Rename it to `Error Trigger - PM Scheduler Failures`.

### Step 3: Add Notification Node

Node name: `Notify Admin PM Failure`

Purpose:

- notifies the admin that the calendar search or create step failed

Recommended message:

```text
The Velvet Crema PM scheduler failed.

Workflow: {{$json.workflow.name}}
Execution ID: {{$json.execution.id}}
Failed Node: {{$json.lastNodeExecuted}}
Error Message: {{$json.error.message}}
Time: {{$now}}
```

Recommended destination:

- Slack admin channel, or
- Gmail email to the Team Lead or technical admin

---

## 6.4 Service Reset Rule

This project needs one rule to prevent confusion:

Scheduling a service is not the same as completing a service.

Recommended operating rule:

- when the appointment is booked, set `Service Scheduled = Yes`
- do not reset `Shots Since Last Service` yet
- after the technician completes the maintenance, a human updates:
  - `Shots Since Last Service = 0`
  - `Shots Remaining = 1000`
  - `Service Scheduled = No`
  - `Calendar Event ID` = blank
  - `Scheduled Service Date` = blank
  - `Current Status = Ready for next cycle`

This keeps the workflow logically correct and avoids premature resets.

---

## 6.5 Important Build Notes

### About the 4-8 Node Preference

The client's preference mentions a `4-8` node workflow export. A technically accurate build with:

- status lookup
- threshold decision
- sheet update
- calendar search
- calendar create
- sheet write-back
- Slack notification

usually lands at `8-9` business nodes, plus the separate error workflow. If the build must be forced into eight nodes, the simplest compromise is:

- keep the main logic the same
- move the `Current Status` text into a Google Sheets formula instead of updating it from n8n

The guide above favors correctness and operational clarity over artificially reducing the node count.

### Calendar Window Strategy

Use one of these patterns:

1. Create placeholder calendar blocks titled `Maintenance Window`
2. Search the next available placeholder block
3. Create the real maintenance event in that time slot

This is the easiest approach to demo and explain.

### Duplicate Prevention

Never rely on the shot log tab alone to stop duplicate bookings.  
Use these two fields in `Machine Status`:

- `Service Scheduled`
- `Calendar Event ID`

### Missing Machine Row

If a shot-log row arrives for a machine that does not exist in `Machine Status`, the workflow should fail clearly so the admin can create that status row first.

---

## 7. Recommended Node Naming Convention

Use descriptive node names so future debugging is fast.

| Node Type | Recommended Name |
|---|---|
| Google Sheets Trigger | `Google Sheets Trigger - New Shot Log Row` |
| Edit Fields (Set) | `Normalize Shot Log Row` |
| Google Sheets lookup | `Read Machine Status` |
| Code | `Calculate Maintenance Need` |
| IF | `Should Schedule Maintenance?` |
| Google Sheets update | `Update Remaining Shots` |
| Google Calendar search | `Find Maintenance Window` |
| Google Calendar create | `Create Deep Clean Appointment` |
| Google Sheets update | `Mark Service Scheduled` |
| Slack | `Notify Team Lead Scheduled` |
| Error Trigger | `Error Trigger - PM Scheduler Failures` |
| Slack or Gmail failure alert | `Notify Admin PM Failure` |

## Naming Rules

- start with the business action, not just the vendor name
- keep names short enough to read on the canvas
- use consistent verbs such as `Read`, `Calculate`, `Find`, `Create`, `Mark`, `Notify`

---

## 8. Testing Strategy

Run the workflow against controlled test data before activation.

## General Test Setup

- use a dedicated test calendar first
- keep one or two placeholder `Maintenance Window` events available
- use a copy of the Google Sheet for testing if needed
- send Slack notifications to a test channel first

## Case A - Threshold Not Reached

Test setup:

- `Machine A` currently has `640` shots since last service
- new log row adds `180` shots

Expected result:

- new total becomes `820`
- no calendar event is created
- `Shots Remaining = 180`
- `Current Status = 180 shots remaining until service`

## Case B - Threshold Reached

Test setup:

- `Machine B` currently has `800` shots since last service
- new log row adds `220` shots

Expected result:

- new total becomes `1020`
- workflow searches the calendar for the next `Maintenance Window`
- a `Machine B - Deep Clean & Calibration` event is created
- `Service Scheduled = Yes`
- `Calendar Event ID` is written back
- Slack sends a dynamic confirmation message

## Case C - Duplicate Prevention

Test setup:

- `Machine B` already has `Service Scheduled = Yes`
- `Calendar Event ID` is already populated
- another daily shot row is added

Expected result:

- no second maintenance booking is created
- the status row remains protected from duplicate scheduling

## Case D - No Maintenance Window Found

Test setup:

- remove all future placeholder maintenance windows from the calendar
- trigger a row that should schedule maintenance

Expected result:

- the main workflow fails at the calendar search or create step
- the error workflow sends a failure notification

---

## 9. Debugging Checklist

Use this checklist when the workflow does not behave as expected.

| Issue | Likely cause | What to check |
|---|---|---|
| Threshold not triggering | Wrong cumulative math | Confirm `prior_shots + shots_pulled` is being calculated correctly |
| Duplicate appointment created | Service-state fields not updated | Verify `Service Scheduled` and `Calendar Event ID` are written back |
| Slack message missing machine ID | Lost upstream fields | Confirm the machine ID survives through the calendar nodes |
| No slot found | Wrong calendar query or no placeholder events | Check event title and future event availability |
| Wrong machine row updated | Weak lookup key | Always update by `Machine ID`, never by free text status |
| Wrong scheduled date | Calendar search returning multiple rows | Limit or sort the search so only the earliest valid slot is used |

## Practical Debug Sequence

1. Confirm the trigger row contains the correct `Machine ID` and `Shot Count`.
2. Confirm the machine status lookup returns the correct row.
3. Inspect the calculation output for `new_total_shots` and `should_schedule`.
4. If scheduling should happen, inspect the calendar-search result.
5. Inspect the created calendar event data.
6. Confirm the status tab write-back happened after the calendar step.
7. Confirm the Slack message used the dynamic machine and date values.

---

## 10. Deliverables / Proof of Work Plan

Capture the following proof during build and testing.

## Required Deliverables

| Deliverable | What it should include |
|---|---|
| n8n workflow export | The completed JSON export of the main workflow |
| Workflow structure screenshot | Full n8n canvas with all nodes visible |
| Test run evidence | Screenshots or recording showing input row, created calendar event, and Slack message |

## Required Screenshots

| Screenshot | What it should show |
|---|---|
| Google Sheets input | A new `Daily Shot Log` row being used for the test |
| Machine Status before run | A machine near the threshold before the workflow executes |
| Workflow canvas overview | Full node layout and branch structure |
| Threshold calculation output | The node output showing `new_total_shots` and `should_schedule` |
| Calendar search result | The maintenance window found by the calendar node |
| Created event | The `Deep Clean & Calibration` appointment visible in Google Calendar |
| Machine Status after run | `Service Scheduled`, `Scheduled Service Date`, and `Calendar Event ID` populated |
| Slack notification | The Team Lead confirmation message with dynamic machine and date values |
| Failure case | Error Trigger notification from a forced no-slot test |

## Suggested Proof-of-Work Sequence

1. Show the `Machine Status` row before the threshold is crossed.
2. Add the new shot-count row in `Daily Shot Log`.
3. Run the workflow or trigger it through the sheet.
4. Capture the calculation output showing the threshold was exceeded.
5. Capture the maintenance appointment created in Google Calendar.
6. Capture the Slack confirmation message.
7. Capture the updated `Machine Status` row.

---

## 11. Optimization Ideas

Once the first version is stable, consider these upgrades.

| Enhancement | Business value |
|---|---|
| Multi-machine dashboard | Gives managers a quick service-readiness view across all machines |
| Technician assignment field | Stores the preferred service provider or internal technician |
| Maintenance completion form | Lets staff reset the machine status through a controlled workflow |
| Escalation when no slot exists | Alerts leadership if no maintenance window is available within the next two days |
| Daily digest | Sends a summary of all machines and remaining shots each morning |
| Separate thresholds by machine type | Supports different PM thresholds for main bar, decaf, or backup equipment |

---

## 12. Recommended Build Order Summary

If the implementer wants the cleanest execution path, build in this order:

1. Create the `Daily Shot Log` and `Machine Status` tabs.
2. Add sample machine rows to `Machine Status`.
3. Create future placeholder `Maintenance Window` events in Google Calendar.
4. Configure Google Sheets, Google Calendar, and Slack credentials in n8n.
5. Build the trigger, normalization, and machine-status lookup nodes.
6. Add the calculation node and test threshold math.
7. Add the branch for `threshold reached` vs `not reached`.
8. Add the Google Calendar search and create nodes.
9. Add the write-back node that marks service as scheduled.
10. Add the Slack confirmation node.
11. Build the separate error workflow.
12. Run the success and failure test cases before activation.

---

## 13. Final Consultant Recommendation

For a real coffee-bar environment, the strongest first version is:

- Google Sheets trigger on new shot-log rows
- separate `Machine Status` tab for running totals
- 1,000-shot threshold logic in n8n
- calendar-slot search using placeholder `Maintenance Window` events
- Slack confirmation using dynamic machine and date values
- `Service Scheduled` plus `Calendar Event ID` for duplicate prevention

That combination is practical, easy to demo, easy for shift leads to understand, and strong enough for a production pilot without overengineering the first release.
