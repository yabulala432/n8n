# Mandated Case Sync and Compliance Automation for Clear Path Recovery

## 1. Create Workflow

1. Log in to n8n in your browser.
2. Click `Create Workflow`.
3. Click the workflow title.
4. Rename it to `Clear Path Recovery - Case Sync`.
5. Click `Save`.
6. Create a second workflow.
7. Rename it to `Clear Path Recovery - Error Log`.
8. Click `Save`.

## 2. Node Order

Primary workflow:

Webhook  
-> Set Session Data  
-> Airtable Session Log  
-> Switch Case Type  
-> Standard Complete  
-> Format Mandated Summary  
-> Gmail Legal Liaison  
-> Google Sheets Audit Log

Error workflow:

Error Trigger  
-> Google Sheets Error Log

## 3. Build Each Node

### Webhook

Purpose:  
Receive session data from a form or external app.

Add Node:  
Click `+` -> search `Webhook`.

Settings:  
- HTTP Method: `POST`
- Path: `clear-path-session-intake`
- Response Mode: `On Received`

Fields:  
- `counselor_name`
- `patient_id`
- `session_date`
- `case_type`
- `session_notes`

Expressions:  
None.

Output:  
One item with submitted session data.

### Set Session Data

Purpose:  
Clean the input and keep only the fields needed.

Add Node:  
Click `+` after Webhook -> search `Edit Fields` or `Set` -> rename it to `Set Session Data`.

Settings:  
- Turn on `Keep Only Set`.

Fields:  
- `counselor_name` = `{{$json.body.counselor_name || $json.counselor_name}}`
- `patient_id` = `{{$json.body.patient_id || $json.patient_id}}`
- `session_date` = `{{$json.body.session_date || $json.session_date}}`
- `case_type` = `{{$json.body.case_type || $json.case_type}}`
- `session_notes` = `{{$json.body.session_notes || $json.session_notes}}`
- `received_at` = `{{$now}}`

Expressions:  
- `{{$now}}`
- `{{$json.body.counselor_name || $json.counselor_name}}`
- `{{$json.body.patient_id || $json.patient_id}}`
- `{{$json.body.session_date || $json.session_date}}`
- `{{$json.body.case_type || $json.case_type}}`
- `{{$json.body.session_notes || $json.session_notes}}`

Output:  
Clean session data.

### Airtable Session Log

Purpose:  
Append every session to the master Airtable log.

Add Node:  
Click `+` after Set Session Data -> search `Airtable`.

Settings:  
- Connect your Airtable account.
- Resource: `Record`
- Operation: `Create`
- Base: `Session Log`
- Table: `Session Log`

Fields:  
- `Counselor Name` = `{{$json.counselor_name}}`
- `Patient ID` = `{{$json.patient_id}}`
- `Session Date` = `{{$json.session_date}}`
- `Case Type` = `{{$json.case_type}}`
- `Session Notes` = `{{$json.session_notes}}`
- `Received At` = `{{$json.received_at}}`

Expressions:  
- `{{$json.counselor_name}}`
- `{{$json.patient_id}}`
- `{{$json.session_date}}`
- `{{$json.case_type}}`
- `{{$json.session_notes}}`
- `{{$json.received_at}}`

Output:  
Session added to Airtable master log.

### Switch Case Type

Purpose:  
Send mandated sessions to the compliance branch.

Add Node:  
Click `+` after Airtable Session Log -> search `Switch`.

Settings:  
- Mode: `Rules`

Fields:  
- Value to check = `{{$json.case_type}}`

Create 2 outputs:
- `Standard`
- `Mandated`

Expressions:  
- `{{$json.case_type}}`

Output:  
Routes item to the correct branch.

### Standard Complete

Purpose:  
Finish the standard path with no extra legal routing.

Add Node:  
Click `+` from the `Standard` output of Switch Case Type -> search `No Operation, do nothing` or `Set`.

Settings:  
Optional.

Fields:  
If using `Set`:
- `status` = `standard_logged`

Expressions:  
- `{{$json.case_type}}`

Output:  
Standard session is complete after Airtable logging.

### Format Mandated Summary

Purpose:  
Create a clean summary for legal review.

Add Node:  
Click `+` from the `Mandated` output of Switch Case Type -> search `OpenAI Chat Model`, `Basic LLM Chain`, or `AI Agent`.

Settings:  
- Use your AI credentials.
- Ask for plain text only.

Fields:  
Prompt:

```text
Summarize these court-ordered counseling session notes in a clean professional format.

Keep it short and factual.
Do not add advice.
Do not add extra labels beyond the summary.

Counselor Name: {{$json.counselor_name}}
Patient ID: {{$json.patient_id}}
Session Date: {{$json.session_date}}
Case Type: {{$json.case_type}}
Session Notes: {{$json.session_notes}}
```

Expressions:  
- `{{$json.counselor_name}}`
- `{{$json.patient_id}}`
- `{{$json.session_date}}`
- `{{$json.case_type}}`
- `{{$json.session_notes}}`

Output:  
Clean mandated case summary.

### Gmail Legal Liaison

Purpose:  
Email the mandated case details to the legal liaison.

Add Node:  
Click `+` after Format Mandated Summary -> search `Gmail`.

Settings:  
- Resource: `Message`
- Operation: `Send`
- To: `legal.liaison@clearpathrecovery.com`
- Email Type: `Text`

Fields:  
- To = `legal.liaison@clearpathrecovery.com`
- Subject = `Mandated Session Alert - {{$json.patient_id}} - {{$json.session_date}}`
- Message =

```text
Court-Ordered Session Notice

Counselor Name: {{$json.counselor_name}}
Patient ID: {{$json.patient_id}}
Session Date: {{$json.session_date}}
Case Type: {{$json.case_type}}

Summary:
{{$json.text || $json.summary || $json.output || $json.session_notes}}
```

Expressions:  
- `{{$json.patient_id}}`
- `{{$json.session_date}}`
- `{{$json.counselor_name}}`
- `{{$json.case_type}}`
- `{{$json.text || $json.summary || $json.output || $json.session_notes}}`

Output:  
Legal liaison email sent.

### Google Sheets Audit Log

Purpose:  
Create a second audit record for mandated sessions only.

Add Node:  
Click `+` after Gmail Legal Liaison -> search `Google Sheets`.

Settings:  
- Resource: `Sheet Within Document`
- Operation: `Append Row`
- Spreadsheet: `Court Ordered Audit Log`
- Sheet: `Mandated Audit`

Fields:  
- `Counselor Name` = `{{$json.counselor_name}}`
- `Patient ID` = `{{$json.patient_id}}`
- `Session Date` = `{{$json.session_date}}`
- `Case Type` = `{{$json.case_type}}`
- `Summary` = `{{$json.text || $json.summary || $json.output || $json.session_notes}}`
- `Logged At` = `{{$now}}`

Expressions:  
- `{{$json.counselor_name}}`
- `{{$json.patient_id}}`
- `{{$json.session_date}}`
- `{{$json.case_type}}`
- `{{$json.text || $json.summary || $json.output || $json.session_notes}}`
- `{{$now}}`

Output:  
Mandated session added to audit sheet.

## 4. Error Workflow

### Error Trigger

Purpose:  
Catch failed executions from the main workflow.

Add Node:  
In the second workflow click `+` -> search `Error Trigger`.

Settings:  
None.

Fields:  
None.

Expressions:  
None.

Output:  
Error details from failed workflow runs.

### Google Sheets Error Log

Purpose:  
Write every failure to a separate error log sheet.

Add Node:  
Click `+` after Error Trigger -> search `Google Sheets`.

Settings:  
- Resource: `Sheet Within Document`
- Operation: `Append Row`
- Spreadsheet: `Court Ordered Audit Log`
- Sheet: `Error Log`

Fields:  
- `Workflow Name` = `{{$json.workflow.name}}`
- `Execution ID` = `{{$json.execution.id}}`
- `Last Node` = `{{$json.lastNodeExecuted}}`
- `Error Message` = `{{$json.error.message}}`
- `Failed At` = `{{$now}}`

Expressions:  
- `{{$json.workflow.name}}`
- `{{$json.execution.id}}`
- `{{$json.lastNodeExecuted}}`
- `{{$json.error.message}}`
- `{{$now}}`

Output:  
Failure logged to Google Sheets.

## 5. Test Data

Use this sample payload in the Webhook test:

```json
{
  "counselor_name": "Dana Holt",
  "patient_id": "CPR-2048",
  "session_date": "2026-04-24",
  "case_type": "Mandated",
  "session_notes": "Patient attended scheduled court-ordered counseling session, completed progress review, and discussed next compliance check-in."
}
```

## 6. Expected Result

- Every submission is added to the Airtable `Session Log`.
- `Standard` sessions stop after the Airtable step.
- `Mandated` sessions also create a clean summary.
- `Mandated` sessions send a Gmail notice to the legal liaison.
- `Mandated` sessions append a row to the Google Sheets audit log.
- Any workflow failure is written to the `Error Log` sheet.

## 7. Suggested Names

Use these node names in n8n:

- `Webhook`
- `Set Session Data`
- `Airtable Session Log`
- `Switch Case Type`
- `Standard Complete`
- `Format Mandated Summary`
- `Gmail Legal Liaison`
- `Google Sheets Audit Log`
- `Error Trigger`
- `Google Sheets Error Log`
