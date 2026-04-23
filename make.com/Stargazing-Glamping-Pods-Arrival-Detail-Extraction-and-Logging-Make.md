# Stargazing Glamping Pods: Arrival Detail Extraction & Logging in Make.com

Prepared for: The Dark Sky Den  
Document type: Make.com implementation guide  
Project size: XS  
Purpose: Build a lightweight Make.com scenario that extracts guest arrival details from Gmail and logs them into Google Sheets, then sends a confirmation reply

## 1. Executive Summary

The Dark Sky Den is transitioning away from a manual booking and arrival workflow that currently depends too heavily on email. Important arrival details are being sent by guests, but they still need to be reviewed, copied, and logged by hand.

This Make.com scenario acts as a practical transition tool. It does not replace the full booking system. Instead, it captures arrival information from a controlled Gmail label or subject pattern, extracts the required fields, logs them into a central Google Sheet, and immediately replies to the guest with a confirmation message.

This gives the owner three immediate operational benefits:

- fewer manual copy-and-paste tasks
- a consistent arrival log for staff reference
- a simple guest confirmation loop that reduces uncertainty

Because this is an XS project, the build should stay intentionally lean and use a minimal scenario footprint:

1. Gmail trigger
2. Data parser or tools module
3. Google Sheets append row
4. Gmail reply

---

## 2. Project Scope

## Business Objective

Automatically process guest arrival emails and turn unstructured email content into a structured sheet row.

## Required Data Points

The scenario must extract:

- Guest Name
- Scheduled Arrival Time
- Dietary Restrictions
- Pod Number

## Required Systems

| System | Role |
|---|---|
| `Gmail` | Trigger source and reply channel |
| `Make.com` | Orchestration and parsing logic |
| `Google Sheets` | Central arrival tracking log |

## Required Output

Each valid email should result in:

1. one new row in Google Sheets
2. one confirmation reply email back to the sender

---

## 3. Solution Architecture

## Scenario Structure

This should remain an XS scenario with one trigger and three downstream modules.

`Gmail Watch Emails`  
`-> Tools / Text Parser`  
`-> Google Sheets Add a Row`  
`-> Gmail Send a Reply`

## Data Flow

`New email arrives with approved label or subject`  
`-> Gmail trigger collects subject, sender, body, and received timestamp`  
`-> Parser extracts Guest Name, Arrival Time, Dietary Restrictions, and Pod Number`  
`-> Google Sheets appends a new row with extracted values plus scenario timestamp`  
`-> Gmail replies to the guest confirming receipt`

## Trigger Rules

The scenario should only process emails that match one of these approved filters:

- a specific Gmail label such as `Pending Arrival Details`
- a specific subject pattern such as `Arrival Details`

## Recommended Trigger Strategy

Use a Gmail label if possible.

Why:

- more reliable than loose subject matching
- easier for the owner to control manually during the transition period
- reduces accidental processing of unrelated inbox traffic

---

## 4. Suggested Time Breakdown

This is an XS build, but it still needs proper testing and screenshot capture.

| Task | Hours |
|---|---:|
| Requirements review and scenario planning | 0.5 |
| Gmail and Google Sheets connection setup | 0.5 |
| Gmail trigger configuration | 0.5 |
| Email body format review and parsing setup | 0.75 |
| Google Sheets structure and field mapping | 0.5 |
| Gmail confirmation reply setup | 0.5 |
| Filter and empty dietary field handling | 0.5 |
| Testing with sample emails | 0.75 |
| Screenshot capture and handoff prep | 0.5 |
| **Total Estimated Effort** | **5.0** |

---

## 5. Pre-Build Preparation Checklist

- [ ] Make.com access with permission to create scenarios
- [ ] Gmail account connected in Make.com
- [ ] Google Sheets account connected in Make.com
- [ ] Gmail label created, such as `Pending Arrival Details`
- [ ] Destination Google Sheet created
- [ ] Header row added to the sheet
- [ ] At least three sample emails prepared for testing
- [ ] Consistent email body format confirmed

## Required Google Sheet Columns

Create a sheet with these columns:

| Column | Purpose |
|---|---|
| `Guest Name` | Primary guest identifier |
| `Arrival Time` | Scheduled arrival time from email |
| `Dietary Info` | Dietary restrictions or blank value |
| `Pod Number` | Accommodation unit reference |
| `Logged At` | Timestamp when scenario added the row |

---

## 6. Expected Email Format

This scenario assumes a consistent plain-text email layout.

## Recommended Email Template

```text
Guest Name: Amelia Hart
Scheduled Arrival Time: 7:30 PM
Dietary Restrictions: Vegetarian
Pod Number: Pod 3
```

## Acceptable Variation

`Dietary Restrictions` may be empty:

```text
Guest Name: Noah Ellis
Scheduled Arrival Time: 6:00 PM
Dietary Restrictions:
Pod Number: Pod 1
```

## Important Constraint

Because this is an XS build, the scenario should assume the format is consistent.  
Do not over-engineer the parser. Keep it lightweight and predictable.

---

## 7. Step-by-Step Make.com Build Plan

## Scenario Name

`Dark Sky Den - Arrival Detail Logging`

## Module Count

The final scenario should contain:

1. Gmail trigger
2. parser/tools module
3. Google Sheets add row module
4. Gmail reply module

---

## Beginner Build Overview

If the builder is new to Make.com, the easiest way to complete this project is to work in this order:

1. prepare the Gmail label and the Google Sheet first
2. create the scenario shell
3. configure the Gmail trigger and test it alone
4. add the parser module and test extraction
5. add the Google Sheets module and test row creation
6. add the Gmail reply module and test the full flow
7. run three sample emails through the scenario
8. export the blueprint and capture the required screenshots

This order matters. New builders usually make fewer mistakes if they test one module at a time instead of building the whole scenario before the first run.

## Before Opening Make.com

Complete these setup steps first:

1. Open Gmail.
2. Create or confirm a label named `Pending Arrival Details`.
3. Make sure your sample guest emails are either:
   - already in Gmail and tagged with that label, or
   - ready to be sent to the inbox for testing.
4. Open Google Sheets.
5. Create a spreadsheet for the log.
6. Name one worksheet tab something simple such as `Arrivals`.
7. In row 1, create these exact headers:
   - `Guest Name`
   - `Arrival Time`
   - `Dietary Info`
   - `Pod Number`
   - `Logged At`
8. Save the spreadsheet and leave it accessible from the same Google account you will connect to Make.com.

## Exact Final Scenario Layout

Your scenario should look like this:

`Watch Arrival Detail Emails`  
`-> Extract Arrival Details`  
`-> Log Arrival Details to Sheet`  
`-> Send Arrival Confirmation`

If you are using a subject filter instead of label-only filtering, the filter should sit between the Gmail trigger and the parser.

---

### Phase 1 - Create the Scenario and Gmail Trigger

## Goal

Create the scenario and make sure it detects only the correct incoming guest emails.

## Recommended Module

`Gmail > Watch Emails`

## Detailed Beginner Steps

1. Log in to Make.com.
2. From the main dashboard, click `Create a new scenario`.
3. Click the large `+` button in the center of the canvas.
4. Search for `Gmail`.
5. Select the Gmail app.
6. Choose the trigger module `Watch Emails`.
7. If Make.com asks for a connection:
   - click `Add`
   - sign in to the correct Google account
   - allow the requested permissions
   - save the connection with a clear name such as `Dark Sky Den Gmail`
8. Rename the trigger module to `Watch Arrival Detail Emails`.

## Trigger Configuration Guidance

Inside the trigger settings, configure the module so it watches the correct location.

Recommended setup:

- connection: the Gmail account used for guest communication
- source: the inbox or label containing arrival emails
- filter basis: `Pending Arrival Details` label if available
- limit: start with `1` or a small number for testing

## What to Check in the Trigger Output

After configuration, the trigger should be able to return these pieces of data:

- sender email address
- email subject
- email body or plain text body
- message ID
- thread ID if available

These fields matter because:

- sender email is needed for the confirmation
- body text is needed for extraction
- message or thread details help if you choose a reply-style email action

## First Trigger Test

1. Click `Run once` in Make.com.
2. Send yourself or place one test email into Gmail with the correct label.
3. Wait for Make.com to catch it.
4. Click the small bubble above the Gmail module after it runs.
5. Inspect the output.

Confirm:

- the correct email was picked up
- the subject is the one you expect
- the body text is visible
- the sender address is visible

Do not continue until the trigger is reliably catching the right email.

## If You Need a Subject Filter

If the Gmail trigger cannot be limited well enough by label alone:

1. click the line between the Gmail module and the next module slot
2. add a filter
3. give it a clear name such as `Only Arrival Detail Emails`
4. set a rule such as:
   - Subject contains `Arrival Details`
   - or Subject equals your exact required phrase

Use this only if needed. The simplest reliable setup is still best.

---

### Phase 2 - Add the Data Parser / Tools Module

## Goal

Turn the email body into four clean values:

- Guest Name
- Scheduled Arrival Time
- Dietary Restrictions
- Pod Number

## Recommended Module Approach

Use one tools or text parser module that can extract labeled values from the email body.

Because this is an XS project, do not add multiple helper modules unless absolutely necessary. Keep the extraction logic in one parser step if possible.

## Detailed Beginner Steps

1. Click the `+` to the right of the Gmail trigger.
2. Search for the parser or tools app you want to use.
3. Choose the text-parsing style module that best supports extracting values from labeled lines.
4. Rename the module to `Extract Arrival Details`.

## What the Parser Needs to Read

The module should read the email body from the Gmail trigger output.

Use the field that contains the readable email body, ideally plain text.  
Avoid using HTML body content unless plain text is unavailable.

## Source Format to Expect

```text
Guest Name: Amelia Hart
Scheduled Arrival Time: 7:30 PM
Dietary Restrictions: Vegetarian
Pod Number: Pod 3
```

## Extraction Targets

| Output Field | Source Label |
|---|---|
| Guest Name | `Guest Name:` |
| Arrival Time | `Scheduled Arrival Time:` |
| Dietary Info | `Dietary Restrictions:` |
| Pod Number | `Pod Number:` |

## Beginner Parsing Strategy

The parser should conceptually do this:

1. find the line beginning with `Guest Name:`
2. take the text after the colon
3. repeat for the other three labels
4. trim extra spaces from each result

If your chosen parser asks for patterns, make each pattern closely match the visible labels in the email.  
If it asks for delimiters, use the labels and line breaks carefully.

## Important Rule for Dietary Restrictions

This field may be blank.

That means your parser must not fail if the email contains:

```text
Dietary Restrictions:
```

The expected result is either:

- blank text, or
- a safe default like `None provided`

For acceptance purposes, blank is perfectly acceptable.

## Parser Test Procedure

1. Leave the scenario in `Run once` mode.
2. Trigger a sample email.
3. Open the output bubble for `Extract Arrival Details`.
4. Confirm all four extracted values appear correctly.

Check especially:

- `Guest Name` should contain only the guest name, not the whole line
- `Arrival Time` should contain only the time value
- `Pod Number` should not include unrelated line breaks
- `Dietary Info` should be blank, not erroring, when empty

## Do Not Move Forward Until

- all required fields extract correctly from at least one normal email
- the dietary field works with both filled and blank examples

---

### Phase 3 - Add Google Sheets Logging

## Goal

Append one clean row to the arrival log every time a valid guest email is processed.

## Recommended Module

`Google Sheets > Add a Row`

## Detailed Beginner Steps

1. Click the `+` to the right of `Extract Arrival Details`.
2. Search for `Google Sheets`.
3. Select the Google Sheets app.
4. Choose the module `Add a Row`.
5. If Make.com asks for a connection:
   - click `Add`
   - sign in to the correct Google account
   - allow access
   - save the connection with a clear name such as `Dark Sky Den Sheets`
6. Rename the module to `Log Arrival Details to Sheet`.

## Module Configuration Steps

1. Select the destination spreadsheet.
2. Select the worksheet tab, for example `Arrivals`.
3. Wait for Make.com to load the header row.
4. Map each sheet column carefully.

## Exact Mapping Guidance

| Sheet Column | What to Map |
|---|---|
| `Guest Name` | Guest Name from the parser module |
| `Arrival Time` | Scheduled Arrival Time from the parser module |
| `Dietary Info` | Dietary Restrictions from the parser module |
| `Pod Number` | Pod Number from the parser module |
| `Logged At` | Current date/time generated by Make.com |

## Beginner Mapping Tips

- Click inside each field to open the mapping panel.
- Choose values from the parser module, not from the Gmail trigger unless needed.
- Double-check that `Guest Name` is not accidentally mapped into `Arrival Time`.
- Make sure `Logged At` is a timestamp, not a static text string.

## Google Sheets Test Procedure

1. Click `Run once`.
2. Trigger one valid sample email.
3. Let the scenario run through the Sheets module.
4. Open the destination spreadsheet immediately after the run.
5. Confirm a new row appears.

Verify:

- the name is in the correct column
- the arrival time is in the correct column
- the dietary field behaves correctly
- the pod number is correct
- the timestamp was added

If even one field lands in the wrong column, fix the mapping before continuing.

---

### Phase 4 - Add the Gmail Confirmation Reply

## Goal

Send a confirmation email only after the sheet row has been added successfully.

## Recommended Module

Use a Gmail send or reply action that can send a short confirmation to the original sender.

## Detailed Beginner Steps

1. Click the `+` to the right of `Log Arrival Details to Sheet`.
2. Search for `Gmail`.
3. Select a Gmail send or reply module.
4. Rename it to `Send Arrival Confirmation`.
5. Use the same Gmail connection as the trigger unless there is a reason to use another mailbox.

## What to Map in the Reply Module

At minimum, configure:

- recipient email
- subject
- body

## Recommended Mapping

| Reply Field | Recommended Source |
|---|---|
| Recipient | sender email from the Gmail trigger |
| Subject | original subject or a clean confirmation subject |
| Body | dynamic message using Guest Name from the parser |

## Recommended Confirmation Message

```text
Hi {{Guest Name}},

We have received and logged your arrival details. Thank you, and we look forward to welcoming you to The Dark Sky Den.
```

## Acceptance-Criteria Reminder

The greeting must use the dynamic guest name.  
Do not send a generic message such as `Hi there`.

## Full End-to-End Test

1. Click `Run once`.
2. Send or label a valid sample email.
3. Confirm the scenario runs through all four modules.
4. Check the sheet for the new row.
5. Check Gmail for the sent confirmation.

The ideal successful run will show a `1` bubble above each module.  
That is exactly the kind of screenshot required for delivery.

---

### Phase 5 - Final Review for New Builders

## Before Turning the Scenario On

Confirm the following:

- the Gmail trigger is only catching the intended emails
- the parser extracts all four fields correctly
- the Google Sheets row lands in the correct columns
- the confirmation email uses the guest name dynamically
- empty dietary values do not break the run

## Turn the Scenario On

Once testing is complete:

1. click `Save`
2. turn scheduling on
3. confirm the active schedule or watch state

For a new builder, it is smart to leave the scenario monitored closely for the first few test emails instead of assuming it is finished immediately.

---

## 8. Recommended Filters and Logic

## Trigger Qualification Rules

Only process an email if one of the following is true:

- the email has the `Pending Arrival Details` label
- the subject exactly matches or clearly contains the expected arrival-details phrase

## Fail-Safe Recommendations

Although this is an XS project, include light logic safeguards:

- do not proceed if Guest Name is blank
- do not proceed if Arrival Time is blank
- do not proceed if Pod Number is blank
- allow Dietary Info to be blank

If a message is missing required values, it is better to skip logging than to create a corrupted row.

---

## 9. Sample Test Data

Use at least three emails for testing.

## Sample Email 1

```text
Guest Name: Amelia Hart
Scheduled Arrival Time: 7:30 PM
Dietary Restrictions: Vegetarian
Pod Number: Pod 3
```

## Sample Email 2

```text
Guest Name: Noah Ellis
Scheduled Arrival Time: 6:00 PM
Dietary Restrictions:
Pod Number: Pod 1
```

## Sample Email 3

```text
Guest Name: Priya Bennett
Scheduled Arrival Time: 8:15 PM
Dietary Restrictions: Gluten-free
Pod Number: Pod 5
```

---

## 10. Testing Plan

## Core Acceptance Tests

### Test A - Correct Label and Full Data

| Input | Expected Result |
|---|---|
| Email arrives with correct label and all four fields populated | Scenario runs successfully |
| Google Sheets result | New row added with correct mapping |
| Gmail result | Confirmation reply sent |

### Test B - Empty Dietary Restrictions

| Input | Expected Result |
|---|---|
| Dietary Restrictions line is present but blank | Scenario does not fail |
| Google Sheets result | Row is added with blank or default dietary value |
| Gmail result | Confirmation reply still sent |

### Test C - Wrong Subject or Missing Label

| Input | Expected Result |
|---|---|
| Email does not meet trigger condition | Scenario does not process the email |
| Logging result | No sheet row created |

### Test D - Dynamic Guest Name in Confirmation

| Input | Expected Result |
|---|---|
| Valid email with guest name present | Reply begins with guest-specific greeting |
| Gmail result | Greeting uses mapped name, not a hardcoded phrase |

### Test E - Missing Pod Number

| Input | Expected Result |
|---|---|
| Email missing a required field like Pod Number | Scenario should not create a malformed row |
| Recommended handling | Stop or flag the run for manual review |

---

## 11. Deliverables Checklist

The requested deliverables for this project are:

- [ ] Make.com Blueprint exported as `.json`
- [ ] scenario screenshot showing the full module map
- [ ] execution log screenshot showing a successful run with `1` above each module
- [ ] view-only Google Sheet link with at least three successful test rows

## Screenshot Requirements

| Screenshot | What to Capture |
|---|---|
| Scenario map | All 4 modules and their connections |
| Successful execution | The `1` bubble above each module |
| Data mapping proof | Execution details showing extracted values flowing into Google Sheets |

## Sheet Proof Requirements

The shared sample Google Sheet should visibly contain at least:

- three successful rows
- correct guest names
- correct arrival times
- dietary info including one blank case
- correct pod numbers
- logged timestamps

---

## 12. Debugging Checklist

- [ ] Gmail trigger is pointed at the correct label or folder
- [ ] Subject filter is not too broad or too narrow
- [ ] Email body is being read as plain text, not malformed HTML
- [ ] Parser is trimming spaces and line breaks correctly
- [ ] Guest Name mapping goes to `Guest Name`, not another column
- [ ] Arrival Time maps into the correct header
- [ ] Pod Number maps into the correct header
- [ ] Dietary Info allows blanks without stopping the run
- [ ] Logged At column is receiving a timestamp
- [ ] Gmail reply is using the sender email and dynamic guest name

## Common Failure Points

| Issue | Likely Cause |
|---|---|
| Scenario triggers on wrong emails | Label or subject filtering is too loose |
| Empty values in Google Sheets | Parser labels do not match the email text exactly |
| Reply email not personalized | Guest Name variable not mapped into reply text |
| Workflow fails on blank dietary field | Parser or mapping expects a non-empty value |
| Row added to wrong sheet | Spreadsheet or tab selected incorrectly |

---

## 13. Recommended Naming Convention

Use clear names so the scenario screenshot is easy to review.

| Module | Recommended Name |
|---|---|
| Gmail trigger | `Watch Arrival Detail Emails` |
| Parser/tools module | `Extract Arrival Details` |
| Google Sheets module | `Log Arrival Details to Sheet` |
| Gmail reply module | `Send Arrival Confirmation` |

---

## 14. Final Build Summary

This is a compact but valuable Make.com scenario. It is intentionally small, but it still needs careful mapping and testing because the project will be judged primarily on:

- trigger accuracy
- parser accuracy
- correct Google Sheets mapping
- dynamic guest confirmation
- safe handling of blank dietary fields

If the implementer follows the structure in this guide, they should end with a clean XS scenario that is easy to demonstrate, easy to screenshot, and easy to validate against the stated acceptance criteria.
