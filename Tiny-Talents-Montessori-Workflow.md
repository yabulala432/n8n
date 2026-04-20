# Tiny Talents Montessori Workflow Guide

## 1. Create New Workflow

1. Log in to n8n.
2. Click `Create Workflow`.
3. Click the workflow title.
4. Rename it to `Tiny Talents Montessori`.
5. Click `Save`.

## 2. Node Order

Form Trigger  
→ Set Data  
→ AI Classifier  
→ Switch  
→ Milestone Branch  
→ Incident Branch  
→ Error Branch

## 3. Build Each Node

### Form Trigger

Purpose:  
Get teacher notes from a form.

Add Node:  
Click `+` → search `n8n Form Trigger` or `Webhook`.

Settings:  
Set form to public.

Fields:  
- `child_name`
- `teacher_name`
- `classroom`
- `raw_notes`
- `date`

Expressions:  
None.

Output:  
Form data from teacher.

### Set Data

Purpose:  
Clean and prepare data.

Add Node:  
Click `+` after Form Trigger → search `Set`.

Settings:  
Turn on `Keep Only Set` if you want clean output.

Fields:  
- `child_name` = `{{$json.child_name}}`
- `teacher_name` = `{{$json.teacher_name}}`
- `classroom` = `{{$json.classroom}}`
- `raw_notes` = `{{$json.raw_notes}}`
- `date` = `{{$json.date}}`
- `submitted_at` = `{{$now}}`
- `status` = `New`

Expressions:  
- `{{$now}}`

Output:  
Clean teacher note data.

### AI Classifier

Purpose:  
Classify the note and write parent message.

Add Node:  
Click `+` after Set Data → search `OpenAI Chat Model` or `AI Agent`.

Settings:  
Use your OpenAI credentials.  
Ask for JSON only.

Fields:  
Prompt:

```text
You are a preschool admin assistant.

Read teacher notes.

Decide one category only:

- Milestone
- Incident
- Unclear

Detect emotional tone:
- Positive
- Neutral
- Concerning

Write a warm professional parent message.

Return ONLY JSON:

{
  "category":"",
  "tone":"",
  "confidence":"",
  "parent_message":"",
  "reason":""
}

Teacher Notes:
{{$json.raw_notes}}
```

Expressions:  
- `{{$json.raw_notes}}`

Output:  
JSON with:
- `category`
- `tone`
- `confidence`
- `parent_message`
- `reason`

### Switch

Purpose:  
Send data to the correct branch.

Add Node:  
Click `+` after AI Classifier → search `Switch`.

Settings:  
Mode: `Rules`

Fields:  
Value to check:

- `{{$json.category}}`

Create 3 outputs:
- `Milestone`
- `Incident`
- `Unclear`

Expressions:  
- `{{$json.category}}`

Output:  
Routes item to one branch.

### Milestone Branch

Purpose:  
Save milestone and create parent email draft.

Add Node:  
First add `Airtable`, then add `Email` or `Gmail`.

Settings:  
Airtable operation: `Create Record`

Fields:  
Airtable table: `Growth Tracker`

Save:
- `child_name` = `{{$json.child_name}}`
- `teacher_name` = `{{$json.teacher_name}}`
- `raw_notes` = `{{$json.raw_notes}}`
- `parent_message` = `{{$json.parent_message}}`
- `submitted_at` = `{{$json.submitted_at}}`

Email Draft:
- Subject = `Today’s Growth Update for {{$json.child_name}}`
- Body = `{{$json.parent_message}}`

Expressions:  
- `{{$json.child_name}}`
- `{{$json.parent_message}}`
- `{{$json.submitted_at}}`

Output:  
Milestone saved. Email draft created.

### Incident Branch

Purpose:  
Send urgent alert and save for review.

Add Node:  
First add `Slack` or `Discord`, then add `Airtable`.

Settings:  
Slack/Discord action: `Send Message`

Fields:  
Alert message:

```text
🚨 Incident Alert

Child: {{$json.child_name}}
Teacher: {{$json.teacher_name}}
Notes: {{$json.raw_notes}}
```

Airtable table: `Priority Review`

Save:
- `child_name`
- `teacher_name`
- `classroom`
- `raw_notes`
- `date`
- `submitted_at`
- `category`
- `tone`
- `confidence`
- `parent_message`
- `reason`

Expressions:  
- `{{$json.child_name}}`
- `{{$json.teacher_name}}`
- `{{$json.raw_notes}}`

Output:  
Alert sent. Incident saved.

### Error Branch

Purpose:  
Notify admin when AI is unsure.

Add Node:  
Click `+` from `Unclear` output → search `Email`.

Settings:  
Send to admin email.

Fields:  
Subject:
`AI Could Not Classify Submission`

Body:

```text
Review this entry:

{{$json.raw_notes}}
```

Expressions:  
- `{{$json.raw_notes}}`

Output:  
Admin gets review email.

## 4. Form Trigger Setup

Use `n8n Form Trigger` or `Webhook`.

Create fields:
- `child_name`
- `teacher_name`
- `classroom`
- `raw_notes`
- `date`

Steps:
1. Open the Form Trigger node.
2. Click `Add Field`.
3. Add each field above.
4. Make `raw_notes` a long text field.
5. Copy the public form URL.
6. Test the form in browser.

## 5. Set Node

Clean data.

Create:
- `submitted_at = {{$now}}`
- `status = New`

Steps:
1. Add a `Set` node after the form.
2. Click `Add Field`.
3. Add all form fields again.
4. Add `submitted_at`.
5. Add `status`.
6. Paste the expressions.

## 6. AI Node Setup

Use `OpenAI Chat Model` or `AI Agent` node.

Use this exact prompt:

```text
You are a preschool admin assistant.

Read teacher notes.

Decide one category only:

- Milestone
- Incident
- Unclear

Detect emotional tone:
- Positive
- Neutral
- Concerning

Write a warm professional parent message.

Return ONLY JSON:

{
  "category":"",
  "tone":"",
  "confidence":"",
  "parent_message":"",
  "reason":""
}

Teacher Notes:
{{$json.raw_notes}}
```

Steps:
1. Add the AI node after `Set Data`.
2. Connect OpenAI credentials.
3. Paste the prompt.
4. Turn on JSON output if available.
5. Run test once.
6. Check that the result includes:
   - `category`
   - `tone`
   - `confidence`
   - `parent_message`
   - `reason`

## 7. Switch Node

Use value:

```text
{{$json.category}}
```

Create branches:
- `Milestone`
- `Incident`
- `Unclear`

Steps:
1. Add `Switch` after AI node.
2. Set value to `{{$json.category}}`.
3. Add 3 rules:
   - equals `Milestone`
   - equals `Incident`
   - equals `Unclear`

## 8. Milestone Branch

Use `Airtable` node.

Table:

`Growth Tracker`

Save:
- `child_name`
- `teacher_name`
- `raw_notes`
- `parent_message`
- `submitted_at`

Then create `Email Draft` node.

Subject:

```text
Today’s Growth Update for {{$json.child_name}}
```

Body:

```text
{{$json.parent_message}}
```

Steps:
1. From `Milestone` output, add `Airtable`.
2. Choose base and table `Growth Tracker`.
3. Map the fields.
4. Add `Email` or `Gmail` after Airtable.
5. Set the subject and body.
6. Test with milestone sample.

## 9. Incident Branch

Use `Slack` or `Discord` node.

Message:

```text
🚨 Incident Alert

Child: {{$json.child_name}}
Teacher: {{$json.teacher_name}}
Notes: {{$json.raw_notes}}
```

Then `Airtable` node:

`Priority Review`

Save all fields.

Steps:
1. From `Incident` output, add `Slack` or `Discord`.
2. Select channel.
3. Paste the alert message.
4. Add `Airtable` after the alert.
5. Choose table `Priority Review`.
6. Map all fields.
7. Test with incident sample.

## 10. Unclear Branch

Send `Email` to admin.

Subject:

```text
AI Could Not Classify Submission
```

Body:

```text
Review this entry:

{{$json.raw_notes}}
```

Steps:
1. From `Unclear` output, add `Email`.
2. Enter admin email.
3. Set subject.
4. Set body.
5. Test with unclear sample.

## 11. Error Handling

Create second workflow:

Error Trigger  
→ Email Admin

Subject:

```text
n8n Montessori Workflow Error
```

Body include:
- workflow name
- failed node
- error message

Steps:
1. Click `Create Workflow`.
2. Rename it to `Tiny Talents Montessori Error Handler`.
3. Add `Error Trigger`.
4. Add `Email`.
5. Set subject to `n8n Montessori Workflow Error`.
6. In body, include:
   - `Workflow: {{$json.workflow.name}}`
   - `Failed Node: {{$json.execution.lastNodeExecuted}}`
   - `Error: {{$json.execution.error.message}}`
7. Save and activate this workflow.

## 12. Test Data

Use these samples:

### Test 1

`Sam shared blocks today and helped clean up.`

Expected:  
Milestone

### Test 2

`Lina fell while running and bumped knee, cried 3 mins.`

Expected:  
Incident

### Test 3

`Not sure what happened after lunch.`

Expected:  
Unclear

## 13. Final Check

Make sure:
- Form works
- AI returns JSON
- Switch routes correctly
- Airtable saves rows
- Alerts send
- Admin gets unclear items

## 14. Export

How to submit:
- Export workflow JSON
- Screenshot canvas
- Screenshot successful Milestone run
- Screenshot successful Incident run