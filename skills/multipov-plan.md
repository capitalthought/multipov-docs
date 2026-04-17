Run a multi-agent design-doc review on `$ARGUMENTS` (path to a markdown design doc, typically `docs/plans/YYYY-MM-DD-topic-design.md`) via the multipov.ai MCP server.

This is the **plan** sibling of `/multipov-code` — use it BEFORE you write any code, to pressure-test the architecture while fixes are still cheap.

---

## Architecture (as of 2026-04-13)

This skill **calls the multipov.ai MCP server** — it no longer dispatches local reviewer agents, doesn't shell out to `llm-review.ts`, doesn't fan out to GPT-4o/Gemini/Grok directly. All of that now happens server-side in multipov's `executeReview` against a content-routed 8-person panel (5 technical + 3 outside perspectives) with **plan-review framing** (architecture, correctness, missing requirements, failure modes — NOT code style, since there is no code yet).

Benefits over the old local pipeline:
- **One MCP call instead of 5 parallel subagents** — faster wall clock, lower complexity.
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

### Step 2: Resolve and read the design doc

Plan review requires an **explicit target** — there is no fallback to "last committed work" like `/multipov-code` has. The user must pass a path.

```bash
DOC_PATH="$1"

if [ -z "$DOC_PATH" ]; then
  echo "❌ /multipov-plan requires a path to a design doc (e.g., docs/plans/2026-04-13-foo-design.md)"
  exit 0
fi

# Accept both absolute and repo-relative paths
if [ "${DOC_PATH:0:1}" != "/" ]; then
  DOC_PATH="$(pwd)/$DOC_PATH"
fi

if [ ! -f "$DOC_PATH" ]; then
  echo "❌ Design doc not found: $DOC_PATH"
  exit 0
fi

DOC_CONTENT=$(cat "$DOC_PATH")
DOC_CHARS=$(printf "%s" "$DOC_CONTENT" | wc -c | tr -d ' ')
DOC_WORDS=$(printf "%s" "$DOC_CONTENT" | wc -w | tr -d ' ')
DOC_BASENAME=$(basename "$DOC_PATH")

if [ "$DOC_CHARS" -gt 950000 ]; then
  echo "⚠️ Design doc is ${DOC_CHARS} chars, approaching the 1M cap. Consider splitting it."
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

Use the MCP tool `mcp__multipov__submit_plan_review`. If the tool isn't available in the session, the MCP config isn't loaded — re-run `/prep-multipov`.

Tool call shape:

```typescript
await mcp__multipov__submit_plan_review({
  content: /* the context-prefixed doc from Step 3 */,
  file_name: DOC_BASENAME,  // e.g., "2026-04-13-foo-design.md"
  mode: "deep",  // always deep for plan reviews — architectural calls are high-stakes
  custom_instructions: /* optional extra framing, e.g., "focus on the auth boundary" */,
})
```

The server auto-picks the 8-person panel via content-aware routing with **plan-review framing** (architecture / correctness / missing requirements / failure modes — NOT code style). You don't need to pass `personas` unless you want to override.

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

### ⚠️ Progress visibility rule

The polling loop MUST run in the **main conversation context**, not inside a subagent. If a subagent is used for submission (e.g., to handle large file reads), it must return the `job_id` immediately after `submit_plan_review` returns. The main context then takes over for polling + progress bar rendering.

**Why:** Subagent tool calls and text output are invisible to the user until the subagent finishes. A 2-minute poll loop inside a subagent means 2 minutes of silence with no progress bar.

**Pattern:**
1. (Optional) Subagent reads the design doc and calls `submit_plan_review` → returns `{ job_id, total_count, estimated_seconds }`
2. Main context prints initial progress bar immediately
3. Main context runs the poll loop with `get_review_status({ job_id, wait_seconds: 20 })`, printing progress between each call

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
// report has: executiveSummary, consensusFindings, disputedFindings,
//             singleModelFindings, personaResults, reviewerRelevance, durationMs
```

Present a consensus-first markdown report — same shape as `/multipov-code`, but the header says "Plan Review Report" and the metadata line names the doc instead of a diff range:

```markdown
## 🧭 Plan Review Report

**Doc:** `${DOC_PATH}` · **Words:** ${DOC_WORDS}
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

### Step 7: Blocker / Revision / Nit triage

Plan review does **not** auto-fix. There's no code yet — the output of this skill is a revised design doc, not a patched file. Classify every finding into one of three buckets and recommend action on the **doc**, not on a codebase:

- **🚫 Blocker** — the plan as written is wrong. Architecture flaw, security hole in the design, missing requirement that would force a rewrite mid-implementation, unclear scope that no engineer could execute against. **Do not start coding until the design doc is revised.**
- **✏️ Revision** — the plan works, but a section needs to be tightened. Missing failure mode, unclear DB schema, hand-waved error handling, ambiguous API contract. **Update the design doc before opening the implementation PR** — cheaper than discovering it mid-diff.
- **💡 Nit** — preference, naming, doc-hygiene. Log as an implementation consideration; don't block on it.

Report format:

```
🚫 Blockers: {N} — revise the plan before writing code
✏️ Revisions: {M} — tighten these sections before the implementation PR
💡 Nits: {K} — fold in during implementation
```

If there are zero blockers and zero revisions, report "✅ Plan is ready to implement — kick off `/multipov-code` once you have a diff."

**Never dispatch auto-fix subagents from `/multipov-plan`.** That's `/multipov-code`'s job, and only after the code exists.

### Step 8: Log the review

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
# Keep the URL for follow-ups. Josh can pull it up in the web UI and revisit
# after the design doc has been revised.
```

---

## Error handling

- **`invalid_input`** — usually means the design doc is empty or too short for the server's minimum. Ask the user to expand it (at least a Problem + Solution section) before re-submitting.
- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap ($100). Rare. `details.reset_at` has the rollover. Not your problem to fix.
- **`not_found`** on poll — probably a stale `job_id`. Did you switch terminals? Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down, review still runs but with a dumber panel. Quality will be lower than usual; note it in the report header.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch the status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Fallback to local pipeline (emergency only)

If multipov.ai is down and you need a plan review NOW, the old local implementation is archived at `~/icloud/Claude/commands-archive/review-plan-local.md`. Copy it to `~/icloud/Claude/commands/multipov-plan.md` temporarily. Do NOT leave it swapped in — the MCP version is the canonical flow. Restore this file from git / iCloud once multipov is back up.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — you lose the 5+3 engineering-vs-outside-perspective split that makes plan review useful.
- **Always use `mode: "deep"`** for plan review. Architectural calls are cheap to fix on paper and expensive to fix in code — the quick-mode Claude-only pass isn't worth it here.
- Plan review is **the highest-leverage review in the whole pipeline.** A `🚫 Blocker` caught here costs minutes; the same thing caught in `/multipov-code` after implementation costs hours.
- User persona reviewers (in the outside-perspective slot) are especially valuable at the plan stage — they catch UX and product friction before code exists.
- Keep the design doc concise. A 1-page plan gets better reviews than a 10-page spec.
- Once the design is approved and you've implemented it, run `/multipov-code` on the resulting diff. That's the sibling skill for code review, same MCP server, same panel shape — different framing.

---

## Migration note (2026-04-13)

This skill was rewritten to call the multipov.ai MCP server instead of dispatching local reviewer subagents. It migrated alongside `/multipov-code` on the same day. The old local implementation is preserved at `~/icloud/Claude/commands-archive/review-plan-local.md` as a break-glass fallback. Same rationale as the pipeline migration:
1. A consistent reviewer pool across machines
2. Cost observability via multipov's shared daily cap
3. An audit trail in `list_my_reviews`
4. Fewer moving parts (one MCP call vs 5 parallel subagents + 15 LLM calls)

**One-time setup:** if you don't have a multipov API key yet, generate one at `https://multipov.ai/settings/api-keys`, save it to 1Password as "multipov.ai — MCP API Key (Claude Code)" in the Employee vault, then run `/prep-multipov` to register it in `~/.claude.json`. Setup is documented at `https://github.com/capitalthought/multipov/blob/main/docs/mcp/README.md`.

See `docs/mcp/README.md` in the multipov repo for the full tool catalog.
