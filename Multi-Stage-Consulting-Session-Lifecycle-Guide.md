# Multi-Stage Consulting Session Lifecycle Guide

Prepared for: Pivot Strategy Group  
Document type: n8n implementation guide  
Purpose: Clear step-by-step instructions to build the full consulting-session app inside n8n

## 1. What You Are Building

This app watches a Google Calendar for client strategy sessions, checks the client in Airtable, sends the right internal Slack alert, creates a session log, waits until two hours after the meeting ends, and then sends a Gmail follow-up.

The workflow should do all of this:

1. Detect a strategy session from Google Calendar.
2. Ignore anything that is not a real client strategy session.
3. Extract the client name from the event title.
4. Look up the client in Airtable.
5. Decide whether the client is:
   - matched and healthy
   - matched but low on hours
   - matched but inactive
   - not found
6. Send the correct Slack notification.
7. Create a `Session Notes` record in Airtable.
8. Wait until `session end time + 2 hours`.
9. Send a Gmail follow-up only when the client is valid and active.
10. Update Airtable to mark the follow-up as sent.

## 2. Final Workflow Shape

Build this main workflow:

`Trigger - Strategy Session Started`  
`-> Normalize - Event Data`  
`-> Route - Strategy Event?`

`False` branch: stop

`True` branch:  
`-> Extract - Client Name`  
`-> Lookup - Retainer Record`  
`-> Combine - Retainer Context`  
`-> Route - Client Match Found?`

`No Match` branch:  
`-> Alert - Slack Unresolved Client`  
`-> Create - Session Note`  
`-> Route - Send Follow-Up?`  
`-> End`

`Match Found` branch:  
`-> Validate - Contract Active?`

`Inactive` branch:  
`-> Alert - Slack Inactive Contract`  
`-> Create - Session Note`  
`-> Route - Send Follow-Up?`  
`-> End`

`Active` branch:  
`-> Route - Low Balance?`

`Healthy` branch:  
`-> Alert - Slack Healthy Balance`  
`-> Create - Session Note`

`Low Balance` branch:  
`-> Alert - Slack Low Balance`  
`-> Create - Session Note`

From `Create - Session Note`:  
`-> Route - Send Follow-Up?`

`Yes` branch:  
`-> Wait - Two Hours After Session End`  
`-> Send - Gmail Follow-Up`  
`-> Update - Session Note Follow-Up Sent`

`No` branch: end

Create a second workflow for errors:

`Error Trigger - Consulting Session Workflow`  
`-> Alert - Slack Workflow Error`

## 3. Build This First

Before touching the n8n canvas, prepare these systems.

### Accounts

- n8n access with permission to create credentials and activate workflows
- Google account with access to the consulting calendar
- Airtable workspace access
- Slack access or Slack app access
- Gmail account for client follow-up emails

### Credentials to create in n8n

- Google Calendar credential
- Airtable credential
- Slack credential
- Gmail credential

### Business rules to confirm

- Calendar to monitor
- Standard title format for strategy sessions
- Low-balance threshold: `< 2 hours`
- Slack channel for normal session alerts
- Slack channel for exception alerts
- Gmail sender name and sender mailbox
- Follow-up email wording
- Feedback form link or next-step link to include in the email

## 4. Airtable Structure

Create these two Airtable tables first.

### Table 1: `Retainer Log`

Required fields:

| Field Name | Type | Purpose |
|---|---|---|
| `Client Name` | Single line text | Canonical client name |
| `Normalized Client Key` | Formula or single line text | Lowercase trimmed lookup value |
| `Remaining Hours` | Number | Retainer balance |
| `Contract Status` | Single select | `Active`, `Paused`, `Closed` |
| `Email` | Email | Follow-up recipient |
| `Slack Contact` | Single line text | Optional internal owner |
| `Notes` | Long text | Account notes |

Recommended formula for `Normalized Client Key`:

```text
LOWER(TRIM({Client Name}))
```

Sample rows:

| Client Name | Normalized Client Key | Remaining Hours | Contract Status | Email |
|---|---|---|---|---|
| `Acme Corp` | `acme corp` | `5` | `Active` | `ops@acme.com` |
| `BrightPath LLC` | `brightpath llc` | `1` | `Active` | `team@brightpath.com` |
| `Northwind Labs` | `northwind labs` | `0` | `Paused` | `hello@northwindlabs.com` |

### Table 2: `Session Notes`

Required fields:

| Field Name | Type | Purpose |
|---|---|---|
| `Client Name` | Single line text | Matched or parsed client name |
| `Session Date` | Date with time | Meeting start time |
| `Duration` | Number | Hours |
| `Calendar Event URL` | URL | Link back to Google Calendar |
| `Follow-up Sent` | Checkbox or single select | `Yes` or `No` |
| `Workflow Status` | Single select | `Waiting`, `Follow-up Sent`, `Unresolved`, `Contract Exception` |
| `Match Status` | Single select | `Matched`, `No Client Match` |
| `Follow-up Sent At` | Date with time | When Gmail sends |
| `Notes` | Long text | Audit notes |

## 5. Suggested Build Order

If you want the fastest clean build, do it in this order:

1. Create the Airtable tables and sample rows.
2. Build the Google Calendar trigger.
3. Add the event cleanup node.
4. Add the `[STRAT]` filter.
5. Add the client-name extraction logic.
6. Add the Airtable lookup.
7. Add the match and contract-status branches.
8. Add the Slack alert branches.
9. Add the Session Notes creation node.
10. Add the follow-up decision, wait, Gmail, and Airtable update.
11. Build the separate error workflow.
12. Run the test cases at the end of this guide.

## 6. Main Workflow Step by Step

## 6.1 Create the Workflow

1. In n8n, click `Create Workflow`.
2. Rename it to `PSG - Consulting Session Lifecycle`.
3. Click `Save`.

## 6.2 Node 1: `Trigger - Strategy Session Started`

Use a `Google Calendar Trigger` node.

### Exact setup

| Field | Value |
|---|---|
| `Credential` | Your Google Calendar credential |
| `Calendar` | Your consulting calendar |
| `Event` | `Event Started` |

### Why use `Event Started`

This is the safest default because the workflow starts when the meeting is actually happening, not when it is first created on the calendar.

### Important note

Use timed events, not all-day events. This workflow expects a start and end time.

## 6.3 Node 2: `Normalize - Event Data`

Add an `Edit Fields (Set)` node after the trigger.

### Exact setup

| Field | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

### Paste into `JSON Output`

```json
{
  "event_id": "{{ $json.id || '' }}",
  "title": "{{ $json.summary || '' }}",
  "description": "{{ $json.description || '' }}",
  "start_time": "{{ $json.start?.dateTime || $json.start?.date || '' }}",
  "end_time": "{{ $json.end?.dateTime || $json.end?.date || '' }}",
  "event_url": "{{ $json.htmlLink || '' }}",
  "calendar_id": "{{ $json.organizer?.email || '' }}",
  "feedback_form_url": "https://your-feedback-form-url-here",
  "processed_at": "{{ $now }}"
}
```

### Why this node matters

Later nodes can now use simple field names like:

- `title`
- `start_time`
- `end_time`
- `event_url`
- `feedback_form_url`

## 6.4 Node 3: `Route - Strategy Event?`

Add an `IF` node after `Normalize - Event Data`.

### Rule

Only continue if the title starts with `[STRAT]`.

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `String` |
| `Value 1` | `{{ $json.title }}` |
| `Operation` | `startsWith` |
| `Value 2` | `[STRAT]` |

### Result

- `true` branch = continue
- `false` branch = stop

## 6.5 Node 4: `Extract - Client Name`

Add a `Code` node on the `true` branch.

### Exact settings

| Field | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for Each Item` |

### Paste this code

```javascript
const title = ($json.title || '').trim();
const stripped = title.replace(/^\[STRAT\]\s*/i, '');
const clientSegment = stripped.split('|')[0].trim().replace(/\s+/g, ' ');

const startTime = $json.start_time || '';
const endTime = $json.end_time || '';

const startMs = startTime ? new Date(startTime).getTime() : null;
const endMs = endTime ? new Date(endTime).getTime() : null;

const durationHours = startMs && endMs
  ? Number(((endMs - startMs) / (1000 * 60 * 60)).toFixed(2))
  : null;

const followUpAt = endMs
  ? new Date(endMs + (2 * 60 * 60 * 1000)).toISOString()
  : null;

return {
  json: {
    ...$json,
    client_name_clean: clientSegment,
    client_key_normalized: clientSegment.toLowerCase(),
    duration_hours: durationHours,
    follow_up_at: followUpAt,
  },
};
```

### What this node does

- removes the `[STRAT]` prefix
- trims extra spaces
- keeps only the client name before `|`
- calculates duration
- calculates the exact wait time for the follow-up

### Example

| Event Title | Output Client Name |
|---|---|
| `[STRAT] Acme Corp` | `Acme Corp` |
| `[STRAT] Acme Corp | Q2 Planning` | `Acme Corp` |

## 6.6 Node 5: `Lookup - Retainer Record`

Add an `Airtable` node after `Extract - Client Name`.

### Exact setup

| Field | Value |
|---|---|
| `Credential` | Your Airtable credential |
| `Resource` | `Record` |
| `Operation` | `List` |
| `Base` | Your Airtable base |
| `Table` | `Retainer Log` |
| `Return All` | `Off` |
| `Limit` | `1` |

### Required option

Turn `Always Output Data` on. This is important because the next node still needs to run even when Airtable finds no record.

### Filter formula

Switch `Filter By Formula` to expression mode and paste:

```text
{{ "{Normalized Client Key}='" + $json.client_key_normalized + "'" }}
```

## 6.7 Node 6: `Combine - Retainer Context`

Add a `Code` node after the Airtable lookup.

This node merges the cleaned event data with the Airtable result.

### Exact settings

| Field | Value |
|---|---|
| `Language` | `JavaScript` |
| `Mode` | `Run Once for Each Item` |

### Paste this code

```javascript
const base = $('Extract - Client Name').first().json;
const row = $json.fields ?? $json;

const clientFound = Boolean(($json.id || '') && (row['Client Name'] || ''));

return {
  json: {
    ...base,
    client_found: clientFound,
    retainer_record_id: clientFound ? ($json.id || '') : '',
    client_name: clientFound ? (row['Client Name'] || base.client_name_clean) : base.client_name_clean,
    remaining_hours: clientFound ? Number(row['Remaining Hours'] || 0) : null,
    contract_status: clientFound ? (row['Contract Status'] || '') : '',
    client_email: clientFound ? (row['Email'] || '') : '',
    slack_contact: clientFound ? (row['Slack Contact'] || '') : '',
  },
};
```

### Output fields to use later

- `client_found`
- `client_name`
- `remaining_hours`
- `contract_status`
- `client_email`
- `slack_contact`

## 6.8 Node 7: `Route - Client Match Found?`

Add an `IF` node after `Combine - Retainer Context`.

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `Boolean` |
| `Value 1` | `{{ $json.client_found }}` |
| `Operation` | `is true` |

### Result

- `true` = matched client
- `false` = no Airtable match

## 6.9 Node 8: `Alert - Slack Unresolved Client`

On the `false` branch from `Route - Client Match Found?`, add a `Slack` node.

### Exact settings

| Field | Value |
|---|---|
| `Resource` | `Message` |
| `Operation` | `Send` |
| `Channel` | Your exceptions channel |

### Message expression

```text
{{ `Unresolved session detected.

Parsed client: ${$json.client_name_clean}
Calendar title: ${$json.title}
Start time: ${$json.start_time}
Event URL: ${$json.event_url || 'No event URL returned'}

No Airtable retainer record was found. Manual review is required.` }}
```

## 6.10 Node 9: `Validate - Contract Active?`

On the `true` branch from `Route - Client Match Found?`, add an `IF` node.

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `String` |
| `Value 1` | `{{ $json.contract_status }}` |
| `Operation` | `equal` |
| `Value 2` | `Active` |

### Result

- `true` = contract is active
- `false` = contract is paused, closed, or otherwise invalid

## 6.11 Node 10: `Alert - Slack Inactive Contract`

On the `false` branch from `Validate - Contract Active?`, add a `Slack` node.

### Message expression

```text
{{ `Contract exception detected.

Client: ${$json.client_name}
Status: ${$json.contract_status || 'Unknown'}
Start time: ${$json.start_time}
Remaining hours: ${$json.remaining_hours ?? 'Unknown'}

Do not rely on automatic follow-up until this account is reviewed.` }}
```

## 6.12 Node 11: `Route - Low Balance?`

On the `true` branch from `Validate - Contract Active?`, add an `IF` node.

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `Number` |
| `Value 1` | `{{ $json.remaining_hours }}` |
| `Operation` | `smaller` |
| `Value 2` | `2` |

### Result

- `true` = low balance path
- `false` = healthy balance path

## 6.13 Node 12: `Alert - Slack Healthy Balance`

On the `false` branch from `Route - Low Balance?`, add a `Slack` node.

### Message expression

```text
{{ `Strategy session started for ${$json.client_name}.
Start time: ${$json.start_time}
Remaining hours: ${$json.remaining_hours}
Status: Healthy balance` }}
```

## 6.14 Node 13: `Alert - Slack Low Balance`

On the `true` branch from `Route - Low Balance?`, add a `Slack` node.

### Message expression

```text
{{ `Urgent: strategy session started for ${$json.client_name}.
Start time: ${$json.start_time}
Remaining hours: ${$json.remaining_hours}

This account is below the 2-hour threshold and should be reviewed.` }}
```

## 6.15 Node 14: `Create - Session Note`

Connect these three nodes into the same Airtable create node:

- `Alert - Slack Unresolved Client`
- `Alert - Slack Inactive Contract`
- `Alert - Slack Healthy Balance`
- `Alert - Slack Low Balance`

Use an `Airtable` node.

### Exact setup

| Field | Value |
|---|---|
| `Resource` | `Record` |
| `Operation` | `Create` |
| `Table` | `Session Notes` |

### Field mapping

| Airtable Field | Value |
|---|---|
| `Client Name` | `{{ $('Combine - Retainer Context').first().json.client_name || $('Combine - Retainer Context').first().json.client_name_clean }}` |
| `Session Date` | `{{ $('Combine - Retainer Context').first().json.start_time }}` |
| `Duration` | `{{ $('Combine - Retainer Context').first().json.duration_hours }}` |
| `Calendar Event URL` | `{{ $('Combine - Retainer Context').first().json.event_url }}` |
| `Follow-up Sent` | `No` |
| `Match Status` | `{{ $('Combine - Retainer Context').first().json.client_found ? 'Matched' : 'No Client Match' }}` |
| `Workflow Status` | `{{ !$('Combine - Retainer Context').first().json.client_found ? 'Unresolved' : ($('Combine - Retainer Context').first().json.contract_status !== 'Active' ? 'Contract Exception' : 'Waiting') }}` |

### Notes field expression

```text
{{ `Auto-created by n8n at session start.
Retainer match: ${$('Combine - Retainer Context').first().json.client_found ? 'Yes' : 'No'}
Remaining hours: ${$('Combine - Retainer Context').first().json.remaining_hours ?? 'Unknown'}
Contract status: ${$('Combine - Retainer Context').first().json.contract_status || 'Unknown'}
Follow-up target time: ${$('Combine - Retainer Context').first().json.follow_up_at || 'Not calculated'}` }}
```

## 6.16 Node 15: `Route - Send Follow-Up?`

Add an `IF` node after `Create - Session Note`.

Only allow the email branch when:

- a client was matched
- contract status is `Active`
- `client_email` is not empty

### Exact settings

| Field | Value |
|---|---|
| `Condition Type` | `Boolean` |
| `Value 1` | `{{ $('Combine - Retainer Context').first().json.client_found && $('Combine - Retainer Context').first().json.contract_status === 'Active' && !!$('Combine - Retainer Context').first().json.client_email }}` |
| `Operation` | `is true` |

### Result

- `true` = continue to wait and Gmail
- `false` = end

## 6.17 Node 16: `Wait - Two Hours After Session End`

Add a `Wait` node on the `true` branch.

### Exact setup

| Field | Value |
|---|---|
| `Mode` | `Wait Until Date and Time` |
| `Date and Time` | `{{ $('Combine - Retainer Context').first().json.follow_up_at }}` |

### Example

If the meeting ends at `2026-04-22T16:00:00-04:00`, the wait should resume at `2026-04-22T18:00:00-04:00`.

## 6.18 Node 17: `Send - Gmail Follow-Up`

Add a `Gmail` node after the wait.

### Exact settings

| Field | Value |
|---|---|
| `To` | `{{ $('Combine - Retainer Context').first().json.client_email }}` |
| `Subject` | `{{ "Thank you for today's strategy session - " + $('Combine - Retainer Context').first().json.client_name }}` |

### Body

Use HTML or plain text. This plain-text version is a safe starting point:

```text
Hi {{ $('Combine - Retainer Context').first().json.client_name }},

Thank you for today's strategy session with Pivot Strategy Group.

This is a quick follow-up to keep momentum moving. If you have any additional feedback, priorities, or follow-up items, please share them here:

{{ $('Combine - Retainer Context').first().json.feedback_form_url }}

We will use that input to help shape the next checkpoint and next actions.

Best,
Pivot Strategy Group
```

If your Gmail node expects expression mode for the entire body, paste this instead:

```text
{{ `Hi ${$('Combine - Retainer Context').first().json.client_name},

Thank you for today's strategy session with Pivot Strategy Group.

This is a quick follow-up to keep momentum moving. If you have any additional feedback, priorities, or follow-up items, please share them here:

${$('Combine - Retainer Context').first().json.feedback_form_url}

We will use that input to help shape the next checkpoint and next actions.

Best,
Pivot Strategy Group` }}
```

## 6.19 Node 18: `Update - Session Note Follow-Up Sent`

Add an `Airtable` update node after Gmail.

### Exact setup

| Field | Value |
|---|---|
| `Resource` | `Record` |
| `Operation` | `Update` |
| `Table` | `Session Notes` |
| `Record ID` | `{{ $('Create - Session Note').first().json.id }}` |

### Fields to update

| Airtable Field | Value |
|---|---|
| `Follow-up Sent` | `Yes` |
| `Workflow Status` | `Follow-up Sent` |
| `Follow-up Sent At` | `{{ $now }}` |

### Important note

This update node uses the record ID returned by `Create - Session Note`, so do not remove or overwrite that ID before the update step.

## 7. Error Workflow

Create a second workflow so you can see failures quickly.

### Workflow name

`PSG - Consulting Session Lifecycle Error Handler`

### Node order

`Error Trigger - Consulting Session Workflow`  
`-> Alert - Slack Workflow Error`

### Node 1

Add `Error Trigger`.

### Node 2

Add a `Slack` node named `Alert - Slack Workflow Error`.

### Slack message expression

```text
{{ `Workflow error detected.

Workflow: ${$json.workflow?.name || 'Unknown'}
Failed node: ${$json.execution?.lastNodeExecuted || 'Unknown'}
Error: ${$json.execution?.error?.message || 'No message returned'}
Execution ID: ${$json.execution?.id || 'Unavailable'}
Time: ${$now}` }}
```

### Final connection

In the main workflow:

1. Open workflow `Settings`.
2. Find `Error workflow`.
3. Select `PSG - Consulting Session Lifecycle Error Handler`.
4. Save.

## 8. Testing Plan

Run these tests before activation.

### Test 1: Healthy client

Calendar title:

```text
[STRAT] Acme Corp
```

Airtable row:

- `Remaining Hours = 5`
- `Contract Status = Active`
- email populated

Expected result:

- healthy Slack alert
- `Session Notes` record created
- wait resumes correctly
- Gmail follow-up sends
- `Follow-up Sent` updates to `Yes`

### Test 2: Low-balance client

Calendar title:

```text
[STRAT] BrightPath LLC
```

Airtable row:

- `Remaining Hours = 1`
- `Contract Status = Active`

Expected result:

- urgent Slack alert
- `Session Notes` record created
- Gmail still sends unless your policy changes that

### Test 3: Missing client

Calendar title:

```text
[STRAT] Unknown Advisory
```

Expected result:

- unresolved Slack alert
- `Session Notes` record created with `Match Status = No Client Match`
- no Gmail send

### Test 4: Inactive contract

Calendar title:

```text
[STRAT] Northwind Labs
```

Airtable row:

- `Contract Status = Paused`

Expected result:

- contract exception Slack alert
- `Session Notes` record created with `Workflow Status = Contract Exception`
- no Gmail send

### Test 5: Non-strategy event

Calendar title:

```text
Internal Ops Sync
```

Expected result:

- workflow stops at `Route - Strategy Event?`
- no Slack
- no Airtable row
- no Gmail

## 9. Debugging Checklist

Use this when something does not behave as expected.

| Problem | Likely Cause | Fix |
|---|---|---|
| Wait resumes at wrong time | Timezone mismatch | Re-test with known timestamps and confirm timezone settings |
| Airtable never matches | Normalized key mismatch | Check `client_key_normalized` and Airtable formula field |
| Gmail does not send | Missing email or expired auth | Reconnect Gmail and confirm `client_email` is populated |
| Slack alerts missing | Wrong channel or credential | Recheck Slack channel and credential |
| Session Notes created but update fails | Wrong `Record ID` in update node | Confirm the update node still receives the create-node output |
| Trigger fires on wrong meetings | Title rule is too weak | Enforce `[STRAT] Client Name` consistently |

## 10. Recommended Node Names

Use these names exactly or stay very close to them:

- `Trigger - Strategy Session Started`
- `Normalize - Event Data`
- `Route - Strategy Event?`
- `Extract - Client Name`
- `Lookup - Retainer Record`
- `Combine - Retainer Context`
- `Route - Client Match Found?`
- `Alert - Slack Unresolved Client`
- `Validate - Contract Active?`
- `Alert - Slack Inactive Contract`
- `Route - Low Balance?`
- `Alert - Slack Healthy Balance`
- `Alert - Slack Low Balance`
- `Create - Session Note`
- `Route - Send Follow-Up?`
- `Wait - Two Hours After Session End`
- `Send - Gmail Follow-Up`
- `Update - Session Note Follow-Up Sent`

## 11. Delivery Checklist

When the app is working, capture these items:

- screenshot of the full main workflow canvas
- screenshot of the error workflow
- one successful healthy-balance run
- one low-balance run
- one unresolved-client run
- Airtable `Retainer Log` sample rows
- Airtable `Session Notes` record after Gmail update
- Slack notification example
- Gmail follow-up example

## 12. Practical Improvements for Later

After the base app works, these upgrades are worth adding:

- automatically deduct session hours from the retainer
- add duplicate protection using calendar `event_id`
- store Slack message timestamps for audit
- add a dashboard for low-balance accounts
- support client-specific follow-up templates
- add a manual approval branch before sending email for risky accounts

## 13. Official References

Helpful product references:

- https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/
- https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
- https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/
- https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/
- https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/
- https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
- https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/
- https://docs.n8n.io/flow-logic/error-handling/
