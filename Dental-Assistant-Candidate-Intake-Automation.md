# Dental Assistant Candidate Intake Automation Guide

## 1. Recommended Build

This guide shows you how to build the workflow manually in n8n.

For the trigger, the most reliable production option is `Webhook` because it is available in every n8n instance and gives you predictable field names.

If your n8n version includes `Google Forms Trigger`, you can use it instead and keep the rest of the workflow the same. The only part you will usually need to adjust is the field mapping inside the `Prepare Candidate Data` node.

## 2. Workflow Goal

When a candidate submits the intake form, the workflow should:

1. Capture applicant details
2. Clean the data
3. Create a record in Airtable
4. Send a confirmation email from Gmail
5. Notify the hiring team in Slack

## 3. Node Order

`Candidate Form Trigger`  
-> `Prepare Candidate Data`  
-> `Save to Airtable`  
-> `Send Confirmation Email`  
-> `Notify Hiring Team`

## 4. Create the Workflow

1. Log in to n8n.
2. Click `Create Workflow`.
3. Rename the workflow to `Dental Assistant Candidate Intake Automation`.
4. Click `Save`.

## 5. Build Each Node

### Candidate Form Trigger

Purpose:  
Start the workflow when a new application is submitted.

Recommended node:  
`Webhook`

Add Node:  
Click `+` -> search `Webhook`.

Rename it to:  
`Candidate Form Trigger`

Settings:

- `HTTP Method`: `POST`
- `Path`: `dental-assistant-candidate-intake`
- `Authentication`: `None` or your preferred webhook auth
- If your n8n version shows `Response Mode`, choose `On Received`

Expected request body:

```json
{
  "full_name": "Sarah Johnson",
  "email": "sarah@email.com",
  "years_experience": "4",
  "certification_status": "Certified Dental Assistant"
}
```

How to test:

1. Click `Listen for Test Event`.
2. Send a test `POST` request to the webhook test URL.
3. Confirm the execution shows the four fields above.

### Google Forms Trigger Alternative

If your n8n instance includes `Google Forms Trigger`, you can use it instead of `Webhook`.

Rename the node to:

`Candidate Form Trigger`

Recommended setup:

- Connect your Google account
- Select the recruitment form
- Use form questions named:
  - `Full Name`
  - `Email`
  - `Years of Experience`
  - `Certification Status`

Important:

Google Forms output keys can vary by n8n version and form structure. After your first test submission, open the execution data and use the exact response keys you see in the next node.

### Prepare Candidate Data

Purpose:  
Normalize the form data and derive `first_name`.

Add Node:  
Click `+` after `Candidate Form Trigger` -> search `Set`.

Rename it to:  
`Prepare Candidate Data`

Settings:

- Turn on `Keep Only Set`

Add these fields:

- `full_name` = `{{ $json["full_name"] }}`
- `first_name` = `{{ (($json["full_name"] || "").trim().split(/\s+/))[0] || "" }}`
- `email` = `{{ $json["email"] }}`
- `years_experience` = `{{ $json["years_experience"] }}`
- `certification_status` = `{{ $json["certification_status"] }}`
- `submitted_at` = `{{ $now.toISO() }}`

If you are using `Google Forms Trigger`, replace the source expressions with the actual keys from your trigger output. A common mapping looks like this:

- `full_name` = `{{ $json["Full Name"] }}`
- `email` = `{{ $json["Email"] }}`
- `years_experience` = `{{ $json["Years of Experience"] }}`
- `certification_status` = `{{ $json["Certification Status"] }}`

Output from this node should look like:

```json
{
  "full_name": "Sarah Johnson",
  "first_name": "Sarah",
  "email": "sarah@email.com",
  "years_experience": "4",
  "certification_status": "Certified Dental Assistant",
  "submitted_at": "2026-04-21T09:00:00.000Z"
}
```

### Save to Airtable

Purpose:  
Create the candidate record in Airtable.

Add Node:  
Click `+` after `Prepare Candidate Data` -> search `Airtable`.

Rename it to:  
`Save to Airtable`

Settings:

- `Resource`: `Record`
- `Operation`: `Create`
- `Base`: `Candidates`
- `Table`: `Applicants`

Map the Airtable fields like this:

- `Full Name` = `{{ $json["full_name"] }}`
- `First Name` = `{{ $json["first_name"] }}`
- `Email` = `{{ $json["email"] }}`
- `Years of Experience` = `{{ $json["years_experience"] }}`
- `Certification Status` = `{{ $json["certification_status"] }}`
- `Submitted At` = `{{ $json["submitted_at"] }}`

### Send Confirmation Email

Purpose:  
Send the applicant a professional confirmation email immediately.

Add Node:  
Click `+` after `Save to Airtable` -> search `Gmail`.

Rename it to:  
`Send Confirmation Email`

Settings:

- `Resource`: `Message`
- `Operation`: `Send`
- `To` = `{{ $node["Prepare Candidate Data"].json["email"] }}`
- `Subject` = `Thank You for Applying - Thorne Oral Surgery & Implants`

Body:

```text
Hello {{ $node["Prepare Candidate Data"].json["first_name"] }},

Thank you for applying for the Dental Assistant opportunity with Thorne Oral Surgery & Implants.

We've successfully received your application and our hiring team will review it shortly.

Learn more about our team culture here:
https://example.com/new-hire-culture-video

We appreciate your interest and look forward to connecting with you.

Best regards,
HR Team
Thorne Oral Surgery & Implants
```

Why this expression style matters:

The Gmail node reads directly from `Prepare Candidate Data` instead of the Airtable node output. This avoids broken mappings if Airtable returns a different response shape than the original input.

### Notify Hiring Team

Purpose:  
Alert the internal hiring team in Slack.

Add Node:  
Click `+` after `Send Confirmation Email` -> search `Slack`.

Rename it to:  
`Notify Hiring Team`

Settings:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: `#hiring-alerts`

Message:

```text
New Candidate Application

Name: {{ $node["Prepare Candidate Data"].json["full_name"] }}
Experience: {{ $node["Prepare Candidate Data"].json["years_experience"] }} years
Certification: {{ $node["Prepare Candidate Data"].json["certification_status"] }}

Review in Airtable.
```

If you want to match the original visual style exactly, you can manually add a tooth emoji to the first line in the Slack message field inside n8n.

Note:

If your Slack node requires a channel selection instead of a typed name, choose the `#hiring-alerts` channel from the dropdown and make sure the Slack bot has been invited to that channel.

## 6. Connections

Connect the nodes in this exact order:

1. `Candidate Form Trigger` -> `Prepare Candidate Data`
2. `Prepare Candidate Data` -> `Save to Airtable`
3. `Save to Airtable` -> `Send Confirmation Email`
4. `Send Confirmation Email` -> `Notify Hiring Team`

## 7. Credential Setup

### Google Forms Trigger Credentials

Only needed if you use `Google Forms Trigger`.

1. In n8n, go to `Credentials`.
2. Create a new Google credential for the Google Forms node.
3. Sign in with the Google account that owns or can access the form.
4. Approve the requested permissions.
5. Return to the node and select the credential.
6. Select the target form.
7. Submit one test response to confirm n8n receives the data.

### Airtable Credentials

1. In Airtable, create a Personal Access Token.
2. Give it access to the `Candidates` base.
3. Include record permissions for read and write.
4. In n8n, go to `Credentials`.
5. Create a new `Airtable` credential.
6. Paste the Personal Access Token.
7. Save the credential.
8. In the `Save to Airtable` node, select that credential and verify you can see base `Candidates` and table `Applicants`.

### Gmail Credentials

1. In n8n, go to `Credentials`.
2. Create a new `Gmail OAuth2` credential.
3. Sign in with the mailbox that should send the confirmation emails.
4. Approve Gmail access.
5. Save the credential.
6. In the `Send Confirmation Email` node, select that Gmail credential.
7. Run a test execution and confirm the email is delivered.

### Slack Credentials

1. In n8n, go to `Credentials`.
2. Create a new `Slack` credential.
3. Connect the Slack workspace used by your hiring team.
4. Make sure the app has permission to post messages.
5. Save the credential.
6. In Slack, invite the connected app or bot user to `#hiring-alerts`.
7. In the `Notify Hiring Team` node, select the Slack credential and channel.

## 8. Airtable Schema Notes

Create this base and table before testing the workflow:

- Base: `Candidates`
- Table: `Applicants`

Recommended columns:

- `Full Name` -> Single line text
- `First Name` -> Single line text
- `Email` -> Email
- `Years of Experience` -> Single line text or Number
- `Certification Status` -> Single line text
- `Submitted At` -> Date and time

Recommendation:

If applicants will always submit numeric values such as `1`, `2`, or `4`, you can make `Years of Experience` a Number field. If you may receive values like `4+` or `Less than 1`, use Single line text instead.

## 9. Field Mapping Notes

| Workflow Field | Source | Used In |
| --- | --- | --- |
| `full_name` | Form submission | Airtable `Full Name`, Slack message |
| `first_name` | Derived from `full_name` | Gmail greeting, Airtable `First Name` |
| `email` | Form submission | Airtable `Email`, Gmail `To` |
| `years_experience` | Form submission | Airtable `Years of Experience`, Slack message |
| `certification_status` | Form submission | Airtable `Certification Status`, Slack message |
| `submitted_at` | Generated in n8n with `{{$now.toISO()}}` | Airtable `Submitted At` |

## 10. Final Checklist

Before activating the workflow, confirm:

1. The trigger receives all four applicant fields
2. `Prepare Candidate Data` outputs `first_name` correctly
3. Airtable creates a new record in `Candidates -> Applicants`
4. Gmail sends the confirmation email to the applicant
5. Slack posts the alert to `#hiring-alerts`
6. The workflow is switched to `Active`

## 11. Recommended Production Note

If you are choosing between `Google Forms Trigger` and `Webhook`, use `Webhook` for the most predictable build. Use `Google Forms Trigger` only if it is available in your n8n version and you have already confirmed the response keys during testing.
