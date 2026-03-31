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

### 2. Install and authenticate the Google Workspace CLI

```bash
# via npm
npm install -g @googleworkspace/cli

# via Homebrew (Mac, no Node.js required)
brew install googleworkspace-cli
```

Follow the official setup guide to connect your Gmail account: [googleworkspace-cli.mintlify.app](https://googleworkspace-cli.mintlify.app)

The short version: create a GCP project, enable the Gmail API, create Desktop OAuth credentials, save `client_secret.json` to `~/.config/gws/`, then run:

```bash
gws auth login -s gmail
```

**Two common setup issues:**

- **Add yourself as a test user.** In the Google Cloud Console, go to OAuth consent screen → Audience and add your Gmail address as a test user. Without this, Google will block the login with an "app not verified" error.

- **Redirect URI mismatch.** `gws auth login` opens a browser and listens on a random local port (e.g., `http://localhost:52638`). If you see a redirect URI mismatch error, go to your OAuth client credentials in the Cloud Console and add `http://localhost` to the list of authorized redirect URIs. This covers all localhost ports.

- **"Google hasn't verified this app" warning.** This is expected for personal OAuth apps in testing mode. Click "Advanced" → "Go to [app name] (unsafe)" to continue. It is safe for personal use.

- **Gmail API not enabled.** If you see a 403 `accessNotConfigured` error on first use, the Gmail API needs to be enabled in your Google Cloud project. Go to APIs & Services → Library, search for "Gmail API", and click Enable. Wait a minute or two for it to propagate before retrying.

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

- Claude Code CLI
- Google Workspace CLI (`gws`)
- A Gmail account

## Privacy

The `gws` CLI fetches emails directly from Gmail to your local machine. Email content is processed by Claude via the Anthropic API to extract structured data — this means email snippets are included in prompts sent to Anthropic's API. Anthropic's [privacy policy](https://www.anthropic.com/legal/privacy) governs how that data is handled. Extracted data is saved locally to `~/.capsulate/` and is not sent anywhere else.

## License

MIT

---

Built by the team behind [Capsulate](https://capsulate.app) — the cloud version with automatic sync, AI analysis, and multi-device access. [Join the waitlist →](https://capsulate.app)
