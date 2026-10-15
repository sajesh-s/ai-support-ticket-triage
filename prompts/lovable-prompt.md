# Lovable prompt: Cloudnest Support Portal

The complete specification of the portal, including the evaluation and correction-feedback features. Paste everything below the line into a new Lovable project and enable Lovable Cloud when asked. Then run `supabase/seed.sql` (paste it into the chat and ask Lovable to run it as a migration).

---

Build a support portal proof of concept for "Cloudnest", a fictional B2B SaaS company. It demonstrates AI-assisted ticket triage with a human in the loop. AI triage happens in an external n8n workflow; this app stores tickets, calls the workflow through a backend function, shows results live, and lets a support engineer review, correct and send the reply.

## Backend (Lovable Cloud)

No authentication for this POC; allow public read/write via RLS policies (demo only).

**kb_articles**: id (text, primary key, e.g. "KB-01"), title (text), body (text)

**sample_tickets**: id (text, primary key, e.g. "T-001"), customer, plan, subject, body, expected_category, expected_priority (all text)

**tickets**
- id (uuid, primary key, default gen_random_uuid()), created_at (timestamptz, default now())
- customer_name, customer_email, plan, subject, body (text)
- sample_ticket_id (text, nullable: set when created from sample_tickets)
- status (text, default 'new'): new, triaging, awaiting_review, sent, failed
- triage_error (text)
- ai_category, ai_priority, ai_escalate_to (text), ai_kb_articles (text[]), ai_confidence (numeric), ai_reasoning, ai_draft_reply (text), triaged_at (timestamptz)
- final_category, final_priority, final_escalate_to, final_reply (text)
- category_corrected, priority_corrected, escalation_corrected, reply_edited (boolean)
- correction_reasons (text[]), correction_comment (text)
- reviewed_by (text), sent_at (timestamptz)

Enable Realtime on tickets.

Allowed values:
- Categories (11): Access & Login, Billing, Bug, Performance, Integration, How-to, Feature request, Account & Compliance, Security, Outage, Feedback
- Priorities: P1, P2, P3, P4
- Escalation: None, Engineering, Security, Billing, Account

## AI triage (backend function)

Secrets: `N8N_TRIAGE_WEBHOOK_URL` and `N8N_WEBHOOK_SECRET` (ask me for the values).

Edge function `triage-ticket`:
- receives `{ "ticket_id": "<uuid>" }`
- sets status 'triaging' and clears triage_error
- loads the ticket and all kb_articles
- POSTs to `N8N_TRIAGE_WEBHOOK_URL` with header `x-webhook-secret: <N8N_WEBHOOK_SECRET>` and JSON body `{ ticket_id, customer_name, plan, subject, body, kb_articles: [{ id, title, body }] }`
- waits up to 60 seconds; reads the response with `response.text()` and then JSON.parse; if the result is an array, uses its first element
- expects `ai_category, ai_priority, ai_escalate_to, ai_kb_articles, ai_confidence, ai_reasoning, ai_draft_reply`; validates them against the allowed values
- saves them with status 'awaiting_review' and triaged_at = now()
- on any error, timeout, non-2xx status or invalid response: status 'failed', and triage_error = HTTP status plus the first 300 characters of the raw response

The frontend invokes this function after creating a ticket and does not wait for it; the console updates through Realtime.

## Pages

### 1. Customer portal (/)
- Form: name, email, plan (Starter / Business / Enterprise), subject, message. On submit: create the ticket, invoke triage, show a confirmation with the ticket number.
- Button "Simulate incoming ticket": picks a random row from sample_tickets, creates a ticket from it (email support-demo@cloudnest.example, sample_ticket_id set) and invokes triage.

### 2. Agent console (/console)
Queue on the left, ticket detail on the right.

**Queue**: live via Realtime, newest first; priority badge (P1 red, P2 orange, P3 blue, P4 grey), subject, customer, category, status, age. "AI triaging..." shimmer while status is 'triaging'. Filters: status, priority. P1 first among awaiting_review.

**Ticket detail**
- Original ticket: customer, plan, subject, message, received time.
- AI suggestion panel: category, priority, escalation, confidence bar; yellow "Low confidence – review carefully" banner below 0.7; collapsible reasoning; knowledge-base articles (id and title; click opens the article in a side drawer).
- Failed tickets: show triage_error and a "Retry AI triage" button.
- Review panel, pre-filled with the AI values: category, priority and escalation dropdowns; reply textarea; "Changed from AI: <original>" hint next to each changed field; reviewer name (remembered in localStorage).
- If anything was changed: a "What was wrong?" section with multi-select chips (Wrong category, Wrong priority, Wrong escalation, Inaccurate information, Missing information, Tone or format, Too long, Wrong KB article, Other) and an optional comment. At least one chip is required before sending. Hidden when nothing was changed.
- "Approve & send": saves final_* values, the *_corrected / reply_edited flags (comparing final to AI values), correction reasons and comment, reviewed_by, sent_at = now(), status 'sent'. Toast "Reply sent to <email>", plus "Escalated to <team>" if escalation is not None.
- Sent tickets are read-only, showing AI vs final values side by side and a word-level diff of AI draft vs final reply.

### 3. Insights (/insights)
From tickets with status 'sent':
- Tickets handled; category accepted as-is (%); priority accepted as-is (%); reply sent without edits (%); average AI confidence; median time from created_at to sent_at
- Bar chart: tickets per category
- Bar chart: correction reasons (count per reason)
- Table of corrected tickets: subject, reasons, comment, link
- **Evaluation**: for tickets created from sample_tickets, compare ai_category and ai_priority with the expected values; show accuracy percentages and a table of mismatches
- "Export corrections (CSV)": subject, AI values, final values, reasons, comment

## Design
Clean, professional SaaS look; light theme; neutral greys with a teal accent. Top nav: Customer portal · Agent console · Insights. Footer on every page: "Demo with synthetic data. AI drafts are always reviewed by a human before sending."
