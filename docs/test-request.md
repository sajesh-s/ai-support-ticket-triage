# Test the n8n workflow without the portal

With the Webhook node listening for a test event (or the workflow active), run this in a terminal. Replace the URL and secret.

```bash
curl -X POST "https://YOUR-N8N.app.n8n.cloud/webhook-test/triage-ticket" \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: YOUR_SECRET" \
  -d '{
    "ticket_id": "test-001",
    "customer_name": "Brightwell Retail",
    "plan": "Enterprise",
    "subject": "Cannot log in after password reset",
    "body": "Since this morning none of our 40 users can log in. The reset email arrives but the link says expired.",
    "kb_articles": [
      {"id": "KB-01", "title": "Password reset link expired", "body": "Reset links are valid for 30 minutes and only the most recent link works. If many users at one company are affected at once, escalate to engineering as a possible incident."},
      {"id": "KB-22", "title": "Outages", "body": "For a full outage affecting a customer, escalate immediately as a P1 incident."}
    ]
  }'
```

Expected: a JSON answer with something like `"ai_category": "Access & Login"`, `"ai_priority": "P1"`, `"ai_escalate_to": "Engineering"`.

The test URL contains `/webhook-test/`; the production URL (active workflow) contains `/webhook/`.
