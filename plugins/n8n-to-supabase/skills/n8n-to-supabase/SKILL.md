---
name: n8n-to-supabase
description: Migrate n8n workflows to Supabase Edge Functions, PostgreSQL tables, secrets, and pg_cron schedules. Use this skill whenever the user wants to convert n8n workflows to Supabase, migrate from n8n, translate n8n JSON to Edge Functions, or mentions "n8n to supabase", "n8n 搬到 supabase", "n8n 遷移", "migrate n8n", "convert n8n workflow", "n8n json to supabase". Also trigger when the user pastes what looks like n8n workflow JSON (containing "nodes", "connections", and node types like "n8n-nodes-base.*") and wants it running on Supabase. This skill handles the full migration: analyzing the workflow, creating database tables, writing Edge Functions, setting up secrets, and configuring pg_cron schedules.
---

# n8n → Supabase Migration

Convert n8n workflows into working Supabase infrastructure: Edge Functions (Deno/TypeScript), PostgreSQL tables, secrets, and pg_cron schedules.

## Input

The user provides one of:
1. **n8n workflow JSON** — pasted directly in chat or as a file path (`.json`)
2. **Markdown file** containing n8n JSON — a file path to a `.md` doc with embedded workflow JSON

If the user says "migrate my n8n workflow" without providing JSON, ask them to either:
- Export from n8n: **Workflow menu → Export → Download as JSON**
- Or paste the JSON directly

## Prerequisites Check

Before starting migration, verify the environment:

```bash
# 1. Supabase CLI installed?
supabase --version

# 2. Linked to a project?
supabase projects list
```

If not set up, guide the user:
```bash
npm install -g supabase    # or: brew install supabase/tap/supabase
supabase login
supabase link --project-ref <ref>   # ref is in Dashboard URL
supabase init                        # if no supabase/ directory exists
```

## Migration Workflow

Follow these steps in order. Present the migration plan (Step 2) to the user and wait for confirmation before executing (Step 3+).

### Step 1: Parse & Analyze

Parse the n8n JSON and extract:

1. **Trigger type** — look at the first node(s) with no incoming connections:
   - `n8n-nodes-base.webhook` → Edge Function HTTP endpoint
   - `n8n-nodes-base.scheduleTrigger` / `cronTrigger` → pg_cron + Edge Function
   - `n8n-nodes-base.manualTrigger` → Edge Function (manual invoke)
   - DB trigger nodes → Postgres TRIGGER + pg_net
   - Other trigger types → check `references/node-mapping.md`

2. **Node chain** — walk `connections` from trigger through all downstream nodes. Build the execution order.

3. **Credentials** — collect all `credentials` references from nodes. These become Supabase secrets. n8n JSON never contains secret values, only names and types.

4. **Data storage needs** — identify nodes that read/write data (Google Sheets, Postgres, Airtable, etc.). These need PostgreSQL tables.

5. **Feasibility check** — evaluate against Edge Function limits (read `references/limits.md`). Flag any issues.

### Step 2: Present Migration Plan

Show the user a structured plan before executing anything:

```
## Migration Plan: [workflow name]

### Trigger
- Type: [webhook / cron / manual / db-trigger]
- [If webhook]: URL will be https://<ref>.supabase.co/functions/v1/<fn-name>
- [If cron]: Schedule: <expression> (UTC — original was <timezone>)

### Edge Function
- Name: `<fn-name>`
- JWT verification: [yes/no — no for public webhooks]
- Estimated wall time: <Xs> | CPU time: <Xms>
- [If over limits]: ⚠️ Needs splitting — see plan below

### Database Tables
- `<table_name>`: <columns and types>
- [List each table needed]

### Secrets Required
- `<SECRET_NAME>`: <what it is> — ⚠️ user must provide value
- [List each secret]

### pg_cron Jobs (if any)
- `<job-name>`: `<cron-expression>` → calls <fn-name>

### Shared Modules (if any)
- `_shared/<module>.ts`: <what it provides>
```

Wait for user confirmation. The user must provide secret values before proceeding.

### Step 3: Create Database Migration

Generate a migration SQL file for all required tables:

```bash
supabase migration new n8n_<workflow-slug>
```

Write the SQL to the generated migration file. Always include:
- `CREATE TABLE` with proper types
- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
- `CREATE POLICY "service_role_all" ON <table> FOR ALL TO service_role USING (true)`
- Indexes on frequently queried columns
- Any needed extensions (`pg_cron`, `pg_net`)

For the workflow log table (include if not already present):

```sql
CREATE TABLE IF NOT EXISTS n8n_workflow_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_name TEXT NOT NULL,
  function_name TEXT NOT NULL,
  trigger_type TEXT NOT NULL,
  payload JSONB,
  status TEXT DEFAULT 'pending',
  result JSONB,
  error_message TEXT,
  execution_time_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
ALTER TABLE n8n_workflow_logs ENABLE ROW LEVEL SECURITY;
CREATE POLICY "service_role_all" ON n8n_workflow_logs FOR ALL TO service_role USING (true);
CREATE INDEX IF NOT EXISTS idx_wf_logs_name ON n8n_workflow_logs(workflow_name);
CREATE INDEX IF NOT EXISTS idx_wf_logs_created ON n8n_workflow_logs(created_at DESC);
```

Push the migration:
```bash
supabase db push --linked
```

### Step 4: Set Secrets

```bash
supabase secrets set \
  SECRET_NAME="value-from-user" \
  ANOTHER_SECRET="value" \
  --project-ref <ref>
```

All Edge Functions share the same secrets pool. Name them clearly: `<SERVICE>_<FIELD>` (e.g., `SLACK_BOT_TOKEN`, `OPENAI_API_KEY`).

Built-in secrets (already available, don't set these):
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `SUPABASE_DB_URL`

### Step 5: Write & Deploy Edge Function

Create the function:
```bash
supabase functions new <fn-name>
```

Write the TypeScript to `supabase/functions/<fn-name>/index.ts`. See `references/edge-function-templates.md` for templates by trigger type.

Key conversion rules — read `references/node-mapping.md` for the complete table, but the essentials:

| n8n Pattern | Supabase Edge Function |
|-------------|----------------------|
| `{{ $json.field }}` | `payload.field` |
| `{{ $env.KEY }}` | `Deno.env.get('KEY')` |
| HTTP Request node | `fetch(url, { method, headers, body })` |
| Google Sheets read | `supabase.from('table').select(...)` |
| Google Sheets write | `supabase.from('table').insert(...)` |
| IF node | `if (condition) { ... }` |
| Code node (JS) | Inline TypeScript |
| Code node (Python) | Rewrite as TypeScript |
| LangChain Agent | `callAIWithTools()` loop — see templates |
| Wait node | Write to `pending_tasks` table + pg_cron |

Always use `Deno.serve()` as the entry point:

```typescript
import { createClient } from "npm:@supabase/supabase-js@2"

const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
)

Deno.serve(async (req) => {
  // CORS handling
  if (req.method === "OPTIONS") {
    return new Response(null, {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
      },
    })
  }

  const startTime = Date.now()
  try {
    // ... converted workflow logic ...

    return new Response(JSON.stringify({ success: true }), {
      headers: { "Content-Type": "application/json" },
    })
  } catch (err) {
    return new Response(JSON.stringify({ error: err.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    })
  }
})
```

If the workflow is complex, extract shared logic into `supabase/functions/_shared/` modules. These are automatically bundled on deploy.

Deploy:
```bash
# Public webhook (no JWT required):
supabase functions deploy <fn-name> --no-verify-jwt --project-ref <ref>

# Internal function (JWT required):
supabase functions deploy <fn-name> --project-ref <ref>
```

### Step 6: Set Up pg_cron (if needed)

For cron/schedule triggers, set up pg_cron. Remember: **pg_cron uses UTC**.

| Common timezone | UTC offset | Example |
|----------------|-----------|---------|
| Asia/Taipei (台灣) | UTC+8 | 台灣 12:00 = UTC 04:00 → `0 4 * * *` |
| Asia/Tokyo | UTC+9 | Tokyo 12:00 = UTC 03:00 → `0 3 * * *` |
| America/New_York (EST) | UTC-5 | NY 12:00 = UTC 17:00 → `0 17 * * *` |

```bash
supabase db query "
  SELECT cron.schedule(
    '<job-name>',
    '<cron-expression-in-utc>',
    \$\$
    SELECT net.http_post(
      url := 'https://<ref>.supabase.co/functions/v1/<fn-name>',
      headers := jsonb_build_object(
        'Content-Type', 'application/json',
        'Authorization', 'Bearer <service-role-key>',
        'x-cron-secret', '<cron-secret>'
      ),
      body := '{}'::jsonb
    );
    \$\$
  );
" --linked
```

Add cron validation to the Edge Function:
```typescript
const CRON_SECRET = Deno.env.get("CRON_SECRET")!
const cronSecret = req.headers.get("x-cron-secret")
if (cronSecret !== CRON_SECRET) {
  return new Response("Unauthorized", { status: 401 })
}
```

Set the cron secret:
```bash
supabase secrets set CRON_SECRET="<generate-a-random-string>" --project-ref <ref>
```

### Step 7: Set Up DB Triggers (if needed)

For workflows triggered by database changes:

```sql
CREATE OR REPLACE FUNCTION trigger_<fn_name>()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM net.http_post(
    url := 'https://<ref>.supabase.co/functions/v1/<fn-name>',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
    ),
    body := jsonb_build_object(
      'type', TG_OP,
      'table', TG_TABLE_NAME,
      'record', row_to_json(NEW),
      'old_record', CASE WHEN TG_OP = 'UPDATE' THEN row_to_json(OLD) ELSE NULL END
    )
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER <trigger_name>
  AFTER INSERT OR UPDATE ON <table_name>
  FOR EACH ROW
  EXECUTE FUNCTION trigger_<fn_name>();
```

### Step 8: Verify

Run all verification checks:

```bash
# Functions deployed?
supabase functions list --project-ref <ref>

# Secrets set?
supabase secrets list --project-ref <ref>

# Tables created?
supabase db query "SELECT tablename FROM pg_tables WHERE schemaname = 'public'" --linked

# pg_cron jobs?
supabase db query "SELECT jobname, schedule, command FROM cron.job" --linked

# Test the function
curl -X POST "https://<ref>.supabase.co/functions/v1/<fn-name>" \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

Present results to the user with a summary of what was created.

## Edge Cases & Special Handling

### Wait/Delay Nodes
n8n's Wait node can't be directly translated (Edge Functions have wall time limits). Instead:
1. Create a `pending_tasks` table
2. Write the delayed task to it with `execute_after` timestamp
3. Set up pg_cron to check and process pending tasks every minute

### executeWorkflow (Sub-workflows)
Convert to `fetch()` calling another Edge Function, or extract shared logic to `_shared/` modules and import directly.

### Python Code Nodes
Supabase Edge Functions only run Deno (TypeScript/JavaScript). Python code must be rewritten as TypeScript. If the Python logic is complex, explain to the user what needs manual review.

### Large Payload (>6MB)
Use Supabase Storage as a relay:
1. Upload data to a Storage bucket
2. Pass the storage path to the function
3. Function downloads from Storage, processes, and uploads result

### n8n Expressions in Prompts
n8n templates like `{{ $('Node Name').first().json.field }}` and `{{ $now.format('yyyy-MM-dd') }}` must be converted to template literal interpolation or a `buildPrompt()` helper. See `references/node-mapping.md` for the full expression conversion table.

## Reference Files

- **`references/node-mapping.md`** — Complete n8n node type → Supabase implementation mapping, including expression conversion and credential mapping
- **`references/edge-function-templates.md`** — Copy-paste templates for webhook, cron, DB trigger, and AI tool-calling functions
- **`references/limits.md`** — Edge Function wall time, CPU time, and payload limits with assessment of which n8n patterns are safe vs need splitting
