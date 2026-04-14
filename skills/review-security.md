---
name: review-security
description: Run a security-focused multi-agent code review via the multipov.ai MCP server. Same panel as /review-pipeline, steered toward auth, injection, data exposure, and supply-chain concerns via custom_instructions.
---

# Review Security — Security-Focused Code Review via multipov.ai

Runs a security-focused multi-agent code review on `$ARGUMENTS` (default: current branch changes) via the multipov.ai MCP server. Same panel shape and input handling as `/review-pipeline`, but steered toward auth, injection, data exposure, and supply-chain concerns via `custom_instructions`.

Determine the diff range from arguments: `--staged`, `--unstaged`, or a ref range like `main..HEAD` (default).

## Fallback: Review Last Committed Work

If the determined diff range is empty and there are no staged or unstaged changes, fall back to `git diff HEAD~1...HEAD` and review the last committed work. If that's also empty, report "Nothing to review" and stop.

---

## What this skill does

This skill calls `mcp__multipov__submit_pipeline_review` with **security-focused `custom_instructions`**. The server appends the framing to the type-specific system prompt AND passes it as the routing hint, so the 8-person panel naturally biases toward reviewers who care about auth boundaries, injection, secret handling, and supply chain.

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

### Step 5: Poll with a progress bar

Use `wait_seconds=20` so the server holds the connection until something changes:

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

Print the progress bar as regular text output (not inside a tool call) so the user sees it update in real time.

### Step 6: Fetch the report and present findings

```typescript
const report = await mcp__multipov__get_review_report({ job_id })
```

```markdown
## Security Review Report

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

### Step 7: Fix-first triage — with a hard stop on Critical

Security reviews have their own escalation rule on top of the standard Fix-first flow.

**CRITICAL SECURITY FINDING** (escalate immediately — do not keep processing lower-severity items in the background):
- Auth bypass, RLS gap, secret in repo/logs, unauthenticated write/delete, cross-tenant data leak, confirmed injection, SSRF reaching cloud metadata, a dependency with a known exploitable CVE in the diff.
- **Output:** `Critical security finding — do not merge until fixed: <one-line summary>`. List each Critical finding with `file:line`, attack vector, and the minimum fix. Stop the pipeline and hand control back to the user before continuing with lower-severity items.

**AUTO-FIX** (implement immediately without asking, for obvious fixes only):
- Missing output encoding on an HTML template, missing `httpOnly` / `secure` / `sameSite` cookie flags, hard-coded test credentials in a non-test file, missing CSRF token on a form, obvious missing `nosniff` / `CSP` header, a regex that's clearly too permissive on an ID field.

**ASK** (batch into one question):
- Anything touching auth flow refactors, RLS policy changes, cryptographic decisions, trust boundary redesign, new dependencies, rate-limiting strategy, logging-policy changes that affect PII.

For Critical: **never auto-fix silently** — the user must see the finding and the fix together. Dispatch an implementation subagent only after explicit confirmation.

Report: "{C} critical security issues — do not merge. Auto-fixed {N} mechanical security issues. {M} items need your input (see below)."

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

Link the review URL from the PR description — that's your audit trail.

---

## Error handling

- **`rate_limited`**, **`cost_cap_reached`**, **`not_found`**, **`persona_selection_method = "fallback_alphabetical"`** — same as `/review-pipeline`. For security, `fallback_alphabetical` is worth rerunning once routing recovers, because security is exactly the domain where reviewer specialization matters most.
- **Timeout after 10 min** — come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel.
- **Always use `mode: "deep"`** for security reviews. False negatives cost more than false positives — cross-model consensus catches more than any one model alone.
- **Always link the review URL from the PR description.** That's your audit trail for compliance work.
- For AI-integrated code (LLM calls, tool use), the framing above already covers prompt injection. If you want *just* prompt-injection work, add a one-liner `custom_instructions` addendum: "Focus exclusively on prompt injection, system-prompt leakage, and tool-call abuse."
- Use `/review-pipeline` for general code review. Use `/review-security` before any public launch, any auth change, any new AI integration, or any dependency bump.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
