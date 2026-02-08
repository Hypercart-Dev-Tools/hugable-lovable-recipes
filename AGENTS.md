# AGENTS.md - Checklist-Driven Architecture Guide (Lovable TS/JS/React/Vite/Supabase)
**Version:** 1.5
**Last Updated:** 2026-02-08
**Last Audited:** _not yet audited_
**Purpose:** Canonical architecture rules and checklists. For quick help, start with [QUICKHELP.md](./QUICKHELP.md). For build cycle status, see [DASHBOARD.md](./DASHBOARD.md).

---

**License:** CC BY 4.0 (Creative Commons Attribution 4.0 International)
**Copyright:** © 2026 Hypercart DBA Neochrome, Inc.
**Attribution Required:** When sharing or adapting this work, you must:
- Credit "Hypercart DBA Neochrome, Inc." as the original author
- Provide a link to https://creativecommons.org/licenses/by/4.0/
- Indicate if changes were made
- Not remove this attribution notice

**License Terms:** https://creativecommons.org/licenses/by/4.0/legalcode

---

## TL;DR) Read This First (Non-Negotiables)
- Use `UnifiedLayout` + `UnifiedSidebar` for all pages; do not reintroduce deprecated layout patterns.
- Use `useAuthStore()` as auth source of truth; no hardcoded user/org IDs.
- Use `useCmsConfig` + `src/services/cms-config.ts` for CMS; no direct page-level Supabase calls.
- Implement RLS (including tenant isolation rules) before shipping user-facing features.
- Log intentional architectural violations in `CHANGELOG.md` with `#lessonslearned`.

## 0) How to Use This Doc
- Treat each section as a pass/fail checklist.
- Do not mark items done without code evidence (files, tests, policies, or logs).
- Follow order: pre-build → build contract → post-build → continuous loop.
- **First time in this repo?** Extract the DASHBOARD.md and QUICKHELP.md templates (see Section 0.0 below).
- **Need quick help?** Start with [QUICKHELP.md](./QUICKHELP.md).
- **Checking compliance status?** See [DASHBOARD.md](./DASHBOARD.md).
- **New build cycle?** Reset the dashboard checkboxes and update dates in [DASHBOARD.md](./DASHBOARD.md).

## 0.0) File Extraction (One-Time Setup)

> **AI Agent Instruction:** On first use in a new repo, check whether `DASHBOARD.md` and `QUICKHELP.md` exist in the project root. If they do not exist, extract them from the templates in Appendix A and Appendix B below. After extraction:
> 1. Create `DASHBOARD.md` from the Appendix A template.
> 2. Create `QUICKHELP.md` from the Appendix B template.
> 3. Confirm both files exist before proceeding with any build work.
>
> If both files already exist, skip this step.

## 0.1) Compliance Matrix

The compliance matrix tracks pass/fail status for each architecture section across build, QA, and manual testing phases. It is maintained as a living dashboard in a dedicated file.

→ **[DASHBOARD.md — Compliance Matrix](./DASHBOARD.md#compliance-matrix)**

## 0.2) AI Agent Working Guardrails
- Make one change cluster at a time, verify it, then proceed.
- Edit surgically; do not delete or rewrite files wholesale unless explicitly requested.
- Check existing code before creating new files, hooks, or services.
- If a pattern is unclear, ask: "Which AGENTS.md pattern applies here?"
- Prefer smallest working diff that preserves current contracts.

## 0.3) Dependency and Import Contract
- Before creating a new utility/helper, search existing implementations in `src/services/`, `src/lib/`, and `src/utils/`.
- Keep dependency direction one-way: pages/components -> hooks/services -> integrations.
- Avoid circular imports and cross-feature back-references.
- Reuse public module exports before adding parallel helpers.
- Keep one canonical helper per concern to prevent drift.

## 0.4) Build Break Recovery Protocol
- Capture the failing command and first actionable error.
- Isolate the latest change that introduced failure.
- Fix the smallest root cause first (types/imports/contracts before refactors).
- Re-run the failing command, then rerun the full verification set.
- If unresolved, revert only your latest change chunk and retry with a narrower diff.

## 0.5) Current Architecture Snapshot

The architecture snapshot captures the current state of layout, CMS, state management, security, and observability decisions. It is updated after each release.

→ **[DASHBOARD.md — Architecture Snapshot](./DASHBOARD.md#current-architecture-snapshot)**

## 1) Pre-Build Checklist (Before First Feature)
- Define domain boundaries: auth, monitoring/events, admin CMS, shared UI, **tenant isolation**.
- Define tenant model: single-tenant vs. multi-tenant, user-level vs. org-level isolation, shared vs. tenant-scoped tables.
- Define source of truth: Supabase for persisted data, Zustand for shared client state, component state for local ephemeral UI.
- Freeze folder contracts: `src/pages`, `src/components`, `src/stores`, `src/integrations/supabase`.
- Design `app_config` keys and RLS before building CMS-driven pages.
- Design RLS policies for tenant data tables before building user-facing features.
- Define typed contracts first (query result types, union states, service interfaces).

## 2) Build Contract Checklist (Current Repo Rules)
- All pages use `UnifiedLayout` from `src/components/layouts/`.
- `UnifiedSidebar.tsx` remains the single sidebar for all auth states.
- `UnifiedSidebar` nav states remain:
  - logged out: Home, Status, Changelog, Terms (`/tos`), Sign-up, Sign-in
  - logged in user: Dashboard, Monitors, Profile, Status, Changelog, Terms, Sign out
  - logged in admin: Dashboard, Monitors, Profile, Admin, Status, Changelog, Terms, Sign out
- Footer remains embedded in `UnifiedSidebar.tsx`.
- Standalone `<Footer />` is not rendered in pages.
- Do not reintroduce deprecated patterns: `PublicNavbar.tsx`, custom per-page wrappers, standalone page footers.

## 3) CMS Content Checklist (`app_config`)
- CMS values are JSON strings in `app_config` table.
- Keys remain: `tos_html` (Terms content), `footer_html` (Footer content).
- All CMS reads/writes go through `src/services/cms-config.ts` service layer (DRY + SRP).
- Service exposes typed interface: `getConfig(key)`, `setConfig(key, value)` (admin-only).
- Service handles JSON parse/stringify internally; consumers receive typed data.
- Components/pages use `useCmsConfig(key)` hook, never direct Supabase calls (Dependency Inversion).
- Hook returns `{ data, loading, error }` with proper async state modeling (see Section 5).
- RLS enforced: public read, admin-only write (cross-ref Section 10).

## 4) State Management Defaults Checklist
- Auth source of truth is `useAuthStore()` in `src/stores/auth-store.ts`.
- Admin checks use `isAdmin()`.
- Logout uses `signOut()`.
- Tenant context available via `useAuthStore()` (e.g., `currentUserId`, `currentOrgId` if org-level tenancy).
- All Supabase queries automatically filter by tenant context (via RLS policies).
- No hardcoded user/org IDs in components or services.
- Events feed uses `src/stores/events-store.ts` with `MAX_EVENTS = 500`.
- No duplicated writable entity state across component state + store + Supabase.

## 5) Design Rules (Required Paragraphs)
**DRY:** In this stack, each behavior must have one canonical implementation: one layout system, one auth state source (`useAuthStore()`), one CMS storage pattern (`app_config` JSON), and one data-access path per feature; when duplicate logic appears in multiple pages/components, extract only after confirming the abstraction preserves readability and does not hide Supabase query intent.

**S - Single Responsibility:** Keep React components focused on rendering and user interaction, move Supabase I/O into service functions/hooks, and keep stores responsible for state transitions; split files that mix view rendering, remote orchestration, and domain rules.

**O - Open/Closed:** Extend behavior through composition (feature modules, store actions, layout variants, feature flags) instead of rewriting stable flows; new work should plug into existing contracts without breaking current consumers.

**L - Liskov Substitution:** Any replaceable implementation (mock service, alternate adapter, layout variant) must preserve the same input/output and side-effect expectations so call sites never branch on concrete type.

**I - Interface Segregation:** Expose small task-specific interfaces (for example auth session, CMS config client, monitor event reader) so pages/hooks depend only on the methods they use.

**D - Dependency Inversion:** Depend on feature-boundary abstractions (typed services/interfaces) rather than concrete Supabase/browser primitives inside UI components, improving testability and replacement.

**State hygiene:** Keep local UI state local, shared cross-page state in scoped Zustand slices, and persisted truth in Supabase; model async workflows with explicit status unions (`idle | loading | success | error`), reset transient state on route/identity changes, and avoid duplicate writable copies of the same entity.

## 6) FSM Decision Matrix + Trigger Checklist
| Situation | Recommended Pattern |
|---|---|
| 2 or fewer independent toggles, no invalid combinations | `useState` booleans |
| 3+ mutually exclusive modes, invalid combinations possible | discriminated union + reducer |
| multi-step async flow (retry/cancel/timeout/backoff), guards, role-dependent transitions | explicit FSM (state + event + transition map) |

Switch trigger checklist:
- More than one boolean is needed to represent a single UI mode.
- Impossible states can be represented accidentally.
- Transition rules depend on role, async outcome, or retries/timeouts.

Concrete example (boolean drift -> FSM-style union):
```ts
// Before (easy to create impossible states)
const [isLoading, setIsLoading] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [hasError, setHasError] = useState(false);

// After (single source of truth for mode)
type SaveState =
  | { status: 'idle' }
  | { status: 'saving' }
  | { status: 'success' }
  | { status: 'error'; error: AppError };
```

## 7) Observability + Error Contract Checklist (Tiered)

Baseline tier (required for Lovable-hosted and self-hosted):
- Every async boundary (Supabase call, auth transition, edge function call) emits `start/success/failure` logs.
- Logs include: `feature`, `requestId`, `durationMs`, and `errorCode` on failures.
- A shared app error shape is enforced across UI/store/server boundaries.
- Raw errors are normalized at boundaries before reaching UI components.
- User-facing error handling maps to contract categories (`validation`, `auth`, `permission`, `not_found`, `conflict`, `network`, `db`, `unknown`).

Advanced tier (required when self-hosting or using dedicated telemetry tooling):
- Forward boundary events to a centralized sink (log aggregation/APM).
- Propagate correlation IDs across frontend, edge functions, and Supabase operations.
- Alert on error-rate and latency thresholds by feature/domain.
- Track release-over-release trend lines for error categories and latency.

Recommended contract:
```ts
type AppError = {
  code: string;
  message: string;
  category: 'validation' | 'auth' | 'permission' | 'not_found' | 'conflict' | 'network' | 'db' | 'unknown';
  retryable: boolean;
  httpStatus?: number;
  requestId?: string;
  details?: Record<string, unknown>;
};

type Result<T> = { ok: true; data: T } | { ok: false; error: AppError };
```

## 8) Post First Build Checklist
- Critical journeys pass in local + preview (auth, dashboard, monitor lifecycle, admin CMS edit/publish).
- Verification commands pass: `bun run typecheck`, `bun run lint`, `bun run test`, `supabase db diff`.
- Security checks pass: RLS on writable tables, admin-only writes verified.
- Multi-tenant isolation verified (cross-ref Section 10.5):
  - User A cannot read/write User B's data (monitors, events, profiles, etc.)
  - Unauthenticated users cannot access any tenant data
  - Cross-tenant queries are blocked by RLS policies
  - Admin access scoped correctly (global config vs. tenant data)
- Resilience checks pass: loading/error/empty states exist for each remote surface.
- `Observability + Error Contract Checklist` (Section 7) baseline tier is passed; advanced tier is passed when self-hosting/telemetry tooling exists.
- `CHANGELOG.md` is updated with version bump and architecture deltas.
- Metrics captured: RLS policy coverage %, avg page load time, error rate by category.

## 9) Continuous Audit -> Fix -> Iterate Checklist
- Audit runtime signals: errors, slow queries, flaky journeys, UX dead-ends, schema/policy drift.
- Prioritize by user impact, security risk, and recurrence.
- Fix in small diffs (one pattern/file cluster at a time).
- Re-verify with typecheck/lint/tests/db diff and re-test touched journeys.
- Record violations/lessons in `CHANGELOG.md` with `#lessonslearned`.
- Repeat at least once per release cycle.

## 10) Supabase RLS Requirements Checklist
- `app_config` public read is enabled.
- `app_config` writes are restricted to authenticated admins (`is_admin(auth.uid())`).
- RLS policies are verified before adding new `app_config` keys.

## 10.5) Multi-Tenant Isolation Checklist

### Tenant Model Definition:
- Tenant scope defined: user-level (each user owns their data), org-level (users share org data), or hybrid.
- Tenant context stored in: `auth.uid()` for user-level, `auth.jwt() ->> 'org_id'` for org-level, or separate tenant table.
- Shared vs. tenant-scoped tables documented (e.g., `app_config` is shared, `monitors`/`events` are tenant-scoped).

### RLS Policies (Tenant Data):
- All tenant data tables have RLS enabled (monitors, events, profiles, user-specific settings, etc.).
- Policies use `auth.uid()` or `auth.jwt() ->> 'org_id'` for isolation.
- SELECT policies prevent cross-tenant reads: `WHERE user_id = auth.uid()` or equivalent.
- INSERT/UPDATE/DELETE policies prevent cross-tenant writes.
- No table allows cross-tenant SELECT/INSERT/UPDATE/DELETE without explicit admin override.
- Admin override policies (if needed) are explicitly documented and audited.

### Application Layer:
- Tenant context available in `useAuthStore()` (e.g., `currentUserId`, `currentOrgId`).
- All Supabase queries automatically filter by tenant (via RLS, not manual WHERE clauses).
- No hardcoded user/org IDs in components, services, or hooks.
- Service layer functions accept tenant context from auth store, never from props/params.

### Testing & Verification:
- Cross-tenant read test: Create User A and User B, verify User A cannot SELECT User B's monitors/events.
- Cross-tenant write test: Verify User A cannot INSERT/UPDATE/DELETE User B's data.
- Unauthenticated access test: Verify no tenant data is accessible without valid auth token.
- Admin scope test: Verify admins can access only intended scopes (global `app_config` vs. tenant data).
- RLS policy coverage: 100% of tables with user/tenant data have RLS enabled.
- Policy audit: Review all RLS policies quarterly for correctness and completeness.

## 11) Theme/Branding (Future-Ready) Checklist
- Check `// Future:` comments before adding branding/layout logic.
- `UnifiedSidebar.tsx` is the extension point for logo/product/colors.
- `UnifiedLayout.tsx` is the extension point for layout variants.
- Planned components (`TopNavLayout.tsx`, `MobileDrawer.tsx`, `LayoutPicker.tsx`) are added only when explicitly requested.

## 12) Key Paths
- Layouts: `src/components/layouts/`
- Pages: `src/pages/`
- UI: `src/components/ui/`
- Stores: `src/stores/`
- Services: `src/services/` (CMS config, monitor service, etc.)
- Supabase client: `src/integrations/supabase/`
- Edge functions: `supabase/functions/`

## 13) Violations -> `CHANGELOG.md` Policy
Do not log rule violations in this file. Log each violation and lesson in `CHANGELOG.md`.

**Instructions:**
- Add a changelog entry whenever a rule from Sections 0-12 is intentionally broken.
- Include `#lessonslearned` in each entry.
- Include: date, violated section, business reason, technical outcome, and fix/next action.
- Review `#lessonslearned` entries quarterly and convert repeated patterns into checklist updates.

**Entry template (in `CHANGELOG.md`):**
`- YYYY-MM-DD: [Section X] reason -> outcome -> next action #lessonslearned`

## 14) Revision History (This Document)
- 2026-02-08 (v1.5): Embedded DASHBOARD.md and QUICKHELP.md templates as appendices with one-time extraction instructions (Section 0.0); single-file delivery model.
- 2026-02-08 (v1.4): Extracted Compliance Matrix and Architecture Snapshot to [DASHBOARD.md](./DASHBOARD.md); added [QUICKHELP.md](./QUICKHELP.md) as first-layer help; updated navigation pointers throughout.
- 2026-02-08 (v1.3): Added TL;DR non-negotiables, AI working guardrails, dependency/import contract, build-break protocol, architecture snapshot, tiered observability guidance, FSM concrete example, and revision history section.
- 2026-02-08 (v1.2): Converted to checklist-driven architecture format; added violations policy routing to `CHANGELOG.md`.

---

## Appendix A: DASHBOARD.md Template

> **AI Agent Instruction:** If `DASHBOARD.md` does not exist in the project root, create it with the content below (everything between the `<!-- BEGIN DASHBOARD TEMPLATE -->` and `<!-- END DASHBOARD TEMPLATE -->` markers). Do not include the marker comments in the extracted file.

<!-- BEGIN DASHBOARD TEMPLATE -->

# DASHBOARD.md - Build Cycle Compliance Dashboard
**Version:** 1.0
**Last Updated:** 2026-02-08
**Last Audited:** _not yet audited_
**Current Build Cycle:** _not started_

---

## How to Use This Dashboard

This is the **living status tracker** for each build cycle. Agents and humans check here first to understand current compliance state.

- Mark `[x]` only with code evidence (files, tests, policies, or logs).
- **New build cycle?** Reset all checkboxes to `[ ]` and update `Last Audited` and `Current Build Cycle` in the header.
- Each row maps to a numbered section in [AGENTS.md](./AGENTS.md). See that file for full checklist details.

---

## Compliance Matrix

| # | Section | Verify | Build | QA | Human |
|---|---------|--------|:-----:|:--:|:-----:|
| 1 | Pre-Build | Domains, folders, tenant model, typed contracts defined | [ ] | [ ] | [ ] |
| 2 | Build Contract | `UnifiedLayout` on all pages, no deprecated patterns | [ ] | [ ] | [ ] |
| 3 | CMS Content | All CMS via `useCmsConfig` + service layer, no direct Supabase | [ ] | [ ] | [ ] |
| 4 | State Mgmt | Auth via `useAuthStore`, tenant context available, no duplicate state | [ ] | [ ] | [ ] |
| 5 | Design Rules | DRY + SOLID + state hygiene applied | [ ] | [ ] | [ ] |
| 6 | FSM | Correct pattern per complexity tier, no impossible states | [ ] | [ ] | [ ] |
| 7 | Observability | Async boundaries emit telemetry, errors normalize to `AppError` | [ ] | [ ] | [ ] |
| 8 | Post Build | `typecheck`/`lint`/`test` green, journeys pass, RLS + tenant isolation verified | [ ] | [ ] | [ ] |
| 9 | Continuous Audit | Runtime signals reviewed, fixes shipped, changelog updated | [ ] | [ ] | [ ] |
| 10 | RLS | `app_config` public read, admin-only write, policies current | [ ] | [ ] | [ ] |
| 10.5 | Multi-Tenant | Tenant isolation enforced, cross-tenant access blocked, 100% RLS coverage | [ ] | [ ] | [ ] |
| 11 | Theme | Extension points untouched, no premature branding | [ ] | [ ] | [ ] |
| 12 | Key Paths | File structure matches contracted paths | [ ] | [ ] | [ ] |

**Column key:** Build = 1st build pass | QA = post-build code review | Human = manual testing

---

## Current Architecture Snapshot

> Update this section after each release. It captures the current state of key architectural decisions.

- **Layout/navigation:** Unified layout system via `src/components/layouts/` (`UnifiedLayout`, `UnifiedSidebar`).
- **CMS:** `app_config` keys `tos_html` and `footer_html`, accessed via `useCmsConfig` + `src/services/cms-config.ts`.
- **State:** Auth in `src/stores/auth-store.ts`, events in `src/stores/events-store.ts`.
- **Data security:** RLS on `app_config` and tenant-scoped tables; tenant isolation enforced by policy.
- **Observability baseline:** Structured async boundary logging + normalized `AppError` contract.
- **Snapshot fields to maintain each release:** Active routes, key tables, edge functions, and open architecture decisions.

---

## Orchestration Notes (For Agents)

### When to Reset the Dashboard
- At the **start of every new build cycle**, reset all checkboxes to `[ ]`.
- Update the `Last Audited` date in the header when an audit is completed.
- Update `Current Build Cycle` with a short label (e.g., "v1.4 feature batch" or "hotfix-auth-rls").

### When to Update the Architecture Snapshot
- After each **release or significant architecture change**.
- When new routes, tables, edge functions, or architectural decisions are added.
- Keep entries concise — this is a quick-reference, not a design doc.

### Verification Workflow
1. Run verification commands: `bun run typecheck`, `bun run lint`, `bun run test`, `supabase db diff`.
2. Walk through each matrix row and mark `Build` column based on code evidence.
3. `QA` column is marked during post-build code review.
4. `Human` column is marked after manual testing of critical journeys.
5. All three columns must be `[x]` before a build cycle is considered complete.

### Metrics to Capture (Post-Build)
- RLS policy coverage: % of tenant-data tables with RLS enabled.
- Average page load time.
- Error rate by category (from `AppError` contract).
- Record in [CHANGELOG.md](./CHANGELOG.md) alongside the version bump.

<!-- END DASHBOARD TEMPLATE -->

## Appendix B: QUICKHELP.md Template

> **AI Agent Instruction:** If `QUICKHELP.md` does not exist in the project root, create it with the content below (everything between the `<!-- BEGIN QUICKHELP TEMPLATE -->` and `<!-- END QUICKHELP TEMPLATE -->` markers). Do not include the marker comments in the extracted file.

<!-- BEGIN QUICKHELP TEMPLATE -->

# QUICKHELP.md - First-Layer Help & FAQ
**Version:** 1.0
**Last Updated:** 2026-02-08

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

---

## Document Map

| Document | Purpose | When to check |
|----------|---------|---------------|
| **[QUICKHELP.md](./QUICKHELP.md)** (this file) | First-layer help, common tasks, FAQ | First stop for any question |
| **[DASHBOARD.md](./DASHBOARD.md)** | Build cycle status and architecture snapshot | During and after each build cycle |
| **[AGENTS.md](./AGENTS.md)** | Full architecture rules and checklists | When you need detailed implementation rules |
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

<!-- END QUICKHELP TEMPLATE -->