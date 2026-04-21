# High-Priority Network Emergency Alert System

## Step-by-Step n8n Build Guide

This guide shows you how to build the workflow manually in n8n using only these real nodes:

- `Webhook`
- `Set`
- `IF`
- `Code`
- `Slack`
- `Google Sheets`
- `Date & Time`
- `Error Trigger`

The workflow handles emergency outage submissions from a public form, filters serious incidents, alerts engineers in Slack, logs everything to Google Sheets, and captures system-level failures.

---

## 1. What You Are Building

You will create **two workflows**:

1. `High-Priority Network Emergency Alert System`
2. `High-Priority Network Emergency Alert System - Error Handler`

The main workflow:

- accepts `POST` requests from a webhook
- checks whether required fields are present
- rejects incomplete submissions cleanly
- filters only `Critical` and `Emergency` severity incidents
- sends a Slack alert for valid emergencies
- logs all outcomes to Google Sheets
- records whether Slack was sent successfully

The error workflow:

- listens for workflow failures using `Error Trigger`
- logs the workflow name, failed node, message, and timestamp to Google Sheets

---

## 2. Example Incoming Payload

Your webhook should accept JSON like this:

```json
{
  "client_name": "Westside Dental Clinic",
  "location": "Chicago Office",
  "severity": "Critical",
  "description": "Entire network offline. Phones down."
}
```

---

## 3. Google Sheet Preparation

Before building the workflow, prepare one Google Spreadsheet with these tabs.

### Tab 1: `Invalid Requests`

Create these columns in row 1:

- `Timestamp`
- `Client Name`
- `Location`
- `Severity`
- `Description`
- `Rejection Reason`
- `Missing Fields`

### Tab 2: `Non-Emergency Requests`

Create these columns:

- `Timestamp`
- `Client Name`
- `Location`
- `Severity`
- `Description`
- `Reason`

### Tab 3: `Emergency Audit Log`

Create these columns:

- `Timestamp`
- `Client Name`
- `Location`
- `Severity`
- `Description`
- `Slack Sent`

### Tab 4: `System Errors`

Create these columns:

- `Workflow Name`
- `Failed Node`
- `Error Message`
- `Timestamp`

---

## 4. Slack Preparation

Make sure you have:

- a Slack app or Slack credentials connected in n8n
- permission to post messages to your emergency channel
- a real target channel for your on-call engineers

Example channel:

- `#blue-screen-busters-emergency`

---

## 5. Create the Main Workflow

In n8n:

1. Click `Create Workflow`
2. Name it `High-Priority Network Emergency Alert System`
3. Save the workflow

Now build the nodes in the order below.

---

## 6. Node 1: Incoming Emergency Form

### Node Type

`Webhook`

### Rename It

`Incoming Emergency Form`

### Purpose

This is the public entry point for emergency outage submissions.

### Configure It

- `HTTP Method`: `POST`
- `Path`: `high-priority-network-emergency-alert`
- `Response Mode`: `On Received`

### Why `On Received`

This makes n8n acknowledge the request immediately instead of waiting for the whole workflow to finish. That is important for fast emergency intake.

### Input Expectation

The request body should contain:

- `client_name`
- `location`
- `severity`
- `description`

---

## 7. Node 2: Capture Received Time

### Node Type

`Date & Time`

### Rename It

`Capture Received Time`

### Purpose

Capture the moment the incident entered the workflow.

### Configure It

- Operation: `Get Current Date`
- Turn on current time inclusion
- Output field name: `received_at_iso`
- Include input fields: `true`

### Result

This gives you an ISO timestamp you can reuse later.

---

## 8. Node 3: Format Incident Timestamp

### Node Type

`Date & Time`

### Rename It

`Format Incident Timestamp`

### Purpose

Convert the raw timestamp into a readable format for Slack and Google Sheets.

### Configure It

- Operation: `Format Date`
- Date field:

```text
{{ $json["received_at_iso"] }}
```

- Format mode: `Custom`
- Custom format:

```text
yyyy-LL-dd HH:mm:ss ZZZZ
```

- Output field name: `timestamp`
- Include input fields: `true`

### Result

You now have a human-readable `timestamp`.

---

## 9. Node 4: Normalize Request

### Node Type

`Code`

### Rename It

`Normalize Request`

### Purpose

This node:

- reads the webhook body
- trims input values
- checks missing required fields
- normalizes severity to lowercase
- identifies whether the request is a real emergency
- builds clean fields for later nodes

### Configure It

Use JavaScript and paste this:

```javascript
const firstItem = $input.first();
const source = firstItem.json.body ?? firstItem.json;

const clean = (value) => {
  if (value === undefined || value === null) return '';
  return String(value).trim();
};

const clientName = clean(source.client_name);
const location = clean(source.location);
const severity = clean(source.severity);
const description = clean(source.description);

const missingFields = [];

if (!clientName) missingFields.push('client_name');
if (!location) missingFields.push('location');
if (!severity) missingFields.push('severity');
if (!description) missingFields.push('description');

const normalizedSeverity = severity.toLowerCase();
const emergencyLevels = ['critical', 'emergency'];
const isValid = missingFields.length === 0;
const isEmergency = isValid && emergencyLevels.includes(normalizedSeverity);
const shortDescription =
  description.length > 120 ? `${description.slice(0, 117)}...` : description;

return [
  {
    json: {
      client_name: clientName,
      location,
      severity,
      severity_normalized: normalizedSeverity,
      description,
      short_description: shortDescription,
      received_at_iso: firstItem.json.received_at_iso,
      timestamp: firstItem.json.timestamp,
      missing_fields: missingFields.join(', '),
      rejection_reason: isValid
        ? ''
        : `Missing required field(s): ${missingFields.join(', ')}`,
      non_emergency_reason:
        isValid && !isEmergency
          ? 'Severity is not Critical or Emergency'
          : '',
      is_valid: isValid,
      is_emergency: isEmergency
    }
  }
];
```

### Why This Node Matters

It makes the rest of the workflow simple. Instead of repeating logic in many places, you validate and standardize once here.

---

## 10. Node 5: Validate Fields

### Node Type

`IF`

### Rename It

`Validate Fields`

### Purpose

Check whether all required fields were provided.

### Condition

Use a boolean comparison:

- Left Value:

```text
{{ $json["is_valid"] }}
```

- Operator: `Equals`
- Right Value: `true`

### Outputs

- `true` path: continue to severity filtering
- `false` path: go to the missing data branch

---

## 11. Missing Data Branch

If required fields are missing, route to Google Sheets and end cleanly.

### 11A. Node: Build Invalid Request Row

#### Node Type

`Set`

#### Rename It

`Build Invalid Request Row`

#### Purpose

Map clean values into sheet-ready column names.

#### Add These Fields

- `Timestamp` =

```text
{{ $json["timestamp"] }}
```

- `Client Name` =

```text
{{ $json["client_name"] }}
```

- `Location` =

```text
{{ $json["location"] }}
```

- `Severity` =

```text
{{ $json["severity"] }}
```

- `Description` =

```text
{{ $json["description"] }}
```

- `Rejection Reason` =

```text
{{ $json["rejection_reason"] }}
```

- `Missing Fields` =

```text
{{ $json["missing_fields"] }}
```

### 11B. Node: Log Invalid Request

#### Node Type

`Google Sheets`

#### Rename It

`Log Invalid Request`

#### Purpose

Append incomplete submissions to the `Invalid Requests` tab.

#### Configure It

- Resource/Operation: append a row
- Spreadsheet: your emergency spreadsheet
- Sheet: `Invalid Requests`
- Mapping: map fields from the previous node

#### Expected Outcome

Any invalid submission is recorded with the reason for rejection and then the workflow stops naturally.

---

## 12. Node 6: Check Severity

### Node Type

`IF`

### Rename It

`Check Severity`

### Purpose

Only allow urgent incidents to continue to Slack.

### Condition

- Left Value:

```text
{{ $json["is_emergency"] }}
```

- Operator: `Equals`
- Right Value: `true`

### Outputs

- `true` path: send Slack alert
- `false` path: log as non-emergency

Because the Code node already normalized severity, this check works case-insensitively for values like:

- `Critical`
- `critical`
- `EMERGENCY`
- `Emergency`

---

## 13. Non-Emergency Branch

For submissions that are valid but not urgent enough.

### 13A. Node: Build Non-Emergency Row

#### Node Type

`Set`

#### Rename It

`Build Non-Emergency Row`

#### Add These Fields

- `Timestamp` =

```text
{{ $json["timestamp"] }}
```

- `Client Name` =

```text
{{ $json["client_name"] }}
```

- `Location` =

```text
{{ $json["location"] }}
```

- `Severity` =

```text
{{ $json["severity"] }}
```

- `Description` =

```text
{{ $json["description"] }}
```

- `Reason` =

```text
{{ $json["non_emergency_reason"] }}
```

### 13B. Node: Log Non-Emergency Request

#### Node Type

`Google Sheets`

#### Rename It

`Log Non-Emergency Request`

#### Purpose

Append lower-priority tickets to the `Non-Emergency Requests` tab.

#### Configure It

- Spreadsheet: your emergency spreadsheet
- Sheet: `Non-Emergency Requests`
- Operation: append row

#### Expected Outcome

Requests with severity other than `Critical` or `Emergency` are stored for review but do not alert engineers.

---

## 14. Emergency Branch

Now build the urgent alerting path.

### 14A. Node: Send Slack Alert

#### Node Type

`Slack`

#### Rename It

`Send Slack Alert`

#### Purpose

Post an urgent outage message to the on-call channel.

#### Configure It

- Resource: `Message`
- Operation: `Post`
- Select by: `Channel`
- Channel: your on-call Slack channel
- Message type: `Text`

#### Slack Message

Use this expression:

```text
{{ `🚨 NETWORK OUTAGE ALERT

Client: ${$json["client_name"]}
Location: ${$json["location"]}
Severity: ${$json["severity"]}
Issue: ${$json["description"]}
Time: ${$json["timestamp"]}` }}
```

#### Important Setting

In the node settings, enable error handling so the workflow continues even if Slack fails.

Use:

- `On Error` -> `Continue (using error output)`

### Why This Matters

If Slack is down or credentials break, the incident must still be logged in Google Sheets with `Slack Sent = No`.

---

## 15. Slack Success Branch

This is the normal path when Slack posts successfully.

### 15A. Node: Build Audit Row (Slack Yes)

#### Node Type

`Set`

#### Rename It

`Build Audit Row (Slack Yes)`

#### Add These Fields

- `Timestamp` =

```text
{{ $json["timestamp"] }}
```

- `Client Name` =

```text
{{ $json["client_name"] }}
```

- `Location` =

```text
{{ $json["location"] }}
```

- `Severity` =

```text
{{ $json["severity"] }}
```

- `Description` =

```text
{{ $json["description"] }}
```

- `Slack Sent` = `Yes`

### 15B. Node: Write Audit Log (Slack Yes)

#### Node Type

`Google Sheets`

#### Rename It

`Write Audit Log (Slack Yes)`

#### Configure It

- Spreadsheet: your emergency spreadsheet
- Sheet: `Emergency Audit Log`
- Operation: append row

#### Expected Outcome

Every valid emergency that successfully alerts Slack is saved in the audit log.

---

## 16. Slack Failure Branch

When `Send Slack Alert` fails, use its error output.

### 16A. Node: Build Audit Row (Slack No)

#### Node Type

`Set`

#### Rename It

`Build Audit Row (Slack No)`

#### Add These Fields

- `Timestamp` =

```text
{{ $json["timestamp"] }}
```

- `Client Name` =

```text
{{ $json["client_name"] }}
```

- `Location` =

```text
{{ $json["location"] }}
```

- `Severity` =

```text
{{ $json["severity"] }}
```

- `Description` =

```text
{{ $json["description"] }}
```

- `Slack Sent` = `No`

### 16B. Node: Write Audit Log (Slack No)

#### Node Type

`Google Sheets`

#### Rename It

`Write Audit Log (Slack No)`

#### Configure It

- Spreadsheet: your emergency spreadsheet
- Sheet: `Emergency Audit Log`
- Operation: append row

#### Expected Outcome

Even if Slack fails, the incident is still captured in the audit trail.

---

## 17. Main Workflow Connection Order

Connect the nodes like this:

```text
Incoming Emergency Form
-> Capture Received Time
-> Format Incident Timestamp
-> Normalize Request
-> Validate Fields
```

From `Validate Fields`:

- `true` -> `Check Severity`
- `false` -> `Build Invalid Request Row` -> `Log Invalid Request`

From `Check Severity`:

- `true` -> `Send Slack Alert`
- `false` -> `Build Non-Emergency Row` -> `Log Non-Emergency Request`

From `Send Slack Alert`:

- success output -> `Build Audit Row (Slack Yes)` -> `Write Audit Log (Slack Yes)`
- error output -> `Build Audit Row (Slack No)` -> `Write Audit Log (Slack No)`

---

## 18. Create the Error Workflow

Create a second workflow in n8n.

### Name It

`High-Priority Network Emergency Alert System - Error Handler`

This workflow will capture failures from the main workflow and log them to Google Sheets.

---

## 19. Node 1: Workflow Error Trigger

### Node Type

`Error Trigger`

### Rename It

`Workflow Error Trigger`

### Purpose

This starts the workflow whenever another workflow connected to it fails.

### Setup

No extra settings required.

---

## 20. Node 2: Build System Error Row

### Node Type

`Set`

### Rename It

`Build System Error Row`

### Purpose

Map the incoming error details to your Google Sheet column structure.

### Add These Fields

- `Workflow Name` =

```text
{{ $json["workflow"]["name"] }}
```

- `Failed Node` =

```text
{{ $json["execution"]["lastNodeExecuted"] || "Unknown" }}
```

- `Error Message` =

```text
{{ $json["execution"]["error"]["message"] || "Unknown error" }}
```

- `Timestamp` =

```text
{{ $now.toISO() }}
```

---

## 21. Node 3: Log System Error

### Node Type

`Google Sheets`

### Rename It

`Log System Error`

### Purpose

Append workflow failures to the `System Errors` tab.

### Configure It

- Spreadsheet: your emergency spreadsheet
- Sheet: `System Errors`
- Operation: append row

---

## 22. Error Workflow Connection Order

Connect the nodes like this:

```text
Workflow Error Trigger
-> Build System Error Row
-> Log System Error
```

---

## 23. Link the Main Workflow to the Error Workflow

After both workflows exist:

1. Open the main workflow
2. Go to workflow settings
3. Find the `Error Workflow` setting
4. Select `High-Priority Network Emergency Alert System - Error Handler`
5. Save the workflow

This ensures failures in the main workflow get passed to the error workflow.

---

## 24. Webhook Testing

Use the webhook test URL from the `Incoming Emergency Form` node.

Send a `POST` request with JSON.

### Test 1: Valid Emergency

```json
{
  "client_name": "Westside Dental Clinic",
  "location": "Chicago Office",
  "severity": "Critical",
  "description": "Entire network offline. Phones down."
}
```

Expected:

- workflow starts immediately
- Slack message is sent
- row is appended to `Emergency Audit Log`
- `Slack Sent` is `Yes`

### Test 2: Missing Fields

```json
{
  "client_name": "Westside Dental Clinic",
  "severity": "Critical"
}
```

Expected:

- no Slack message
- row is appended to `Invalid Requests`
- rejection reason explains which fields are missing

### Test 3: Non-Emergency

```json
{
  "client_name": "Westside Dental Clinic",
  "location": "Chicago Office",
  "severity": "Medium",
  "description": "Wi-Fi is slow in the front office."
}
```

Expected:

- no Slack alert
- row is appended to `Non-Emergency Requests`

### Test 4: Slack Failure

Temporarily disconnect Slack credentials or use a bad channel.

Expected:

- workflow does not silently fail
- row still lands in `Emergency Audit Log`
- `Slack Sent` is `No`

---

## 25. Final Pre-Activation Checklist

Before activating:

- Slack credentials are connected
- Google Sheets credentials are connected
- all required sheet tabs exist
- all sheet headers match the Set node field names
- the webhook path is correct
- the main workflow is linked to the error workflow
- Slack node is set to continue using the error output

---

## 26. Recommended Node Names

Use these names exactly to keep the workflow easy to read:

- `Incoming Emergency Form`
- `Capture Received Time`
- `Format Incident Timestamp`
- `Normalize Request`
- `Validate Fields`
- `Build Invalid Request Row`
- `Log Invalid Request`
- `Check Severity`
- `Build Non-Emergency Row`
- `Log Non-Emergency Request`
- `Send Slack Alert`
- `Build Audit Row (Slack Yes)`
- `Write Audit Log (Slack Yes)`
- `Build Audit Row (Slack No)`
- `Write Audit Log (Slack No)`
- `Workflow Error Trigger`
- `Build System Error Row`
- `Log System Error`

---

## 27. Summary

This design gives you:

- immediate webhook intake
- field validation
- graceful handling of incomplete submissions
- case-insensitive emergency filtering
- urgent Slack notification
- Google Sheets logging for every branch
- clear tracking of Slack success vs failure
- centralized workflow error logging

If you want, I can also turn this same guide into a cleaner client-facing handoff document with less technical wording and more copy-paste-friendly sections.
