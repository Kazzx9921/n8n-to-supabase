# n8n Node → Supabase Implementation Mapping

## Trigger Nodes

| n8n Node Type | Supabase Implementation | Deploy Flag |
|---------------|------------------------|-------------|
| `n8n-nodes-base.webhook` | Edge Function endpoint | `--no-verify-jwt` (public) |
| `n8n-nodes-base.scheduleTrigger` | pg_cron → Edge Function | Standard deploy |
| `n8n-nodes-base.cronTrigger` | pg_cron → Edge Function | Standard deploy |
| `n8n-nodes-base.manualTrigger` | Edge Function (curl to invoke) | Standard deploy |
| `n8n-nodes-base.emailReadImap` | pg_cron polling + Edge Function | Standard deploy |
| `n8n-nodes-base.executeWorkflow` | Extract to `_shared/` module or `fetch()` another function | — |
| DB trigger nodes | Postgres TRIGGER + `net.http_post()` | Standard deploy |

## Core Logic Nodes

| n8n Node | Supabase TypeScript |
|----------|-------------------|
| `httpRequest` | `const res = await fetch(url, { method, headers, body: JSON.stringify(data) }); const json = await res.json();` |
| `code` (JavaScript) | Inline TypeScript — paste code directly, fix type issues |
| `code` (Python) | Rewrite as TypeScript (no Python runtime in Deno) |
| `if` | `if (condition) { /* true branch */ } else { /* false branch */ }` |
| `switch` | `switch (value) { case 'a': ...; break; case 'b': ...; break; }` |
| `set` | `const result = { ...inputData, newField: value }` |
| `merge` (append) | `const merged = [...arrayA, ...arrayB]` |
| `merge` (combine) | `const [a, b] = await Promise.all([taskA(), taskB()])` |
| `splitInBatches` | `for (let i = 0; i < items.length; i += batchSize) { const batch = items.slice(i, i + batchSize); ... }` |
| `wait` | **Cannot sleep** — write to `pending_tasks` table, process via pg_cron |
| `respondToWebhook` | `return new Response(body, { status: 200, headers })` |
| `noOp` | Remove entirely |
| `executeWorkflow` | `await fetch('https://<ref>.supabase.co/functions/v1/<other-fn>', { headers: { Authorization: 'Bearer ...' }, body: ... })` |

## Data Nodes

| n8n Node | Supabase Implementation |
|----------|------------------------|
| `postgres` (query) | `const { data, error } = await supabase.from('table').select('*').eq('col', val)` |
| `postgres` (insert) | `await supabase.from('table').insert([{ ... }])` |
| `postgres` (update) | `await supabase.from('table').update({ ... }).eq('id', id)` |
| `postgres` (delete) | `await supabase.from('table').delete().eq('id', id)` |
| `postgres` (raw SQL) | `await supabase.rpc('function_name', { params })` |
| `googleSheets` (read) | Convert to PostgreSQL table + `.select()` |
| `googleSheets` (append) | Convert to PostgreSQL table + `.insert()` |
| `googleSheets` (update) | Convert to PostgreSQL table + `.update().eq()` |
| `googleSheets` (lookup) | Convert to PostgreSQL table + `.select().eq()` |
| `airtable` | `fetch('https://api.airtable.com/v0/...', { headers: { Authorization: 'Bearer ' + Deno.env.get('AIRTABLE_API_KEY') } })` |
| `notion` | `fetch('https://api.notion.com/v1/...', { headers: { Authorization: 'Bearer ' + Deno.env.get('NOTION_API_KEY'), 'Notion-Version': '2022-06-28' } })` |

## AI / LLM Nodes

| n8n Node | Supabase Implementation |
|----------|------------------------|
| `openAi` (chat) | `fetch('https://api.openai.com/v1/chat/completions', { headers: { Authorization: 'Bearer ' + Deno.env.get('OPENAI_API_KEY') }, body: JSON.stringify({ model, messages }) })` |
| LangChain Agent (no tools) | Direct API call to LLM |
| LangChain Agent (with tools) | Tool-calling loop — see `edge-function-templates.md` |
| Perplexity tool | `fetch('https://api.perplexity.ai/chat/completions', { headers: { Authorization: 'Bearer ' + Deno.env.get('PERPLEXITY_API_KEY') }, body: ... })` |

## External Service Nodes

| n8n Node | Supabase Implementation |
|----------|------------------------|
| `slack` (send) | `fetch('https://slack.com/api/chat.postMessage', { headers: { Authorization: 'Bearer ' + Deno.env.get('SLACK_BOT_TOKEN') }, body: ... })` |
| `gmail` (send) | Use Resend or direct Gmail API with OAuth |
| `telegram` (send) | `fetch('https://api.telegram.org/bot' + Deno.env.get('TELEGRAM_BOT_TOKEN') + '/sendMessage', { body: ... })` |
| `discord` (send) | `fetch(Deno.env.get('DISCORD_WEBHOOK_URL'), { body: JSON.stringify({ content: '...' }) })` |
| LINE Messaging (push) | `fetch('https://api.line.me/v2/bot/message/push', { headers: { Authorization: 'Bearer ' + Deno.env.get('LINE_CHANNEL_ACCESS_TOKEN') }, body: ... })` |
| LINE Messaging (reply) | `fetch('https://api.line.me/v2/bot/message/reply', { headers: { Authorization: 'Bearer ' + Deno.env.get('LINE_CHANNEL_ACCESS_TOKEN') }, body: ... })` |
| `httpRequest` (generic) | `fetch(url, { method, headers, body })` |
| Browserless | `fetch(Deno.env.get('BROWSERLESS_URL'), { body: JSON.stringify({ code: '...puppeteer code...' }) })` |

## Utility Nodes

| n8n Node | Supabase TypeScript |
|----------|-------------------|
| `crypto` (hash) | `const hash = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(data))` |
| `crypto` (hmac) | `const key = await crypto.subtle.importKey('raw', ...); const sig = await crypto.subtle.sign('HMAC', key, data)` |
| `dateTime` | `new Date().toISOString()` / `new Date().toLocaleString('zh-TW', { timeZone: 'Asia/Taipei' })` |
| `xml` (parse) | Use `npm:fast-xml-parser` |
| `html` (extract) | Use `npm:cheerio` or Deno DOM parser |
| `spreadsheetFile` (CSV) | Use `npm:csv-parse` |
| `itemLists` (sort) | `items.sort((a, b) => a.field - b.field)` |
| `itemLists` (limit) | `items.slice(0, limit)` |
| `itemLists` (remove duplicates) | `[...new Map(items.map(i => [i.key, i])).values()]` |

## Expression Conversion

| n8n Expression | TypeScript Equivalent |
|---------------|----------------------|
| `{{ $json.field }}` | `payload.field` |
| `{{ $json.nested.deep }}` | `payload.nested.deep` |
| `{{ $('Node Name').first().json.field }}` | Variable from that node's step (already in scope) |
| `{{ $env.KEY_NAME }}` | `Deno.env.get('KEY_NAME')` |
| `{{ $now.toISO() }}` | `new Date().toISOString()` |
| `{{ $now.format('yyyy-MM-dd') }}` | `new Date().toISOString().slice(0, 10)` |
| `{{ $now.setZone('Asia/Taipei').toFormat('yyyy-MM-dd HH:mm') }}` | `new Date().toLocaleString('zh-TW', { timeZone: 'Asia/Taipei' })` |
| `{{ $json.items.length }}` | `data.items.length` |
| `{{ $if($json.status === "active", "yes", "no") }}` | `data.status === "active" ? "yes" : "no"` |
| `{{ $evaluateExpression('...') }}` | Direct JS evaluation |
| `{{ $json.field \|\| 'default' }}` | `payload.field ?? 'default'` |

## Prompt Template Expressions

n8n workflows often embed expressions inside prompt strings. Convert these to template literals or a `buildPrompt()` helper:

```typescript
// n8n prompt: "Today is {{ $now.format('yyyy-MM-dd') }}, analyze {{ $json.topic }}"
// Supabase:
const prompt = `Today is ${new Date().toISOString().slice(0, 10)}, analyze ${payload.topic}`
```

For prompts stored in DB with placeholders:
```typescript
function buildPrompt(template: string, vars: Record<string, string>): string {
  let result = template
  for (const [key, value] of Object.entries(vars)) {
    result = result.replaceAll(`{{${key}}}`, value)
  }
  return result
}
```

## Credential → Secret Name Mapping

| n8n Credential Type | Supabase Secret Name | Fields |
|--------------------|---------------------|--------|
| `openAiApi` | `OPENAI_API_KEY` | apiKey |
| `slackApi` / `slackOAuth2Api` | `SLACK_BOT_TOKEN` | accessToken |
| `telegramApi` | `TELEGRAM_BOT_TOKEN` | accessToken |
| `notionApi` | `NOTION_API_KEY` | apiKey |
| `airtableTokenApi` | `AIRTABLE_API_KEY` | accessToken |
| `googleSheetsOAuth2Api` | (Convert to PostgreSQL — no Google auth needed) | — |
| `stripeApi` | `STRIPE_SECRET_KEY` | secretKey |
| `sendGridApi` | `SENDGRID_API_KEY` | apiKey |
| `httpBasicAuth` | `<SERVICE>_USERNAME` + `<SERVICE>_PASSWORD` | username, password |
| `httpHeaderAuth` | `<SERVICE>_API_KEY` | value |
| `lineMessagingApi` | `LINE_CHANNEL_ACCESS_TOKEN` + `LINE_CHANNEL_SECRET` | channelAccessToken, channelSecret |
| `discordWebhookApi` | `DISCORD_WEBHOOK_URL` | webhookUri |
| `perplexityApi` | `PERPLEXITY_API_KEY` | apiKey |
| `anthropicApi` | `ANTHROPIC_API_KEY` | apiKey |
| `googleGeminiApi` | `GEMINI_API_KEY` | apiKey |
