# QUICKHELP.md - First-Layer Help & FAQ
**Version:** 1.1
**Last Updated:** 2026-02-14

Start here before diving into the full architecture guide. Most common tasks and questions are answered below with links to detailed rules when needed.

---

## Quick Reference — "I need to..."

### Add a new page
All pages use a single unified layout system. Follow the build contract for page structure.  
→ [AGENTS.md §2 — Build Contract](./AGENTS.md#2-build-contract-checklist-current-repo-rules)

### Add or edit CMS content
CMS content is managed through a dedicated service layer and hook pattern, not direct database calls.  
→ [AGENTS.md §3 — CMS Content](./AGENTS.md#3-cms-content-checklist-app_config)

### Check build cycle compliance
The compliance dashboard tracks pass/fail status for each architecture section per build cycle.  
→ [DASHBOARD.md — Compliance Matrix](./DASHBOARD.md#compliance-matrix)

### Assess overall project health
The health grade provides a high-level summary of architectural compliance and build quality.
→ [DASHBOARD.md — Overall Health Grade](./DASHBOARD.md#overall-health-grade)

### Fix a broken build
Follow the recovery protocol: isolate the change, fix the smallest root cause, re-verify.  
→ [AGENTS.md §0.4 — Build Break Recovery](./AGENTS.md#04-build-break-recovery-protocol)

### Handle authentication or user state
Auth has a single source of truth via a centralized store. All tenant context flows from there.  
→ [AGENTS.md §4 — State Management](./AGENTS.md#4-state-management-defaults-checklist)

### Add security policies for user data
All user-facing data requires row-level security policies before shipping. Tenant isolation is enforced at the database level.  
→ [AGENTS.md §10 — RLS](./AGENTS.md#10-supabase-rls-requirements-checklist)  
→ [AGENTS.md §10.5 — Multi-Tenant Isolation](./AGENTS.md#105-multi-tenant-isolation-checklist)

### Manage complex UI state
Choose the right state pattern based on complexity — simple booleans, discriminated unions, or full state machines.  
→ [AGENTS.md §6 — FSM Decision Matrix](./AGENTS.md#6-fsm-decision-matrix--trigger-checklist)

### Add error handling or observability
Errors follow a standardized contract across all boundaries. Async operations emit structured telemetry.  
→ [AGENTS.md §7 — Observability](./AGENTS.md#7-observability--error-contract-checklist-tiered)

### Log a rule violation
Intentional rule breaks are logged in the changelog with context and follow-up actions.  
→ [AGENTS.md §13 — Violations Policy](./AGENTS.md#13-violations---changelogmd-policy)

### Understand the current architecture
The architecture snapshot captures the current state of key decisions and is updated each release.
→ [DASHBOARD.md — Architecture Snapshot](./DASHBOARD.md#current-architecture-snapshot)

### Test Supabase features outside Lovable AI chat
Use the reusable sandbox table for prototyping queries, RLS policies, and data structures without deploying edge functions.
→ [AGENTS.md §12.5 — Dev Testing Workflow](./AGENTS.md#125-working-outside-of-lovable-ai-chat-dev-testing-workflow)
→ [SUPABASE-TEMP-DEV.md — Complete Guide](./SUPABASE-TEMP-DEV.md)

---

## Document Map

| Document | Purpose | When to check |
|----------|---------|---------------|
| **[QUICKHELP.md](./QUICKHELP.md)** (this file) | First-layer help, common tasks, FAQ | First stop for any question |
| **[DASHBOARD.md](./DASHBOARD.md)** | Build cycle status and architecture snapshot | During and after each build cycle |
| **[AGENTS.md](./AGENTS.md)** | Full architecture rules and checklists | When you need detailed implementation rules |
| **[SUPABASE-TEMP-DEV.md](./SUPABASE-TEMP-DEV.md)** | Dev testing guide with sandbox table | When prototyping Supabase features |
| **[CHANGELOG.md](./CHANGELOG.md)** | Version history, violations, lessons learned | After changes or when reviewing history |
| **[REFERENCES.md](./REFERENCES.md)** | Source material for design principles | When you want to understand *why* a rule exists |

---

## Scenarios & FAQ

### "The AI agent keeps going in circles"
This usually means the agent can't find the right pattern to apply. Point it to a specific AGENTS.md section:  
`"Follow AGENTS.md §[number] for this task."`  
If the issue persists, check whether the task conflicts with an existing contract.

### "I'm not sure which state pattern to use"
Start with the simplest option that avoids impossible states. The FSM decision matrix provides clear escalation triggers.  
→ [AGENTS.md §6](./AGENTS.md#6-fsm-decision-matrix--trigger-checklist)

### "My RLS policy isn't working"
Verify the policy exists and uses the correct auth context. The multi-tenant checklist includes specific test scenarios for cross-tenant access.  
→ [AGENTS.md §10.5 — Testing & Verification](./AGENTS.md#105-multi-tenant-isolation-checklist)

### "Should I create a new file or edit an existing one?"
Check existing implementations first. The dependency contract requires reusing existing services and helpers before creating new ones.  
→ [AGENTS.md §0.3 — Dependency Contract](./AGENTS.md#03-dependency-and-import-contract)

### "How do I extend the layout or add branding?"
The layout system has designated extension points. Don't add new layout patterns without explicit need.  
→ [AGENTS.md §11 — Theme/Branding](./AGENTS.md#11-themebranding-future-ready-checklist)

### "When do I reset the compliance dashboard?"
At the start of every new build cycle. Reset all checkboxes and update the header dates.
→ [DASHBOARD.md — Orchestration Notes](./DASHBOARD.md#orchestration-notes-for-agents)

### "Where do I log bugs or architecture exceptions?"
In the changelog, using the standard entry format with `#lessonslearned` tag.
→ [AGENTS.md §13](./AGENTS.md#13-violations---changelogmd-policy) | [CHANGELOG.md](./CHANGELOG.md)

### "How do I test Supabase features without deploying edge functions?"
Use the `temp_dev_records` table pattern. Ask your AI assistant to run the one-time setup wizard, then use the sandbox for prototyping queries, RLS policies, and data structures. Reset between test runs with `DELETE ?action=reset_feature&feature_key={key}`.
→ [SUPABASE-TEMP-DEV.md](./SUPABASE-TEMP-DEV.md)

### "The sandbox setup failed with 404/401/RLS errors"
Check the **Lovable Back-and-Forth Scenarios** troubleshooting table in SUPABASE-TEMP-DEV.md. Common fixes:
- **404**: Wait 1-2 minutes for edge function deployment propagation
- **401**: Refresh your access token
- **Table not found**: Apply migration and reload PostgREST schema cache
- **RLS errors**: Verify `user_id` matches `auth.uid()`

When escalating to Lovable, include: curl command, response JSON, project ref, migration filename, and timestamp.
→ [SUPABASE-TEMP-DEV.md — Troubleshooting](./SUPABASE-TEMP-DEV.md#lovable-back-and-forth-scenarios)

