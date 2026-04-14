---
name: review-all
description: Run a comprehensive multi-agent codebase review on the whole repo, a subdirectory, or a file glob via the multipov.ai MCP server. Flags both immediate issues AND systemic patterns.
---

# Review All — Comprehensive Codebase Review via multipov.ai

Runs a comprehensive multi-agent codebase review on `$ARGUMENTS` (default: the entire repo; may also be a subdir like `src/api` or a file glob like `src/auth/*.ts`) via the multipov.ai MCP server.

This is the **codebase-wide** sibling of `/review-pipeline` — use it for pre-launch, pre-release, monthly audits, or whenever you want a read on systemic patterns rather than just the latest diff.

---

## What this skill does

This skill calls the multipov.ai MCP server's `submit_codebase_review` tool. Server-side, the review runs against a content-routed 8-person panel (5 technical reviewers + 3 outside perspectives) with **codebase-review framing** — reviewers flag both immediate issues AND systemic patterns suggesting tech debt or design erosion.

- **One MCP call** — no local reviewer fan-out.
- **Consistent reviewer pool across machines.**
- **Shared daily quota** — counts against your multipov review quota (10/day).
- **Audit trail** — every review lands in `list_my_reviews`.

Cost: ~$0.26 per review in quick mode, ~$0.98 in deep mode.

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
  echo "Concatenated codebase is ${CONTENT_CHARS} chars, approaching the 1M cap."
  echo "Narrow the scope (pass a subdir or glob) or run /review-all on modules one at a time."
  exit 0
fi

if [ "$CONTENT_CHARS" -eq 0 ]; then
  echo "Nothing to review — content is empty."
  exit 0
fi
```

### Step 3: Build the review body

```
CODEBASE REVIEW: ${REPO_NAME} — ${SCOPE_LABEL}
Files (${FILE_COUNT}):
${FILES}

${CONTENT}
```

### Step 4: Submit via MCP

```typescript
await mcp__multipov__submit_codebase_review({
  content: /* the context-prefixed concat from Step 3 */,
  file_name: `${REPO_NAME}-codebase.txt`,
  mode: "deep",  // always deep for codebase reviews — systemic findings need cross-model consensus
  custom_instructions: /* optional extra framing, e.g., "focus on error handling and retry logic" */,
})
```

The server auto-picks the 8-person panel via content-aware routing with **codebase-review framing** (immediate issues + systemic patterns / tech debt / design erosion). You don't need to pass `personas`.

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
## Comprehensive Codebase Review Report

**Scope:** `${SCOPE_LABEL}` · **Files:** ${FILE_COUNT} · **Chars:** ${CONTENT_CHARS}
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

Same Fix-first classification as `/review-pipeline` — real code is being reviewed, so auto-fix is fair game for mechanical issues:

**AUTO-FIX** (implement immediately without asking):
- Dead code removal, stale comments, missing logging, unused imports, magic numbers, obvious typos, missing `await`, simple null checks.

**ASK** (batch into one question):
- Security changes, race conditions, architecture decisions, removing functionality, adding dependencies, performance trade-offs, public API changes.

**One caveat unique to `/review-all`:** codebase reviews frequently flag **systemic patterns** (e.g., "17 handlers share this unsafe unmarshal pattern"). For those, don't dispatch 17 parallel fix agents — that's merge-conflict hell. Pick one file as the reference fix, agree on the pattern, then sweep the rest in a single follow-up pass.

For AUTO-FIX items: dispatch implementation subagents immediately (one per file, in parallel).
For ASK items: present all in a single summary grouped by file with the recommended fix.

Report: "Auto-fixed {N} mechanical issues. {M} items need your input (see below)."

### Step 8: Log the review URL

```bash
REVIEW_ID=${job_id}
echo "Review: https://multipov.ai/review/${REVIEW_ID}"
```

Keep the URL — codebase reviews are worth revisiting over time to track debt.

---

## Error handling

- **`rate_limited`** — 10/day or 3-concurrent cap. `details.kind` says which.
- **`cost_cap_reached`** — server-wide daily cost cap hit. Rare.
- **`file_too_large`** — your concat blew past 1M chars. Narrow the scope or review modules one at a time.
- **`not_found`** on poll — stale `job_id`. Run `list_my_reviews`.
- **`persona_selection_method = "fallback_alphabetical"`** — routing was down; quality will be lower than usual.
- **Timeout after 10 min** — come back with `get_review_report({ job_id })` later.

---

## Tips

- **Don't pass `personas`** unless you explicitly want to override the content-aware panel.
- **Always use `mode: "deep"`** for codebase reviews. Systemic findings need cross-model consensus.
- Use `/review-pipeline` for a diff-focused review of the latest branch. Use `/review-all` when you want the reviewers to read the whole surface and flag systemic patterns.
- For focused codebase reviews — scaling, performance, or security — use `/review-scaling`, `/review-performance`, or `/review-security`. They call the same MCP but pass `custom_instructions` to steer the framing.
- Run `/review-all` monthly, or before any public launch.

---

## References

- MCP tool catalog and error reference: see the multipov MCP docs repo README.
- multipov.ai settings: `https://multipov.ai/settings`
