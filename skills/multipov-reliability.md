Run a multi-agent **reliability-focused** code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server. Same panel shape and input handling as `/multipov-code`, but steered toward uptime, maintainability, logging, observability, and failure recovery via `custom_instructions`.

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

---

## Architecture (as of 2026-04-14)

This skill **calls the multipov.ai MCP server** — server-side content-aware routing picks an 8-person panel (5 technical + 3 outside perspectives). No local fan-out, no direct GPT-4o/Gemini/Grok calls.

**Reliability framing is delivered via `custom_instructions`.** The router biases toward reviewers who care about failure modes, retry behavior, logging coverage, alerting, dependency failure handling, maintainability, and operational debuggability. No server-side `scope` parameter exists today — `custom_instructions` is the right knob.

Benefits over a local pipeline:
- **One MCP call instead of 3 parallel panel subagents** — faster wall clock, lower complexity.
- **Consistent reviewer pool across machines** — same panel whether you run from your MacBook or Desk.
- **Shared daily quota** — counts against your multipov review quota (10/day), not your local API keys.
- **Audit trail** — every review lands in `list_my_reviews` with a `persona_selection_method` field.

Cost: ~$0.26 per review in quick mode, ~$0.98 in deep mode (8 personas × 4 LLMs + routing). Under the $100/day global cap.

---

## Process

### Step 1: Verify MCP is reachable

Run `/prep-multipov` first if this is the first time in the session, or if `multipov` isn't already in `~/.claude.json` mcpServers. If you don't have a multipov API key yet, `/prep-multipov` walks you through the one-time setup.

Quick check this session can skip:

```bash
if ! grep -q '"multipov"' ~/.claude.json 2>/dev/null; then
  echo "⚠️ multipov MCP server not configured — run /prep-multipov first"
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
  echo "⚠️ Diff is ${DIFF_CHARS} chars, approaching the 1M cap."
  echo "Consider narrowing the range or running /multipov-all on individual files."
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

Use the MCP tool `mcp__multipov__submit_pipeline_review`. If the tool isn't available in the session, the MCP config isn't loaded — re-run `/prep-multipov`.

Tool call shape:

```typescript
await mcp__multipov__submit_pipeline_review({
  content: /* the context-prefixed diff from Step 3 */,
  file_name: `${REPO_NAME}-${BRANCH}.reliability.diff`,
  mode: "deep",  // always deep — reliability findings span many failure modes and benefit from cross-model consensus
  custom_instructions: `
RELIABILITY-FOCUSED REVIEW. Frame every finding around uptime, maintainability, and operational debuggability when things go wrong at 3am:

- Failure modes: what happens when this network call times out, this dependency returns 500, this DB is read-only, this file is missing, this env var is unset, this queue backs up? Is the failure contained or does it cascade?
- Retry & idempotency: retries on non-idempotent operations (duplicate charges, duplicate emails), missing exponential backoff, no jitter (thundering herd), infinite retry loops, retries that mask real bugs
- Timeouts & deadlines: calls with no timeout, timeouts longer than the caller's timeout, no circuit breaker when a downstream is failing, request budgets not enforced end-to-end
- Error handling: swallowed exceptions (\`catch {}\`), errors logged but not acted on, throwing through async boundaries, losing stack traces, generic error messages that hide the root cause
- Logging & observability: missing logs at decision points, logs without correlation IDs / request IDs, PII in logs, structured vs unstructured logs, log levels used inconsistently, no way to trace a request through the system
- Metrics & alerting: silent failures (nothing counted), missing SLI/SLO-aligned metrics, alerts on symptoms instead of causes, alerts that will page for known-bad conditions
- Health checks: /health endpoints that only return 200 regardless of real state, no dependency checks, liveness vs readiness confusion, cron jobs with no dead-man switch
- Data integrity: partial writes that can leave inconsistent state, no transactions around multi-step updates, missing constraints, migrations that can't be rolled back, dual-writes without reconciliation
- State & concurrency: race conditions, shared mutable state without locks, stale caches, out-of-order message processing, assumptions about single-instance deployment
- Configuration & secrets: hard-coded values that should be env vars, secrets in code or logs, config drift between environments, missing defaults for non-prod
- Dependency management: unpinned versions that can break overnight, transitive deps with known CVEs, no lockfile, single point of failure on an unmaintained package
- Deployment safety: no gradual rollout, no feature flags for risky changes, no rollback plan, migrations tied to code deploys, breaking API changes without versioning
- Maintainability: code that's hard to debug at 3am, magic numbers without comments, deep call chains that obscure control flow, mutable globals, naming that misleads future readers
- Operational hygiene: no runbook for common failure modes, alerts that page without context, missing dashboards, logs that are only useful once you already know the bug

For each finding, name the failure scenario concretely — "what breaks, when, and how it's detected" beats "this could fail." Be explicit about which observability signal (log line, metric, alert, trace) would surface the issue in production. Reliability bugs hide until they fire; the whole point of this review is to make them cheaper to diagnose when they do.
`.trim(),
})
```

The server auto-picks the 8-person panel via content-aware routing and biases toward reviewers who care about the above when the custom framing is in play. You don't need to pass `personas` unless you want to override.

### Step 4.5: Show the selected panel

Before the progress bar starts, print the panel so the user sees who's about to review their work. The submit response returns `selected_personas` — each entry has `id`, `display_name`, `category`, and a `rationale` explaining why the router picked them. Print this once as regular text output (not inside a tool call), then move on to polling.

Format:

```
🎭 Panel: {selected_personas.length} reviewers — selection: {persona_selection_method}

Technical ({N})
  • {display_name} ({id}) — {rationale, trimmed to ~100 chars}
  …

Outside perspectives ({M})
  • {display_name} ({id}) — {rationale, trimmed to ~100 chars}
  …
```

Group by `category` — technical pool first, outside-perspective pool second. Trim long rationales so each row fits on one line. If `persona_selection_method = "fallback_alphabetical"`, call it out in the header — routing was down, the panel is alphabetical rather than content-aware, and quality will be lower than usual.

### Step 5: Poll with progress bar (server-side long-poll)

**Colors + format:** Apply the recipe from `~/icloud/Claude/progress-bar-spec.md` — terminal-native 16-color ANSI (`\033[34m` blue, `\033[90m` gray, `\033[97m` bright white, `\033[33m` yellow), time rendered as `Xm Ys` via the `formatElapsed()` helper. The runtime output MUST include literal `\033[...m` escapes around each segment. Do not print unstyled examples — the plain-text ones below are for readability only.

Use `wait_seconds=20` so the server holds the connection until something changes — no client-side sleep needed. Call in a tight loop, render a progress bar between calls:

```
Loop (max 30 iterations = 10 min ceiling):
  status = mcp__multipov__get_review_status({ job_id, wait_seconds: 20 })
  if complete → break
  if failed/cancelled → report error and stop

  Render progress bar:
    filled = status.completed_count
    total = status.total_count
    bar = "█".repeat(filled) + "░".repeat(total - filled)
    Print: [bar] {filled}/{total} personas — {status.phase} — {status.elapsed_seconds}s elapsed
```

**Print the progress bar as regular text output** (not inside a tool call) so the user sees it update in real-time. Each iteration takes ~20s (server holds the connection) so you'll print ~5-6 progress updates for a typical 110s deep-mode review.

Example output:
```
[░░░░░] 0/5 personas — queued — 3s elapsed
[██░░░] 2/5 personas — reviewing — 45s elapsed
[████░] 4/5 personas — reviewing — 82s elapsed
[█████] 5/5 personas — synthesizing — 98s elapsed
[█████] 5/5 personas — complete — 114s
```

### Step 6: Fetch report + present findings

```typescript
const report = await mcp__multipov__get_review_report({ job_id })
```

Present a consensus-first markdown report:

```markdown
## 🛡️ Reliability Review Report

**Range:** `${DIFF_RANGE}` · **Files:** ${FILE_COUNT} · **Chars:** ${DIFF_CHARS}
**Panel:** ${report.personaResults.length} reviewers (selection: ${persona_selection_method})
**Duration:** ${report.durationMs / 1000}s

### Executive summary
${report.executiveSummary}

### 🔴 Consensus findings (flagged by 2+ models)
${report.consensusFindings.map(f => formatFinding(f)).join("\n")}

### 🟡 Disputed findings (2+ models agree, but not all)
${report.disputedFindings.map(f => formatFinding(f)).join("\n")}

### 🟢 Single-model findings (verify before acting)
${report.singleModelFindings.map(f => formatFinding(f)).join("\n")}
```

### Step 7: Fix-first triage (with operational caveat)

Same Fix-first classification as `/multipov-code`, with one reliability-specific caveat: **some reliability findings are low-risk to auto-fix, but many require operational judgment you don't have from the diff alone.**

**AUTO-FIX** (implement immediately without asking):
- Add a log line at an obvious decision point, add a `try/catch` around a call that can clearly throw, add a timeout to a `fetch` that has none, add a correlation ID to a log statement that's missing one, pin an unpinned dependency version, add a missing `/health` dependency check, replace `catch {}` with `catch (err) { logger.error(...) }`.

**ASK / DISCUSS FIRST** (batch into one question):
- Anything that changes retry semantics (risk of duplicate side effects), alerting thresholds (risk of pager fatigue or silent outages), circuit breaker logic (risk of false-positive trip), migration rollback strategy, feature flag rollout plans, or introduces new observability infra. Get the failure scenario and the detection story straight before changing runtime behavior.

For AUTO-FIX items: dispatch implementation subagents immediately (one per file, in parallel).
For ASK items: present all in a single summary grouped by file with the recommended fix **and** a note on the failure scenario / detection signal it's protecting against.

Report: "🔧 Auto-fixed {N} mechanical reliability issues. {M} items need operational review (see below)."

### Step 8: Log the review

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

---

## Error handling

- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap ($100). Rare. `details.reset_at` has the rollover. Not your problem to fix.
- **`not_found`** on poll — probably a stale `job_id`. Did you switch terminals? Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down, review still runs but with a dumber panel. Quality will be lower than usual; note it in the report header.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch the status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel.
- **Always use `mode: "deep"`** for reliability reviews. Failure-mode coverage is high-variance across models — cross-model consensus is the cheapest filter.
- **Name the failure.** A reliability finding without a concrete failure scenario ("what breaks, when, and how it's detected") is just a style opinion. If a finding can't answer those three questions, it's probably not actually a reliability issue.
- Use `/multipov-code` for general code review. Use `/multipov-reliability` when you want the panel's full attention on what happens when things go wrong.
- Pair with `/multipov-scaling` (breaks under growth), `/multipov-performance` (slow today), and `/multipov-security` (breaks under attack) for the full operational picture.
- **Future improvement:** a dedicated `scope` / `focus` parameter on `submit_pipeline_review` would be cleaner than stuffing framing into `custom_instructions`. File an issue on the multipov repo if this skill starts feeling cramped.

---

## Companion skills

- `/multipov-performance` — user-perceived speed and runtime cost
- `/multipov-scaling` — what breaks as load grows
- `/multipov-security` — what breaks under attack
- `/multipov-tests` — test coverage and quality
- `/multipov-code` — general-purpose code review

Reliability overlaps with all of these but focuses specifically on the gap between "this works in happy-path prod" and "this is debuggable when it doesn't."

**One-time setup:** if you don't have a multipov API key yet, generate one at `https://multipov.ai/settings/api-keys`, save it to 1Password as "multipov.ai — MCP API Key (Claude Code)" in the Employee vault, then run `/prep-multipov` to register it in `~/.claude.json`. Setup is documented at `https://github.com/capitalthought/multipov/blob/main/docs/mcp/README.md`.

See `docs/mcp/README.md` in the multipov repo for the full tool catalog.
