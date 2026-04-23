# Daily Study Companion — AI Summary System

## 1. Project Overview

The Daily Study Companion is an n8n-based automation that converts raw study notes or topics into concise, structured learning summaries and delivers them by email to students. The system accepts a webhook payload (student name, student email, study content), validates required fields, constructs a tutoring prompt, sends the prompt to the Gemini AI model to generate a readable summary, emails the summary to the student, and logs the interaction into Airtable for audit and tracking.

This project intentionally keeps the architecture minimal and linear: there are no fallback models, error workflows, or external alerting systems. The goal is a simple, reliable pipeline that transforms unstructured educational content into organized study material.

Assumptions

- n8n (browser) is available and can expose a public webhook (or be fronted by a proxy).  
- You have a Gemini API key (or equivalent provider access) and an Airtable base for logging.  
- Email credentials (SMTP or n8n Mail node) are configured.

---

## 2. Requirements

- Webhook input: receive `student_name`, `student_email`, and `study_content`.
- Validation: verify presence and simple formatting (email regex) before processing.
- Prompt construction: build a fixed prompt via a `Set` node to instruct Gemini how to produce the summary.
- AI generation: call the Gemini model and receive a clear, structured summary.
- Delivery: email the formatted summary to the student with a fixed subject line.
- Logging: write the interaction to Airtable with student info, original content, generated summary, timestamp, and processing status.
- Minimal architecture: single linear workflow, no fallback models, no separate error-triggered workflows.

---

## 3. Final Workflow Logic (plain language)

1. n8n `Webhook` trigger receives a POST with `{ student_name, student_email, study_content }`.
2. `Code` (or `Set`) node validates fields: non-empty `student_name`, valid `student_email`, non-empty `study_content`.
   - If validation fails: set `processing_status` = `validation_failed`, write an Airtable record, and stop.
3. `Set` node constructs the Gemini prompt (fixed template with placeholders filled from the payload).
4. `HTTP Request` (or dedicated AI node) calls Gemini and receives `generated_summary`.
5. `Set` node formats the email body using the returned summary.
6. `Email` node sends the summary to `student_email` with a fixed subject.
7. `Airtable` node logs the run (student details, original input, generated summary, timestamp, run_id, processing_status = processed).
8. Workflow ends—linear flow with no branching to alternate models.

---

## 4. Tools & Nodes

- `Webhook` trigger — receive POST payloads.  
- `Code` node — input validation (or `Set` for simple checks).  
- `Set` node — build the prompt and format email.  
- `HTTP Request` node (or provider-specific Gemini node) — send prompt to Gemini.  
- `Email` node — send the summary to the student.  
- `Airtable` node — create a record for auditing.  

---

## 5. Webhook Payload & Input Validation

Expected JSON payload (POST):

```json
{
  "student_name": "Amina Tesfaye",
  "student_email": "amina@example.com",
  "study_content": "Explain photosynthesis, light-dependent reactions and Calvin cycle in simple terms. Include 3 key study points."
}
```

Validation (Code node example — JavaScript):

```javascript
// Run once for each item
const name = ($json.student_name || '').trim();
const email = ($json.student_email || '').trim();
const content = ($json.study_content || '').trim();

const emailRx = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;

if (!name || !email || !content || !emailRx.test(email)) {
  return { json: { ...$json, processing_status: 'validation_failed' } };
}

return { json: { ...$json, processing_status: 'validated' } };
```

Notes:
- Per the "minimal" requirement, validation failures are recorded and stop the workflow; no separate error workflow is invoked.

---

## 6. Prompt Construction (Set node)

Node: `Build Prompt`

Use a `Set` node to create a single `prompt` field containing a clear instruction for Gemini. Example prompt template (n8n expression-friendly):

```
You are a patient, expert tutor. Create a concise study summary for the student below.

Student: {{$json.student_name}}

Input material:
{{$json.study_content}}

Produce:
1) A 2–4 sentence Overview.
2) 4–6 Key Points (bulleted).
3) Short Definitions for any technical terms (1–2 lines each).
4) One Example or Analogy to aid memory.
5) Two Quick Practice Questions (with brief answers).

Keep language simple, actionable, and suitable for quick revision. Return the result as plain text with Markdown headings.
```

Set node output (example field):
- `prompt` → the full text above with expressions substituted.

---

## 7. Gemini AI Call (HTTP Request node)

Node: `Call Gemini`

- Method: `POST`  
- URL: your provider's Gemini endpoint (e.g., Vertex AI / provider API)  
- Headers: `Authorization: Bearer {{ $credentials.geminiApiKey }}` and `Content-Type: application/json`  
- Body (JSON): include `model` and `prompt` fields per your provider's API.

Example request body (provider-dependent):

```json
{
  "model": "gemini-1.0",
  "prompt": "{{$json.prompt}}",
  "max_tokens": 800
}
```

Response handling:
- Parse the response and set `generated_summary` to the returned text.  
- For safety keep the workflow linear: if the AI call returns an error, write an Airtable record with `processing_status: ai_failed` and stop.

---

## 8. Output Formatting Guidelines

Ask Gemini to return Markdown headings and bullet lists to make the email readable. Example expected structure:

# Overview

Two–three sentences.

# Key Points

- Point 1
- Point 2

# Definitions

- Term: short definition

# Example

A short analogy or worked example.

# Quick Practice Questions

1. Q? — A.
2. Q? — A.

---

## 9. Email Delivery (Email node)

Node: `Send Summary Email`

- To: `{{$json.student_email}}`  
- From: configured sender (e.g., `no-reply@yourdomain.com`)  
- Subject (fixed): `Daily Study Companion — Your Study Summary`  
- Body: use the `generated_summary` directly (HTML or plain text). Optionally include a short greeting:

```
Hi {{$json.student_name}},

Here's your study summary generated by the Daily Study Companion:

{{$json.generated_summary}}

Happy studying!

— Daily Study Companion
```

---

## 10. Airtable Logging (audit)

Create an Airtable base (or table) named `study_summaries` with fields:

- `student_name` (Single line text)
- `student_email` (Email)
- `original_input` (Long text)
- `generated_summary` (Long text)
- `processing_status` (Single select: validated, processed, validation_failed, ai_failed)
- `timestamp` (Created time or DateTime)
- `run_id` (Text)

After email is sent (or after validation failure), create a record with the above fields. This provides a searchable history and grounds for manual review if required.

---

## 11. Deliverables

- `n8n` Workflow JSON export (Webhook, Validation Code, Build Prompt Set, Gemini call, Email, Airtable create).  
- Example webhook POST payload for testing.  
- Sample email template and example generated summary.  
- Airtable schema description and sample CSV for import.

---

## 12. Testing Plan

- Test 1 — Happy path: valid `student_name`, valid `student_email`, substantive `study_content`. Expect: `generated_summary` emailed and logged with `processing_status: processed`.
- Test 2 — Missing email: expect Airtable record with `processing_status: validation_failed`, no AI call, no email.
- Test 3 — Empty content: same as Test 2.
- Test 4 — AI service error (simulate by invalid API key): expect Airtable record `ai_failed`.

For each test, capture the webhook request, the n8n execution trace, Airtable row, and the sent email.

---

## 13. Success Criteria

- Webhook accepts payload and validates required fields.
- Gemini returns a coherent, structured summary following the requested format >80% of the time in spot checks.
- Student receives the emailed summary with fixed subject line.
- Airtable contains a logged record per run with correct fields and processing status.
- The workflow remains a single linear pipeline with no fallback or separate alerting flows.

---

## 14. Next Steps (optional enhancements)

- Add light-weight retry logic for transient AI API failures.  
- Add a small web UI for manual re-processing of `validation_failed` or `ai_failed` records.  
- Build an opt-in digest (daily/weekly) summarizing generated notes per student.

---

If you want, I can now: export a ready-to-import `n8n` workflow JSON for this design, generate an example webhook POST file for testing, or scaffold the `Airtable` CSV. Tell me which and I'll proceed.