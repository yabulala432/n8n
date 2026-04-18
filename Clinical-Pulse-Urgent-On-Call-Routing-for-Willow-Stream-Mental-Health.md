# Clinical Pulse: Urgent On-Call Routing for Willow Stream Mental Health

## 1. Project Overview

Clinical Pulse is an internal urgent-routing workflow for Willow Stream Mental Health Hub.

When a therapist needs a fast peer consultation, the workflow:

1. Receives the urgent request from a webhook or form.
2. Normalizes the request into consistent fields.
3. Checks a shared Google Calendar to see which clinicians are currently busy.
4. Uses a Code node to determine who is available right now.
5. Sends a targeted Slack alert if any clinicians are free.
6. Escalates to the Clinical Director if everyone is busy.
7. Logs every routed request in Google Sheets.
8. Uses a separate error workflow to notify admins and log failures.

Why this matters:

- Therapists can request help quickly without broadcasting to everyone.
- Clinicians who are actively in session are not interrupted unnecessarily.
- Escalations happen automatically when nobody is available.
- Every request is recorded for audit and operations review.

This guide uses privacy-safe dummy names and dummy client initials only.

---

## 2. Final Workflow Logic

### Main workflow

An urgent consultation request enters the workflow through either a webhook or a form trigger.

The workflow then:

1. Cleans the incoming request fields.
2. Defines a short schedule-check window starting at the submitted timestamp.
3. Pulls Google Calendar events from a shared team availability calendar for that time window.
4. Compares the returned busy events against a clinician roster.
5. Decides whether at least one clinician is available.

### Available path

If one or more clinicians are available:

- send a targeted Slack alert that mentions only the available clinicians
- log the routed request to Google Sheets

### Escalation path

If no clinicians are available:

- send a high-priority escalation alert to the Clinical Director
- log the escalation to Google Sheets

### Error path

If the main workflow fails:

- trigger a separate error workflow
- send an admin Slack notification
- log the failure to Google Sheets

---

## 3. Tools Needed

Use these in n8n:

| Tool | Why it is used |
|---|---|
| `Webhook` or `Typeform Trigger` | Starts the urgent-routing workflow |
| `Edit Fields (Set)` | Normalizes request fields |
| `Google Calendar` | Reads busy schedule events from a shared calendar |
| `Code` | Determines clinician availability from calendar event data |
| `If` | Splits the workflow into available vs escalation path |
| `Slack` or `Telegram` | Sends the urgent targeted alert |
| `Google Sheets` | Logs routed requests and failures |
| `Error Trigger` | Runs a separate failure workflow |
| `Merge` | Optional if you want to combine branches later |

### Concrete stack used in this guide

To keep the build specific and directly buildable, this guide uses:

- `Webhook` as the main trigger
- `Google Calendar` with a shared availability calendar
- `Slack` for urgent clinician alerts
- `Google Sheets` for request and failure logs
- `Error Trigger` in a separate workflow

### Trigger note

`Typeform Trigger` is included as the formal second trigger option because it has an official n8n trigger node.

If your practice prefers `Tally`, configure Tally to submit the same field names to the n8n webhook URL. That gives you the same downstream workflow without changing the rest of the build.

---

## 4. Final Recommended Workflow Architecture

### Primary Workflow

`Webhook / Form Trigger`  
`-> Set - Normalize Urgent Request`  
`-> Google Calendar - Get Clinician Schedule Data`  
`-> Code - Determine Availability`  
`-> IF - Any Clinician Available?`

### Available Path

`-> Slack - Targeted Available Clinician Alert`  
`-> Google Sheets - Log Available Route`

### Escalation Path

`-> Slack - Clinical Director Escalation`  
`-> Google Sheets - Log Escalation Route`

### Error Workflow

`Error Trigger - Clinical Pulse Failures`  
`-> Slack - Admin Failure Alert`  
`-> Google Sheets - Log Failure`

### Recommended workflow names

- Primary workflow: `Clinical Pulse - Urgent Routing`
- Error workflow: `Clinical Pulse - Error Handler`

---

## 5. Intake Form Fields

Create these exact fields:

| Field Label | Final n8n Field | Required | Example Dummy Value |
|---|---|---|---|
| `Requesting Therapist` | `requesting_therapist` | Yes | `Dr. Lane` |
| `Urgency Level` | `urgency_level` | Yes | `High` |
| `Client Initials` | `client_initials` | Yes | `J.S.` |
| `Consult Topic` | `consult_topic` | Yes | `Risk escalation concern` |
| `Notes` | `notes` | Yes | `Need immediate peer input after session pause.` |
| `Preferred Response Method` | `preferred_response_method` | Yes | `Slack` |
| `Timestamp` | `timestamp` | Yes | `2026-04-18T10:30:00Z` |

### Important privacy rule

Do not collect:

- full client names
- birth dates
- diagnoses
- chart numbers
- insurance data

Only use dummy initials in tests.

---

## 6. Trigger Setup (Two Options)

### Option A: Webhook Trigger

Use a POST request with a JSON body.

### Sample payload

```json
{
  "requesting_therapist": "Dr. Lane",
  "urgency_level": "High",
  "client_initials": "J.S.",
  "consult_topic": "Risk escalation concern",
  "notes": "Need immediate peer input after session pause.",
  "preferred_response_method": "Slack",
  "timestamp": "2026-04-18T10:30:00Z"
}
```

### Option B: Typeform Trigger

Use a Typeform with question labels matching the field labels above.

Recommended question labels:

- `Requesting Therapist`
- `Urgency Level`
- `Client Initials`
- `Consult Topic`
- `Notes`
- `Preferred Response Method`
- `Timestamp`

### Tally note

If you use Tally instead of Typeform, point Tally's webhook or form action at the same n8n webhook URL and use the same field keys as the JSON payload above.

---

## 7. Prerequisites Before Building

Set up these two data sources first.

### A. Shared Google Calendar for availability

Create a shared Google Calendar called:

```text
Clinical Pulse - Team Availability
```

This calendar should contain events only for clinicians who are busy and should not be interrupted.

Use this event-title convention:

```text
BUSY | Dr. Avery
BUSY | Dr. Kim
BUSY | Dr. Shah
BUSY | Dr. Perez
```

This guide's Code node checks event titles for those clinician names.

### B. Google Sheets log file

Create a spreadsheet named:

```text
Clinical Pulse Logs
```

Create two sheets:

1. `Request Log`
2. `Failure Log`

Use these headers in `Request Log` row 1:

```text
request_id
requesting_therapist
urgency_level
client_initials
consult_topic
notes
preferred_response_method
timestamp
window_start
window_end
available_clinicians
busy_clinicians
route_status
escalation_target
logged_at
```

Use these headers in `Failure Log` row 1:

```text
failure_timestamp
workflow_name
execution_id
execution_url
last_node_executed
error_message
```

---

## 8. Exact Workflow Build (Step by Step)

Build the main workflow first.

### Node 1: Webhook - Clinical Pulse Intake

### Node Name

`Webhook - Clinical Pulse Intake`

### Purpose

Accepts urgent consultation requests from an internal tool, form, or secure middleware.

### How to Add It

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Webhook`.
4. Add the node.
5. Rename it to `Webhook - Clinical Pulse Intake`.

### Exact Settings

| Setting | Value |
|---|---|
| `HTTP Method` | `POST` |
| `Path` | `clinical-pulse-urgent` |
| `Authentication` | `None` while testing, then secure it in production |
| `Respond` | `Immediately` |
| `Response Code` | `200` |

### Expressions

None.

### Expected Output

The payload arrives under `$json.body`.

Example:

```json
{
  "body": {
    "requesting_therapist": "Dr. Lane",
    "urgency_level": "High",
    "client_initials": "J.S.",
    "consult_topic": "Risk escalation concern",
    "notes": "Need immediate peer input after session pause.",
    "preferred_response_method": "Slack",
    "timestamp": "2026-04-18T10:30:00Z"
  }
}
```

---

### Node 2: Set - Normalize Urgent Request

### Node Name

`Set - Normalize Urgent Request`

### Purpose

Maps webhook or form data into one stable JSON structure and defines the schedule-check window.

### How to Add It

1. Click the `+` after `Webhook - Clinical Pulse Intake`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Set - Normalize Urgent Request`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into the `JSON Output` field:

```json
{
  "request_id": "{{ 'CP-' + $now.toMillis() }}",
  "requesting_therapist": "{{ ($json.body?.requesting_therapist ?? $json['Requesting Therapist'] ?? $json.requesting_therapist ?? '').trim() }}",
  "urgency_level": "{{ ($json.body?.urgency_level ?? $json['Urgency Level'] ?? $json.urgency_level ?? 'High').trim() }}",
  "client_initials": "{{ ($json.body?.client_initials ?? $json['Client Initials'] ?? $json.client_initials ?? '').trim() }}",
  "consult_topic": "{{ ($json.body?.consult_topic ?? $json['Consult Topic'] ?? $json.consult_topic ?? '').trim() }}",
  "notes": "{{ ($json.body?.notes ?? $json['Notes'] ?? $json.notes ?? '').trim() }}",
  "preferred_response_method": "{{ ($json.body?.preferred_response_method ?? $json['Preferred Response Method'] ?? $json.preferred_response_method ?? 'Slack').trim() }}",
  "timestamp": "{{ ($json.body?.timestamp ?? $json['Timestamp'] ?? $json.timestamp ?? $now.toISO()).trim() }}",
  "window_start": "{{ ($json.body?.timestamp ?? $json['Timestamp'] ?? $json.timestamp ?? $now.toISO()).trim() }}",
  "window_end": "{{ new Date(new Date($json.body?.timestamp ?? $json['Timestamp'] ?? $json.timestamp ?? $now.toISO()).getTime() + 30 * 60 * 1000).toISOString() }}",
  "logged_at": "{{ $now.toISO() }}"
}
```

### Expressions

Key expressions used in this node:

```text
{{ 'CP-' + $now.toMillis() }}
```

```text
{{ new Date(new Date($json.body?.timestamp ?? $json['Timestamp'] ?? $json.timestamp ?? $now.toISO()).getTime() + 30 * 60 * 1000).toISOString() }}
```

### Expected Output

```json
{
  "request_id": "CP-1776489600000",
  "requesting_therapist": "Dr. Lane",
  "urgency_level": "High",
  "client_initials": "J.S.",
  "consult_topic": "Risk escalation concern",
  "notes": "Need immediate peer input after session pause.",
  "preferred_response_method": "Slack",
  "timestamp": "2026-04-18T10:30:00Z",
  "window_start": "2026-04-18T10:30:00Z",
  "window_end": "2026-04-18T11:00:00.000Z",
  "logged_at": "2026-04-18T..."
}
```

---

### Node 3: Google Calendar - Get Clinician Schedule Data

### Node Name

`Google Calendar - Get Clinician Schedule Data`

### Purpose

Fetches all busy events from the shared availability calendar during the consultation window.

### How to Add It

1. Click the `+` after `Set - Normalize Urgent Request`.
2. Search for `Google Calendar`.
3. Add the `Google Calendar` node.
4. Rename it to `Google Calendar - Get Clinician Schedule Data`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Resource` | `Event` |
| `Operation` | `Get Many` |
| `Calendar` | `Clinical Pulse - Team Availability` |
| `Return All` | `On` |
| `After` | `{{ $('Set - Normalize Urgent Request').first().json.window_start }}` |
| `Before` | `{{ $('Set - Normalize Urgent Request').first().json.window_end }}` |

Recommended options:

| Option | Value |
|---|---|
| `Recurring Event Handling` | `All Occurrences` |
| `Show Deleted` | `Off` |
| `Timezone` | Your practice timezone |

### Expressions

Use these exact expressions:

```text
{{ $('Set - Normalize Urgent Request').first().json.window_start }}
```

```text
{{ $('Set - Normalize Urgent Request').first().json.window_end }}
```

### Expected Output

One item per event found in the schedule window.

Example:

```json
{
  "summary": "BUSY | Dr. Avery",
  "start": {
    "dateTime": "2026-04-18T10:15:00Z"
  },
  "end": {
    "dateTime": "2026-04-18T11:00:00Z"
  }
}
```

If no events exist in the window, the node may return zero items. The Code node below handles that safely.

---

### Node 4: Code - Determine Availability

### Node Name

`Code - Determine Availability`

### Purpose

Compares busy calendar events to a fixed clinician roster and outputs who is available right now.

### How to Add It

1. Click the `+` after `Google Calendar - Get Clinician Schedule Data`.
2. Search for `Code`.
3. Add the `Code` node.
4. Rename it to `Code - Determine Availability`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for All Items` |
| `Language` | `JavaScript` |

Paste this into the code editor:

```javascript
const request = $('Set - Normalize Urgent Request').first().json;
const events = $input.all().map(item => item.json);

const roster = [
  { name: 'Dr. Avery', summaryToken: 'Dr. Avery', slackMention: '<@U_AVERY>' },
  { name: 'Dr. Kim', summaryToken: 'Dr. Kim', slackMention: '<@U_KIM>' },
  { name: 'Dr. Shah', summaryToken: 'Dr. Shah', slackMention: '<@U_SHAH>' },
  { name: 'Dr. Perez', summaryToken: 'Dr. Perez', slackMention: '<@U_PEREZ>' }
];

const windowStart = new Date(request.window_start);
const windowEnd = new Date(request.window_end);

function eventOverlapsWindow(event) {
  const startValue = event.start?.dateTime || event.start?.date || request.window_start;
  const endValue = event.end?.dateTime || event.end?.date || request.window_end;
  const start = new Date(startValue);
  const end = new Date(endValue);
  return start < windowEnd && end > windowStart;
}

const busyNames = new Set();

for (const event of events) {
  if (!eventOverlapsWindow(event)) continue;

  const searchableText = `${event.summary || ''} ${event.description || ''}`.toLowerCase();

  for (const clinician of roster) {
    if (searchableText.includes(clinician.summaryToken.toLowerCase())) {
      busyNames.add(clinician.name);
    }
  }
}

const availableClinicians = roster.filter(clinician => !busyNames.has(clinician.name));
const busyClinicians = roster.filter(clinician => busyNames.has(clinician.name));

return [
  {
    json: {
      ...request,
      checked_clinicians: roster.map(clinician => clinician.name),
      busy_clinicians: busyClinicians.map(clinician => clinician.name),
      available_clinicians: availableClinicians.map(clinician => clinician.name),
      available_slack_mentions: availableClinicians.map(clinician => clinician.slackMention).join(' '),
      available_count: availableClinicians.length,
      route_status: availableClinicians.length > 0 ? 'available_route' : 'director_escalation',
      escalation_target: availableClinicians.length > 0 ? '' : 'Clinical Director'
    }
  }
];
```

### Expressions

None in the node UI. The logic is inside the code block.

### Expected Output

```json
{
  "request_id": "CP-1776489600000",
  "requesting_therapist": "Dr. Lane",
  "urgency_level": "High",
  "client_initials": "J.S.",
  "consult_topic": "Risk escalation concern",
  "notes": "Need immediate peer input after session pause.",
  "preferred_response_method": "Slack",
  "timestamp": "2026-04-18T10:30:00Z",
  "window_start": "2026-04-18T10:30:00Z",
  "window_end": "2026-04-18T11:00:00.000Z",
  "checked_clinicians": ["Dr. Avery", "Dr. Kim", "Dr. Shah", "Dr. Perez"],
  "busy_clinicians": ["Dr. Avery", "Dr. Kim"],
  "available_clinicians": ["Dr. Shah", "Dr. Perez"],
  "available_slack_mentions": "<@U_SHAH> <@U_PEREZ>",
  "available_count": 2,
  "route_status": "available_route",
  "escalation_target": ""
}
```

### Important customization

Replace these placeholder Slack user IDs:

- `<@U_AVERY>`
- `<@U_KIM>`
- `<@U_SHAH>`
- `<@U_PEREZ>`

with real Slack user mentions from your workspace.

---

### Node 5: IF - Any Clinician Available?

### Node Name

`IF - Any Clinician Available?`

### Purpose

Splits the workflow into available vs escalation path.

### How to Add It

1. Click the `+` after `Code - Determine Availability`.
2. Search for `If`.
3. Add the `If` node.
4. Rename it to `IF - Any Clinician Available?`.

### Exact Settings

Create one condition:

| Setting | Value |
|---|---|
| `Data Type` | `Number` |
| `Operation` | `is greater than` |
| `Value 1` | `{{ $json.available_count }}` |
| `Value 2` | `0` |

### Expressions

```text
{{ $json.available_count }}
```

### Expected Output

- `true` output -> at least one clinician is available
- `false` output -> no clinicians are available, escalate

---

### Node 6: Slack - Targeted Available Clinician Alert

### Node Name

`Slack - Targeted Available Clinician Alert`

### Purpose

Sends a targeted alert that mentions only the clinicians who are currently available.

### How to Add It

1. Drag from the `true` output of `IF - Any Clinician Available?`.
2. Search for `Slack`.
3. Add the `Slack` node.
4. Rename it to `Slack - Targeted Available Clinician Alert`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send a message` |
| `Channel` | `#clinical-pulse-urgent` |
| `Text` | Use the expression below |

### Expressions

Paste this into the `Text` field:

```text
{{ $json.available_slack_mentions }}

CLINICAL PULSE URGENT CONSULT
Request ID: {{ $json.request_id }}
Requesting Therapist: {{ $json.requesting_therapist }}
Urgency Level: {{ $json.urgency_level }}
Client Initials: {{ $json.client_initials }}
Consult Topic: {{ $json.consult_topic }}
Notes: {{ $json.notes }}
Preferred Response Method: {{ $json.preferred_response_method }}
Available Clinicians: {{ $json.available_clinicians.join(', ') }}
Request Timestamp: {{ $json.timestamp }}
```

### Expected Output

A Slack alert posts in `#clinical-pulse-urgent` and mentions only the available clinicians.

---

### Node 7: Google Sheets - Log Available Route

### Node Name

`Google Sheets - Log Available Route`

### Purpose

Logs successful targeted routing events.

### How to Add It

1. Click the `+` after `Slack - Targeted Available Clinician Alert`.
2. Search for `Google Sheets`.
3. Add the `Google Sheets` node.
4. Rename it to `Google Sheets - Log Available Route`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Sheets credential |
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append Row` |
| `Document` | `Clinical Pulse Logs` |
| `Sheet` | `Request Log` |

Map these columns:

| Column | Value |
|---|---|
| `request_id` | `{{ $json.request_id }}` |
| `requesting_therapist` | `{{ $json.requesting_therapist }}` |
| `urgency_level` | `{{ $json.urgency_level }}` |
| `client_initials` | `{{ $json.client_initials }}` |
| `consult_topic` | `{{ $json.consult_topic }}` |
| `notes` | `{{ $json.notes }}` |
| `preferred_response_method` | `{{ $json.preferred_response_method }}` |
| `timestamp` | `{{ $json.timestamp }}` |
| `window_start` | `{{ $json.window_start }}` |
| `window_end` | `{{ $json.window_end }}` |
| `available_clinicians` | `{{ $json.available_clinicians.join(', ') }}` |
| `busy_clinicians` | `{{ $json.busy_clinicians.join(', ') }}` |
| `route_status` | `{{ $json.route_status }}` |
| `escalation_target` | `{{ $json.escalation_target }}` |
| `logged_at` | `{{ $json.logged_at }}` |

### Expressions

```text
{{ $json.available_clinicians.join(', ') }}
```

```text
{{ $json.busy_clinicians.join(', ') }}
```

### Expected Output

A new row appears in `Clinical Pulse Logs -> Request Log`.

---

### Node 8: Slack - Clinical Director Escalation

### Node Name

`Slack - Clinical Director Escalation`

### Purpose

Escalates the request when no clinician is currently available.

### How to Add It

1. Drag from the `false` output of `IF - Any Clinician Available?`.
2. Add a `Slack` node.
3. Rename it to `Slack - Clinical Director Escalation`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send a message` |
| `Channel` | `#clinical-director-escalations` |
| `Text` | Use the expression below |

### Expressions

```text
CLINICAL PULSE ESCALATION
No available clinicians found during the current check window.

Request ID: {{ $json.request_id }}
Requesting Therapist: {{ $json.requesting_therapist }}
Urgency Level: {{ $json.urgency_level }}
Client Initials: {{ $json.client_initials }}
Consult Topic: {{ $json.consult_topic }}
Notes: {{ $json.notes }}
Busy Clinicians: {{ $json.busy_clinicians.length ? $json.busy_clinicians.join(', ') : 'None returned from calendar' }}
Escalation Target: Clinical Director
Request Timestamp: {{ $json.timestamp }}
```

### Expected Output

A high-priority escalation alert posts in `#clinical-director-escalations`.

---

### Node 9: Google Sheets - Log Escalation Route

### Node Name

`Google Sheets - Log Escalation Route`

### Purpose

Logs escalated requests when everyone is busy.

### How to Add It

1. Click the `+` after `Slack - Clinical Director Escalation`.
2. Add a `Google Sheets` node.
3. Rename it to `Google Sheets - Log Escalation Route`.

### Exact Settings

Use the same document and sheet as Node 7:

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Sheets credential |
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append Row` |
| `Document` | `Clinical Pulse Logs` |
| `Sheet` | `Request Log` |

Map the same fields as Node 7.

### Expressions

Use the same mapping expressions as Node 7.

### Expected Output

A new row appears in `Request Log` with `route_status` equal to `director_escalation`.

---

### Optional Node 10: Merge - Rejoin Results

This node is optional. Add it only if you want both branches to continue into one shared reporting or post-processing step.

### Node Name

`Merge - Rejoin Results`

### Purpose

Combines the available and escalation branches.

### How to Add It

1. Search for `Merge`.
2. Connect both Google Sheets log nodes into it.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Append` |

### Expressions

None.

### Expected Output

Both branches become one output stream.

---

## 9. Trigger Alternative Build

If you use Typeform instead of a webhook, replace Node 1 with the node below.

### Node: Typeform Trigger - Clinical Pulse Intake

### Node Name

`Typeform Trigger - Clinical Pulse Intake`

### Purpose

Starts the workflow when a Typeform response is submitted.

### How to Add It

1. Create a second version of the workflow or replace the webhook node.
2. Search for `Typeform Trigger`.
3. Add the node.
4. Rename it to `Typeform Trigger - Clinical Pulse Intake`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Typeform credential |
| `Form` | Your Clinical Pulse intake form |

### Expressions

None in the trigger node itself.

### Expected Output

The Typeform response data is passed into the workflow. Keep `Set - Normalize Urgent Request` exactly as written, because it already supports either form labels or direct webhook keys.

---

## 10. Notification Text Templates

### Available clinician Slack alert

```text
{{ $json.available_slack_mentions }}

CLINICAL PULSE URGENT CONSULT
Request ID: {{ $json.request_id }}
Requesting Therapist: {{ $json.requesting_therapist }}
Urgency Level: {{ $json.urgency_level }}
Client Initials: {{ $json.client_initials }}
Consult Topic: {{ $json.consult_topic }}
Notes: {{ $json.notes }}
Preferred Response Method: {{ $json.preferred_response_method }}
Available Clinicians: {{ $json.available_clinicians.join(', ') }}
Request Timestamp: {{ $json.timestamp }}
```

### Escalation Slack alert

```text
CLINICAL PULSE ESCALATION
No available clinicians found during the current check window.

Request ID: {{ $json.request_id }}
Requesting Therapist: {{ $json.requesting_therapist }}
Urgency Level: {{ $json.urgency_level }}
Client Initials: {{ $json.client_initials }}
Consult Topic: {{ $json.consult_topic }}
Notes: {{ $json.notes }}
Busy Clinicians: {{ $json.busy_clinicians.length ? $json.busy_clinicians.join(', ') : 'None returned from calendar' }}
Escalation Target: Clinical Director
Request Timestamp: {{ $json.timestamp }}
```

### Telegram alternative

If you prefer Telegram, replace both Slack nodes with `Telegram -> Message -> Send Message` and reuse the same message text.

---

## 11. Separate Error Workflow Build

Create this as a second workflow.

### Node 1: Error Trigger - Clinical Pulse Failures

### Node Name

`Error Trigger - Clinical Pulse Failures`

### Purpose

Starts automatically when the main workflow fails during an automatic execution.

### How to Add It

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Error Trigger`.
4. Add the node.
5. Rename it to `Error Trigger - Clinical Pulse Failures`.

### Exact Settings

No extra parameters are required.

### Expressions

None.

### Expected Output

The node provides workflow failure details such as workflow name, execution ID, execution URL, last node executed, and error message.

---

### Node 2: Slack - Admin Failure Alert

### Node Name

`Slack - Admin Failure Alert`

### Purpose

Sends an immediate private admin alert when the workflow fails.

### How to Add It

1. Click the `+` after `Error Trigger - Clinical Pulse Failures`.
2. Search for `Slack`.
3. Add the `Slack` node.
4. Rename it to `Slack - Admin Failure Alert`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send a message` |
| `Channel` | `#clinical-pulse-admin` |
| `Text` | Use the expression below |

### Expressions

```text
CLINICAL PULSE WORKFLOW FAILURE
Workflow: {{ $json.workflow.name || 'Unknown Workflow' }}
Execution ID: {{ $json.execution.id || 'Unknown' }}
Execution URL: {{ $json.execution.url || 'Unavailable' }}
Last Node Executed: {{ $json.execution.lastNodeExecuted || 'Unknown' }}
Error Message: {{ $json.execution.error.message || 'No message returned' }}
Failure Timestamp: {{ $now.toISO() }}
```

### Expected Output

An admin Slack alert posts immediately when the primary workflow fails.

---

### Node 3: Google Sheets - Log Failure

### Node Name

`Google Sheets - Log Failure`

### Purpose

Writes the failure details into a permanent failure log.

### How to Add It

1. Click the `+` after `Slack - Admin Failure Alert`.
2. Search for `Google Sheets`.
3. Add the `Google Sheets` node.
4. Rename it to `Google Sheets - Log Failure`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Sheets credential |
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append Row` |
| `Document` | `Clinical Pulse Logs` |
| `Sheet` | `Failure Log` |

Map these columns:

| Column | Value |
|---|---|
| `failure_timestamp` | `{{ $now.toISO() }}` |
| `workflow_name` | `{{ $json.workflow.name || 'Unknown Workflow' }}` |
| `execution_id` | `{{ $json.execution.id || 'Unknown' }}` |
| `execution_url` | `{{ $json.execution.url || 'Unavailable' }}` |
| `last_node_executed` | `{{ $json.execution.lastNodeExecuted || 'Unknown' }}` |
| `error_message` | `{{ $json.execution.error.message || 'No message returned' }}` |

### Expressions

```text
{{ $json.execution.error.message || 'No message returned' }}
```

### Expected Output

A new failure row appears in `Clinical Pulse Logs -> Failure Log`.

### Important linking step

After saving the error workflow:

1. Open the primary workflow.
2. Go to workflow `Settings`.
3. Find `Error workflow`.
4. Select `Clinical Pulse - Error Handler`.
5. Save the primary workflow.

Without this step, the separate error workflow will not run.

---

## 12. Testing Scenarios

Test each routing outcome on purpose.

### Test 1: Two clinicians available

Calendar window contains:

- `BUSY | Dr. Avery`
- `BUSY | Dr. Kim`

Request payload:

```json
{
  "requesting_therapist": "Dr. Lane",
  "urgency_level": "High",
  "client_initials": "J.S.",
  "consult_topic": "Risk escalation concern",
  "notes": "Need immediate peer input after session pause.",
  "preferred_response_method": "Slack",
  "timestamp": "2026-04-18T10:30:00Z"
}
```

Expected result:

- `available_clinicians` = `Dr. Shah, Dr. Perez`
- Slack targeted alert sends to those clinicians
- request is logged in `Request Log`

### Test 2: One clinician available

Calendar window contains:

- `BUSY | Dr. Avery`
- `BUSY | Dr. Kim`
- `BUSY | Dr. Perez`

Expected result:

- `available_clinicians` = `Dr. Shah`
- available route is used
- request is logged

### Test 3: No clinicians available

Calendar window contains:

- `BUSY | Dr. Avery`
- `BUSY | Dr. Kim`
- `BUSY | Dr. Shah`
- `BUSY | Dr. Perez`

Expected result:

- `available_count` = `0`
- escalation Slack alert is sent to the Clinical Director channel
- request is logged with `route_status = director_escalation`

### Test 4: No busy events returned

Calendar window contains no events.

Expected result:

- all clinicians are treated as available
- targeted Slack alert includes the full roster
- request is logged

### Test 5: Error workflow

To test the error workflow:

1. Activate the primary workflow.
2. Temporarily break the Google Sheets credential or select an invalid sheet.
3. Send a real webhook request to the production webhook URL.
4. Confirm the primary workflow fails.
5. Confirm the error workflow sends the admin Slack alert.
6. Confirm the failure is logged to `Failure Log`.
7. Restore the broken setting immediately after the test.

---

## 13. Export Checklist

When the build is finished, export both workflows and capture proof of operation.

### Export the primary workflow

1. Open `Clinical Pulse - Urgent Routing`.
2. Click the three-dot menu in the top-right corner.
3. Select `Download`.
4. Save the file as:

```text
clinical-pulse-urgent-routing.json
```

### Export the error workflow

1. Open `Clinical Pulse - Error Handler`.
2. Click the three-dot menu.
3. Select `Download`.
4. Save the file as:

```text
clinical-pulse-error-handler.json
```

### Screenshot checklist

Capture screenshots of:

- primary workflow canvas
- error workflow canvas
- webhook node configuration
- Set node JSON mapping
- Google Calendar `Get Many` settings
- Code node roster and availability logic
- IF node condition
- Slack available alert node
- Slack escalation node
- Google Sheets request log node
- Google Sheets failure log node
- one available-route execution
- one escalation execution
- one error-workflow execution

### Evidence checklist

- Slack targeted alert message
- Slack escalation message
- `Request Log` rows
- `Failure Log` row
- downloaded `.json` exports

---

## 14. Practical Build Notes

- Keep the shared availability calendar clean. Only put interrupt-blocking events on it.
- Use one consistent naming rule in calendar event titles, for example `BUSY | Dr. Avery`.
- Update the Code node roster whenever your clinician pool changes.
- Replace placeholder Slack mentions with real workspace user IDs before activating the workflow.
- Set `Retry On Fail` on Slack and Google Sheets nodes if transient API issues are common.
- Secure the webhook before production use.
- If you need channel selection later, add a second `If` or `Switch` based on `preferred_response_method`.
- If you prefer Telegram, swap the Slack nodes and keep the same routing logic.
- Do not include PHI in alerts. Keep the workflow limited to internal consult metadata and dummy initials.

---

## 15. Final Output Example

After the `Code - Determine Availability` node, the item should look like this:

```json
{
  "request_id": "CP-1776489600000",
  "requesting_therapist": "Dr. Lane",
  "urgency_level": "High",
  "client_initials": "J.S.",
  "consult_topic": "Risk escalation concern",
  "notes": "Need immediate peer input after session pause.",
  "preferred_response_method": "Slack",
  "timestamp": "2026-04-18T10:30:00Z",
  "window_start": "2026-04-18T10:30:00Z",
  "window_end": "2026-04-18T11:00:00.000Z",
  "checked_clinicians": ["Dr. Avery", "Dr. Kim", "Dr. Shah", "Dr. Perez"],
  "busy_clinicians": ["Dr. Avery", "Dr. Kim"],
  "available_clinicians": ["Dr. Shah", "Dr. Perez"],
  "available_slack_mentions": "<@U_SHAH> <@U_PEREZ>",
  "available_count": 2,
  "route_status": "available_route",
  "escalation_target": "",
  "logged_at": "2026-04-18T..."
}
```

This is the final normalized data contract that both routing paths and the request log use.

---

## Official Reference Links

These official n8n docs were used to align this guide with the current browser UI and node names:

- Webhook: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- Google Calendar: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/
- Google Calendar Event operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/event-operations/
- Code node: https://docs.n8n.io/code/code-node/
- If node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
- Slack node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- Telegram message operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/message-operations/
- Google Sheets: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- Typeform Trigger: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.typeformtrigger/
- Error Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.errortrigger/
- Merge: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/
- Workflow export/import: https://docs.n8n.io/workflows/export-import/
