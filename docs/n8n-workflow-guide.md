# n8n workflow: setup guide

The portal's backend function sends the ticket plus the knowledge base to this workflow. n8n asks Claude to triage the ticket, **returns the result in the webhook response**, and alerts the team for urgent tickets. n8n never touches the database.

```
Webhook -> Build Claude request -> Claude API -> Parse & validate -> Respond to Webhook -> Urgent? -> Alert team
```

## Quick start: import

1. In n8n: **Create Workflow** → **⋯** → **Import from File** → `workflow/triage-workflow.json`.
2. **Webhook** node → Credential for Header Auth → create one with name `x-webhook-secret` and a long random value (your shared secret).
3. **Claude API** node → Credential Type **Anthropic** → select or create your Anthropic credential (API key from console.anthropic.com; API usage needs prepaid credit in the Anthropic Console, separate from a Claude subscription).
4. **Save**, then **Publish/activate** the workflow.
5. Copy the **Production URL** of the Webhook node (`.../webhook/triage-ticket`) and give it, plus the shared secret, to the portal as `N8N_TRIAGE_WEBHOOK_URL` and `N8N_WEBHOOK_SECRET`.

Credentials are never included in the exported file, so the JSON is safe to commit.

## The nodes

### 1. Webhook
- HTTP Method: **POST**, Path: `triage-ticket`
- Authentication: **Header Auth** (`x-webhook-secret`)
- Respond: **Using 'Respond to Webhook' Node**

Expected request body:

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

### 2. Build Claude request (Code, Run Once for All Items)

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
7. Keep "reasoning" to at most 2 short sentences.
Respond with JSON only:
{"category":"","priority":"","kb_articles":[""],"escalate_to":"","confidence":0.0,"reasoning":"","draft_reply":""}`;

const user = `Knowledge base:\n${kb}\n\nTicket:\n- Customer: ${t.customer_name} (${t.plan} plan)\n- Subject: ${t.subject}\n- Message: ${t.body}`;

return [{ json: {
  ticket_id: t.ticket_id,
  body: {
    model: 'claude-haiku-5-5',
    max_tokens: 2500,
    system,
    messages: [{ role: 'user', content: user }]
  }
}}];
```

### 3. Claude API (HTTP Request)
- Method: POST, URL: `https://api.anthropic.com/v1/messages`
- Authentication: **Predefined Credential Type** → **Anthropic**
- Send Headers: `anthropic-version` = `2023-06-01`
- Send Body: JSON → Using JSON → `{{ JSON.stringify($json.body) }}`
- Settings: Retry On Fail on (2 tries)

### 4. Parse & validate (Code, Run Once for All Items)

```javascript
const CATS = ['Access & Login','Billing','Bug','Performance','Integration','How-to',
  'Feature request','Account & Compliance','Security','Outage','Feedback'];
const PRIOS = ['P1','P2','P3','P4'];
const ESC = ['None','Engineering','Security','Billing','Account'];

// Read Claude's answer, whatever shape the previous node returns it in
const res = $input.first().json;
const fromBlocks = blocks => blocks.filter(b => b && b.type === 'text').map(b => b.text).join('');
let text = '';
if (Array.isArray(res.content)) text = fromBlocks(res.content);
else if (typeof res.content === 'string') text = res.content;
else if (typeof res.text === 'string') text = res.text;
else if (typeof res.data === 'string') {
  try { text = fromBlocks(JSON.parse(res.data).content || []); } catch (e) { text = res.data; }
}

if (res.stop_reason === 'max_tokens') {
  throw new Error('Claude answer was cut off: increase max_tokens in "Build Claude request".');
}
const match = text.match(/\{[\s\S]*\}/);
if (!match) {
  throw new Error('No JSON found in Claude answer. Fields received: ' + Object.keys(res).join(', ') + ' | text: ' + text.slice(0, 200));
}
let r;
try { r = JSON.parse(match[0]); } catch (e) {
  throw new Error('Claude returned broken JSON: ' + text.slice(0, 300));
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
- Respond With: **First Incoming Item**
- Options: Response Code `200`; Response Header `Content-Type: application/json`

### 6. Urgent? (IF), after Respond to Webhook
`ai_priority` equals `P1` **or** `ai_escalate_to` equals `Security`. The workflow keeps running after responding, so the alert doesn't slow down the portal.

### 7. Alert team
A placeholder in the import. Replace it with a Gmail or Slack node, e.g.:
`🚨 {{ $json.ai_priority }} ticket {{ $json.ticket_id }} needs attention. Escalate to: {{ $json.ai_escalate_to }}`

## Testing

- **Without the portal:** see [`test-request.md`](test-request.md).
- **Test URL vs Production URL:** the Test URL (`/webhook-test/`) only works right after you click **Execute workflow** on the canvas, for one request. "Listen for test event" inside the Webhook node runs only that node. The Production URL (`/webhook/`) works once the workflow is published.
- **After every change:** save and publish again. The Production URL always runs the last published version.

## Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| Portal: "n8n response was not valid JSON" or 403 | Call rejected before the workflow started: wrong secret, test URL not listening, or workflow not published | Check the secret on both sides, use the Production URL, publish |
| Portal: "n8n responded with 500" | The workflow failed inside n8n | Open **Executions**, click the failed run, then the red node |
| Claude API: "credit balance is too low" | No prepaid API credit | Add credit in the Anthropic Console (Billing) |
| Parse & validate: "answer was cut off" | Claude ran out of tokens | Raise `max_tokens` in Build Claude request |
| Parse & validate: "No JSON found" / "broken JSON" | Claude added text that breaks the JSON | The error shows the text; tighten the prompt |
| Portal: "Invalid category" | Webhook responds immediately, or Respond to Webhook sends the wrong item | Webhook: "Using 'Respond to Webhook' Node"; Respond With: First Incoming Item |

## Before the n8n Cloud trial ends
Export the workflow again if you changed it (⋯ → Download). The hosted portal needs a public webhook URL, so a local n8n only works with a tunnel (for example ngrok or Cloudflare Tunnel).
