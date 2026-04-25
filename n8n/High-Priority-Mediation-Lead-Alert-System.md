# High-Priority Mediation Lead Alert System

## Detailed n8n Implementation Guide

## 1. What You Are Building

FairPath Mediation needs a simple intake workflow that does three things reliably:

1. Accept a new consultation request from a website form
2. Log every lead into Google Sheets
3. Send a Slack alert only when the case is priority-level complex

For this project, "priority" means:

- `International`
- `High-Net-Worth`

Standard leads should still be logged, but they should not trigger Slack.

## 2. Final Workflow Shape

Build the workflow in this order:

`Webhook - Receive Lead`  
`-> Set - Normalize Lead Data`  
`-> Google Sheets - Append Lead Row`  
`-> IF - Is High Priority Case`  
`-> Slack - Send Priority Alert`  
`-> Respond to Webhook - Success Response`

### Why this order is safest

This order stores the lead in Google Sheets before Slack runs.

That matters because logging the lead is the most important business requirement. If Slack ever fails, the lead is still safely captured.

## 3. Workflow Name

Create one workflow and name it exactly:

`High-Priority Mediation Lead Alert System`

## 4. Google Sheet Preparation

Before opening the workflow editor, create a Google Sheet named:

`FairPath Mediation Lead Tracker`

Create one tab named:

`Leads`

Add these column headers in row `1` exactly:

- `submitted_at`
- `name`
- `email`
- `case_type`
- `estimated_assets`
- `priority_label`
- `slack_alert_sent`

## 5. Incoming Payload Contract

Your website form or test request should send JSON in this shape:

```json
{
  "name": "Jordan Lee",
  "email": "jordan.lee@example.com",
  "case_type": "High-Net-Worth",
  "estimated_assets": 2450000
}
```

### Required fields

- `name`
- `email`
- `case_type`
- `estimated_assets`

### Accepted `case_type` values

- `Standard`
- `International`
- `High-Net-Worth`

## 6. Credentials You Need

Set up these credentials in n8n before testing:

- `Google Sheets`
- `Slack`

The `Webhook` node does not need a third-party credential for basic setup.

## 7. Create the Workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to `High-Priority Mediation Lead Alert System`.
4. Click `Save`.

## 8. Node 1: Webhook - Receive Lead

### Purpose

This is the public entry point for new consultation requests.

### Add the node

1. Click `Add first step`.
2. Search for `Webhook`.
3. Add the node.
4. Rename it to `Webhook - Receive Lead`.

### Configure the node

- `HTTP Method`: `POST`
- `Path`: `fairpath-mediation-lead`
- `Authentication`: `None` for initial testing
- `Response Mode`: `Using Respond to Webhook Node`

### Why use `Respond to Webhook`

This gives you clean control over the success message returned to the form or API client.

### Expected input

The node should receive:

- `name`
- `email`
- `case_type`
- `estimated_assets`

## 9. Node 2: Set - Normalize Lead Data

### Purpose

Clean the incoming webhook payload and create reusable fields for logging and Slack routing.

### Add the node

1. Click the `+` after `Webhook - Receive Lead`.
2. Search for `Set` or `Edit Fields`.
3. Add the node.
4. Rename it to `Set - Normalize Lead Data`.

### Configure the node

- Turn on `Keep Only Set`

### Add these fields

- `submitted_at` = `{{$now}}`
- `name` = `{{$json.body.name || $json.name}}`
- `email` = `{{$json.body.email || $json.email}}`
- `case_type` = `{{$json.body.case_type || $json.case_type}}`
- `estimated_assets` = `{{ Number($json.body.estimated_assets || $json.estimated_assets) }}`
- `priority_label` = `{{ ['International', 'High-Net-Worth'].includes($json.body.case_type || $json.case_type) ? 'Priority' : 'Standard' }}`
- `slack_alert_sent` = `{{ ['International', 'High-Net-Worth'].includes($json.body.case_type || $json.case_type) ? 'Pending' : 'Not Required' }}`

### Why these expressions are used

- `{{$now}}` stamps the submission time.
- `{{$json.body... || $json...}}` makes the workflow more forgiving in case the webhook payload appears either inside `body` or directly at the root.
- `Number(...)` ensures asset values are stored as numbers.

### Output of this node

One clean lead object like:

```json
{
  "submitted_at": "2026-04-25T10:20:00.000Z",
  "name": "Jordan Lee",
  "email": "jordan.lee@example.com",
  "case_type": "High-Net-Worth",
  "estimated_assets": 2450000,
  "priority_label": "Priority",
  "slack_alert_sent": "Pending"
}
```

## 10. Node 3: Google Sheets - Append Lead Row

### Purpose

Log every lead immediately into the central tracker.

### Add the node

1. Click the `+` after `Set - Normalize Lead Data`.
2. Search for `Google Sheets`.
3. Add the node.
4. Rename it to `Google Sheets - Append Lead Row`.

### Configure the node

- Credential: your Google Sheets credential
- Resource: `Sheet Within Document`
- Operation: `Append Row`
- Document: `FairPath Mediation Lead Tracker`
- Sheet: `Leads`

### Map the columns exactly

- `submitted_at` = `{{$json.submitted_at}}`
- `name` = `{{$json.name}}`
- `email` = `{{$json.email}}`
- `case_type` = `{{$json.case_type}}`
- `estimated_assets` = `{{$json.estimated_assets}}`
- `priority_label` = `{{$json.priority_label}}`
- `slack_alert_sent` = `{{$json.slack_alert_sent}}`

### Expected result

Every webhook submission creates a new row in the `Leads` tab.

## 11. Node 4: IF - Is High Priority Case

### Purpose

Check whether the lead should trigger Slack.

### Add the node

1. Click the `+` after `Google Sheets - Append Lead Row`.
2. Search for `IF`.
3. Add the node.
4. Rename it to `IF - Is High Priority Case`.

### Configure the logic

Use an expression-based condition that evaluates whether the case type is one of the two priority types.

### Condition

- Left Value:

```text
{{ ['International', 'High-Net-Worth'].includes($json.case_type) }}
```

- Operation: `is true`

### Branch meaning

- `true` output = send Slack alert
- `false` output = do not send Slack alert

## 12. Node 5: Slack - Send Priority Alert

### Purpose

Post an alert for high-priority mediation leads only.

### Add the node

1. From the `true` output of `IF - Is High Priority Case`, click `+`.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Slack - Send Priority Alert`.

### Configure the node

- Credential: your Slack credential
- Resource: `Message`
- Operation: `Post`
- Channel: your target intake channel, for example `#priority-leads`

### Slack message text

Use this message:

```text
Priority Lead Alert

Name: {{$json.name}}
Email: {{$json.email}}
Case Type: {{$json.case_type}}
Estimated Assets: ${{$json.estimated_assets}}
Priority: {{$json.priority_label}}
Submitted At: {{$json.submitted_at}}
```

### Recommended Slack formatting improvement

If your Slack node supports markdown-style formatting, use:

```text
*Priority Lead Alert*
*Name:* {{$json.name}}
*Email:* {{$json.email}}
*Case Type:* {{$json.case_type}}
*Estimated Assets:* ${{$json.estimated_assets}}
*Priority:* {{$json.priority_label}}
*Submitted At:* {{$json.submitted_at}}
```

### Important behavior

Only `International` and `High-Net-Worth` leads should ever reach this node.

## 13. Node 6: Respond to Webhook - Success Response

### Purpose

Return a clean success response to the sender after processing.

### Add the node

You need two inbound connections into this node:

1. One from the `false` branch of `IF - Is High Priority Case`
2. One from `Slack - Send Priority Alert`

### Add the node

1. Search for `Respond to Webhook`.
2. Add the node.
3. Rename it to `Respond to Webhook - Success Response`.

### Configure the node

- `Respond With`: `JSON`
- `Response Code`: `200`

### Response Body

```json
{
  "status": "success",
  "message": "Lead received and processed."
}
```

### Why both branches should connect here

This makes the workflow return the same clean response whether the lead is standard or priority.

## 14. Final Canvas Layout

Your finished canvas should look like this:

`Webhook - Receive Lead`  
`-> Set - Normalize Lead Data`  
`-> Google Sheets - Append Lead Row`  
`-> IF - Is High Priority Case`

From `true`:

`-> Slack - Send Priority Alert`  
`-> Respond to Webhook - Success Response`

From `false`:

`-> Respond to Webhook - Success Response`

## 15. Node Naming Standard

To satisfy the clarity requirement, keep the node names exactly functional and readable:

- `Webhook - Receive Lead`
- `Set - Normalize Lead Data`
- `Google Sheets - Append Lead Row`
- `IF - Is High Priority Case`
- `Slack - Send Priority Alert`
- `Respond to Webhook - Success Response`

## 16. Test Payloads

### Test 1: Standard lead

This should be logged to Google Sheets and should not send Slack.

```json
{
  "name": "Taylor Morgan",
  "email": "taylor@example.com",
  "case_type": "Standard",
  "estimated_assets": 180000
}
```

### Test 2: International lead

This should be logged and should send Slack.

```json
{
  "name": "Avery Shah",
  "email": "avery@example.com",
  "case_type": "International",
  "estimated_assets": 920000
}
```

### Test 3: High-Net-Worth lead

This should be logged and should send Slack.

```json
{
  "name": "Jordan Lee",
  "email": "jordan.lee@example.com",
  "case_type": "High-Net-Worth",
  "estimated_assets": 2450000
}
```

## 17. How to Execute the Test Run

### In n8n

1. Open the workflow.
2. Click `Test Workflow`.
3. Copy the test webhook URL from `Webhook - Receive Lead`.
4. Send one of the sample payloads using Postman, Insomnia, curl, or your site form.

### What to verify for a standard lead

- The workflow execution succeeds
- A new row appears in Google Sheets
- No Slack message is posted

### What to verify for a priority lead

- The workflow execution succeeds
- A new row appears in Google Sheets
- A Slack alert appears in the selected channel

## 18. Screenshot Checklist for Deliverables

To match the requested deliverables, capture these:

### A. Workflow canvas screenshot

Take one full high-resolution screenshot showing:

- all nodes
- both `true` and `false` branches
- clear node labels

### B. Successful webhook execution

Capture the webhook test showing:

- the sample JSON payload
- successful execution in n8n

### C. Google Sheets proof

Capture the new row in `FairPath Mediation Lead Tracker`

### D. Slack proof

For the `International` or `High-Net-Worth` test case, capture the Slack message showing:

- name
- email
- case type
- estimated assets
- `Priority` label

## 19. Success Criteria

Your workflow is correct when all of these are true:

- every webhook submission creates a Google Sheets row
- `Standard` leads stop after logging
- `International` leads trigger Slack
- `High-Net-Worth` leads trigger Slack
- the Slack message clearly displays the lead details and `Priority` label
- the workflow nodes are clearly named and easy to follow on the canvas

## 20. Recommended Small Improvement

If you want slightly better audit accuracy later, add one more step after Slack to update the sheet row from `Pending` to `Sent`.

That is optional for this version because your current requirement only asks that all leads are logged and that priority leads generate a Slack alert.
