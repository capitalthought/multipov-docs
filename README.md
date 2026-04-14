# Connect to multipov.ai from your MCP client

multipov.ai exposes its review, rewrite, and persona surface as a [Model Context Protocol](https://modelcontextprotocol.io) server. Any MCP client — Claude Code, Cursor, Codex, Zed, Raycast, Claude Desktop — can call multi-perspective document reviews from a terminal or editor, billed against the same daily quota as the web UI.

**Endpoint:** `https://multipov.ai/mcp`
**Transport:** Streamable HTTP (sessionless)
**Auth:** API key (Bearer token)
**Spec version:** MCP 2025-03-26

---

## 1. Get an API key

1. Log into [multipov.ai](https://multipov.ai).
2. Open `/settings/api-keys`.
3. Click **Generate**.
4. Copy the token (`mpov_live_...`). **It is shown once.** Store it in your password manager.

API keys carry your full account permissions. Treat a leaked key the same as a leaked password — revoke it immediately at `/settings/api-keys` and generate a new one.

---

## 2. Add the server to your MCP client

### Claude Code

Claude Code defaults to **stdio** transport. You must pass `--transport http` or the connection will fail silently.

```bash
claude mcp add --transport http multipov https://multipov.ai/mcp \
  --header "Authorization: Bearer mpov_live_YOUR_TOKEN_HERE"
```

Verify:

```bash
claude mcp list
# multipov: connected
```

In any Claude Code session, `/mcp` lists the tools, or just ask directly: "use multipov to review this diff with 5 engineers."

### Cursor

Add to `~/.cursor/mcp.json` (or **Settings → MCP → Add Server**):

```json
{
  "mcpServers": {
    "multipov": {
      "url": "https://multipov.ai/mcp",
      "headers": {
        "Authorization": "Bearer mpov_live_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

Restart Cursor. The tools appear under the MCP icon in the chat composer.

### Codex

```bash
codex mcp add multipov https://multipov.ai/mcp \
  --header "Authorization: Bearer mpov_live_YOUR_TOKEN_HERE"
```

### Claude Desktop, Zed, Raycast, other clients

Any client that supports MCP over HTTP works. The server URL is `https://multipov.ai/mcp` and the auth header is `Authorization: Bearer mpov_live_...`. Follow your client's instructions for adding an HTTP MCP server with a custom header.

---

## 3. What you can do

14 tools across four groups:

**Personas** — browse the persona catalog, fetch a single persona, or ask the server to recommend a panel for a given document.

**Reviews** — submit a document for multi-perspective review, poll status, fetch the synthesized report, cancel in-flight reviews, list recent reviews.

**Rewrites** — rewrite a document in a chosen persona's voice. Intensity ranges from voice-only to full rewrite. Annotations explain each change.

**Code reviews** — engineering-focused variants: `submit_plan_review` (design doc / implementation plan before you write code), `submit_pipeline_review` (PR-sized diff), `submit_codebase_review` (entire module / multi-file audit). Each uses a content-aware 8-person panel with type-specific framing.

All three code-review tools accept `content` OR `file_url`, `file_name`, `mode` (`quick` | `deep`), `personas` (to override the auto-selected panel), `custom_instructions`, and `skepticism` (1–5).

---

## 4. Example session

From a Claude Code session after installing:

```
> use multipov to review this PRD with 5 startup investors. save as prd-review.md when done.
```

Claude will call `submit_review`, poll `get_review_status` with the server-recommended interval, and fetch `get_review_report` once complete. A typical deep-mode review takes 60–120 seconds and returns consensus findings, disputed findings, and single-model findings.

Or, for a code review:

```
> use multipov's submit_pipeline_review to review my current branch diff against main.
```

---

## 5. Rate limits & cost

The MCP surface **shares the same daily quota as the web UI** — there is no separate MCP bucket.

| Limit                              | Default |
|------------------------------------|---------|
| Reviews per day                    | 10      |
| Concurrent reviews                 | 3       |
| Deep-mode reviews per day (global) | 5       |
| Rewrites per day                   | 10      |
| Concurrent rewrites                | 3       |

Quick mode uses Claude only. Deep mode fans out to Claude + GPT-4o + Gemini + Grok for cross-model consensus. Cost is tracked server-side and returns `rate_limited` with `details.kind` when a bucket trips.

---

## 6. Errors

Every error returns a structured JSON payload:

```json
{
  "code": "rate_limited",
  "message": "Daily review limit reached. Resets at 2026-04-14T05:00:00Z.",
  "details": {
    "kind": "daily",
    "reset_at": "2026-04-14T05:00:00Z",
    "retry_after_seconds": 3600
  }
}
```

| Code               | When                                                                 |
|--------------------|----------------------------------------------------------------------|
| `invalid_input`    | Required field missing or malformed. `details.field` names which.    |
| `file_too_large`   | Content > 1M chars.                                                  |
| `persona_not_found`| A persona id you passed isn't in your visible catalog.               |
| `rate_limited`     | Hit one of the buckets above. `details.kind` says which.             |
| `not_found`        | Unknown id, or id owned by another user (same response for both).    |
| `job_not_complete` | You called `get_review_report` before status is `complete`.          |
| `internal_error`   | Something unexpected. `details.error_id` correlates with server logs.|

---

## 7. Privacy & data processing

Submitted content is processed by **Anthropic** (Claude), **OpenAI** (GPT-4o), **Google** (Gemini), and **xAI** (Grok) in deep mode. Quick mode uses Claude only. See [multipov.ai/privacy](https://multipov.ai/privacy) for the full sub-processor list.

API keys are stored as **SHA-256 hashes** server-side. The plaintext is shown exactly once at creation and discarded. There is no recovery mechanism — if you lose a key, generate a new one and revoke the old.

---

## 8. Known limitations

1. **Sessionless transport** → no `notifications/progress`, no `resources/subscribe`. Polling is the only progress channel. Clients should honor the `recommended_poll_interval_ms` field in status responses.
2. **API key revocation propagates within ~60 seconds** across edge isolates due to per-isolate caching.
3. **No local file uploads** — pass `content` (raw text) or `file_url` (allowlisted https hosts only: Google Docs/Drive, Dropbox public links, GitHub raw, GitHub gist raw, Notion public pages).
4. **`list_my_reviews` is capped at 100 jobs.** Older review ids are dropped from the index.
5. **No per-key scopes yet.** A compromised key has full user-level access until revoked.

---

## 9. Support

- **Errors:** include the `details.error_id` from any `internal_error` payload.
- **Security:** email `security@multipov.ai` (do not file public issues for security reports).
- **Docs / install bugs:** [open an issue on this repo](https://github.com/capitalthought/multipov-docs/issues).
- **Product bugs / feature requests:** email `support@multipov.ai`.

---

## 10. Claude Code users — `/prep-multipov` skill

If you use [Claude Code](https://claude.com/claude-code), the [`skills/prep-multipov.md`](./skills/prep-multipov.md) skill automates the install + health-check flow above:

1. Save it to `~/.claude/skills/prep-multipov/SKILL.md`
2. Run `/prep-multipov` in any Claude Code session
3. It checks whether the MCP server is registered, prompts for your API key if missing, registers it, and verifies the connection

It's a shell-based skill that relies on the `MULTIPOV_API_KEY` environment variable or your preferred secrets manager — adapt as needed for other MCP clients.
