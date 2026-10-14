# n8n workflow: AI ticket triage

The portal (a Lovable Cloud backend function) sends the ticket plus the knowledge base to this workflow. n8n asks Claude to triage the ticket, **returns the result in the webhook response**, and alerts the team for urgent tickets. The portal saves the result and shows it live.

n8n never touches the database. That's deliberate: Lovable Cloud doesn't expose database keys to outside tools, and it keeps the design simple and secure.

```
Portal backend function --POST ticket + KB--> Webhook -> Build Claude request -> Claude API
                                                     -> Parse & validate -> Respond to Webhook
                                                     -> IF urgent -> Alert team
Portal backend function <--JSON result-- (saves it, status = awaiting_review)
```

## Before you start

- A **Claude API key** from console.anthropic.com
- A **shared secret**: any long random string (for example from a password manager). The portal sends it in a header so only your portal can call the workflow.

In n8n, create two credentials:
1. **Header Auth** named "Anthropic": name `x-api-key`, value = your Claude API key
2. **Header Auth** named "Portal secret": name `x-webhook-secret`, value = your shared secret

## What the portal sends

```json
{
  "ticket_id": "uuid",
  "customer_name": "Brightwell Retail",
  "plan": "Enterprise",
  "subject": "Cannot log in after password reset",
  "body": "Since this morning none of our 40 users can log in...",
  "kb_articles": [ { "id": "KB-01", "title": "...", "body": "..." } ]
}
```

## Nodes

### 1. Webhook
- HTTP Method: POST
- Path: `triage-ticket`
- Authentication: Header Auth → "Portal secret"
- Respond: **Using 'Respond to Webhook' Node**
- Copy the **Production URL** (not the Test URL) for the portal, and **activate** the workflow.

### 2. Code ("Build Claude request")
Mode: Run Once for All Items. Paste:

```javascript
const t = $('Webhook').first().json.body;
const kb = (t.kb_articles || [])
  .map(a => `[${a.id}] ${a.title}\n${a.body}`)
  .join('\n\n');

const system = `You are a support triage assistant for Cloudnest, a B2B SaaS company.
For each ticket:
1. Category: exactly one of Access & Login, Billing, Bug, Performance, Integration, How-to, Feature request, Account & Compliance, Security, Outage, Feedback.
2. Priority: P1 = outage, security incident, or many users fully blocked. P2 = important function broken or financial/legal impact. P3 = single user affected, or a question needing a specific action. P4 = general question, feature request or feedback. Enterprise customers get one level higher only if the issue blocks their work.
3. Pick the most relevant knowledge-base article ids.
4. Draft a short, friendly reply (max 120 words) based ONLY on the knowledge base. If it doesn't answer the question, say a specialist will follow up. Never invent features or prices.
5. Escalation: Engineering, Security, Billing, Account or None.
6. Confidence from 0 to 1.
Respond with JSON only:
{"category":"","priority":"","kb_articles":[""],"escalate_to":"","confidence":0.0,"reasoning":"","draft_reply":""}`;

const user = `Knowledge base:\n${kb}\n\nTicket:\n- Customer: ${t.customer_name} (${t.plan} plan)\n- Subject: ${t.subject}\n- Message: ${t.body}`;

return [{ json: {
  ticket_id: t.ticket_id,
  body: {
    model: 'claude-haiku-5-5',
    max_tokens: 800,
    system,
    messages: [{ role: 'user', content: user }]
  }
}}];
```

### 3. HTTP Request ("Claude API")
- Method: POST, URL: `https://api.anthropic.com/v1/messages`
- Authentication: Generic Credential Type → Header Auth → "Anthropic"
- Send Headers: `anthropic-version` = `2023-06-01`
- Send Body: JSON, Specify Body: Using JSON → `{{ JSON.stringify($json.body) }}`
- Settings → Retry On Fail: on (2 tries)

### 4. Code ("Parse & validate")

```javascript
const CATS = ['Access & Login','Billing','Bug','Performance','Integration','How-to',
  'Feature request','Account & Compliance','Security','Outage','Feedback'];
const PRIOS = ['P1','P2','P3','P4'];
const ESC = ['None','Engineering','Security','Billing','Account'];

const text = $json.content?.[0]?.text ?? '';
const match = text.match(/\{[\s\S]*\}/);
let r;
try { r = JSON.parse(match[0]); } catch (e) {
  throw new Error('Claude did not return valid JSON: ' + text.slice(0, 200));
}

const valid = CATS.includes(r.category) && PRIOS.includes(r.priority) && ESC.includes(r.escalate_to);

return [{ json: {
  ticket_id: $('Build Claude request').first().json.ticket_id,
  ai_category: CATS.includes(r.category) ? r.category : 'How-to',
  ai_priority: PRIOS.includes(r.priority) ? r.priority : 'P3',
  ai_escalate_to: ESC.includes(r.escalate_to) ? r.escalate_to : 'None',
  ai_kb_articles: Array.isArray(r.kb_articles) ? r.kb_articles : [],
  // an invalid answer is forced below the 0.7 review threshold
  ai_confidence: valid ? Number(r.confidence) || 0 : 0.3,
  ai_reasoning: r.reasoning || '',
  ai_draft_reply: r.draft_reply || '',
}}];
```

### 5. Respond to Webhook
- Respond With: **First Incoming Item** (returns the JSON from step 4)

### 6. IF ("Urgent?"), connected after Respond to Webhook
The workflow keeps running after it has responded, so the alert doesn't slow down the portal.
- `{{ $json.ai_priority }}` equals `P1`
- OR `{{ $json.ai_escalate_to }}` equals `Security`

### 7. Gmail or Slack ("Alert team"), on the true branch
- Message: `🚨 {{ $json.ai_priority }} ticket {{ $json.ticket_id }} needs attention. Escalate to: {{ $json.ai_escalate_to }}`

## Error handling
If Claude fails or returns bad JSON, the workflow errors and the webhook returns an error status. The portal then marks the ticket "failed" and shows a Retry button.

## Testing
1. Test the workflow on its own first: click **Listen for test event** on the Webhook node and send a sample request (see `test-request.md`), or temporarily switch authentication off and use the portal with the Test URL.
2. Activate the workflow, put the Production URL in the portal, and click **Simulate incoming ticket**.
3. Good first tests: T-001 (login outage, P1), T-022 (full outage, P1), T-029 (suspicious logins, Security), T-028 (feedback, P4).

## Before the n8n Cloud trial ends
Export the workflow (⋯ → Download) and save it as `workflow/triage-workflow.json` in the repo. Credentials are not included in the export.
The hosted portal needs a public webhook URL, so a local n8n on your own computer only works with a tunnel (for example ngrok or Cloudflare Tunnel).
