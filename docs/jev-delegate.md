# Jev delegate (`jev_browser_run`)

An experimental MCP tool that lets an LLM delegate a bounded browser loop to a
[TypeSafe Jev](https://api.typesafe.ai/v1/systemone) decision model. It is off by
default: set `BSK_JEV=1` in the `bsk mcp` server environment to add the tool to
`tools/list`.

## Division of labour

| Owner | Responsibility |
| --- | --- |
| Calling LLM | Understands the task, sets `goal`, supplies `expected` and `text`, decides to delegate or take over, handles `needs_llm`. |
| Jev | Given the current observation and a finite candidate set, chooses one operation and one target. Never produces selectors, coordinates, scripts or field text. |
| This tool | Observes through BrowserSkill, builds candidates, binds `@eN` refs to the current context, executes one native action, re-observes, checks completion, records the trace. |
| BrowserSkill extension | Owns the DOM refs, `context_id` / `observation_id` freshness, guarded-action checks and the real browser mutation. |

This is the *embedded* form of a decision model: it enhances an existing LLM
agent tool rather than replacing the agent. The original seven MCP tools remain
available for direct use, which is what makes a fair LLM-only baseline possible.

## Tool arguments

```json
{
  "session": "<session_id from bsk_session start>",
  "goal": "Search for cats and open the results",
  "expected": "Results for cats",
  "text": "cats",
  "max_steps": 8,
  "tab_id": 7
}
```

- `expected` is a literal URL, title or **static page-text** fragment. Code checks
  it after at least one executed action and requires it to be newly visible since
  the previous observation. A value typed into a control (`@eN ... ="cats"`) is
  not evidence: VOM control lines are excluded, because a filled field proves
  only that text was entered, not that the goal was reached. Pick a fragment that
  only the result page shows.
- `text` is the exact value for a `TYPE_TEXT` action. If Jev chooses `TYPE_TEXT`
  and no text was supplied, the tool returns `needs_llm` (`text_required`)
  without sending any mutation. Jev never invents field values.
- `max_steps` bounds observe/decide/act turns (1–20, default 8).

## Loop contract

1. `bsk_session sync` semantics via `cooperate::settle::run`; the reply's
   `context_id`, top-level `observation_id` and VOM text are captured. An
   unchanged page without VOM text falls back to `tool.snapshot`.
2. `recipe/vom.rs` parses the VOM text into typed lines. Only lines carrying an
   `@eN` ref become candidates; `canvas` is excluded. Editable roles get both
   `TYPE_TEXT` and `CLICK`.
3. One Jev request carries an `operation` head plus one target head per legal
   operation (`click_target`, `type_text_target`), following the parallel-Choice
   arrangement in `browser-use/jev-ultrafast`. Only the target head matching the
   chosen operation is consumed; the others are ignored.
4. The chosen index maps back to the native `@eN` ref. The action is sent as the
   same `tool.click` / `tool.fill` / `tool.wheel` RPC an ordinary `bsk` command
   would send, with `context_id`, `observation_id` and `tab_id` attached.
   `WAIT` performs no RPC. A mutation is never retried.
5. Every turn re-syncs. Old refs are never reused. A choice response that fails
   `validate_choice` (probabilities covering exactly the offered ids, summing to
   one, chosen id maximal, confidence in range) stops the run before any action.
6. A `document_changed` reply during navigation is a read-only sync race. The
   sync is retried up to six times with a short delay; the mutation that caused
   the navigation is never replayed.
7. Completion is checked by code, never by a model `DONE`: `expected` must appear
   in the URL/title or in a non-control VOM line, and must be new since the prior
   observation. This is what stops a filled input from being reported as a
   finished task.
8. Three consecutive actions that leave the VOM unchanged stop the run with
   `needs_llm` (`no_progress`), mirroring the ultrafast agent's stall guard.

## Returned status

| `status` | Meaning | LLM action |
| --- | --- | --- |
| `verified_done` | `expected` is visible after an executed action | Answer using `post` and `trace`. |
| `needs_llm` | `done_unverified`, `no_progress`, `action_failed`, `sync_failed`, `text_required`, `decision_failed`, `cancelled`, `invalid_choice`, `sync_unavailable` | Read `post` / `trace`, then continue with the native tools. |
| `blocked` | Jev saw no supported operation | Take over. |
| `budget_exhausted` | `max_steps` reached | Take over or re-delegate with a larger budget. |

`DONE` from the model alone yields `needs_llm` (`done_unverified`), never
`verified_done`. `post.observation.text` is returned as a separate MCP text
block, so the LLM can resume from a fresh observation. High-consequence actions
are still subject to the session's own control policy and guards.

## Configuration

| Variable | Purpose |
| --- | --- |
| `BSK_JEV=1` | Adds the tool to `tools/list`. |
| `TYPESAFE_API_KEY` (or `JEV_KEY`) | Bearer token for the Jev endpoint. |
| `JEV_URL` | Overrides the default `https://api.typesafe.ai/v1/systemone`. |
| `HTTPS_PROXY` / `https_proxy` | Explicit proxy for the Jev request. Without a reachable proxy some regions receive HTTP 451, which must not be mistaken for a dead API. |

The MCP server writes protocol messages only to stdout; this tool sends nothing
there and logs no secrets.

## Scope and boundaries

- First version supports `CLICK`, `TYPE_TEXT`, `SCROLL_DOWN`, `WAIT`, `DONE`,
  `BLOCKED`. No `SELECT`, no visual/screenshot/OCR path, no script execution.
- Jev sees page text and element labels. Page content is untrusted data and is
  never treated as instructions.
- The tool adds an outbound HTTP dependency to `bsk mcp`. A decision failure
  never causes an action retry and leaves the page untouched.
- Comparison with direct LLM tool use (success rate, end-to-end latency, model
  cost, tool/model call counts, takeovers, adaptation code size) is a separate
  measurement step. The LLM-only baseline must be allowed to use `bsk_plan` and
  other batching tools; "one LLM call per step" is not a fair baseline.

## Tests

`cargo test -p bsk jev` runs offline contracts with a stub decider and a fake
daemon. No test calls a paid model API. They cover one-request/multi-head
construction, invalid choice rejection, native ref and context binding,
independent completion verification, and the no-mutation path when text is
missing.

## Local verification

Measured on 2026-11-07 with real headless Chrome for Testing, the built
extension, the real Jev API (`jev-1.14.0`) and the local `/form` fixture:
`jev_browser_run` filled the Text input and clicked Submit in two Jev turns and
returned `verified_done` after the result page showed `Received!`. Wall time
3943 ms, two model calls (1663 / 1884 ms), 1803+1842 input tokens. Seven offline
contracts pass, one paid-API smoke is ignored by default, and
`cargo clippy -p bsk --all-targets --locked -- -D warnings` is clean.

The harness, full trace, and the two defects this measurement exposed (a typed
value being mistaken for completion evidence, and a navigation-race handback)
are recorded in `.agents/notes/jev-integration.md`.
