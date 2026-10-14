# Lovable follow-up prompt: connect to n8n through a backend function

Paste this into the Lovable chat.

---

Change how AI triage is triggered. The browser should no longer call n8n directly; use a backend (edge) function instead.

1. Add two secrets: `N8N_TRIAGE_WEBHOOK_URL` and `N8N_WEBHOOK_SECRET`. Ask me for their values.

2. Add a nullable text column `triage_error` to `tickets`.

3. Create an edge function `triage-ticket` that:
   - receives `{ "ticket_id": "<uuid>" }`
   - sets the ticket's status to 'triaging' and clears triage_error
   - loads the ticket and all rows from `kb_articles`
   - sends a POST to `N8N_TRIAGE_WEBHOOK_URL` with header `x-webhook-secret: <N8N_WEBHOOK_SECRET>` and JSON body:
     `{ ticket_id, customer_name, plan, subject, body, kb_articles: [{ id, title, body }] }`
   - waits up to 60 seconds for the JSON response, which contains:
     `ai_category, ai_priority, ai_escalate_to, ai_kb_articles (array), ai_confidence (number), ai_reasoning, ai_draft_reply`
   - validates the values against the allowed categories, priorities and escalation teams
   - saves them on the ticket with status 'awaiting_review' and triaged_at = now()
   - on any error or timeout: sets status 'failed' and stores a short error message in triage_error

4. In the frontend, after a ticket is created (form or "Simulate incoming ticket"), invoke the `triage-ticket` edge function instead of calling the webhook. Don't wait for it in the UI; the console updates through Realtime.

5. On failed tickets in the agent console, show the triage_error text and a "Retry AI triage" button that invokes the edge function again.

6. Make sure "Simulate incoming ticket" picks from the `sample_tickets` table (not hard-coded examples), and remove any demo tickets you invented.
