# Automated SLA Monitor — Pinnacle Tax & Advisory Services

## 1. Project Overview

Pinnacle Tax & Advisory Services needs an n8n monitoring and escalation system to ensure document reviews and client queries meet strict SLAs during peak filing seasons. This workflow polls a central Airtable task board, evaluates elapsed time against dynamic SLAs, categorizes tasks into On Track / Warning / Breached, and escalates via Slack and email while preventing notification spam. All evaluations are logged in an `SLA Audit Trail` table for auditability.

This brief documents the required logic, node-by-node design, Airtable schema, escalation rules, testing plan, and expected deliverables.

---

## 2. Key Requirements

- Task Monitoring: Read active tasks from an Airtable `Tasks` table (fields: Task Name, Owner, Priority, Status, Start Time, SLA Flags).
- Dynamic Time Logic: High-priority tasks → 24-hour SLA; Standard tasks → 72-hour SLA.
- Three-Tier Classification: On Track, Warning (within 4 hours of breach), Breached.
- Escalation Path:
  - Warning: Slack notification to the task owner (once per task).
  - Breached: Email escalation to Operations Manager and set `Manager Review` in Airtable (once).
- State Awareness: Use flags in Airtable (e.g., `warning_sent`, `breach_sent`) to avoid duplicate notifications.
- Audit Logging: Write every evaluation and escalation action to `SLA Audit Trail` with timestamp and details.
- Error Handling: Sub-workflow to notify admin via Slack on Airtable/API failures or malformed data.

---

## 3. Final Workflow Logic (plain language)

1. `Cron` (or Schedule) trigger runs periodically (e.g., every 15 minutes).
2. `Airtable` node queries `Tasks` where `Status` != `Closed` and `SLA Active` = true.
3. For each active task, a `Code` node computes elapsed time and compares it to SLA threshold (24h or 72h) and derives `time_to_breach_hours`.
4. `Code` node classifies task state:
   - `time_to_breach_hours > 4` → `On Track`
   - `0 < time_to_breach_hours <= 4` → `Warning`
   - `time_to_breach_hours <= 0` → `Breached`
5. Before sending notifications, check Airtable flags `warning_sent` and `breach_sent` to prevent repeats.
6. If `Warning` and `warning_sent` is false:
   - Send Slack message to the `Owner`
   - Set `warning_sent` = true in Airtable
   - Insert an audit row in `SLA Audit Trail`
7. If `Breached` and `breach_sent` is false:
   - Send Gmail to Operations Manager with task details
   - Update `Manager Review` checkbox and set `breach_sent` = true in Airtable
   - Insert an audit row in `SLA Audit Trail`
8. If `On Track` and both flags are false, do nothing (but log evaluation optionally).
9. Any node-level error routes a standardized payload into the Error sub-workflow which posts to `#ops-errors` Slack channel.

---

## 4. Tools & Nodes

- `Cron` / `Schedule Trigger` — periodic checks.
- `Airtable` — query tasks and update flags; write audit rows.
- `Code` — compute elapsed time, SLA thresholds, and classification.
- `IF` or `Switch` — branch on classification and flag checks.
- `Slack` node — send warning notifications to owners.
- `Gmail` (or `Email`) node — send breach escalations to Operations Manager.
- `Error Trigger` sub-workflow + `Slack` — notify admin on service errors.

---

## 5. Airtable Schema

Table: `Tasks`

- `Task Name` — Single line text
- `Owner` — Single line text (email or Slack handle)
- `Priority` — Single select (`High`, `Standard`)
- `Status` — Single select (`Open`, `In Progress`, `Closed`)
- `Start Time` — Date/Time (when the task was created or assigned)
- `SLA Active` — Checkbox (true if SLA applies)
- `warning_sent` — Checkbox (default false)
- `breach_sent` — Checkbox (default false)
- `Manager Review` — Checkbox (set when breached)
- `Notes` — Long text

Table: `SLA Audit Trail`

- `Timestamp` — Created time
- `Task Name` — Link to Tasks (or text)
- `Action` — Single select (`Evaluated`, `Warning Sent`, `Breach Sent`, `Error`)
- `Details` — Long text (includes elapsed, SLA hours, run id)
- `Run ID` — Text

---

## 6. Time Logic (example Code node)

Node: `Compute SLA State` (JavaScript, Run Once for Each Item)

```javascript
const now = new Date();
const start = new Date($json['Start Time']);
const elapsedMs = now - start;
const elapsedHours = elapsedMs / (1000*60*60);

const priority = ($json.Priority || 'Standard');
const slaHours = (priority === 'High') ? 24 : 72;
const timeToBreach = slaHours - elapsedHours; // hours

let state = 'On Track';
if (timeToBreach <= 0) state = 'Breached';
else if (timeToBreach <= 4) state = 'Warning';

return {
  json: {
    ...$json,
    elapsed_hours: Number(elapsedHours.toFixed(2)),
    sla_hours: slaHours,
    time_to_breach_hours: Number(timeToBreach.toFixed(2)),
    sla_state: state
  }
};
```

Notes:
- Use server timezone consistently (store times as ISO UTC).
- For tasks with future `Start Time`, treat `elapsed_hours` as 0.

---

## 7. Notification & State Update Details

Warning Path:
- Check `warning_sent` flag. If false:
  - Slack message to `Owner` channel or direct message.
  - Update `warning_sent` = true in `Tasks` row.
  - Insert `SLA Audit Trail` row with `Action = Warning Sent` and details.

Breached Path:
- Check `breach_sent` flag. If false:
  - Send email to Operations Manager (Gmail node) with subject: `SLA Breach — <Task Name>` and body containing task link and timing details.
  - Set `breach_sent` = true and `Manager Review` = true in Airtable.
  - Insert `SLA Audit Trail` row with `Action = Breach Sent` and details.

State awareness ensures each task receives at most one Warning and one Breach notification.

---

## 8. Error Handling Sub-workflow

Design a small `Error Handler` workflow that accepts an error payload (source node, run id, error message, sample record) and does:
- Post a standardized message to Slack `#ops-errors`.
- Create an `SLA Audit Trail` row with `Action = Error` and error details.

Trigger: use `Execute Workflow` or call the Error sub-workflow from `IF` / `On Error` paths.

---

## 9. Deliverables

- `n8n` workflow JSON export for main SLA monitor and Error sub-workflow.
- Screenshots: full workflow graph and the time-comparison `Compute SLA State` code node.
- Proof of Execution: screenshots or short video showing a Warning triggered for one task and a Breach for another; include Slack message and email, plus updated Airtable rows (flags and audit entries).
- Template Base: Airtable base (share link) populated with at least 5 sample tasks covering High/Standard, near-breach and already-breached cases.

---

## 10. Testing Plan

- Seed Airtable with sample tasks:
  - Task A: High priority, Start Time 23 hours ago → expect `Warning` (within 4 hours)
  - Task B: Standard priority, Start Time 74 hours ago → expect `Breached`
  - Task C: Standard priority, Start Time 10 hours ago → `On Track`
  - Tasks D/E: edge cases (future start time, missing fields)

- Run schedule trigger or execute workflow manually and verify:
  - Slack Warning to owner for Task A and `warning_sent` = true
  - Email to Ops Manager for Task B and `breach_sent` + `Manager Review` = true
  - `SLA Audit Trail` contains entries for each action with timestamps and run IDs

---

## 11. Success Criteria

- Correct classification of tasks into On Track / Warning / Breached.
- Exactly one Warning and one Breach notification per task (no duplicates).
- Audit trail captures every evaluation and escalation with useful detail.
- Error handler reports Airtable/API failures to `#ops-errors` and logs them.

---

## 12. Next Steps & Enhancements

- Add retry logic and exponential backoff for Airtable API calls.  
- Add owner mapping to Slack user IDs for direct messages.  
- Integrate task creation for resequencing breached tasks (create follow-up tasks automatically).  
- Create a dashboard (Airtable view or external dashboard) summarizing SLA health over time.

---

If you want, I can now: export the n8n workflow JSON, generate the sample Airtable CSV and share link, or create the test payloads and record a demo run — which should I do next?