---
name: consulting-experts
description: Use when the user's problem would benefit from domain-expert
  perspectives — explicit asks ("what would a neuroscientist say"),
  contested questions with no single right answer, or moments when
  you've hit a wall and need a fresh angle. Drives the `companions`
  MCP server (discover, list_models, consult, check_balance). Not for
  things you already know cold.
---

# Consulting experts via the Companions MCP

You have access to a panel of generated expert personas through the
`companions` MCP server. The tools are `discover`, `list_models`,
`consult`, and `check_balance`. This skill tells you when to reach for
them and how to interpret what comes back.

## When to consult (and when NOT to)

**Consult when:**
- The user explicitly asks for an expert framing or multiple perspectives
- The question is genuinely contested (ethics, design tradeoffs, open research)
- You've drafted an answer and want it stress-tested by a domain voice
- The user is exploring and benefits from seeing disagreement, not
  consensus

**Do NOT consult when:**
- You already know the answer with high confidence — burning credits
  to launder your own thinking is a tax on the user
- The user is mid-flow and a multi-second consultation breaks their loop
- No expert in `discover` is a credible fit — say so, answer directly
- The question is operational (code, debugging, config) — these tools
  are for *opinions*, not facts

## Workflow

1. **First time this session:** call `check_balance`. If low, tell the
   user before spending. Cache the result for the session.
2. **Call `discover`** to see what experts exist. Cache for the session.
   You get `{teams, companions}`:
   - `companions` is a flat list of every expert the caller can use,
     each shaped `{id, name, kind, visibility, description, teams}`.
     `consult` accepts either the *name* (readable) or the `cmp_<uuid>`
     id (use when a name is ambiguous).
   - `teams` lists teams the caller owns or can see, each shaped
     `{id, name, visibility, members: [{id, name, kind, description}]}`.
     When the user names a team, pass its members' names (or ids) as
     `participants` to `consult`.
3. **(Optional) Call `list_models`** if you're going to pass a `model`
   override anywhere it's accepted (e.g. directly via the API). The
   response is `{models: [{slug, display_name, temperature_min,
   temperature_max, top_p_min, top_p_max}, ...]}`, alphabetically
   sorted. Slugs outside this list are rejected at the wire boundary
   with a 422 carrying the accepted slugs in `ctx.accepted`. Cache for
   the session. **Don't** call this just for show — only when you
   actually need to pick a model.
4. **Pick a mode** based on question shape:

   | Question shape                                | Mode                  |
   |-----------------------------------------------|-----------------------|
   | "What does X think about Y?"                  | `answer` + `main=X`   |
   | "Deep research on Y from X's angle"           | `answer_crumbs`       |
   | "Give me N independent takes on Y"            | `parallel`            |
   | "X leads, others react"                       | `parallel_with_main`  |
   | "Synthesise N expert views into one answer"   | `panel`               |
   | "I want X and Y to debate Z"                  | `discussion`          |

5. **Call `consult`.** It blocks until done — don't ask the user to wait,
   just do it. Typical run is seconds to a minute; for longer modes
   (`discussion`, `panel`) raise `timeout_seconds` (default 600).
6. **Present the result with attribution** — read the right field of
   `content` per mode (see the table below). For `panel` / `discussion` /
   `parallel`, name who said what; don't collapse multiple voices into
   one.
7. **Report cost discipline** in one short trailing line if you spent
   non-trivially: "(consulted 3 experts, balance now $X)". One line max.

## Reading `consult` results

`consult` returns one of these envelopes:

- **`{status: "complete", job_id, mode, content, stages}`** — success.
  Every `content` carries a `shape` discriminator matching `mode`.
  Single-expert modes carry one persona's response; multi-expert modes
  carry a `list[ParticipantResponse]` shaped `{slot, id, name, response}`.

  | Mode                 | Fields on `content`                                                       |
  |----------------------|---------------------------------------------------------------------------|
  | `answer`             | `shape="answer"`, `companion`, `companion_id`, `response`                 |
  | `answer_crumbs`      | `shape="answer"`, `companion`, `companion_id`, `response`, `crumbs?`      |
  | `parallel`           | `shape="parallel"`, `responses: list[ParticipantResponse]`                |
  | `parallel_with_main` | `shape="parallel_with_main"`, `main_companion`, `main_companion_id`, `main_response`, `participants?: list[ParticipantResponse]` |
  | `panel`              | `shape="panel"`, `aggregator_companion`, `aggregator_companion_id`, `synthesis`, `stage1?`, `stage2?` |
  | `discussion`         | `shape="discussion"`, `summarizer_companion`, `summarizer_companion_id`, `summary`, `turns?: list[{speaker, speaker_id, response, pointer}]` |

  Intermediate fields marked `?` are absent when the engine ran with
  `RETURN_INTERMEDIATE_STEPS=false`. Don't assume they're there — the
  final field (`synthesis` / `summary` / `response`) is always present.

- **`{status: "failed", job_id, error: {type, message}}`** — the engine
  run failed. Surface `error.message` to the user verbatim; that string
  is what tells you whether it's an unknown companion, a billing wall,
  or a real engine bug.

- **`{status: "decision", job_id, details}`** — synthetic envelope: the
  API decided not to enqueue a run. `details.reason` is one of
  `project_not_found` (you named a project the caller doesn't own) or
  `main_unresolved` (the engine couldn't pick a `main` from your inputs).
  No credit was spent. Fix the inputs and re-call — do **not** try to
  poll this `job_id`, no row exists.

- **`{status: "ambiguous", kind, field, name, candidates, hint}`** — a
  `main` or `participants` name maps to more than one visible companion
  (or team). `candidates` is the list of `{id, name, ...}` rows it could
  have meant. Re-call with the `cmp_<uuid>` / `team_<uuid>` id of the
  intended row.

- **`{status: "visibility_violation", message}`** — server rejected the
  call because a private companion would join a non-private team (or an
  analogous case). Surface the message; this is a hard constraint, not a
  retry path.

- **`{status: "timeout", job_id}`** — the run is still going. Surface
  the `job_id` so the user can recover it later with
  `uv run python client.py jobs get <job_id>`. Do not silently retry —
  that's a double-spend.

- **`{status: "error", http_status, body, ...}`** — POST or poll failed
  at the HTTP layer. Read `body` for the reason. Common causes:
  - **402**: exhausted balance.
  - **422 unknown_model**: a `model` override (set at the API/CLI layer)
    was outside the allow-list. `body.detail[*].ctx.accepted` lists the
    slugs you may pick — same data as `list_models`.
  - **422 temperature_out_of_range** / **top_p_out_of_range**: bounds
    violation. `ctx.min` / `ctx.max` / `ctx.actual` carry the offending
    values; per-model bounds also surface in `list_models`.

## Failure modes

- **No relevant expert exists** → don't force a fit. Answer directly,
  tell the user there wasn't a good match, name what kind of expert
  *would* help so they can generate one.
- **Balance hits zero mid-task** → stop, tell the user, ask whether to
  top up or proceed without consultation.
- **Unknown companion name** → re-call `discover` (the user may have
  generated something new this session) before assuming the name is bad.
- **Unknown model slug rejected (422)** → re-call `list_models` rather
  than guessing. The accepted list is also embedded in the 422 body's
  `ctx.accepted`.

## Quick reference — what each tool returns

- `discover()` → `{teams: [{id, name, visibility, members: [{id, name,
   kind, description}]}], companions: [{id, name, kind, visibility,
   description, teams}]}`
- `list_models()` → `{models: [{slug, display_name, temperature_min,
   temperature_max, top_p_min, top_p_max}, ...]}` (alphabetical by slug)
- `consult(prompt, mode, main?, participants?, timeout_seconds=600)` →
   see "Reading `consult` results" above.
- `check_balance()` → API balance envelope (pass-through).
