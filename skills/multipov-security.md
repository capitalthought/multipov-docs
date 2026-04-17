Run a multi-agent **security-focused** code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server. Same panel shape and input handling as `/multipov-code`, but steered toward auth, injection, data exposure, and supply-chain concerns via `custom_instructions`.

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

## Architecture (as of 2026-04-13)

This skill **calls the multipov.ai MCP server** — it no longer dispatches 3 local security panel subagents, doesn't shell out to `llm-review.ts`, doesn't fan out to GPT-4o/Gemini/Grok directly. All of that now happens server-side in multipov's `executeReview` against a content-routed 8-person panel (5 technical + 3 outside perspectives).

**Security framing is delivered via `custom_instructions`.** The server's `submit_pipeline_review` tool appends `custom_instructions` to the type-specific framing and the routing hint, so the router biases toward reviewers who care about auth boundaries, injection, secret handling, and supply chain. No server-side `scope` parameter exists today — `custom_instructions` is the right knob until one does.

Benefits over the old local pipeline:
- **One MCP call instead of 3 parallel panel subagents** — faster wall clock, lower complexity.
- **Consistent reviewer pool across machines** — same panel whether you run from your MacBook or Desk.
- **Shared daily quota** — counts against your multipov review quota (10/day), not your local API keys.
- **Audit trail** — every review lands in `list_my_reviews` with a `persona_selection_method` field — useful for compliance.

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
  file_name: `${REPO_NAME}-${BRANCH}.security.diff`,
  mode: "deep",  // always deep — security findings demand cross-model consensus
  custom_instructions: `
SECURITY-FOCUSED REVIEW. Frame every finding around attacker reachability and blast radius:

- AuthN / AuthZ: missing auth checks, broken access control, privilege escalation, IDOR, session fixation, trust-boundary crossings
- Injection: SQL injection, command injection, prompt injection (for any LLM calls), SSTI, XXE, header injection
- XSS / CSRF: unsafe HTML output, missing output encoding, missing CSRF tokens, cookie flags, same-site handling
- SSRF: user-controlled URLs, cloud metadata endpoints (169.254.169.254), allowlist gaps, DNS rebinding
- Secrets: hard-coded secrets, leaked env vars, secrets in logs or error responses, secrets in VCS history
- Input validation & output encoding: missing sanitization, type coercion bugs, path traversal, over-permissive regex
- Data exposure: RLS gaps, PII in logs, over-broad SELECTs, unprotected backups, cross-tenant leaks
- Supply chain: new/updated dependencies, unlocked versions, typosquats, post-install scripts, CVE-flagged packages
- Rate limiting & abuse: unbounded endpoints, missing per-user quotas, cost amplification via AI calls, log spam vectors

For each finding, label: **who can exploit** (unauthenticated / authenticated user / owner / admin) and **blast radius** (single-user / cross-tenant / all-users / financial / data-breach). Be explicit about the attack vector — reviewers should be able to explain the exploit in one sentence.
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
## 🔒 Security Review Report

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

### Step 7: Fix-first triage — with a hard stop on Critical

Security reviews have their own escalation rule on top of the standard Fix-first flow.

**🚨 CRITICAL SECURITY FINDING** (escalate immediately — do not keep processing lower-severity items in the background):
- Auth bypass, RLS gap, secret in repo/logs, unauthenticated write/delete, cross-tenant data leak, confirmed injection, SSRF reaching cloud metadata, a dependency with a known exploitable CVE in the diff.
- **Output:** `🚨 Critical security finding — do not merge until fixed: <one-line summary>`. List each Critical finding with `file:line`, attack vector, and the minimum fix. Stop the pipeline and hand control back to the user before continuing with lower-severity items.

**AUTO-FIX** (implement immediately without asking, for obvious fixes only):
- Missing output encoding on an HTML template, missing `httpOnly` / `secure` / `sameSite` cookie flags, hard-coded test credentials in a non-test file, missing CSRF token on a form, obvious missing `nosniff` / `CSP` header, a regex that's clearly too permissive on an ID field.

**ASK** (batch into one question):
- Anything touching auth flow refactors, RLS policy changes, cryptographic decisions, trust boundary redesign, new dependencies, rate-limiting strategy, logging-policy changes that affect PII.

For Critical: **never auto-fix silently** — the user must see the finding and the fix together. Dispatch an implementation subagent only after explicit confirmation.

For AUTO-FIX items: dispatch implementation subagents immediately (one per file, in parallel).
For ASK items: present all in a single summary grouped by file with the recommended fix.

Report: "🚨 {C} critical security issues — do not merge. 🔧 Auto-fixed {N} mechanical security issues. {M} items need your input (see below)."

### Step 8: Log the review

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
# Keep the URL — security reviews are worth linking from the PR description for audit trail.
```

---

## Error handling

- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap ($100). Rare. `details.reset_at` has the rollover. Not your problem to fix.
- **`not_found`** on poll — probably a stale `job_id`. Did you switch terminals? Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down, review still runs but with a dumber panel. Quality will be lower than usual; note it in the report header AND consider re-running once routing recovers, because security is exactly the domain where reviewer specialization matters most.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch the status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel.
- **Always use `mode: "deep"`** for security reviews. False negatives cost more than false positives in this domain — cross-model consensus catches more than any one model alone.
- **Always link the review URL from the PR description.** That's your audit trail. Josh can pull it up in the web UI later, and so can anyone doing compliance work.
- For AI-integrated code (LLM calls, tool use), the security framing above already covers prompt injection — but if you want *just* prompt-injection work, add a one-liner `custom_instructions` addendum: "Focus exclusively on prompt injection, system-prompt leakage, and tool-call abuse."
- Use `/multipov-code` for general code review. Use `/multipov-security` before any public launch, any auth change, any new AI integration, or any dependency bump.
- **Future improvement:** a dedicated `scope` / `focus` parameter on `submit_pipeline_review` would be cleaner than stuffing framing into `custom_instructions`. File an issue on the multipov repo if this skill starts feeling cramped.

---

## Migration note (2026-04-13)

This skill was rewritten to call the multipov.ai MCP server instead of dispatching 3 local security panel subagents. It migrated alongside `/multipov-code`, `/multipov-plan`, `/multipov-all`, `/multipov-scaling`, and `/multipov-performance` on the same day. Same rationale as the pipeline migration:
1. A consistent reviewer pool across machines
2. Cost observability via multipov's shared daily cap
3. An audit trail in `list_my_reviews` — especially valuable for security work
4. Fewer moving parts (one MCP call vs 3 parallel panel subagents + 12 LLM calls)

The focused framing (scaling / performance / security) is currently delivered via the shared `custom_instructions` parameter on `submit_pipeline_review` — there is no dedicated `scope` parameter on the MCP tool yet. If that changes, update this skill to pass `scope: "security"` directly.

**One-time setup:** if you don't have a multipov API key yet, generate one at `https://multipov.ai/settings/api-keys`, save it to 1Password as "multipov.ai — MCP API Key (Claude Code)" in the Employee vault, then run `/prep-multipov` to register it in `~/.claude.json`. Setup is documented at `https://github.com/capitalthought/multipov/blob/main/docs/mcp/README.md`.

See `docs/mcp/README.md` in the multipov repo for the full tool catalog.
