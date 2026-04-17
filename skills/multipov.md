Review a document by dispatching multiple persona agents against it. The document can be a PDF, markdown, or text file. Pass the file path as $ARGUMENTS.

## Process

Run the review-doc orchestrator with a single command:

```bash
npx tsx ~/icloud/Claude/bin/src/review-doc-runner.ts "$ARGUMENTS"
```

### Flags

| Flag | Description |
|------|-------------|
| `--type <hint>` | Override document type detection (pitch, plan, email, briefing, proposal) |
| `--add p1,p2` | Add specific personas beyond router selections |
| `--only p1,p2` | Bypass routing -- review with only these personas (+ josh-baer) |
| `--max-reviewers N` | Cap number of personas (default: 3 normal, 10 with --full; josh-baer excluded from count) |
| `--full` | Multi-LLM fan-out: 4 models x up to 10 personas (~40 API calls, ~$2-5). Default is Claude-only + 3 personas |
| `--dry-run` | Show routing results and cost estimate, then exit without making API calls |
| `--preview` | Show plan and exit -- for manual review before running |
| `--full-output` | Print full narrative report to terminal (default: summary to stdout, full report auto-saved to disk) |
| `--json` | Machine-readable JSON output for pipeline integration |
| `--force` | Proceed even if secret scan detects potential secrets in the document |
| `--relationship-context <text>` | Optional context about existing relationship with persona |
| `--delivery-context <text>` | Optional context about how/when document will be delivered |

### Examples

```bash
# Default: Claude-only + 3 personas (fast, cheap, data stays local)
/review-doc ~/Downloads/"_Arsenal Fund Deck.pdf"

# Full multi-LLM fan-out (4 models x 10 personas, ~$2-5)
/review-doc ~/Downloads/deck.pdf --full

# Dry run to see persona selection and cost estimate
/review-doc ~/Downloads/deck.pdf --dry-run

# Force specific personas only
/review-doc ~/Downloads/email.txt --only michael-dell-investor,hunt-companies-investor

# Print full report to terminal (not just summary)
/review-doc ~/Downloads/deck.pdf --full-output

# JSON output for pipeline integration
/review-doc ~/Downloads/deck.pdf --json

# Preview plan before running
/review-doc ~/Downloads/deck.pdf --preview
```

### What the orchestrator does

1. **Loads accuracy feedback** from previous reviews in the same project (`docs/reviews/accuracy-feedback.json`) to calibrate reviewer sensitivity and avoid repeating known false positives
2. **Extracts text** from PDF (via pdftotext), markdown, or text files
3. **Validates document size** -- warns at >20K tokens, hard-caps at 60K tokens
4. **Scans for secrets** -- blocks dispatch to external models in --full mode if secrets detected (unless --force). Default mode keeps data local
5. **Routes to personas** -- Haiku-based classification (falls back to keyword matching if no API key)
6. **Dispatches reviews** -- default: Claude-only per persona; --full: parallel fan-out to Claude + GPT-4o + Gemini + Grok per persona (with shared p-limit concurrency, circuit breaker, and retry with exponential backoff)
7. **Merges narratives** -- Haiku synthesizes per-persona consensus from multi-model reviews
8. **Synthesizes executive summary** -- cross-persona themes via final Haiku call
9. **Assembles report** -- markdown with fit matrix, executive summary, per-persona reviews, and accuracy feedback template
10. **Saves** to `docs/reviews/YYYY-MM-DD-{doc-name}-review.md`

### Present to user

After the orchestrator completes, present:
- The Fit Assessment Matrix
- The executive summary (cross-persona themes)
- Top 3-5 improvement suggestions
- Link to the full saved report
- Remind that the report includes an accuracy feedback section for marking findings

### Accuracy Feedback

After the review is complete, check for any accuracy feedback from previous reviews in the same project to calibrate reviewer sensitivity and avoid repeating false positives. The orchestrator does this automatically by loading `docs/reviews/accuracy-feedback.json`.

**How it works:**

1. Each saved review report includes an "Accuracy Feedback" section at the bottom with a table listing every finding/suggestion
2. Josh marks each finding as accurate, inaccurate, or partially accurate, and adds notes explaining why
3. To record feedback, edit the accuracy feedback JSON file directly or use the companion command:

```bash
# Record feedback for a specific review
npx tsx ~/icloud/Claude/bin/src/accuracy-feedback.ts log \
  --review docs/reviews/2025-03-25-deck-review.md \
  --finding 1 --accurate false --notes "This investor would actually love this structure"

# View accuracy stats by persona
npx tsx ~/icloud/Claude/bin/src/accuracy-feedback.ts stats

# View false positive patterns
npx tsx ~/icloud/Claude/bin/src/accuracy-feedback.ts patterns
```

4. On subsequent reviews, the orchestrator loads this feedback history and includes a calibration note in the prompt for personas that have had inaccurate findings flagged, helping them avoid the same mistakes

**Feedback file:** `docs/reviews/accuracy-feedback.json` -- auto-created on first use, checked into the repo so feedback persists across sessions.

## Tips

- Default mode (no flags) is Claude-only with 3 personas -- fast (~30s), cheap (~$0.05), and data stays local. This is the recommended starting point.
- Use `--preview` first on unfamiliar documents to see which personas will be selected and the cost estimate before committing.
- Use `--dry-run` for scripting and CI integration -- it outputs JSON routing results with no API calls.
- The `--full` flag enables multi-LLM fan-out (Claude + GPT-4o + Gemini + Grok) for richer consensus analysis, but costs ~$2-5 per run.
- Secret scanning only blocks dispatch in `--full` mode (external models). Default Claude-only mode keeps data local regardless.
- The orchestrator handles all retry logic, circuit breaking, and concurrency control internally -- no manual intervention needed.
- Reports auto-save to `docs/reviews/` in the current working directory. Use `--full-output` to also print to terminal.
- Persona files live in `~/icloud/Claude/agents/`. If iCloud files are evicted, run `brctl download ~/icloud/Claude/agents/` to materialize them.
- Requires `pdftotext` for PDF files: `brew install poppler`
- After reviewing a report, fill in the accuracy feedback section to help the system learn. Even marking 2-3 findings per review builds a useful calibration dataset over time.
