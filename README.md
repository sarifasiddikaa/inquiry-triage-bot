# Inquiry Triage Bot

An n8n workflow that reads incoming course inquiries in Bangla, English or Banglish, scores each lead with an LLM, drafts a Bangla reply, and routes it to the right place automatically.

Hot leads should never go cold because someone had to read every message by hand.

## How it works

```
Webhook (POST, token-protected)
   -> Normalize input -> Config
   -> Validate (phone + message required)  -> reject + log
   -> Duplicate check                       -> ignore + log
   -> LLM classification (strict JSON)
   -> Validate JSON -> retry once -> fallback route
   -> Route by score / intent
   -> Log every run
```

## Features

- **Secured webhook** with header-token authentication
- **Validation:** missing phone or message is rejected and logged
- **Duplicate detection** for repeated submissions from the same person
- **Strict JSON output** from the model:
  `intent`, `lead_score` (1-10), `language`, `summary`, `suggested_reply` (Bangla, max 80 words), `needs_human`
- **Retry once** on invalid JSON, then a safe **fallback route** for human review
- **Routing**

  | Condition | Action |
  |---|---|
  | Score >= 7 | Slack alert in `#sales-hot-leads` + row in the Google Sheet `Hot` tab |
  | Score 4-6 | Logged + Gmail **draft** (never auto-sent) |
  | Complaint or needs human | Slack `#support-escalation` |
  | Spam | Logged only |

- **Full logging:** timestamp, input, model output, route, token usage and estimated cost for every run
- **Error workflow:** a separate workflow posts to Slack whenever any node fails

## Design decisions

- **Small, fast model.** Short-text classification into a fixed schema does not need a large model. A small model keeps latency and cost low.
- **Temperature 0.** The same input gives the same score and route.
- **Max tokens 600.** Enough for an 80-word Bangla reply plus JSON, while capping cost per call.
- **One Config node.** Hot threshold (7), warm threshold (4), model name, temperature, max tokens, duplicate window and token prices all live in a single node. Change them without touching the rest of the flow.
- **PII minimisation.** The phone number is never sent to the model. Phone numbers and emails typed inside the message are masked before sending, and logs store the phone number masked. The full number is only written to the `Hot` tab for the sales team.
- **Prompt-injection defence.** The message is passed as untrusted data inside tags, the system prompt forbids following instructions inside it, and a pattern check forces suspicious messages to human review.
- **Resilience.** Retry on fail with a wait between attempts (handles rate limits), a request timeout, and a fallback route if the model is unavailable.
- **Provider-agnostic.** Only the LLM call nodes and the response parser depend on the provider. Swapping the model provider does not change the rest of the flow.

## Setup

1. Import `workflow.json` into n8n, and import the error workflow too.
2. Create these credentials in n8n (no keys are stored in the workflow):
   - **Header Auth** for the webhook: name `X-Webhook-Token`, value = your secret token
   - **Header Auth** for the LLM API key (`Authorization: Bearer <key>`)
   - **Slack**, **Google Sheets**, **Gmail**
3. Create a Google Sheet with two tabs:
   - `Logs`: `timestamp, execution_id, mode, input, claude_output, route, intent, lead_score, needs_human, input_tokens, output_tokens, est_cost_usd, note`
   - `Hot`: `timestamp, name, phone, email, course_interest, intent, lead_score, language, summary, suggested_reply`
4. Create the Slack channels `#sales-hot-leads` and `#support-escalation`.
5. In each node, select the credential, and in each Sheets node select your sheet.
6. In the workflow settings, set the error workflow.
7. Click **Publish**.

## Usage

```bash
curl -X POST "https://<your-n8n-host>/webhook/course-inquiry" \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Token: <your-token>" \
  -d '{"name":"Rahim","phone":"01712345678","email":"rahim@example.com","course_interest":"AI Automation","message":"bhai course er fee koto, installment ase?"}'
```

Example response:

```json
{"status":"processed","route":"hot","intent":"price","lead_score":8}
```

Use the **Production URL** (`/webhook/`). The Test URL (`/webhook-test/`) only works while the editor is listening.

## Configuration

Edit the `Config` node:

| Field | Default | Purpose |
|---|---|---|
| `hot_threshold` | 7 | Score for the hot-lead route |
| `warm_threshold` | 4 | Score for the warm-lead route |
| `model` | your model | Model name sent to the API |
| `temperature` | 0 | Deterministic output |
| `max_tokens` | 600 | Output cap |
| `max_message_chars` | 2000 | Longer messages are truncated |
| `duplicate_window_minutes` | 10 | Duplicate detection window |
| `price_input_per_mtok`, `price_output_per_mtok` | set to your model | Cost estimate in the log |

## Tech

n8n, Groq API, Slack, Google Sheets, Gmail
