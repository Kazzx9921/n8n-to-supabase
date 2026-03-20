# Supabase Edge Function Limits & Feasibility Assessment

## Resource Limits by Plan

| Resource | Free | Pro | Team/Enterprise |
|----------|------|-----|-----------------|
| **Wall Time** (request start → end) | 150s | 400s | 400s |
| **CPU Time** (actual computation) | 50ms | 2,000ms (2s) | 2,000ms |
| **Memory** | 150 MB | 256 MB | 256 MB |
| **Request Payload** | 1 MB | 6 MB | 6 MB |
| **Response Size** | 1 MB | 6 MB | 6 MB |
| **Deployed Functions** | 10 | 100 | 500 |

## Understanding Wall Time vs CPU Time

- **Wall Time** = total elapsed time from request received to response sent. Includes all `await` pauses (network I/O, database queries, `fetch()` calls, `setTimeout`).
- **CPU Time** = only the time the CPU is actively computing. Does NOT include time spent waiting for I/O.

Most n8n workflows are **I/O-bound** — they spend the vast majority of time waiting for API responses and database queries. A workflow that takes 30 seconds wall time might only use 50-100ms of CPU time.

## Feasibility Matrix

### Safe (No changes needed)

| Pattern | Wall Time | CPU Time | Notes |
|---------|-----------|----------|-------|
| Webhook → write to DB | < 1s | < 10ms | Simplest case |
| Webhook → 1-2 API calls → DB | 1-5s | < 30ms | Most common pattern |
| Webhook → 3-5 API calls → DB | 3-15s | < 50ms | Still very comfortable |
| Cron → read 100 rows → process → write | 5-30s | 50-200ms | Typical batch job |
| Webhook → AI chat completion → DB | 5-30s | < 50ms | AI response time varies |
| Webhook → AI with 1-2 tool calls → DB | 10-60s | < 100ms | Multiple AI roundtrips |

### Caution (May need optimization)

| Pattern | Wall Time | CPU Time | Risk | Mitigation |
|---------|-----------|----------|------|------------|
| Cron → process 500+ rows sequentially | 60-150s | 200-500ms | Wall time on Free | Use `Promise.all()` for parallel, or batch smaller |
| AI with 3+ tool calls + web scraping | 60-120s | 100-300ms | Wall time on Free | Optimize prompts to reduce tool rounds |
| Sequential 10+ API calls | 30-120s | < 100ms | Wall time on Free | Parallelize with `Promise.all()` |
| Heavy JSON parsing (>1MB) | < 5s | 500ms-1s | CPU time on Free | Use streaming if possible |

### Unsafe (Requires restructuring)

| Pattern | Problem | Solution |
|---------|---------|----------|
| Wait/Delay node (>2 min) | Exceeds wall time | Use `pending_tasks` table + pg_cron |
| Process 1000+ items sequentially | Exceeds wall time | Paginate: process 100 per invocation with pg_cron |
| Payload >6MB (Pro) or >1MB (Free) | Exceeds payload limit | Upload to Storage first, pass URL |
| Long polling / keep-alive | Exceeds wall time | Convert to pg_cron periodic check |
| CPU-intensive computation (image processing, ML) | Exceeds CPU time | Offload to external service |
| Chained executeWorkflow (3+ deep) | Combined wall time | Each function has independent limits, but total latency adds up |

## Optimization Techniques

### Parallelize API calls
```typescript
// BAD: Sequential (30s wall time for 10 calls × 3s each)
for (const item of items) {
  await fetch(apiUrl, { body: JSON.stringify(item) })
}

// GOOD: Parallel (3s wall time for 10 calls × 3s each)
await Promise.all(items.map(item =>
  fetch(apiUrl, { body: JSON.stringify(item) })
))

// BETTER: Parallel with concurrency limit (avoid rate limits)
const BATCH_SIZE = 5
for (let i = 0; i < items.length; i += BATCH_SIZE) {
  await Promise.all(items.slice(i, i + BATCH_SIZE).map(item =>
    fetch(apiUrl, { body: JSON.stringify(item) })
  ))
}
```

### Paginated processing with cursor
```typescript
// Process in batches across multiple cron invocations
const { data: state } = await db
  .from("n8n_workflow_state")
  .select("state")
  .eq("workflow_name", "batch-processor")
  .single()

const cursor = state?.state?.cursor ?? 0

const { data: batch } = await db
  .from("large_table")
  .select("*")
  .gt("id", cursor)
  .order("id")
  .limit(100)  // Only 100 per invocation

// Process batch...

// Save cursor for next run
await db.from("n8n_workflow_state").upsert({
  workflow_name: "batch-processor",
  state: { cursor: batch[batch.length - 1].id },
})
```

### Large file handling via Storage
```typescript
// Instead of passing large data in request body:
// 1. Upload to Storage
const { data } = await db.storage
  .from("workflow-files")
  .upload(`temp/${crypto.randomUUID()}.json`, JSON.stringify(largeData))

// 2. Pass storage path to another function
await fetch(otherFunctionUrl, {
  body: JSON.stringify({ storagePath: data.path })
})

// 3. Other function downloads and processes
const { data: fileData } = await db.storage
  .from("workflow-files")
  .download(storagePath)
```

## Assessment Checklist

When analyzing an n8n workflow, check each node against these questions:

1. **Total API calls** — Count all HTTP Request / integration nodes. If > 5 sequential, consider parallelizing.
2. **Data volume** — How many rows/items flow through? If > 500, consider pagination.
3. **AI rounds** — How many LLM calls (including tool use loops)? Each call adds 5-30s wall time.
4. **Wait/Delay** — Any Wait nodes? Must convert to queue pattern.
5. **File size** — Any binary data or large payloads? Check against limits.
6. **Frequency** — How often does it run? High-frequency cron (< 1 min) needs pg_cron minimum (1 min).

If everything checks out, flag as ✅. If anything needs adjustment, flag as ⚠️ with the specific mitigation strategy.
