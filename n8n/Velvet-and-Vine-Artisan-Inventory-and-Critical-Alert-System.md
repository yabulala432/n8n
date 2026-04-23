# Velvet & Vine: Artisan Inventory & Critical Alert System

## 1. Project Overview

Velvet & Vine is a boutique direct-to-consumer brand producing small-batch botanical skincare. The operations team needs a lightweight, reliable automation to collect inventory updates from distributed artisans, classify stock levels, log every submission, and surface only critical alerts so the team can act quickly without notification fatigue.

This document describes an n8n browser workflow that:

- accepts inventory updates via an `n8n` form trigger,
- classifies stock levels (NORMAL, LOW_STOCK, OUT_OF_STOCK),
- logs all submissions into a persistent Data Table (`inventory_logs`), and
- sends structured Slack alerts only when action is required.

Why this matters

- Contributors are distributed and manual tracking introduces risk of stockouts.
- Frequent, noisy alerts reduce operational effectiveness.
- Targeted automation keeps the team informed and reduces manual work.

Assumptions

- n8n browser instance is available and can host a Form Trigger.
- A Data Table named `inventory_logs` exists in n8n (or will be created).
- Slack credentials are configured for the operations channel.
- Thresholds for LOW_STOCK are known (example below uses <10 units).

---

## 2. Requirements

- Centralized Logging: Persist every submission with metadata for auditability.
- Intelligent Classification: Use a JavaScript `Code` node to classify stock into `NORMAL`, `LOW_STOCK`, or `OUT_OF_STOCK`.
- Minimal Noise: Slack alerts only for `LOW_STOCK` and `OUT_OF_STOCK`.
- Clear Alerts: Slack messages should include product, status, count, location, and an action line.
- Resilience: Basic input validation and safe type conversion; option to add an `Error Trigger` workflow.

---

## 3. Final Workflow Logic (plain language)

1. Artisan submits inventory update via `Form Trigger`.
2. `Set` node normalizes/cleans incoming fields.
3. `Code` node classifies stock using numeric thresholds.
4. Insert a row into `inventory_logs` Data Table with classification result.
5. `IF` node checks `is_critical`.
   - If `false`: record silent success (optional confirmation message) and finish.
   - If `true`: send a formatted Slack alert to `#operations` and finish.
6. Optionally, wire an `Error Trigger` to email the admin on failures.

---

## 4. Tools & Nodes

- n8n (browser)
- Form Trigger (data input)
- Set (normalize fields)
- Code (classification)
- Data Table (inventory_logs)
- IF (decision)
- Slack (alerts)
- Email / Error Trigger (optional resilience)

---

## 5. Data Table Schema (inventory_logs)

| Field Name         | Type     | Description                     |
| ------------------ | -------- | ------------------------------- |
| product_name       | Text     | Product identifier / SKU        |
| stock_count        | Number   | Submitted stock count           |
| warehouse_location | Text     | Location of the stock           |
| status             | Text     | NORMAL / LOW_STOCK / OUT_OF_STOCK |
| is_critical        | Boolean  | True when alert required        |
| timestamp          | DateTime | Submission time (ISO)           |
| submitter_contact  | Text     | Optional contact info           |

---

## 6. Classification Logic (Code node)

Node name: `Stock Classification Engine`  
Execution: `Run Once for Each Item`

```javascript
const product = $json.product_name || "Unknown Product";
const stock = Number($json.stock_count || 0);
const location = $json.warehouse_location || "Unknown Location";

let status;
if (stock === 0) {
  status = "OUT_OF_STOCK";
} else if (stock > 0 && stock < 10) {
  status = "LOW_STOCK";
} else {
  status = "NORMAL";
}

const isCritical = status !== "NORMAL";

return {
  json: {
    product_name: product,
    stock_count: stock,
    warehouse_location: location,
    status: status,
    is_critical: isCritical,
    timestamp: new Date().toISOString(),
    submitter_contact: $json.submitter_contact || ''
  }
};
```

Notes:
- Adjust the `10` threshold as business needs change.
- Use defensive parsing to avoid exceptions on bad input.

---

## 7. Decision & Routing

- IF node checks `is_critical`:
  - `false` → `Set` node records a silent acknowledgement and the Data Table entry remains the authoritative record.
  - `true` → Slack node sends a structured alert to operations and includes a direct link to the `inventory_logs` entry when possible.

---

## 8. Slack Alert Template

Channel: `#operations`  
Message (rich text / blocks recommended):

Inventory Alert — Velvet & Vine

- *Product:* Lavender Body Oil
- *Status:* LOW_STOCK
- *Stock Remaining:* 4
- *Location:* Addis Ababa Central Storage
- *Submitted By:* artisan@example.com

Action Required: Please review and restock.

---

## 9. Success Path (NORMAL)

- No Slack message.
- A confirmation Set node can return a short success response to the Form submitter (optional).
- The `inventory_logs` table contains the submission for audit.

---

## 10. Critical Path (LOW_STOCK / OUT_OF_STOCK)

- Slack alert sent to `#operations` with context and action line.
- Optionally send an email to purchasing or create a task in the team's backlog system.

---

## 11. Error Handling

- Validate and cast `stock_count` safely in the Code node.
- Wrap API calls with retry/backoff where supported.
- Create a lightweight `Error Trigger` workflow that emails `ops-leads@velvetandvine.com` with the failing run ID and payload.

---

## 12. Deliverables

- n8n Workflow JSON export (Form Trigger, Code, Data Table, IF, Slack, optional Error Trigger).
- Screenshots: Form Trigger setup, Code node content, Data Table with entries, Slack alert sample.
- Short screen recording (60–90s) demonstrating: a normal submission, a low-stock submission, and the Slack alert.

---

## 13. Testing Plan

- Test Case 1: `stock_count = 15` → `NORMAL`, no Slack alert.
- Test Case 2: `stock_count = 3` → `LOW_STOCK`, Slack alert.
- Test Case 3: `stock_count = 0` → `OUT_OF_STOCK`, Slack alert.
- Test Case 4: malformed `stock_count` (text) → numeric fallback to `0` or reject with an explanatory message.

---

## 14. Success Criteria

- All submissions are stored in `inventory_logs`.
- Classification aligns with thresholds.
- Slack alerts are triggered only for critical cases.
- Admins are notified on workflow-level failures.

---

## 15. Next Steps (optional)

- Add multi-warehouse threshold configuration.
- Integrate simple replenishment workflows (create a ticket in the procurement board).
- Add analytics: weekly report of LOW_STOCK occurrences by SKU.

---

If you want, I can now: export the n8n workflow JSON, generate the Data Table CSV and a screenshot, or build the n8n canvas and run test submissions—tell me which and I'll proceed.