# Lovable to Hugable
Makes your Lovable code into hugable for users.

This doc/system can be used by **non-technical users, rapid prototypers, and senior developers**.

> **Need help? For both humans and AI agents** Start with **[QUICKHELP.md](./QUICKHELP.md)** — it covers common tasks, scenarios, and FAQ with links to detailed rules.

## Who Should Use This System?

This system is for you if you want to build a Lovable app that is:
- ✅ **Easy to maintain** - Clear patterns prevent technical debt
- ✅ **Easy to extend** - New features plug into existing contracts
- ✅ **Easy for future developers to understand** - Self-documenting architecture with compliance checkpoints

**Multi-user (tenant) ready:** This system supports opening your app to friends, family, and other users when you're ready. You don't have to decide now; the multi-user data protection patterns are built in from day one, so you can more easily scale from personal use to multi-user SaaS without a rewrite.

## How to Use This System

### With Lovable AI:

1. **Upload** [AGENTS.md](./AGENTS.md) into Lovable chat before your first draft — it contains everything the AI needs, including embedded templates for DASHBOARD.md and QUICKHELP.md.
2. **Instruct** Lovable to follow AGENTS.md and extract the DASHBOARD.md and QUICKHELP.md templates from the appendices (see [§0.0](./AGENTS.md#00-file-extraction-one-time-setup)).
3. **Audit** each draft against the [Compliance Matrix](./DASHBOARD.md#compliance-matrix).
4. **Review** the **[Overall Health Grade](./DASHBOARD.md#overall-health-grade)** for a quick summary of architectural health.
5. **Each build cycle** — reset [DASHBOARD.md](./DASHBOARD.md) checkboxes and update header dates.
6. **When bugs pile up** — re-audit compliance before adding features. A broken contract upstream is often the root cause.

### With Other AI Assistants (Cursor, GitHub Copilot, Augment, etc.):

1. **Add** [AGENTS.md](./AGENTS.md) to your project root — it contains embedded templates for the other files.
2. **First run** — ask the AI to extract DASHBOARD.md and QUICKHELP.md from the appendices (see [AGENTS.md §0.0](./AGENTS.md#00-file-extraction-one-time-setup)).
3. **During development** — reference AGENTS.md in prompts: "Follow architecture rules in AGENTS.md"
4. **Before commits** — verify against the [Compliance Matrix](./DASHBOARD.md#compliance-matrix) and check the **[Overall Health Grade](./DASHBOARD.md#overall-health-grade)**.
5. **When breaking rules** — log in [CHANGELOG.md](./CHANGELOG.md) per [AGENTS.md §13](./AGENTS.md#13-violations---changelogmd-policy).

See [QUICKHELP.md](./QUICKHELP.md) for common tasks or [AGENTS.md §0](./AGENTS.md#0-how-to-use-this-doc) for the full workflow.

## Documentation

This system is organized into layered documents. Start at the top and go deeper only when needed.

| Document | Purpose |
|----------|---------|
| 📖 **[QUICKHELP.md](./QUICKHELP.md)** | First stop — common tasks, scenarios, and FAQ |
| 📊 **[DASHBOARD.md](./DASHBOARD.md)** | Build cycle compliance status and architecture snapshot |
| 📋 **[AGENTS.md](./AGENTS.md)** | Full architecture rules and checklists |
| 🧪 **[SUPABASE-TEMP-DEV.md](./SUPABASE-TEMP-DEV.md)** | Dev testing guide — reusable sandbox table for prototyping |
| 📝 **[CHANGELOG.md](./CHANGELOG.md)** | Version history, violations, and lessons learned |
| 📚 **[REFERENCES.md](./REFERENCES.md)** | Source material for design principles |

## Technical Assumptions

### **Multi-Tenant Architecture**
- **Tenancy Model**: Multi-tenant SaaS with user-level isolation (each user owns their data)
- **Security**: Row-level security (RLS) enforced on all tenant data tables
- **Isolation Strategy**: `auth.uid()` filtering via Supabase RLS policies
- **Shared vs. Tenant Data**:
  - Shared: `app_config` (global CMS content)
  - Tenant-scoped: `monitors`, `events`, `profiles`, user-specific settings
- **Future-Ready**: Architecture supports migration to org-level tenancy if needed

See [AGENTS.md Section 10.5](./AGENTS.md#105-multi-tenant-isolation-checklist) for complete multi-tenant isolation requirements.

### **Tech Stack**
- **Frontend**: React + TypeScript + Vite
- **Backend**: Supabase (database + auth + RLS + edge functions)
- **State Management**: Zustand (with tenant context in `useAuthStore()`)
- **UI Components**: Component-based with unified layout system

## Why This System Works

This architecture guide is built on **proven software engineering principles** that have stood the test of time over decades:

### **SOLID Principles**
- **Single Responsibility**: Each component, service, and store has one clear purpose
- **Open/Closed**: Extend behavior through composition, not modification
- **Liskov Substitution**: Implementations are swappable without breaking contracts
- **Interface Segregation**: Small, focused interfaces prevent unnecessary dependencies
- **Dependency Inversion**: Depend on abstractions, not concrete implementations

### **DRY (Don't Repeat Yourself)**
- One canonical implementation per behavior (one layout system, one auth source, one CMS pattern)
- Service layers eliminate duplicate data access logic
- Typed contracts prevent reimplementation of the same logic

### **State Hygiene**
- Clear separation: local UI state vs. shared state vs. persisted data
- Explicit async state modeling (`idle | loading | success | error`)
- No duplicate writable copies of the same entity

### **Observability & Error Handling**
- Standardized error contracts across all boundaries
- Telemetry at every async operation
- Normalized error categories for consistent UX

### **Continuous Verification**
- [Compliance matrix](./DASHBOARD.md#compliance-matrix) provides measurable checkpoints
- [Violations log](./CHANGELOG.md) creates a learning feedback loop
- Automated + manual checks prevent architectural drift

**Result:** Maintainable, testable, and scalable codebases that AI assistants and human developers can confidently extend without introducing technical debt.

## Getting Started

```bash
# Install dependencies
bun install

# Run development server
bun run dev

# Type checking
bun run typecheck

# Linting
bun run lint

# Tests
bun run test

# Database diff
supabase db diff
```

## Dev Testing Workflow (Outside Lovable AI Chat)

When prototyping Supabase features or testing RLS policies outside the Lovable AI chat:

### Quick Start
1. **Set up the sandbox** (one-time): Ask your AI assistant to run the setup wizard from [AGENTS.md §12.5](./AGENTS.md#125-working-outside-of-lovable-ai-chat-dev-testing-workflow)
   - Creates `temp_dev_records` and `temp_dev_reset_log` tables
   - Optionally deploys edge function for curl/external access
   - Optionally adds React components for in-app testing

2. **Start a spike**: Choose a `feature_key` (e.g., `kanban_insert_order`, `md_editor_preview`)
   ```ts
   await supabase.from('temp_dev_records').insert({
     user_id: user.id,
     feature_key: 'my_spike',
     json_value: { test: 'data' }
   });
   ```

3. **Test with curl** (if edge function deployed):
   ```bash
   curl -X POST "$SUPABASE_URL/functions/v1/temp-dev-sandbox?action=insert" \
     -H "Authorization: Bearer $JWT" \
     -H "apikey: $ANON_KEY" \
     -d '{"record":{"feature_key":"my_spike","json_value":{"test":"data"}}}'
   ```

4. **Reset between test runs**:
   ```bash
   # Via edge function
   curl -X DELETE "$SUPABASE_URL/functions/v1/temp-dev-sandbox?action=reset_feature&feature_key=my_spike"

   # Via direct SDK
   await supabase.from('temp_dev_records').delete().eq('feature_key', 'my_spike');
   ```

5. **Graduate to production**: When spike proves out, create dedicated table/repository/hook, then clean up spike data

### Use Cases
- ✅ Quick JSONB query experiments
- ✅ RLS policy prototyping
- ✅ Auth flow validation
- ✅ Drag-drop ordering tests (see Kanban Positioning Spike example)
- ✅ Cache tuning experiments
- ❌ Never reference in production code

### Troubleshooting
If sandbox setup fails, check the **Lovable Back-and-Forth Scenarios** table in [SUPABASE-TEMP-DEV.md](./SUPABASE-TEMP-DEV.md#lovable-back-and-forth-scenarios):

| Symptom | Quick Fix |
|---------|-----------|
| `404` from edge function | Wait 1-2 min for deployment propagation |
| `401 Unauthorized` | Refresh your access token |
| `Could not find table` | Apply migration + run `NOTIFY pgrst, 'reload schema';` |
| `403` / RLS errors | Verify `user_id` matches `auth.uid()` |
| Edge fails, SDK works | Check `supabase/config.toml` function settings |

**When escalating to Lovable:** Include curl command, response JSON, project ref, migration filename, and timestamp in one message.

See **[SUPABASE-TEMP-DEV.md](./SUPABASE-TEMP-DEV.md)** for complete API reference, curl examples, troubleshooting guide, and graduation workflow.

## License

**Code:** This project's code is available under standard open-source terms.

**Documentation (AGENTS.md, DASHBOARD.md, QUICKHELP.md):**
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (Creative Commons Attribution 4.0 International)
© 2026 Hypercart DBA Neochrome, Inc.

When sharing or adapting these documents, you must:
- Credit "Hypercart DBA Neochrome, Inc." as the original author
- Provide a link to the CC BY 4.0 license
- Indicate if changes were made
- Not remove attribution notices

---

For help, start with **[QUICKHELP.md](./QUICKHELP.md)**. For full architecture rules, see **[AGENTS.md](./AGENTS.md)**.
