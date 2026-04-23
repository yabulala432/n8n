# HR Compliance Sync Workflow Guide

## 1. Create Workflow

1. Log in to n8n in your browser.
2. Click `Create Workflow`.
3. Click the workflow title.
4. Rename it to `HR Compliance Sync`.
5. Click `Save`.
6. Create a second workflow.
7. Rename it to `HR Compliance Sync - Error Log`.
8. Click `Save`.

## 2. Node Order

Primary workflow:

Google Sheets Trigger  
→ Set Data  
→ Airtable Search  
→ If Record Found  
→ Switch Status  
→ Approved Branch  
→ Denied Branch  
→ Pending Branch

Error workflow:

Error Trigger  
→ Google Sheets Error Log

## 3. Build Each Node

### Google Sheets Trigger

Purpose:  
Start the workflow when a row is added or updated.

Add Node:  
Click `+` → search `Google Sheets Trigger`.

Settings:  
- Connect your Google Sheets account.
- Select your spreadsheet.
- Sheet: `Background Check Log`
- Trigger on:
  - `New Row`
  - `Updated Row`

Fields:  
- Spreadsheet: `Background Check Log`
- Sheet: `Background Check Log`

Expressions:  
None.

Output:  
One item with sheet row data.

### Set Data

Purpose:  
Clean the row data and keep only the fields you need.

Add Node:  
Click `+` after Google Sheets Trigger → search `Edit Fields` or `Set` → rename it to `Set Data`.

Settings:  
- Turn on `Keep Only Set`.

Fields:  
- `applicant_name` = `{{$json.applicant_name}}`
- `email` = `{{$json.email}}`
- `status` = `{{$json.clearance_status}}`
- `reason` = `{{$json.reason}}`
- `updated_at` = `{{$json.updated_at}}`
- `sync_time` = `{{$now}}`

Expressions:  
- `{{$now}}`
- `{{$json.applicant_name}}`
- `{{$json.email}}`
- `{{$json.clearance_status}}`
- `{{$json.reason}}`
- `{{$json.updated_at}}`

Output:  
Clean applicant data.

### Airtable Search

Purpose:  
Find the applicant in Airtable by email.

Add Node:  
Click `+` after Set Data → search `Airtable`.

Settings:  
- Connect your Airtable account.
- Resource: `Record`
- Operation: `Search`
- Base: `Substitute Teacher CRM`
- Table: `Applicants`
- Return All: `false`
- Limit: `1`
- Formula: `{Email}='{{$node["Set Data"].json["email"]}}'`
- Open the `Settings` tab and turn on `Always Output Data`

Fields:  
- Base = `Substitute Teacher CRM`
- Table = `Applicants`
- Limit = `1`
- Formula = `{Email}='{{$node["Set Data"].json["email"]}}'`

Expressions:  
- `{{$node["Set Data"].json["email"]}}`

Output:  
Matching Airtable record, or one empty item if not found.

### If Record Found

Purpose:  
Check if Airtable returned a record.

Add Node:  
Click `+` after Airtable Search → search `If`.

Settings:  
- Condition type: `Single`

Fields:  
- Left Value = `{{$json.id}}`
- Operation = `Exists`

Expressions:  
- `{{$json.id}}`

Output:  
`True` if record exists.  
`False` if no record exists.

### Switch Status

Purpose:  
Send the item to the correct status branch.

Add Node:  
Click `+` from the `true` output of If Record Found → search `Switch`.

Settings:  
- Mode: `Rules`
- Rules:
  - `Approved`
  - `Denied`
  - `Pending`

Fields:  
- Value to check = `{{$node["Set Data"].json["status"]}}`
- Rule 1 = `Approved`
- Rule 2 = `Denied`
- Rule 3 = `Pending`

Expressions:  
- `{{$node["Set Data"].json["status"]}}`

Output:  
Routes the item to the correct branch.

### Approved Branch

Purpose:  
Update the applicant to active and set the compliance date.

Add Node:  
Click `+` from the `Approved` output of Switch Status → search `Airtable`.

Settings:  
- Resource: `Record`
- Operation: `Update`
- Base: `Substitute Teacher CRM`
- Table: `Applicants`

Fields:  
- Record ID = `{{$json.id}}`
- `Status` = `Active`
- `Compliance Date` = `{{$now}}`

Expressions:  
- `{{$json.id}}`
- `{{$now}}`

Output:  
Airtable record updated to active.

### Denied Branch

Purpose:  
Update the applicant to ineligible and alert HR.

Add Node:  
1. Click `+` from the `Denied` output of Switch Status → search `Airtable`  
2. Click `+` after that Airtable node → search `Gmail`

Settings:  
Airtable node:
- Resource: `Record`
- Operation: `Update`
- Base: `Substitute Teacher CRM`
- Table: `Applicants`

Gmail node:
- Resource: `Message`
- Operation: `Send`
- To: `hr@yourcompany.com`

Fields:  
Airtable node:
- Record ID = `{{$json.id}}`
- `Status` = `Ineligible`

Gmail node:
- To = `hr@yourcompany.com`
- Subject = `Applicant Denied Clearance - {{$node["Set Data"].json["applicant_name"]}}`
- Email Type = `Text`
- Message =

```text
Applicant: {{$node["Set Data"].json["applicant_name"]}}
Email: {{$node["Set Data"].json["email"]}}
Status: Denied
Reason: {{$node["Set Data"].json["reason"]}}
```

Expressions:  
- `{{$json.id}}`
- `{{$node["Set Data"].json["applicant_name"]}}`
- `{{$node["Set Data"].json["email"]}}`
- `{{$node["Set Data"].json["reason"]}}`

Output:  
Airtable record updated. Gmail alert sent.

### Pending Branch

Purpose:  
Keep the applicant pending, or do nothing.

Add Node:  
Optional. Click `+` from the `Pending` output of Switch Status → search `Airtable`.

Settings:  
- Resource: `Record`
- Operation: `Update`
- Base: `Substitute Teacher CRM`
- Table: `Applicants`

Fields:  
- Record ID = `{{$json.id}}`
- `Status` = `Pending`

Expressions:  
- `{{$json.id}}`

Output:  
Pending status kept.  
If you want no action, leave this branch empty.

### Error Trigger

Purpose:  
Start the error workflow when the main workflow fails.

Add Node:  
Open the second workflow → click `+` → search `Error Trigger`.

Settings:  
No extra settings.

Fields:  
None.

Expressions:  
None.

Output:  
Error details from the failed workflow.

### Google Sheets Error Log

Purpose:  
Write sync errors into the `Sync Errors` tab.

Add Node:  
Click `+` after Error Trigger → search `Google Sheets`.

Settings:  
- Connect your Google Sheets account.
- Resource: `Sheet Within Document`
- Operation: `Append Row`
- Spreadsheet: `Background Check Log`
- Sheet: `Sync Errors`

Fields:  
- `time` = `{{$now}}`
- `workflow` = `{{$json.workflow.name}}`
- `node` = `{{$json.execution.lastNodeExecuted}}`
- `error` = `{{$json.execution.error.message}}`

Expressions:  
- `{{$now}}`
- `{{$json.workflow.name}}`
- `{{$json.execution.lastNodeExecuted}}`
- `{{$json.execution.error.message}}`

Output:  
One new row in `Sync Errors`.

## 4. Google Sheet Setup

Create sheet:

`Background Check Log`

Columns:

- `applicant_name`
- `email`
- `clearance_status`
- `reason`
- `updated_at`

Create second tab:

`Sync Errors`

Columns:

- `time`
- `workflow`
- `node`
- `error`

## 5. Trigger Node

Use `Google Sheets Trigger`.

Watch:

`Background Check Log`

Trigger on:

- `New Row`
- `Updated Row`

## 6. Set Node

Create fields:

- `sync_time` = `{{$now}}`
- `email` = `{{$json.email}}`
- `status` = `{{$json.clearance_status}}`

Also add:

- `applicant_name` = `{{$json.applicant_name}}`
- `reason` = `{{$json.reason}}`
- `updated_at` = `{{$json.updated_at}}`

## 7. Airtable Setup

Base:

`Substitute Teacher CRM`

Table:

`Applicants`

Columns:

- `Name`
- `Email`
- `Status`
- `Compliance Date`
- `Notes`

## 8. Airtable Search Node

Search by email.

Formula:

`{Email}='{{$node["Set Data"].json["email"]}}'`

This email is the unique key.

Turn on:

- `Always Output Data`

## 9. IF Node

Check if record exists.

Condition:

`{{$json.id}}` `Exists`

True:  
Continue

False:  
Optional create record or send admin notice

## 10. Switch Node

Use value:

`{{$node["Set Data"].json["status"]}}`

Branches:

- `Approved`
- `Denied`
- `Pending`

## 11. Approved Branch

Use `Airtable Update Record`.

Update:

- `Status` = `Active`
- `Compliance Date` = `{{$now}}`

Use Airtable record ID from Search:

`{{$json.id}}`

## 12. Denied Branch

Use `Airtable Update Record`.

Update:

- `Status` = `Ineligible`

Then add `Gmail`.

To:

`HR Manager email`

Subject:

`Applicant Denied Clearance - {{$node["Set Data"].json["applicant_name"]}}`

Body:

```text
Applicant: {{$node["Set Data"].json["applicant_name"]}}
Email: {{$node["Set Data"].json["email"]}}
Status: Denied
Reason: {{$node["Set Data"].json["reason"]}}
```

## 13. Pending Branch

Optional Airtable Update:

- `Status` = `Pending`

Or no action.

## 14. Error Handling Workflow

Create second workflow.

Node Order:

Error Trigger  
→ Google Sheets Append Row

Sheet tab:

`Sync Errors`

Fields:

- `time` = `{{$now}}`
- `workflow` = `{{$json.workflow.name}}`
- `failed node` = `{{$json.execution.lastNodeExecuted}}`
- `error message` = `{{$json.execution.error.message}}`

## 15. Test Data

### Test 1

Name: `Sarah Lee`  
Email: `sarah@test.com`  
Status: `Approved`

Expected:  
Airtable = `Active`

### Test 2

Name: `John Reed`  
Email: `john@test.com`  
Status: `Denied`

Expected:  
Airtable = `Ineligible` + Gmail sent

### Test 3

Name: `Mia Cole`  
Email: `mia@test.com`  
Status: `Pending`

Expected:  
No issue / Pending

## 16. Final Check

Make sure:

- Trigger works
- Search uses email
- Record updates correctly
- Gmail sends
- Errors log to sheet

## 17. Export

Submit:

- Workflow JSON
- Screenshot full canvas
- Screenshot Airtable updated row
- Screenshot Gmail notification
- Screenshot Sync Errors test
