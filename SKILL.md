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
4. Subagents extract structured data from batches of emails in parallel
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

## Re-running an Existing Canvas

If the user runs `/capsulate` and canvases already exist at `~/.capsulate/`, list them and ask:

> You have existing canvases:
> - `subscription-trials` (47 items, last updated Mar 28)
> - `package-tracking` (12 items, last updated Mar 25)
>
> What would you like to do?
> 1. **Refresh** an existing canvas (fetch only new emails)
> 2. **Open** an existing canvas dashboard
> 3. **Update schema** for an existing canvas (re-extract all items with new fields)
> 4. **Create** a new canvas

### Option: Open
Run `open ~/.capsulate/<CANVAS_NAME>.html` and exit.

### Option: Refresh
Skip Step 1. Load the saved canvas JSON to get the existing query, schema, known IDs, and `last_updated` timestamp. Proceed to Step 2 in **incremental mode**.

### Option: Update schema
Ask the user what fields to add, remove, or change. Show the current schema and proposed new schema for confirmation. Then re-run Steps 2–5 against **all** emails (not incremental), replacing all existing items. Merge by `_id`: overwrite existing records, keep any that weren't in the new result set.

### Option: Create
Proceed from Step 1 as normal.

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

Once confirmed, search Gmail for matching emails.

**Incremental mode** (refreshing an existing canvas): append a date filter to the query to only fetch emails newer than the canvas's `last_updated` timestamp. Format: `after:YYYY/MM/DD`.

```bash
gws gmail +triage --query "<GENERATED_QUERY> after:YYYY/MM/DD" --format json --max 500
```

**Full mode** (new canvas or schema update):
```bash
gws gmail +triage --query "<GENERATED_QUERY>" --format json --max 500
```

This returns a JSON object with a `messages` array, where each item has `id`, `from`, `subject`, and `date`.

- If 0 results (incremental): tell the user "No new emails since last refresh." Re-open the dashboard and exit.
- If 0 results (full): tell the user and suggest a broader query. Offer to try again with a revised query.
- If the result contains 500 items (the requested max), warn the user that there may be more emails not included and offer to re-run with `--max 1000`.
- Parse the JSON and extract the list of `id` values from `result.messages`.

**Incremental deduplication:** If refreshing, load the existing canvas JSON and collect all known `_id` values. Remove any IDs from the new fetch result that are already in the saved canvas — do this before spawning any subagents. Tell the user: "Found X new emails to process (Y already extracted, skipping)."

---

## Step 3 — Extract Data (Parallel Subagents)

Group the email IDs into batches of **5 emails per subagent**, running **4 subagents concurrently** (20 emails per round). Pause 2 seconds between rounds to respect rate limits.

After each round completes, report progress to the user: "Processing emails 21–40 of 200…"

Each subagent receives this prompt:

---
*Subagent prompt:*

You are extracting structured data from emails for a Capsulate canvas.

**Canvas:** `<CANVAS_NAME>`
**Schema:** `<SCHEMA_JSON>`

**Your task:** For each email ID listed below, do the following:
1. Fetch the email: `gws gmail +read --id <EMAIL_ID> --format json --headers`
2. Read the full email body (decode if needed)
3. Extract ONLY the fields defined in the schema
4. Use `null` for fields not found in the email — do not invent or guess data
5. Include these metadata fields in each result:
   - `_id`: the email ID
   - `_subject`: the email subject
   - `_from`: the sender email address
   - `_date`: the email date in ISO 8601 format

**Email IDs to process:**
`<EMAIL_ID_1>`, `<EMAIL_ID_2>`, `<EMAIL_ID_3>`, `<EMAIL_ID_4>`, `<EMAIL_ID_5>`

Return ONLY a valid JSON array of objects, one per email, in the same order as the IDs above. No explanation, no markdown — just the JSON array. If an email cannot be read, include the item with all schema fields as `null` and add `"_error": "Could not read email body"`.

---

Collect all subagent responses. Each response should be a JSON array — flatten all arrays into a single list. Discard any items that fail JSON parsing, but keep a count. After all rounds complete, tell the user: "Extracted data from X of Y emails (Z failed)." If more than 20% failed, warn the user and suggest re-running.

**On refresh:** merge extracted items with existing canvas items. Deduplicate by `_id`, with new records overwriting old ones (reflects updated email state).

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
  "schema": "<SCHEMA_JSON>",
  "last_updated": "<ISO_8601_TIMESTAMP>",
  "total": "<NUMBER_OF_ITEMS>",
  "items": [
    "...extracted items..."
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
- **Summary stats bar** below the header: compute and display 2–4 useful aggregate stats from the data. Examples:
  - Subscriptions: "3 active trials · $47.99/mo total · 2 expiring within 7 days"
  - Packages: "5 in transit · 3 delivered this week"
  - Invoices: "12 invoices · $1,240.00 total"
  - For any canvas: count non-null items per enum value if an enum field exists
- Two views: **Card view** (default) and **Table view** — toggle button in header
- **Search bar** that filters items client-side across all fields
- **Date range filter**: if any schema field contains "date", show From/To date pickers that filter items client-side
- **Group by** dropdown: if any field is an enum, allow grouping cards/rows by that field
- Each card/row shows all schema fields with smart formatting:
  - Dates formatted as "Mar 28, 2026"
  - Booleans as colored badges ("Yes" green / "No" gray)
  - Null values shown as "—" in muted gray
  - Status enums as colored badges (customize colors per canvas type)
- Sort by any column in table view (click header to cycle asc/desc)
- **CSV export button** in the header: clicking it downloads all currently-filtered items as a `.csv` file named `<canvas-name>-<date>.csv`
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

## Error Handling

- **gws rate limit**: the 2-second pause between rounds (Step 3) should prevent this. If a rate limit error still occurs, pause 10 seconds and retry the current batch once before discarding.
- **Email body empty or unreadable**: include the item with all schema fields as `null` and `_error: "Could not read email body"`
- **gws auth expired**: tell the user to run `gws auth login -s gmail` and try again
- **Disk write error**: tell the user to check permissions on `~/.capsulate/`
