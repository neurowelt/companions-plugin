---
description: List LLM model slugs valid for the Companions API `model` override.
---

Call the `list_models` MCP tool. Present the result as a short table:

- One row per model.
- Columns: `slug`, `display_name`, `temperature` range, `top_p` range.

Slugs outside this list are rejected at the wire with a 422; `ctx.accepted` on that error carries the same set.

If the call returns 401 or the env var is unset, run the same fallback as `/companions-setup` instead.
