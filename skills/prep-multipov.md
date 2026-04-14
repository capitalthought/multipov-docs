---
name: prep-multipov
description: Verify that the multipov.ai MCP server is configured, authenticated, and responsive in Claude Code. Run before any session that uses multipov tools, or when you want to confirm the server is reachable.
---

# Prep Multipov — Verify multipov.ai MCP Server

Ensures the multipov.ai MCP server is registered in `~/.claude.json`, the API key is valid, and the server responds to tool calls.

**Estimated time:** ~10 seconds (first-time setup adds ~30s for the key generation prompt)

---

## Phase 1 — Check MCP Registration

Check if `multipov` is already registered as an MCP server:

```bash
if command -v jq >/dev/null 2>&1; then
  multipov_config=$(jq -r '.mcpServers.multipov // empty' ~/.claude.json 2>/dev/null)
  if [ -n "$multipov_config" ]; then
    echo "MCP_REGISTERED=yes"
    echo "$multipov_config" | jq .
  else
    echo "MCP_REGISTERED=no"
  fi
else
  if grep -q '"multipov"' ~/.claude.json 2>/dev/null; then
    echo "MCP_REGISTERED=yes"
  else
    echo "MCP_REGISTERED=no"
  fi
fi
```

Also check `claude mcp list` for runtime status:

```bash
claude mcp list 2>/dev/null | grep -i multipov || echo "MCP_RUNTIME=not_loaded"
```

---

## Phase 2 — Get the API Key

The multipov API key should be available in your environment. Check in this order:

1. **Environment variable:** `MULTIPOV_API_KEY`
2. **Your secrets manager** (1Password, macOS Keychain, Bitwarden, etc.)
3. **Prompt the user to generate one** if none exists

```bash
if [ -n "$MULTIPOV_API_KEY" ]; then
  echo "KEY_SOURCE=env"
else
  echo "KEY_SOURCE=none"
fi
```

**If no key is available, prompt the user:**

```
No multipov API key found.

To generate one:
  1. Go to https://multipov.ai/settings/api-keys
  2. Click "Generate"
  3. Copy the token (starts with mpov_live_) — it's shown ONCE
  4. Store it somewhere safe (password manager, env var, etc.)
  5. Export it before re-running this skill:
       export MULTIPOV_API_KEY="mpov_live_..."

Then re-run /prep-multipov.
```

**Stop here if no key is available.** Nothing else works without it.

---

## Phase 3 — Register the MCP Server

Skip if `MCP_REGISTERED=yes` and the config is correct (right URL + Authorization header).

Otherwise, register it:

```bash
if [ -z "$MULTIPOV_API_KEY" ]; then
  echo "Set MULTIPOV_API_KEY first."
  exit 1
fi

claude mcp add --transport http multipov https://multipov.ai/mcp \
  --header "Authorization: Bearer $MULTIPOV_API_KEY"
```

Verify:

```bash
jq '.mcpServers.multipov' ~/.claude.json 2>/dev/null
```

Expected shape:

```json
{
  "type": "http",
  "url": "https://multipov.ai/mcp",
  "headers": {
    "Authorization": "Bearer mpov_live_..."
  }
}
```

**After registering a new MCP server, Claude Code must be restarted** for the server to load. Tell the user:

```
multipov MCP server registered in ~/.claude.json.
Restart Claude Code for the server to load, then re-run /prep-multipov to verify.
```

---

## Phase 4 — Health Check

Skip if Phase 3 just registered the server (it won't be loaded until restart).

### 4A. Server health (direct HTTP)

```bash
curl -s -o /tmp/multipov-health.json -w "%{http_code}" \
  --max-time 5 "https://multipov.ai/health" 2>/dev/null
echo " <- HTTP code"
```

Expect `200`. Anything else means the server is unreachable or degraded.

### 4B. MCP tool check

If the MCP server is loaded in the current Claude Code session, make a lightweight tool call:

```
Call mcp__multipov__list_personas with { "limit": 1 }
```

- **Success:** server is connected and authenticated.
- **Auth error:** the API key is invalid or revoked. Go back to Phase 2.
- **Connection error:** the server may be down. Check `https://multipov.ai/health`.
- **Tool not found:** the MCP server isn't loaded in this session. Restart Claude Code.

### 4C. Quota check

```
Call mcp__multipov__list_my_reviews with { "limit": 1 }
```

This confirms auth works and lets you eyeball how much of your daily quota is already used.

---

## Phase 5 — Status Report

### All clear

```
multipov MCP Server — Ready

| Check             | Status                       |
|-------------------|------------------------------|
| MCP registered    | yes (~/.claude.json)         |
| API key           | yes                          |
| Server health     | 200 OK                       |
| MCP tools loaded  | N tools available            |

Ready to use multipov review, rewrite, and persona tools.
```

### Needs setup

```
multipov MCP Server — Setup required

Action items:
  1. Generate an API key at https://multipov.ai/settings/api-keys
  2. Store it (env var or password manager)
  3. Re-run /prep-multipov
```

### Registered but needs restart

```
multipov MCP Server — Restart required

The server is registered in ~/.claude.json but isn't loaded in this session.
Restart Claude Code, then re-run /prep-multipov.
```

### Server down

```
multipov MCP Server — Unreachable

multipov.ai appears to be down. Check https://multipov.ai/health directly.
```

---

## Notes

- `~/.claude.json` is **per-machine**. You'll need to register the MCP server once on each machine you use Claude Code from.
- The API key itself is portable across machines — generate once, share across your own devices however you normally share secrets.
- Daily quota and concurrency limits are enforced **per account**, not per key. Using the same account from two machines shares the same bucket.
- See `https://multipov.ai/docs/mcp` (or the `docs/mcp/README.md` in the public repo, once it's open) for the full tool catalog and error reference.
