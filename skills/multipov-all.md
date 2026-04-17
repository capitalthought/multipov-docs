Run a comprehensive multi-agent codebase review on `$ARGUMENTS` (default: the entire repo; may also be a subdir like `src/api` or a file glob like `src/auth/*.ts`) via the multipov.ai MCP server.

This is the **codebase-wide** sibling of `/multipov-code` — use it for pre-launch, pre-release, monthly audits, or whenever you want a read on systemic patterns rather than just the latest diff.

---

## Architecture (as of 2026-04-13)

This skill **calls the multipov.ai MCP server** — it no longer dispatches 5+ local review panels, doesn't shell out to `llm-review.ts`, doesn't fan out to GPT-4o/Gemini/Grok directly. All of that now happens server-side in multipov's `executeReview` against a content-routed 8-person panel (5 technical + 3 outside perspectives) with **codebase-review framing** — reviewers flag both immediate issues AND systemic patterns suggesting tech debt or design erosion.

Benefits over the old local pipeline:
- **One MCP call instead of 5 parallel panel subagents** — faster wall clock, lower complexity.
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

### Step 2: Gather the codebase content

`$ARGUMENTS` is optional. Three shapes, in order of precedence:

1. **No argument** → review the whole repo (honoring `.gitignore`). Use `git ls-files` so you get tracked files only and skip build artifacts.
2. **A subdirectory** (e.g., `src/api`) → review all tracked files under that path.
3. **A glob** (e.g., `src/auth/*.ts`) → review just the files that match.

```bash
TARGET="${1:-}"
REPO_NAME=$(basename "$(git rev-parse --show-toplevel)")

if [ -z "$TARGET" ]; then
  FILES=$(git ls-files)
  SCOPE_LABEL="entire repo"
elif [ -d "$TARGET" ]; then
  FILES=$(git ls-files -- "$TARGET")
  SCOPE_LABEL="directory: $TARGET"
else
  # Treat as glob — let the shell expand it, then intersect with tracked files
  FILES=$(ls -1 $TARGET 2>/dev/null | while read f; do
    git ls-files --error-unmatch "$f" 2>/dev/null && echo "$f"
  done | sort -u)
  SCOPE_LABEL="glob: $TARGET"
fi

FILE_COUNT=$(printf "%s\n" "$FILES" | grep -c . || echo 0)

if [ "$FILE_COUNT" -eq 0 ]; then
  echo "Nothing to review — no tracked files match '$TARGET'."
  exit 0
fi
```

Concatenate the files into a single blob with clear per-file headers so reviewers can cite `file:line`:

```bash
CONTENT=""
for f in $FILES; do
  CONTENT="${CONTENT}

=== FILE: $f ===
$(cat "$f")
"
done

CONTENT_CHARS=$(printf "%s" "$CONTENT" | wc -c | tr -d ' ')

if [ "$CONTENT_CHARS" -gt 950000 ]; then
  echo "⚠️ Concatenated codebase is ${CONTENT_CHARS} chars, approaching the 1M cap."
  echo "Narrow the scope (pass a subdir or glob) or run /multipov-all on modules one at a time."
  exit 0
fi

if [ "$CONTENT_CHARS" -eq 0 ]; then
  echo "Nothing to review — content is empty."
  exit 0
fi
```

### Step 3: Build the review body

Prefix the concatenated files with a context block so reviewers know what they're looking at:

```
CODEBASE REVIEW: ${REPO_NAME} — ${SCOPE_LABEL}
Files (${FILE_COUNT}):
${FILES}

${CONTENT}
```

### Step 4: Submit via MCP

Use the MCP tool `mcp__multipov__submit_codebase_review`. If the tool isn't available in the session, the MCP config isn't loaded — re-run `/prep-multipov`.

Tool call shape:

```typescript
await mcp__multipov__submit_codebase_review({
  content: /* the context-prefixed concat from Step 3 */,
  file_name: `${REPO_NAME}-codebase.txt`,
  mode: "deep",  // always deep for codebase reviews — systemic findings need cross-model consensus
  custom_instructions: /* optional extra framing, e.g., "focus on error handling and retry logic" */,
})
```

The server auto-picks the 8-person panel via content-aware routing with **codebase-review framing** (immediate issues + systemic patterns / tech debt / design erosion). You don't need to pass `personas` unless you want to override.

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
## 📚 Comprehensive Codebase Review Report

**Scope:** `${SCOPE_LABEL}` · **Files:** ${FILE_COUNT} · **Chars:** ${CONTENT_CHARS}
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

### Step 7: Fix-first triage

Same Fix-first classification as `/multipov-code` — real code is being reviewed, so auto-fix is fair game for mechanical issues:

**AUTO-FIX** (implement immediately without asking):
- Dead code removal, stale comments, missing logging, unused imports, magic numbers, obvious typos, missing `await`, simple null checks

**ASK** (batch into one question):
- Security changes, race conditions, architecture decisions, removing functionality, adding dependencies, performance trade-offs, public API changes

**One caveat unique to `/multipov-all`:** codebase reviews frequently flag **systemic patterns** (e.g., "17 handlers share this unsafe unmarshal pattern"). For those, don't dispatch 17 parallel fix agents — that's merge-conflict hell. Pick one file as the reference fix, agree on the pattern, then sweep the rest in a single follow-up pass.

For AUTO-FIX items: dispatch implementation subagents immediately (one per file, in parallel).
For ASK items: present all in a single summary grouped by file with the recommended fix.

Report: "🔧 Auto-fixed {N} mechanical issues. {M} items need your input (see below)."

### Step 8: Log the review

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
# Keep the URL — codebase reviews are worth revisiting over time to track debt.
```

---

## Error handling

- **`rate_limited`** — you hit the 10/day or 3-concurrent cap. `details.kind` says which. Wait for the rollover or cancel an in-flight review.
- **`cost_cap_reached`** — server-wide daily cost cap ($100). Rare. `details.reset_at` has the rollover. Not your problem to fix.
- **`file_too_large`** — your concat blew past 1M chars. Narrow the scope (subdir or glob) or review modules one at a time.
- **`not_found`** on poll — probably a stale `job_id`. Did you switch terminals? Run `list_my_reviews` to see your recent jobs.
- **`persona_selection_method = "fallback_alphabetical"`** on the submit response — routing was down, review still runs but with a dumber panel. Quality will be lower than usual; note it in the report header.
- **Timeout after 10 min** — deep mode can occasionally run long. Fetch the status one more time; if it's still running, let it finish in the background and come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel — you lose the 5+3 engineering-vs-outside-perspective split.
- **Always use `mode: "deep"`** for codebase reviews. Systemic findings need cross-model consensus to separate signal from noise.
- Use `/multipov-code` for a diff-focused review of the latest branch. Use `/multipov-all` when you want the reviewers to read the whole surface and flag systemic patterns (tech debt, design erosion, inconsistency across modules).
- For **focused** codebase reviews — scaling, performance, or security — use `/multipov-scaling`, `/multipov-performance`, or `/multipov-security`. They call the same MCP but pass `custom_instructions` to steer the framing.
- Run `/multipov-all` monthly, or before any public launch.

---

## Migration note (2026-04-13)

This skill was rewritten to call the multipov.ai MCP server instead of dispatching 5 local panel subagents. It migrated alongside `/multipov-code`, `/multipov-plan`, `/multipov-scaling`, `/multipov-performance`, and `/multipov-security` on the same day. Same rationale as the pipeline migration:
1. A consistent reviewer pool across machines
2. Cost observability via multipov's shared daily cap
3. An audit trail in `list_my_reviews`
4. Fewer moving parts (one MCP call vs 5 parallel panel subagents + 20 LLM calls)

**One-time setup:** if you don't have a multipov API key yet, generate one at `https://multipov.ai/settings/api-keys`, save it to 1Password as "multipov.ai — MCP API Key (Claude Code)" in the Employee vault, then run `/prep-multipov` to register it in `~/.claude.json`. Setup is documented at `https://github.com/capitalthought/multipov/blob/main/docs/mcp/README.md`.

See `docs/mcp/README.md` in the multipov repo for the full tool catalog.
