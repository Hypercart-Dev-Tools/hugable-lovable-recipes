# CHANGELOG

All notable changes and architecture exceptions are documented here.

## Lessons Learned
Use this section for rule violations and follow-ups.

Entry format:
- `YYYY-MM-DD: [Section X] reason -> outcome -> next action #lessonslearned`

Entries:
- 2026-02-08: [v1.5] Embedded DASHBOARD.md and QUICKHELP.md templates as appendices inside AGENTS.md with extraction markers and agent instructions (Section 0.0). Single-file delivery: users upload only AGENTS.md; AI agent extracts templates into separate files on first run. Double-trigger pattern: extraction instructions in both AGENTS.md and the user's paste prompt. Updated README.md to reflect single-file upload workflow.
- 2026-02-08: [v1.4] Extracted Compliance Matrix and Architecture Snapshot from AGENTS.md into DASHBOARD.md; created QUICKHELP.md as first-layer help with quick reference and FAQ; updated README.md doc hierarchy to route through QUICKHELP → DASHBOARD → AGENTS; all cross-references use stable intent summaries that won't require edits when target details change.
- 2026-02-08: Initialized violations logging policy and entry format #lessonslearned
