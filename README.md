# n8n WhatsApp AI Agent Automation

A WhatsApp customer-support bot built with n8n. It receives customer messages through the WhatsApp Business Cloud API, answers them with an AI agent (Google Gemini) that reads course and FAQ data from a Google Sheet, and replies on WhatsApp.

## How it works

```
Customer (WhatsApp) -> Meta Cloud API -> ngrok -> n8n
  WhatsApp Trigger -> AI Agent (Gemini + Memory + Google Sheets tools) -> WhatsApp Send message
```
<img width="1600" height="435" alt="WhatsApp Image 2026-09-19 at 5 06 23 PM" src="https://github.com/user-attachments/assets/79184de7-5ea8-4a86-ad87-8201877474f4" />

<img width="1600" height="686" alt="WhatsApp Image 2026-09-19 at 3 31 08 PM" src="https://github.com/user-attachments/assets/68a85ea1-01bc-4ae0-ba4a-37151c0b6e63" />

<img width="1600" height="533" alt="WhatsApp Image 2026-09-19 at 4 37 13 PM" src="https://github.com/user-attachments/assets/98f58a05-3668-41d2-b6b1-76e7b9fd8540" />

<img width="1600" height="435" alt="WhatsApp Image 2026-09-19 at 5 06 23 PM" src="https://github.com/user-attachments/assets/a2ff41ce-1893-464a-9333-a9e651676a5e" />

<img width="1919" height="826" alt="Screenshot 2026-09-20 230728" src="https://github.com/user-attachments/assets/106c94bd-b4ad-4af3-9c6a-5df7ae2b67be" />

<img width="1291" height="475" alt="Screenshot 2026-09-20 232357" src="https://github.com/user-attachments/assets/0d042408-74c7-425b-b36f-1f2870e0a7c7" />

<img width="1910" height="794" alt="Screenshot 2026-09-20 230852" src="https://github.com/user-attachments/assets/c5ef0bae-340f-4139-bd88-bb3aa692be99" />

<img width="1276" height="476" alt="Screenshot 2026-09-20 232406" src="https://github.com/user-attachments/assets/8a668b80-f263-46b2-bfb7-5900e897fbd3" />




- **WhatsApp Trigger** receives incoming messages.
- **AI Agent** answers only from the sheet data (course price, mode, enroll link, refund, installment, payment, etc.).
- **Simple Memory** keeps a separate conversation per customer, keyed by their phone number.
- **Send message** replies to the same customer.

## Tech stack

- n8n (self-hosted, Docker)
- WhatsApp Business Platform (Cloud API)
- Google Gemini (chat model)
- Google Sheets (bot knowledge base)
- ngrok (public HTTPS URL for webhooks during development)

## Repository contents

| File | Purpose |
|---|---|
| `docker-compose.yml` | Runs n8n in Docker |
| `.env.example` | Environment variable template |
| `.gitignore` | Keeps `n8n_data/` and `.env` out of Git |

> `n8n_data/` (database and credential encryption key) and `.env` are intentionally not committed.

## Prerequisites

- Docker and Docker Compose
- A Meta developer account and a Business app with the WhatsApp product
- A Google account (Sheets and Gemini API key)
- An ngrok account (free plan is enough for testing)

## Setup

### 1. Clone and configure

```bash
git clone https://github.com/Raihanroo/n8n-WhatsApp-ai-agennt-automation-.git
cd n8n-WhatsApp-ai-agennt-automation-
cp .env.example .env
```

Edit `.env` and set `WEBHOOK_URL` to your public ngrok URL (with a trailing `/`).

### 2. Start ngrok

```bash
ngrok http 5678
```

Keep this terminal open. Copy the `https://...ngrok-free.dev` URL into `.env`.

### 3. Start n8n

```bash
docker compose up -d
```

Open `http://localhost:5678`. Use `localhost` for the editor.

### 4. Google Sheet

Create a sheet with two tabs, spelled exactly:

- `Courses` - course name, price, mode, seat status, enroll link
- `FAQ` - question and answer rows (refund, installment, payment, batch transfer, internship, contact)

Keep lead or personal customer data in a separate sheet and do not expose it to the bot.

### 5. Meta / WhatsApp

1. Create a Meta app and add **WhatsApp**.
2. In **Step 1. Try it out**, use the free test number and add your own number as a verified recipient.
3. Generate an access token (temporary tokens expire in 24 hours).
4. In the n8n WhatsApp **Trigger** credential, use the app's **App ID** and **App Secret**.
5. In the **Send message** credential, use the access token and the WhatsApp Business Account ID.
6. Subscribe your app to the WhatsApp Business Account:

```bash
curl -X POST "https://graph.facebook.com/<API_VERSION>/<WABA_ID>/subscribed_apps" \
  -H "Authorization: Bearer <ACCESS_TOKEN>"
```

### 6. Workflow

Nodes and key expressions:

- AI Agent prompt: `{{ $json.messages[0].text.body }}`
- Memory session key: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
- Send message recipient: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
- Send message text: `{{ $json.output }}`
- Sender: your WhatsApp **Phone Number ID** (not the phone number itself)

To run without pressing Execute, **Publish** the workflow and set the Meta webhook Callback URL to the trigger's **Production URL** (`/webhook/...`, not `/webhook-test/...`).

## Troubleshooting

| Problem | Cause / fix |
|---|---|
| `Invalid parameter` on Send message | Recipient number must be digits only with country code (`8801...`), no `+`, spaces, or leading `0`. Also check Text Body is not empty. |
| `Invalid parameter` on WhatsApp Trigger | n8n has no public HTTPS URL. Set `WEBHOOK_URL` to the ngrok URL and recreate the container. |
| `Callback verification failed ... 404` | ngrok tunnel is offline (`ERR_NGROK_3200`). Restart `ngrok http 5678`. |
| `Authorization failed` / 401 | Access token expired or the wrong credential is selected on the node. Generate a new token. |
| Message reaches Meta but nothing arrives in n8n | Your app is not subscribed to the WhatsApp Business Account. Run the `subscribed_apps` request above. |
| Trigger shows old data | Old pinned or previous execution data. Unpin and clear the execution, then listen again. |
| Text message not delivered | The 24-hour window is closed. The user must message the number first, or use a template message. |
| Gemini `429` quota error | Free-tier daily limit reached. Enable billing or use another model. |
| Sender dropdown is empty | Token rejected. Enter the Phone Number ID manually. |

## Security notes

- Never commit access tokens, App Secret, ngrok authtoken, or `n8n_data/`.
- Meta's test access token is temporary. For production, create a permanent token with a **System User**.
- The test number only works with verified recipient numbers. Serving real customers requires registering your own business number.
- Keep customer lead data out of the bot's data sources.

## Roadmap

- [ ] Courses and FAQ tools as separate Google Sheets nodes
- [ ] Permanent token via System User
- [ ] Production business number
- [ ] Hosting on n8n Cloud or a VPS
- [ ] Export sanitized workflow JSON to `workflows/`

## License

Add a license of your choice (for example MIT).
