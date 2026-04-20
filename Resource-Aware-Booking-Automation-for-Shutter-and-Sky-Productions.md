# Resource-Aware Booking Automation for Shutter & Sky Productions

## 1. Project Overview

This workflow watches the `Master Shoots Calendar` for newly created shoot bookings, reads the booking title and description, detects which resources are needed, checks each resource in Airtable, and then either:

1. reserves the resources in a second Google Calendar if everything is usable, or
2. alerts the team and marks the original booking for manual review if anything is blocked.

Why this matters:

- broken gear never gets assigned by mistake
- manually reserved assets can be treated as unavailable
- bookings that need attention are obvious right away
- the ops team gets one repeatable process instead of checking calendars and Airtable by hand

This guide is written for the n8n browser version and uses node names and settings that are practical to build directly in the canvas.

---

## 2. Final Workflow Logic

Here is the end-to-end logic in plain language:

1. A new event is created in the `Master Shoots Calendar`.
2. n8n normalizes the calendar payload into clean fields like title, description, start time, and end time.
3. A `Code` node scans the title and description for known resource keywords such as `Drone`, `Studio A`, and `Lighting Kit`.
4. If no resource keywords are found, the workflow can stop cleanly or send the booking to a manual review queue.
5. If resource keywords are found, n8n expands the resource list into one item per resource.
6. The Airtable node checks the `Maintenance Status` table for each resource name.
7. Another `Code` node groups the Airtable results back together and decides:
   - which resources are available
   - which resources are blocked
   - whether all requested resources are usable
8. If all requested resources are available, n8n creates a reservation event in the `Equipment/Space Assets` calendar for the same time window.
9. If any requested resource is blocked, n8n sends a Slack alert and updates the original booking title with `NEEDS ATTENTION -`.
10. If the workflow itself fails, a separate `Error Trigger` workflow notifies an admin.

---

## 3. Tools Needed

Use these nodes and services:

| Tool | Why it is used |
|---|---|
| `n8n` browser version | Build and run the automation |
| `Google Calendar Trigger` | Starts the workflow when a new shoot booking is created |
| `Edit Fields (Set)` | Normalizes the incoming booking data |
| `Code` | Detects resource keywords, expands items, and classifies availability |
| `Airtable` | Stores maintenance and manual reserved status for assets |
| `IF` | Routes bookings based on keyword detection and availability result |
| `Google Calendar` | Creates the reservation event and updates the original booking |
| `Slack` | Sends conflict and admin alerts |
| `Error Trigger` | Starts the separate error workflow |
| `Sticky Notes` | Labels branches and explains sections of the canvas |

Recommended services to prepare before you build:

- one Google Calendar named `Master Shoots Calendar`
- one Google Calendar named `Equipment/Space Assets`
- one Airtable base named `Operations Assets`
- one Slack channel such as `#ops-alerts`

---

## 4. Final Recommended Workflow Architecture

### Required logical architecture

`Google Calendar Trigger (new shoot event)`  
`-> Set Node (normalize event data)`  
`-> Code Node (extract requested resource keywords)`  
`-> Airtable Lookup (maintenance status)`  
`-> IF Node (resource available?)`

### Practical buildable architecture

For a beginner-friendly build that can handle multiple resources in one booking, use this exact node order:

`Google Calendar Trigger - New Shoot Booking`  
`-> Set - Normalize Event Data`  
`-> Code - Detect Resource Keywords`  
`-> IF - Resource Keywords Found?`

`True branch`  
`-> Code - Expand Requested Resources`  
`-> Airtable - Lookup Resource Status`  
`-> Code - Classify Availability`  
`-> IF - All Resources Available?`

`Available branch`  
`-> Google Calendar - Create Assets Reservation`

`Conflict branch`  
`-> Slack - Resource Conflict Alert`  
`-> Google Calendar - Update Original Event`

`No-keyword branch`  
`-> Optional Slack - Review Missing Resources`

### Separate error workflow

`Error Trigger - Booking Workflow Errors`  
`-> Slack - Notify Admin`

### Suggested sticky note labels

- `Trigger and Cleanup`
- `Resource Detection`
- `Airtable Maintenance Check`
- `Reservation Path`
- `Conflict Path`
- `Error Workflow`

---

## 5. Supported Resource Keywords

This guide detects these exact resource names:

- `Drone`
- `Studio A`
- `Studio B`
- `Product Rig`
- `Lighting Kit`
- `Macro Lens Kit`
- `Turntable Set`

Important rule:

- keep these names identical in Google Calendar descriptions, Airtable `Resource Name` values, Slack messages, and any future dashboards

You can expand the list later by editing one array in the keyword detection code node.

---

## 6. Trigger Setup

Use a `Google Calendar Trigger` node.

### Node Name

`Trigger - New Shoot Booking`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Calendar` | `Master Shoots Calendar` |
| `Event` | `Event Created` |

### Sample booking event

Sample title:

```text
Drone Shoot - Acme Product Launch
```

Sample description:

```text
Need Drone + Studio A + Lighting Kit from 2PM to 5PM
```

### What should come into n8n

The Google Calendar trigger typically provides fields similar to:

```json
{
  "id": "4v4f9m4c2r2s7t123abc",
  "summary": "Drone Shoot - Acme Product Launch",
  "description": "Need Drone + Studio A + Lighting Kit from 2PM to 5PM",
  "start": {
    "dateTime": "2026-04-20T14:00:00-04:00"
  },
  "end": {
    "dateTime": "2026-04-20T17:00:00-04:00"
  },
  "organizer": {
    "email": "producer@shutterandsky.com"
  }
}
```

---

## 7. Set Node Cleanup Logic

Add an `Edit Fields (Set)` node immediately after the trigger.

### Node Name

`Normalize - Event Data`

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this JSON into `JSON Output`:

```json
{
  "event_id": "{{ $json.id }}",
  "title": "{{ $json.summary }}",
  "description": "{{ $json.description || '' }}",
  "start_time": "{{ $json.start.dateTime }}",
  "end_time": "{{ $json.end.dateTime }}",
  "organizer": "{{ $json.organizer?.email || $json.creator?.email || '' }}",
  "request_status": "Pending",
  "processed_at": "{{ $now }}"
}
```

### Required expressions

Use these copy-paste expressions:

```text
{{$json.id}}
```

```text
{{$json.summary}}
```

```text
{{$json.description}}
```

```text
{{$json.start.dateTime}}
```

```text
{{$json.end.dateTime}}
```

```text
{{$now}}
```

### Output after this node

```json
{
  "event_id": "4v4f9m4c2r2s7t123abc",
  "title": "Drone Shoot - Acme Product Launch",
  "description": "Need Drone + Studio A + Lighting Kit from 2PM to 5PM",
  "start_time": "2026-04-20T14:00:00-04:00",
  "end_time": "2026-04-20T17:00:00-04:00",
  "organizer": "producer@shutterandsky.com",
  "request_status": "Pending",
  "processed_at": "2026-04-20T..."
}
```

---

## 8. Code Node Keyword Extraction

Add a `Code` node after `Normalize - Event Data`.

### Node Name

`Detect - Resource Keywords`

### Exact Settings

| Setting | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for Each Item` |

Paste this code:

```javascript
const resourceMap = [
  { label: 'Drone', patterns: ['drone'] },
  { label: 'Studio A', patterns: ['studio a'] },
  { label: 'Studio B', patterns: ['studio b'] },
  { label: 'Product Rig', patterns: ['product rig'] },
  { label: 'Lighting Kit', patterns: ['lighting kit'] },
  { label: 'Macro Lens Kit', patterns: ['macro lens kit'] },
  { label: 'Turntable Set', patterns: ['turntable set'] },
];

const title = ($json.title || '').toLowerCase();
const description = ($json.description || '').toLowerCase();
const combinedText = `${title} ${description}`;

function escapeRegex(text) {
  return text.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

const requestedResources = [];

for (const resource of resourceMap) {
  const matched = resource.patterns.some((pattern) => {
    const safePattern = escapeRegex(pattern);
    const regex = new RegExp(`(^|[^a-z0-9])${safePattern}([^a-z0-9]|$)`, 'i');
    return regex.test(combinedText);
  });

  if (matched && !requestedResources.includes(resource.label)) {
    requestedResources.push(resource.label);
  }
}

return {
  json: {
    ...$json,
    requested_resources: requestedResources,
    resource_count: requestedResources.length,
  },
};
```

### What this code does

- checks both title and description
- matches resource names case-insensitively
- avoids duplicates
- returns an empty array if nothing is found

### Output example

```json
{
  "requested_resources": ["Drone", "Studio A", "Lighting Kit"],
  "resource_count": 3
}
```

### Add a keyword check before Airtable

Add an `IF` node after this code node.

### Node Name

`Route - Resource Keywords Found?`

### Exact Settings

| Setting | Value |
|---|---|
| `Condition Type` | `Number` |
| `Value 1` | `{{ $json.resource_count }}` |
| `Operation` | `larger` |
| `Value 2` | `0` |

Result:

- `true` branch = continue to Airtable lookup
- `false` branch = no resource keywords found

---

## 9. Airtable Maintenance Base Structure

Create this Airtable base before connecting the Airtable node.

### Base name

`Operations Assets`

### Table name

`Maintenance Status`

### Required columns

| Column Name | Type | Purpose |
|---|---|---|
| `Resource Name` | Single line text | Exact resource keyword |
| `Status` | Single select | Current usability of the resource |
| `Last Checked` | Date | Most recent maintenance review |
| `Notes` | Long text | Maintenance notes or reservation notes |
| `Assigned Technician` | Single line text | Responsible technician or ops owner |

### Allowed `Status` values

- `Available`
- `In Repair`
- `Out of Service`
- `Reserved`

### Sample rows

| Resource Name | Status | Last Checked | Notes | Assigned Technician |
|---|---|---|---|---|
| `Drone` | `Available` | `2026-04-19` | `Battery cycle completed` | `Nina` |
| `Studio A` | `Available` | `2026-04-19` | `Ready for use` | `Marco` |
| `Lighting Kit` | `In Repair` | `2026-04-18` | `Power supply issue` | `Jules` |
| `Product Rig` | `Available` | `2026-04-19` | `Clean and packed` | `Nina` |
| `Macro Lens Kit` | `Available` | `2026-04-19` | `Stored in cage 2` | `Marco` |
| `Turntable Set` | `Reserved` | `2026-04-20` | `Held for studio demo` | `Jules` |

### Important setup rule

Create one Airtable row for every supported resource before testing the workflow.

If Airtable is missing a resource row, the workflow should treat that resource as blocked for safety.

---

## 10. Airtable Lookup Node

To check multiple requested resources, add one small helper `Code` node before the Airtable lookup.

### Helper node before Airtable

### Node Name

`Prepare - Resource Items`

### Exact Settings

| Setting | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for All Items` |

Paste this code:

```javascript
const booking = $input.first().json;

return booking.requested_resources.map((resource) => ({
  json: {
    resource_name: resource
  }
}));
```

This turns:

```json
{
  "requested_resources": ["Drone", "Studio A", "Lighting Kit"]
}
```

into three separate items:

```json
[
  { "resource_name": "Drone" },
  { "resource_name": "Studio A" },
  { "resource_name": "Lighting Kit" }
]
```

### Why this matters

n8n processes one input item at a time. By expanding the array into separate items first, the Airtable node effectively loops over each requested resource automatically.

### Airtable node

### Node Name

`Lookup - Airtable Status`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Airtable credential |
| `Operation` | `List data from a table` |
| `Base` | `Operations Assets` |
| `Table` | `Maintenance Status` |
| `Return All` | `Off` |
| `Limit` | `1` |

Under `Options`, add `Filter By Formula` and paste this expression:

```text
{{ "{Resource Name}='" + $json.resource_name + "'" }}
```

### What the lookup is doing

For each requested resource item, the Airtable node searches:

```text
Resource Name = extracted resource
```

Examples:

- if the current item is `Drone`, Airtable searches for `Resource Name = Drone`
- if the current item is `Studio A`, Airtable searches for `Resource Name = Studio A`
- if the current item is `Lighting Kit`, Airtable searches for `Resource Name = Lighting Kit`

### Result you want back from Airtable

Each returned record should contain at least:

- `Resource Name`
- `Status`
- `Notes`

---

## 11. Availability Logic

After the Airtable lookup, add another `Code` node to group the records and decide whether the whole booking is safe to reserve.

### Node Name

`Classify - Availability`

### Exact Settings

| Setting | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for All Items` |

Paste this code:

```javascript
const booking = $('Detect - Resource Keywords').first().json;
const requested = booking.requested_resources || [];

const rows = $input.all().map((item) => {
  const row = item.json.fields ?? item.json;

  return {
    resource: row['Resource Name'] || '',
    status: row.Status || 'Unknown',
    notes: row.Notes || '',
  };
});

const rowMap = new Map(rows.map((row) => [row.resource, row]));
const available = [];
const blocked = [];
const blockedDetails = [];

for (const resource of requested) {
  const match = rowMap.get(resource);

  if (!match) {
    blocked.push(resource);
    blockedDetails.push({
      resource,
      reason: 'Missing from Airtable',
    });
    continue;
  }

  if (match.status === 'Available') {
    available.push(resource);
  } else {
    blocked.push(resource);
    blockedDetails.push({
      resource,
      reason: match.status,
    });
  }
}

return [
  {
    json: {
      ...booking,
      available_resources: available,
      blocked_resources: blocked,
      blocked_details: blockedDetails,
      all_available: blocked.length === 0,
      blocked_summary: blockedDetails.map((item) => `${item.resource} (${item.reason})`).join(', ') || 'None',
    },
  },
];
```

### Example output

```json
{
  "available": ["Drone", "Studio A"],
  "blocked": ["Lighting Kit"]
}
```

### Add the final IF decision

Add an `IF` node after `Classify - Availability`.

### Node Name

`Route - Availability Check`

### Exact Settings

| Setting | Value |
|---|---|
| `Condition Type` | `Boolean` |
| `Value 1` | `{{ $json.all_available }}` |
| `Operation` | `is true` |

Logic:

- `true` branch = all requested resources are `Available`
- `false` branch = at least one requested resource is `In Repair`, `Out of Service`, `Reserved`, or missing from Airtable

### Availability examples

If Airtable says:

- `Drone = Available`
- `Studio A = Available`
- `Lighting Kit = In Repair`

then the workflow should classify:

```json
{
  "available_resources": ["Drone", "Studio A"],
  "blocked_resources": ["Lighting Kit"],
  "all_available": false
}
```

---

## 12. Reservation Path

On the `true` output of `Route - Availability Check`, add a `Google Calendar` node to create the reservation event.

### Node Name

`Reserve - Assets Calendar`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Resource` | `Event` |
| `Operation` | `Create` |
| `Calendar` | `Equipment/Space Assets` |
| `Start` | `{{ $json.start_time }}` |
| `End` | `{{ $json.end_time }}` |
| `Summary` | `{{ 'Reserved: ' + $json.requested_resources.join(' / ') }}` |
| `Description` | Use expression below |

Paste this into the `Description` field:

```text
Linked Booking: {{ $json.title }}
Original Event ID: {{ $json.event_id }}
Organizer: {{ $json.organizer }}
Requested Resources: {{ $json.requested_resources.join(', ') }}
Request Status: Reserved
```

### Expected result

If `Drone`, `Studio A`, and `Lighting Kit` are all available, n8n creates a new event in `Equipment/Space Assets` for the same time as the shoot.

Expected reservation title:

```text
Reserved: Drone / Studio A / Lighting Kit
```

### Optional success logging

If you want, add a sticky note after this node that says:

```text
Reservation successful. Booking did not require manual review.
```

---

## 13. Conflict Path

On the `false` output of `Route - Availability Check`, add a `Slack` node.

### Node Name

`Alert - Slack Conflict`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#ops-alerts` |
| `Text` | Use the exact expression below |

Paste this into the `Text` field:

```text
RESOURCE CONFLICT DETECTED

Booking: {{ $json.title }}
Requested: {{ $json.requested_resources.join(', ') }}
Blocked Resource(s): {{ $json.blocked_resources.join(', ') }}
Reason: {{ $json.blocked_details.map(item => item.resource + ' = ' + item.reason).join(', ') }}

Original Event requires manual review.
```

### Example Slack output

```text
RESOURCE CONFLICT DETECTED

Booking: Drone Shoot - Acme Product Launch
Requested: Drone, Studio A, Lighting Kit
Blocked Resource(s): Lighting Kit
Reason: Lighting Kit = In Repair

Original Event requires manual review.
```

---

## 14. Update Original Calendar Event

After the Slack alert on the conflict path, add another `Google Calendar` node configured to update the original booking.

### Node Name

`Update - Needs Attention Event`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Resource` | `Event` |
| `Operation` | `Update` |
| `Calendar` | `Master Shoots Calendar` |
| `Event ID` | `{{ $json.event_id }}` |
| `Start` | `{{ $json.start_time }}` |
| `End` | `{{ $json.end_time }}` |
| `Summary` | Use the expression below |
| `Description` | Use the expression below |

Use this for `Summary`:

```text
{{ $json.title.startsWith('NEEDS ATTENTION - ') ? $json.title : 'NEEDS ATTENTION - ' + $json.title }}
```

Use this for `Description`:

```text
{{ ($json.description || '') + '\n\nBlocked Resource(s): ' + $json.blocked_resources.join(', ') + '\nReason: ' + $json.blocked_details.map(item => item.resource + ' = ' + item.reason).join(', ') }}
```

### Result

The original booking title becomes:

```text
NEEDS ATTENTION - Drone Shoot - Acme Product Launch
```

And the description gets an appended note such as:

```text
Blocked Resource(s): Lighting Kit
Reason: Lighting Kit = In Repair
```

---

## 15. Error Handling

Build a second workflow for failures.

### Error workflow name

`Shutter & Sky - Booking Error Handler`

### Node 1

`Error Trigger - Booking Workflow Errors`

This node must be the first node in the error workflow.

### Node 2

`Error - Notify Admin`

Use a `Slack` node or `Send Email` node. This guide uses Slack for speed.

### Slack error node settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#ops-alerts` |
| `Text` | Use expression below |

Paste this into `Text`:

```text
Workflow Error Detected

Workflow name: {{ $json.workflow.name || 'Unknown Workflow' }}
Failed node: {{ $json.execution.lastNodeExecuted || 'Unknown Node' }}
Error details: {{ $json.execution.error.message || 'No error message returned' }}
Time: {{ $now.toISO() }}
Execution ID: {{ $json.execution.id || 'Unknown' }}
```

### Link the error workflow

In the main workflow:

1. open workflow `Settings`
2. find `Error workflow`
3. choose `Shutter & Sky - Booking Error Handler`
4. save the workflow

### How this guide handles common problems

| Problem | Handling |
|---|---|
| `Airtable timeout` | Main workflow fails and error workflow alerts admin |
| `Missing event description` | Set node safely passes an empty string |
| `No keyword found` | Normal `false` branch from `Route - Resource Keywords Found?` |
| `Google Calendar permission issue` | Main workflow fails and error workflow alerts admin |

### Optional no-keyword review node

If you want a human to review bookings with no detected resources, add a Slack node on the `false` branch of `Route - Resource Keywords Found?`.

Suggested message:

```text
Booking review needed.

No supported resource keywords were detected.
Booking: {{ $json.title }}
Description: {{ $json.description || 'No description provided' }}
Event ID: {{ $json.event_id }}
```

---

## 16. Testing Section

Run these four tests before you call the workflow complete.

### Test 1 Successful Reservation

Calendar event title:

```text
Drone Shoot - Acme Product Launch
```

Calendar event description:

```text
Need Drone + Studio A from 2PM to 5PM
```

Expected result:

- `Drone` and `Studio A` are found in Airtable
- both are `Available`
- reservation event is created in `Equipment/Space Assets`
- original event remains unchanged

### Test 2 Maintenance Conflict

Calendar event title:

```text
Studio Demo - New Lighting Setup
```

Calendar event description:

```text
Need Lighting Kit + Studio A from 10AM to 1PM
```

Expected result:

- `Lighting Kit` is found
- Airtable status is `In Repair`
- Slack conflict alert is sent
- original title becomes `NEEDS ATTENTION - Studio Demo - New Lighting Setup`

### Test 3 No Resource Mentioned

Calendar event title:

```text
Client Review Call
```

Calendar event description:

```text
Discuss final shot list and delivery schedule
```

Expected result:

- `requested_resources = []`
- `resource_count = 0`
- reservation path is skipped
- optional review alert can be sent if you added that node

### Test 4 Airtable Failure

Test method:

1. temporarily disconnect or invalidate the Airtable credential
2. create a booking that includes a supported resource
3. let the main workflow fail naturally
4. confirm the error workflow posts the admin alert
5. restore the Airtable credential immediately after the test

Expected result:

- main workflow execution fails
- error workflow runs
- admin alert contains workflow name, failed node, error details, time, and execution ID

---

## 17. Deliverables Section

When handing this project in, provide these items:

- exported workflow JSON for the main workflow
- exported workflow JSON for the error workflow
- screenshot of the full n8n canvas with at least 6 nodes visible
- screenshot of the `Route - Availability Check` logic
- screenshot of the successful reservation event in `Equipment/Space Assets`
- screenshot of the Slack conflict alert
- screenshot of the Airtable `Maintenance Status` table rows
- short video showing one success run and one conflict run

### Export steps

For each workflow:

1. open the workflow in n8n
2. click the three-dot menu
3. click `Download`
4. save the file with a clear name

Recommended filenames:

- `shutter-and-sky-booking-automation.json`
- `shutter-and-sky-booking-error-handler.json`

---

## 18. Naming Convention Best Practices

Use clean, action-first node names so the execution log is easy to read.

Recommended names:

- `Trigger - New Shoot Booking`
- `Normalize - Event Data`
- `Detect - Resource Keywords`
- `Route - Resource Keywords Found?`
- `Prepare - Resource Items`
- `Lookup - Airtable Status`
- `Classify - Availability`
- `Route - Availability Check`
- `Reserve - Assets Calendar`
- `Alert - Slack Conflict`
- `Update - Needs Attention Event`
- `Error - Notify Admin`

Why this helps:

- executions are easier to troubleshoot
- screenshots look professional
- teammates can understand the workflow faster

---

## 19. Pro Tips

- Use sticky notes above each branch so someone can understand the workflow in under a minute.
- Keep resource names exactly the same across Airtable and booking descriptions.
- Turn on retries for Airtable, Slack, and Google Calendar nodes if your workspace sees transient API failures.
- Keep keyword matching lowercase in the code node, even if users type mixed case in calendar descriptions.
- Keep both Google calendars on the same timezone to avoid shifted reservation times.
- Review the `Equipment/Space Assets` calendar weekly and archive past reservations if the calendar becomes hard to scan.
- Add one Airtable row for every supported resource before your first test run.
- If you ever rename a resource, update both the Airtable `Resource Name` value and the resource list in `Detect - Resource Keywords`.

---

## 20. Bonus Improvements

Once the core workflow is working, extend it with these upgrades:

### Prevent double-booking by checking the assets calendar first

Before creating the reservation, add a Google Calendar `Get Many` check against `Equipment/Space Assets` for the same time window. If an overlapping event already exists for the same resource, route to the conflict path instead of creating a new reservation.

### Auto-release reservation after the shoot ends

Add a second workflow with a schedule trigger that checks for completed reservations and updates related records or statuses after the end time passes.

### QR check-in for equipment pickup

Create a QR code tied to each resource and let staff scan gear at pickup time to confirm handoff.

### Daily utilization dashboard

Write reservation data to Airtable or Google Sheets and build a simple daily dashboard showing the most-booked resources.

### SMS alerts for urgent conflicts

Add Twilio after the conflict branch to text an ops manager when premium equipment is blocked.

### Approval flow for premium gear

Add an approval branch so bookings involving high-value drone packages or specialty lens kits need a manager approval before reservation creation.

---

## Official Reference Links

These official n8n docs were used to align the workflow to current node names and browser behavior:

- Google Calendar Trigger: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/
- Edit Fields (Set): https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
- Code node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/
- Code usage guide: https://docs.n8n.io/code/code-node/
- Airtable node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/
- Airtable credentials: https://docs.n8n.io/integrations/builtin/credentials/airtable/
- If node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
- Google Calendar node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/
- Referencing previous nodes: https://docs.n8n.io/data/data-mapping/referencing-other-nodes/
- Error handling: https://docs.n8n.io/flow-logic/error-handling/
