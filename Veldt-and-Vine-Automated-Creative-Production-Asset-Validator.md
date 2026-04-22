# Veldt & Vine: Automated Creative Production Asset Validator

## 1. Project Overview

Veldt & Vine needs an n8n workflow to automatically validate production bookings made by the creative team. The automation sits between the team's Google Calendar and an Airtable inventory of studio/equipment assets and prevents double-bookings, schedules around maintenance windows, and records validation results. The goal is to reduce manual cross-checking, cut schedule friction, and ensure assets under maintenance never get scheduled.

This guide targets the **n8n browser version** and describes the full workflow design, node-by-node logic, sample code used for keyword detection, Airtable schema used for testing, and recommended error handling.

Why this matters

- Production uses shared, limited assets: studios, rigs, specialty lenses, and overhead hardware.
- Manual checks cause missed conflicts and late rescheduling.
- Automation prevents wasted studio time and protects gear from being used while under maintenance.

Assumptions

- You control the `Creative Shoots` Google Calendar (where bookings are created).
- There is an Airtable base called `Production Assets` with a `Assets` table.
- There is a secondary calendar called `Production Coordination` for confirmed reservations.
- Slack is available for Ops alerts and an email address exists for admin notifications.

---

## 2. Requirements (mapped to user ask)

- Intelligent Event Parsing: Trigger on new Google Calendar events and parse title & description to identify requested assets using pattern matching (e.g., "Macro", "Studio A", "Overhead Rig").
- Inventory Logic: Query Airtable `Assets` table for each identified asset and evaluate availability for the requested time range (`Available`, `Maintenance`, `In-Use`).
- Validation & Logging:
  - Success: Create a secondary event in the `Production Coordination` calendar and update the original event description to include `Validated` and a link to the Airtable record.
  - Conflict: Post a detailed alert to Slack and append `CONFLICT DETECTED` to the original calendar event with a diagnostic note describing blocked assets and reasons.
- Resilience: Provide a separate `Error Trigger` workflow to notify the administrator by email on main workflow failures (API timeouts, missing records, credential issues).


## 3. Final Workflow Logic (plain language)

1. A creative schedules a shoot in `Creative Shoots` calendar.
2. `Google Calendar Trigger` (Event Created) starts the workflow.
3. `Set` node normalizes payload fields (`event_id`, `title`, `description`, `start_time`, `end_time`, `organizer`).
4. `Code` node scans title + description for supported resource keywords and returns `requested_resources` array.
5. If `requested_resources` is empty, optionally notify a manual review queue and stop.
6. For each detected resource:
   - Lookup the asset by name in Airtable `Assets` table.
   - Evaluate whether the asset is `Available` for the requested time window.
7. Group lookup results; if *all* assets are available:
   - Create a reservation event in `Production Coordination` calendar (title & linked metadata).
   - Update the original event description to include `Validated` and an Airtable record link.
8. If any asset is `In-Use` or `Maintenance`:
   - Post a Slack message to `#production-ops` summarizing conflicts (asset, status, reason).
   - Update the original calendar event title/description to prepend `CONFLICT DETECTED` and include diagnostic notes.
9. Add a short `Workflow Run` log record (optional: write a row to an Airtable `Validation Logs` table).
10. On node-level failures (timeouts, missing Airtable records, credentials): let the node error and route the error into the `Error Trigger` workflow, which emails the admin and records the failure.

---

## 4. Tools & Nodes Needed

- n8n (browser)
- Google Calendar Trigger (Event Created)
- Google Calendar (to create/update events)
- Edit Fields (Set) node (normalize incoming payload)
- Code (JavaScript) nodes (keyword detection, grouping, decision helpers)
- Airtable node (search / lookup records, update logs)
- IF node (route availability / conflict branches)
- Slack node (conflict alerts)
- Email node (admin notifications)
- Error Trigger workflow (separate workflow triggered by errors)


## 5. Airtable Schema (sample for testing)

Create an Airtable base named `Production Assets` with at least these two tables:

Table: `Assets`

| Asset Name | Asset ID | Type | Status | Notes | Airtable Record URL (formula) |
|---|---:|---|---|---|---|
| Studio A | A-001 | Space | Available |  | https://airtable.example/recA001 |
| Overhead Rig | H-043 | Rig | Maintenance | Motor replaced 2026-04-01 | https://airtable.example/recH043 |
| Macro Lens Kit | L-021 | Lens Kit | In-Use | In-use until 2026-04-22 10:00 | https://airtable.example/recL021 |
| Lighting Kit - Day | E-009 | Lighting | Available | Spare bulbs in storage | https://airtable.example/recE009 |
| Tripod Heavy | T-005 | Support | Available |  | https://airtable.example/recT005 |

Table: `Validation Logs` (optional)

- Fields: `event_id`, `title`, `requested_resources`, `result`, `notes`, `processed_at`, `run_id`

> Note: For the deliverable, include a screenshot of the `Assets` view showing at least five items with mixed statuses.

---

## 6. Example Keyword Detection Code (n8n `Code` node)

Node name: `Detect - Resource Keywords`

Language: JavaScript (Run Once for Each Item = false)

```javascript
// resource map: label + search patterns
const resourceMap = [
  { label: 'Studio A', patterns: ['studio a', 'studio-a'] },
  { label: 'Overhead Rig', patterns: ['overhead rig', 'overhead-rig', 'overhead'] },
  { label: 'Macro Lens Kit', patterns: ['macro lens', 'macro', 'macro lens kit'] },
  { label: 'Lighting Kit', patterns: ['lighting kit', 'lighting'] },
  { label: 'Tripod Heavy', patterns: ['tripod'] },
];

const text = `${($json.title||'').toLowerCase()} ${($json.description||'').toLowerCase()}`;

function escapeRegex(s){return s.replace(/[.*+?^${}()|[\\]\\]/g,'\\$&');}

const requested = [];
for (const r of resourceMap){
  const found = r.patterns.some(p=>{
    const rx = new RegExp(`(^|[^a-z0-9])${escapeRegex(p)}([^a-z0-9]|$)`, 'i');
    return rx.test(text);
  });
  if(found && !requested.includes(r.label)) requested.push(r.label);
}

return { json: { ...$json, requested_resources: requested } };
```

Output example after this node:

```json
{
  "requested_resources": ["Studio A", "Overhead Rig", "Macro Lens Kit"]
}
```

Notes:
- The regex anchors reduce false positives (e.g., avoid matching "macrobiotic").
- Add or remove entries from `resourceMap` as assets change; no schema changes required.

---

## 7. Airtable Lookup & Availability Logic

1. Use an Airtable `Search` node that filters `Assets` table by `Asset Name` equals the resource label.
2. For timeline conflicts, you can optionally add a `Bookings` table to store reservations and then check for overlapping bookings. For a minimal implementation, use the `Status` field (`Available`, `In-Use`, `Maintenance`):
   - `Available` → pass
   - `In-Use` or `Maintenance` → considered blocked
3. After collecting all lookups, use a `Code` node to assemble results into a single object like:

```json
{
  "resourceChecks": [
    {"asset":"Studio A","status":"Available","recordUrl":"https://..."},
    {"asset":"Macro Lens Kit","status":"In-Use","notes":"Expected free at 2026-04-22T10:00Z"}
  ],
  "allAvailable": false
}
```

4. Route with an `IF` node on `allAvailable`.

---

## 8. Success Path: Create Reservation & Annotate Event

If all assets are available:

- Create a new event in `Production Coordination` calendar with title: `RESERVED - <original title>` and a description that includes:
  - `Validated` marker
  - `Airtable links` for reserved assets (use the `recordUrl` found in Airtable lookup)
  - `Validation run id` for tracing
- Update the original event in `Creative Shoots` calendar: append or prepend `Validated` and a short note like:

```
Validated by Asset Validator on 2026-04-20 10:25 UTC
Reserved assets: Studio A (recA001), Lighting Kit (recE009)
Validation record: https://airtable.example/recLog123
```

- Optionally create a row in `Validation Logs` for auditing.

---

## 9. Conflict Path: Slack Alert & Annotate Original Event

If any asset is blocked:

- Post to Slack channel `#production-ops` with a structured message:
  - Event title + organizer + start/end times
  - List of blocked assets and their statuses
  - Suggested next steps (reschedule; pick alternate asset)
- Update original calendar event title or description to prepend `CONFLICT DETECTED` and append a short diagnostic, for example:

```
CONFLICT DETECTED - Overhead Rig under maintenance until 2026-04-30
Please check #production-ops for details.
```

- Keep the `Production Coordination` calendar untouched in this case.

---

## 10. Error Handling Workflow (resilience)

Create a separate n8n workflow with an `Error Trigger` that:

1. Receives the error metadata (node name, error message, run id, original payload).
2. Sends an email to the admin (e.g., `ops-leads@veldtandvine.com`) describing the failure and linking to the failed n8n execution.
3. Optionally posts a high-priority Slack message in `#ops-alerts`.

Typical errors to handle:
- Airtable timeouts or rate limits
- Missing asset record (asset name detection but no Airtable record found)
- Google API credential errors
- Slack API errors

Implementation note: In n8n, enable the `Continue On Fail` setting where you want to capture errors and route them to the `Error Trigger`.

---

## 11. Deliverables (what you'll hand over)

- n8n Workflow JSON: Exported workflow with all nodes (Trigger, Set, Code, Airtable, IF, Google Calendar, Slack, Email, and Error Trigger) ready to import.
- Airtable Schema Screenshot: `Assets` table screenshot showing at least five items with mixed statuses.
- Proof of Execution: 1) screenshot of a successful validation run in n8n execution view; 2) screenshot of a conflict run showing the Slack alert and updated calendar event; 3) resulting `Validated` update inside the original Google Calendar event.
- This project brief (this Markdown file) describing logic and code snippets for maintainability.

---

## 12. Proof of Concept Notes & Testing Plan

- Populate `Assets` table with the five sample records above (mix `Available`, `In-Use`, `Maintenance`).
- In Google Calendar, create two test events within the `Creative Shoots` calendar:
  1. Event A: includes keywords that map to `Available` assets → expect `Validated` annotation and a `Production Coordination` reservation.
  2. Event B: includes a keyword that maps to `Maintenance` asset → expect Slack alert and `CONFLICT DETECTED` annotation.
- Capture screenshots of n8n execution for both runs and the Slack message.

---

## 13. Success Criteria (how we judge this)

- Automation correctly detects resources from raw text (>90% keyword match for the provided map).
- Available bookings create a reservation event and update original event with `Validated` and links.
- Conflicts generate Slack alerts and annotate the original event with `CONFLICT DETECTED` and diagnostic notes.
- Error workflow notifies admin for workflow-level failures.
- Airtable screenshot and n8n run screenshots are provided as proof of execution.

---

## 14. Next Steps I can do for you

- Build the n8n canvas and export the workflow JSON for import.
- Generate sample Airtable CSV and a quick screenshot image for the `Assets` table.
- Create example Google Calendar event JSON and a sample Slack message payload for testing.

If you want me to proceed with any of those, say which one and I will build it next.
