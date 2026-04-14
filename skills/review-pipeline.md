---
name: review-pipeline
description: Run a multi-agent code review on the current branch (or a specified diff range) via the multipov.ai MCP server. Content-aware 8-person panel, consensus-first findings, auto-fix for mechanical issues.
---

# Review Pipeline — Multi-Agent Code Review via multipov.ai

Runs a multi-agent code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server.

Determine the diff range from arguments: `--staged`, `--unstaged`, or a ref range like `main..HEAD` (default).

## Fallback: Review Last Committed Work

Before proceeding, check whether the determined diff range contains changes:

```bash
DIFF_OUTPUT=$(git diff --stat $DIFF_RANGE 2>/dev/null)
```

If `$DIFF_OUTPUT` is empty AND there are no staged or unstaged changes:

1. Fall back to reviewing the last batch of committed work
2. Get the last commit: `git log -1 --format="%H %s"`
3. Get the diff: `git diff HEAD~1...HEAD`
4. If that diff is also empty, report: "Nothing to review — no pending or recent changes." and stop
5. Otherwise, tell the user: "No pending changes found. Reviewing the last committed work instead: [commit hash short] [commit message]"
6. Override the diff range to `HEAD~1...HEAD` and continue

For a smarter batch size, check `git log --oneline -10` for a logical boundary — if the last several commits share a common prefix (e.g., all `feat(admin):`), include them all as `HEAD~N...HEAD`.

---

## What this skill does

This skill calls the multipov.ai MCP server's `submit_pipeline_review` tool. All of the multi-LLM fan-out and consensus synthesis happens server-side against a content-routed 8-person panel (5 technical reviewers + 3 outside perspectives).

- **One MCP call** — no local reviewer subagents, no direct GPT-4o/Gemini/Grok plumbing.
- **Consistent reviewer pool across machines** — same panel shape whether you run from your laptop or your desktop.
- **Shared daily quota** — counts against your multipov review quota (10/day), not your own LLM API keys.
- **Audit trail** — every review lands in `list_my_reviews` with a `persona_selection_method` field so you can see whether routing succeeded or fell back.

Cost: ~$0.26 per review in quick mode, ~$0.98 in deep mode (8 personas × 4 LLMs + routing). Tracked server-side against multipov's global cost cap.

---

## Prerequisites

- The multipov MCP server must be registered in your Claude Code config and loaded in the current session. If it isn't, run `/prep-multipov` first (see [`skills/prep-multipov.md`](./prep-multipov.md) in this repo) — it walks you through the one-time install.
- You must have a multipov API key. Generate one at `https://multipov.ai/settings/api-keys` and store it in your preferred secrets manager.

---

## Process

### Step 1: Verify MCP is reachable

Quick check this session can skip:

```bash
if ! grep -q '"multipov"' ~/.claude.json 2>/dev/null; then
  echo "multipov MCP server not configured — run /prep-multipov first"
  exit 0
fi
```

If the MCP tools aren't available in the session even though the config is present, restart Claude Code.

### Step 2: Capture the diff

```bash
DIFF_RANGE="${1:-main..HEAD}"
DIFF_CONTENT=$(git diff $DIFF_RANGE 2>/dev/null)
DIFF_CHARS=$(printf "%s" "$DIFF_CONTENT" | wc -c | tr -d ' ')
FILES_CHANGED=$(git diff --name-only $DIFF_RANGE 2>/dev/null | head -30)
FILE_COUNT=$(echo "$FILES_CHANGED" | grep -c . || echo 0)

if [ "$DIFF_CHARS" -gt 950000 ]; then
  echo "Diff is ${DIFF_CHARS} chars, approaching the 1M cap."
  echo "Consider narrowing the range or running /review-all on individual modules."
fi

if [ "$DIFF_CHARS" -eq 0 ]; then
  echo "Nothing to review — diff is empty."
  exit 0
fi
```

### Step 3: Build the review body

Prefix the diff with a context block so reviewers know what they're looking at:

```
FILES CHANGED ($FILE_COUNT):
$FILES_CHANGED

DIFF ($DIFF_CHARS chars):
$DIFF_CONTENT
```

### Step 4: Submit via MCP

Call `mcp__multipov__submit_pipeline_review`:

```typescript
await mcp__multipov__submit_pipeline_review({
  content: /* the context-prefixed diff from Step 3 */,
  file_name: `${REPO_NAME}-${BRANCH}.diff`,
  mode: "deep",  // always deep for code reviews — worth the latency
  custom_instructions: /* optional extra framing, e.g., "focus on the auth flow" */,
})
```

The server auto-picks the 8-person panel via content-aware routing. You don't need to pass `personas` unless you want to override — doing so skips the routing that balances technical and outside perspectives.

### Step 5: Poll with a progress bar

Print the initial progress bar immediately after submission using data from the submit response — do NOT wait for the first poll:

```
total = status.selected_personas.length (from submit response)
pct = 0
bar = "░".repeat(total)
Print: [bar] 0/{total} personas — queued — 0s elapsed (estimated ~{estimated_seconds}s)
```

Then start the polling loop. Use `wait_seconds=20` so the server holds the connection until something changes — no client-side sleep needed:

```
Loop (max 30 iterations = 10 min ceiling):
  status = mcp__multipov__get_review_status({ job_id, wait_seconds: 20 })
  if complete → break
  if failed/cancelled → report error and stop

  Render progress bar:
    filled = status.completed_count
    total = status.total_count
    pct = total > 0 ? Math.round(filled / total * 100) : 0
    bar = "█".repeat(filled) + "░".repeat(total - filled)
    Print: [bar] {filled}/{total} personas — {status.phase} — {status.elapsed_seconds}s elapsed
```

Print the progress bar as regular text output (not inside a tool call) so the user sees it update in real time. Each iteration takes ~20s (server holds the connection) so you'll print ~5–6 progress updates for a typical 110s deep-mode review.

Example:
```
[░░░░░] 0/5 personas — queued — 3s elapsed
[██░░░] 2/5 personas — reviewing — 45s elapsed
[████░] 4/5 personas — reviewing — 82s elapsed
[█████] 5/5 personas — synthesizing — 98s elapsed
[█████] 5/5 personas — complete — 114s
```

### Step 6: Fetch the report and present findings

```typescript
const report = await mcp__multipov__get_review_report({ job_id })
// report has: executiveSummary, consensusFindings, disputedFindings,
//             singleModelFindings, personaResults, reviewerRelevance, durationMs
```

Present a consensus-first markdown report:

```markdown
## Review Pipeline Report

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

### Step 7: Fix-first triage

Classify findings as AUTO-FIX vs ASK:

**AUTO-FIX** (implement immediately without asking):
- Dead code removal, stale comments, missing logging, unused imports, magic numbers, obvious typos, missing `await`, simple null checks.

**ASK** (batch into one question):
- Security changes, race conditions, architecture decisions, removing functionality, adding dependencies, performance trade-offs, public API changes.

For AUTO-FIX items: dispatch implementation subagents immediately (one per file, in parallel).
For ASK items: present all in a single summary grouped by file with the recommended fix.

Report: "Auto-fixed {N} mechanical issues. {M} items need your input (see below)."

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

Keep the URL for follow-ups — you can pull it up in the multipov web UI.

---

## Error handling

- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap hit. Rare. `details.reset_at` has the rollover.
- **`not_found`** on poll — probably a stale `job_id`. Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down; the review still runs but with a dumber panel. Quality will be lower than usual. Note it in the report header.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — you lose the 5+3 engineering-vs-outside-perspective split that makes the consensus findings useful.
- **Always use `mode: "deep"`** for code reviews unless you're short on quota. Quick mode is Claude-only and misses the cross-model consensus.
- For design-doc reviews (before code is written), use `/review-plan` instead — it calls `submit_plan_review` which uses plan-focused framing.
- For codebase-wide audits (not just a PR-sized diff), use `/review-all` which calls `submit_codebase_review`.
- For focused lenses, see the sibling skills: `/review-performance`, `/review-security`, `/review-scaling`, `/review-tests`.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings (API keys, quota, billing): `https://multipov.ai/settings`
