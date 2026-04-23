# Automated Lead Dispatch for High-Voltage Heritage

## Detailed n8n Implementation Guide

## 1. Recommended Build Path

This guide shows you how to build the workflow manually in n8n without generating workflow JSON.

For a real, current, official n8n build, the safest implementation is:

- `Webhook`
- `Edit Fields (Set)` or `Set`
- `Date & Time`
- `Google Sheets`
- `Slack`

### Important reality check

As of `April 21, 2026`, I could verify official n8n documentation for:

- `Webhook`
- `Edit Fields (Set)`
- `Date & Time`
- `Google Sheets`
- `Slack`

I could **not** verify an official `Google Forms Trigger` node in the current n8n docs.

Because of that, this guide is built around `Webhook`, which is the real production-safe option for this project.

If your n8n instance somehow includes a custom or internal `Google Forms Trigger`, you can still use most of this guide, but you will need to adapt the first node and the field mappings in `Prepare Lead Data`.

## 2. What You Are Building

You are creating a lead intake workflow for `High-Voltage Heritage`, a contractor focused on historic home electrical restoration.

When someone submits the public `Historic Assessment Request` form, the workflow should:

1. Receive the submission immediately
2. Normalize the incoming fields into a clean lead record
3. Convert the submission timestamp into a readable date
4. Save the lead into a Google Sheet called `Master Lead Tracker`
5. Send a readable Slack alert to `#field-leads`

## 3. Why This Guide Uses Webhook Instead of Google Forms Trigger

This matters enough to be explicit.

If you must keep a native Google Form as the public form, and you want an instant workflow with only official built-in n8n nodes from your allowed list, the specification is not fully achievable as written.

Why:

- I could not verify an official `Google Forms Trigger` node in n8n docs
- Google Forms does not natively submit directly into an n8n workflow in the same way a custom web form can post to a webhook
- Routing Google Form submissions instantly into n8n usually requires something outside your allowed node list, such as Apps Script or a different trigger path

So this guide assumes one of these real-world intake options:

- your website form can send a `POST` request directly to the n8n webhook
- your web developer can update the public form action or frontend submission logic to point to the n8n webhook
- you replace the old public form with a webhook-backed form on your site

## 4. Final Workflow Shape

### Recommended production order

To reduce the risk of losing leads if Slack fails, I recommend this order:

`Historic Assessment Trigger`  
`-> Prepare Lead Data`  
`-> Format Inquiry Date`  
`-> Append to Master Lead Tracker`  
`-> Send Slack Lead Alert`

### Why I recommend Sheets before Slack

Your original outline listed Slack first and Sheets second.

That is valid, but if the Slack step fails and you have no separate error-handling branch, you may lose the lead before it is stored.

Since your business goal is to reduce missed leads, storing the record first is safer.

If you want to follow the original order exactly, you can swap the last two nodes after testing:

`Format Inquiry Date`  
`-> Send Slack Lead Alert`  
`-> Append to Master Lead Tracker`

Everything else in this guide stays the same.

## 5. Workflow Success Criteria

Your finished workflow is correct when all of the following are true:

- a public form submission creates a new n8n execution
- the incoming lead fields are normalized into consistent field names
- `raw_timestamp` becomes `formatted_date` in `MM/DD/YYYY`
- a new row appears in Google Sheets
- a message is posted in Slack
- the Slack alert clearly highlights `Property Age` and `Urgency Level`

## 6. Data Contract for the Public Form

Your public form should send these fields:

- `full_name`
- `email`
- `phone`
- `property_address`
- `property_age`
- `urgency_level`
- `project_notes`
- `submitted_at`

### Example JSON payload

```json
{
  "full_name": "James Carter",
  "email": "james@email.com",
  "phone": "555-1122",
  "property_address": "14 Oak Street",
  "property_age": "Built 1890",
  "urgency_level": "High",
  "project_notes": "Frequent breaker trips and exposed wiring.",
  "submitted_at": "2026-04-21T14:00:00Z"
}
```

### Best practice

Ask whoever owns the form frontend to send the payload exactly like the example above.

That will keep your n8n mapping simple and reliable.

## 7. Before You Open n8n

Prepare these assets first.

### A. Google Sheet

Create a spreadsheet named:

`Master Lead Tracker`

Create a worksheet tab named:

`Leads`

Create these column headers in row `1` exactly:

- `Date`
- `Full Name`
- `Email`
- `Phone`
- `Property Address`
- `Property Age`
- `Urgency Level`
- `Project Notes`

### B. Slack Channel

Create or confirm the channel:

`#field-leads`

Make sure the Slack app or bot that n8n uses is invited to the channel.

### C. A real public form endpoint

Your website or form tool must be able to send a `POST` request to the n8n webhook.

If your current form cannot do that, stop here and fix that part first. The n8n workflow can only react to what actually reaches it.

## 8. Credentials You Need

You need:

- one `Slack` credential in n8n
- one `Google Sheets` credential in n8n

The `Webhook` node itself does not need third-party credentials unless you choose webhook authentication.

## 9. Create the Workflow

1. Log in to n8n.
2. Click `Create Workflow`.
3. Rename the workflow to `Automated Lead Dispatch for High-Voltage Heritage`.
4. Click `Save`.

## 10. Add Node 1: Historic Assessment Trigger

### Purpose

Receive a lead submission from the public form.

### Add the node

1. Click `Add first step`.
2. Search for `Webhook`.
3. Add the `Webhook` node.
4. Rename it to `Historic Assessment Trigger`.

### Configure the node

Use these settings:

- `HTTP Method`: `POST`
- `Path`: `historic-assessment-request`
- `Authentication`: `None` for initial setup
- `Respond`: `Immediately`

### Why `Respond: Immediately`

This returns a response as soon as the workflow starts, which is usually friendlier for public forms and frontend submit handlers.

If you need a custom response body later, that usually involves `Respond to Webhook`, which is outside your allowed node list for this project.

### Optional webhook hardening later

After the base flow works, you can revisit:

- `Basic auth`
- `Header auth`
- CORS restrictions
- IP allow lists

For your first build, keep it simple and get the data flowing.

### What the webhook outputs

In most standard webhook executions, the submitted JSON body is available under:

`$json.body`

That is why the next node will map from expressions like:

`{{ $json["body"]["full_name"] }}`

## 11. Test the Trigger Before Building More

1. Open `Historic Assessment Trigger`.
2. Click `Listen for Test Event` or `Execute Workflow`, depending on your n8n version.
3. Send a `POST` request to the test webhook URL from your form or from a testing tool.
4. Confirm the execution shows the body fields.

### Minimum test payload

```json
{
  "full_name": "James Carter",
  "email": "james@email.com",
  "phone": "555-1122",
  "property_address": "14 Oak Street",
  "property_age": "Built 1890",
  "urgency_level": "High",
  "project_notes": "Frequent breaker trips and exposed wiring.",
  "submitted_at": "2026-04-21T14:00:00Z"
}
```

Do not continue until this works.

## 12. Add Node 2: Prepare Lead Data

### Purpose

Normalize the incoming submission into a clean internal record so every later node uses the same field names.

### Add the node

1. Click the `+` after `Historic Assessment Trigger`.
2. Search for `Set` or `Edit Fields`.
3. Add the node.
4. Rename it to `Prepare Lead Data`.

### Recommended settings

- `Mode`: `Manual Mapping`
- `Keep Only Set Fields`: `On`

### Why manual mapping

It is slower to set up than automatic mapping, but it gives you a much more stable workflow and makes the downstream Google Sheets and Slack steps easier to maintain.

### Create these fields exactly

Add these output fields one by one.

#### Field 1

- `Name`: `full_name`
- `Value`:

```text
{{ $json["body"]["full_name"] || $json["body"]["Full Name"] || $json["full_name"] || $json["Full Name"] || "" }}
```

#### Field 2

- `Name`: `email`
- `Value`:

```text
{{ $json["body"]["email"] || $json["body"]["Email"] || $json["email"] || $json["Email"] || "" }}
```

#### Field 3

- `Name`: `phone`
- `Value`:

```text
{{ $json["body"]["phone"] || $json["body"]["Phone"] || $json["phone"] || $json["Phone"] || "" }}
```

#### Field 4

- `Name`: `property_address`
- `Value`:

```text
{{ $json["body"]["property_address"] || $json["body"]["Property Address"] || $json["property_address"] || $json["Property Address"] || "" }}
```

#### Field 5

- `Name`: `property_age`
- `Value`:

```text
{{ $json["body"]["property_age"] || $json["body"]["Property Age"] || $json["property_age"] || $json["Property Age"] || "" }}
```

#### Field 6

- `Name`: `urgency_level`
- `Value`:

```text
{{ $json["body"]["urgency_level"] || $json["body"]["Urgency Level"] || $json["urgency_level"] || $json["Urgency Level"] || "" }}
```

#### Field 7

- `Name`: `project_notes`
- `Value`:

```text
{{ $json["body"]["project_notes"] || $json["body"]["Project Notes"] || $json["project_notes"] || $json["Project Notes"] || "" }}
```

#### Field 8

- `Name`: `raw_timestamp`
- `Value`:

```text
{{ $json["body"]["submitted_at"] || $json["body"]["Submitted At"] || $json["body"]["timestamp"] || $json["body"]["submittedAt"] || $json["submitted_at"] || $json["Submitted At"] || $json["timestamp"] || $json["submittedAt"] || "" }}
```

### Why the expressions look long

They intentionally support multiple possible input styles:

- snake_case keys
- title-case keys
- direct JSON
- webhook body JSON

That gives you some protection if the form developer changes the frontend naming format.

### Expected output of this node

After `Prepare Lead Data`, you should see JSON like this:

```json
{
  "full_name": "James Carter",
  "email": "james@email.com",
  "phone": "555-1122",
  "property_address": "14 Oak Street",
  "property_age": "Built 1890",
  "urgency_level": "High",
  "project_notes": "Frequent breaker trips and exposed wiring.",
  "raw_timestamp": "2026-04-21T14:00:00Z"
}
```

## 13. Test Node 2 Carefully

Execute `Prepare Lead Data` and check for these exact things:

- every field exists
- field names are lowercase snake_case
- no values are accidentally blank
- `raw_timestamp` contains a real date value

If any field is empty:

1. open the input view from `Historic Assessment Trigger`
2. inspect the exact key name the webhook received
3. adjust the expression in `Prepare Lead Data`

This is one of the most important debugging steps in the whole build.

## 14. Add Node 3: Format Inquiry Date

### Purpose

Convert the raw submission timestamp into a readable date for sheets and Slack.

### Add the node

1. Click the `+` after `Prepare Lead Data`.
2. Search for `Date & Time`.
3. Add the node.
4. Rename it to `Format Inquiry Date`.

### Configure the node

Use these settings:

- `Operation`: `Format a Date`
- `Date`: `{{ $json["raw_timestamp"] }}`
- `Format`: `MM/DD/YYYY`
- `Output Field Name`: `formatted_date`

### Important options

Turn this on:

- `Include Input Fields`

This matters because you need all of the original lead fields to continue downstream.

### What this node should output

You should now have all previous lead fields plus:

```json
{
  "formatted_date": "04/21/2026"
}
```

## 15. Test Node 3

Run `Format Inquiry Date` and confirm:

- `formatted_date` exists
- the format is `MM/DD/YYYY`
- example output looks like `04/21/2026`

If the node fails:

- the incoming timestamp may be blank
- the timestamp may be in a non-standard format
- the form may not be sending the field at all

If needed, fix the form so it sends ISO timestamps, for example:

`2026-04-21T14:00:00Z`

## 16. Add Node 4: Append to Master Lead Tracker

### Purpose

Store every inquiry in Google Sheets before sending the Slack alert.

### Add the node

1. Click the `+` after `Format Inquiry Date`.
2. Search for `Google Sheets`.
3. Add the node.
4. Rename it to `Append to Master Lead Tracker`.

### Configure the node

Use these settings:

- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: `Master Lead Tracker`
- `Sheet`: `Leads`
- `Mapping Column Mode`: `Map Each Column Manually`

### Why manual mapping here

Your sheet uses human-readable column names with spaces.

Your workflow uses internal snake_case field names.

Manual mapping removes ambiguity and makes the final sheet stable.

### Map the columns exactly like this

| Google Sheets Column | Value to Send |
| --- | --- |
| `Date` | `{{ $json["formatted_date"] }}` |
| `Full Name` | `{{ $json["full_name"] }}` |
| `Email` | `{{ $json["email"] }}` |
| `Phone` | `{{ $json["phone"] }}` |
| `Property Address` | `{{ $json["property_address"] }}` |
| `Property Age` | `{{ $json["property_age"] }}` |
| `Urgency Level` | `{{ $json["urgency_level"] }}` |
| `Project Notes` | `{{ $json["project_notes"] }}` |

### Recommended options

If your sheet is clean and does not contain blank broken ranges, you can turn on:

- `Use Append`

If you are unsure, leave it off at first.

### Notes for sheet stability

Do not rename columns after the node is configured unless you plan to refresh the mapping.

If you later edit sheet header names, reopen the Google Sheets node and refresh the mapping.

## 17. Test Node 4

Execute `Append to Master Lead Tracker` and confirm:

- a new row appears in `Master Lead Tracker`
- the row lands in the `Leads` worksheet
- the `Date` column uses `formatted_date`
- the `Property Age` and `Urgency Level` columns show the expected values

If the row does not appear:

- verify the credential is connected
- verify the correct spreadsheet is selected
- verify the correct worksheet tab is selected
- verify the bot or credentialed account can edit the sheet

## 18. Add Node 5: Send Slack Lead Alert

### Purpose

Notify field technicians in Slack with a clean, readable message.

### Add the node

1. Click the `+` after `Append to Master Lead Tracker`.
2. Search for `Slack`.
3. Add the node.
4. Rename it to `Send Slack Lead Alert`.

### Slack action

Choose the Slack action in your n8n version that posts a message into a channel.

Depending on your version, this may appear as a message-posting action with fields like:

- channel
- text

### Channel

Set the target channel to:

`#field-leads`

If your Slack node requires selecting from a list, choose the channel from the dropdown.

If it requires manual entry, enter the channel name exactly as your workspace expects it.

### Message body

Paste this into the Slack text or message field:

```text
:zap: New Historic Assessment Lead

Client: {{ $json["full_name"] }}
Phone: {{ $json["phone"] }}
Address: {{ $json["property_address"] }}

:house_abandoned: Property Age: {{ $json["property_age"] }}
:rotating_light: Urgency Level: {{ $json["urgency_level"] }}

Notes:
{{ $json["project_notes"] }}

Submitted:
{{ $json["formatted_date"] }}
```

### Why I used Slack emoji shortcodes

They stay ASCII-safe in this guide and render well in Slack:

- `:zap:`
- `:house_abandoned:`
- `:rotating_light:`

### Goal of the message layout

The alert is intentionally structured so technicians can scan it top to bottom:

1. lead headline
2. contact basics
3. address
4. age and urgency
5. notes
6. submitted date

## 19. Test Node 5

Execute `Send Slack Lead Alert` and confirm:

- the message appears in `#field-leads`
- `Property Age` is clearly visible
- `Urgency Level` is clearly visible
- line breaks are preserved

If the message fails:

- confirm the Slack credential is connected
- confirm the app is installed in the correct workspace
- confirm the app has permission to post messages
- confirm the bot is invited to `#field-leads`

## 20. Connect the Workflow in Final Order

Your final recommended connection order should be:

`Historic Assessment Trigger`  
`-> Prepare Lead Data`  
`-> Format Inquiry Date`  
`-> Append to Master Lead Tracker`  
`-> Send Slack Lead Alert`

## 21. Save and Publish

Once all nodes work in testing:

1. Click `Save`.
2. Publish or activate the workflow.
3. Copy the production webhook URL from `Historic Assessment Trigger`.
4. Update your public form so production submissions go to that production URL.

Do not leave your live form pointed at the test URL.

## 22. Setup Guide for Slack Credentials

## Recommended credential method

Use `OAuth2` for the Slack node if possible.

The official n8n Slack credential docs recommend OAuth2 for the Slack node and note that API access token works but is not the preferred method for this node.

### If you are using n8n Cloud

Usually this is the easiest path:

1. Add a new `Slack` credential in n8n.
2. Choose the OAuth-based connection flow.
3. Click the connect button.
4. Sign in to Slack.
5. Authorize the app for the correct workspace.
6. Save the credential.

### If you are self-hosting n8n

Use this process:

1. Open the Slack API apps page.
2. Create a new Slack app.
3. Choose `From scratch`.
4. Give the app a name like `n8n High-Voltage Heritage Alerts`.
5. Select the correct Slack workspace.
6. In Slack app settings, open `Basic Information`.
7. Copy the `Client ID` and `Client Secret`.
8. In n8n, create a Slack credential and paste both values.
9. In the Slack app, open `OAuth & Permissions`.
10. Copy the OAuth callback URL shown by n8n.
11. Add that callback URL as a redirect URL in Slack.
12. Add scopes.
13. Install the app to the workspace.
14. Complete the connection in n8n.

### Minimum useful Slack permissions

At a minimum, ensure the app can post messages to channels.

The official n8n Slack credential docs list `chat:write` among the useful scopes.

### Very common production problem

Do not use a Slack app with token rotation enabled for a production credential unless you fully understand the consequences.

The current n8n docs note that token rotation can cause credentials to expire after `12` hours.

### Final Slack checks

Before go-live:

- the credential is connected
- the bot is installed to the correct workspace
- the bot is invited to `#field-leads`
- a manual test message from the Slack node succeeds

## 23. Setup Guide for Google Sheets Credentials

## Recommended credential method

Use `OAuth2` for Google Sheets unless you have a strong reason to use a service account.

The current n8n Google credential docs recommend OAuth2 because it is more widely available and easier to set up.

### If you are using n8n Cloud

This is usually the simplest path:

1. Create a new Google credential in n8n for Google Sheets.
2. Use `Managed OAuth2` if it is available in your n8n Cloud account.
3. Click `Sign in with Google`.
4. Choose the correct Google account.
5. Approve access.
6. Save the credential.

### If you are self-hosting n8n

Use Custom OAuth2:

1. Open Google Cloud Console.
2. Create a project.
3. Enable the required APIs.
4. Make sure `Google Sheets API` is enabled.
5. Also enable `Google Drive API` because the current n8n docs note that Google Sheets requires it as well.
6. Configure the OAuth consent screen.
7. Add the domain of your n8n instance as an authorized domain.
8. Create OAuth client credentials for a `Web application`.
9. Copy the OAuth redirect URL from n8n.
10. Paste it into Google Cloud as an authorized redirect URI.
11. Copy the Google `Client ID` and `Client Secret`.
12. Paste them into the n8n Google credential.
13. Click `Sign in with Google` in n8n.
14. Save the credential.

### Final Google Sheets checks

Before go-live:

- the credential is connected
- the account can open `Master Lead Tracker`
- the account can edit the `Leads` worksheet
- a manual append from the Google Sheets node succeeds

## 24. Trigger Setup Notes

Because this build uses `Webhook`, there are no Google Forms trigger credentials to set up in the official build path.

Instead, your trigger setup work is:

1. build the webhook node
2. test with the webhook test URL
3. publish the workflow
4. update your public form to send production submissions to the production webhook URL

## 25. Field Mapping Notes

This section is the exact internal field model for the workflow.

| Internal Field | Comes From | Used In |
| --- | --- | --- |
| `full_name` | Webhook body or direct input | Slack, Google Sheets |
| `email` | Webhook body or direct input | Google Sheets |
| `phone` | Webhook body or direct input | Slack, Google Sheets |
| `property_address` | Webhook body or direct input | Slack, Google Sheets |
| `property_age` | Webhook body or direct input | Slack, Google Sheets |
| `urgency_level` | Webhook body or direct input | Slack, Google Sheets |
| `project_notes` | Webhook body or direct input | Slack, Google Sheets |
| `raw_timestamp` | Webhook body or direct input | Date & Time node |
| `formatted_date` | Date & Time node output | Slack, Google Sheets |

## 26. Google Sheets Column Mapping Notes

Use this exact mapping in `Append to Master Lead Tracker`.

| Sheet Column | Workflow Expression |
| --- | --- |
| `Date` | `{{ $json["formatted_date"] }}` |
| `Full Name` | `{{ $json["full_name"] }}` |
| `Email` | `{{ $json["email"] }}` |
| `Phone` | `{{ $json["phone"] }}` |
| `Property Address` | `{{ $json["property_address"] }}` |
| `Property Age` | `{{ $json["property_age"] }}` |
| `Urgency Level` | `{{ $json["urgency_level"] }}` |
| `Project Notes` | `{{ $json["project_notes"] }}` |

## 27. Slack Field Mapping Notes

Use these workflow values in the Slack message:

| Slack Section | Expression |
| --- | --- |
| Client | `{{ $json["full_name"] }}` |
| Phone | `{{ $json["phone"] }}` |
| Address | `{{ $json["property_address"] }}` |
| Property Age | `{{ $json["property_age"] }}` |
| Urgency Level | `{{ $json["urgency_level"] }}` |
| Notes | `{{ $json["project_notes"] }}` |
| Submitted | `{{ $json["formatted_date"] }}` |

## 28. Full End-to-End Test Plan

Run these tests one by one.

### Test 1: Standard lead

Payload:

```json
{
  "full_name": "James Carter",
  "email": "james@email.com",
  "phone": "555-1122",
  "property_address": "14 Oak Street",
  "property_age": "Built 1890",
  "urgency_level": "High",
  "project_notes": "Frequent breaker trips and exposed wiring.",
  "submitted_at": "2026-04-21T14:00:00Z"
}
```

Expected result:

- webhook runs
- Google Sheets gets one new row
- Slack gets one alert
- `formatted_date` equals `04/21/2026`

### Test 2: Medium urgency lead

Change only:

`"urgency_level": "Medium"`

Expected result:

- workflow still runs
- Google Sheets still logs the lead
- Slack still alerts the team
- the urgency line displays `Medium`

### Test 3: Historic property with long notes

Use a long `project_notes` value with two or three sentences.

Expected result:

- notes appear fully in Google Sheets
- Slack message remains readable
- line breaks are not collapsed unexpectedly

### Test 4: Alternate field labels

Submit title-case keys instead of snake_case:

```json
{
  "Full Name": "Maria Ellis",
  "Email": "maria@email.com",
  "Phone": "555-8811",
  "Property Address": "77 Willow Avenue",
  "Property Age": "Built 1912",
  "Urgency Level": "High",
  "Project Notes": "Panel appears outdated and lights flicker in the upstairs hallway.",
  "Submitted At": "2026-04-21T18:30:00Z"
}
```

Expected result:

- `Prepare Lead Data` still maps everything correctly

### Test 5: Missing timestamp

Submit a payload with no `submitted_at`.

Expected result:

- `Prepare Lead Data` works
- `Format Inquiry Date` likely fails or produces no usable date

This is a good failure because it proves the timestamp is truly required for the current design.

### Test 6: Slack permission issue

Temporarily point the Slack node to a channel the bot cannot post to.

Expected result:

- Google Sheets row still exists if you kept the recommended order
- Slack node fails

This is exactly why I recommend Sheets before Slack.

## 29. Validation Checklist

Do not call the build complete until all of these are true.

- [ ] The workflow is saved
- [ ] The workflow is published
- [ ] The form uses the production webhook URL
- [ ] The webhook receives `POST` requests
- [ ] `Prepare Lead Data` outputs all eight normalized fields
- [ ] `Format Inquiry Date` outputs `formatted_date`
- [ ] `Master Lead Tracker` exists
- [ ] `Leads` worksheet exists
- [ ] Google Sheets mapping is correct
- [ ] Slack credential is connected
- [ ] Slack app can post to `#field-leads`
- [ ] A real submission creates both a sheet row and a Slack alert

## 30. Common Build Mistakes

### Mistake 1: Using the test webhook URL in production

Symptom:

- manual tests work
- live website submissions do not

Fix:

- publish the workflow
- switch the form to the production URL

### Mistake 2: Field names do not match

Symptom:

- `Prepare Lead Data` outputs blank fields

Fix:

- inspect the webhook payload
- update the expressions to match the real incoming keys

### Mistake 3: Google Sheets headers changed after setup

Symptom:

- append node errors
- mapping no longer matches

Fix:

- refresh the Google Sheets mapping
- confirm the sheet headers still use the intended names

### Mistake 4: Slack app is not in the channel

Symptom:

- the Slack credential connects
- the message still fails to post

Fix:

- invite the Slack bot or app to `#field-leads`

### Mistake 5: Timestamp format is inconsistent

Symptom:

- the date node fails
- `formatted_date` is blank

Fix:

- send ISO timestamps from the form
- example: `2026-04-21T14:00:00Z`

## 31. Production Go-Live Sequence

Use this order on launch day.

1. Confirm Google Sheets and Slack credentials are connected.
2. Confirm the `Leads` worksheet column headers are exact.
3. Confirm the workflow is published.
4. Confirm the public form points to the production webhook URL.
5. Submit one real internal test lead.
6. Confirm the row appears in Google Sheets.
7. Confirm the Slack alert appears in `#field-leads`.
8. Submit one more test with a different urgency level.
9. Confirm both records are stored correctly.
10. Only then announce the workflow as live.

## 32. Recommended Operating Notes

These are small choices that improve survivability in production.

- Keep the normalized field names in snake_case forever.
- Avoid renaming Google Sheets columns casually.
- Save the workflow after every stable milestone.
- Re-test the whole flow after any credential reconnect.
- If the form frontend changes, inspect the webhook payload again before trusting the live system.

## 33. If You Insist on Matching the Original Slack-Then-Sheets Order

You can do that.

Use this connection sequence instead:

`Historic Assessment Trigger`  
`-> Prepare Lead Data`  
`-> Format Inquiry Date`  
`-> Send Slack Lead Alert`  
`-> Append to Master Lead Tracker`

But understand the tradeoff:

- if Slack fails first, the lead may never be stored

That is why I do not recommend that order for production if missed leads are the core business risk.

## 34. Final Build Summary

The real, current, official n8n build path for this project is:

- `Webhook` as the trigger
- `Set` to normalize lead fields
- `Date & Time` to create `formatted_date`
- `Google Sheets` to append every lead to `Master Lead Tracker -> Leads`
- `Slack` to alert `#field-leads`

This gives you a workflow that is honest, import-free, manually buildable, and aligned with currently documented n8n capabilities.

## 35. Official Reference Links

- Webhook: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- Edit Fields (Set): https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
- Date & Time: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.datetime/
- Google Sheets: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- Google Sheets sheet operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/sheet-operations/
- Slack node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- Slack credentials: https://docs.n8n.io/integrations/builtin/credentials/slack/
- Google credentials: https://docs.n8n.io/integrations/builtin/credentials/google/
- Google OAuth2 single service: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/
- Expressions for data transformation: https://docs.n8n.io/data/expressions-for-transformation/
