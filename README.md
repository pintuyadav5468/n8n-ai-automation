# AI-Powered Sheet Automation with n8n + Claude + Slack

An end-to-end automation workflow that watches a Google Sheet for new rows, uses Anthropic Claude to summarize and categorize the text, writes the results back to the sheet, and posts an alert to Slack — all running locally for free using Docker.

---

## How It Works

```
Google Sheet (new row)
        │
        ▼  polling every 60s
┌───────────────────┐
│ Google Sheets     │  Detects new rows via polling.
│ Trigger Node      │  Outputs the row data (including "Input" column).
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ HTTP Request Node │  POST to https://api.anthropic.com/v1/messages
│ (Claude API)      │  Model: claude-sonnet-4-6
│                   │  Sends the "Input" text with a structured prompt.
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Code Node         │  Parses Claude's JSON response.
│ (Parse Response)  │  Extracts: { summary, category, row_number }
└────────┬──────────┘
         │
    ┌────┴────┐  (runs in parallel)
    │         │
    ▼         ▼
┌───────┐ ┌─────────┐
│Google │ │  Slack  │
│Sheets │ │  Node   │
│Update │ │         │
└───────┘ └─────────┘
Writes      Posts:
Summary +   "New [Category]: [Summary]"
Category    to #automations channel
back to
same row
```

---

## The Claude Prompt

The exact prompt sent to Claude on every row. It instructs the model to return **strict JSON only** — no markdown, no explanation — so the Code node can parse it reliably with `JSON.parse()`.

```
You are a text classification and summarization assistant. Analyze the following
text and return ONLY a valid JSON object — no markdown, no explanation, no extra text.

Text to analyze:
"""
[INPUT TEXT FROM THE SHEET]
"""

Return ONLY this JSON:
{
  "summary": "one sentence capturing the main point (max 20 words)",
  "category": "Support OR Sales OR Feedback OR Spam"
}

Rules:
- summary: Exactly one concise sentence, under 20 words.
- category: Must be exactly one of: Support, Sales, Feedback, Spam.
- Output ONLY the raw JSON object. No markdown code fences. No extra words.
```

### Why this prompt structure works
- **Explicit format constraint** ("ONLY a valid JSON object") prevents Claude from adding conversational wrappers.
- **Repeating the rule** at the bottom ("No markdown code fences") addresses the most common failure mode.
- **Limited vocabulary for category** ("Must be exactly one of") eliminates ambiguous outputs like "customer support" or "spam-like".
- **The Code node has a regex fallback** — even if Claude occasionally wraps output in ` ```json ` fences, the node strips them and still parses correctly.

---

## Sample Google Sheet Structure

| Input | Summary | Category |
|---|---|---|
| My subscription is not working and I cannot log in to my account. I've tried resetting my password three times but keep getting errors. | *(auto-filled)* | *(auto-filled)* |
| I'm interested in upgrading my team to the Pro plan. Can you send pricing and annual discount info? | *(auto-filled)* | *(auto-filled)* |
| Just wanted to say your new dashboard redesign is amazing! Dark mode is a game changer. Keep it up! | *(auto-filled)* | *(auto-filled)* |
| 🔥 Buy 10000 Instagram followers for $5! Click NOW before offer expires! | *(auto-filled)* | *(auto-filled)* |

Expected Claude output for row 4:
```json
{ "summary": "Spam message offering to sell fake social media followers.", "category": "Spam" }
```

---

## Project Structure

```
n8n-ai-automation/
├── docker-compose.yml              # Runs n8n locally on port 5678
├── README.md                       # This file
├── workflow/
│   └── ai-automation-workflow.json # Importable n8n workflow (5 nodes)
├── docs/
│   └── setup-guide.md              # Step-by-step credential setup
└── sample/
    └── google-sheet-sample.csv     # Sample sheet with 4 example rows
```

---

## Tech Stack

| Tool | Role | Cost |
|---|---|---|
| **n8n** (self-hosted via Docker) | Workflow automation engine | Free |
| **Google Sheets API** | Trigger source + output destination | Free |
| **Anthropic Claude** (`claude-sonnet-4-6`) | AI summarization + categorization | Pay-per-token (very cheap) |
| **Slack API** | Notification delivery | Free |
| **Docker** | Local container runtime | Free |

---

## Quick Start

```bash
# 1. Start n8n
docker compose up -d

# 2. Open http://localhost:5678 and create your account

# 3. Follow docs/setup-guide.md to connect credentials

# 4. Import workflow/ai-automation-workflow.json

# 5. Update the Google Sheet ID in both Google Sheets nodes

# 6. Toggle the workflow Active and add a row to your sheet
```

Full instructions: [docs/setup-guide.md](docs/setup-guide.md)

---

## Portfolio Notes

This project demonstrates:
- **API integration** — calling a third-party LLM API (Anthropic) via HTTP with header-based auth inside a workflow node
- **Prompt engineering** — structuring a prompt to return machine-parseable output reliably
- **Event-driven architecture** — polling trigger that reacts to external data changes without a server
- **Parallel execution** — the parsed result fans out to two independent actions (sheet update + Slack) simultaneously
- **Error resilience** — the Code node includes a regex fallback for malformed LLM output
- **Infrastructure as code** — the entire environment is defined in `docker-compose.yml` and reproducible in one command
