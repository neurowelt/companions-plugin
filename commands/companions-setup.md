---
description: Verify the Companions API key is set and the MCP is reachable; otherwise walk the user through setup.
---

# Companions — setup check

Call the `check_balance` MCP tool and report the first matching outcome:

- Returns a balance envelope → say "Companions is set up. Balance: $X." Done.
- Returns 401 / authentication error, **or** the MCP request fails because the `Authorization` header expanded to an empty/missing key → the env var is either unset or the key is invalid. Print the "Setup walkthrough" below (note: if the user already had a key, they need to replace it).
- Network / 5xx error → the key is fine but the service is unreachable. Print the service URL (`https://api.humx.ai/mcp`) and ask the user to confirm they're online.

Do not try to read `COMPANIONS_API_KEY` from the shell — the key is a secret and must not be echoed. The MCP call itself is the auth check.

## Setup walkthrough

If the user has no key:

1. Contact the team to acquire your API key (currently the only way).
3. Export it in your shell. For zsh / bash add this line to `~/.zshrc` or `~/.bashrc`:
   ```bash
   export COMPANIONS_API_KEY=cmp_live_...
   ```
4. Reload your shell (`source ~/.zshrc`) and **restart Claude Code** so the new env is picked up by the MCP transport.
5. Re-run `/companions-setup` to verify.

If the user already has a key but it's invalid, the only step is to regenerate at the dashboard and replace the export line.

> The plugin sends `Authorization: ApiKey ${COMPANIONS_API_KEY}` on every MCP request. The key never leaves your machine via this plugin's code — only via the MCP request itself, which goes directly to `api.humx.ai`.
