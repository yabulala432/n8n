# Meeting Transcript Summarizer & Action‑Item Extractor

## 1. Project Overview

This n8n workflow converts raw meeting transcripts into structured meeting intelligence: a concise meeting summary, key discussion points, and extracted action items (with assignees and suggested due dates when possible). Input is a text transcript (via webhook or file input). The workflow validates the transcript, builds a precise prompt for the Gemini AI model, parses the model's structured JSON response, normalizes the output with Set nodes, and stores everything in Airtable for a centralized meeting dashboard.

Goals

- Automate meeting documentation and reduce manual note-taking.
- Produce consistent, machine-readable meeting summaries and action item lists.
- Keep the workflow linear and minimal while ensuring deterministic parsing.

Assumptions

- n8n (browser) or hosted instance is available with Webhook/file access.  
- Gemini API access is available for the AI steps.  
- An Airtable base will store meetings and action items.

---

## 2. Requirements

- Accept meeting transcripts via a `Webhook` (POST) or file upload.  
- Validate that the transcript is present and not empty.  
- Build a structured prompt that directs Gemini to return strict JSON containing: `summary`, `discussion_points`, and `action_items` (array of objects).  
- Parse the AI output and normalize fields for Airtable.  
- Persist meeting records and extracted action items in Airtable.  
- Keep a single linear pipeline with deterministic failure modes (validation failed, ai_parse_failed).

---

## 3. Final Workflow Logic (plain language)

1. `Webhook` or `Read Binary File` trigger receives the transcript and metadata (meeting_title, meeting_date, participants).
2. `Code` node validates the transcript is non-empty. If validation fails: set `processing_status: validation_failed` and stop (optionally write an Airtable record).
3. `Set` node builds a precise prompt for Gemini asking for strict JSON output.
4. `HTTP Request` (Gemini) node sends prompt and receives `ai_output` (text).
5. `Code` node attempts `JSON.parse(ai_output)`. On parse failure: set `processing_status: ai_parse_failed` and stop.
6. Use `Set` nodes to normalize parsed fields (`summary`, `discussion_points`, `action_items`) into Airtable-friendly fields.
7. `Airtable` node creates a `Meeting` record (Meeting Title, Date, Participants, Transcript, Summary, Processing Status, Run ID).
8. Optionally, create linked `Action Items` rows per extracted task (task, assignee, suggested_due_date, status, meeting link).
9. Workflow ends.

---

## 4. Tools & Nodes

- `Webhook` or `Read Binary File` trigger (input).  
- `Code` node (validation and parsing).  
- `Set` nodes (prompt builder, normalization).  
- `HTTP Request` node (Gemini API call).  
- `Airtable` node(s) (Meeting record, optionally Action Items table).  

---

## 5. Expected Input & Validation

Sample webhook JSON payload:

```json
{
  "meeting_title": "Product Roadmap Sync",
  "meeting_date": "2026-04-22T09:00:00Z",
  "participants": "Amina Tesfaye, John Doe",
  "transcript": "(full meeting transcript text here)"
}
```

Validation (Code node example):

```javascript
const transcript = ($json.transcript || '').trim();
if (!transcript) {
  return { json: { ...$json, processing_status: 'validation_failed' } };
}
return { json: { ...$json, processing_status: 'validated' } };
```

Behavior: validation failure should stop the pipeline; record the failure in Airtable if you want an audit trail.

---

## 6. Prompt Construction (Set node)

Create a `prompt` field with a strict instruction requesting JSON output. Example template:

```
You are an accurate meeting summarizer. Given the following raw meeting transcript, return ONLY valid JSON with these keys:
- "summary": a concise summary (2-4 sentences)
- "discussion_points": an array of short strings (key topics discussed)
- "action_items": an array of objects, each with { "task": string, "assignee": string or null, "suggested_due": string (ISO date) or null }

Transcript:
"{{$json.transcript}}"

Return only JSON. Do not add commentary or extraneous text.
```

Why strict JSON: deterministic parsing reduces brittle failures and simplifies downstream normalization.

---

## 7. Gemini AI Call (HTTP Request node)

- Method: `POST`  
- Endpoint: provider-specific Gemini endpoint  
- Headers: set `Authorization` with your API key and `Content-Type: application/json`  
- Body example (provider-dependent):

```json
{ "model": "gemini-1.0", "prompt": "{{$json.prompt}}", "max_tokens": 800 }
```

Response handling: store the raw model text in `ai_output` for audit and parsing.

Fail-safe: on non-200 responses set `processing_status: ai_failed` and stop (record to Airtable if desired).

---

## 8. Parse AI Output (Code node)

Attempt to parse the model output into JSON and extract fields. Example:

```javascript
let parsed;
try {
  parsed = JSON.parse($json.ai_output);
} catch (err) {
  return { json: { ...$json, processing_status: 'ai_parse_failed', parse_error: err.message } };
}

return { json: { ...$json, summary: parsed.summary, discussion_points: parsed.discussion_points, action_items: parsed.action_items, processing_status: 'parsed' } };
```

Notes: Keep the parse logic defensive — verify array shapes before proceeding.

---

## 9. Normalization & Airtable Mapping (Set nodes)

- `summary` → store as Long Text in Airtable.  
- `discussion_points` → join with newline or store as a linked record list if normalized.  
- `action_items` → ideally create a separate `Action Items` table with fields: `Task`, `Assignee`, `Suggested Due`, `Status`, `Meeting Link`.

Minimal approach (single table): store `action_items` as JSON or newline-delimited strings in a `Action Items` long-text field.

Airtable `Meetings` table suggested fields:
- `Meeting Title` (Single line text)
- `Meeting Date` (Date/Time)
- `Participants` (Long text)
- `Transcript` (Long text)
- `Summary` (Long text)
- `Discussion Points` (Long text)
- `Action Items` (Long text / JSON)
- `Processing Status` (Single select: validated, parsed, ai_failed, validation_failed)
- `Run ID` (Text)

Optional `Action Items` table:
- `Task` (Long text)
- `Assignee` (Single line text)
- `Suggested Due` (Date)
- `Status` (Single select: Open, In Progress, Done)
- `Meeting` (Link to Meetings table)

---

## 10. Deliverables

- n8n Workflow JSON export (Webhook/File trigger, Validation Code, Prompt Set, Gemini call, Parse Code, Normalization, Airtable create).  
- Airtable base template and suggested field types (Meetings and optional Action Items).  
- Example test transcript(s) and a `curl` POST payload for testing.  
- Screenshots: n8n canvas with nodes and a sample successful execution.  
- Short screen recording (30–90s) demonstrating a test transcript being sent and resulting Airtable records.

---

## 11. Testing Plan

- Test A — Short transcript with clear assignments: expect `action_items` parsed with assignees and suggested due dates.  
- Test B — Long transcript with decisions and follow-ups: expect concise `summary` and multiple `action_items`.  
- Test C — Empty transcript: expect `processing_status: validation_failed`, no AI call.  
- Test D — Model returns malformed output: expect `processing_status: ai_parse_failed` and record for manual review.

Capture execution traces and Airtable rows for each test.

---

## 12. Success Criteria

- Only valid transcripts proceed to AI call.  
- Gemini returns valid JSON and parsing succeeds >90% in spot checks.  
- Action items are extracted with at least task descriptions, and assigned person when mentioned.  
- Meeting summaries are concise (2–4 sentences) and capture main decisions.

---

## 13. Next Steps (optional enhancements)

- Automatically create tasks in a task manager (Asana/Trello/Jira) from extracted action items.  
- Add participant name normalization / directory lookup to convert text names into user records.  
- Add confidence scoring (ask the model to include a confidence field per action item).  
- Support multilingual transcripts with language detection and translation pre-step.

---

If you'd like, I can now: export an n8n workflow JSON for this design, generate sample `curl` payloads and example transcripts, or create the Airtable CSV template with the Meetings and Action Items tables. Tell me which and I'll proceed.