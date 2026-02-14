# Supabase Temp Dev Sandbox

## Purpose

A portable, reusable prototyping system for Lovable Cloud projects. Drop two tables and one edge function into any project to get a flexible, user-scoped scratchpad for feature spikes — no dedicated tables, no custom edge functions, no schema churn until you're ready.

---

## Quick-Start for New Projects

### 1. Run the Migration

Paste this into a Lovable Cloud migration (or `supabase db push`):

```sql
-- ══════════════════════════════════════════════
-- Temp Dev Sandbox: reusable spike tables
-- ══════════════════════════════════════════════

CREATE TABLE IF NOT EXISTS public.temp_dev_records (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL,
  feature_key   TEXT NOT NULL DEFAULT 'general',
  record_key    TEXT,
  text_value    TEXT,
  number_value  DOUBLE PRECISION,
  bool_value    BOOLEAN,
  json_value    JSONB,
  tags          TEXT[] DEFAULT '{}',
  status        TEXT NOT NULL DEFAULT 'active',
  sort_index    INTEGER NOT NULL DEFAULT 0,
  notes         TEXT,
  expires_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS public.temp_dev_reset_log (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL,
  feature_key    TEXT NOT NULL,
  action         TEXT NOT NULL,          -- reset_feature | reset_all | purge_expired
  reason         TEXT,
  deleted_count  INTEGER NOT NULL DEFAULT 0,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RLS
ALTER TABLE public.temp_dev_records   ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.temp_dev_reset_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own temp records"
  ON public.temp_dev_records FOR ALL
  USING  (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users manage own reset log"
  ON public.temp_dev_reset_log FOR ALL
  USING  (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Auto-update timestamp
CREATE OR REPLACE FUNCTION public.update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql SET search_path = public;

CREATE TRIGGER update_temp_dev_records_updated_at
  BEFORE UPDATE ON public.temp_dev_records
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at_column();
```

### 2. Deploy the Edge Function

Create `supabase/functions/temp-dev-sandbox/index.ts` with the edge function code from this project (see **Edge Function API** below).

Add to `supabase/config.toml`:
```toml
[functions.temp-dev-sandbox]
verify_jwt = false
```

### 3. Optional: App-Side Layer

Copy these files for in-app usage:
- `src/repositories/TempDevRepository.ts` — Supabase client CRUD
- `src/hooks/useTempDevSandbox.ts` — React hook
- `src/pages/TempDevSandbox.tsx` — UI spike page (route: `/spike/db-dev-sandbox`)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Spike Page / Component                                 │
│  useTempDevSandbox({ userId, featureKey })               │
├──────────────┬──────────────────────────────────────────┤
│  Direct SDK  │  Edge Function (curl / external tools)   │
│  TempDevRepo │  POST /temp-dev-sandbox?action=insert    │
├──────────────┴──────────────────────────────────────────┤
│  temp_dev_records  (user_id + feature_key scoped)       │
│  temp_dev_reset_log (audit trail)                       │
└─────────────────────────────────────────────────────────┘
```

Two access paths, one data layer:
- **Direct SDK** — fastest for in-app React code via the repository/hook.
- **Edge Function** — for curl testing, external LLMs, CI scripts, webhooks.

---

## Feature-Key Pattern

Every record is scoped by `feature_key`. This lets multiple spikes share one table without collision.

| Feature Key | Purpose |
|---|---|
| `kanban_insert_order` | Test drag-drop ordering logic |
| `md_editor_preview_experiment` | Preview rendering tests |
| `issues_cache_tuning` | Cache TTL experiments |
| `onboarding_flow_v2` | New user flow prototype |

**Rule:** Always set an explicit `feature_key`. The default is `'general'` but avoid it for real spikes.

## Kanban Positioning Spike Workflow

Use route:
- `/spike/db-dev-sandbox`

Recommended feature key:
- `kanban_insert_order`

The page now includes a **Kanban Positioning Spike (Two-Set Insert)** panel that lets you:
- model current-column reorder (`source_column_id === destination_column_id`)
- model cross-column insert (`source_column_id !== destination_column_id`)
- persist each case snapshot as one temp record

Stored JSON payload shape:

```json
{
  "source_column_id": "doing",
  "destination_column_id": "testing",
  "moving_card_id": "PR-123",
  "source_cards_before": ["PR-101", "PR-123", "PR-204"],
  "destination_cards_before": ["PR-305", "PR-401"],
  "destination_index_requested": 1,
  "destination_index_applied": 1,
  "source_cards_after": ["PR-101", "PR-204"],
  "destination_cards_after": ["PR-305", "PR-123", "PR-401"]
}
```

This gives a repeatable DB-backed dataset for evaluating insert-position algorithms before production schema changes.

---

## Edge Function API

Base URL: `{SUPABASE_URL}/functions/v1/temp-dev-sandbox`

All requests require `Authorization: Bearer {USER_JWT}` and `apikey: {ANON_KEY}` headers.

### GET — Fetch Data

| Endpoint | Description |
|---|---|
| `?resource=records&feature_key={key}` | Fetch records for a feature key |
| `?resource=reset_log&feature_key={key}&limit=10` | Fetch reset audit log |

### POST — Write Data

| Endpoint | Body | Description |
|---|---|---|
| `?action=insert` | `{ "record": { "feature_key": "...", ... } }` | Insert a record |
| `?action=update` | `{ "record": { "id": "uuid", "feature_key": "...", ... } }` | Update a record |
| `?action=smoke_test&feature_key={key}` | _(none)_ | Insert→fetch→delete round-trip |

#### Record Fields

```json
{
  "feature_key": "required",
  "record_key": "optional unique label",
  "text_value": "string",
  "number_value": 42,
  "bool_value": true,
  "json_value": { "any": "structured data" },
  "tags": ["tag1", "tag2"],
  "status": "active",
  "sort_index": 0,
  "notes": "free text",
  "expires_at": "2026-03-01T00:00:00Z"
}
```

### DELETE — Remove Data

| Endpoint | Body | Description |
|---|---|---|
| `?action=record&id={uuid}` | _(none)_ | Delete one record |
| `?action=reset_feature&feature_key={key}` | `{ "reason": "..." }` _(optional)_ | Delete all records for feature key |
| `?action=reset_all` | `{ "reason": "..." }` _(optional)_ | Delete all user's temp records |
| `?action=purge_expired&feature_key={key}` | `{ "reason": "..." }` _(optional)_ | Delete expired records only |

All delete/reset actions are logged in `temp_dev_reset_log`.

---

## Curl Examples

```bash
# ── Setup ──
export SUPABASE_URL="https://YOUR_PROJECT.supabase.co"
export ANON_KEY="your-anon-key"
export USER_JWT="your-access-token"
export FN="$SUPABASE_URL/functions/v1/temp-dev-sandbox"

# ── Smoke Test ──
curl -sS -X POST "$FN?action=smoke_test&feature_key=edge-test" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "apikey: $ANON_KEY" \
  -H "Content-Type: application/json"

# ── Insert ──
curl -sS -X POST "$FN?action=insert" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "apikey: $ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"record":{"feature_key":"kanban_insert_order","record_key":"case-001","text_value":"test","json_value":{"from":"doing","to":"testing"}}}'

# ── Fetch ──
curl -sS "$FN?resource=records&feature_key=kanban_insert_order" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "apikey: $ANON_KEY"

# ── Delete One ──
curl -sS -X DELETE "$FN?action=record&id=RECORD_UUID" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "apikey: $ANON_KEY"

# ── Reset Feature ──
curl -sS -X DELETE "$FN?action=reset_feature&feature_key=kanban_insert_order" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "apikey: $ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"reason":"cleanup after testing"}'
```

## Direct REST API (No Edge Function)

You can also hit the tables directly via Supabase PostgREST:

```bash
# Insert
curl -sS -X POST "$SUPABASE_URL/rest/v1/temp_dev_records" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $USER_JWT" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"user_id":"YOUR_UUID","feature_key":"my_spike","record_key":"case-001","json_value":{"test":true}}'

# Fetch
curl -sS "$SUPABASE_URL/rest/v1/temp_dev_records?feature_key=eq.my_spike&select=*" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $USER_JWT"
```

---

## AJAX Example (supabase-js)

```ts
import { supabase } from '@/integrations/supabase/client';

// Insert
const { error } = await supabase.from('temp_dev_records').insert({
  user_id: user.id,
  feature_key: 'kanban_insert_order',
  record_key: 'ajax-001',
  json_value: { from: 'doing', to: 'backlog' },
});

// Fetch
const { data } = await supabase
  .from('temp_dev_records')
  .select('*')
  .eq('user_id', user.id)
  .eq('feature_key', 'kanban_insert_order')
  .order('sort_index');
```

---

## Reset Strategy

| Action | Scope | Logged |
|---|---|---|
| **Reset Feature** | Deletes records for current `feature_key` | ✅ |
| **Erase All** | Deletes all temp records for current user | ✅ |
| **Purge Expired** | Deletes rows where `expires_at <= now()` | ✅ |

All actions write to `temp_dev_reset_log` with `deleted_count` and optional `reason`.

---

## Graduation: Spike → Production

When a spike proves out and is ready for production:

1. **Create a dedicated table** with a proper migration (typed columns, constraints, indexes).
2. **Create a dedicated repository** in `src/repositories/`.
3. **Create a dedicated hook** in `src/hooks/`.
4. **Clean up** — run `DELETE ?action=reset_feature&feature_key={key}` to remove spike data.
5. **Document** — log the graduation in `CHANGELOG.md`.

---

## Guardrails

- This data is **non-production and disposable**. Never rely on it for real users.
- **Do not store secrets** in `temp_dev_records`.
- Keep `feature_key` **explicit** for every spike.
- Prefer adding new JSON payload keys over altering the schema.
- Set `expires_at` for time-boxed experiments so `purge_expired` can clean up automatically.

---

## Lovable Back-and-Forth Scenarios

Use this as the escalation checklist when Codex/app-side tests fail and you need Lovable to adjust cloud-side setup.

| Symptom | Likely Cause | What to Ask Lovable |
|---|---|---|
| `404` from `/functions/v1/temp-dev-sandbox` | Function not deployed yet, wrong project ref, or propagation delay | Confirm function deployed to the correct project ref; wait 1-2 minutes and re-test. |
| `401 Unauthorized` | Missing/expired `USER_JWT`, malformed `Authorization` header | Confirm auth flow is active and provide fresh access token retrieval steps. |
| `Could not find the table 'public.temp_dev_records' in the schema cache` | Migration not applied remotely, or PostgREST schema cache not refreshed | Apply `20260214214000_temp_dev_db_sandbox.sql` on hosted DB and run `NOTIFY pgrst, 'reload schema';`. |
| `403` / RLS policy errors | Missing or incorrect RLS policies, or `user_id` not matching `auth.uid()` | Re-verify table RLS policies and tenant isolation checks for both temp tables. |
| Edge calls fail but direct SDK fallback works | Edge function config mismatch (`verify_jwt`, env headers, claim parsing) | Confirm `supabase/config.toml` function settings and edge auth validation logic match this repo contract. |
| Smoke test passes but UI page still errors | Frontend env/session issue (`VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, missing session) | Validate Lovable environment variables and session availability in browser runtime. |
| Build/type errors after Lovable rewrites files | Drift between generated cloud edits and local repo types/contracts | Ask Lovable for exact file diff + failing compiler output; then patch locally with minimal contract-preserving changes. |

### Minimum Handoff Bundle to Send Lovable

When escalation is needed, include all of the following in one message:

1. Exact failing curl command and response JSON.
2. Project ref and function URL.
3. Current migration filename expected (`20260214214000_temp_dev_db_sandbox.sql`).
4. Whether failure is edge-only or also direct SDK.
5. Timestamp of latest deploy/migration attempt.

This short bundle prevents slow back-and-forth and makes cloud-side triage much faster.

---

## LLM Handoff Prompt

Use this prompt when handing off to another LLM for spike work:

> You have access to a Temp Dev Sandbox on this Lovable Cloud project. It provides two tables (`temp_dev_records`, `temp_dev_reset_log`) and an edge function at `/functions/v1/temp-dev-sandbox` for CRUD operations scoped by `user_id` and `feature_key`.
>
> **To start a new spike:**
> 1. Choose a descriptive `feature_key` (e.g., `my_new_feature_v1`).
> 2. Use `POST ?action=insert` to store test data with `json_value` for structured payloads.
> 3. Use `GET ?resource=records&feature_key={key}` to read back.
> 4. Use `DELETE ?action=reset_feature&feature_key={key}` to clean up when done.
> 5. Run `POST ?action=smoke_test&feature_key={key}` to verify the system works.
>
> All requests need `Authorization: Bearer {JWT}` and `apikey: {ANON_KEY}` headers.
> See `SUPABASE-TEMP-DEV.md` for the full API reference and curl examples.

---

## Files Reference

| File | Purpose |
|---|---|
| `supabase/migrations/20260214214000_temp_dev_db_sandbox.sql` | Schema migration |
| `supabase/functions/temp-dev-sandbox/index.ts` | Edge function (external API) |
| `src/repositories/TempDevRepository.ts` | Direct SDK repository |
| `src/hooks/useTempDevSandbox.ts` | React hook |
| `src/pages/TempDevSandbox.tsx` | Spike UI page (`/spike/db-dev-sandbox`) |
| `SUPABASE-TEMP-DEV.md` | This doc |
