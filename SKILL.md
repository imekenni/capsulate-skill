---
name: capsulate
description: Turn Gmail into structured dashboards — extract and track subscriptions, packages, invoices, job applications, and more.
disable-model-invocation: true
---

# Capsulate — Email Intelligence Skill

Transform your inbox into structured, queryable dashboards using AI. No backend required — everything runs locally.

## What This Does

1. You describe what you want to track from your emails
2. Claude generates an extraction schema and Gmail search query
3. Your emails are searched and fetched via the `gws` CLI
4. Subagents extract structured data from each email in parallel
5. Data is saved locally to `~/.capsulate/<canvas>.json`
6. A self-contained HTML dashboard is generated and opened in your browser

---

## Prerequisites Check

Before starting, verify the user has the required tools. Run these silently:

```bash
which gws
gws auth status
```

If `gws` is not installed, tell the user:
> Install the Google Workspace CLI: `npm install -g @googleworkspace/cli`
> Then authenticate: `gws auth login -s gmail`
> Then run this skill again.

If `gws` is installed but the auth check fails for any reason, show the user the actual error output and tell them:
> Run `gws auth login -s gmail` to connect your Gmail account, then try again.

---

## Step 1 — Define the Canvas

Ask the user:

> **What do you want to track from your emails?**
> Examples: "subscription trials and renewals", "package deliveries", "invoices and receipts", "job applications", "event invitations"

Based on their answer, generate three things and confirm with the user before proceeding:

### 1a. Canvas Name
A short slug, e.g. `subscription-trials`, `package-tracking`, `invoices`

Sanitize the slug: lowercase, replace spaces and special characters with hyphens, strip anything that is not `a-z`, `0-9`, or `-`. Example: "Q1 invoices / March" → `q1-invoices-march`.

### 1b. Gmail Search Query
A Gmail search string that will find the most relevant emails. Be specific to avoid noise.
Examples:
- Subscriptions: `subject:(trial OR subscription OR renewal OR "free trial") category:updates`
- Packages: `subject:(shipped OR delivered OR tracking OR "out for delivery")`
- Invoices: `subject:(invoice OR receipt OR "order confirmation" OR payment) has:attachment`
- Job apps: `subject:(application OR interview OR "thank you for applying" OR offer)`

### 1c. Extraction Schema
A JSON object defining what to extract from each email. Keep fields concise and typed.
Example for subscriptions:
```json
{
  "company": "string — name of the company or service",
  "plan": "string — plan or tier name if mentioned",
  "trial_end_date": "date (YYYY-MM-DD) — when trial ends, null if not found",
  "amount": "string — price or amount, include currency symbol",
  "status": "enum: trial | active | cancelled | expired | unknown",
  "action_required": "boolean — does this email require the user to take action",
  "notes": "string — any other relevant detail in one sentence, null if nothing notable"
}
```

Show the canvas name, query, and schema to the user and ask: "Does this look right? I'll search your Gmail and extract data based on this."

---

## Step 2 — Search Gmail

Once confirmed, search Gmail for matching emails:

```bash
gws gmail +triage --query "<GENERATED_QUERY>" --format json --max 500
```

This returns a JSON object with a `messages` array, where each item has `id`, `from`, `subject`, and `date`.

- If 0 results: tell the user and suggest a broader query. Offer to try again with a revised query.
- If the result contains 500 items (the requested max), warn the user that there may be more emails not included and offer to re-run with `--max 1000`.
- Parse the JSON and extract the list of `id` values from `result.messages`.

---

## Step 3 — Extract Data (Parallel Subagents)

For each email ID, spawn parallel subagents to extract structured data. Process in batches of 10 to avoid rate limits.

After each batch completes, report progress to the user: "Processing batch 3 of 20…"

Each subagent receives this prompt:

---
*Subagent prompt:*

You are extracting structured data from a single email for a Capsulate canvas.

**Canvas:** `<CANVAS_NAME>`
**Schema:** `<SCHEMA_JSON>`

**Your task:**
1. Fetch the email: `gws gmail +read --id <EMAIL_ID> --format json --headers`
2. Read the full email body (decode if needed)
3. Extract ONLY the fields defined in the schema
4. Return a single JSON object matching the schema exactly
5. Use `null` for fields not found in the email
6. Do not invent or guess data — only extract what is explicitly stated

Return ONLY valid JSON. No explanation, no markdown, just the JSON object.
Also include these fields in your response:
- `_id`: the email ID (`<EMAIL_ID>`)
- `_subject`: the email subject
- `_from`: the sender email address
- `_date`: the email date in ISO 8601 format

---

Collect all subagent responses. Discard any that fail or return invalid JSON, but keep a count. After all batches complete, tell the user: "Extracted data from X of Y emails (Z failed)." If more than 20% failed, warn the user and suggest re-running.

---

## Step 4 — Persist Locally

Ensure the capsulate data directory exists:
```bash
mkdir -p ~/.capsulate
```

Write the canvas data to `~/.capsulate/<CANVAS_NAME>.json`:

```json
{
  "canvas": "<CANVAS_NAME>",
  "description": "<USER'S ORIGINAL DESCRIPTION>",
  "query": "<GMAIL_QUERY>",
  "schema": <SCHEMA_JSON>,
  "last_updated": "<ISO_8601_TIMESTAMP>",
  "total": <NUMBER_OF_ITEMS>,
  "items": [
    ...extracted items...
  ]
}
```

Tell the user: "Saved `<TOTAL>` items to `~/.capsulate/<CANVAS_NAME>.json`"

---

## Step 5 — Generate Dashboard

Generate a self-contained HTML file at `~/.capsulate/<CANVAS_NAME>.html`.

### Design Requirements
- Single HTML file, no external dependencies (all CSS and JS inline)
- Clean, minimal design: white background, subtle shadows, Inter or system-ui font
- Header showing: canvas name, description, item count, last updated time
- Two views: **Card view** (default) and **Table view** — toggle button in header
- Search bar that filters items client-side across all fields
- Each card/row shows all schema fields with smart formatting:
  - Dates formatted as "Mar 28, 2026"
  - Booleans as colored badges ("Yes" green / "No" gray)
  - Null values shown as "—" in muted gray
  - Status enums as colored badges (customize colors per canvas type)
- Sort by any column in table view (click header)
- Footer: "Generated by Capsulate · <TIMESTAMP>"
- The JSON data is embedded directly in the HTML as a JS variable — no file reading needed

### Data Embedding
Embed the extracted data in a JSON script tag and parse it at runtime to avoid XSS if email content contains `</script>`:
```html
<script type="application/json" id="canvas-data">
<JSON_DATA>
</script>
<script>
const CANVAS_DATA = JSON.parse(document.getElementById('canvas-data').textContent);
</script>
```

After writing the file, open it:
```bash
open ~/.capsulate/<CANVAS_NAME>.html
```

Tell the user:
> "Dashboard generated at `~/.capsulate/<CANVAS_NAME>.html`"
> "Run `/capsulate` again anytime to refresh with new emails."

---

## Re-running an Existing Canvas

If the user runs `/capsulate` and a canvas already exists at `~/.capsulate/`, ask:

> You have existing canvases:
> - `subscription-trials` (47 items, last updated Mar 28)
> - `package-tracking` (12 items, last updated Mar 25)
>
> **Refresh an existing canvas** or **Create a new one**?

If refreshing: skip Step 1, use the saved query and schema, re-run Steps 2–5, and merge new items with existing ones (deduplicate by `_id`, overwriting the existing record if the same `_id` appears — this reflects updated email state).

---

## Error Handling

- **gws rate limit**: pause 2 seconds between batches of email fetches
- **Email body empty or unreadable**: include the item with all schema fields as `null` and `_error: "Could not read email body"`
- **gws auth expired**: tell the user to run `gws auth login -s gmail` and try again
- **Disk write error**: tell the user to check permissions on `~/.capsulate/`
