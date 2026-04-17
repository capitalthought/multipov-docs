---
name: all-recommendations
description: Launch a team of implementation agents to apply ALL recommended changes from the most recent review — every severity level (Critical, High, Medium, Low), one agent per file, with false-positive validation baked in.
---

# All Recommendations — Batch-Implement Every Review Finding

Launch a team of agents to implement ALL recommended changes from the most recent review — every severity level (Critical, High, Medium, Low). Input: `$ARGUMENTS` (path to a review report file, or omit to use findings from the current conversation).

This is the companion to the `/review-*` skills in this repo. Most review workflows only auto-fix Critical and High findings and log the rest as TODOs; this one goes all the way through Medium and Low, one file at a time, with every finding validated before it's applied.

---

## What this skill does

Unlike the `/review-*` skills, this skill does **not** call the multipov MCP server. It runs entirely locally:

1. Gathers findings from a review report or the current conversation
2. Deduplicates and groups them by file
3. Dispatches one local Claude Code subagent per file, in parallel, to implement the fixes
4. Each agent validates every finding before applying it (roughly half of all review findings are false positives on well-maintained codebases)
5. Collects results, runs a build to verify nothing broke, and presents a consolidated report

You run this **after** a review has already produced findings. Typical flow:

```
/multipov-code          → produces a consensus-first report
/all-recommendations      → implements everything that isn't a false positive
/multipov-code          → (optional) verifies the fixes didn't introduce regressions
```

---

## Prerequisites

- A recent review whose findings you want to implement. Either:
  - A review report file you pass as `$ARGUMENTS`, or
  - Findings already present in the current Claude Code conversation (from a prior `/multipov-code`, `/multipov-plan`, `/multipov-all`, or similar).
- A clean working tree — or at least a commit point you can roll back to. This skill modifies files.
- A working build command so the verification step can tell you whether the changes compile.

---

## Process

### Step 1: Gather Findings

Collect all findings from the review output. Sources (checked in order):

1. **`$ARGUMENTS`** — if a file path is provided, read the review report from that file.
2. **Current conversation** — if no path given, look for the most recent review output in this conversation (from `/multipov-all`, `/multipov-code`, `/multipov-plan`, or similar).
3. **Abort** — if no findings can be located, tell the user to run a review first.

Parse all findings into a structured list. Each finding should have:

- `severity`: Critical | High | Medium | Low
- `file`: file path (if available)
- `line`: line number (if available)
- `description`: what's wrong
- `recommendation`: how to fix it
- `reviewer`: which reviewer found it
- `consensus`: how many models agreed (if available)

### Step 2: Validate & Prioritize

Before dispatching agents, validate the finding list:

1. **Remove duplicates** — same file + same issue from different reviewers counts once (keep the highest severity and note all reviewers who flagged it).
2. **Group by file** — cluster findings by file path. (Critical rule: **one agent per file for writes.**)
3. **Order by severity** — Critical → High → Medium → Low within each file group.
4. **Flag conflicts** — if two findings for the same file contradict each other (e.g., "add error handling" vs "remove unnecessary error handling"), flag for manual review and exclude from automated implementation.

Present the implementation plan to the user:

```
## Implementation Plan

### Files to modify: {count}
### Total findings to implement: {count}

| File | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| path/to/file.ts | 1 | 2 | 0 | 1 | 4 |
| ... | ... | ... | ... | ... | ... |

### Conflicts requiring manual review: {count}
- {file}: {description of conflict}

Proceeding with implementation...
```

### Step 3: Dispatch Implementation Agents

Dispatch one subagent per file group **in parallel** using the Agent tool. Each agent should work in an isolated worktree (if your harness supports it) so parallel agents don't collide.

**CRITICAL RULE:** One agent per file. Never assign the same file to multiple agents — parallel edits to the same file will produce merge conflicts.

For each file group, dispatch a subagent with this prompt:

```
You are an implementation agent. Your job is to implement specific review findings in a single file.

## File: {file_path}

## Findings to implement (in priority order):

{for each finding in this file group:}
### [{severity}] {description}
- **Reviewer:** {reviewer}
- **Line:** {line}
- **Recommendation:** {recommendation}
- **Consensus:** {consensus}

## Instructions

1. Read the file and understand its context.
2. For EACH finding, in priority order:
   a. VERIFY the finding is valid — check if the issue actually exists in the current code.
      (Roughly half of review findings are false positives on well-maintained codebases.)
   b. If valid: implement the recommended fix.
   c. If false positive: skip it and note why.
   d. If the fix would break other code: note the risk and implement conservatively.
3. After all changes, verify the file still compiles/parses correctly.
4. Return a summary of what was implemented vs skipped:

## Results for {file_path}
### Implemented
- [{severity}] {description} — {what you did}

### Skipped (false positive)
- [{severity}] {description} — {why it's not actually an issue}

### Skipped (risky)
- [{severity}] {description} — {what risk was identified}
```

**Batching:** If there are more than 8 files to modify, dispatch in waves of 8 to avoid overwhelming the system. Wait for each wave to complete before dispatching the next.

**Progress-aware dispatch:** If your harness supports background agent dispatch with completion notifications, dispatch agents that way so you can print a progress bar that updates as each agent finishes. If not, fall back to sequential dispatch with a progress bar update between each agent.

Pattern:

1. Print the initial progress bar with all empty blocks.
2. Dispatch all agents (in parallel if possible, sequentially otherwise).
3. As each agent completes, parse its results and update the progress bar.
4. After all agents complete, proceed to Step 4.

### Step 3b: Progress Bar

Print a progress bar as regular text output (not inside a tool call) so the user sees it update in real time. Update it at these points:

1. **Before first dispatch** — show the starting state with all empty blocks.
2. **On each agent completion** — increment the filled count, update the impl/skip/risky tallies from that agent's results.
3. **After all agents complete** — show the final tally line.

```
filled = completed agents
total  = total files to modify
bar    = "█".repeat(filled) + "░".repeat(total - filled)
Print: [bar] {filled}/{total} files — {implemented}/{skipped_fp}/{skipped_risky} impl/skip/risky
```

Example output for a 12-file review:

```
[░░░░░░░░░░░░] 0/12 files — 0/0/0 impl/skip/risky
[█░░░░░░░░░░░] 1/12 files — 2/0/0 impl/skip/risky
[██░░░░░░░░░░] 2/12 files — 3/1/0 impl/skip/risky
[█████░░░░░░░] 5/12 files — 7/4/1 impl/skip/risky
[████████░░░░] 8/12 files — 12/5/1 impl/skip/risky
[██████████░░] 10/12 files — 15/7/1 impl/skip/risky
[████████████] 12/12 files — Complete — 18/8/2 impl/skip/risky
```

**Running tallies:** As each agent completion arrives, parse its Implemented / Skipped (false positive) / Skipped (risky) counts and add them to the running totals. This gives the user a real-time sense of the false-positive rate as the run progresses, instead of a stuck 0/N bar for several minutes.

### Step 4: Collect Results & Verify

After all agents complete:

1. **Aggregate results** from all agents into a summary report.
2. **Run the build** to verify nothing is broken. Use the build command appropriate for the project — for example:

   ```bash
   # Xcode project
   xcodebuild -scheme {scheme} -destination 'platform=macOS' \
     SWIFT_TREAT_WARNINGS_AS_ERRORS=YES \
     CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
     build 2>&1 | grep -E "error:|warning:" | head -20

   # Node / TypeScript
   npm run build && npm run typecheck

   # Python
   python -m compileall . && ruff check .
   ```

3. **Run tests** if available to catch regressions.

### Step 5: Present Summary

Present the final implementation report:

```
## Implementation Complete

### Stats
- Implemented: {count} findings
- Skipped (false positive): {count} findings
- Skipped (risky): {count} findings
- Conflicts (manual review needed): {count} findings

### By Severity
| Severity | Implemented | Skipped | Total |
|----------|-------------|---------|-------|
| Critical | X | Y | Z |
| High     | X | Y | Z |
| Medium   | X | Y | Z |
| Low      | X | Y | Z |

### Build Status: Clean / {error count} errors
### Test Status:  All passing / {failure count} failures

### Changes by File
{for each file:}
**{file_path}** — {count} changes
- [{severity}] {description}
- [{severity}] {description} (skipped — false positive: {reason})

### Items Requiring Manual Attention
{any conflicts, risky skips, or build/test failures}
```

### Step 6: Recommend Next Steps

- **Build failures:** offer to fix them immediately.
- **Test failures:** offer to investigate and fix.
- **Skipped risky items:** suggest manual review.
- **Conflicts:** present both sides and ask the user to decide.
- **All clean:** suggest running `/multipov-code` or `/multipov-all` again to verify the fixes, then committing.

---

## Tips

- This skill is designed to run **after** a review (`/multipov-code`, `/multipov-plan`, `/multipov-all`, `/multipov-security`, etc.).
- Unlike most workflows that only fix Critical/High, this implements all severity levels including Medium and Low.
- The ~50% false-positive rate means roughly half the findings will be skipped — this is normal and expected on well-maintained codebases. If your false-positive rate is much higher or lower, it's usually a sign you need to adjust the review framing, not the implementation step.
- Each agent validates findings before implementing, so you won't get blindly applied bad fixes.
- If the review was on a design doc (from `/multipov-plan`), findings may not have specific file/line references — agents will need to locate the relevant code themselves.
- For very large reviews (50+ findings), consider running in waves and reviewing intermediate results before proceeding to the next wave.
- Always verify with a build after implementation — agent changes can interact in unexpected ways, especially when two files import from each other and both got edited in the same run.

---

## References

- `/multipov-code`, `/multipov-plan`, `/multipov-all`, `/multipov-security`, `/multipov-performance`, `/multipov-scaling`, `/multipov-tests` — the review skills that produce findings this skill consumes.
- Issues and suggestions: [capitalthought/multipov-docs](https://github.com/capitalthought/multipov-docs/issues).
