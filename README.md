# AI Support Ticket Triage

An AI workflow that reads incoming support tickets, classifies them, sets a priority, finds the right help article, drafts a reply for a human to approve, and logs the result.

Built with **n8n** and the **Claude API**. Demo project using synthetic data for a fictional SaaS company ("Cloudnest").

> Status: in progress

## The problem

In a busy support team, engineers spend a large share of their time reading, sorting and routing tickets before any real work starts. Urgent issues (outages, security) can sit in the same queue as "how do I export to Excel?", and first replies are often slow and inconsistent.

## The solution

A working proof of concept: a support portal (built with Lovable) connected to an AI triage workflow (n8n + Claude).

```
Customer submits ticket (portal)
  -> n8n webhook
  -> Claude: category, priority, KB articles, draft reply, escalation, confidence
  -> result saved to the database -> appears live in the agent console
  -> P1 / security tickets alert the team
Support engineer reviews -> corrects category, priority or reply if needed -> approves & sends
  -> every correction is recorded -> Insights page shows how often the AI was right
```

- **Classification** into 11 categories and priorities P1 to P4
- **Knowledge-base lookup**: the reply is based only on approved help articles
- **Human in the loop**: AI drafts, a person approves before anything is sent
- **Escalation routing** to Engineering, Security, Billing or Account

## Results

_To be filled in after testing on the 30 sample tickets in `data/tickets.csv`._

| Metric | Result |
|---|---|
| Category accuracy vs. expected labels | ?% |
| Priority accuracy | ?% |
| Average time per ticket (AI) | ? s |
| Estimated manual triage time saved | ? min/ticket |

## Responsible AI

- No reply is sent without human approval
- Replies may only use the knowledge base, not invented facts
- Low-confidence results (below 0.7) are flagged for manual review
- Only synthetic data is used; in production, personal data would need a data processing agreement and GDPR-compliant handling
- Under the EU AI Act, customers should be told when a reply was drafted with AI

## Repository

| Folder | Content |
|---|---|
| `data/` | 30 synthetic tickets with expected labels |
| `knowledge-base/` | 22 fictional help articles |
| `prompts/` | The triage prompt and the Lovable build prompt |
| `supabase/` | Seed data for the knowledge base and sample tickets |
| `workflow/` | Exported n8n workflow (JSON) |
| `docs/` | n8n build guide, screenshots and demo |

## About me

Sajesh Sankaranarayanan: 8 years at Databricks in technical support engineering and support operations management, now working on AI consulting and integration.
