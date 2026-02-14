# CHANGELOG

All notable changes and architecture exceptions are documented here.

## Lessons Learned
Use this section for rule violations and follow-ups.

Entry format:
- `YYYY-MM-DD: [Section X] reason -> outcome -> next action #lessonslearned`

Entries:
- 2026-02-14: [v1.6.1] Added AI Agent Troubleshooting Protocol to AGENTS.md §12.5 with escalation checklist and Minimum Handoff Bundle instructions; added troubleshooting quick-reference table to README.md with common symptoms and fixes; added sandbox setup troubleshooting FAQ to QUICKHELP.md; all based on real-world "Lovable Back-and-Forth Scenarios" section added to SUPABASE-TEMP-DEV.md.
- 2026-02-14: [v1.6] Added Section 12.5 "Working Outside of Lovable AI Chat (Dev Testing Workflow)" to AGENTS.md with AI agent instructions for one-time setup wizard and context-aware suggestions; created comprehensive SUPABASE-TEMP-DEV.md guide based on real-world Lovable implementation with `temp_dev_records` + `temp_dev_reset_log` tables, feature-key scoping pattern, dual access paths (Direct SDK + Edge Function), curl examples, Kanban Positioning Spike workflow, and spike-to-production graduation workflow; updated README.md with practical quick-start examples and use cases; updated QUICKHELP.md (v1.1) with sandbox pattern reference and FAQ entry.
- 2026-02-08: [v1.5] Embedded DASHBOARD.md and QUICKHELP.md templates as appendices inside AGENTS.md with extraction markers and agent instructions (Section 0.0). Single-file delivery: users upload only AGENTS.md; AI agent extracts templates into separate files on first run. Double-trigger pattern: extraction instructions in both AGENTS.md and the user's paste prompt. Updated README.md to reflect single-file upload workflow.
- 2026-02-08: [v1.4] Extracted Compliance Matrix and Architecture Snapshot from AGENTS.md into DASHBOARD.md; created QUICKHELP.md as first-layer help with quick reference and FAQ; updated README.md doc hierarchy to route through QUICKHELP → DASHBOARD → AGENTS; all cross-references use stable intent summaries that won't require edits when target details change.
- 2026-02-08: Initialized violations logging policy and entry format #lessonslearned
