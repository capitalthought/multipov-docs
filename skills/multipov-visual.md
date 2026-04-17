Run the Visual Review pipeline — review iOS apps, web pages, PDFs, or entire websites using multi-LLM visual consensus.

Part of the multipov command family (alongside `/multipov-code`, `/multipov-plan`, `/multipov-all`, `/multipov-performance`, `/multipov-reliability`, `/multipov-scaling`, `/multipov-security`, `/multipov-tests`). Note: implementation is still a local CLI (`~/Xcode/mikey/bin/visual-review.ts`) that calls GPT-4o/Gemini/Grok directly — the multipov.ai MCP server does not yet expose a visual-review endpoint. When server-side image support lands, this skill should be migrated to `mcp__multipov__submit_visual_review`.

## What this does

1. Captures screenshots (iOS simulator, Playwright browser, or PDF rendering)
2. Sends each screenshot to LLMs (GPT-4o, Gemini, Grok) for visual analysis
3. Deduplicates findings across models
4. Writes a report with sections: Failures, Design & Polish, Warnings, Passed Screens/Pages

## Usage

The argument $ARGUMENTS is passed as flags to the visual-review CLI.

### iOS App Review
```bash
/multipov-visual --project serendipity --fast
/multipov-visual --project serendipity --screen home-grid
/multipov-visual --all --estimate
```

### Web Page Review
```bash
/multipov-visual --url https://capitalfactory.com --fast
/multipov-visual --url https://app.example.com --storage-state auth.json
/multipov-visual --url https://example.com --dark-mode
```

### Whole Site Review
```bash
/multipov-visual --site https://capitalfactory.com --fast
/multipov-visual --site https://capitalfactory.com --sitemap --max-pages 20
/multipov-visual --site https://capitalfactory.com --urls-file pages.txt
```

### PDF Review
```bash
/multipov-visual --file ~/Downloads/mockup.pdf --fast
/multipov-visual --file ~/Downloads/deck.pdf --pages 1-10
```

### Common Options
```
--fast              Use one model only (~75% cheaper)
--estimate          Print cost without running
--budget-cap N      Abort if cost exceeds $N
--models a,b,c      Select specific models
--no-cache          Force fresh review
--output-dir PATH   Custom report location
```

## Run the command

Execute from the mikey directory:

```bash
cd /Users/joshuabaer/Xcode/mikey && npx tsx bin/visual-review.ts $ARGUMENTS
```

After the review completes, read the generated report and present a summary of findings to the user. If there are Design & Polish findings, highlight those specifically — they're the most valuable output.
