---
name: review-plan
description: Run a multi-agent design-doc review via the multipov.ai MCP server BEFORE you write code. Pressure-tests architecture, requirements, and failure modes while fixes are still cheap.
---

# Review Plan — Multi-Agent Design Doc Review via multipov.ai

Run a multi-agent design-doc review on `$ARGUMENTS` (path to a markdown design doc, typically `docs/plans/YYYY-MM-DD-topic-design.md`) via the multipov.ai MCP server.

This is the **plan** sibling of `/review-pipeline` — use it BEFORE you write any code, to pressure-test the architecture while fixes are still cheap.

---

## What this skill does

This skill calls the multipov.ai MCP server's `submit_plan_review` tool. Server-side, the review runs against a content-routed 8-person panel (5 technical reviewers + 3 outside perspectives) with **plan-review framing** (architecture, correctness, missing requirements, failure modes — NOT code style, since there is no code yet).

- **One MCP call** — no local reviewer fan-out.
- **Consistent reviewer pool across machines.**
- **Shared daily quota** — counts against your multipov review quota (10/day).
- **Audit trail** — every review lands in `list_my_reviews` with a `persona_selection_method` field.

Cost: ~$0.26 per review in quick mode, ~$0.98 in deep mode (8 personas × 4 LLMs + routing).

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

### Step 2: Resolve and read the design doc

Plan review requires an **explicit target** — there is no fallback to "last committed work" like `/review-pipeline` has. The user must pass a path.

```bash
DOC_PATH="$1"

if [ -z "$DOC_PATH" ]; then
  echo "/review-plan requires a path to a design doc (e.g., docs/plans/2026-04-13-foo-design.md)"
  exit 0
fi

# Accept both absolute and repo-relative paths
if [ "${DOC_PATH:0:1}" != "/" ]; then
  DOC_PATH="$(pwd)/$DOC_PATH"
fi

if [ ! -f "$DOC_PATH" ]; then
  echo "Design doc not found: $DOC_PATH"
  exit 0
fi

DOC_CONTENT=$(cat "$DOC_PATH")
DOC_CHARS=$(printf "%s" "$DOC_CONTENT" | wc -c | tr -d ' ')
DOC_WORDS=$(printf "%s" "$DOC_CONTENT" | wc -w | tr -d ' ')
DOC_BASENAME=$(basename "$DOC_PATH")

if [ "$DOC_CHARS" -gt 950000 ]; then
  echo "Design doc is ${DOC_CHARS} chars, approaching the 1M cap. Consider splitting it."
fi

if [ "$DOC_CHARS" -eq 0 ]; then
  echo "Nothing to review — design doc is empty."
  exit 0
fi
```

### Step 3: Build the review body

Prefix the doc with a short context line so reviewers know what file they're looking at:

```
DESIGN DOC: $DOC_BASENAME (${DOC_WORDS} words)

$DOC_CONTENT
```

That's it — no file list, no diff, no git context. A plan review is just the plan.

### Step 4: Submit via MCP

```typescript
await mcp__multipov__submit_plan_review({
  content: /* the context-prefixed doc from Step 3 */,
  file_name: DOC_BASENAME,  // e.g., "2026-04-13-foo-design.md"
  mode: "deep",  // always deep for plan reviews — architectural calls are high-stakes
  custom_instructions: /* optional extra framing, e.g., "focus on the auth boundary" */,
})
```

The server auto-picks the 8-person panel via content-aware routing with **plan-review framing** (architecture / correctness / missing requirements / failure modes — NOT code style). You don't need to pass `personas`.

### Step 5: Poll with a progress bar

**Progress visibility rule:** the polling loop MUST run in the main conversation context, not inside a subagent. If a subagent is used for submission (e.g., to handle large file reads), it must return the `job_id` immediately after `submit_plan_review` returns. The main context then takes over for polling + progress bar rendering.

**Why:** subagent tool calls and text output are invisible to the user until the subagent finishes. A 2-minute poll loop inside a subagent means 2 minutes of silence with no progress bar.

Use `wait_seconds=20` so the server holds the connection until something changes — no client-side sleep needed:

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
```

Present a consensus-first markdown report:

```markdown
## Plan Review Report

**Doc:** `${DOC_PATH}` · **Words:** ${DOC_WORDS}
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

### Step 7: Blocker / Revision / Nit triage

Plan review does **not** auto-fix. There's no code yet — the output of this skill is a revised design doc, not a patched file. Classify every finding into one of three buckets:

- **Blocker** — the plan as written is wrong. Architecture flaw, security hole in the design, missing requirement that would force a rewrite mid-implementation, unclear scope that no engineer could execute against. **Do not start coding until the design doc is revised.**
- **Revision** — the plan works, but a section needs to be tightened. Missing failure mode, unclear DB schema, hand-waved error handling, ambiguous API contract. **Update the design doc before opening the implementation PR** — cheaper than discovering it mid-diff.
- **Nit** — preference, naming, doc-hygiene. Log as an implementation consideration; don't block on it.

Report format:

```
Blockers: {N} — revise the plan before writing code
Revisions: {M} — tighten these sections before the implementation PR
Nits: {K} — fold in during implementation
```

If there are zero blockers and zero revisions, report "Plan is ready to implement — kick off `/review-pipeline` once you have a diff."

**Never dispatch auto-fix subagents from `/review-plan`.** That's `/review-pipeline`'s job, and only after the code exists.

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

Keep the URL for follow-ups — you can pull it up in the web UI and revisit after the design doc has been revised.

---

## Error handling

- **`invalid_input`** — usually means the design doc is empty or too short for the server's minimum. Ask the user to expand it (at least a Problem + Solution section) before re-submitting.
- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which.
- **`cost_cap_reached`** — server-wide daily cost cap hit. Rare.
- **`not_found`** on poll — probably a stale `job_id`. Run `list_my_reviews`.
- **`persona_selection_method = "fallback_alphabetical"`** — routing was down; the review runs with a dumber panel. Quality will be lower than usual.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch status one more time, or come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — you lose the 5+3 engineering-vs-outside-perspective split that makes plan review useful.
- **Always use `mode: "deep"`** for plan review. Architectural calls are cheap to fix on paper and expensive to fix in code.
- Plan review is **the highest-leverage review in the pipeline.** A blocker caught here costs minutes; the same thing caught in `/review-pipeline` after implementation costs hours.
- Outside-perspective reviewers are especially valuable at the plan stage — they catch UX and product friction before code exists.
- Keep the design doc concise. A 1-page plan gets better reviews than a 10-page spec.
- Once the design is approved and you've implemented it, run `/review-pipeline` on the resulting diff.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
