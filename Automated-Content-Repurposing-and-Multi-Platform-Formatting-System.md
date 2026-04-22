# Automated Content Repurposing and Multi-Platform Formatting System

## 1. Project Overview

This project is an n8n-based automation that converts long-form source content into multiple platform-optimized formats. The workflow accepts a webhook payload containing a single content piece (article, draft, or script), validates it, sends a structured prompt to the Gemini AI model to generate a strict JSON response with platform-specific outputs, normalizes the outputs with Set nodes, and stores them in Airtable for easy retrieval and multi-channel publishing.

Primary goals

- Reduce manual rewriting by producing ready-to-publish variants for LinkedIn, Twitter/X, short summaries, and email re-writes.  
- Keep meaning and brand voice consistent across outputs while adapting tone and structure for each channel.  
- Provide organized storage and filtered views in Airtable so content teams can quickly pick formats for distribution.

Assumptions

- n8n (browser or hosted) with Webhook access.  
- Gemini API access (or provider equivalent).  
- Airtable base for `Master Content` with views filtered by platform type.  
- Slack available for error alerts.

---

## 2. Requirements

- Webhook-triggered workflow that accepts raw content and validates non-empty + minimum length (e.g., 400 characters).  
- AI Transformation: Gemini returns a strict JSON object containing: `linkedin_post`, `twitter_thread` (array), `short_summary`, and `email_rewrite`.  
- Normalization: Use Set nodes to place each format into separate fields for Airtable.  
- Storage: Create records in `Master Content` table and provide filtered views (e.g., `Social Media View`, `Email View`).  
- Error Handling: Dedicated error workflow routes validation or AI/storage failures to Slack with standardized messages.  
- Minimal but robust: linear pipeline with a clear error sub-workflow.

---

## 3. Final Workflow Logic (plain language)

1. `Webhook` trigger receives `title`, `author`, `source_content`.
2. `Code` node validates presence and minimum length; if invalid, route to Error workflow and stop.  
3. `Set` node builds a strict prompt instructing Gemini to return only JSON with the required fields.  
4. `HTTP Request` node calls Gemini and stores the raw `ai_output`.  
5. `Code` node attempts to `JSON.parse(ai_output)` and extracts the four platform-formatted fields (and any metadata like `tags`).  
6. `Set` nodes normalize fields (truncate where necessary, join arrays into newline text for Airtable, etc.).  
7. `Airtable` node creates a `Master Content` record with each format in separate fields and sets `Processing Status: processed`.  
8. On any validation/AI/Airtable failure, the Error workflow standardizes the error and posts to Slack.

---

## 4. Tools & Nodes

- `Webhook` trigger — receive raw content.  
- `Code` node — validation and AI output parsing.  
- `Set` node — prompt construction and normalization.  
- `HTTP Request` node — call Gemini.  
- `Airtable` node — create Master Content record and manage views.  
- `Error Trigger` + Slack node — capture and report errors.

---

## 5. Webhook Payload & Validation

Example POST JSON:

```json
{
  "title": "How We Throw a Better Pot: Glazing Techniques",
  "author": "S. Alem",
  "source_content": "(long article text...)"
}
```

Validation rules (Code node):
- `title` non-empty
- `source_content` non-empty and length >= 400 characters (adjustable)

Validation snippet:

```javascript
const title = ($json.title||'').trim();
const content = ($json.source_content||'').trim();
if(!title || !content || content.length < 400){
  return { json: { ...$json, processing_status: 'validation_failed' } };
}
return { json: { ...$json, processing_status: 'validated' } };
```

On failure: call the Error workflow (post standardized message to Slack and optionally log the event in Airtable).

---

## 6. Prompt Construction (Set node)

Create a `prompt` field with a clear instruction requesting strict JSON. Example:

```
You are an expert content editor. Given the article below, return ONLY valid JSON with these keys:
- "linkedin_post": a single professional LinkedIn post (max 300 words)
- "twitter_thread": an array of up to 6 short tweets (each <280 chars) that form a coherent thread
- "short_summary": a 50–120 character mobile-friendly summary
- "email_rewrite": a professional, friendly newsletter-style rewrite (subject + body)

Article:
"{{$json.source_content}}"

Return only valid JSON. Do not include commentary or extra text.
```

Why: strict JSON reduces parsing errors and simplifies normalization.

---

## 7. Gemini AI Call (HTTP Request node)

- Method: POST  
- Endpoint: provider's Gemini endpoint  
- Headers: `Authorization: Bearer {{ $credentials.geminiKey }}` and `Content-Type: application/json`  
- Body example:

```json
{
  "model": "gemini-1.0",
  "prompt": "{{$json.prompt}}",
  "max_tokens": 1200
}
```

Store the response text in `ai_output`. On non-200 or empty response, trigger the Error workflow with `processing_status: ai_failed`.

---

## 8. Parse & Normalize AI Output (Code + Set nodes)

1. `Code` node: `JSON.parse($json.ai_output)` into `linkedin_post`, `twitter_thread`, `short_summary`, `email_rewrite`. If parse fails, set `processing_status: ai_parse_failed` and route to Error workflow.  
2. `Set` nodes:
   - `linkedin_post` → store raw text
   - `twitter_thread` (array) → join with newline separators for Airtable display or keep as a linked-record array if normalized
   - `short_summary` → store in `Summary` field
   - `email_rewrite` → split into `email_subject` and `email_body` if the model returns both

Truncation and validation: ensure each field meets platform limits; truncate or flag otherwise.

---

## 9. Airtable Schema & Views

Create an Airtable base named `Content Multiplier` with a table `Master Content` and these fields:

- `Title` — Single line text
- `Author` — Single line text
- `Source Content` — Long text (original)
- `LinkedIn Post` — Long text
- `Twitter Thread` — Long text
- `Short Summary` — Single line text
- `Email Subject` — Single line text
- `Email Body` — Long text
- `Processing Status` — Single select (validated, processed, validation_failed, ai_failed, ai_parse_failed)
- `Tags` — Multiple select
- `Created` — Created time

Views:
- `Social Media View` — filter to show `LinkedIn Post` and `Twitter Thread` columns
- `Email/Direct View` — filter to show `Email Subject` and `Email Body`

---

## 10. Error Handling Workflow

Create a small separate workflow that the main flow triggers on errors. Responsibilities:

- Accept standardized error payloads (source node, run_id, error_type, message, payload snippet).  
- Post a formatted message to a Slack channel (e.g., `#content-ops-errors`) with details.  
- Optionally write an `Error Logs` row to Airtable for auditing.

Example Slack message format:

```
Content Multiplier Error — ai_parse_failed
Title: How We Throw a Better Pot
Run ID: abc123
Error: Unexpected token < in JSON
Payload (truncated): {"title":"..."}
```

---

## 11. Deliverables

- `n8n` workflow JSON export for both the main pipeline and the Error workflow.  
- High-resolution screenshot of the full workflow graph.  
- Example test run screenshots (input payload and resulting Airtable record).  
- Airtable base template or CSV and screenshots of the two filtered views.  
- Short screen recording (30–90s) showing a successful end-to-end run.

---

## 12. Testing Plan

- Test 1 — Valid long-form article: expect all four formats generated and a `Master Content` row with `Processing Status: processed`.  
- Test 2 — Short content below threshold: expect validation failure and an Error workflow Slack message.  
- Test 3 — Malformed AI output: simulate and check `ai_parse_failed` is reported and logged.  
- Test 4 — Field limits: ensure Twitter thread tweets are below 280 chars and LinkedIn post within target length.

Capture n8n execution logs and Airtable entries for each test.

---

## 13. Success Criteria

- All four formats generated with consistent meaning and tone.  
- Airtable contains properly typed fields and useful filtered views.  
- Error workflow posts clear, actionable alerts when failures occur.  
- Parsing robustness: model returns clean JSON >90% in spot tests; parse failures handled gracefully.

---

## 14. Next Steps (optional enhancements)

- Add scheduled auto-posting connectors to social platforms (with scheduling controls per record).  
- Add a human review step (approval toggle) before publishing.  
- Implement A/B prompt variants to produce multiple tone options per platform.  

---

If you want, I can now export the ready-to-import `n8n` workflow JSON, generate example `curl` test payloads, or create the Airtable CSV and view screenshots—tell me which and I will proceed.