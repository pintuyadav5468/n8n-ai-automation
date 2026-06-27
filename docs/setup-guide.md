# Step-by-Step Setup Guide

## Prerequisites
- Docker and Docker Compose installed
- A Google account
- An Anthropic account (free tier works)
- A Slack workspace where you are an admin

---

## Step 1 — Start n8n with Docker

```bash
# Clone or enter the project folder
cd n8n-ai-automation

# Start n8n (runs in background)
docker compose up -d

# Check it's running
docker compose logs -f n8n
```

Open your browser: **http://localhost:5678**

On first launch, n8n asks you to create an owner account (email + password). Fill that in — it's stored locally.

> **Tip:** To stop n8n run `docker compose down`. Your data persists in the `n8n_data` Docker volume.

---

## Step 2 — Connect Google Sheets (OAuth2)

### 2a. Create a Google Cloud Project & OAuth credentials

1. Go to https://console.cloud.google.com/
2. Create a new project (e.g., `n8n-automation`).
3. In the left menu: **APIs & Services → Library**
4. Enable **Google Sheets API** and **Google Drive API**.
5. Go to **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
6. Application type: **Web application**
7. Add this Authorized Redirect URI exactly:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
8. Click **Create**. Copy the **Client ID** and **Client Secret**.

### 2b. Add the credential in n8n

1. In n8n: **Settings (gear icon) → Credentials → Add Credential**
2. Search for **Google Sheets OAuth2 API**
3. Paste your Client ID and Client Secret
4. Click **Connect my account** — a browser popup will ask you to sign in with Google and grant access
5. Save. Note the credential name (e.g., "Google Sheets OAuth2").

---

## Step 3 — Get an Anthropic API Key

1. Go to https://console.anthropic.com/
2. Sign up or log in.
3. Navigate to **API Keys → Create Key**
4. Copy the key (starts with `sk-ant-...`).

### Add it in n8n

1. In n8n: **Settings → Credentials → Add Credential**
2. Search for **HTTP Header Auth**
3. Set:
   - **Name:** `x-api-key`
   - **Value:** `sk-ant-YOUR_KEY_HERE`
4. Save with the name "Anthropic API Key".

> The free tier gives you enough credits to test hundreds of runs. Monitor usage at console.anthropic.com.

---

## Step 4 — Connect Slack

### 4a. Create a Slack Bot

1. Go to https://api.slack.com/apps → **Create New App → From scratch**
2. Name it (e.g., `n8n-automations`) and choose your workspace.
3. Go to **OAuth & Permissions → Bot Token Scopes** and add:
   - `chat:write`
   - `chat:write.public` (lets the bot post without being invited)
4. Click **Install to Workspace** → **Allow**
5. Copy the **Bot User OAuth Token** (starts with `xoxb-...`).

### 4b. Add the credential in n8n

1. In n8n: **Settings → Credentials → Add Credential**
2. Search for **Slack API**
3. Paste the Bot Token
4. Save with the name "Slack Bot Token".

### 4c. Create the Slack channel

In Slack, create a channel named `#automations` if it doesn't exist.

---

## Step 5 — Prepare your Google Sheet

1. Create a new Google Sheet.
2. Name the first sheet tab: `Sheet1`
3. Add these exact column headers in row 1:

| A | B | C |
|---|---|---|
| Input | Summary | Category |

4. Copy some sample rows from `sample/google-sheet-sample.csv` into rows 2–5.
5. Copy the **Sheet ID** from the URL:
   ```
   https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit
   ```

---

## Step 6 — Import and Configure the Workflow

1. In n8n: click **+** (New Workflow) → **Import from file**
2. Upload `workflow/ai-automation-workflow.json`
3. Open each node and update:

| Node | What to change |
|---|---|
| **Google Sheets Trigger** | Replace `YOUR_GOOGLE_SHEET_ID_HERE` with your real ID. Select your Google credential. |
| **Claude AI — Analyze Text** | Select your "Anthropic API Key" credential. |
| **Update Google Sheet** | Replace `YOUR_GOOGLE_SHEET_ID_HERE` again. Select your Google credential. |
| **Send Slack Alert** | Select your "Slack Bot Token" credential. |

4. Click **Save** (top right), then toggle **Active** to ON.

---

## Step 7 — Test It

1. Add a new row to your Google Sheet with text in the **Input** column.
2. Wait up to 60 seconds (the trigger polls every minute).
3. Check:
   - The **Summary** and **Category** columns in the same row are filled in.
   - A message appears in Slack `#automations`: `New [Category]: [Summary]`
4. In n8n, click **Executions** to see the run log and debug any issues.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Trigger not firing | Make sure the workflow is **Active** (toggle in top-right). Check Executions log. |
| Google Sheets auth error | Re-connect the OAuth2 credential. Make sure Drive and Sheets APIs are enabled. |
| Anthropic 401 error | Check the credential — Name must be `x-api-key` (lowercase), value must be your full key. |
| Slack message not posting | Ensure the bot token has `chat:write` scope and the channel name in the node matches exactly. |
| Claude returns non-JSON | The Code node has a regex fallback. Check the Executions log to see Claude's raw output. |
