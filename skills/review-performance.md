---
name: review-performance
description: Run a performance-focused multi-agent code review via the multipov.ai MCP server. Same panel as /review-pipeline, steered toward user-perceived speed and runtime cost via custom_instructions.
---

# Review Performance — Performance-Focused Code Review via multipov.ai

Runs a performance-focused multi-agent code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server. Same panel shape and input handling as `/review-pipeline`, but steered toward user-perceived speed and runtime cost via `custom_instructions`.

Determine the diff range from arguments: `--staged`, `--unstaged`, or a ref range like `main..HEAD` (default).

## Fallback: Review Last Committed Work

If the determined diff range is empty and there are no staged or unstaged changes, fall back to `git diff HEAD~1...HEAD` and review the last committed work. If that's also empty, report "Nothing to review" and stop.

---

## What this skill does

This skill calls `mcp__multipov__submit_pipeline_review` with **performance-focused `custom_instructions`**. The server appends the framing to the type-specific system prompt AND passes it as the routing hint, so the 8-person panel biases toward reviewers who care about latency, cold-start, allocation pressure, and rendering cost.

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
  file_name: `${REPO_NAME}-${BRANCH}.performance.diff`,
  mode: "deep",  // always deep — perf findings need cross-model consensus
  custom_instructions: `
PERFORMANCE-FOCUSED REVIEW. Frame every finding around user-perceived speed and runtime cost:

- Latency & critical path: time-to-first-byte, time-to-interactive, blocking renders, data-loading waterfalls, sequential fetches that could be parallel
- Cold start & initialization: serverless spin-up, import-time work, lazy vs eager loading, connection setup per request
- Algorithmic complexity: O(n^2) in hot loops, repeated work across iterations, unnecessary sorts, missing memoization
- Allocation pressure: temporary objects in hot paths, string concatenation in tight loops, closure-per-call waste, GC churn
- Rendering cost: unnecessary re-renders, large client components, hydration bloat, layout thrash, image/font weight
- Bundle size: dead code, heavy deps, missing code-splitting, accidental server → client bundling
- Caching: data that rarely changes but is refetched every request, missing HTTP cache headers, cache stampedes
- Database: N+1, missing indexes on hot queries, SELECT * on wide tables, chatty transactions

For each finding, quantify the impact where you can — "saves ~200ms on every page load" beats "might be faster." Be explicit about what to measure before the fix; many performance findings need profiling to confirm they're on the critical path.
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
## Performance Review Report

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

### Step 7: Fix-first triage (with profiling caveat)

Same Fix-first classification as `/review-pipeline`, with one perf-specific caveat: **many performance findings should not be fixed blind.** Without a profiler flame graph or a real user metric, a "perf fix" is a guess — and guessed fixes regress as often as they help.

**AUTO-FIX** (implement immediately without asking):
- Missing `React.memo` on a proven-hot list item, obvious `Promise.all` where serial `await`s exist, unoptimized `<img>` → `<Image>`, trivially parallelizable DB calls, missing indexes on explicitly-hot queries, obvious dead code in hot paths.

**ASK / PROFILE FIRST** (batch into one question):
- Anything where the fix is "rewrite rendering strategy," "move to server components," "introduce caching layer," "swap algorithms," or "denormalize the schema." Get numbers first — what's the current latency, what's the target, what does a profiler say about where the time actually goes.

Report: "Auto-fixed {N} mechanical perf issues. {M} items need profiling + your input (see below)."

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
- **Always use `mode: "deep"`** for performance reviews. Perf findings are high-variance — cross-model consensus is the cheapest filter.
- **Profile before fixing.** User-perceived speed is measured in real browsers on real networks, not in review comments. A finding that says "this is slow" is a hypothesis until a flame graph agrees.
- Use `/review-pipeline` for general code review. Use `/review-performance` when you want the panel's full attention on the speed dimension.
- Pair with `/review-scaling` for a one-two punch: scaling catches what breaks as you grow, performance catches what's slow today.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
