# Resource-Aware Booking Automation for Shutter & Sky Productions

## 1. Project Overview

This n8n workflow watches the `Master Shoots Calendar` for new shoot bookings, reads the booking title and description, detects which resources are needed, checks those resources in Airtable, and then either:

1. reserves the resources in a second calendar if everything is available, or
2. alerts the team and marks the booking for manual review if anything is blocked.

This matters because Shutter & Sky Productions uses limited, high-value assets:

- drones
- studio rooms
- lighting kits
- product rigs
- specialty gear

Without automation, someone has to manually read the booking, cross-check Airtable, and remember to reserve the equipment. That is slow and easy to get wrong. This workflow gives you one repeatable process that reduces double-booking risk and keeps damaged gear out of active jobs.

This guide is written for the **n8n browser version** and focuses on the **Airtable-based approach** for maintenance and availability tracking.

---

## 2. Final Workflow Logic

Here is the full logic in plain language:

1. A producer creates a new event in `Master Shoots Calendar`.
2. The `Google Calendar Trigger` starts the workflow.
3. An `Edit Fields (Set)` node normalizes the Google payload into clean fields such as:
   - `event_id`
   - `title`
   - `description`
   - `start_time`
   - `end_time`
   - `organizer`
4. A `Code` node scans the title and description for supported resource keywords such as `Drone`, `Studio A`, and `Lighting Kit`.
5. If no keywords are found, the workflow skips reservation and can optionally notify a manual review queue.
6. If keywords are found, another `Code` node turns the resource array into one item per resource so n8n can look them up one at a time.
7. The `Airtable` node searches the `Maintenance Status` table for each requested resource.
8. A `Code` node groups all Airtable results back together and classifies them into:
   - available resources
   - blocked resources
   - blocked reasons
9. An `IF` node checks whether **all** requested resources are available.
10. If all resources are available:
   - create a reservation event in `Equipment/Space Assets`
11. If one or more resources are blocked:
   - send a Slack conflict alert
   - update the original event title with `NEEDS ATTENTION -`
   - append blocked-resource notes to the original booking description
12. If the workflow fails because of credentials, permissions, or API errors:
   - a separate error workflow sends an admin alert

---

## 3. Tools Needed

Use these tools and nodes:

| Tool / Node | Purpose |
|---|---|
| `n8n` browser version | Build and run the workflow |
| `Google Calendar Trigger` | Detects new shoot bookings |
| `Edit Fields (Set)` | Cleans and normalizes incoming booking data |
| `Code` | Extracts keywords, expands resource items, and classifies availability |
| `Airtable` | Stores maintenance and resource status |
| `IF` | Routes the workflow based on keyword presence and availability |
| `Google Calendar` | Creates reservation events and updates the original booking |
| `Slack` | Sends conflict alerts and admin error alerts |
| `Error Trigger` | Starts the error-handling workflow |
| `Sticky Notes` | Annotates the canvas so it is easy to understand and present |

### Services to prepare before building

Create or confirm these are ready:

- Google Calendar: `Master Shoots Calendar`
- Google Calendar: `Equipment/Space Assets`
- Airtable base: `Operations Assets`
- Airtable table: `Maintenance Status`
- Slack channel: `#ops-alerts`

### Credentials you will need in n8n

- Google Calendar credential
- Airtable credential
- Slack credential

---

## 4. Final Recommended Workflow Architecture

### Primary workflow

Required business logic:

`Google Calendar Trigger (new shoot event)`  
`-> Set Node (normalize event data)`  
`-> Code Node (extract requested resource keywords)`  
`-> Airtable Lookup (maintenance status)`  
`-> IF Node (resource available?)`

Recommended buildable canvas for n8n:

`Trigger - New Shoot Booking`  
`-> Normalize - Event Data`  
`-> Detect - Resource Keywords`  
`-> Route - Resource Keywords Found?`

`True` branch:  
`-> Prepare - Resource Items`  
`-> Lookup - Airtable Status`  
`-> Classify - Availability`  
`-> Route - Availability Check`

`Available` branch:  
`-> Reserve - Assets Calendar`

`Conflict` branch:  
`-> Alert - Slack Conflict`  
`-> Update - Needs Attention Event`

`False` branch from Resource Keywords Found?`:`  
`-> Optional - Review Missing Resources`

### Error workflow

`Error Trigger - Booking Workflow Errors`  
`-> Error - Notify Admin`

### Sticky note labels to add on the canvas

- `1. Trigger and Cleanup`
- `2. Resource Detection`
- `3. Airtable Status Check`
- `4. Reservation Path`
- `5. Conflict Path`
- `6. Error Handling Workflow`

---

## 5. Supported Resource Keywords

This guide detects these exact keywords:

- `Drone`
- `Studio A`
- `Studio B`
- `Product Rig`
- `Lighting Kit`
- `Macro Lens Kit`
- `Turntable Set`

### Important naming rule

Keep resource names identical across all systems:

- Google Calendar descriptions
- Airtable `Resource Name` values
- Slack alerts
- future reports and dashboards

### Easy future expansion

You can add more resources later by editing one array inside the keyword-detection `Code` node. That means the workflow is simple to maintain as Shutter & Sky adds more equipment.

---

## 6. Trigger Setup

Use a `Google Calendar Trigger` node.

### Node name

`Trigger - New Shoot Booking`

### Exact setup steps

1. Drag a `Google Calendar Trigger` node onto the canvas.
2. Rename it to `Trigger - New Shoot Booking`.
3. Select your Google Calendar credential.
4. Choose the calendar `Master Shoots Calendar`.
5. Set the event to `Event Created`.
6. Save the node.

### Exact settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Calendar` | `Master Shoots Calendar` |
| `Event` | `Event Created` |

### Sample event title

```text
Drone Shoot - Acme Product Launch
```

### Sample event description

```text
Need Drone + Studio A + Lighting Kit from 2PM to 5PM
```

### Important beginner note

Use **timed events**, not all-day events. This workflow expects event data in `start.dateTime` and `end.dateTime`. If you use all-day events, Google may send `start.date` instead, which changes the payload shape.

### Example trigger payload

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

Add an `Edit Fields (Set)` node directly after the trigger.

### Node name

`Normalize - Event Data`

### Exact setup steps

1. Drag an `Edit Fields (Set)` node onto the canvas.
2. Connect it after `Trigger - New Shoot Booking`.
3. Rename it to `Normalize - Event Data`.
4. Set `Mode` to `JSON Output`.
5. Turn `Keep Only Set Fields` on.
6. Paste the JSON below into the `JSON Output` field.

### Exact node settings

| Field | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

### Paste this into `JSON Output`

```json
{
  "event_id": "{{ $json.id }}",
  "title": "{{ $json.summary || '' }}",
  "description": "{{ $json.description || '' }}",
  "start_time": "{{ $json.start.dateTime || '' }}",
  "end_time": "{{ $json.end.dateTime || '' }}",
  "organizer": "{{ $json.organizer?.email || $json.creator?.email || '' }}",
  "request_status": "Pending",
  "processed_at": "{{ $now }}"
}
```

### Required expressions

Copy-paste expressions used above:

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

### Why this node is important

This node gives the rest of the workflow a clean, predictable structure. Instead of referencing raw Google fields everywhere, every later node can use:

- `event_id`
- `title`
- `description`
- `start_time`
- `end_time`
- `organizer`

### Example output after this node

```json
{
  "event_id": "4v4f9m4c2r2s7t123abc",
  "title": "Drone Shoot - Acme Product Launch",
  "description": "Need Drone + Studio A + Lighting Kit from 2PM to 5PM",
  "start_time": "2026-04-20T14:00:00-04:00",
  "end_time": "2026-04-20T17:00:00-04:00",
  "organizer": "producer@shutterandsky.com",
  "request_status": "Pending",
  "processed_at": "2026-04-20T10:48:00.000+03:00"
}
```

---

## 8. Code Node Keyword Extraction

Add a `Code` node after `Normalize - Event Data`.

### Node name

`Detect - Resource Keywords`

### Exact settings

| Field | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for Each Item` |

### Paste this code

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

- scans both title and description
- matches case-insensitively
- avoids duplicates
- returns an empty array if nothing matches

### Output example

```json
{
  "requested_resources": ["Drone", "Studio A", "Lighting Kit"],
  "resource_count": 3
}
```

### Add the first routing check

After `Detect - Resource Keywords`, add an `IF` node.

### Node name

`Route - Resource Keywords Found?`

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `Number` |
| `Value 1` | `{{ $json.resource_count }}` |
| `Operation` | `larger` |
| `Value 2` | `0` |

### Result

- `true` branch = continue to Airtable lookup
- `false` branch = no supported resource keywords were found

---

## 9. Airtable Maintenance Base Structure

This project uses **Airtable** as the source of truth for resource availability and maintenance state.

### Base name

`Operations Assets`

### Table name

`Maintenance Status`

### Required columns

| Column Name | Type | Purpose |
|---|---|---|
| `Resource Name` | Single line text | Exact resource keyword |
| `Status` | Single select | Current resource state |
| `Last Checked` | Date | Last maintenance review date |
| `Notes` | Long text | Maintenance notes or reservation notes |
| `Assigned Technician` | Single line text | Owner of the asset |

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
| `Studio B` | `Available` | `2026-04-19` | `Backdrop reset done` | `Marco` |
| `Product Rig` | `Available` | `2026-04-19` | `Clean and packed` | `Nina` |
| `Lighting Kit` | `In Repair` | `2026-04-18` | `Power supply issue` | `Jules` |
| `Macro Lens Kit` | `Available` | `2026-04-19` | `Stored in cage 2` | `Marco` |
| `Turntable Set` | `Reserved` | `2026-04-20` | `Held for studio demo` | `Jules` |

### Important Airtable rules

- Create one row for every supported resource before testing.
- Keep `Resource Name` values identical to the names in the code node.
- If Airtable is missing a resource row, treat it as blocked for safety.

---

## 10. Airtable Lookup Node

Because one booking may request multiple resources, you need one small helper `Code` node before Airtable.

### Helper node

### Node name

`Prepare - Resource Items`

### Exact settings

| Field | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for All Items` |

### Paste this code

```javascript
const booking = $input.first().json;

return booking.requested_resources.map((resource) => ({
  json: {
    resource_name: resource,
  },
}));
```

### What this node does

It converts:

```json
{
  "requested_resources": ["Drone", "Studio A", "Lighting Kit"]
}
```

into:

```json
[
  { "resource_name": "Drone" },
  { "resource_name": "Studio A" },
  { "resource_name": "Lighting Kit" }
]
```

That lets n8n send one item at a time into Airtable, which is the simplest way to check multiple resources.

### Airtable node

Add an `Airtable` node after `Prepare - Resource Items`.

### Node name

`Lookup - Airtable Status`

### Exact setup steps

1. Drag an `Airtable` node onto the canvas.
2. Connect it after `Prepare - Resource Items`.
3. Rename it to `Lookup - Airtable Status`.
4. Choose your Airtable credential.
5. Select resource `Record`.
6. Select operation `List`.
7. Choose the base `Operations Assets`.
8. Choose the table `Maintenance Status`.
9. Turn `Return All` off.
10. Set `Limit` to `1`.
11. Add the option `Filter By Formula`.
12. Paste the formula expression below.

### Exact settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Airtable credential |
| `Resource` | `Record` |
| `Operation` | `List` |
| `Base` or `Base ID` | `Operations Assets` |
| `Table` | `Maintenance Status` |
| `Return All` | `Off` |
| `Limit` | `1` |

### Filter By Formula expression

Switch the field to **Expression** mode and paste:

```text
{{ "{Resource Name}='" + $json.resource_name + "'" }}
```

### What this lookup does

For each resource item, Airtable searches:

```text
{Resource Name}='Drone'
```

or:

```text
{Resource Name}='Studio A'
```

or:

```text
{Resource Name}='Lighting Kit'
```

### Important note about multiple resources

You do **not** need a Loop node here. n8n automatically processes every incoming item. Because `Prepare - Resource Items` creates one item per resource, the Airtable node will run once for each requested resource.

---

## 11. Availability Logic

After the Airtable lookup, add another `Code` node to regroup the results and decide whether the booking is safe.

### Node name

`Classify - Availability`

### Exact settings

| Field | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for All Items` |

### Paste this code

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
      available,
      blocked,
      available_resources: available,
      blocked_resources: blocked,
      blocked_details: blockedDetails,
      all_available: blocked.length === 0,
      blocked_summary: blockedDetails
        .map((item) => `${item.resource} (${item.reason})`)
        .join(', ') || 'None',
    },
  },
];
```

### What the code returns

- `available`
- `blocked`
- `available_resources`
- `blocked_resources`
- `blocked_details`
- `all_available`

### Required example

```json
{
  "available": ["Drone", "Studio A"],
  "blocked": ["Lighting Kit"]
}
```

### Add the final IF node

After `Classify - Availability`, add another `IF` node.

### Node name

`Route - Availability Check`

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `Boolean` |
| `Value 1` | `{{ $json.all_available }}` |
| `Operation` | `is true` |

### True path

All requested resources are available. Continue to the reservation branch.

### False path

One or more resources are:

- `In Repair`
- `Out of Service`
- `Reserved`
- missing from Airtable

### Example interpretation

If Airtable returns:

- `Drone = Available`
- `Studio A = Available`
- `Lighting Kit = In Repair`

Then the node output should look like:

```json
{
  "available": ["Drone", "Studio A"],
  "blocked": ["Lighting Kit"],
  "all_available": false
}
```

---

## 12. Reservation Path

On the `true` output of `Route - Availability Check`, add a `Google Calendar` node.

### Node name

`Reserve - Assets Calendar`

### Exact setup steps

1. Drag a `Google Calendar` node onto the canvas.
2. Connect it to the `true` output of `Route - Availability Check`.
3. Rename it to `Reserve - Assets Calendar`.
4. Choose your Google Calendar credential.
5. Set `Resource` to `Event`.
6. Set `Operation` to `Create`.
7. Choose the calendar `Equipment/Space Assets`.
8. Map start and end time from the original booking.
9. Set the event summary and description exactly as shown below.

### Exact settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Resource` | `Event` |
| `Operation` | `Create` |
| `Calendar` | `Equipment/Space Assets` |
| `Start Time` | `{{ $json.start_time }}` |
| `End Time` | `{{ $json.end_time }}` |
| `Summary` | `{{ 'Reserved: ' + $json.requested_resources.join(' / ') }}` |

### Description expression

Switch the `Description` field to **Expression** mode and paste:

```text
{{ `Linked Booking: ${$json.title}
Original Event ID: ${$json.event_id}
Organizer: ${$json.organizer}
Requested Resources: ${$json.requested_resources.join(', ')}
Request Status: Reserved` }}
```

### Expected reservation event

Title:

```text
Reserved: Drone / Studio A / Lighting Kit
```

Description:

```text
Linked Booking: Drone Shoot - Acme Product Launch
Original Event ID: 4v4f9m4c2r2s7t123abc
Organizer: producer@shutterandsky.com
Requested Resources: Drone, Studio A, Lighting Kit
Request Status: Reserved
```

### Optional success note

Add a sticky note near this node:

```text
Success path: all resources available, reservation created in assets calendar.
```

---

## 13. Conflict Path

On the `false` output of `Route - Availability Check`, add a `Slack` node.

### Node name

`Alert - Slack Conflict`

### Exact settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#ops-alerts` |

### Slack message expression

Switch the message text field to **Expression** mode and paste:

```text
{{ `🚨 Resource Conflict Detected

Booking: ${$json.title}
Requested: ${$json.requested_resources.join(', ')}
Blocked Resource(s): ${$json.blocked_resources.join(', ')}
Reason: ${$json.blocked_details.map(item => `${item.resource} = ${item.reason}`).join(', ')}

Original Event requires manual review.` }}
```

### Example Slack alert

```text
🚨 Resource Conflict Detected

Booking: Drone Shoot - Acme Product Launch
Requested: Drone, Studio A, Lighting Kit
Blocked Resource(s): Lighting Kit
Reason: Lighting Kit = In Repair

Original Event requires manual review.
```

---

## 14. Update Original Calendar Event

After `Alert - Slack Conflict`, add another `Google Calendar` node to update the original shoot booking.

### Node name

`Update - Needs Attention Event`

### Exact settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Google Calendar credential |
| `Resource` | `Event` |
| `Operation` | `Update` |
| `Calendar` | `Master Shoots Calendar` |
| `Event ID` | `{{ $json.event_id }}` |

### Summary expression

Switch `Summary` to **Expression** mode and paste:

```text
{{ $json.title.startsWith('NEEDS ATTENTION - ') ? $json.title : 'NEEDS ATTENTION - ' + $json.title }}
```

### Start expression

Switch `Start` to **Expression** mode and paste:

```text
{{ $json.start_time }}
```

### End expression

Switch `End` to **Expression** mode and paste:

```text
{{ $json.end_time }}
```

### Description expression

Switch `Description` to **Expression** mode and paste:

```text
{{ `${$json.description || ''}

Blocked Resource(s): ${$json.blocked_resources.join(', ')}
Reason: ${$json.blocked_details.map(item => `${item.resource} = ${item.reason}`).join(', ')}` }}
```

### Result

Original calendar title becomes:

```text
NEEDS ATTENTION - Drone Shoot - Acme Product Launch
```

Description will be appended with something like:

```text
Blocked Resource(s): Lighting Kit
Reason: Lighting Kit = In Repair
```

---

## 15. Error Handling

Create a second workflow for errors. This is the cleanest way to catch API failures, permission problems, and broken credentials.

### Error workflow name

`Shutter & Sky - Booking Error Handler`

### Node 1

`Error Trigger - Booking Workflow Errors`

This must be the first node in the error workflow.

### Node 2

`Error - Notify Admin`

Use a `Slack` node.

### Slack error node settings

| Field | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#ops-alerts` |

### Error message expression

Switch the message field to **Expression** mode and paste:

```text
{{ `Workflow Error Detected

Workflow name: ${$json.workflow?.name || 'Unknown Workflow'}
Failed node: ${$json.execution?.lastNodeExecuted || $json.trigger?.error?.node?.name || 'Unknown Node'}
Error details: ${$json.execution?.error?.message || $json.trigger?.error?.message || 'No error message returned'}
Time: ${$now.toISO()}
Execution ID: ${$json.execution?.id || 'Unavailable'}` }}
```

### Connect the error workflow to the main workflow

In the main workflow:

1. Click the three-dot menu in the upper-right corner.
2. Open `Settings`.
3. Find `Error workflow`.
4. Select `Shutter & Sky - Booking Error Handler`.
5. Save the workflow.

### How common problems are handled

| Problem | Handling |
|---|---|
| `Airtable timeout` | Main workflow fails, error workflow sends admin alert |
| `Missing event description` | Set node safely uses empty string |
| `No keyword found` | Workflow follows the no-keyword branch, not the error branch |
| `Google Calendar permission issue` | Main workflow fails, error workflow sends admin alert |

### Optional no-keyword review branch

If you want a manual review step when no resource keywords are detected, add a Slack node on the `false` output of `Route - Resource Keywords Found?`.

Suggested message expression:

```text
{{ `Booking review needed.

No supported resource keywords were detected.
Booking: ${$json.title}
Description: ${$json.description || 'No description provided'}
Event ID: ${$json.event_id}` }}
```

---

## 16. Testing Section

Run these four tests and capture screenshots as you go.

### Test 1 Successful Reservation

#### Calendar title

```text
Drone Shoot - Acme Product Launch
```

#### Calendar description

```text
Need Drone + Studio A from 2PM to 5PM
```

#### Airtable statuses

- `Drone = Available`
- `Studio A = Available`

#### Expected result

- keywords are detected correctly
- Airtable returns both rows
- `all_available = true`
- reservation event is created in `Equipment/Space Assets`
- original booking remains unchanged

### Test 2 Maintenance Conflict

#### Calendar title

```text
Studio Demo - New Lighting Setup
```

#### Calendar description

```text
Need Lighting Kit + Studio A from 10AM to 1PM
```

#### Airtable statuses

- `Lighting Kit = In Repair`
- `Studio A = Available`

#### Expected result

- both resources are detected
- `Lighting Kit` is blocked
- Slack conflict alert is sent
- original title is changed to `NEEDS ATTENTION - Studio Demo - New Lighting Setup`

### Test 3 No Resource Mentioned

#### Calendar title

```text
Client Review Call
```

#### Calendar description

```text
Discuss final shot list and delivery schedule
```

#### Expected result

- `requested_resources = []`
- `resource_count = 0`
- reservation path is skipped
- optional review alert runs only if you added that branch

### Test 4 Airtable Failure

#### Test method

1. Temporarily disconnect or invalidate the Airtable credential in n8n.
2. Create a booking that includes a supported resource.
3. Let the main workflow fail.
4. Confirm the error workflow sends the admin alert.
5. Restore the Airtable credential immediately after the test.

#### Expected result

- main workflow fails
- error workflow runs
- Slack admin alert includes workflow name, failed node, error details, time, and execution ID

---

## 17. Deliverables Section

When you submit this project, provide these deliverables:

- export of the main workflow JSON
- export of the error workflow JSON
- screenshot of the full n8n canvas with at least 6 nodes visible
- screenshot of the `Route - Availability Check` branch logic
- screenshot of the successful reservation inside `Equipment/Space Assets`
- screenshot of the Slack conflict alert
- screenshot of the Airtable `Maintenance Status` table rows
- short video showing one success scenario and one conflict scenario

### Export checklist

- [ ] Main workflow exported
- [ ] Error workflow exported
- [ ] Full canvas screenshot captured
- [ ] Branch logic screenshot captured
- [ ] Assets calendar reservation screenshot captured
- [ ] Slack alert screenshot captured
- [ ] Airtable base screenshot captured
- [ ] Short demo video recorded

### Export steps

For each workflow:

1. Open the workflow in n8n.
2. Click the three-dot menu.
3. Click `Download`.
4. Save the JSON file.

### Recommended filenames

- `shutter-and-sky-booking-automation.json`
- `shutter-and-sky-booking-error-handler.json`

---

## 18. Naming Convention Best Practices

Use short, action-first node names so the execution log is readable and your screenshots look professional.

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

Why this naming style works:

- it makes the workflow easier to present
- it makes failed executions easier to debug
- it helps anyone else understand the flow quickly

---

## 19. Pro Tips

- Use sticky notes above each branch so someone can understand the whole workflow in under a minute.
- Keep resource names identical across Google Calendar, Airtable, and Slack.
- Turn on retries for Airtable, Slack, and Google Calendar nodes if your workspace has occasional API hiccups.
- Keep keyword matching lowercase in the code node even if users type mixed case in calendar descriptions.
- Keep both Google calendars on the same timezone so start and end times stay aligned.
- Add one Airtable row for every supported resource before your first real test.
- Review the `Equipment/Space Assets` calendar weekly and archive old reservations if it becomes cluttered.
- If you rename a resource later, update both the Airtable row and the resource list in `Detect - Resource Keywords`.

---

## 20. Bonus Improvements

Once the base workflow is working, extend it with these upgrades:

### Prevent double-booking by checking the assets calendar first

Before creating the reservation, add a Google Calendar check against `Equipment/Space Assets` for the same time window. If an overlapping reservation already exists for the same resource, route to the conflict path instead of creating a new event.

### Auto-release reservation after the shoot ends

Add a second workflow with a schedule trigger that finds completed reservations and updates related statuses after the shoot ends.

### QR check-in for equipment pickup

Generate a QR code for each resource and scan it when gear is collected so the team has a pickup log.

### Daily utilization dashboard

Write reservation events into Airtable or Google Sheets and build a daily dashboard showing the most-used assets.

### SMS alerts for urgent conflicts

Add Twilio after the conflict branch so high-priority equipment problems trigger a text message to the ops lead.

### Approval flow for premium gear

Add a manager approval branch for premium drones, specialty lens kits, or other expensive equipment before the reservation is created.

---

## Suggested Build Order

If you want to finish this quickly without getting lost, build in this order:

1. Create the Airtable base and rows first.
2. Build the Google Calendar Trigger and Set node.
3. Add the keyword detection code and test it with sample data.
4. Add the resource expansion node and Airtable lookup.
5. Add the availability classification code and IF routing.
6. Build the success branch.
7. Build the conflict branch.
8. Build the error workflow last.
9. Run all four tests.
10. Capture screenshots and export JSON files.

---

## Sample End-to-End Test Data

Use this exact sample booking for your first full run:

| Field | Value |
|---|---|
| `Title` | `Drone Shoot - Acme Product Launch` |
| `Description` | `Need Drone + Studio A + Lighting Kit from 2PM to 5PM` |
| `Start Time` | `2026-04-20T14:00:00-04:00` |
| `End Time` | `2026-04-20T17:00:00-04:00` |
| `Organizer` | `producer@shutterandsky.com` |

Expected logic:

- `Drone` detected
- `Studio A` detected
- `Lighting Kit` detected
- Airtable checks all three
- if `Lighting Kit = In Repair`, conflict path runs
- if all three = `Available`, reservation path runs

---

## Official Reference Links

These official n8n sources were used to align the guide with the current browser version and node naming:

- Google Calendar Trigger node: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/
- Edit Fields (Set) node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
- Code node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/
- Code usage guide: https://docs.n8n.io/code/code-node/
- Airtable node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/
- Slack node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- Google Calendar node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/
- Google Calendar event operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/event-operations/
- IF node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
- Workflow settings: https://docs.n8n.io/workflows/settings/
- Error handling: https://docs.n8n.io/flow-logic/error-handling/
