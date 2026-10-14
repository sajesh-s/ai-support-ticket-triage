# n8n workflow: AI ticket triage

The portal sends `{ "ticket_id": "<uuid>" }` to this workflow. The workflow reads the ticket, asks Claude to triage it using only the knowledge base, writes the result back to the database, and alerts the team for urgent tickets. The portal shows the update live.

```
Webhook -> Mark "triaging" -> Get ticket -> Get KB articles -> Build Claude request
       -> Claude API -> Parse & validate -> Save AI result -> IF urgent -> Alert
```

## Before you start

You need, from Lovable Cloud (Supabase):
- **Project URL** (https://xxxx.supabase.co)
- **Service role key** (secret: only ever put it in n8n credentials, never in the app or GitHub)

And a **Claude API key** from console.anthropic.com.

In n8n, create two credentials:
1. **Supabase API**: host = project URL, service role secret = service role key
2. **Header Auth** named "Anthropic": name `x-api-key`, value = your Claude API key

## Nodes

### 1. Webhook
- HTTP Method: POST
- Path: `triage-ticket`
- Respond: **Immediately** (the portal shouldn't wait for the AI)
- Options → Allowed Origins (CORS): `*` (fine for a demo)
- Copy the **Production URL** into `N8N_TRIAGE_WEBHOOK_URL` in Lovable, and activate the workflow.

### 2. Supabase: Update a row ("Mark triaging")
- Table: `tickets`
- Select condition: `id` equals `{{ $json.body.ticket_id }}`
- Field: `status` = `triaging`

### 3. Supabase: Get a row ("Get ticket")
- Table: `tickets`, condition `id` equals `{{ $('Webhook').item.json.body.ticket_id }}`

### 4. Supabase: Get many rows ("Get KB")
- Table: `kb_articles`, Return All: on
- Settings → **Execute Once**: on

### 5. Code ("Build Claude request")
Mode: Run Once for All Items. Paste:

```javascript
const t = $('Get ticket').first().json;
const kb = $('Get KB').all()
  .map(a => `[${a.json.id}] ${a.json.title}\n${a.json.body}`)
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
  ticket_id: t.id,
  body: {
    model: 'claude-haiku-5-5',
    max_tokens: 800,
    system,
    messages: [{ role: 'user', content: user }]
  }
}}];
```

### 6. HTTP Request ("Claude API")
- Method: POST, URL: `https://api.anthropic.com/v1/messages`
- Authentication: Generic → Header Auth → "Anthropic"
- Send Headers: `anthropic-version` = `2023-06-01`
- Send Body: JSON, Specify Body: Using JSON → `{{ JSON.stringify($json.body) }}`
- Settings → Retry On Fail: on (2 tries)

### 7. Code ("Parse & validate")

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

### 8. Supabase: Update a row ("Save AI result")
- Table `tickets`, condition `id` equals `{{ $json.ticket_id }}`
- Fields: all `ai_*` fields from the previous node, plus `status` = `awaiting_review` and `triaged_at` = `{{ $now.toISO() }}`
- For `ai_kb_articles` (an array column) use `{{ $json.ai_kb_articles }}`

### 9. IF ("Urgent?")
- `{{ $('Parse & validate').item.json.ai_priority }}` equals `P1`
- OR `{{ $('Parse & validate').item.json.ai_escalate_to }}` equals `Security`

### 10. Gmail or Slack ("Alert team"), on the true branch
- Message: `🚨 {{ $('Parse & validate').item.json.ai_priority }} ticket: {{ $('Get ticket').item.json.subject }} ({{ $('Get ticket').item.json.customer_name }}). Escalate to: {{ $('Parse & validate').item.json.ai_escalate_to }}`

## Error handling
Add an **Error Trigger** workflow (or connect node error outputs) that sets the ticket's `status` to `failed`, so a ticket never stays stuck in "triaging".

## Testing
1. Activate the workflow.
2. In the portal, click **Simulate incoming ticket** a few times.
3. Watch the agent console: the ticket appears, shows "AI triaging...", then fills in within a few seconds.
4. Good first tests: T-001 (login outage, P1), T-022 (full outage, P1), T-029 (suspicious logins, Security), T-028 (feedback, P4).

## Before the n8n Cloud trial ends
Export the workflow (⋯ → Download) and save it as `workflow/triage-workflow.json` in the repo. Credentials are not included in the export.
The hosted portal needs a public webhook URL, so a local n8n on your own computer only works with a tunnel (for example ngrok or Cloudflare Tunnel).
