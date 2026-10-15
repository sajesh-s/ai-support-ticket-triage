# Prompt and workflow changelog

Each change: what I changed, why, and what it did to the results. One change at a time, tested on the same sample tickets.

| Version | Date | Change | Why | Result |
|---|---|---|---|---|
| v1 | 2026-10-14 | First version: 11 categories, P1–P4 definitions, replies only from the knowledge base, JSON output | Baseline | First tickets triaged correctly end to end (e.g. "Slow search": Performance, P3, KB-16, Engineering) |
| v1.1 | 2026-10-14 | Knowledge base replaced with one consistent set of 22 articles | The database held two overlapping sets with contradicting facts (API limit 600 vs 100 requests/min) | Claude no longer gets conflicting sources |
| v1.2 | 2026-10-14 | Parse step reads the answer in every shape and reports clear errors | An unclear "invalid JSON" error made failures hard to debug | Failures now name their cause (cut off, missing JSON, broken JSON) |
| v2 | 2026-10-14 | `max_tokens` 800 → 2500; reasoning limited to 2 short sentences | "Data missing after import" failed because the answer was cut off mid-JSON | Long tickets complete; answers are shorter and faster |
| v3 | _next_ | Reply format rules with an example (greeting, numbered steps, sign-off) | Most frequent correction reason: _fill in from Insights_ | _Replies sent without edits: before x % → after x %_ |

## How I evaluate a change

1. Simulate the same set of sample tickets.
2. Compare on the Insights page: category and priority accuracy against the expected labels, replies sent without edits, and correction reasons.
3. Keep the change only if the numbers improve; otherwise roll it back.
