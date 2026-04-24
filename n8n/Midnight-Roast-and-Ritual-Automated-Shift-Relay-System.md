# Midnight Roast & Ritual: Automated Shift Relay System

## 1) Project Objective

Midnight Roast & Ritual needs a single operational workflow that replaces fragmented SMS group chats with a clean, auditable relay system.

This n8n design captures barista updates from a public input point, logs every event to Google Sheets, and routes notifications to Slack based on urgency.

Primary outcomes:

- Every staff update is captured in one master log.
- Routine shift starts are visible in `#team-updates`.
- Critical low-stock issues are escalated to `#urgent-ops` with high-visibility formatting.
- Integration failures notify the founder automatically.

---

## 2) Business Inputs and Outputs

### Input Data (from form/webhook)
Each submission should include:

- `barista_name` (text)
- `venue` (text)
- `status` (enum: `Shift Started` or `Critical Low Stock`)
- `item_low` (text, optional; e.g., `Oat Milk`, `Espresso Beans`)
- `notes` (text, optional)
- `submitted_at` (timestamp; generated if missing)

### Output Targets

1. **Google Sheets (Master Shift Log)**
   - Append one row for every submission.
2. **Slack `#team-updates`**
   - Receive routine “Shift Started” notices.
3. **Slack `#urgent-ops`**
   - Receive urgent “Critical Low Stock” notices.
4. **Founder failure notification**
   - Receive alerts if Slack or Google Sheets steps fail.

---

## 3) Required External Setup (Before Building)

1. **Google Sheet**
   - Create a sheet named `Master Shift Log`.
   - Recommended headers in row 1:
     - `Timestamp`
     - `Barista Name`
     - `Venue`
     - `Status`
     - `Low Stock Item`
     - `Notes`
     - `Run ID`

2. **Slack Channels**
   - `#team-updates`
   - `#urgent-ops`
   - (Optional) `#founder-alerts` for error notifications

3. **Credentials in n8n**
   - Google Sheets OAuth2 credential
   - Slack OAuth2/API credential

4. **Trigger Choice**
   - Option A: Google Form -> Google Sheets Trigger pipeline
   - Option B (recommended for direct control): **Webhook Trigger** (public URL)

This guide implements **Webhook Trigger** because it is simple, explicit, and easy to test with Postman/browser forms.

---

## 4) Workflow Architecture (Node-by-Node)

## Workflow Name
`Midnight Roast - Shift Relay`

## Node 1 — `Webhook: Shift Intake`
**Type:** Webhook (Trigger)

**Purpose:** Public intake endpoint for staff submissions.

**Configuration:**
- HTTP Method: `POST`
- Path: `midnight-roast-shift-intake`
- Response Mode: `Using Respond to Webhook Node` (preferred) or immediate response
- Authentication: Optional (add secret token if needed)

**Expected payload example:**

```json
{
  "barista_name": "Avery",
  "venue": "Moonlight Gallery",
  "status": "Critical Low Stock",
  "item_low": "Oat Milk",
  "notes": "~1 carton left"
}
```

---

## Node 2 — `Set: Normalize Submission`
**Type:** Set

**Purpose:** Clean and standardize fields before logging/routing.

**Keep Only Set:** `true`

**Fields to set:**
- `barista_name` -> `{{$json.body.barista_name || $json.barista_name || "Unknown"}}`
- `venue` -> `{{$json.body.venue || $json.venue || "Unknown Venue"}}`
- `status` -> `{{$json.body.status || $json.status || "Shift Started"}}`
- `item_low` -> `{{$json.body.item_low || $json.item_low || "N/A"}}`
- `notes` -> `{{$json.body.notes || $json.notes || ""}}`
- `submitted_at` -> `{{$now}}`
- `run_id` -> `{{$execution.id}}`

**Why this matters:**
- Prevents null/undefined data from breaking downstream nodes.
- Produces predictable keys for Sheets and Slack templates.

---

## Node 3 — `Google Sheets: Append Master Log`
**Type:** Google Sheets

**Purpose:** Create immutable audit trail of all submissions.

**Operation:** `Append Row`

**Target:**
- Spreadsheet: `Master Shift Log`
- Sheet/tab: `ShiftEvents` (or first tab)

**Column mapping:**
- `Timestamp` -> `{{$json.submitted_at}}`
- `Barista Name` -> `{{$json.barista_name}}`
- `Venue` -> `{{$json.venue}}`
- `Status` -> `{{$json.status}}`
- `Low Stock Item` -> `{{$json.item_low}}`
- `Notes` -> `{{$json.notes}}`
- `Run ID` -> `{{$json.run_id}}`

**Expected result:**
Each webhook call appends exactly one row.

---

## Node 4 — `Switch: Route by Status`
**Type:** Switch

**Purpose:** Send routine and urgent updates to different Slack channels.

**Mode:** `Rules`

**Value to evaluate:** `{{$json.status}}`

**Rules / Outputs:**
1. Output 0: equals `Shift Started`
2. Output 1: equals `Critical Low Stock`
3. Output 2 (fallback): anything else (optional monitoring path)

**Why Switch over IF:**
- Cleaner branching for multiple statuses and easy future extension (`Shift Ended`, `Delayed Setup`, etc.).

---

## Node 5A — `Slack: Team Update`
**Type:** Slack

**Connected from:** `Switch` output for `Shift Started`

**Purpose:** Friendly visibility for normal shift kickoffs.

**Operation:** `Post Message`

**Channel:** `#team-updates`

**Message template:**

```text
☕ *Shift Started*
*Barista:* {{$json.barista_name}}
*Venue:* {{$json.venue}}
*Time:* {{$json.submitted_at}}

Let’s have a smooth service tonight 🚀
```

---

## Node 5B — `Slack: Urgent Ops Alert`
**Type:** Slack

**Connected from:** `Switch` output for `Critical Low Stock`

**Purpose:** High-visibility escalation for supply risk.

**Operation:** `Post Message`

**Channel:** `#urgent-ops`

**Message template:**

```text
:rotating_light: *CRITICAL LOW STOCK ALERT* :rotating_light:
*Venue:* {{$json.venue}}
*Barista:* {{$json.barista_name}}
*Item at Risk:* {{$json.item_low}}
*Reported At:* {{$json.submitted_at}}
*Notes:* {{$json.notes || "None"}}

*Action Required:* Dispatch restock support immediately.
```

Formatting choices (emoji + bold + action line) make this visibly distinct from routine posts.

---

## Node 5C (Optional) — `Slack: Unknown Status Monitor`
**Type:** Slack

**Connected from:** `Switch` fallback output

**Purpose:** Catch malformed status values without losing observability.

**Channel:** `#founder-alerts` (or internal ops channel)

**Message example:**

```text
⚠️ *Unrecognized status received*
*Barista:* {{$json.barista_name}}
*Venue:* {{$json.venue}}
*Status Payload:* {{$json.status}}
*Run ID:* {{$json.run_id}}
```

---

## Node 6 — `Respond to Webhook`
**Type:** Respond to Webhook

**Purpose:** Return a clean API response to submitter/client.

**Response code:** `200`

**Response body example:**

```json
{
  "ok": true,
  "message": "Shift relay received and processed.",
  "run_id": "{{$json.run_id}}"
}
```

---

## 5) Error-Handling Workflow (Second Workflow)

Create a second workflow dedicated to failures.

## Error Workflow Name
`Midnight Roast - Integration Failure Alerts`

## Error Node 1 — `Error Trigger`
**Type:** Error Trigger

**Purpose:** Activates whenever any workflow execution fails.

## Error Node 2 — `Slack: Founder Failure Alert`
**Type:** Slack

**Channel:** `#founder-alerts` (or DM founder)

**Message template:**

```text
❌ *Workflow Failure Detected*
*Workflow:* {{$json.workflow.name}}
*Failed Node:* {{$json.execution.lastNodeExecuted}}
*Error:* {{$json.execution.error.message}}
*Execution URL:* {{$json.execution.url}}
*Time:* {{$now}}
```

This satisfies the requirement to notify founder when Slack API or Google Sheets integration fails.

---

## 6) End-to-End Connection Map

Main workflow wiring:

1. `Webhook: Shift Intake` -> `Set: Normalize Submission`
2. `Set: Normalize Submission` -> `Google Sheets: Append Master Log`
3. `Google Sheets: Append Master Log` -> `Switch: Route by Status`
4. `Switch (Shift Started)` -> `Slack: Team Update` -> `Respond to Webhook`
5. `Switch (Critical Low Stock)` -> `Slack: Urgent Ops Alert` -> `Respond to Webhook`
6. `Switch (Fallback)` -> `Slack: Unknown Status Monitor` (optional) -> `Respond to Webhook`

Error workflow wiring:

1. `Error Trigger` -> `Slack: Founder Failure Alert`

---

## 7) Test Plan (Proof of Execution)

Run at least 2 concrete tests and capture evidence.

## Test A — Routine Shift Kickoff
Input:
- `status = Shift Started`

Expected:
1. Row appears in Google Sheet.
2. Friendly message appears in `#team-updates`.
3. No message posted to `#urgent-ops`.

## Test B — Critical Low Stock
Input:
- `status = Critical Low Stock`
- `item_low = Espresso Beans`

Expected:
1. Row appears in Google Sheet.
2. Urgent formatted alert appears in `#urgent-ops`.
3. Message includes venue, barista, item, and action-required line.

## Test C — Integration Failure Simulation (optional but recommended)
- Temporarily break Slack credentials or target invalid sheet.
- Confirm `Error Trigger` workflow posts founder-facing failure alert.

---

## 8) Deliverables Checklist

1. **Workflow Export JSON**
   - Export the main workflow (`Midnight Roast - Shift Relay`).
   - Export the error workflow (`Midnight Roast - Integration Failure Alerts`).

2. **Canvas Screenshot**
   - Full n8n canvas showing trigger, normalization, Sheets logging, branching, Slack terminals, and response node.

3. **Proof of Execution Assets**
   - Screenshot (or short recording) showing:
     - Form/webhook submission payload,
     - appended Google Sheet row,
     - Slack post in correct channel.

---

## 9) Suggested File Naming Inside Project Folder

- `Midnight-Roast-Shift-Relay-main-workflow.json`
- `Midnight-Roast-Shift-Relay-error-workflow.json`
- `Midnight-Roast-Shift-Relay-canvas.png`
- `Midnight-Roast-Shift-Relay-test-proof-01.png`
- `Midnight-Roast-Shift-Relay-test-proof-02.png`
- `Midnight-Roast-and-Ritual-Automated-Shift-Relay-System.md`

---

## 10) Success Criteria Mapping

- **Logic accuracy:** Switch routes exactly by `status` value.
- **Professionalism of messages:** Bold labels, emojis, clear action language.
- **Operational clarity:** Founder can glance at Slack and understand active shifts vs supply emergencies immediately.
- **Auditability:** Every submission is present in the master Google Sheet log.

