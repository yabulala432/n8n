# AI-Powered Vintage Instrument Intake Formatter for Luthier's Ledger

## 1. Project Overview

This workflow takes rough buyer intake notes about vintage string instruments and turns them into clean, structured inventory data.

Instead of having a manager read freeform notes like "late 50s Gibson, neck reset, crack repaired, one tuner replaced," the workflow:

1. Accepts the messy notes.
2. Checks whether the notes are usable.
3. Sends the notes to an AI model trained through prompting to act like a vintage instrument appraiser.
4. Returns a strict JSON record with condition details, repairs, likely year, grade, and a short summary.
5. Optionally sends a clean human-readable alert to Slack or email.

This guide is written for the n8n browser interface and assumes you want the fastest path to a working result.

---

## 2. Final Workflow Logic

### Main Path

`Manual Trigger`  
`-> Input - Messy Notes (Edit Fields / Set)`  
`-> Check Notes Present (If)`  
`-> AI Vintage Intake Extractor (OpenAI)`  
`-> Normalize Intake JSON (Code)`  
`-> Slack - Intake Alert (optional)`

### Validation Side Path

If the notes are empty, null, or too short:

`Check Notes Present (true output)`  
`-> Return Validation Error (Edit Fields / Set)`

### What the workflow does at each stage

| Stage | What happens |
|---|---|
| Trigger | You start the workflow manually while testing |
| Input | A Set node stores the messy intake note in a field called `raw_notes` |
| Validation | An IF node checks that the text exists and is at least 10 characters long |
| AI extraction | The OpenAI node reads the note and returns structured JSON |
| Cleanup | A Code node parses and normalizes the AI response into reliable final fields |
| Notification | An optional Slack or email step sends a readable summary to a human |

---

## 3. Tools Needed

Use the following inside n8n:

| Tool | Why you need it |
|---|---|
| `Manual Trigger` | Fast testing while building |
| `Edit Fields (Set)` | Create sample messy notes and return error payloads |
| `If` | Reject empty or too-short note text |
| `OpenAI` node | Extract, classify, and format the intake notes |
| `Code` node | Parse and normalize the AI response |
| `Slack` or `Send Email` node | Optional human notification |
| OpenAI credentials | Required for the AI step |

### Recommended AI choice

Use the `OpenAI` node with:

- `Resource`: `Text`
- `Operation`: `Generate a Chat Completion`
- `Model`: `gpt-4o-mini` for speed and cost control, or `gpt-4o` for higher accuracy

Note: In newer n8n versions, the OpenAI node uses updated naming. The older "Message a Model" label was renamed to `Generate a Chat Completion`.

---

## 4. Exact Workflow Build (Step by Step)

## Before You Build

1. Open n8n in the browser.
2. Create a new workflow.
3. Name it `Luthier's Ledger - Vintage Intake Formatter`.
4. Make sure your OpenAI credential is already connected in n8n.
5. Build the nodes in the exact order below.

---

### Node 1: Manual Trigger

### Node Name

`Manual Trigger`

### Purpose

Starts the workflow manually while you test the build.

### How to Add It

1. Click `Add first step`.
2. Search for `Manual Trigger`.
3. Select it.

### Exact Settings

No configuration is required.

### Expressions

None.

### Expected Output

When you click `Execute Workflow`, this node starts the run and passes control to the next node.

---

### Node 2: Input - Messy Notes

### Node Name

`Input - Messy Notes`

### Purpose

Stores the messy buyer notes in a predictable field so every later node can read the same input key: `raw_notes`.

### How to Add It

1. Click the `+` on the `Manual Trigger` node.
2. Search for `Edit Fields`.
3. Add `Edit Fields (Set)`.
4. Rename it to `Input - Messy Notes`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |
| `Include in Output` | `No Input Fields` |

Paste this into the `JSON Output` field:

```json
{
  "raw_notes": "Picked up estate piece maybe late 50s Gibson. Sunburst mostly intact. Neck reset done years ago. Crack near lower bout repaired. Original tuners maybe replaced one peg. Frets worn medium. Case smells old. Plays warm.",
  "source": "manual_test",
  "buyer_name": "Field Buyer A",
  "received_at": "{{ $now.toISO() }}"
}
```

### Expressions

Use this expression for the timestamp:

```text
{{ $now.toISO() }}
```

### Expected Output

```json
{
  "raw_notes": "Picked up estate piece maybe late 50s Gibson. Sunburst mostly intact. Neck reset done years ago. Crack near lower bout repaired. Original tuners maybe replaced one peg. Frets worn medium. Case smells old. Plays warm.",
  "source": "manual_test",
  "buyer_name": "Field Buyer A",
  "received_at": "2026-04-17T..."
}
```

---

### Node 3: Check Notes Present

### Node Name

`Check Notes Present`

### Purpose

Rejects notes that are empty, null, or too short to be useful.

### How to Add It

1. Click the `+` after `Input - Messy Notes`.
2. Search for `If`.
3. Add the `If` node.
4. Rename it to `Check Notes Present`.

### Exact Settings

Create one condition:

| Setting | Value |
|---|---|
| `Data Type` | `Boolean` |
| `Operation` | `is true` |
| `Value 1` | Expression below |

### Expressions

Use this exact IF expression:

```text
{{ !($json.raw_notes ?? '').trim() || ($json.raw_notes ?? '').trim().length < 10 }}
```

### How the branch works

- `true` output: the note is invalid, empty, null, or too short
- `false` output: the note is valid and should continue to the AI node

### Expected Output

- If the input is invalid, the item goes to the `true` branch.
- If the input is valid, the item goes to the `false` branch.

---

### Node 3B: Return Validation Error

This is the small side branch that handles bad input cleanly.

### Node Name

`Return Validation Error`

### Purpose

Returns a strict JSON error object when no usable note text exists.

### How to Add It

1. Drag from the `true` output of `Check Notes Present`.
2. Add `Edit Fields (Set)`.
3. Rename it to `Return Validation Error`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `JSON Output` |
| `Keep Only Set Fields` | `On` |

Paste this into `JSON Output`:

```json
{
  "error": "No usable intake notes found"
}
```

### Expressions

None.

### Expected Output

```json
{
  "error": "No usable intake notes found"
}
```

---

### Node 4: AI Vintage Intake Extractor

### Node Name

`AI Vintage Intake Extractor`

### Purpose

Reads the messy note and returns structured instrument inventory data as JSON.

### How to Add It

1. Drag from the `false` output of `Check Notes Present`.
2. Search for `OpenAI`.
3. Select the `OpenAI` node.
4. Rename it to `AI Vintage Intake Extractor`.

### Exact Settings

| Setting | Value |
|---|---|
| `Credential to connect with` | Your OpenAI credential |
| `Resource` | `Text` |
| `Operation` | `Generate a Chat Completion` |
| `Model` | `gpt-4o-mini` |
| `Simplify Output` | `On` |
| `Output Content as JSON` | `On` if available in your node version |
| `Options > Output Randomness (Temperature)` | `0.2` |
| `Options > Maximum Number of Tokens` | `500` |

Add two messages:

#### Message 1

| Field | Value |
|---|---|
| `Role` | `System` |
| `Text` | Use the full system prompt from Section 6 |

#### Message 2

| Field | Value |
|---|---|
| `Role` | `User` |
| `Text` | Expression below |

### Expressions

Use this exact user message:

```text
Analyze these vintage instrument intake notes and return the JSON object now:

{{ $json.raw_notes }}
```

### Expected Output

Best case, the node returns a parsed JSON object directly.

In some n8n/OpenAI combinations, it may instead return a string that contains JSON. That is why the next `Code` node is included.

Typical AI output:

```json
{
  "manufacturer": "Gibson",
  "model": "Unknown",
  "year_of_production": "Late 1950s",
  "category": "Acoustic Guitar",
  "defects": ["fret wear", "replaced tuning peg", "case odor"],
  "repairs_detected": ["neck reset", "lower bout crack repair"],
  "originality_notes": "Original tuners appear mostly present, but one peg may have been replaced.",
  "confidence_score": 83,
  "shop_grade": "Good",
  "summary": "Vintage Gibson acoustic with prior structural work, moderate wear, and warm playing character."
}
```

---

### Node 5: Normalize Intake JSON

### Node Name

`Normalize Intake JSON`

### Purpose

Parses the AI response safely and normalizes every field so the final output is always consistent.

### How to Add It

1. Click the `+` after `AI Vintage Intake Extractor`.
2. Search for `Code`.
3. Add the `Code` node.
4. Rename it to `Normalize Intake JSON`.

### Exact Settings

| Setting | Value |
|---|---|
| `Mode` | `Run Once for Each Item` |
| `Language` | `JavaScript` |

Paste this into the code editor:

```javascript
const input = $json;

function cleanText(value) {
  return typeof value === 'string' ? value.trim() : value;
}

function asArray(value) {
  if (Array.isArray(value)) {
    return value.map((item) => String(item).trim()).filter(Boolean);
  }
  if (typeof value === 'string' && value.trim()) {
    return [value.trim()];
  }
  return [];
}

function clampConfidence(value) {
  const num = Number(value);
  if (Number.isNaN(num)) return 0;
  return Math.max(0, Math.min(100, Math.round(num)));
}

const candidate =
  input.content ??
  input.text ??
  input.response ??
  input.output ??
  input.message?.content ??
  input.choices?.[0]?.message?.content ??
  input;

let parsed = candidate;

if (typeof candidate === 'string') {
  const stripped = candidate
    .trim()
    .replace(/^```json\s*/i, '')
    .replace(/^```\s*/i, '')
    .replace(/\s*```$/, '');

  parsed = JSON.parse(stripped);
}

if (!parsed || typeof parsed !== 'object' || Array.isArray(parsed)) {
  throw new Error('AI output was not a valid JSON object');
}

const allowedGrades = ['Mint', 'Excellent', 'Good', 'Fair', 'Poor'];

return [
  {
    json: {
      manufacturer: cleanText(parsed.manufacturer) || 'Unknown',
      model: cleanText(parsed.model) || 'Unknown',
      year_of_production: cleanText(parsed.year_of_production) || 'Unknown',
      category: cleanText(parsed.category) || 'Unknown',
      defects: asArray(parsed.defects),
      repairs_detected: asArray(parsed.repairs_detected),
      originality_notes: cleanText(parsed.originality_notes) || 'Unknown',
      confidence_score: clampConfidence(parsed.confidence_score),
      shop_grade: allowedGrades.includes(parsed.shop_grade) ? parsed.shop_grade : 'Good',
      summary: cleanText(parsed.summary) || 'No summary returned',
      processed_at: new Date().toISOString()
    }
  }
];
```

### Expressions

None inside the node configuration. The JavaScript handles parsing and cleanup.

### Expected Output

```json
{
  "manufacturer": "Gibson",
  "model": "Unknown",
  "year_of_production": "Late 1950s",
  "category": "Acoustic Guitar",
  "defects": ["fret wear", "replaced tuning peg", "case odor"],
  "repairs_detected": ["neck reset", "lower bout crack repair"],
  "originality_notes": "Original tuners appear mostly present, but one peg may have been replaced.",
  "confidence_score": 83,
  "shop_grade": "Good",
  "summary": "Vintage Gibson acoustic with prior structural work, moderate wear, and warm playing character.",
  "processed_at": "2026-04-17T..."
}
```

---

### Node 6: Slack - Intake Alert (Optional)

If you prefer email, use `Send Email` and place the same message in the body.

### Node Name

`Slack - Intake Alert`

### Purpose

Sends a readable summary to a manager after the AI formatting step succeeds.

### How to Add It

1. Click the `+` after `Normalize Intake JSON`.
2. Search for `Slack`.
3. Choose the message-posting action available in your workspace.
4. Rename it to `Slack - Intake Alert`.

### Exact Settings

Use your Slack credential and set the target channel, for example:

| Setting | Value |
|---|---|
| `Channel` | `#intake-review` |
| `Text` | Use the expression below |

### Expressions

Paste this into the Slack message field:

```text
New Intake Processed
Manufacturer: {{ $json.manufacturer }}
Model: {{ $json.model }}
Year: {{ $json.year_of_production }}
Grade: {{ $json.shop_grade }}
Defects: {{ Array.isArray($json.defects) && $json.defects.length ? $json.defects.join(', ') : 'None noted' }}
Summary: {{ $json.summary }}
```

### Expected Output

A Slack message like this:

```text
New Intake Processed
Manufacturer: Gibson
Model: J-45
Year: Late 1950s
Grade: Good
Defects: crack repair, fret wear
Summary: Vintage acoustic with prior neck reset and moderate player wear.
```

---

## 5. Input Example

Use these messy notes to test the workflow.

### Example 1: Vintage Gibson

```text
Picked up estate piece maybe late 50s Gibson. Sunburst mostly intact. Neck reset done years ago. Crack near lower bout repaired. Original tuners maybe replaced one peg. Frets worn medium. Case smells old. Plays warm.
```

### Example 2: Old Fender Electric

```text
Old Fender electric maybe 60s or early 70s. Blonde finish yellowed, lots of buckle rash. Bridge pickup sounds weak. Refret at some point. Neck stamp hard to read. Non original knobs maybe. Trem arm missing. Still plays, low action.
```

### Example 3: Antique Violin

```text
Antique violin from family collection, labeled maybe German workshop around turn of century. Top has repaired saddle crack. Pegs newer than body. Varnish worn on shoulders. Seam work done on treble side. Tone sweet but a little nasal up high. Old coffin case included.
```

### Quick test method

Change only the `raw_notes` value in `Input - Messy Notes` and run the workflow again.

---

## 6. AI Node Prompt (Very Important)

Use this as the `System` message in the OpenAI node.

```text
You are an expert vintage instrument appraiser for Luthier's Ledger.

Read messy buyer intake notes about string instruments and return ONLY valid JSON.
Do not return markdown.
Do not return code fences.
Do not explain your reasoning.
Do not include any text before or after the JSON object.

Return exactly one JSON object with this schema:
{
  "manufacturer": "string",
  "model": "string",
  "year_of_production": "string",
  "category": "string",
  "defects": ["string"],
  "repairs_detected": ["string"],
  "originality_notes": "string",
  "confidence_score": 0,
  "shop_grade": "Mint | Excellent | Good | Fair | Poor",
  "summary": "string"
}

Extraction rules:
- You are an expert vintage instrument appraiser.
- Infer likely values carefully when the notes strongly suggest them.
- If unknown, use "Unknown".
- If no defects are mentioned, return [].
- If no repairs are mentioned, return [].
- Consider cracks, neck resets, replaced parts, fret wear, finish wear, warping, repaired seams, loose braces, bridge issues, peg replacement, tuner replacement, and other condition clues.
- Keep defects and repairs_detected as short lowercase phrases.
- Do not invent serial numbers, provenance, or prices.
- confidence_score must be an integer from 0 to 100.
- summary must be one concise sentence for a shop manager.

Condition grading rubric:
- Mint = nearly flawless
- Excellent = minimal wear
- Good = used with moderate wear
- Fair = heavy wear or repairs
- Poor = severe structural issues

Return only JSON.
```

Use this as the `User` message:

```text
Analyze these vintage instrument intake notes and return the JSON object now:

{{ $json.raw_notes }}
```

### Why this prompt works

It does four important things:

1. Forces a single schema.
2. Tells the model how to behave when details are uncertain.
3. Defines the grading rubric.
4. Repeats "return only JSON" so the next node can parse the output cleanly.

---

## 7. IF Node Logic

The validation rule is:

- If text is empty
- Or null
- Or only spaces
- Or shorter than 10 characters

Then return:

```json
{
  "error": "No usable intake notes found"
}
```

### Exact IF node expression

```text
{{ !($json.raw_notes ?? '').trim() || ($json.raw_notes ?? '').trim().length < 10 }}
```

### Recommended branch layout

- `true` -> `Return Validation Error`
- `false` -> `AI Vintage Intake Extractor`

This keeps invalid notes away from the AI node and saves tokens.

---

## 8. Human Notification Format

Use this exact notification text in Slack or email:

```text
New Intake Processed
Manufacturer: {{ $json.manufacturer }}
Model: {{ $json.model }}
Year: {{ $json.year_of_production }}
Grade: {{ $json.shop_grade }}
Defects: {{ Array.isArray($json.defects) && $json.defects.length ? $json.defects.join(', ') : 'None noted' }}
Summary: {{ $json.summary }}
```

### Example rendered message

```text
New Intake Processed
Manufacturer: Gibson
Model: J-45
Year: Late 1950s
Grade: Good
Defects: crack repair, fret wear
Summary: Vintage acoustic with prior neck reset and moderate player wear.
```

---

## 9. Testing Section

### Basic test run

1. Open the workflow.
2. Keep the `Manual Trigger` as the first node.
3. Put one of the sample notes into `Input - Messy Notes`.
4. Click `Execute Workflow`.
5. Open each node result from left to right.

### What to verify

| Node | What you should verify |
|---|---|
| `Input - Messy Notes` | `raw_notes` contains your sample text |
| `Check Notes Present` | Valid text goes out through `false` |
| `Return Validation Error` | Only runs when the input is empty or too short |
| `AI Vintage Intake Extractor` | Returns valid JSON or a JSON-like string |
| `Normalize Intake JSON` | Final output contains all required fields with stable data types |
| `Slack - Intake Alert` | Human-readable message looks correct |

### Negative test

Change `raw_notes` to one of these and run again:

```text

```

or

```text
ok
```

Expected result:

```json
{
  "error": "No usable intake notes found"
}
```

### Quality checks for the AI result

Make sure:

- `confidence_score` is a number between 0 and 100
- `defects` is always an array
- `repairs_detected` is always an array
- `shop_grade` is one of `Mint`, `Excellent`, `Good`, `Fair`, `Poor`
- `summary` is one sentence, not a paragraph

---

## 10. Deliverables Section

When the build is finished, collect these deliverables.

### 1. Export the workflow JSON

1. Open the workflow in n8n.
2. Click the three-dot menu in the upper-right corner.
3. Select `Download`.
4. Save the `.json` file with a clear name, for example:

```text
luthiers-ledger-vintage-intake-formatter.json
```

### 2. Screenshot the canvas

Take one full screenshot showing:

- the full main path
- the IF validation side branch
- clear node names

### 3. Screenshot the AI node config

Capture:

- selected OpenAI model
- system prompt
- user prompt
- temperature
- max tokens
- JSON output option if enabled

### 4. Screenshot executions

Capture two execution examples:

- successful execution with valid notes
- failed validation execution with blank or too-short notes

### 5. Keep an execution log sample

From the `Executions` tab, keep:

- one before/after comparison
- one execution showing the structured final JSON

---

## 11. Pro Tips

- Use `Sticky Notes` on the canvas to label `Input`, `Validation`, `AI Extraction`, and `Notification`.
- Rename every node immediately. Default names become confusing fast.
- Keep the AI output strict. If the model starts returning prose, lower temperature and reinforce the JSON-only instruction.
- Enable `Retry On Fail` on the AI node if your environment occasionally hits transient API issues.
- Log `confidence_score` and review low-confidence items manually.
- Keep the `Code` node even if the AI output looks fine today. It protects you from occasional format drift.
- Start with `gpt-4o-mini` while testing. Move to `gpt-4o` only if quality is not good enough.
- If you later save to Airtable, Notion, or Google Sheets, use the output of `Normalize Intake JSON`, not the raw AI node output.

---

## 12. Bonus Section

Once manual testing is stable, replace the `Manual Trigger` with one of these production entry points.

### Option A: Webhook

Use this when another system sends intake notes into n8n.

#### Replace trigger with

`Webhook`

#### Recommended setup

| Setting | Value |
|---|---|
| `HTTP Method` | `POST` |
| `Path` | `luthiers-ledger-intake` |
| `Respond` | `When Last Node Finishes` |

#### Expected incoming JSON

```json
{
  "raw_notes": "Picked up estate piece maybe late 50s Gibson..."
}
```

#### What to change next

Update `Input - Messy Notes` so `raw_notes` pulls from the webhook body:

```json
{
  "raw_notes": "{{ $json.body.raw_notes }}",
  "source": "webhook",
  "received_at": "{{ $now.toISO() }}"
}
```

---

### Option B: Gmail Trigger

Use this when field buyers send notes by email.

#### Replace trigger with

`Gmail Trigger`

#### Recommended setup

| Setting | Value |
|---|---|
| `Event` | `Message Received` |
| `Simplify` | `On` |
| `Read Status` | `Unread emails only` |
| `Search` | `subject:(instrument intake) OR label:luthier-intake` |

#### What to map

Set `raw_notes` from the email body or snippet, depending on your Gmail payload:

```json
{
  "raw_notes": "{{ $json.snippet || $json.textPlain || $json.textHtml || '' }}",
  "source": "gmail",
  "buyer_name": "{{ $json.from?.text || 'Unknown Buyer' }}",
  "received_at": "{{ $now.toISO() }}"
}
```

Tip: If your buyers send long notes, prefer the full plain-text body field over `snippet`.

---

### Option C: Form Submission

Use this when buyers type notes into a simple intake form.

#### Replace trigger with

`n8n Form Trigger`

#### Recommended form fields

| Field Label | Field Name | Type |
|---|---|---|
| `Buyer Name` | `buyer_name` | `Text` |
| `Instrument Notes` | `raw_notes` | `Textarea` |
| `Source Location` | `source_location` | `Text` |

#### What to map

If the form field is already called `raw_notes`, you can keep the same IF logic and AI node prompt with almost no downstream changes.

Use this Set payload after the form trigger:

```json
{
  "raw_notes": "{{ $json.raw_notes }}",
  "source": "form_submission",
  "buyer_name": "{{ $json.buyer_name || 'Unknown Buyer' }}",
  "source_location": "{{ $json.source_location || 'Unknown' }}",
  "received_at": "{{ $now.toISO() }}"
}
```

---

## Final Output Schema

Your final normalized item should look like this:

```json
{
  "manufacturer": "Unknown",
  "model": "Unknown",
  "year_of_production": "Unknown",
  "category": "Unknown",
  "defects": [],
  "repairs_detected": [],
  "originality_notes": "Unknown",
  "confidence_score": 0,
  "shop_grade": "Good",
  "summary": "No summary returned",
  "processed_at": "2026-04-17T..."
}
```

Use that structure as the contract for any database, spreadsheet, or notification step you add later.

---

## Export Checklist

- Workflow named clearly
- All nodes renamed clearly
- Manual test completed with Gibson sample
- Manual test completed with Fender sample
- Manual test completed with violin sample
- Empty-input validation tested
- AI node prompt pasted exactly
- Code node parsing tested
- Optional Slack or email notification tested
- Workflow exported as `.json`
- Canvas screenshot captured
- AI node screenshot captured
- Execution screenshots captured

---

## Official Reference Links

These official docs were used to align this guide with the current n8n browser UI and node naming:

- n8n Manual Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.manualworkflowtrigger/
- n8n Edit Fields (Set): https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
- n8n If node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
- n8n Code node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/
- n8n OpenAI node text operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/text-operations/
- n8n Gmail Trigger: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger/
- n8n Webhook: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- n8n Form Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger/
- n8n workflow export/import: https://docs.n8n.io/workflows/export-import/

### Postwork Project
```
AI-Powered Vintage Instrument Intake Formatter for Luthier’s Ledger
Overview Luthier’s Ledger is a boutique music shop specializing in rare, vintage string instruments. The Project Manager currently receives disorganized intake notes from field buyers via email and needs an n8n workflow that uses AI to instantly clean, categorize, and grade these notes into a structured format for their inventory database.

Requirements

Input Handling: A workflow that can receive a block of "messy" text (e.g., a buyer's rough notes about a 1960s Fender or a 19th-century violin).
AI Transformation: Use an n8n AI node (AI Agent or Chain node) to extract specific data points: Manufacturer, Model, Year of Production, and a list of identified defects.
Automated Grading: The AI must analyze the notes to assign a "Shop Grade" (Mint, Excellent, Good, Fair, or Poor) based on the description of wear and repairs mentioned.
Logical Formatting: The output must be a clean JSON object ready for database insertion and a human-readable summary for a staff Slack or email notification.
Error Handling: Include a basic "No Data" check to ensure the workflow doesn't fail if the input text is empty or nonsensical.
Deliverables

n8n Workflow Export: The exported .json file of the workflow.
Screenshots: Clear images of the full workflow canvas and the internal configuration of the AI node/prompt.
Execution Sample: A screenshot or screen recording showing the "before" (messy input) and "after" (structured output) within the n8n execution log.
Success Looks Like The workflow successfully parses inconsistent, shorthand notes into a perfectly structured format without losing technical details about the instruments. Your work will be judged on the accuracy of the AI extraction and the cleanliness of the workflow logic.
```
