# Capsulate Skill for Claude Code

Turn your Gmail inbox into structured, queryable dashboards — entirely on your machine. No accounts, no backend, no cloud.

Describe what you want to track. Claude searches your email, extracts the data, and generates a dashboard you can open in your browser.

## What You Can Build

- **Subscription tracker** — every trial, renewal, and cancellation in one view
- **Package tracker** — all your shipments with status and expected delivery
- **Invoice log** — receipts and orders with amounts and dates
- **Job application tracker** — every application, interview, and offer
- **Anything else** — describe it in plain English and Claude figures out the rest

## How It Works

1. You describe what you want to track
2. Claude generates a Gmail search query and extraction schema
3. Your emails are fetched locally via the `gws` CLI
4. Claude extracts structured data from each email in parallel
5. Data is saved to `~/.capsulate/<canvas>.json`
6. A self-contained HTML dashboard opens in your browser

Everything runs on your machine. Your email data never leaves your computer.

## Setup

### 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Install the Google Workspace CLI

```bash
npm install -g @googleworkspace/cli
```

### 3. Connect your Gmail

You'll need a Google Cloud project with the Gmail API enabled. This is a one-time setup.

**Quick setup** (requires `gcloud` CLI):
```bash
gws auth setup
gws auth login -s gmail
```

**Manual setup** (without `gcloud`):
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project
3. Enable the Gmail API
4. Create OAuth credentials → Desktop app type
5. Add your Gmail address as a Test user
6. Download `client_secret.json` to `~/.config/gws/`
7. Run `gws auth login -s gmail`

### 4. Install the Capsulate skill

```bash
git clone https://github.com/imekenni/capsulate-skill ~/.claude/skills/capsulate
```

## Usage

```
/capsulate
```

Claude will ask what you want to track, search your Gmail, extract the data, and open a dashboard in your browser.

### Re-run to refresh

Running `/capsulate` again lets you refresh an existing canvas with new emails, or create a new one.

### Your data

All data is saved locally:
```
~/.capsulate/
  subscription-trials.json    ← extracted data
  subscription-trials.html    ← dashboard (open in browser)
  package-tracking.json
  package-tracking.html
```

## Requirements

- Node.js 18+
- Claude Code CLI
- Google Workspace CLI (`gws`)
- A Gmail account

## Privacy

The `gws` CLI fetches emails directly from Gmail to your local machine. Email content is processed by Claude via the Anthropic API to extract structured data — this means email snippets are included in prompts sent to Anthropic's API. Anthropic's [privacy policy](https://www.anthropic.com/legal/privacy) governs how that data is handled. Extracted data is saved locally to `~/.capsulate/` and is not sent anywhere else.

## License

MIT
