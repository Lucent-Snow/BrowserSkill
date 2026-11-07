# Jev delegate: local integration run (2026-11-07)

Branch `jev-integration` of `Lucent-Snow/BrowserSkill` (fork of `Tencent/BrowserSkill`).
This is the M1 experiment of the Jev GUI tool integration plan: one new
LLM-callable MCP tool on top of BrowserSkill. M2–M4 were completed later in the
same work session; see "Follow-up (M2–M4)" at the end of this note.

## Baseline

| Repository | Local path (relative to workspace root) | Commit |
| --- | --- | --- |
| `Tencent/BrowserSkill` fork | `jev-integration/BrowserSkill` | `7e862a4` (2026-11-06) |
| `browser-use/jev-ultrafast` (read-only reference) | `jev-integration/refs/jev-ultrafast` | `8251e27` (2026-10-31) |
| `iFurySt/open-computer-use` (read-only reference) | `jev-integration/refs/open-computer-use` | `613730f` (2026-11-07) |

## What changed

- New `crates/bsk-cli/src/cli/mcp/jev.rs`: the observe/decide/act loop, the
  parallel operation+target Jev request, choice validation, native action
  binding, independent completion check and HTTP client (proxy-aware).
- `crates/bsk-cli/src/cli/mcp/tools.rs` + `mod.rs`: register `jev_browser_run`
  behind `BSK_JEV=1`; the original seven tools are untouched.
- `docs/jev-delegate.md`, `docs/mcp.md`, both CHANGELOGs.

Reused rather than rewritten: `recipe::vom::parse` (typed VOM lines),
`cooperate::settle::run` (read-only sync), `Link`/`CallControl` (daemon RPC and
cancellation), and the extension's own `context_id` / `observation_id` /
`@eN` guards. No protocol schema, daemon or extension change.

## Environment

- macOS, Node 22, Rust 1.95, pnpm, `cargo build -p bsk --locked`,
  `pnpm install --frozen-lockfile && pnpm ext:build`.
- Chrome for Testing 154 (headless), unpacked extension from
  `apps/extension/dist/chrome-mv3`, isolated `BSK_HOME` + profile,
  `BSK_AUTO_START=0`, `BSK_AUTO_UPDATE=off`, `BSK_JEV=1`.
- Jev endpoint `https://api.typesafe.ai/v1/systemone`, model reported as
  `jev-1.14.0`, key from the developer's local secrets store passed as
  `TYPESAFE_API_KEY`, HTTPS through the local proxy (a direct request returns
  HTTP 451 in this region).
- Fixture: `evals/browser` `/form` → `/result`, whose result page shows
  `Received!`. Harness: `jev-integration/e2e_run.py` (outside the repository).

## Measured result (final run)

`jev_browser_run({session, goal, expected:"Received!", text:"cats", max_steps:6})`

| Metric | Value |
| --- | --- |
| status / reason | `verified_done` / `expected_observed` |
| Executed steps | 2 (`TYPE_TEXT @e1`, `CLICK @e4`) |
| Jev model calls | 2 |
| Jev latency | 1663 ms, 1884 ms |
| Jev tokens | 1803 in / 136 out; 1842 in / 135 out |
| Confidence | 0.98, 0.95 |
| End-to-end wall time | 3943 ms |
| Final page | `RootWebArea "Form Result"`, `heading "Received!"`, `paragraph "cats \| \| one"` |

`tools/list` contained the seven original tools plus `jev_browser_run`.
Offline contracts: 7 passed, 1 ignored (paid API); `cargo clippy -p bsk
--all-targets --locked -- -D warnings` clean; explicit live smoke
(`--ignored`) returned `jev-1.14.0` / `TYPE_TEXT` / `e1` in 1769 ms.

## Two defects found by measuring, and their fixes

1. **False completion after one step.** The first run used `expected:"cats"`;
   the VOM line `@e1 textbox ... ="cats"` matched, so a filled input was
   reported as `verified_done` with only one action. Completion evidence is now
   restricted to URL/title and non-control VOM lines, and must be new since the
   previous observation. Locked in by
   `a_typed_value_alone_is_not_completion_evidence`.
2. **Handback during navigation.** After the Submit click, the next sync returned
   `Document changed during observation; observe again` and the run stopped at
   `needs_llm / sync_failed`. Read-only sync is now retried up to six times; the
   mutation is never replayed. Locked in by
   `a_transient_sync_gap_is_retried_without_replaying_an_action`.

Both were real behaviour of the integrated system, not unit-test artifacts.
The LLM-side goal, text and completion evidence remain the caller's job.

## Boundaries at the end of M1

- M1 only at this point. No `SELECT`, no screenshot/OCR/visual coordinates, no
  script execution, no macOS AX experiment (M2) and no LLM-only baseline
  comparison (M3). The OCU repository was pulled for M0 reading only; it has not
  been forked or modified **as of this M1 snapshot**.
- A fair M3 baseline must let the LLM use `bsk_plan` and other batching tools;
  "one LLM call per step" is not a valid baseline. Compare success rate,
  end-to-end latency, total model cost, model/tool calls, takeovers and
  adaptation code size on the same backend.
- `page_meta` cannot always obtain URL/title from a sync reply, so the
  observation's first lines are placed in the decision instructions as the page
  anchor. Completion does not depend on URL/title.
- The tool adds an outbound HTTP dependency to `bsk mcp`. A decision failure
  leaves the page untouched; secrets are read from the environment only.

## Follow-up (M2–M4, completed 2026-11-07)

**M2 — desktop delegation.** Forked `iFurySt/open-computer-use` to
`Lucent-Snow/open-computer-use@jev-desktop` and added `OCU_JEV=1` gated
`jev_desktop_run` in the Node REPL layer (no Swift changes). AX-only: no
screenshots, OCR or coordinates; `setValue` / `click` / `scroll` stay native.
Real Jev end-to-end on the headless fixture took 2 steps / 2 model calls /
3379 ms, with an independent AX re-read confirming `Counter: 1` and `cats`.
A real TCC read-only snapshot (720 localized AX lines) parsed into the 60
finite candidates the request actually offers. `swift test` has an unrelated
upstream Swift 6.4 compile failure in `LockedUseIncidentReplayTests.swift`;
it was not caused by, and was not silently fixed by, the Node change.

**M3 — fair same-backend comparison.** `jev-integration/m3_compare.py` +
`m3_report.json`; readable summary in `jev-integration/protocol/m3-summary.md`.
12/12 runs succeeded with `gpt-5.6-luna` on one gateway. BSK baseline
3.3 LLM calls / 12191 tokens / 18173 ms vs delegated 1 call / 596 LLM tokens /
5186 total model tokens / 9600 ms. OCU baseline 2 calls / 2589.7 tokens /
8413.7 ms vs delegated 1 call / 721 LLM tokens / **5026 total model tokens** /
7470.3 ms. The important negative result: delegation reduces LLM calls and
latency on both backends, but on OCU it roughly doubles total model tokens
because Jev must carry the full AX state on every step. The gateway exposes no
cost field, so tokens were only a proxy. Batch-capable baselines
(`bsk_plan`, persistent `js` REPL) were used deliberately; a one-LLM-call-per-step
baseline would have been a straw man.

**Correction added later the same day (T1 cost measurement).** Treating total tokens as a
cost proxy is misleading when two differently priced models are involved. With published
list prices (LLM $0.20/1M input, $1.20/1M output; Jev $0.042/1M input, output free;
1 USD = 6.6713 CNY), a fresh 20-run measurement of one fixed task found delegation
cheaper on BOTH backends: BSK 0.167x the cost (83.3% less, 39.9% less wall time), OCU
0.599x the cost (40.1% less, 12.5% less wall time) even though OCU used 1.95x the tokens.
Three pricing scenarios agree. See `jev-integration/protocol/m3-summary.md` and
`cost_report.json` in the developer workspace. The gateway still returns no invoice;
the LLM short id is priced at the identically named public model's list rate.

**M4 — protocol, not a shared SDK.** Rust/VOM and JS/AX share a decision
contract, not runtime code. Deliverables: canonical
`jev-integration/protocol/SPEC.md`, 7 executable vectors in
`protocol/vectors.json`, and `protocol/conformance.mjs`. This crate now holds a
byte-identical CI copy at
`crates/bsk-cli/tests/fixtures/jev-protocol/vectors.json` and the Rust test
`jev_protocol_vectors_are_satisfied` checks production `questions()` /
`choose()` / `evidence_text()` / `verified()` against them. OCU holds its own
copy and `jev-protocol.test.mjs`. Local result:
`CONFORMANCE: PASS (7 vectors, 2 implementations)` (Rust 1 test, Node 8 tests).
See `docs/jev-delegate-protocol.md`.

Polish and ablation experiments were explicitly deferred by the user until all
planned stages are complete; the ablation switches (`deterministic`,
`require_controls`, retry counts, candidate limits) are recorded in the M3
summary but have not been run yet.
