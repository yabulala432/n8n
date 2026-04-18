# Field Alert Automation for Oak & Iron Custom Cabinetry

## 1. Project Overview

This automation gives Oak & Iron one reliable place to capture and route job-site problems.

Instead of installers sending issue reports by SMS and hoping the right person sees them, this workflow:

1. Receives a field issue report from a webhook or form.
2. Cleans the incoming data into consistent fields.
3. Logs every report into a master tracking table.
4. Routes urgent issues based on severity.
5. Sends the right alert to the right team.
6. Uses a separate error workflow to notify an admin if the automation itself fails.

Why this matters:

- Critical blockers get seen immediately.
- Nothing gets lost in text messages.
- Production staff can review one master log instead of chasing installers.
- Failures in the workflow itself are surfaced instead of silently disappearing.

---

## 2. Final Workflow Logic

### Primary workflow

An installer or form submits a site issue. The workflow normalizes the incoming fields, writes the issue into a master field log, and then routes the report using a `Switch` node based on severity:

- `Critical` -> instant Slack alert
- `High` -> Slack alert plus email escalation
- `Standard` -> log only
- `Low` -> log only

### Error workflow

If the main workflow fails at any step, a separate workflow with an `Error Trigger` runs and emails the admin with the failure details, including workflow name, execution ID, last node executed, and error message.

---

## 3. Tools Needed

Use these tools in n8n:

| Tool | Why it is used |
|---|---|
| `Webhook` trigger or `Google Sheets Trigger` | Starts the workflow from a direct POST request or Google Form response sheet |
| `Edit Fields (Set)` | Cleans and standardizes the incoming fields |
| `Switch` | Routes the report by severity |
| `Google Sheets` or `Airtable` | Stores every issue in the master log |
| `Slack`, `Discord`, or `Telegram` | Sends urgent field alerts |
| `Gmail` or `Send Email` | Sends escalation or admin failure emails |
| `Error Trigger` | Starts a separate workflow when the main workflow fails |
| `Merge` | Optional if you want to rejoin alert branches later |

### Recommended concrete stack for this guide

To keep the build fast and specific, this guide uses:

- `Webhook` as the primary trigger
- `Google Sheets` as the master log
- `Slack` for urgent notifications
- `Gmail` for high-priority escalation emails
- `Send Email` in the separate error workflow

You can swap these later:

- `Airtable` instead of `Google Sheets`
- `Discord` or `Telegram` instead of `Slack`
- `SMTP Send Email` instead of `Gmail`

---

## 4. Final Recommended Workflow Architecture

### Primary Workflow

`Webhook / Google Sheets Trigger`  
`-> Set - Normalize Issue Report`  
`-> Google Sheets - Append Master Log`  
`-> Switch - Route by Severity`

### Branches

`Critical`  
`-> Slack - Critical Field Alert`

`High`  
`-> Slack - High Priority Field Alert`  
`-> Gmail - High Priority Escalation`

`Standard`  
`-> Logged only`

`Low`  
`-> Logged only`

### Separate Error Workflow

`Error Trigger - Main Workflow Failures`  
`-> Send Email - Admin Failure Notice`

### Recommended workflow names

- Primary workflow: `Oak & Iron - Field Alert Intake`
- Error workflow: `Oak & Iron - Field Alert Error Handler`

---

## 5. Exact Workflow Build (Step by Step)

## Before You Start

Create the master log destination first.

### If using Google Sheets

Create a spreadsheet named `Oak & Iron - Master Field Log` with a sheet named `Master Field Log`.

Use these exact column headers in row 1:

```text
report_id
project_name
installer_name
issue_type
severity
severity_normalized
notes
site_address
date_reported
photo_url
submission_source
logged_at
```

### If using Airtable instead

Create a base named `Oak & Iron Field Alerts` and a table named `Master Field Log` with the same field names.

---

### Node 1: Webhook - Field Issue Intake

This is the primary trigger for the recommended build.

### Node Name

`Webhook - Field Issue Intake`

### Purpose

Accepts site issue reports from a form app, internal portal, mobile app, or any system that can send a POST request.

### How to Add It

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Webhook`.
4. Add the node.
5. Rename it to `Webhook - Field Issue Intake`.

### Exact Settings

| Setting | Value |
|---|---|
| `HTTP Method` | `POST` |
| `Path` | `oak-iron-field-alert` |
| `Authentication` | `None` while testing, then `Header Auth` or `JWT Auth` in production |
| `Respond` | `Immediately` |
| `Response Code` | `200` |

Recommended options:

| Option | Value |
|---|---|
| `Allowed Origins (CORS)` | `*` during testing |
| `IP(s) Whitelist` | Add trusted sources in production if available |

### Expressions

None.

### Expected Output

The incoming JSON will be available under `$json.body`.

Example:

```json
{
  "body": {
    "project_name": "Johnson Kitchen Remodel",
    "installer_name": "Mike T",
    "issue_type": "Hidden Plumbing Conflict",
    "severity": "Critical",
    "notes": "Drain line blocks cabinet depth on sink wall.",
    "site_address": "145 Cedar Ave",
    "date_reported": "2026-04-18",
    "photo_url": ""
  }
}
```

---

### Node 2: Set - Normalize Issue Report

### Node Name

`Set - Normalize Issue Report`

### Purpose

Converts incoming data from either a webhook body or Google Forms and Google Sheets row into one clean, predictable JSON structure.

### How to Add It

1. Click the `+` after `Webhook - Field Issue Intake`.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Set - Normalize Issue Report`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into the `JSON Output` field:

```json
{
  "report_id": "{{ 'FA-' + $now.toMillis() }}",
  "project_name": "{{ ($json.body?.project_name ?? $json['Project Name'] ?? $json.project_name ?? '').trim() }}",
  "installer_name": "{{ ($json.body?.installer_name ?? $json['Installer Name'] ?? $json.installer_name ?? '').trim() }}",
  "issue_type": "{{ ($json.body?.issue_type ?? $json['Issue Type'] ?? $json.issue_type ?? '').trim() }}",
  "severity": "{{ ($json.body?.severity ?? $json['Severity'] ?? $json.severity ?? 'Standard').trim() }}",
  "severity_normalized": "{{ (($json.body?.severity ?? $json['Severity'] ?? $json.severity ?? 'Standard').trim()).toLowerCase() }}",
  "notes": "{{ ($json.body?.notes ?? $json['Notes'] ?? $json.notes ?? '').trim() }}",
  "site_address": "{{ ($json.body?.site_address ?? $json['Site Address'] ?? $json.site_address ?? '').trim() }}",
  "date_reported": "{{ ($json.body?.date_reported ?? $json['Date Reported'] ?? $json.date_reported ?? $now.toISODate()).trim() }}",
  "photo_url": "{{ ($json.body?.photo_url ?? $json['Photo URL'] ?? $json.photo_url ?? '').trim() }}",
  "submission_source": "{{ $json.body ? 'webhook' : 'form_or_sheet' }}",
  "logged_at": "{{ $now.toISO() }}"
}
```

### Expressions

Important expressions used in this node:

```text
{{ 'FA-' + $now.toMillis() }}
```

```text
{{ (($json.body?.severity ?? $json['Severity'] ?? $json.severity ?? 'Standard').trim()).toLowerCase() }}
```

```text
{{ $json.body ? 'webhook' : 'form_or_sheet' }}
```

### Expected Output

```json
{
  "report_id": "FA-1776489600000",
  "project_name": "Johnson Kitchen Remodel",
  "installer_name": "Mike T",
  "issue_type": "Hidden Plumbing Conflict",
  "severity": "Critical",
  "severity_normalized": "critical",
  "notes": "Drain line blocks cabinet depth on sink wall.",
  "site_address": "145 Cedar Ave",
  "date_reported": "2026-04-18",
  "photo_url": "",
  "submission_source": "webhook",
  "logged_at": "2026-04-18T..."
}
```

---

### Node 3: Google Sheets - Append Master Log

If you prefer Airtable, a matching Airtable setup is included immediately after this node.

### Node Name

`Google Sheets - Append Master Log`

### Purpose

Writes every field issue into a single permanent tracking log before any notifications happen.

### How to Add It

1. Click the `+` after `Set - Normalize Issue Report`.
2. Search for `Google Sheets`.
3. Add the `Google Sheets` node.
4. Rename it to `Google Sheets - Append Master Log`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Sheets credential |
| `Resource` | `Sheet Within Document` |
| `Operation` | `Append Row` |
| `Document` | `Oak & Iron - Master Field Log` |
| `Sheet` | `Master Field Log` |

Map each column to the incoming field with expressions:

| Column | Value |
|---|---|
| `report_id` | `{{ $json.report_id }}` |
| `project_name` | `{{ $json.project_name }}` |
| `installer_name` | `{{ $json.installer_name }}` |
| `issue_type` | `{{ $json.issue_type }}` |
| `severity` | `{{ $json.severity }}` |
| `severity_normalized` | `{{ $json.severity_normalized }}` |
| `notes` | `{{ $json.notes }}` |
| `site_address` | `{{ $json.site_address }}` |
| `date_reported` | `{{ $json.date_reported }}` |
| `photo_url` | `{{ $json.photo_url }}` |
| `submission_source` | `{{ $json.submission_source }}` |
| `logged_at` | `{{ $json.logged_at }}` |

### Expressions

```text
{{ $json.report_id }}
```

```text
{{ $json.severity_normalized }}
```

### Expected Output

The node appends a new row to the sheet and returns row metadata from Google Sheets.

At minimum, you should see the mapped values in the execution preview and a new row in the spreadsheet.

---

### Airtable Alternative for Node 3

Use this only if you want Airtable instead of Google Sheets.

### Node Name

`Airtable - Append Master Log`

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Airtable credential |
| `Operation` | `Append the data to a table` |
| `Base` | `Oak & Iron Field Alerts` |
| `Table` | `Master Field Log` |

Map the same fields as above using the same expressions.

---

### Node 4: Switch - Route by Severity

### Node Name

`Switch - Route by Severity`

### Purpose

Routes the issue to the correct path based on severity.

### How to Add It

1. Click the `+` after `Google Sheets - Append Master Log`.
2. Search for `Switch`.
3. Add the `Switch` node.
4. Rename it to `Switch - Route by Severity`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Rules` |

Create four outputs with these rules:

| Output | Data Type | Operation | Value 1 | Value 2 |
|---|---|---|---|---|
| `0` Critical | `String` | `equals` | `{{ $json.severity_normalized }}` | `critical` |
| `1` High | `String` | `equals` | `{{ $json.severity_normalized }}` | `high` |
| `2` Standard | `String` | `equals` | `{{ $json.severity_normalized }}` | `standard` |
| `3` Low | `String` | `equals` | `{{ $json.severity_normalized }}` | `low` |

### Expressions

Use this exact expression for `Value 1` in all four rules:

```text
{{ $json.severity_normalized }}
```

### Expected Output

- Critical issues leave output `0`
- High issues leave output `1`
- Standard issues leave output `2`
- Low issues leave output `3`

For `Standard` and `Low`, the workflow stops after the log step and route decision. That is expected.

---

### Node 5: Slack - Critical Field Alert

### Node Name

`Slack - Critical Field Alert`

### Purpose

Sends an instant urgent message when the issue is `Critical`.

### How to Add It

1. Drag from output `0` of `Switch - Route by Severity`.
2. Search for `Slack`.
3. Add the `Slack` node.
4. Rename it to `Slack - Critical Field Alert`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#field-alerts` |
| `Text` | Use the expression below |

### Expressions

Paste this into the `Text` field:

```text
CRITICAL FIELD ISSUE
Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
```

### Expected Output

A Slack message appears in `#field-alerts` with the critical issue details.

The node output normally includes Slack message metadata such as channel and timestamp.

---

### Node 6: Slack - High Priority Field Alert

### Node Name

`Slack - High Priority Field Alert`

### Purpose

Sends a faster-than-normal notification for `High` issues.

### How to Add It

1. Drag from output `1` of `Switch - Route by Severity`.
2. Add a `Slack` node.
3. Rename it to `Slack - High Priority Field Alert`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Slack credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | `#field-alerts` |
| `Text` | Use the expression below |

### Expressions

```text
HIGH PRIORITY FIELD ISSUE
Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
```

### Expected Output

A Slack message appears in `#field-alerts` for high-priority issues.

---

### Node 7: Gmail - High Priority Escalation

This node sits on the same `High` branch as the Slack alert.

### Node Name

`Gmail - High Priority Escalation`

### Purpose

Emails the project manager, shop lead, or coordinator when the issue is `High`.

### How to Add It

1. Drag a second connection from output `1` of `Switch - Route by Severity`.
2. Search for `Gmail`.
3. Add the `Gmail` node.
4. Rename it to `Gmail - High Priority Escalation`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Gmail credential |
| `Resource` | `Message` |
| `Operation` | `Send` |
| `To` | `ops@oakandiron.example` |
| `Subject` | Expression below |
| `Email Type` | `Text` |
| `Message` | Expression below |

### Expressions

Use this for the `Subject`:

```text
High Field Issue - {{ $json.project_name }} - {{ $json.issue_type }}
```

Use this for the `Message`:

```text
High priority field issue reported.

Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Site Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
Logged At: {{ $json.logged_at }}
```

### Expected Output

An email is sent for every high-priority issue. The node output includes Gmail send metadata.

---

### Optional Node 8: Merge - Rejoin Alert Branches

This is optional. Use it only if you want both the `Critical` and `High` notification paths to feed into a later reporting or summary node.

### Node Name

`Merge - Rejoin Alerts`

### Purpose

Combines alert outputs into a single path for downstream steps such as audit logging or dashboard updates.

### How to Add It

1. Search for `Merge`.
2. Connect the alert nodes into it.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Append` |

### Expressions

None.

### Expected Output

The node emits all incoming alert items in one combined stream.

---

## 6. Intake Form Fields

If you build a form, use these exact fields:

| Form Label | Field Key | Required | Notes |
|---|---|---|---|
| `Project Name` | `project_name` | Yes | Customer job or internal project name |
| `Installer Name` | `installer_name` | Yes | Person on site submitting the issue |
| `Issue Type` | `issue_type` | Yes | Example: Hidden Plumbing Conflict |
| `Severity` | `severity` | Yes | `Critical`, `High`, `Standard`, or `Low` |
| `Notes` | `notes` | Yes | Main issue description |
| `Site Address` | `site_address` | Yes | Job-site location |
| `Date Reported` | `date_reported` | Yes | Use date format |
| `Photo URL` | `photo_url` | No | Link to a photo or shared file |

### How these map into n8n fields

Use this normalized output after the Set node:

| Final n8n Field | Source |
|---|---|
| `project_name` | `$json.body.project_name` or `$json['Project Name']` |
| `installer_name` | `$json.body.installer_name` or `$json['Installer Name']` |
| `issue_type` | `$json.body.issue_type` or `$json['Issue Type']` |
| `severity` | `$json.body.severity` or `$json['Severity']` |
| `severity_normalized` | lowercased `severity` |
| `notes` | `$json.body.notes` or `$json['Notes']` |
| `site_address` | `$json.body.site_address` or `$json['Site Address']` |
| `date_reported` | `$json.body.date_reported` or `$json['Date Reported']` |
| `photo_url` | `$json.body.photo_url` or `$json['Photo URL']` |

---

## 7. Trigger Setup (Two Options)

### Option A: Webhook Trigger

Use a POST request with JSON body.

### Recommended use case

Choose this when:

- another app or internal form can send JSON
- you want instant real-time submission
- you do not want to depend on spreadsheet polling

### Sample payload

```json
{
  "project_name": "Johnson Kitchen Remodel",
  "installer_name": "Mike T",
  "issue_type": "Hidden Plumbing Conflict",
  "severity": "Critical",
  "notes": "Drain line blocks cabinet depth on sink wall.",
  "site_address": "145 Cedar Ave",
  "date_reported": "2026-04-18",
  "photo_url": ""
}
```

### Test method

1. Open the webhook node.
2. Click `Listen for Test Event`.
3. Send the JSON payload to the test URL.
4. Confirm the webhook node receives the payload in `$json.body`.

### What the webhook should do

It should trigger the workflow immediately and hand the data to `Set - Normalize Issue Report`.

---

### Option B: Google Forms + Google Sheets Trigger

Use this when installers will submit a Google Form rather than an app sending JSON directly.

### Recommended use case

Choose this when:

- you want a simple no-code submission experience
- installers can open a Google Form on mobile
- you are comfortable with Google Sheets as the response source

### Setup steps

1. Create a Google Form with these exact question labels:
   - `Project Name`
   - `Installer Name`
   - `Issue Type`
   - `Severity`
   - `Notes`
   - `Site Address`
   - `Date Reported`
   - `Photo URL`
2. Link the form responses to a Google Sheet.
3. In n8n, create a second version of the intake trigger using `Google Sheets Trigger`.

### Google Sheets Trigger Node

#### Node Name

`Google Sheets Trigger - Form Responses`

#### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your Google Sheets credential |
| `Event` | `Row added` |
| `Document` | The Google Form response spreadsheet |
| `Sheet` | The responses tab |

Recommended option:

| Option | Value |
|---|---|
| `DateTime Render` | `Formatted String` |

#### Expressions

None in the trigger itself.

#### Expected Output

The trigger sends each new response row as JSON with column names based on the form question labels, for example:

```json
{
  "Project Name": "Johnson Kitchen Remodel",
  "Installer Name": "Mike T",
  "Issue Type": "Hidden Plumbing Conflict",
  "Severity": "Critical",
  "Notes": "Drain line blocks cabinet depth on sink wall.",
  "Site Address": "145 Cedar Ave",
  "Date Reported": "2026-04-18",
  "Photo URL": ""
}
```

### Important note

If you use Google Forms and Google Sheets Trigger, keep `Set - Normalize Issue Report` exactly as written above. It already supports both webhook and sheet-based input.

---

## 8. Severity Routing Logic

Use these severity rules:

| Severity | Action |
|---|---|
| `Critical` | Send immediate Slack alert |
| `High` | Send Slack alert and escalation email |
| `Standard` | Log only |
| `Low` | Log only |

### Exact Switch logic

The routing field is:

```text
{{ $json.severity_normalized }}
```

Expected values:

- `critical`
- `high`
- `standard`
- `low`

If you want stronger control, force installers to choose from a dropdown instead of typing severity manually.

---

## 9. Notification Formats

### Slack message for critical issues

```text
CRITICAL FIELD ISSUE
Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
```

### Slack message for high issues

```text
HIGH PRIORITY FIELD ISSUE
Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
```

### High-priority email body

```text
High priority field issue reported.

Report ID: {{ $json.report_id }}
Project: {{ $json.project_name }}
Installer: {{ $json.installer_name }}
Issue Type: {{ $json.issue_type }}
Severity: {{ $json.severity }}
Site Address: {{ $json.site_address }}
Date Reported: {{ $json.date_reported }}
Notes: {{ $json.notes }}
Photo URL: {{ $json.photo_url ? $json.photo_url : 'No photo provided' }}
Logged At: {{ $json.logged_at }}
```

If you prefer `Discord` or `Telegram`, keep the same message text and swap the notification node only.

---

## 10. Separate Error Workflow Build

Build this as a second workflow.

### Node 1: Error Trigger - Main Workflow Failures

### Node Name

`Error Trigger - Main Workflow Failures`

### Purpose

Starts automatically when the main workflow fails during an automatic execution.

### How to Add It

1. Create a new workflow.
2. Click `Add first step`.
3. Search for `Error Trigger`.
4. Add the node.
5. Rename it to `Error Trigger - Main Workflow Failures`.

### Exact Settings

No extra node configuration is required.

### Expressions

None.

### Expected Output

The node provides error details from the failed workflow, including execution info, workflow name, last node, and error message.

---

### Node 2: Send Email - Admin Failure Notice

### Node Name

`Send Email - Admin Failure Notice`

### Purpose

Sends an immediate failure email to the workflow owner or operations admin.

### How to Add It

1. Click the `+` after `Error Trigger - Main Workflow Failures`.
2. Search for `Send Email`.
3. Add the node.
4. Rename it to `Send Email - Admin Failure Notice`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your SMTP credential |
| `Operation` | `Send` |
| `From Email` | Your configured sender |
| `To Email` | `admin@oakandiron.example` |
| `Subject` | Expression below |
| `Email Format` | `Text` |
| `Text` | Expression below |

### Expressions

Use this for the subject:

```text
Oak & Iron workflow failure - {{ $json.workflow.name || 'Unknown Workflow' }}
```

Use this for the message body:

```text
Workflow failure detected.

Workflow: {{ $json.workflow.name || 'Unknown Workflow' }}
Execution ID: {{ $json.execution.id || 'Unknown' }}
Execution URL: {{ $json.execution.url || 'Unavailable' }}
Last Node Executed: {{ $json.execution.lastNodeExecuted || 'Unknown' }}
Error Message: {{ $json.execution.error.message || 'No message returned' }}
Timestamp: {{ $now.toISO() }}
```

### Expected Output

An admin email is sent whenever the main workflow fails during an automatic run.

### Important linking step

After saving the error workflow:

1. Open the main workflow.
2. Click `Options` or workflow `Settings`.
3. Find `Error workflow`.
4. Select `Oak & Iron - Field Alert Error Handler`.
5. Save the main workflow.

Without this link, the error workflow will not run for the primary workflow.

---

## 11. Testing Steps

Test the workflow with one issue per severity level.

### Test 1: Critical issue

```json
{
  "project_name": "Johnson Kitchen Remodel",
  "installer_name": "Mike T",
  "issue_type": "Hidden Plumbing Conflict",
  "severity": "Critical",
  "notes": "Drain line blocks cabinet depth on sink wall.",
  "site_address": "145 Cedar Ave",
  "date_reported": "2026-04-18",
  "photo_url": ""
}
```

Expected result:

- row added to master log
- routed to critical branch
- Slack critical alert sent

### Test 2: High issue

```json
{
  "project_name": "Parker Bath Renovation",
  "installer_name": "Luis R",
  "issue_type": "Wall Damage Behind Vanity",
  "severity": "High",
  "notes": "Moisture-damaged drywall behind vanity area will need repair before cabinet install.",
  "site_address": "22 Willow Lane",
  "date_reported": "2026-04-18",
  "photo_url": "https://example.com/photos/wall-damage.jpg"
}
```

Expected result:

- row added to master log
- routed to high branch
- Slack high-priority alert sent
- escalation email sent

### Test 3: Standard issue

```json
{
  "project_name": "Dawson Kitchen Update",
  "installer_name": "Chris B",
  "issue_type": "Missing Crown Molding",
  "severity": "Standard",
  "notes": "Cabinet install can continue, but finish trim pack is incomplete.",
  "site_address": "88 Pine Court",
  "date_reported": "2026-04-18",
  "photo_url": ""
}
```

Expected result:

- row added to master log
- routed to standard branch
- no urgent alert sent

### Test 4: Low issue

```json
{
  "project_name": "Green Residence Laundry Room",
  "installer_name": "Sam D",
  "issue_type": "Minor Touch-Up Needed",
  "severity": "Low",
  "notes": "Small paint scuff on filler strip. Not blocking install.",
  "site_address": "310 Birch Way",
  "date_reported": "2026-04-18",
  "photo_url": ""
}
```

Expected result:

- row added to master log
- routed to low branch
- no urgent alert sent

### What to verify after every test

| Check | Expected result |
|---|---|
| Trigger node | Receives the payload correctly |
| Set node | Produces clean snake_case output |
| Log node | Adds a row to the master log |
| Switch node | Chooses the right severity output |
| Slack or Gmail nodes | Only run for the correct branches |
| Executions tab | Shows a successful run with the correct path |

### Error workflow test

Because `Error Trigger` only fires on automatic workflow failures, test it like this:

1. Activate the main workflow.
2. Temporarily break one downstream node, for example use an invalid spreadsheet selection or invalid Slack credential.
3. Send a real webhook request to the production URL.
4. Confirm the main workflow fails.
5. Confirm the error workflow sends the admin email.
6. Restore the broken node setting immediately after the test.

---

## 12. Export Checklist

When the workflow is complete, export and document these items.

### Workflow export

1. Open the main workflow.
2. Click the three-dot menu.
3. Select `Download`.
4. Save the file as:

```text
oak-and-iron-field-alert-intake.json
```

Do the same for the error workflow:

```text
oak-and-iron-field-alert-error-handler.json
```

### Screenshot checklist

Capture these screenshots:

- full primary workflow canvas
- full error workflow canvas
- webhook configuration
- Set node JSON mapping
- Google Sheets append row mapping
- Switch severity rules
- Slack critical alert node
- Gmail high-priority email node
- error email node
- one successful execution
- one failed execution that triggers the error workflow

### Evidence checklist

- Google Sheet with logged rows
- Slack message for a critical issue
- Gmail message for a high issue
- admin failure email from the error workflow

---

## 13. Practical Build Notes

- Rename nodes as you add them. It makes execution traces much easier to read.
- Turn on `Retry On Fail` for Slack, Gmail, and Google Sheets nodes if transient errors are common.
- Use dropdowns in the installer form for `Severity` so users cannot misspell values.
- Keep severity values exactly `Critical`, `High`, `Standard`, `Low`.
- Store every issue before routing alerts. This prevents alert-only records from being lost.
- If you switch to Airtable later, keep the normalized field names exactly the same.
- Add sticky notes to the canvas for `Trigger`, `Normalize`, `Log`, `Route`, and `Error Workflow`.
- In production, secure the webhook with authentication rather than leaving it open.

---

## 14. Final Output Example

After the `Set - Normalize Issue Report` node, the item should look like this:

```json
{
  "report_id": "FA-1776489600000",
  "project_name": "Johnson Kitchen Remodel",
  "installer_name": "Mike T",
  "issue_type": "Hidden Plumbing Conflict",
  "severity": "Critical",
  "severity_normalized": "critical",
  "notes": "Drain line blocks cabinet depth on sink wall.",
  "site_address": "145 Cedar Ave",
  "date_reported": "2026-04-18",
  "photo_url": "",
  "submission_source": "webhook",
  "logged_at": "2026-04-18T..."
}
```

This is the data contract the rest of the workflow depends on.

---

## Official Reference Links

These official n8n docs were used to align this guide to the current browser UI and node names:

- Webhook: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- Switch: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/
- Google Sheets Trigger: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlesheetstrigger/
- Google Sheets: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- Airtable: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/
- Slack: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- Discord: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.discord/
- Telegram: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/
- Gmail message operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/message-operations/
- Send Email: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.sendemail/
- Error Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.errortrigger/
- Merge: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/
- Workflow export and import: https://docs.n8n.io/workflows/export-import/

### Postwork Projec
```
Field Alert Automation for Oak & Iron Custom Cabinetry
Oak & Iron is a bespoke kitchen and bath cabinetry shop that struggles with communication gaps between installers at job sites and the production team at the shop. When installers encounter "Project Blockers"—such as unexpected plumbing behind a wall or incorrect site measurements—the current manual SMS system leads to delays and forgotten tasks.

Requirements
The company needs a centralized n8n workflow to handle field issue reporting:

Intake: The workflow must trigger via a Webhook or Google Form submission used by installers to report site issues (fields should include Project Name, Issue Type, Severity, and Notes).
Urgency Logic: The automation must use a Switch or If node to branch based on "Severity."
Critical Issues: Must trigger an immediate notification to a communication platform (Slack, Discord, or Telegram) to alert the shop foreman.
Standard Issues: Should only be logged for the weekly production meeting.
Data Persistence: Every submission, regardless of severity, must be logged as a new row in a "Master Field Log" via Google Sheets or Airtable.
Error Resilience: Include an Error Trigger node or a basic "Catch" branch to ensure that if the database logging fails, an admin receives an email notification.
Deliverables
Workflow Export: The n8n JSON file containing the complete workflow.
Visual Documentation: A screenshot of the full n8n node graph and a screenshot of a successful test execution showing the green "success" indicators on all nodes.
Proof of Output: A screenshot of the resulting Google Sheet/Airtable row and the corresponding Slack/Discord/Email notification generated during the test.
Success looks like
A reliable system where no field issue is "lost in the shuffle" and urgent blockers are flagged to the shop within seconds. Your work will be evaluated on the logical flow of the branching nodes, the naming conventions used for nodes, and the successful handling of data from the trigger to the final communication outputs.
```