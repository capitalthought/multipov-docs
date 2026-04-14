---
name: review-tests
description: Run a test-focused multi-agent review via the multipov.ai MCP server. Pressure-tests your tests — missing assertions, uncovered paths, over-mocking, flaky risks, weak coverage. Does NOT auto-fix.
---

# Review Tests — Test-Quality Review via multipov.ai

Runs a test-focused multi-agent review on `$ARGUMENTS` (default: test files touched in the current branch) via the multipov.ai MCP server.

This is the **test-quality** sibling of `/review-pipeline`. Use it to pressure-test your tests — not the production code. It answers: are there untested paths? Do the tests actually assert behavior? Are edge cases covered? Is mocking masking bugs? Are integration tests hitting real systems? Is coverage lopsided (1000 lines of happy-path, 10 lines of error-handling)?

---

## What this skill does

This skill calls `mcp__multipov__submit_pipeline_review` with **test-focused `custom_instructions`**. There is no dedicated `submit_test_review` tool on the server — the shared code-review input shape doesn't accept a `scope` parameter, so client-side framing is the lever. Content-aware routing then naturally picks a test-heavy panel: `test-reviewer`, `staff-reviewer`, `security-reviewer`, and a few outside perspectives.

Cost: same as `/review-pipeline` — ~$0.26 quick, ~$0.98 deep.

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

`/review-tests` reviews **tests**, so the default scope is test files touched in the current branch. If the user passed a glob or path as `$ARGUMENTS` (e.g. `src/mcp/` or `src/__tests__/*.ts`), use that instead.

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
  echo "Diff is ${DIFF_CHARS} chars, approaching the 1M cap."
  echo "Consider narrowing with an explicit scope: /review-tests src/foo/"
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

Use `mcp__multipov__submit_pipeline_review` (same tool `/review-pipeline` uses — there is no dedicated test-review tool). The lever is `custom_instructions`.

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

Do NOT flag production code for refactoring, style, naming, or architecture — that's /review-pipeline's job.
`.trim(),
})
```

The server auto-picks the 8-person panel via content-aware routing. With test-focused framing in `custom_instructions` the router naturally weights toward `test-reviewer` + `staff-reviewer` + `security-reviewer` + a couple of outside perspectives. Don't pass `personas` — let the routing do its thing.

### Step 5: Poll with a progress bar

Use `wait_seconds=20` so the server holds the connection until something changes. Render a progress bar between calls and print it as regular text output so the user sees it update in real time. Max 30 iterations (~10 min ceiling).

### Step 6: Fetch the report and present findings

Present a test-focused consensus-first markdown report. Header says "Test Review Report" and the report adds an **Uncovered code paths** section at the bottom — reviewers are explicitly asked to flag coverage gaps with file:line, even for branches not in the diff.

```markdown
## Test Review Report

**Scope:** `${REVIEW_MODE_NOTE}` · **Files:** ${FILE_COUNT} · **Chars:** ${DIFF_CHARS}
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

### Uncovered code paths
Files or branches the reviewers flagged as untested. These are coverage gaps, not bugs in the tests themselves — they represent production code with no test exercising the branch.
${uncoveredPaths.map(p => `- \`${p.file}:${p.line}\` — ${p.description}`).join("\n")}
```

Extract **Uncovered code paths** by scanning all three findings sections for items tagged `MISSING TEST` or `uncovered`. If nothing qualifies, render "None — reviewers did not flag any uncovered production branches."

### Step 7: Missing / Weak / Flaky / Nit triage

Test review does NOT auto-fix — rewriting a test can mask the bug it was catching. Classify every finding into one of four buckets and present them for human decision:

- **Missing Tests** — a production path has no test. The production code itself may or may not be correct. **Default recommended action: dispatch a test-writing subagent** (one per file) to add the missing test. Don't dispatch from this skill file — recommend the pattern to the user so they can approve.
- **Weak Tests** — a test exists but doesn't actually assert behavior (`toBeTruthy`, missing assertions, over-mocked). **Present for human decision** — rewriting weak tests is risky because the weak test may be the only thing catching a real bug, and tightening it could expose a latent failure the current suite is hiding.
- **Flaky Risks** — tests that work today but will fail intermittently (time-dependent, order-dependent, shared state, brittle selectors, hard-coded waits). **Present for human decision** — fixing flakiness sometimes requires real design changes (dependency injection, clock abstraction, test isolation).
- **Nits** — style, naming, fixture organization. Fold in during the next pass. Don't dispatch fixes.

Report format:

```
Missing Tests: {N} — recommended: dispatch test-writing subagents (one per file)
Weak Tests: {M} — present for human decision (rewriting may mask real bugs)
Flaky Risks: {K} — present for human decision (may require design changes)
Nits: {J} — fold in during next test pass
```

If there are zero Missing and zero Weak findings, report "Test suite looks solid — no missing coverage or weak assertions flagged."

**Never auto-rewrite Weak or Flaky tests from `/review-tests`.** A weak test that passes today might be the only thing catching a bug you don't know about. Present the finding, let the human decide whether to tighten, investigate, or keep as-is.

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

---

## Error handling

- **`invalid_input`** — usually means the diff is empty or malformed. Check `git diff main..HEAD` manually.
- **`rate_limited`**, **`cost_cap_reached`**, **`not_found`** — standard. See `/review-pipeline`.
- **`persona_selection_method = "fallback_alphabetical"`** — routing was down; the review runs with a dumber panel. Test reviews are especially hurt by this because the test-focused framing depends on routing picking `test-reviewer` into the panel.
- **Timeout after 10 min** — come back with `get_review_report({ job_id })` later.

---

## Tips

- **Run `/review-tests` BEFORE merging any PR that adds or modifies tests.** This is the highest-leverage moment — caught here, the fix is a one-line assertion tweak; caught later, it's a silent regression shipped to prod.
- **Run it periodically against `main..HEAD~20`** (or similar) to catch coverage drift. Tests that were thorough six months ago may have gone stale as production code grew around them.
- **Integration tests MUST hit a real database, not mocks.** The custom instructions already call this out — if a reviewer flags a mocked DB in what's labeled an "integration test," that's a real finding, not a nit. Fix it by standing up a test DB or a containerized Postgres, not by relaxing the assertion.
- **Don't over-index on coverage percentage.** 95% line coverage with 50% branch coverage is worse than 70% line coverage with 90% branch coverage. The reviewers are instructed to focus on branch/edge-case coverage, not raw percentages — trust that signal.
- **Don't auto-rewrite Weak Tests.** A weak assertion (`toBeTruthy`) might be the last thing catching a real bug. Tightening it to `toBe(expected)` could surface a latent failure you then have to chase. Present the finding, let the human decide.
- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — the test-focused `custom_instructions` already biases routing toward `test-reviewer` + `staff-reviewer` + `security-reviewer`.
- **Always use `mode: "deep"`** for test reviews. Quick mode is Claude-only and misses the cross-model consensus that makes the Weak Test / Flaky Risk triage worth running.
- Once you've added the missing tests, run `/review-tests` again on the new diff. It's idempotent and the 10/day quota is generous enough for two passes on the same PR.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
