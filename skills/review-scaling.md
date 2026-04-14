---
name: review-scaling
description: Run a scaling-focused multi-agent code review via the multipov.ai MCP server. Same panel as /review-pipeline, steered toward growth bottlenecks (throughput, memory, DB pooling, capacity) via custom_instructions.
---

# Review Scaling — Scaling-Focused Code Review via multipov.ai

Runs a scaling-focused multi-agent code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server. Same panel shape and input handling as `/review-pipeline`, but steered toward scaling concerns via `custom_instructions`.

Determine the diff range from arguments: `--staged`, `--unstaged`, or a ref range like `main..HEAD` (default).

## Fallback: Review Last Committed Work

If the determined diff range is empty and there are no staged or unstaged changes, fall back to `git diff HEAD~1...HEAD` and review the last committed work. If that's also empty, report "Nothing to review" and stop.

---

## What this skill does

This skill calls `mcp__multipov__submit_pipeline_review` with **scaling-focused `custom_instructions`**. The server appends the framing to the type-specific system prompt AND passes it as the routing hint, so the 8-person panel biases toward reviewers who care about throughput, memory, DB connection pooling, and capacity planning.

Cost: ~$0.26 quick, ~$0.98 deep.

---

## Prerequisites

- The multipov MCP server must be registered and loaded. If it isn't, run `/prep-multipov` first (see [`skills/prep-multipov.md`](./prep-multipov.md) in this repo).
- You must have a multipov API key. Generate one at `https://multipov.ai/settings/api-keys`.

---

## Process

### Step 1: Verify MCP is reachable

```bash
if ! grep -q '"multipov"' ~/.claude.json 2>/dev/null; then
  echo "multipov MCP server not configured — run /prep-multipov first"
  exit 0
fi
```

### Step 2: Capture the diff

```bash
DIFF_RANGE="${1:-main..HEAD}"
DIFF_CONTENT=$(git diff $DIFF_RANGE 2>/dev/null)
DIFF_CHARS=$(printf "%s" "$DIFF_CONTENT" | wc -c | tr -d ' ')
FILES_CHANGED=$(git diff --name-only $DIFF_RANGE 2>/dev/null | head -30)
FILE_COUNT=$(echo "$FILES_CHANGED" | grep -c . || echo 0)

if [ "$DIFF_CHARS" -gt 950000 ]; then
  echo "Diff is ${DIFF_CHARS} chars, approaching the 1M cap."
fi

if [ "$DIFF_CHARS" -eq 0 ]; then
  echo "Nothing to review — diff is empty."
  exit 0
fi
```

### Step 3: Build the review body

```
FILES CHANGED ($FILE_COUNT):
$FILES_CHANGED

DIFF ($DIFF_CHARS chars):
$DIFF_CONTENT
```

### Step 4: Submit via MCP

```typescript
await mcp__multipov__submit_pipeline_review({
  content: /* the context-prefixed diff from Step 3 */,
  file_name: `${REPO_NAME}-${BRANCH}.scaling.diff`,
  mode: "deep",  // always deep — scaling findings need cross-model consensus
  custom_instructions: `
SCALING-FOCUSED REVIEW. Frame every finding around how this code behaves under growth:

- Throughput & concurrency: hot paths, lock contention, goroutine/thread explosion, work-queue depth
- Memory pressure: unbounded caches, allocations per request, GC churn, leak-shaped patterns
- Database: connection-pool exhaustion, N+1 queries, missing indexes, table scans, RLS-at-scale cost, long transactions blocking writers
- Cache strategy: cache stampedes, invalidation gaps, TTL correctness, hot-key problems
- Capacity planning: what breaks at 10x users? 100x data? Any fixed-size buffers or O(n^2) loops?
- Backpressure: what happens when downstream slows down? Do retries amplify load?
- Operational: does this change introduce a new single point of failure? Any new per-request expensive setup?

For each finding, call out the **scale trigger** — at what load does this become a real problem? Feel free to recommend benchmarks before fixing; many scaling issues are only worth the work once you have numbers.
`.trim(),
})
```

### Step 5: Poll with a progress bar

Use `wait_seconds=20` so the server holds the connection until something changes. Render a progress bar between calls and print it as regular text output (not inside a tool call) so the user sees it update in real time. Max 30 iterations (~10 min ceiling).

### Step 6: Fetch the report and present findings

```typescript
const report = await mcp__multipov__get_review_report({ job_id })
```

```markdown
## Scaling Review Report

**Range:** `${DIFF_RANGE}` · **Files:** ${FILE_COUNT} · **Chars:** ${DIFF_CHARS}
**Panel:** ${report.personaResults.length} reviewers (selection: ${persona_selection_method})
**Duration:** ${report.durationMs / 1000}s

### Executive summary
${report.executiveSummary}

### Consensus findings (flagged by 2+ models)
${report.consensusFindings.map(f => formatFinding(f)).join("\n")}

### Disputed findings (2+ models agree, but not all)
${report.disputedFindings.map(f => formatFinding(f)).join("\n")}

### Single-model findings (verify before acting)
${report.singleModelFindings.map(f => formatFinding(f)).join("\n")}
```

### Step 7: Fix-first triage (with benchmark caveat)

Same Fix-first classification as `/review-pipeline`, with one scaling-specific caveat: **many scaling findings should not be fixed blind.** If the finding is "this might be an N+1 at 10K users," you want a benchmark first, not a speculative rewrite.

**AUTO-FIX** (implement immediately without asking):
- Missing index on an obviously-scanned column, unbounded `SELECT *`, obvious N+1 inside a small loop, missing `LIMIT` on a query over a growing table, wrong connection-pool size for the runtime, dead caches, obvious missing `await`.

**ASK / BENCHMARK FIRST** (batch into one question):
- Anything where the fix is "rewrite the hot path," "introduce a queue," "shard the table," "add a cache layer," or "rethink the data model." These cost real time and regress easily — get numbers before committing.

Report: "Auto-fixed {N} mechanical scaling issues. {M} items need benchmarks + your input (see below)."

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

---

## Error handling

Same as `/review-pipeline`: `rate_limited`, `cost_cap_reached`, `not_found`, `fallback_alphabetical`, 10-minute timeout.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel.
- **Always use `mode: "deep"`** for scaling reviews. Scaling findings are expensive if wrong — cross-model consensus is the cheapest way to separate signal from noise.
- **Benchmark before fixing.** Scaling review is pattern-matching; real numbers are ground truth. A finding that says "this could be slow at 10K users" is a hypothesis, not a bug.
- Use `/review-pipeline` for general code review. Use `/review-scaling` when you specifically want the panel's attention on growth bottlenecks.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
