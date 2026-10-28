# Inquiry Triage Bot

An n8n workflow that reads incoming course inquiries (Bangla, English or Banglish), scores them with an LLM, drafts a Bangla reply, and routes each lead to the right place automatically.

## What it does

- Accepts inquiries through a secured webhook (`POST`)
- Rejects requests with a missing phone or message, and logs them
- Detects duplicate submissions from the same person
- Classifies each message with an LLM into strict JSON:
  `intent`, `lead_score` (1-10), `language`, `summary`, `suggested_reply` (Bangla, max 80 words), `needs_human`
- Retries once on invalid JSON, then falls back to a safe human-review route
- Routes the result:
  | Condition | Action |
  |---|---|
  | Score >= 7 | Slack alert in `#sales-hot-leads` + row in the Google Sheet `Hot` tab |
  | Score 4-6 | Logged to Sheet + Gmail **draft** (never auto-sent) |
  | Complaint or needs human | Slack `#support-escalation` |
  | Spam | Logged only |
- Logs every run: timestamp, input, model output, route, token usage and estimated cost
- A separate error workflow sends a Slack alert whenever any node fails

## Design decisions

- **Small, fast model.** Classifying short messages into a fixed schema does not need a large model. A small model keeps latency and cost low.
- **Temperature 0.** Same input should give the same score and route.
- **max tokens 600.** Enough for an 80-word Bangla reply plus JSON, while capping cost.
- **Thresholds in one place.** Hot (7) and warm (4) thresholds, model name, temperature, max tokens and prices live in the `Config` node.
- **PII minimisation.** Only the course interest and the message are sent to the model. The phone number is never sent. Phone numbers and emails typed inside the message are masked, and logs store the phone masked.
- **Prompt-injection defence.** The message is treated as untrusted data in the prompt, and a pattern check forces suspicious messages to human review.
- **Resilience.** Retry on fail with a wait between attempts (handles rate limits), a 30-second timeout, and a fallback route if the model is down.
- **Model-agnostic.** Only the `Call` node and the response parser depend on the provider, so the model can be swapped without changing the rest of the flow.

## Setup

1. Import `workflow.json` into n8n (and the error workflow JSON).
2. Create these credentials in n8n:
   - Header Auth for the webhook (`X-Webhook-Token`)
   - Header Auth for the LLM API key
   - Slack, Google Sheets, Gmail
3. Create a Google Sheet with two tabs:
   - `Logs`: `timestamp, execution_id, mode, input, claude_output, route, intent, lead_score, needs_human, input_tokens, output_tokens, est_cost_usd, note`
   - `Hot`: `timestamp, name, phone, email, course_interest, intent, lead_score, language, summary, suggested_reply`
4. Create the Slack channels `#sales-hot-leads` and `#support-escalation`.
5. Select the credentials and the Sheet in each node, set the error workflow in the workflow settings, then **Publish**.

## Usage

```bash
curl -X POST "https://<your-n8n-host>/webhook/course-inquiry" \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Token: <your-token>" \
  -d '{"name":"Rahim","phone":"01712345678","email":"rahim@example.com","course_interest":"AI Automation","message":"bhai course er fee koto, installment ase?"}'
