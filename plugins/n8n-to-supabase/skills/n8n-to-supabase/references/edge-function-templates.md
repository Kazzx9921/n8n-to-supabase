# Edge Function Templates

Copy-paste templates for different trigger types. Adjust to match the specific workflow being migrated.

## Shared Client Module

Create this first — all functions will import it:

```typescript
// supabase/functions/_shared/supabase.ts
import { createClient } from "npm:@supabase/supabase-js@2"

export const db = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
)
```

## Template 1: Webhook (Public)

For n8n Webhook Trigger nodes. Deploy with `--no-verify-jwt`.

```typescript
// supabase/functions/<fn-name>/index.ts
import { db } from "../_shared/supabase.ts"

const CORS_HEADERS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
}

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response(null, { status: 204, headers: CORS_HEADERS })
  }

  const startTime = Date.now()

  try {
    const payload = await req.json()

    // ========================================
    // CONVERTED WORKFLOW LOGIC GOES HERE
    // ========================================

    // Example: write to DB
    // const { data, error } = await db.from("table").insert({ ... })

    // Log execution
    await db.from("n8n_workflow_logs").insert({
      workflow_name: "<original-n8n-name>",
      function_name: "<fn-name>",
      trigger_type: "webhook",
      payload,
      status: "completed",
      result: {},
      execution_time_ms: Date.now() - startTime,
    })

    return new Response(JSON.stringify({ success: true }), {
      headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
    })
  } catch (err) {
    await db.from("n8n_workflow_logs").insert({
      workflow_name: "<original-n8n-name>",
      function_name: "<fn-name>",
      trigger_type: "webhook",
      status: "failed",
      error_message: err.message,
      execution_time_ms: Date.now() - startTime,
    }).catch(() => {}) // don't fail on log error

    return new Response(JSON.stringify({ error: err.message }), {
      status: 500,
      headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
    })
  }
})
```

## Template 2: Webhook with Signature Verification (LINE, Stripe, etc.)

For webhooks that need HMAC signature verification:

```typescript
import { db } from "../_shared/supabase.ts"

async function verifySignature(body: string, signature: string, secret: string): Promise<boolean> {
  const key = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"]
  )
  const sig = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(body))
  const expected = btoa(String.fromCharCode(...new Uint8Array(sig)))
  return expected === signature
}

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response(null, { status: 204, headers: { "Access-Control-Allow-Origin": "*" } })
  }

  const body = await req.text()

  // LINE signature verification
  const signature = req.headers.get("x-line-signature") ?? ""
  const secret = Deno.env.get("LINE_CHANNEL_SECRET")!
  if (!await verifySignature(body, signature, secret)) {
    return new Response("Invalid signature", { status: 401 })
  }

  const payload = JSON.parse(body)

  try {
    // ========================================
    // CONVERTED WORKFLOW LOGIC GOES HERE
    // ========================================

    return new Response(JSON.stringify({ success: true }), {
      headers: { "Content-Type": "application/json" },
    })
  } catch (err) {
    console.error("Error:", err)
    return new Response(JSON.stringify({ error: err.message }), { status: 500 })
  }
})
```

## Template 3: Cron / Schedule

For n8n Schedule/Cron Trigger nodes. Uses `x-cron-secret` header for auth.

```typescript
import { db } from "../_shared/supabase.ts"

Deno.serve(async (req) => {
  // Verify cron caller
  const cronSecret = req.headers.get("x-cron-secret")
  if (cronSecret !== Deno.env.get("CRON_SECRET")) {
    return new Response("Unauthorized", { status: 401 })
  }

  const startTime = Date.now()

  try {
    // Read last execution state (replaces n8n staticData)
    const { data: state } = await db
      .from("n8n_workflow_state")
      .select("state")
      .eq("workflow_name", "<fn-name>")
      .single()

    const cursor = state?.state?.cursor ?? null

    // ========================================
    // CONVERTED CRON WORKFLOW LOGIC GOES HERE
    // Example: fetch new items since last run
    // ========================================

    let query = db.from("target_table").select("*").order("id").limit(100)
    if (cursor) query = query.gt("id", cursor)
    const { data: items } = await query

    if (!items?.length) {
      return new Response(JSON.stringify({ message: "No new items" }))
    }

    // Process items...

    // Update cursor for next run
    await db.from("n8n_workflow_state").upsert({
      workflow_name: "<fn-name>",
      state: { cursor: items[items.length - 1].id },
      updated_at: new Date().toISOString(),
    })

    // Log
    await db.from("n8n_workflow_logs").insert({
      workflow_name: "<original-n8n-name>",
      function_name: "<fn-name>",
      trigger_type: "cron",
      status: "completed",
      result: { processed: items.length },
      execution_time_ms: Date.now() - startTime,
    })

    return new Response(JSON.stringify({ success: true, processed: items.length }), {
      headers: { "Content-Type": "application/json" },
    })
  } catch (err) {
    await db.from("n8n_workflow_logs").insert({
      workflow_name: "<original-n8n-name>",
      function_name: "<fn-name>",
      trigger_type: "cron",
      status: "failed",
      error_message: err.message,
      execution_time_ms: Date.now() - startTime,
    }).catch(() => {})

    return new Response(JSON.stringify({ error: err.message }), { status: 500 })
  }
})
```

**pg_cron setup SQL:**
```sql
SELECT cron.schedule(
  '<job-name>',
  '<cron-expression-utc>',
  $$
  SELECT net.http_post(
    url := 'https://<ref>.supabase.co/functions/v1/<fn-name>',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'x-cron-secret', '<cron-secret-value>'
    ),
    body := '{}'::jsonb
  );
  $$
);
```

## Template 4: DB Trigger Handler

For workflows triggered by database INSERT/UPDATE/DELETE:

```typescript
import { db } from "../_shared/supabase.ts"

interface TriggerPayload {
  type: "INSERT" | "UPDATE" | "DELETE"
  table: string
  record: Record<string, unknown>
  old_record: Record<string, unknown> | null
}

Deno.serve(async (req) => {
  // Verify internal caller
  const authHeader = req.headers.get("Authorization")
  if (!authHeader?.includes(Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!)) {
    return new Response("Unauthorized", { status: 401 })
  }

  try {
    const { type, table, record, old_record }: TriggerPayload = await req.json()

    switch (type) {
      case "INSERT": {
        // ========================================
        // ON INSERT LOGIC
        // ========================================
        break
      }
      case "UPDATE": {
        // ========================================
        // ON UPDATE LOGIC
        // ========================================
        break
      }
      case "DELETE": {
        // ========================================
        // ON DELETE LOGIC
        // ========================================
        break
      }
    }

    return new Response(JSON.stringify({ success: true }), {
      headers: { "Content-Type": "application/json" },
    })
  } catch (err) {
    console.error("Trigger handler error:", err)
    return new Response(JSON.stringify({ error: err.message }), { status: 500 })
  }
})
```

**DB trigger SQL:**
```sql
-- Create the trigger function
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

-- Attach to table
CREATE TRIGGER <trigger_name>
  AFTER INSERT OR UPDATE ON <table_name>
  FOR EACH ROW
  EXECUTE FUNCTION trigger_<fn_name>();
```

## Template 5: AI Tool Calling Loop

Replaces n8n's LangChain Agent node with tool use:

```typescript
// supabase/functions/_shared/ai.ts
const OPENAI_API_KEY = Deno.env.get("OPENAI_API_KEY")!

interface Tool {
  type: "function"
  function: {
    name: string
    description: string
    parameters: Record<string, unknown>
  }
}

interface Message {
  role: "system" | "user" | "assistant" | "tool"
  content?: string
  tool_calls?: Array<{ id: string; function: { name: string; arguments: string } }>
  tool_call_id?: string
}

export async function callAI(
  systemPrompt: string,
  userPrompt: string,
  model = "gpt-4o"
): Promise<string> {
  const res = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userPrompt },
      ],
    }),
  })
  const data = await res.json()
  return data.choices[0].message.content
}

export async function callAIWithTools(
  systemPrompt: string,
  userPrompt: string,
  tools: Tool[],
  executeTool: (name: string, args: Record<string, unknown>) => Promise<string>,
  model = "gpt-4o",
  maxRounds = 5
): Promise<string> {
  const messages: Message[] = [
    { role: "system", content: systemPrompt },
    { role: "user", content: userPrompt },
  ]

  for (let i = 0; i < maxRounds; i++) {
    const res = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${OPENAI_API_KEY}`,
      },
      body: JSON.stringify({ model, messages, tools }),
    })
    const data = await res.json()
    const choice = data.choices[0]

    if (choice.finish_reason === "tool_calls" || choice.message.tool_calls) {
      messages.push(choice.message)
      for (const tc of choice.message.tool_calls) {
        const result = await executeTool(tc.function.name, JSON.parse(tc.function.arguments))
        messages.push({ role: "tool", tool_call_id: tc.id, content: result })
      }
    } else {
      return choice.message.content ?? ""
    }
  }

  return messages[messages.length - 1].content ?? ""
}
```

## Template 6: Pending Tasks Queue (Wait Node Replacement)

For n8n Wait/Delay nodes — write task to queue, process via pg_cron:

```typescript
// Writer function — replaces the Wait node by queuing
async function scheduleDelayedTask(
  db: SupabaseClient,
  workflowName: string,
  taskType: string,
  payload: Record<string, unknown>,
  delayMinutes: number
) {
  const executeAfter = new Date(Date.now() + delayMinutes * 60_000).toISOString()
  await db.from("n8n_pending_tasks").insert({
    workflow_name: workflowName,
    task_type: taskType,
    payload,
    execute_after: executeAfter,
  })
}

// Processor function — runs via pg_cron every minute
// supabase/functions/process-pending-tasks/index.ts
import { db } from "../_shared/supabase.ts"

Deno.serve(async (req) => {
  const cronSecret = req.headers.get("x-cron-secret")
  if (cronSecret !== Deno.env.get("CRON_SECRET")) {
    return new Response("Unauthorized", { status: 401 })
  }

  // Grab tasks ready to execute (with row locking)
  const { data: tasks } = await db
    .from("n8n_pending_tasks")
    .select("*")
    .eq("status", "pending")
    .lte("execute_after", new Date().toISOString())
    .order("execute_after")
    .limit(10)

  if (!tasks?.length) {
    return new Response(JSON.stringify({ processed: 0 }))
  }

  let processed = 0
  for (const task of tasks) {
    try {
      // Mark as processing
      await db.from("n8n_pending_tasks").update({ status: "processing" }).eq("id", task.id)

      // Execute the delayed logic based on task_type
      switch (task.task_type) {
        case "send_email":
          // ... send email logic ...
          break
        case "send_notification":
          // ... notification logic ...
          break
      }

      await db.from("n8n_pending_tasks").update({ status: "completed" }).eq("id", task.id)
      processed++
    } catch (err) {
      const retryCount = (task.retry_count ?? 0) + 1
      await db.from("n8n_pending_tasks").update({
        status: retryCount >= task.max_retries ? "failed" : "pending",
        retry_count: retryCount,
      }).eq("id", task.id)
    }
  }

  return new Response(JSON.stringify({ processed }))
})
```

**pg_cron for task processor (every minute):**
```sql
SELECT cron.schedule(
  'process-pending-tasks',
  '* * * * *',
  $$
  SELECT net.http_post(
    url := 'https://<ref>.supabase.co/functions/v1/process-pending-tasks',
    headers := jsonb_build_object('x-cron-secret', '<cron-secret>'),
    body := '{}'::jsonb
  );
  $$
);
```
