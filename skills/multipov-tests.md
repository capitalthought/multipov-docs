Run a multi-agent **test-focused** review on `$ARGUMENTS` (default: test files touched in the current branch) via the multipov.ai MCP server.

This is the **test-quality** sibling of `/multipov-code`. Use it to pressure-test your tests — not the production code. It answers: are there untested paths? Do the tests actually assert behavior? Are edge cases covered? Is mocking masking bugs? Are integration tests hitting real systems? Is coverage lopsided (1000 lines of happy-path, 10 lines of error-handling)?

---

## Architecture (as of 2026-04-13)

This skill **calls the multipov.ai MCP server** — specifically `submit_pipeline_review` with **test-focused `custom_instructions`**. There is no dedicated `submit_test_review` tool on the server (and we deliberately didn't add one — the routing + framing lever is `custom_instructions`, and the code-review input shape doesn't accept a `scope`/`focus` parameter, so client-side framing is the only lever anyway). Content-aware routing then naturally picks a test-heavy panel: `test-reviewer`, `staff-reviewer`, `security-reviewer`, and a few outside perspectives.

Benefits over a local test-review pipeline:
- **One MCP call instead of N parallel subagents** — faster wall clock.
- **Consistent test-reviewer pool across machines** — same panel on your MacBook or Desk.
- **Shared daily quota** — counts against your multipov review quota (10/day), not your local API keys.
- **Audit trail** — every review lands in `list_my_reviews`.

Cost: same as `/multipov-code` — ~$0.26 quick, ~$0.98 deep. Under the $100/day global cap.

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

`/multipov-tests` reviews **tests**, so the default scope is test files touched in the current branch. If the user passed a glob or path as `$ARGUMENTS` (e.g. `src/mcp/` or `src/__tests__/*.ts`), use that instead.

```bash
SCOPE="${1:-}"

if [ -n "$SCOPE" ]; then
  # User passed an explicit path or glob — diff that scope against main.
  FILES_CHANGED=$(git diff --name-only main..HEAD -- $SCOPE 2>/dev/null)
  DIFF_CONTENT=$(git diff main..HEAD -- $SCOPE 2>/dev/null)
  REVIEW_MODE_NOTE="explicit scope: $SCOPE"
else
  # Default: test files touched in the current branch.
  TEST_FILES=$(git diff --name-only main..HEAD 2>/dev/null | grep -E 'test|spec' || true)

  if [ -n "$TEST_FILES" ]; then
    FILES_CHANGED="$TEST_FILES"
    DIFF_CONTENT=$(git diff main..HEAD -- $TEST_FILES 2>/dev/null)
    REVIEW_MODE_NOTE="test files in main..HEAD"
  else
    # No test files touched. Fall back to the full branch diff and flip
    # framing so reviewers evaluate PRODUCTION CODE for missing test coverage
    # instead of reviewing the tests themselves.
    FILES_CHANGED=$(git diff --name-only main..HEAD 2>/dev/null)
    DIFF_CONTENT=$(git diff main..HEAD 2>/dev/null)
    REVIEW_MODE_NOTE="no test files touched in main..HEAD — reviewing production code for coverage gaps"
  fi
fi

DIFF_CHARS=$(printf "%s" "$DIFF_CONTENT" | wc -c | tr -d ' ')
FILE_COUNT=$(echo "$FILES_CHANGED" | grep -c . || echo 0)

if [ "$DIFF_CHARS" -gt 950000 ]; then
  echo "⚠️ Diff is ${DIFF_CHARS} chars, approaching the 1M cap."
  echo "Consider narrowing with an explicit scope: /multipov-tests src/foo/"
fi

if [ "$DIFF_CHARS" -eq 0 ]; then
  echo "Nothing to review — no test or production changes in main..HEAD."
  exit 0
fi
```

If you can infer the production files that the touched test files cover (e.g. `src/__tests__/foo.test.ts` → `src/foo.ts`), list them as "covered production files" in the context block so reviewers can cross-reference.

### Step 3: Build the review body

Prefix the diff with a framing block so reviewers know the target is **test quality**, not production quality:

```
REVIEW TARGET: test coverage and test quality
MODE: ${REVIEW_MODE_NOTE}

FILES CHANGED (${FILE_COUNT}):
${FILES_CHANGED}

COVERED PRODUCTION FILES (inferred, if any):
${COVERED_PRODUCTION_FILES}

DIFF (${DIFF_CHARS} chars):
${DIFF_CONTENT}
```

### Step 4: Submit via MCP

Use `mcp__multipov__submit_pipeline_review` (same tool `/multipov-code` uses — there is no dedicated test-review tool, and the shared code-review input shape doesn't accept a `scope`/`focus` parameter). The lever is `custom_instructions`.

Tool call shape:

```typescript
await mcp__multipov__submit_pipeline_review({
  content: /* the context-prefixed diff from Step 3 */,
  file_name: `${REPO_NAME}-${BRANCH}.tests.diff`,
  mode: "deep",  // always deep — test-quality findings are nuanced
  custom_instructions: `
Review target: TEST COVERAGE AND TEST QUALITY. This is a test-focused review, not a general code review. Do NOT flag production code quality issues unless they directly impact testability.

Reviewers should look for:
- Missing assertions — tests that exercise code but don't assert behavior
- Uncovered code paths — branches, error handlers, early returns, and guard clauses with no test
- Missing edge cases — empty inputs, null, boundary values, off-by-one, unicode, concurrency races, error paths
- Mocking that masks real bugs — over-mocked dependencies, mocks that drift from the real implementation, stubs that always return happy-path values
- Integration-vs-unit balance — integration tests MUST hit a real database (not mocks); flag any integration test that mocks the DB or network layer it's supposedly integrating with
- Flaky risks — time-dependent tests, order-dependent tests, shared global state, brittle selectors, hard-coded waits/timeouts, network calls in unit tests
- Coverage disproportion — 1000 lines of happy-path assertions paired with 10 lines of error-handling coverage
- Missing test types — smoke tests, e2e, property-based, fuzz, regression tests for past bugs
- Weak assertions — \`expect(x).toBeTruthy()\` where a specific value should be asserted; snapshot tests with no semantic check
- Fragile tests — tests that break on any unrelated refactor, CSS selectors by class, DOM structure assumptions

For each finding, specify file:line AND whether it's a MISSING TEST, WEAK TEST, FLAKY RISK, or NIT. Also call out any UNCOVERED CODE PATHS in the production files (file:line of the uncovered branch, even if it's not in the diff).

Do NOT flag production code for refactoring, style, naming, or architecture — that's /multipov-code's job.
`.trim(),
})
```

The server auto-picks the 8-person panel via content-aware routing. With test-focused framing in `custom_instructions` the router naturally weights toward `test-reviewer` + `staff-reviewer` + `security-reviewer` + a couple of outside perspectives. Don't pass `personas` — let the routing do its thing.

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

Present a test-focused consensus-first markdown report. Header says "Test Review Report" and the report adds an **Uncovered code paths** section at the bottom — reviewers are explicitly asked to flag coverage gaps with file:line, even for branches not in the diff.

```markdown
## 🧪 Test Review Report

**Scope:** `${REVIEW_MODE_NOTE}` · **Files:** ${FILE_COUNT} · **Chars:** ${DIFF_CHARS}
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

### 🧭 Uncovered code paths
Files or branches the reviewers flagged as untested. These are coverage gaps, not bugs in the tests themselves — they represent production code with no test exercising the branch.
${uncoveredPaths.map(p => `- \`${p.file}:${p.line}\` — ${p.description}`).join("\n")}
```

Extract **Uncovered code paths** by scanning all three findings sections for items tagged `MISSING TEST` or `uncovered` (the custom instructions ask reviewers to tag explicitly). If nothing qualifies, render "None — reviewers did not flag any uncovered production branches."

### Step 7: Missing / Weak / Flaky / Nit triage

Test review does NOT auto-fix — rewriting a test can mask the bug it was catching. Classify every finding into one of four buckets and present them for human decision:

- **🧪 Missing Tests** — a production path has no test. The production code itself may or may not be correct. **Default recommended action: dispatch a test-writing subagent** (one per file) to add the missing test. Don't dispatch from this skill file — recommend the pattern to the user so they can approve.
- **⚠️ Weak Tests** — a test exists but doesn't actually assert behavior (`toBeTruthy`, missing assertions, over-mocked). **Present for human decision** — rewriting weak tests is risky because the weak test may be the only thing catching a real bug, and tightening it could expose a latent failure the current suite is hiding.
- **🎲 Flaky Risks** — tests that work today but will fail intermittently (time-dependent, order-dependent, shared state, brittle selectors, hard-coded waits). **Present for human decision** — fixing flakiness sometimes requires real design changes (dependency injection, clock abstraction, test isolation).
- **💡 Nits** — style, naming, fixture organization. Fold in during the next pass. Don't dispatch fixes.

Report format:

```
🧪 Missing Tests: {N} — recommended: dispatch test-writing subagents (one per file)
⚠️  Weak Tests: {M} — present for human decision (rewriting may mask real bugs)
🎲 Flaky Risks: {K} — present for human decision (may require design changes)
💡 Nits: {J} — fold in during next test pass
```

If there are zero Missing and zero Weak findings, report "✅ Test suite looks solid — no missing coverage or weak assertions flagged."

**Never auto-rewrite Weak or Flaky tests from `/multipov-tests`.** A weak test that passes today might be the only thing catching a bug you don't know about. Present the finding, let the human decide whether to tighten, investigate, or keep as-is.

### Step 8: Log the review

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
# Keep the URL for follow-ups. Josh can pull it up in the web UI and revisit
# after tests have been written or tightened.
```

---

## Error handling

- **`invalid_input`** — usually means the diff is empty or malformed. Check `git diff main..HEAD` manually.
- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap ($100). Rare. `details.reset_at` has the rollover. Not your problem to fix.
- **`not_found`** on poll — probably a stale `job_id`. Did you switch terminals? Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down, review still runs but with a dumber panel. Quality will be lower than usual; note it in the report header. Test reviews are especially hurt by this because the test-focused framing depends on routing picking `test-reviewer` into the panel.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch the status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Tips

- **Run `/multipov-tests` BEFORE merging any PR that adds or modifies tests.** This is the highest-leverage moment — caught here, the fix is a one-line assertion tweak; caught later, it's a silent regression shipped to prod.
- **Run it periodically against `main..HEAD~20`** (or similar) to catch coverage drift. Tests that were thorough six months ago may have gone stale as production code grew around them.
- **Integration tests MUST hit a real database, not mocks.** This is Josh's documented preference and the custom instructions already call it out — if a reviewer flags a mocked DB in what's labeled an "integration test," that's a real finding, not a nit. Fix it by standing up a test DB or a containerized Postgres, not by relaxing the assertion.
- **Don't over-index on coverage percentage.** 95% line coverage with 50% branch coverage is worse than 70% line coverage with 90% branch coverage. The reviewers are instructed to focus on branch/edge-case coverage, not raw percentages — trust that signal.
- **Don't auto-rewrite Weak Tests.** A weak assertion (`toBeTruthy`) might be the last thing catching a real bug. Tightening it to `toBe(expected)` could surface a latent failure you then have to chase. Present the finding, let the human decide.
- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — the test-focused `custom_instructions` already biases routing toward `test-reviewer` + `staff-reviewer` + `security-reviewer`.
- **Always use `mode: "deep"`** for test reviews. Quick mode is Claude-only and misses the cross-model consensus that makes the Weak Test / Flaky Risk triage worth running.
- Once you've added the missing tests, run `/multipov-tests` again on the new diff. It's idempotent and the 10/day quota is generous enough for two passes on the same PR.

---

## Migration note (2026-04-13)

This skill is **NEW** as of 2026-04-13 — it is NOT a migration of an existing local skill. Created alongside the batch migration of `/multipov-code` and `/multipov-plan` to the multipov.ai MCP server, since test-focused review is a distinct lens that deserves its own entry point.

**MCP tool:** This skill routes to `mcp__multipov__submit_pipeline_review` (confirmed registered at `src/mcp/build-server.ts:487`). There is NO dedicated `submit_test_review` tool on the server today, and we deliberately did not add one:

1. The shared code-review input shape (`sharedInputShape` at `src/mcp/tools-code-review.ts:148`) is identical across `submit_plan_review`, `submit_pipeline_review`, and `submit_codebase_review` — there is no `scope` / `focus` / `framing` parameter to toggle. Adding a fourth server-side tool just for test-focused framing would duplicate ~90% of `submit_pipeline_review`'s logic for a framing-string delta.
2. The existing `custom_instructions` field (capped at `MAX_CUSTOM_INSTRUCTIONS_CHARS`, verified at `src/mcp/tools-code-review.ts:183`) is appended to the type-specific framing AND passed as the routing hint, so test-focused framing in `custom_instructions` biases both the LLM prompts AND the persona routing toward `test-reviewer` / `staff-reviewer` / `security-reviewer` naturally. That's exactly the lever we need.
3. Keeping this client-side means we can iterate on the test-focused framing by editing this one markdown file instead of shipping a Worker deploy.

If the test-focused framing becomes unwieldy in `custom_instructions` (e.g. we start wanting it longer than `MAX_CUSTOM_INSTRUCTIONS_CHARS`, or the routing hint stops picking up test reviewers reliably), the right next step is to add a `review_type: "test"` option to the existing `submit_pipeline_review` shape — NOT a separate tool. But we're nowhere near that threshold today.

**One-time setup:** if you don't have a multipov API key yet, generate one at `https://multipov.ai/settings/api-keys`, save it to 1Password as "multipov.ai — MCP API Key (Claude Code)" in the Employee vault, then run `/prep-multipov` to register it in `~/.claude.json`. Setup is documented at `https://github.com/capitalthought/multipov/blob/main/docs/mcp/README.md`.

See `docs/mcp/README.md` in the multipov repo for the full tool catalog.
