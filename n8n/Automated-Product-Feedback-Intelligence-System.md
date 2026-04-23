# Automated Product Feedback Intelligence System — E-commerce Insights

## 1. Project Overview

This project is an n8n-based automation that captures customer product feedback via a webhook, uses the Gemini AI model to analyze and structure the feedback for business insight, applies simple priority logic based on sentiment, and logs the final structured record to Airtable for reporting and review. The system is intentionally minimal and linear: no email responses to customers, no fallback models, and no external alerting systems.

Primary goals

- Turn unstructured customer feedback into concise, actionable intelligence.
- Classify sentiment, extract key issues, and assign a priority level for downstream review.
- Persist all runs in Airtable to support dashboards and analysis.

Assumptions

- n8n (browser) is available with Webhook access.  
- Gemini API credentials are available.  
- An Airtable base will receive structured records (Product Health table).

---

## 2. Key Requirements

- Webhook Entry Point: Accept JSON payloads containing `customer_name`, `product_name`, and `feedback_text`.
- Data Validation: Reject and stop on missing or empty required fields.
- AI Analysis (Gemini): Produce structured output including `Sentiment` (Positive/Neutral/Negative), `Summary` (one sentence), and `Identified Issues` (comma-separated list).
- Priority Logic: Map sentiment to priority (`Negative`→High, `Neutral`→Medium, `Positive`→Low) using an IF or Switch node.
- Persistence: Store the full structured result in Airtable with timestamp and mapped fields.
- Minimal Architecture: Linear flow only; no notifications or fallback strategies.

---

## 3. Final Workflow Logic (plain language)

1. `Webhook` trigger receives a POST payload.
2. `Code` node validates `customer_name`, `product_name`, and `feedback_text` are present and non-empty; if invalid, stop.
3. `Set` node constructs the AI prompt with the feedback.
4. `HTTP Request` (Gemini) node sends the prompt and retrieves a structured response.
5. `Code` node parses the AI response into fields: `sentiment`, `summary`, `issues`.
6. `IF` or `Switch` node maps `sentiment` → `priority` (High/Medium/Low).
7. `Airtable` node creates a record with all original and AI-generated fields plus `priority` and timestamp.
8. Workflow ends.

---

## 4. Tools & Nodes

- `Webhook` trigger — accept feedback payloads.  
- `Code` node — validation and parsing.  
- `Set` node — build Gemini prompt.  
- `HTTP Request` node — call Gemini API.  
- `IF`/`Switch` node — map sentiment to priority.  
- `Airtable` node — create structured record.

---

## 5. Webhook Payload & Validation

Expected POST JSON:

```json
{
  "customer_name": "Tsegaye Alem",
  "product_name": "Handwoven Rug - Indigo",
  "feedback_text": "Loved the pattern but the edges started fraying after first wash. Delivery took 10 days."
}
```

Validation (Code node example):

```javascript
const name = ($json.customer_name||'').trim();
const product = ($json.product_name||'').trim();
const feedback = ($json.feedback_text||'').trim();
if(!name || !product || !feedback){
  // Stop workflow by returning a flagged item; downstream logic should not call AI
  return { json: { ...$json, processing_status: 'invalid_payload' } };
}
return { json: { ...$json, processing_status: 'validated' } };
```

Note: Per minimal design, invalid payloads are recorded or ignored and do not trigger further nodes.

---

## 6. Prompt Construction (Set node)

Construct a concise instruction for Gemini to return a strict JSON-like structure. Example template:

```
Analyze the following customer feedback and return a JSON object with keys: "sentiment" (Positive, Neutral, Negative), "summary" (one sentence, <20 words), and "issues" (comma-separated list of specific issues or highlights).

Feedback:
"{{$json.feedback_text}}"

Return only valid JSON in this exact format.
```

Place the resulting text into a `prompt` field.

---

## 7. Gemini AI Call (HTTP Request node)

- Method: POST  
- Endpoint: provider-specific Gemini endpoint  
- Body: include `model` and `prompt` fields; set `max_tokens` as needed.

Example (provider-dependent):

```json
{ "model": "gemini-1.0", "prompt": "{{$json.prompt}}", "max_tokens": 400 }
```

On response, extract the model's text output into `ai_output`.

Important: Instruct Gemini to return clean JSON so parsing in the next step is deterministic.

---

## 8. Parse AI Output (Code node)

Attempt to `JSON.parse(ai_output)` and extract `sentiment`, `summary`, and `issues`. If parsing fails, set `processing_status: ai_parse_failed` and stop (per minimal design).

Example parsing snippet:

```javascript
let out;
try{
  out = JSON.parse($json.ai_output);
} catch(e){
  return { json: { ...$json, processing_status: 'ai_parse_failed' } };
}
return { json: { ...$json, sentiment: out.sentiment, summary: out.summary, issues: out.issues } };
```

---

## 9. Priority Mapping (IF / Switch node)

Rules:

- If `sentiment` === `Negative` → `priority` = `High`
- If `sentiment` === `Neutral` → `priority` = `Medium`
- If `sentiment` === `Positive` → `priority` = `Low`

Attach the chosen `priority` to the record for Airtable.

---

## 10. Airtable Schema (Product Health table)

Create an Airtable base with a table named `Product Health` and fields:

- `Date` — Created time
- `Customer` — Single line text
- `Product` — Single line text
- `Original Feedback` — Long text
- `AI Summary` — Long text
- `Sentiment` — Single select (Positive, Neutral, Negative)
- `Priority` — Single select (High, Medium, Low)
- `Key Issues` — Long text (or multi-select if normalized)
- `Processing Status` — Single select (validated, processed, invalid_payload, ai_parse_failed)
- `Run ID` — Text

Field types must match (Single select for Sentiment and Priority, Date for timestamps).

---

## 11. Deliverables

- `n8n` workflow JSON export (Webhook, Validation Code, Set prompt, Gemini call, Parse Code, IF/Switch, Airtable create).  
- Airtable base template populated with at least 5 sample records showing varied sentiment and priority.  
- Screenshot of a successful n8n execution (green checks on nodes).  
- Short screen capture demonstrating a test POST and the resulting Airtable entry.

---

## 12. Testing Plan

- Test 1 — Positive feedback: expect `Sentiment: Positive`, `Priority: Low`.  
- Test 2 — Neutral feedback: expect `Sentiment: Neutral`, `Priority: Medium`.  
- Test 3 — Negative feedback: expect `Sentiment: Negative`, `Priority: High`.  
- Test 4 — Missing `product_name` or empty `feedback_text`: expect `processing_status: invalid_payload`, no AI call.  
- Test 5 — Malformed AI output: expect `processing_status: ai_parse_failed`.

For each test, capture the webhook payload, n8n execution trace, and Airtable row.

---

## 13. Acceptance Criteria

- Priority mapping: Negative feedback must map to `High` priority consistently during testing.  
- Summary quality: AI-generated `summary` must be concise (under 20 words) and reflect the feedback sentiment in spot checks.  
- Data integrity: All runs are written to Airtable with correct field-types.  
- Resilience: Workflow ignores invalid inputs without failing the whole n8n instance.

---

## 14. Notes & Gotchas

- Encourage the AI to return strict JSON to avoid brittle parsing.  
- If you expect diverse languages or noisy input, consider normalizing text before sending to the model.  
- The minimal architecture simplifies deployment but does not surface urgent high-priority items automatically; integrate alerting later if desired.

---

## 15. Next Steps

- I can export the n8n workflow JSON for import, generate sample webhook `curl` commands for testing, or create a sample Airtable CSV with 5 test rows — tell me which and I will proceed.
