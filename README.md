# n8n-to-supabase

A [Claude Code](https://claude.ai/claude-code) skill that migrates n8n workflows to Supabase — converting visual workflows into Edge Functions, PostgreSQL tables, secrets, and pg_cron schedules.

## What it does

Paste your n8n workflow JSON and this skill will:

1. **Analyze** — Parse nodes, connections, triggers, and credentials
2. **Plan** — Generate a migration plan with tables, secrets, functions, and cron jobs
3. **Execute** — Create everything via Supabase CLI (`supabase db push`, `functions deploy`, `secrets set`, etc.)
4. **Verify** — Run checks to confirm everything is working

## Install

```bash
/plugin marketplace add Kazzx9921/n8n-to-supabase
/plugin install n8n-to-supabase@n8n-to-supabase
```

## Usage

```bash
/n8n-to-supabase
```

Then paste your n8n workflow JSON (exported from n8n: **Workflow menu → Export → Download as JSON**).

Or provide a file path:

```bash
/n8n-to-supabase /path/to/workflow.json
```

## What gets migrated

| n8n | Supabase |
|-----|----------|
| Webhook Trigger | Edge Function endpoint |
| Schedule/Cron Trigger | pg_cron + Edge Function |
| HTTP Request node | `fetch()` |
| Code node (JS) | Inline TypeScript |
| IF/Switch/Set nodes | TypeScript logic |
| Google Sheets | PostgreSQL table |
| LangChain Agent | AI tool-calling loop |
| Credentials | `supabase secrets set` |
| Data Tables | `CREATE TABLE` via migration |

40+ node types supported. See [references/node-mapping.md](plugins/n8n-to-supabase/skills/n8n-to-supabase/references/node-mapping.md) for the full mapping.

## Edge Function limits

The skill evaluates your workflow against Supabase Edge Function limits (wall time, CPU time, payload size) and flags anything that needs restructuring.

| Limit | Free | Pro |
|-------|------|-----|
| Wall Time | 150s | 400s |
| CPU Time | 50ms | 2s |
| Payload | 1MB | 6MB |

> 90%+ of typical n8n workflows (webhook → API calls → DB) fit comfortably within these limits.

## Prerequisites

- [Supabase CLI](https://supabase.com/docs/guides/cli) installed and linked to a project
- [Claude Code](https://claude.ai/claude-code) installed

```bash
npm install -g supabase
supabase login
supabase link --project-ref <your-project-ref>
```

## Skill structure

```
plugins/n8n-to-supabase/skills/n8n-to-supabase/
├── SKILL.md                    # Main migration workflow (8 steps)
└── references/
    ├── node-mapping.md         # 40+ n8n node → Supabase conversion table
    ├── edge-function-templates.md  # 6 templates (webhook, cron, DB trigger, AI, queue)
    └── limits.md               # Edge Function limits & feasibility assessment
```

## License

MIT
