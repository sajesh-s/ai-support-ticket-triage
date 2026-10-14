# Lovable prompt: Cloudnest Support Portal (POC)

Paste everything below the line into Lovable as your first message. Enable Lovable Cloud (Supabase) when it asks.

---

Build a support portal proof of concept for "Cloudnest", a fictional B2B SaaS company. It demonstrates AI-assisted ticket triage with a human in the loop. AI triage happens in an external n8n workflow; this app stores tickets, triggers the workflow, shows results live, and lets a support engineer review, correct and send the reply.

## Backend (Lovable Cloud / Supabase)

Create these tables. No authentication for this POC, so allow public read/write via RLS policies (demo only).

**kb_articles**
- id (text, primary key, e.g. "KB-01")
- title (text)
- body (text)

**sample_tickets** (used by the "Simulate incoming ticket" button)
- id (text, primary key, e.g. "T-001")
- customer, plan, subject, body (text)
- expected_category, expected_priority (text)

**tickets**
- id (uuid, primary key, default gen_random_uuid())
- created_at (timestamptz, default now())
- customer_name, customer_email, plan, subject, body (text)
- status (text, default 'new'): one of new, triaging, awaiting_review, sent, failed
- ai_category, ai_priority, ai_escalate_to (text)
- ai_kb_articles (text[])
- ai_confidence (numeric)
- ai_reasoning, ai_draft_reply (text)
- triaged_at (timestamptz)
- final_category, final_priority, final_escalate_to, final_reply (text)
- category_corrected, priority_corrected, escalation_corrected, reply_edited (boolean)
- reviewed_by (text)
- sent_at (timestamptz)

Enable Realtime on the tickets table.

Allowed values:
- Categories (11): Access & Login, Billing, Bug, Performance, Integration, How-to, Feature request, Account & Compliance, Security, Outage, Feedback
- Priorities: P1, P2, P3, P4
- Escalation: None, Engineering, Security, Billing, Account

## Trigger the AI workflow

When a ticket is created: insert it into `tickets` with status 'new', then invoke a backend (edge) function `triage-ticket` with `{ "ticket_id": "<uuid>" }`. That function sends the ticket and the knowledge base to the n8n webhook, waits for the AI result and saves it on the ticket (status 'awaiting_review'). See `lovable-followup-prompt.md` for the full specification; on error it sets status 'failed'.

## Pages

### 1. Customer portal (/)
- Simple form: name, email, plan (Starter / Business / Enterprise), subject, message.
- On submit: create the ticket as above and show a confirmation with the ticket number.
- A secondary button "Simulate incoming ticket" picks a random row from `sample_tickets` and submits it as a ticket (email: support-demo@cloudnest.example). This is for live demos.

### 2. Agent console (/console)
Layout: ticket queue on the left, ticket detail on the right.

**Queue**
- Live list of tickets via Supabase Realtime, newest first. New tickets slide in with a subtle highlight.
- Each row: priority badge (P1 red, P2 orange, P3 blue, P4 grey), subject, customer, category, status, age (e.g. "2 min ago").
- While status is 'triaging', show an "AI triaging..." shimmer instead of category and priority.
- Filters: status, priority. Default sort: P1 first among awaiting_review, then newest.

**Ticket detail**
- Original ticket: customer, plan, subject, message, received time.
- AI suggestion panel (clearly labelled "AI suggestion"):
  - Category, priority, escalation, confidence (as a percentage bar).
  - If confidence < 0.7, show a yellow "Low confidence – review carefully" banner.
  - Reasoning (collapsible).
  - Knowledge-base articles used: show id and title; clicking opens the article body in a side drawer.
- Review panel (editable, pre-filled with the AI values):
  - Category dropdown, priority dropdown, escalation dropdown.
  - Reply textarea pre-filled with the AI draft.
  - Next to each field that the engineer changes, show a small "Changed from AI: <original>" hint.
  - Reviewer name input (remember it in localStorage).
  - Button "Approve & send": saves final_* fields, sets the four *_corrected / reply_edited booleans by comparing final values to the AI values, sets reviewed_by, sent_at = now(), status = 'sent'. Show a toast "Reply sent to <email>".
  - If escalation is not None, the toast also says "Escalated to <team>".
- Sent tickets are read-only and show AI vs final values side by side.

### 3. Insights (/insights)
Metrics from tickets with status 'sent':
- Tickets handled
- Category accepted as-is (% where category_corrected = false)
- Priority accepted as-is (%)
- Reply sent without edits (%)
- Average AI confidence
- Median time from created_at to sent_at
- Bar chart: tickets per category
- Table of corrections: ticket, field, AI value, final value

## Design
- Clean, professional SaaS look; light theme; neutral greys with one accent colour (teal).
- Top nav: Customer portal · Agent console · Insights.
- Footer note on every page: "Demo with synthetic data. AI drafts are always reviewed by a human before sending."
