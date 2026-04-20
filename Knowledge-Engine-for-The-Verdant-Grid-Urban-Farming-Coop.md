# Knowledge Engine for The Verdant Grid Urban Farming Coop (n8n Browser Build Guide)

This guide walks you through building an internal, **zero-hallucination** Crop Consultant in **n8n (browser version)** using **two workflows**:

- **Workflow A — Knowledge Ingestion:** captures and verifies crop knowledge into an internal database
- **Workflow B — Field Consultant AI:** answers questions **only** from approved internal records using semantic search + strict grounding

---

## 1. Project Overview

The Verdant Grid coop has valuable growing knowledge spread across teams and zones. This system:

1. **Collects** crop care knowledge via a form (staff + guests)
2. **Normalizes & labels** each submission (metadata + category + keywords)
3. **Stores** it in a structured internal **n8n Data Table**
4. **Indexes** only *Approved* knowledge into a vector store for semantic search
5. **Answers** field questions using an AI agent that is **strictly grounded** in retrieved internal records

Why it matters:

- Consistent crop care across zones
- Faster pest/irrigation responses
- Reliable onboarding for new managers
- Prevents misinformation: **if there’s no approved record, the AI refuses**

---

## 2. Final System Architecture

### Workflow A: Knowledge Ingestion (Form → Clean → Trust Check → AI Metadata → Store)

Form Trigger  
→ Set Node (normalize data)  
→ IF Node (security contributor check)  
→ AI Node (crop classification + metadata)  
→ Data Table Insert  
→ Optional notification

### Workflow B: Field Consultant AI (Chat/Webhook → Memory → Semantic Search → Grounded AI → Output)

Chat Trigger / Webhook  
→ Memory Node  
→ Semantic Search (vector store) + record fetch (Data Table)  
→ AI Agent with strict grounding  
→ Response Output

### Workflow C (Recommended): Error Handling

Error Trigger workflow  
→ Slack/Email admin alert (execution details)

---

## 3. Tools Needed

Inside n8n (browser version), you will use:

- **Form Trigger** (n8n Forms) *or* **Webhook** (for Google Forms forwarding)
- **Set** node
- **IF** node
- **AI Agent** node *or* **Basic LLM Chain** node (Agent recommended)
- **Embeddings** node
- **Vector Store** node (for semantic search)
- **n8n Data Table** node
- **Chat Trigger** *or* **Webhook** (chat endpoint)
- **Memory** node
- Optional **Slack** *or* **Email** node
- **Error Trigger** node (separate workflow)

External accounts/keys (pick one path and stick to it):

- **LLM provider** credential (for the AI nodes)
- **Embedding provider** credential (often the same provider)
- **Vector database** credential (e.g., Pinecone/Qdrant/Supabase vector store) used by the Vector Store node

This guide is written so you can complete the build with **any** provider supported by your n8n instance. Where a setting depends on a provider (model selection, index dimension), the guide explicitly calls it out.

### 3.1 Recommended Reference Setup (Fastest Path)

If you want a single, “just works” reference stack, use:

- **LLM + Embeddings:** your preferred provider credential (same provider for both is simplest)
- **Vector DB:** a managed vector database supported by your n8n **Vector Store** node

Below is a concrete setup you can follow end-to-end.

#### Option A (Recommended): OpenAI + Pinecone

1. **Choose your embedding model first** (because it determines the vector dimension).
   - Example (common): `text-embedding-3-small` (dimension typically `1536`)
   - To avoid guessing: run any **Embeddings** node once with a short test string and confirm the embedding array length (that length is your dimension).
2. In Pinecone:
   - Create an **index** named: `verdant-grid-crop-kb`
   - Set **dimension** to your embedding model’s dimension
   - Set **metric** to `cosine`
3. In n8n:
   - Create credentials for your **LLM provider** (for AI nodes)
   - Create credentials for your **Embedding provider** (for Embeddings nodes)
   - Create credentials for **Pinecone** (API key, index name, environment/host as required by your n8n node)

#### Option B: OpenAI + Qdrant Cloud

1. Choose your embedding model and note its **dimension**.
   - To avoid guessing: run any **Embeddings** node once with a short test string and confirm the embedding array length.
2. In Qdrant Cloud:
   - Create a **collection** named: `verdant_grid_crop_kb`
   - Set vector size to your embedding model’s dimension
   - Distance: `Cosine`
3. In n8n:
   - Create credentials for the provider (AI + Embeddings)
   - Create credentials for Qdrant (URL + API key)

Important: In **Workflow A** indexing and **Workflow B** querying, you must use the **same embedding model** (same dimension), or your vector store queries will fail.

---

## 4. Core Business Rules (Enforced by Workflow Logic)

1. **Internal database is the only source of truth.**
2. **No matching approved crop data → AI must refuse** (no outside farming knowledge).
3. **Guest Contributors** submissions are stored as **Pending** (excluded from semantic search).
4. **Staff** submissions are **Approved** automatically.
5. The agent supports **follow-up questions** using session memory.
6. Advice must reflect **Verdant Grid methods only** (zone-specific if available).

---

## 5. Workflow A – Ingestion Pipeline

### What you will build

A form intake that creates:

- A structured row in `Crop_Knowledge_Base` (Data Table)
- For **Approved** rows only: a semantic-search document in your **Vector Store**

### 5.1 Form Fields

Create a form with these fields (labels exactly as shown to match mappings below):

| Field Label | Type | Required |
|---|---|---|
| Contributor Name | Text | Yes |
| Contributor Role (Staff / Guest Contributor) | Dropdown | Yes |
| Species | Text | Yes |
| Soil pH | Text (or Number if you prefer) | No |
| Irrigation Frequency | Text | No |
| Pest Warnings | Long text | No |
| Notes | Long text | No |
| Zone Location | Text | Yes |

Recommended dropdown values for **Contributor Role**:

- `Staff`
- `Guest Contributor`

### 5.2 Trigger Setup

#### Option A (Recommended): n8n Form Trigger

1. Create a new workflow named: **VG — Workflow A — Knowledge Ingestion**
2. Add node: **Form Trigger**
3. In the Form Trigger node:
   - Create a new form (use the fields table above)
   - Ensure the form is **published/active** (whatever your n8n UI calls it)
4. Submit one test entry so n8n captures sample input for mapping.

#### Option B: Google Form → n8n Webhook (only if you must)

If you already use Google Forms, you can forward submissions to n8n via an Apps Script web app. This is outside n8n, so keep it only if required by your organization.

High-level:

- Google Form submits → Apps Script receives → Apps Script `UrlFetchApp.fetch()` posts JSON to n8n **Webhook** node.

If you want this option, tell me your preferred Google Form schema and I’ll generate the exact Apps Script + deployment steps.

---

### 5.3 Set Node Cleanup Logic (Normalize Fields)

Add node: **Set**

Rename it: **Normalize — Submission Data**

In the Set node:

- Turn on: **Keep Only Set** (recommended to avoid carrying unexpected fields)
- Add these fields (exact names below) and map values using expressions:

| Output Field | Type | Value (Expression) |
|---|---:|---|
| record_id | String | `{{ "VG-" + Date.now() }}` |
| created_at | String | `{{ $now }}` |
| contributor_name | String | `{{ ($json["Contributor Name"] || "").trim() }}` |
| contributor_role | String | `{{ ($json["Contributor Role (Staff / Guest Contributor)"] || "").trim() }}` |
| species | String | `{{ ($json["Species"] || "").trim().toLowerCase() }}` |
| soil_ph | String | `{{ ($json["Soil pH"] || "").toString().trim() }}` |
| irrigation_frequency | String | `{{ ($json["Irrigation Frequency"] || "").trim() }}` |
| pest_warnings | String | `{{ ($json["Pest Warnings"] || "").trim() }}` |
| notes | String | `{{ ($json["Notes"] || "").trim() }}` |
| zone_location | String | `{{ ($json["Zone Location"] || "").trim() }}` |
| status | String | `{{ "Pending" }}` |

Also add an internal “document text” field for embeddings later:

| Output Field | Type | Value (Expression) |
|---|---:|---|
| source_text | String | `{{ "species: " + $json.species + "\nzone: " + $json.zone_location + "\nsoil_ph: " + $json.soil_ph + "\nirrigation_frequency: " + $json.irrigation_frequency + "\npest_warnings: " + $json.pest_warnings + "\nnotes: " + $json.notes }}` |

---

### 5.3.1 Required Field Validation (Prevents Empty Submissions)

Add node: **IF**

Rename: **Gate — Required Fields Present?**

Condition expression:

```text
{{
  ($json.contributor_name || "").trim().length > 0
  && ($json.contributor_role || "").trim().length > 0
  && ($json.species || "").trim().length > 0
  && ($json.zone_location || "").trim().length > 0
}}
```

Routing:

- **True:** continue to **Route — Contributor Trust Check**
- **False:** optional Slack/Email to admins + stop the branch (do not write to tables)

---

### 5.4 Security Filter Logic (Staff vs Guest)

Add node: **IF**

Rename it: **Route — Contributor Trust Check**

Condition (Boolean):

- **Value 1 (Expression):** `{{ $json.contributor_role }}`
- **Operation:** `equals`
- **Value 2:** `Staff`

Branch behavior:

- **True (Staff):** mark as **Approved**, proceed to AI classification + vector indexing
- **False (Guest Contributor):** keep **Pending**, store in table, do *not* index into vector store

To set the status value cleanly, add a **Set** node on each branch:

#### True branch Set

Add node: **Set** (connect from IF → true)

Rename: **Set — Status Approved**

- Keep Only Set: **off** (so you don’t drop the normalized fields)
- Field: `status` = `Approved` (string)

#### False branch Set

Add node: **Set** (connect from IF → false)

Rename: **Set — Status Pending**

- Keep Only Set: **off**
- Field: `status` = `Pending` (string)

---

### 5.5 AI Classification Node (Crop Category + Metadata)

On the **Approved** branch only, add node: **AI** (use **AI Agent** or **Basic LLM Chain**; for ingestion, Basic LLM Chain is usually simpler).

Rename: **AI — Crop Classification**

#### Prompt (copy/paste)

System / Instructions:

```text
You are an agricultural taxonomy assistant for The Verdant Grid coop.

You must return ONLY valid JSON.
Do not include markdown. Do not include explanations.

Input fields:
- Species
- Soil pH
- Irrigation Frequency
- Pest Warnings
- Notes

Classify into exactly one category from this list:
- Leafy Greens
- Root Veg
- Fruit Crop
- Herb
- Legume
- Vine Crop
- Unknown

Also return:
- care_intensity: "Low" | "Medium" | "High"
- likely_risks: array of strings (pests/diseases/conditions suggested by the input)
- search_keywords: array of strings (include synonyms and common terms; keep them lowercase)
- short_summary: string (one sentence max)

Return JSON with keys:
category, care_intensity, likely_risks, search_keywords, short_summary
```

User / Input mapping (use expressions so the model sees your normalized fields):

```text
Species: {{$json.species}}
Soil pH: {{$json.soil_ph}}
Irrigation Frequency: {{$json.irrigation_frequency}}
Pest Warnings: {{$json.pest_warnings}}
Notes: {{$json.notes}}
```

#### Output handling

Immediately after the AI node, add a **Set** node that **assembles a complete “Approved record” payload** (because many AI nodes output only the model result and do not automatically keep your original input fields).

Add node: **Set**

Rename: **Assemble — Approved Record**

Add fields:

| Field | Value (Expression) |
|---|---|
| record_id | `{{ $node["Set — Status Approved"].json.record_id }}` |
| created_at | `{{ $node["Set — Status Approved"].json.created_at }}` |
| contributor_name | `{{ $node["Set — Status Approved"].json.contributor_name }}` |
| contributor_role | `{{ $node["Set — Status Approved"].json.contributor_role }}` |
| species | `{{ $node["Set — Status Approved"].json.species }}` |
| soil_ph | `{{ $node["Set — Status Approved"].json.soil_ph }}` |
| irrigation_frequency | `{{ $node["Set — Status Approved"].json.irrigation_frequency }}` |
| pest_warnings | `{{ $node["Set — Status Approved"].json.pest_warnings }}` |
| notes | `{{ $node["Set — Status Approved"].json.notes }}` |
| zone_location | `{{ $node["Set — Status Approved"].json.zone_location }}` |
| status | `{{ $node["Set — Status Approved"].json.status }}` |
| category | `{{ $json.category }}` |
| care_intensity | `{{ $json.care_intensity }}` |
| likely_risks | `{{ $json.likely_risks }}` |
| search_keywords | `{{ $json.search_keywords }}` |
| short_summary | `{{ $json.short_summary }}` |

If your AI node returns the JSON as a string field (some configurations do), add one extra step before mapping:

- Use a node that parses JSON (if your n8n build provides it) **or**
- Configure the AI node to “Return JSON / Structured output” if available

---

### 5.6 Data Table Structure (Internal Source of Truth)

Create an **n8n Data Table** named:

**`Crop_Knowledge_Base`**

Columns (create exactly these; use text type unless you prefer number/JSON where supported):

| Column | Suggested Type |
|---|---|
| record_id | Text |
| species | Text |
| category | Text |
| soil_ph | Text |
| irrigation_frequency | Text |
| pest_warnings | Text |
| zone_location | Text |
| contributor_name | Text |
| contributor_role | Text |
| status | Text |
| care_intensity | Text |
| likely_risks | JSON/Text |
| short_summary | Text |
| notes | Text |
| created_at | Text/DateTime |

Recommended additional column (makes admin review + auditing easier):

| Column | Suggested Type |
|---|---|
| search_keywords | JSON/Text |

---

### 5.7 Insert Logic (Store Structured Records)

You will insert **both** Approved and Pending submissions into `Crop_Knowledge_Base`.

#### Approved branch insert

Add node: **Data Table**

Rename: **Save — Data Table Record (Approved)**

Operation: **Insert**

Table: `Crop_Knowledge_Base`

Map columns:

- `record_id` → `{{$json.record_id}}`
- `species` → `{{$json.species}}`
- `category` → `{{$json.category}}`
- `soil_ph` → `{{$json.soil_ph}}`
- `irrigation_frequency` → `{{$json.irrigation_frequency}}`
- `pest_warnings` → `{{$json.pest_warnings}}`
- `zone_location` → `{{$json.zone_location}}`
- `contributor_name` → `{{$json.contributor_name}}`
- `contributor_role` → `{{$json.contributor_role}}`
- `status` → `{{$json.status}}`
- `care_intensity` → `{{$json.care_intensity}}`
- `likely_risks` → `{{$json.likely_risks}}`
- `short_summary` → `{{$json.short_summary}}`
- `search_keywords` → `{{$json.search_keywords}}` *(only if you added the optional column)*
- `notes` → `{{$json.notes}}`
- `created_at` → `{{$json.created_at}}`

#### Pending branch insert

Add node: **Data Table**

Rename: **Save — Data Table Record (Pending)**

Operation: **Insert** into the same table using the same mappings.

---

### 5.8 Semantic Indexing (Approved Only)

This is the key to “Approved-only semantic search”.

On the **Approved** branch, after saving the Data Table record:

1. Create an embeddings vector for the record text
2. Upsert it into your vector store with metadata including `status: Approved`

Note: `Crop_Knowledge_Base` remains the **source of truth**. The vector store is only a search index (embedding + text + metadata) used by Workflow B.

#### Step A — Create the final indexable text

Add node: **Set**

Rename: **Build — Index Document**

Fields:

- `record_id` (String): `{{ $node["Assemble — Approved Record"].json.record_id }}`
- `species` (String): `{{ $node["Assemble — Approved Record"].json.species }}`
- `zone_location` (String): `{{ $node["Assemble — Approved Record"].json.zone_location }}`
- `status` (String): `{{ $node["Assemble — Approved Record"].json.status }}`
- `document_text` (String):

```text
{{ 
  "Verdant Grid Crop Knowledge Record\n"
  + "record_id: " + $node["Assemble — Approved Record"].json.record_id + "\n"
  + "species: " + $node["Assemble — Approved Record"].json.species + "\n"
  + "category: " + $node["Assemble — Approved Record"].json.category + "\n"
  + "zone: " + $node["Assemble — Approved Record"].json.zone_location + "\n"
  + "soil_ph: " + $node["Assemble — Approved Record"].json.soil_ph + "\n"
  + "irrigation_frequency: " + $node["Assemble — Approved Record"].json.irrigation_frequency + "\n"
  + "pest_warnings: " + $node["Assemble — Approved Record"].json.pest_warnings + "\n"
  + "care_intensity: " + $node["Assemble — Approved Record"].json.care_intensity + "\n"
  + "likely_risks: " + JSON.stringify($node["Assemble — Approved Record"].json.likely_risks || []) + "\n"
  + "short_summary: " + $node["Assemble — Approved Record"].json.short_summary + "\n"
  + "notes: " + $node["Assemble — Approved Record"].json.notes 
}}
```

#### Step B — Embeddings node

Add node: **Embeddings**

Rename: **Embeddings — Crop Record**

- Input text: `{{$json.document_text}}`
- Model: select the embedding model your provider supports

Important: You must know the embedding **vector dimension** because your vector store index/collection must match it.

#### Step C — Vector Store Upsert node

Add node: **Vector Store**

Rename: **Vector — Upsert Approved Record**

Operation: **Upsert / Insert Documents**

Configure:

- **ID:** `{{$json.record_id}}`
- **Text:** `{{$json.document_text}}`
- **Embedding:** map from the Embeddings node output
- **Metadata (JSON):**

```json
{
  "record_id": "{{$json.record_id}}",
  "species": "{{$json.species}}",
  "zone_location": "{{$json.zone_location}}",
  "status": "{{$json.status}}"
}
```

Your Vector Store node must be configured to use your chosen vector DB (Pinecone/Qdrant/etc.). Create the credentials in n8n first, then select them in this node.

---

### 5.9 Optional Notification (Approved vs Pending)

After each insert, you can add a Slack or Email node:

- Approved message: “New Approved knowledge added: `species` in `zone` (record_id: …)”
- Pending message: “New Pending submission requires review: contributor + species + zone”

Name suggestions:

- **Notify — Slack (Approved)**
- **Notify — Slack (Pending)**

---

## 6. Workflow B – Semantic Field Agent

### What you will build

A chat endpoint that:

1. Loads session memory
2. Performs semantic search (Approved only)
3. If nothing found → refuses
4. If found → AI answers strictly using retrieved records
5. Updates memory for follow-ups (including “last crop” context)

---

### 6.1 Trigger Options

Pick **one**:

#### Option A (Recommended): Chat Trigger

Create a new workflow named: **VG — Workflow B — Field Consultant AI**

Add node: **Chat Trigger**

This typically provides:

- A user message text (often called `chatInput`, `message`, or similar)
- A `sessionId` (or similar) for memory

Immediately after the trigger, add a **Set** node to normalize these fields (so every downstream node can rely on the same keys regardless of trigger type).

Add node: **Set**

Rename: **Normalize — Chat Input**

- Turn on: **Keep Only Set**
- Create fields:
  - `session_id`: map from the Chat Trigger’s session id field (pick it from the input list in the expression editor)
  - `user_question`: map from the Chat Trigger’s message field

#### Option B: Webhook (API endpoint)

Add node: **Webhook**

Settings:

- Method: `POST`
- Path: `verdant-grid/field-consult`
- Response: (use a final “Respond” node or the webhook response setting, depending on your n8n UI)

Expected JSON body (example):

```json
{
  "session_id": "zone3-manager-001",
  "question": "Tomato leaves curling in Zone 3, what should I check?"
}
```

Immediately after the Webhook node, add the same **Normalize — Chat Input** Set node:

- `session_id`: `{{ $json.session_id }}`
- `user_question`: `{{ $json.question }}`

---

### 6.2 User Questions Examples (Use These During Testing)

- “Tomato leaves curling in Zone 3, what should I check?”
- “How often do we irrigate kale beds?”
- “Soil pH for basil in rooftop containers?”
- “Aphids on lettuce — follow-up from earlier advice”

---

### 6.3 Memory Configuration (Multi-turn Follow-ups)

Add node: **Memory**

Rename: **Memory — Session Context**

Configure:

- **Session ID:** map from the trigger’s session field
  - Use: `{{ $node["Normalize — Chat Input"].json.session_id }}`
- **Memory type:** choose a rolling window / conversation buffer if available
- **Window size:** 8–12 messages (enough for follow-ups, not too large)

#### Persist “last discussed crop” (recommended)

Memory nodes usually store chat history, but you also want structured session state. Create a second Data Table:

**`Crop_Consultant_Sessions`**

Columns:

- `session_id`
- `last_species`
- `last_zone_location`
- `last_issue`
- `updated_at`

In Workflow B you will:

1. Load session row by `session_id`
2. Use it to disambiguate follow-ups (“What about aphids?”)
3. Update it after answering

---

### 6.4 Semantic Search Setup (Approved Only)

#### Step 1 — Load session state (optional but recommended)

Add node: **Data Table**

Rename: **Load — Session State**

Operation: **Get / Find**

Table: `Crop_Consultant_Sessions`

Filter:

- `session_id` equals `{{ $node["Normalize — Chat Input"].json.session_id }}`

Important (so new sessions don’t break the workflow):

- In the node’s **Settings** tab, enable **Always Output Data**.

This ensures the workflow continues even when no session row is found.

If no record exists for a new session, you can either:

- Insert a blank session row (recommended), or
- Continue with empty state (works, but follow-ups won’t be disambiguated until after first answer)

#### Step 2 — Build the effective search query

Add node: **Set**

Rename: **Build — Search Query**

Fields:

- `user_question`: `{{ $node["Normalize — Chat Input"].json.user_question }}`
- `session_id`: `{{ $node["Normalize — Chat Input"].json.session_id }}`

- `last_species` (from session table, if present):
  - `{{ ((($items("Load — Session State")[0] || {}).json || {}).last_species) || "" }}`

- `effective_query` (this is what you embed + search):

```text
{{
  ($json.user_question || "").trim().toLowerCase()
  + (
    ($json.last_species || "").trim()
      ? ("\ncontext_last_species: " + $json.last_species.trim().toLowerCase())
      : ""
    )
}}
```

This makes follow-up questions like “What about aphids?” retrieve the same crop record set as the previous turn.

#### Step 3 — Embed the query

Add node: **Embeddings**

Rename: **Embeddings — User Query**

- Input text: `{{$json.effective_query}}`
- Model: the same embedding model you used for indexing (must match dimension)

#### Step 4 — Vector search (Top K)

Add node: **Vector Store**

Rename: **Search — Semantic Lookup**

Operation: **Query / Similarity Search**

Settings:

- **Top K:** `3` to `5`
- **Filter:** `status = "Approved"` (use your vector store node’s metadata filter UI)

Output should include:

- matched document text
- similarity score (if available)
- metadata (must include `record_id`, `species`, `zone_location`, `status`)

#### Step 5 — Zero-hallucination gate

Add node: **IF**

Rename: **Gate — Any Approved Match?**

Condition:

- Check that the search returned items.
- Use expression:

```text
{{ $items("Search — Semantic Lookup").length > 0 }}
```

**False branch:** respond with refusal (Section 7).  
**True branch:** pass retrieved records into the AI Agent.

---

### 6.5 AI Agent Strict Grounding Prompt

On the **True** branch of the gate, add node: **AI Agent**

Rename: **AI — Crop Consultant**

#### Inputs to provide the agent

Make sure the agent receives:

- `question` = `{{$json.user_question}}`
- `retrieved_records` = the vector search results (document text + metadata)
- `session_context` = memory history (from Memory node output)

If your AI Agent node supports “Tools / Context”, paste retrieved records into a single variable (recommended) using a Set node:

Add node: **Set**

Rename: **Build — Retrieved Context**

Field `retrieved_context`:

```text
{{
  $items("Search — Semantic Lookup")
    .map(i =>
      "RECORD\n"
      + "record_id: " + (((i.json.metadata || {}).record_id) || "") + "\n"
      + "species: " + (((i.json.metadata || {}).species) || "") + "\n"
      + "zone_location: " + (((i.json.metadata || {}).zone_location) || "") + "\n"
      + "status: " + (((i.json.metadata || {}).status) || "") + "\n"
      + "content:\n" + (i.json.text || i.json.document || i.json.content || "")
    )
    .join("\n\n---\n\n")
}}
```

#### Strict Grounding Prompt (copy/paste)

System:

```text
You are The Verdant Grid Crop Consultant.

Hard rules (must follow):
1) Use ONLY the retrieved internal records provided in RETRIEVED_RECORDS.
2) Never use outside farming knowledge. Never guess. Never generalize beyond records.
3) If the records do not contain guidance relevant to the question, respond exactly:
"No approved internal guidance exists for that crop yet."
4) Be concise and practical (bulleted steps preferred).
5) Cite the species and zone_location from the records when relevant.
6) If pests are mentioned, prioritize Pest Warnings from records.
7) Use conversation context to interpret follow-up questions, but still answer only from records.
```

User message template:

```text
QUESTION:
{{$json.user_question}}

SESSION_CONTEXT (may be empty):
{{$json.memory || $json.session_context || ""}}

RETRIEVED_RECORDS:
{{$json.retrieved_context}}
```

#### Example Good Response (what “grounded” looks like)

```text
Based on approved Kale records from Zone 2:
- Soil pH: 6.2–6.8
- Irrigation: every 2 days during dry weeks
- Watch for: aphids, mildew (see pest warnings)
```

---

### 6.6 Response Output

Add a final output node depending on your trigger:

- **Chat Trigger:** use the Chat response/output node (whatever your n8n UI pairs with Chat Trigger)
- **Webhook:** use “Respond to Webhook” (or configure the Webhook node response)

Response body (Webhook example):

```json
{
  "answer": "={{$json.text || $json.output || $json.response}}"
}
```

### 6.7 Session State Update (Makes Follow-ups Work Reliably)

After you generate the answer (on the **True** branch where you called the AI), update `Crop_Consultant_Sessions` so the next message can inherit context.

If your trigger requires the response node to be the last node (common for Webhook-based chat), place the session update **before** your **Respond** node.

#### Step 1 — Extract “last crop” from top retrieved match

Add node: **Set**

Rename: **Set — Update Session State Payload**

Fields:

- `session_id`:
  - `{{ $node["Normalize — Chat Input"].json.session_id }}`
- `last_species`:

```text
{{ (($items("Search — Semantic Lookup")[0].json.metadata || {}).species) || "" }}
```

- `last_zone_location`:

```text
{{ (($items("Search — Semantic Lookup")[0].json.metadata || {}).zone_location) || "" }}
```

- `last_issue` (simple default: the latest question text):

```text
{{ $node["Normalize — Chat Input"].json.user_question }}
```

- `updated_at`:

```text
{{ $now }}
```

#### Step 2 — Insert vs Update

Use the output of **Load — Session State** to decide whether to insert a new session row or update an existing one.

Add node: **IF**

Rename: **Gate — Session Exists?**

Condition expression:

```text
{{ $items("Load — Session State").length > 0 }}
```

##### True branch (Update)

Add node: **Data Table**

Rename: **Save — Session State (Update)**

Table: `Crop_Consultant_Sessions`

Operation: **Update**

Filter:

- `session_id` equals `{{$json.session_id}}`

Set columns:

- `last_species` → `{{$json.last_species}}`
- `last_zone_location` → `{{$json.last_zone_location}}`
- `last_issue` → `{{$json.last_issue}}`
- `updated_at` → `{{$json.updated_at}}`

##### False branch (Insert)

Add node: **Data Table**

Rename: **Save — Session State (Insert)**

Table: `Crop_Consultant_Sessions`

Operation: **Insert**

Map columns:

- `session_id` → `{{$json.session_id}}`
- `last_species` → `{{$json.last_species}}`
- `last_zone_location` → `{{$json.last_zone_location}}`
- `last_issue` → `{{$json.last_issue}}`
- `updated_at` → `{{$json.updated_at}}`

---

## 7. Zero Hallucination Enforcement (Required)

If semantic search returns **no** rows (Approved-only filter already applied), do **not** call the AI model.

On the **False** branch of **Gate — Any Approved Match?**, respond:

```text
No approved internal guidance exists for that crop yet.
```

Optional (but useful): also include a next step for staff:

```text
If you have verified Verdant Grid guidance, submit it via the intake form so it can be approved and indexed.
```

---

## 8. Error Handling (Separate Workflow)

Create a third workflow named:

**VG — Workflow C — Error Alerts**

### Nodes

1. **Error Trigger**
2. **Slack** *or* **Email** (to admins)

### Message template (copy/paste)

```text
Verdant Grid n8n Error

Workflow: {{$json.workflow.name}}
Failed node: {{$json.node.name}}
Error: {{$json.error.message}}
Time: {{$now}}
Execution ID: {{$json.execution.id}}
```

### What this catches

- Empty form submissions (validation failures)
- Embedding/vector store failures
- Data Table write errors
- Chat timeouts / provider errors

---

## 9. Testing Section (Do These 5 Tests End-to-End)

### Test 1 — Staff Submission (Auto-approved)

1. Submit form with:
   - Contributor Role: `Staff`
   - Species: `kale`
   - Zone: `Zone 2`
2. Verify in `Crop_Knowledge_Base`:
   - `status` = `Approved`
   - AI fields populated (`category`, `care_intensity`, `likely_risks`, `short_summary`; plus `search_keywords` if you added the optional column)
3. Verify the record is in your vector store (search by `record_id` if your vector DB UI supports it).

### Test 2 — Guest Submission (Pending)

1. Submit form with:
   - Contributor Role: `Guest Contributor`
   - Species: `carrot`
2. Verify in `Crop_Knowledge_Base`:
   - `status` = `Pending`
3. Verify it is **not** present in vector store (or cannot be retrieved due to `status=Approved` filter).

### Test 3 — Agent Answer (Grounded)

1. Ask: “How often do we irrigate kale beds?”
2. Verify response:
   - Mentions Kale (species)
   - Uses internal fields (e.g., irrigation frequency)
   - Does **not** add generic farming advice absent from records

### Test 4 — Unknown Crop Refusal

1. Ask about a crop not in approved records, e.g., “Dragonfruit care?”
2. Verify exact refusal:
   - `No approved internal guidance exists for that crop yet.`

### Test 5 — Follow-up Memory

1. Ask a first question about lettuce pests.
2. Then ask: “What about aphids?”
3. Verify it:
   - Uses last crop context (lettuce) for retrieval
   - Answers only using retrieved records

---

## 10. Deliverables (Submission Checklist)

Produce these artifacts:

- Export JSON for **Workflow A — Knowledge Ingestion**
- Export JSON for **Workflow B — Field Consultant AI**
- Screenshot: `Crop_Knowledge_Base` populated (show at least 3 records)
- Screenshot: AI Agent config (prompt visible)
- Screenshot: vector store setup (index/collection + example record)
- Screenshot: Approved vs Pending records (filter or side-by-side proof)
- 2-minute demo video:
  1) Submit Staff form entry  
  2) Ask field question → grounded answer  
  3) Ask follow-up → memory works  

### Export steps (n8n browser UI)

For each workflow:

1. Open the workflow
2. Use the workflow menu (commonly a **⋯** button)
3. Choose **Download/Export** workflow JSON
4. Name files:
   - `VG-Workflow-A-Knowledge-Ingestion.json`
   - `VG-Workflow-B-Field-Consultant-AI.json`

---

## 11. Naming Convention Best Practices (Use These Exact Node Names)

### Workflow A

- **Trigger — Crop Intake Form**
- **Normalize — Submission Data**
- **Gate — Required Fields Present?**
- **Route — Contributor Trust Check**
- **Set — Status Approved**
- **Set — Status Pending**
- **AI — Crop Classification**
- **Assemble — Approved Record**
- **Save — Data Table Record (Approved)**
- **Save — Data Table Record (Pending)**
- **Build — Index Document**
- **Embeddings — Crop Record**
- **Vector — Upsert Approved Record**

### Workflow B

- **Trigger — Field Chat**
- **Normalize — Chat Input**
- **Memory — Session Context**
- **Load — Session State**
- **Build — Search Query**
- **Embeddings — User Query**
- **Search — Semantic Lookup**
- **Gate — Any Approved Match?**
- **Build — Retrieved Context**
- **AI — Crop Consultant**
- **Set — Update Session State Payload**
- **Gate — Session Exists?**
- **Save — Session State (Update)**
- **Save — Session State (Insert)**
- **Respond — Chat Output**

---

## 12. Pro Tips (Practical)

- Keep `species` standardized (lowercase key); store display variants in `search_keywords`.
- Add zone-specific records instead of mixing multiple zones in one entry.
- Put synonyms in `search_keywords` (e.g., `tomato`, `solanum lycopersicum`, local nicknames).
- Keep AI responses short: 3–7 bullets; field teams need action, not essays.
- Review Pending entries weekly and re-enter as Staff-approved (or build an admin approval workflow later).
- Back up `Crop_Knowledge_Base` regularly (export or replicate to your org’s database).

---

## 13. Bonus Improvements (Extensions You Can Add Later)

### A) Photo-based pest diagnosis intake

- Add a file upload field to the intake form
- Use an image-capable model node to extract symptoms into structured text
- Store image URL + extracted symptoms in the record (still require Staff approval before indexing)

### B) Seasonal planting calendar reminders

- Add a schedule/cron workflow
- Read “seasonal calendar” records from a new Data Table
- Notify zone managers weekly with tailored reminders

### C) Zone performance analytics

- Create a reporting workflow that aggregates:
  - issues by pest type
  - irrigation frequency patterns by zone
  - knowledge gaps (questions with refusals)

### D) Auto-translate for multilingual staff

- Detect user language in Workflow B
- Translate the question → search in English keywords → answer → translate back
- Keep the grounding rule: translation cannot introduce new guidance

### E) Confidence score on retrieval

- If your vector store returns similarity scores:
  - Show “Confidence: High/Medium/Low” based on thresholds
  - If low confidence, ask a clarifying question or refuse

### F) Escalation to a human agronomist

- If the agent refuses or confidence is low:
  - Create a ticket (Slack/Email) with question + retrieved context + zone
  - Track resolution and ingest the verified answer as a new Staff record

---

## Appendix: Review/Approval Workflow (Optional but Useful)

To operationalize “Guest Contributors require Pending review”:

1. Create a workflow that lists `Crop_Knowledge_Base` where `status = Pending`
2. Sends a weekly Slack message to approvers with record summaries
3. Approver clicks a link (Webhook) to mark a `record_id` as Approved
4. On approval, index that record into the vector store (same steps as 5.8)

If you want, I can generate this third workflow with exact nodes and mappings too.
