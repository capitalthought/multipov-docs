---
name: multipov-facts
description: Fact-check a document by fanning it out to GPT-4o + Gemini + Grok in parallel via the multipov MCP server. Only flags claims when ≥2 models agree AND a source URL is present. Use when Josh says "fact-check", "verify claims", "check facts", "are these claims right", or passes a path to a research deep-dive, briefing, board memo, or strategic-research doc. Pass the document path as $ARGUMENTS.
---

Fact-check a document via the multipov MCP server. Pass a single markdown / text / PDF path as `$ARGUMENTS`.

## When to use

Strategic research deep-dives, briefings, board memos, fund updates, market scans — anywhere a wrong claim is expensive. Not for code, not for pitch decks (use `/multipov` for those).

The 4-gate filter (consensus + source-URL + whitelist + known-corrections) targets a 30–50% signal rate on this kind of content. Without those gates, raw multi-model output runs ~60–70% false positives and you spend more time triaging than the fact-check saved.

## Process

Call the multipov MCP server's Fact Check tools:

1. **Resolve inputs.** Read `$ARGUMENTS` as the document path. If a `_CORRECTIONS-APPLIED.md` (or similarly named) file lives beside it, load it as `known_corrections` so already-fixed claims surface as **CONFIRMING** instead of **NEW**. If a sibling `factcheck-whitelist.json` exists, load it as `whitelist` (false-positive substrings to suppress).

2. **Submit the job.** Call `submit_fact_check` with sensible defaults:

   ```
   submit_fact_check({
     document: { content: <doc text>, file_name: <basename> },
     known_corrections: <lines from corrections file, or omit>,
     whitelist: <array of substrings, or omit for server default>,
     consensus_threshold: 2,
     require_source_url: true,
   })
   ```

   The defaults mirror the canonical CLI version of this command (`/multipov-facts`): consensus ≥2 distinct models, source URL required (no URL → severity downgraded to `Note`), built-in whitelist for admin-terminology / leadership-transition / federal-program-name false positives.

3. **Print the panel before progress.** Before showing any "polling…" output, print:
   - The doc name + size
   - The 3 models that will run (GPT-4o, Gemini, Grok)
   - The applied gates: consensus threshold, source-URL requirement, whitelist size, known-corrections count

4. **Poll status.** Call `get_fact_check_status` with `wait_seconds=20` until status is `complete`. A typical fact check takes 60–120 seconds.

5. **Fetch and format the result.** Call `get_fact_check_result`, then print the report consensus-first using these sections from the master summary:

   - **Critical** — confirmed errors, ≥2 models agree, source URL present, high consequence
   - **Important** — confirmed errors, lower consequence
   - **Re-research** — single-model claims that look plausible but lack the consensus or URL to confirm
   - **CONFIRMING** — findings that match `known_corrections` (validates fixes you already made)
   - **False positives** — single-model findings dropped by the source-URL or whitelist gate (kept visible so you can sanity-check the gates)
   - **Model performance** — success rate + avg findings per model

6. **Top-3 actionable.** End with a "Top 3 corrections worth applying right now" callout pulled from the Critical section. Optimize for the user being able to fix the doc immediately.

## Defaults

| Setting | Default | Why |
|---------|---------|-----|
| `consensus_threshold` | `2` | Single-model assertions are too noisy. Two models agreeing on the same correction is the cheapest signal that holds up. |
| `require_source_url` | `true` | A correction without a URL is unverifiable. Downgrade to `Note`, don't drop entirely. |
| `whitelist` | server built-in | Catches the most common LLM mis-flags (recent admin renames, leadership transitions, federal program-name colloquialisms). |
| `known_corrections` | sibling `_CORRECTIONS-APPLIED.md` if present | Lets you re-run after applying fixes without losing track of what's already done. |

## Output rules

- **Print the panel BEFORE the progress bar.** No silent waits.
- **Lead with consensus.** The Critical section is the headline; everything else is supporting context.
- **Surface false positives, don't hide them.** Users learn the gate behavior by seeing what got dropped and why.
- **Always include the report URL** (or local save path) at the end so the user can revisit the full output.
- **Save the full report** to `docs/fact-checks/YYYY-MM-DD-<doc-name>-factcheck.md` in the current working directory if writing to disk is appropriate.

## Tips

- Re-running with `known_corrections` after applying fixes is cheap (3 model calls) and the right move whenever you've changed the source doc — the CONFIRMING tag tells you which fixes the models still see.
- The source-URL gate is regex-only — it does NOT verify reachability. Run a separate link-check job if URL liveness matters.
- A Fact Check job costs ≈ $0.05–0.10 per document (3 model calls). Cheap enough to run on every draft of a high-stakes doc.
- For perspective-based / opinion review (does this argument hold up?) use `/multipov` instead — Fact Check is for verifiable factual claims only.
- The MCP-based version of this skill targets a single document. For batch fact-checking a folder of docs, use the local CLI version `/multipov-facts <folder>` (see [`~/icloud/Claude/commands/multipov-facts.md`](https://github.com/capitalthought/multipov-docs)).

## Why this matters

Today's reference run on a 35-doc strategic-research corpus surfaced ~257 raw findings, ~120 unique after dedup, and ~20 actually consequential — and one of those 20 was a board-chair name correction in a doc that was about to ship to a fund LP. Without the consensus + source-URL gates, that finding would have been buried under 70+ false positives and skipped. The gates ARE the product.
