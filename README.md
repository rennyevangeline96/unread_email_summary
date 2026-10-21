# Unread Email Summariser (n8n AI Agent)

An n8n workflow that reads my latest 5 unread Gmail emails, asks an AI model to summarise each one in a single line, and emails all five summaries back to me as one styled HTML digest.

---

## Problem

Unread mail piles up faster than I can open it. This workflow gives me a quick digest of what is waiting, with one line per email.

## How it works

```
Manual Trigger -> Get 5 Unread Emails -> Unread Email Summarize (+ OpenAI Chat Model) -> combining summaries -> Emailing "Renny"
```

| # | Node name (as on canvas) | Node type | Purpose |
|---|--------------------------|-----------|---------|
| 1 | **Manual Trigger** | Manual Trigger | Runs the workflow on demand |
| 2 | **Get 5 Unread Emails** | Gmail (`getAll: message`) | Fetches the latest 5 unread messages |
| 3 | **Unread Email Summarize** | AI Agent | Writes a one-line summary (max 20 words) per email |
| 4 | **OpenAI Chat Model** | OpenAI Chat Model (`gpt-4o-mini`) | Language model connected to the agent's Chat Model input |
| 5 | **combining summaries** | Aggregate | Merges 5 separate outputs into 1 item |
| 6 | **Emailing "Renny"** | Gmail (`send: message`) | Sends one HTML email with all summaries |

### Item counts on the canvas

| Connection | Items |
|---|---|
| Manual Trigger to Get 5 Unread Emails | 1 |
| Get 5 Unread Emails to Unread Email Summarize | 5 |
| Unread Email Summarize to combining summaries | 5 |
| combining summaries to Emailing "Renny" | 1 |

The model node shows "5 items total" because the agent calls it once per email.

### Key idea

n8n runs a node once per item. Gmail returns 5 items, so the AI Agent runs 5 times and produces 5 outputs. The **combining summaries** node is where those 5 become 1, so exactly one email is sent.

The AI only receives the **sender, subject and snippet** of each email, not the full body. This is cheaper, faster and enough for a one-line summary.

## Tech stack

- [n8n](https://n8n.io) (Cloud or self-hosted)
- Gmail API with Google Cloud OAuth
- OpenAI `gpt-4o-mini` (any chat model works)

## Node settings

### Get 5 Unread Emails (Gmail)

| Setting | Value |
|---|---|
| Resource | Message |
| Operation | Get Many |
| Return All | Off |
| Limit | 5 |
| Simplify | On |
| Filters, Read Status | Unread |

### Unread Email Summarize (AI Agent)

- Source for Prompt: **Define below**
- Chat Model: **OpenAI Chat Model**
- Prompt (Expression mode):

```
Write ONE line (20 words maximum) describing this email. No greeting, no heading, no extra text.

From: {{ $json.From }}
Subject: {{ $json.Subject }}
Preview: {{ $json.snippet }}
```

> Field names are case-sensitive. Check the output of **Get 5 Unread Emails** and use exactly what it shows (`From` / `Subject` or `from` / `subject`).

### combining summaries (Aggregate)

- Aggregate: Individual Fields
- Input Field Name: `output`

### Emailing "Renny" (Gmail)

| Setting | Value |
|---|---|
| Resource | Message |
| Operation | Send |
| To | your own email address |
| Subject | Your 5 unread emails |
| Email Type | HTML |
| Message | HTML template built from `$json.output` (see below) |

Simple plain-text alternative (set Email Type to Text):

```
{{ $json.output.join('\n\n') }}
```

HTML digest message (set Email Type to HTML):

```
{{ '<div style="font-family:Segoe UI,Arial,sans-serif;max-width:620px;margin:auto;border:1px solid #e0e0e0;border-radius:10px;overflow:hidden">'
+ '<div style="background:#0f766e;color:#ffffff;padding:16px 20px"><h2 style="margin:0;font-size:20px">📬 Inbox Digest</h2><p style="margin:4px 0 0;font-size:13px">' + $now.toFormat('dd LLL yyyy') + ' · ' + $json.output.length + ' unread emails summarised</p></div>'
+ '<div style="padding:12px 20px">'
+ $json.output.map((s, i) => '<div style="padding:12px 0;border-bottom:1px solid #eeeeee;line-height:1.5;font-size:15px;color:#222"><span style="display:inline-block;background:#0f766e;color:#ffffff;border-radius:50%;width:24px;height:24px;line-height:24px;text-align:center;font-size:13px;margin-right:10px">' + (i + 1) + '</span>' + String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/\n/g,' ') + '</div>').join('')
+ '</div>'
+ '<div style="background:#f5f5f5;color:#888888;font-size:12px;padding:10px 20px">Sent by your n8n Unread Email Summariser</div></div>' }}
```

## Setup

### 1. Run n8n

- n8n Cloud: sign up for the free trial at n8n.io, or
- Local: `npx n8n`, then open `http://localhost:5678`

### 2. Import the workflow

1. Create a new blank workflow.
2. Paste the workflow JSON onto the canvas (Ctrl+V), or use **Import from file**.
3. Name it **Unread Email Summariser**.

### 3. Connect Gmail (Google Cloud OAuth)

1. In Google Cloud Console, create a project and enable the **Gmail API**.
2. Configure the **OAuth consent screen**: Audience **External**, and add your Gmail address as a **Test user**.
3. Create an **OAuth client ID** of type **Web application**.
4. In n8n, open a Gmail node, choose **Create new credential**, and copy the **OAuth Redirect URL**.
5. Paste it into **Authorized redirect URIs** in Google Cloud.
6. Copy the **Client ID** and **Client Secret** into n8n and click **Sign in with Google**.
7. Select the same credential on both Gmail nodes (**Get 5 Unread Emails** and **Emailing "Renny"**).

### 4. Connect the AI model

Open the **OpenAI Chat Model** node, create a credential and paste your API key. The key is stored as an n8n credential, never in the workflow itself.

### 5. Update imported values

- Change the **To** address in **Emailing "Renny"** to your own email.
- Reconnect all credentials (imported credential IDs belong to the original author's account).
- Confirm the field names in the **Unread Email Summarize** prompt match your Gmail output.

### 6. Run

Click **Execute workflow**, then check your inbox for **Your 5 unread emails**. Save with Ctrl/Cmd+S.

## Example output

```
Subject: Your 5 unread emails

1. Claim your free DSA with Python course today using code PYTHON100 from Adda247 and PrepInsta.
2. Job alert for an Administrative Assistant position in Chennai, offering a monthly salary range.
3. Join a live session with a former Microsoft engineer on building AI agents without coding.
4. LinkedIn recommends posts on trending AI, Deep Learning and Machine Learning topics.
5. Notification of a bank debit from account XX6919.
```

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Expressions are red, summaries blank or "undefined" | Field name case mismatch. Use `From` / `Subject` exactly as in the Gmail output |
| Google "Access blocked" or 403 | Your Gmail is not added as a Test user, or the Gmail API is not enabled |
| Redirect URI mismatch | The URL in Google Cloud must exactly match the one shown in n8n |
| More than one email sent | **combining summaries** is missing or placed after the send node |
| Raw `<div>` tags in the email | Set Email Type to **HTML** |
| All summaries run together as one paragraph | Plain-text message used with Email Type HTML; match the two |
| Summaries longer than one line | Tighten the prompt or add a system message: "Reply with a single sentence only." |

## What I learned

- n8n runs a node once per item, and Aggregate is what combines many items into one.
- Send the AI only what it needs (sender, subject, snippet) to keep it cheap and fast.
- Expressions are case-sensitive; green means the field was found, red means it was not.
- Credentials keep secrets out of the workflow.
- Importing a workflow is a starting point; credentials, recipients and field names must be adapted.

## Notes

- Emails are fetched from all Gmail tabs (Primary, Promotions, Social, Updates), not just Primary.
- Each run picks the 5 newest unread emails, so results change as you read mail.
- Never commit API keys, OAuth secrets or exported credentials to a public repository.

## Repository structure

```
.
├── README.md
├── images/
│   └── workflow-canvas.png
└── Unread Email Summariser.json   (exported workflow, credentials removed)
```