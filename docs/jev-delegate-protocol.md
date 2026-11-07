# Jev delegate protocol v0.1 (cross-implementation contract)

`jev_browser_run` (Rust, VOM) and `jev_desktop_run` (Node, macOS AX) were built
independently and both passed real Jev API end-to-end runs. Their shared part is
**the decision contract and its invariants**, not code: observation parsing,
RPC shape and native action APIs are completely different. A cross-language
"shared SDK" would only be a thin wrapper with no reuse value, so the shared
artefact is a spec plus executable conformance vectors.

The canonical spec lives in the integration workspace
(`jev-integration/protocol/SPEC.md`). This document is the BrowserSkill-side
summary and the CI entry point.

## What is shared

| Rule | Invariant |
| --- | --- |
| One request, many heads | `operation`, `click_target`, `type_text_target` are asked in parallel; only the head matching the chosen operation is consumed. |
| Full distributions | `choice` must be an offered id; `probabilities` must cover every offered id, each in `[0,1]`, sum in `1±0.02`, and `choice` must be the maximum. Otherwise `needs_llm / invalid_choice`, never a guess. |
| Freshness | An `@eN` ref is bound to the `context_id` / `observation_id` that produced it. A target absent from the current observation is refused before execution. |
| Retry policy | Read-only `sync` may retry (`document_changed` navigation race, up to 6 attempts); an executed action is never replayed. |
| Completion is code-owned | `verified_done` requires the caller's `expected` literal in URL/title or **non-control** VOM text, newly appearing after at least one action. A control's own value, pre-existing evidence and a model `DONE` never count. |
| Response | `status`, `reason`, `message`, `goal`, `expected`, `steps`, `model_calls`, `elapsed_ms`, `trace`, `post`. |

Known differences (VOM vs AX, `BSK_JEV=1` vs `OCU_JEV=1`, `SCROLL_DOWN` vs
deterministic `query→action`, `require_controls`) stay in each adapter; the
core contract is identical. A third backend should add an adapter and run the
same vectors, not rewrite the core.

## Vectors and CI

`crates/bsk-cli/tests/fixtures/jev-protocol/vectors.json` is a byte-identical
copy of the canonical `protocol/vectors.json`. Seven decision moments are
covered: `two_step_form`, `result_page_evidence`,
`control_value_is_not_evidence`, `preexisting_evidence`,
`partial_probabilities`, `stale_target`, `done_unverified`.

`jev_protocol_vectors_are_satisfied` in `crates/bsk-cli/src/cli/mcp/jev.rs`
rebuilds a real `Observation` from each vector's VOM text, calls the production
`questions()` / `choose()` / `evidence_text()` / `verified()` functions, and
asserts `heads`, `operation`, `target`, `decision_error` and `verified`.

```sh
cargo test -p bsk --lib jev_protocol_vectors --locked
```

To check both implementations at once (this also refreshes both fixture copies):

```sh
node jev-integration/protocol/conformance.mjs
```

Measured locally: `CONFORMANCE: PASS (7 vectors, 2 implementations)`.
Adding or changing a vector requires the same change on the OCU side and a
re-run of the cross-language runner; the runner fails if the two copies diverge
from the canonical file.

## Scope

The protocol does not cover screenshots, OCR, coordinate actions or the LLM ↔
tool transport (MCP stdio is an implementation detail). v0.1 vectors cover
decision and completion invariants only; loop timeouts, cancellation and cost
accounting remain in each implementation's own tests and in the M3 comparison
report.
