# Automated Site Progress Sync for Suburban Slate & Spruce

Prepared for: Suburban Slate & Spruce  
Document type: n8n implementation guide  
Purpose: Step-by-step build instructions for a Google Form to Slack and Google Sheets workflow that captures field progress updates, routes alerts by status, and logs milestones for invoicing

## 1. Project Overview

Suburban Slate & Spruce needs a simple way for field installers to submit site progress without making a phone call or sending scattered text messages.

This workflow uses a Google Form as the field submission tool, a Google Sheets trigger as the intake point inside n8n, Slack for team alerts, and a Google Sheet called `Master Project Log` for permanent tracking.

When an installer submits the form:

1. The new response appears in Google Sheets.
2. n8n picks up the new row.
3. The data is cleaned and standardized.
4. The update is logged into the `Master Project Log`.
5. The workflow routes the alert by status.
6. Slack receives a professional message with the installer name, project name, image link, and next steps.
7. If the workflow fails, a separate error workflow alerts the `Tech Alerts` Slack channel.

---

## 2. Final Workflow Logic

## Main workflow

`Google Sheets Trigger - Field Progress Form Response`  
`-> Edit Fields (Set) - Normalize Progress Report`  
`-> Google Sheets - Append Master Project Log`  
`-> Switch - Route by Status`

Branches:

- `Prep` -> `Slack - Field Ops Prep Alert`
- `Layout` -> `Slack - Project Layout Alert`
- `Grouting` -> `Slack - Project Grouting Alert`
- `Completed` -> `Slack - Billing Completion Alert`

## Error workflow

`Error Trigger - Progress Sync Failures`  
`-> Slack - Tech Alerts Failure Notice`

This gives you an 8-node main workflow plus a separate 2-node error workflow.

---

## 3. Tools Needed

Use these tools in n8n:

| Tool | Why it is used |
|---|---|
| `Google Sheets Trigger` | Starts the workflow when a new Google Form response arrives in the linked sheet |
| `Edit Fields (Set)` | Cleans and standardizes the form data into predictable field names |
| `Google Sheets` | Appends every submission into the `Master Project Log` |
| `Switch` | Routes alerts based on status |
| `Slack` | Sends the formatted team notifications |
| `Error Trigger` | Starts the separate failure-notification workflow |

---

## 4. Before You Open n8n

Create the Google Form first.

Use this exact form name:

`Field Progress Report - Suburban Slate & Spruce`

Add these exact questions:

| Question Type | Question Title | What to write |
|---|---|---|
| `Short answer` | `Installer Name` | Turn on `Required` |
| `Dropdown` | `Project Name` | Add `The Miller Bath`, `Riverside Kitchen`, and every real project name you need |
| `Dropdown` | `Status` | Add `Prep`, `Layout`, `Grouting`, `Completed` |
| `Short answer` | `Site Photo Link` | Turn on `Required` and set response validation to `URL` |
| `Paragraph` | `Notes` | Optional, for field details |

After the form is ready:

1. Open the `Responses` tab in Google Forms.
2. Click the green Google Sheets icon.
3. Create the linked response sheet.
4. Leave the default response tab in place.

Use this spreadsheet name:

`Suburban Slate & Spruce - Field Progress Intake`

The default response tab will usually be:

`Form Responses 1`

---

## 5. Create the Master Log Sheet

Create a second Google Sheet for permanent logging.

Use this spreadsheet name:

`Suburban Slate & Spruce - Master Project Log`

Use this tab name:

`Master Project Log`

Put these exact headers in row 1:

```text
report_id
submission_timestamp
installer_name
project_name
status
status_normalized
site_photo_link
formatted_photo_link
notes
project_channel
next_steps
logged_at
```

---

## 6. Slack Channels You Should Prepare

Create or confirm these Slack channels before building the workflow:

| Purpose | Suggested Slack Channel |
|---|---|
| Prep updates | `#field-ops` |
| The Miller Bath project updates | `#project-miller-bath` |
| Riverside Kitchen project updates | `#project-riverside-kitchen` |
| Completed job alerts for invoicing | `#billing` |
| Workflow failure alerts | `#tech-alerts` |

In n8n, it is safer to use Slack channel IDs instead of channel names.

When you configure the Slack nodes, use your real Slack channel IDs in place of the examples below:

```text
C_FIELD_OPS
C_PROJECT_MILLER_BATH
C_PROJECT_RIVERSIDE_KITCHEN
C_BILLING
C_TECH_ALERTS
```

---

## 7. Main Workflow Build

## Workflow Name

Create a new workflow and rename it to:

`Suburban Slate & Spruce - Site Progress Sync`

Save the workflow before adding credentials.

---

### Node 1: Google Sheets Trigger - Field Progress Form Response

### Purpose

This node starts the workflow every time a new Google Form submission lands in the linked response sheet.

### How to Add It

1. Click `Create Workflow`.
2. Click `Add first step`.
3. Search for `Google Sheets Trigger`.
4. Add the node.
5. Rename it to `Google Sheets Trigger - Field Progress Form Response`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Google Sheets credential |
| `Event` | `Row Added` |
| `Document` | `Suburban Slate & Spruce - Field Progress Intake` |
| `Sheet` | `Form Responses 1` |
| `Limit` | `1` while testing |

### Notes

- This is the correct way to work with Google Forms in n8n because the trigger listens to the response sheet, not directly to the form itself.
- Submit one manual form entry after connecting this node so you can see the sample output fields.

---

### Node 2: Edit Fields (Set) - Normalize Progress Report

### Purpose

This node cleans the raw Google Form response and turns it into one consistent JSON object for every later node.

### How to Add It

1. Click the `+` after the trigger node.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Edit Fields (Set) - Normalize Progress Report`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into the `JSON Output` box:

```json
{
  "report_id": "{{ 'PR-' + $now.toMillis() }}",
  "submission_timestamp": "{{ $json['Timestamp'] }}",
  "installer_name": "{{ ($json['Installer Name'] ?? '').trim() }}",
  "project_name": "{{ ($json['Project Name'] ?? '').trim() }}",
  "status": "{{ ($json['Status'] ?? '').trim() }}",
  "status_normalized": "{{ (($json['Status'] ?? '').trim()).toLowerCase() }}",
  "site_photo_link": "{{ ($json['Site Photo Link'] ?? '').trim() }}",
  "formatted_photo_link": "{{ ($json['Site Photo Link'] ?? '').trim() ? '<' + ($json['Site Photo Link'] ?? '').trim() + '|View Site Photo>' : 'No photo link provided' }}",
  "notes": "{{ ($json['Notes'] ?? '').trim() || 'No additional notes provided.' }}",
  "project_channel": "{{ ({ 'The Miller Bath': 'C_PROJECT_MILLER_BATH', 'Riverside Kitchen': 'C_PROJECT_RIVERSIDE_KITCHEN' })[($json['Project Name'] ?? '').trim()] ?? 'C_PROJECT_GENERAL' }}",
  "next_steps": "{{ ({ prep: 'Confirm prep is complete and stage materials for layout.', layout: 'Review tile layout alignment and confirm install pattern approval.', grouting: 'Verify grout color, cure timing, and final cleanup readiness.', completed: 'Review completion photo, confirm scope sign-off, and prepare invoicing.' })[((($json['Status'] ?? '').trim()).toLowerCase())] ?? 'Review the update and confirm the next field action.' }}",
  "logged_at": "{{ $now.toISO() }}"
}
```

### What This Node Is Doing

- Creates a unique `report_id`
- Keeps the form timestamp
- Standardizes the project and status text
- Converts the photo URL into a Slack-friendly link label
- Maps the project name to the correct Slack project channel
- Builds the `Next Steps` text automatically from the status

### Important Update Rule

Whenever you add a new project, update this part of the JSON:

```text
{ 'The Miller Bath': 'C_PROJECT_MILLER_BATH', 'Riverside Kitchen': 'C_PROJECT_RIVERSIDE_KITCHEN' }
```

Replace the example channel IDs with your real Slack channel IDs.

---

### Node 3: Google Sheets - Append Master Project Log

### Purpose

This node creates the permanent audit trail. Every submission gets written into the master log before Slack routing happens.

### How to Add It

1. Click the `+` after the normalize node.
2. Search for `Google Sheets`.
3. Add the node.
4. Rename it to `Google Sheets - Append Master Project Log`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Google Sheets credential |
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append Row` |
| `Document` | `Suburban Slate & Spruce - Master Project Log` |
| `Sheet` | `Master Project Log` |

Map these fields exactly:

| Column | Value |
|---|---|
| `report_id` | `{{$json.report_id}}` |
| `submission_timestamp` | `{{$json.submission_timestamp}}` |
| `installer_name` | `{{$json.installer_name}}` |
| `project_name` | `{{$json.project_name}}` |
| `status` | `{{$json.status}}` |
| `status_normalized` | `{{$json.status_normalized}}` |
| `site_photo_link` | `{{$json.site_photo_link}}` |
| `formatted_photo_link` | `{{$json.formatted_photo_link}}` |
| `notes` | `{{$json.notes}}` |
| `project_channel` | `{{$json.project_channel}}` |
| `next_steps` | `{{$json.next_steps}}` |
| `logged_at` | `{{$json.logged_at}}` |

### Why This Order Matters

Logging before Slack is the safest pattern because even if Slack fails later, the report still exists in your master record.

---

### Node 4: Switch - Route by Status

### Purpose

This node decides which Slack notification path should run based on the installer's selected status.

### How to Add It

1. Click the `+` after `Google Sheets - Append Master Project Log`.
2. Search for `Switch`.
3. Add the node.
4. Rename it to `Switch - Route by Status`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Mode` | `Rules` |
| `Value to Check` | `{{$json.status_normalized}}` |

Create these rules in this exact order:

| Output | Rule |
|---|---|
| `0` | `prep` |
| `1` | `layout` |
| `2` | `grouting` |
| `3` | `completed` |

### Expression

```text
{{$json.status_normalized}}
```

---

### Node 5: Slack - Field Ops Prep Alert

### Purpose

This node sends prep-stage updates to the field operations channel so the operations team knows the site is being prepared.

### How to Add It

1. Drag from the `prep` output of the switch node.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Field Ops Prep Alert`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `C_FIELD_OPS` |

Paste this into the `Text` field:

```text
*Field Progress Update*
*Project:* {{$json.project_name}}
*Installer:* {{$json.installer_name}}
*Status:* {{$json.status}}
*Photo:* {{$json.formatted_photo_link}}
*Notes:* {{$json.notes}}

*Next Steps:* {{$json.next_steps}}
```

Replace `C_FIELD_OPS` with your real Slack channel ID for `#field-ops`.

---

### Node 6: Slack - Project Layout Alert

### Purpose

This node sends layout-stage updates into the correct project-specific Slack channel.

### How to Add It

1. Drag from the `layout` output of the switch node.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Project Layout Alert`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `{{$json.project_channel}}` |

Paste this into the `Text` field:

```text
*Project Progress Update*
*Project:* {{$json.project_name}}
*Installer:* {{$json.installer_name}}
*Status:* {{$json.status}}
*Photo:* {{$json.formatted_photo_link}}
*Notes:* {{$json.notes}}

*Next Steps:* {{$json.next_steps}}
```

This node will post to the project channel mapped inside the normalize node.

---

### Node 7: Slack - Project Grouting Alert

### Purpose

This node sends grouting-stage updates into the correct project-specific Slack channel so the project manager gets a clear milestone update.

### How to Add It

1. Drag from the `grouting` output of the switch node.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Project Grouting Alert`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `{{$json.project_channel}}` |

Paste this into the `Text` field:

```text
*Project Progress Update*
*Project:* {{$json.project_name}}
*Installer:* {{$json.installer_name}}
*Status:* {{$json.status}}
*Photo:* {{$json.formatted_photo_link}}
*Notes:* {{$json.notes}}

*Next Steps:* {{$json.next_steps}}
```

---

### Node 8: Slack - Billing Completion Alert

### Purpose

This node notifies billing when a project status is marked `Completed`, which gives the office a clean signal to begin invoicing.

### How to Add It

1. Drag from the `completed` output of the switch node.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Billing Completion Alert`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `C_BILLING` |

Paste this into the `Text` field:

```text
*Project Milestone Reached*
*Project:* {{$json.project_name}}
*Installer:* {{$json.installer_name}}
*Status:* {{$json.status}}
*Photo:* {{$json.formatted_photo_link}}
*Notes:* {{$json.notes}}

*Next Steps:* {{$json.next_steps}}
```

Replace `C_BILLING` with your real Slack channel ID for the billing channel.

---

## 8. Error Workflow Build

Create a second workflow and rename it to:

`Suburban Slate & Spruce - Progress Sync Error Handler`

Save it before adding nodes.

---

### Node 1: Error Trigger - Progress Sync Failures

### Purpose

This node runs automatically when the main workflow fails because of Google Sheets, Slack, or any other node error.

### How to Add It

1. Open the error workflow.
2. Click `Add first step`.
3. Search for `Error Trigger`.
4. Add the node.
5. Rename it to `Error Trigger - Progress Sync Failures`.

### What to Write

No extra expression is required.

This node listens for workflow failures automatically.

---

### Node 2: Slack - Tech Alerts Failure Notice

### Purpose

This node sends a failure message to the `Tech Alerts` Slack channel so broken credentials or sheet issues are seen quickly.

### How to Add It

1. Click the `+` after the error trigger.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Tech Alerts Failure Notice`.

### What to Write

Use these settings:

| Setting | Value |
|---|---|
| `Credential` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Post` |
| `Channel` | `C_TECH_ALERTS` |

Paste this into the `Text` field:

```text
*Workflow Failure Alert*
*Workflow:* {{$json.workflow.name}}
*Execution ID:* {{$json.execution.id}}
*Failed Node:* {{$json.lastNodeExecuted}}
*Error Message:* {{$json.error.message}}

*Next Steps:* Open the failed execution in n8n, reconnect Slack or Google Sheets if needed, then retry the test submission.
```

Replace `C_TECH_ALERTS` with your real Slack channel ID.

---

## 9. Canvas Layout

Arrange the main workflow left to right in this order:

`Google Sheets Trigger - Field Progress Form Response`  
`-> Edit Fields (Set) - Normalize Progress Report`  
`-> Google Sheets - Append Master Project Log`  
`-> Switch - Route by Status`

Then place the Slack nodes slightly to the right of the switch:

- Top: `Slack - Field Ops Prep Alert`
- Upper middle: `Slack - Project Layout Alert`
- Lower middle: `Slack - Project Grouting Alert`
- Bottom: `Slack - Billing Completion Alert`

This creates a clean screenshot for the deliverable.

---

## 10. How to Test the Workflow

Use this exact testing process:

1. Activate the main workflow.
2. Activate the error workflow.
3. Open the Google Form.
4. Submit a test entry with:

```text
Installer Name: Daniel Cruz
Project Name: The Miller Bath
Status: Layout
Site Photo Link: https://example.com/site-photo-1.jpg
Notes: Floor tile layout is dry-fitted and ready for PM review.
```

5. Wait for n8n to detect the new Google Sheets row.
6. Open the latest execution in n8n.
7. Confirm the flow passed through:

```text
Google Sheets Trigger
-> Edit Fields (Set)
-> Google Sheets
-> Switch
-> Slack - Project Layout Alert
```

8. Open the `Master Project Log` sheet and confirm a new row was added.
9. Open the project Slack channel and confirm the message format looks clean.

---

## 11. Exact Message Format You Should See in Slack

For a `Layout` update, the Slack message should look like this:

```text
Field Progress Update
Project: The Miller Bath
Installer: Daniel Cruz
Status: Layout
Photo: View Site Photo
Notes: Floor tile layout is dry-fitted and ready for PM review.

Next Steps: Review tile layout alignment and confirm install pattern approval.
```

For a `Completed` update, the message should look like this:

```text
Project Milestone Reached
Project: The Miller Bath
Installer: Daniel Cruz
Status: Completed
Photo: View Site Photo
Notes: Final walk-through photos uploaded.

Next Steps: Review completion photo, confirm scope sign-off, and prepare invoicing.
```

---

## 12. Export and Deliverables

After testing successfully, collect the deliverables in this order.

### n8n Workflow Export

1. Open the main workflow.
2. Click the menu in the top-right corner.
3. Click `Download`.
4. Save the exported JSON file.

Recommended file name:

`Suburban-Slate-and-Spruce-Site-Progress-Sync.json`

### Visual Documentation

Take one screenshot of the full main workflow canvas showing:

- The trigger
- The normalize node
- The log node
- The switch node
- The four Slack branches

### Execution Proof

Take one screenshot of a successful execution showing:

- A green execution path
- The final Slack node executed
- The execution data for the sample form entry

### Test Assets

Include these links in your handoff:

```text
Google Form public link: [paste link here]
Master Project Log Google Sheet link: [paste link here]
```

---

## 13. Success Result

When the build is complete, a field installer can submit one short form in about 30 seconds and the workflow will immediately:

- log the submission into `Master Project Log`
- send the update to the correct Slack destination
- include the installer name and photo link
- tell the receiving team what should happen next
- notify `Tech Alerts` if the workflow itself breaks

That is the exact operating pattern Suburban Slate & Spruce needs for cleaner field-to-office communication and faster billing visibility.
