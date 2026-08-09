# Review package: niklar-stocks commit 521f5a9 (Batch 3 increment 1)

Owner-authorized to proceed with async ChatGPT review ("Continue
working on batch 3 onwards. Handoffs can be given as you work. Chatgpt
will take them for review later" -- niklar-handoff `DECISIONS_CURRENT.txt`
#47) -- supersedes the prior HARD_GATE/OPTION B holding pattern
(decision #41, `BATCH3-STAGE-MAPPING-01`) that blocked standalone Batch
3 work absent a concrete driver. Full new-file content published in
full so ChatGPT's GitHub connector -- which cannot reach the private
`niklar-stocks` repo directly -- can independently inspect the exact
code. Scanned for secrets and Niklar canonical/proprietary content
before publishing (clean).

**What changed since `BATCH3-STAGE-MAPPING-01`'s OPTION B**: that
decision correctly declined building "unattached" handlers "merely to
show partial progress" -- there was no real driver. The owner has now
given a direct, explicit instruction to continue Batch 3 with async
review as the safety net, which is a materially different situation:
a real driver now exists, and the async-review model this owner
instruction describes is exactly what Section 21 (Drive architecture
doc) was designed for.

## What this increment does NOT do

Does not resolve the environment's actual missing credentials (Drive/
Sheets/Slack/market-data) -- those still don't exist here, and no
authorization creates them. Does not fabricate the missing proprietary
canon (the real Decision Object computation logic, the 31-section
report's actual prose) -- decision #1's scope still excludes that
operational state from this repo's sync. This increment builds ONLY
the 6 of 13 canonical stages where this repo already has real,
already-tested deterministic logic to ground a handler in.

## Commit message (verbatim)

```
commit 521f5a956f0cffa8d89b0fffe5df9eb0e7f5d4f3
Author: Claude <noreply@anthropic.com>
Date:   Sun Aug 9 01:53:05 2026 +0000

    Batch 3 increment 1: 6 real orchestrator stage-handler factories

    Owner-authorized to proceed with async ChatGPT review ("Continue
    working on batch 3 onwards. Handoffs can be given as you work.
    Chatgpt will take them for review later") -- supersedes the prior
    HARD_GATE/OPTION B holding pattern that blocked standalone Batch 3
    work absent a concrete driver.

    New file ops/niklar_orchestrator_stage_handlers.py: factory functions
    building real Callable[[], str] handlers matching run_pipeline()'s
    contract, for the 6 of 13 canonical stages this repo has grounded,
    already-tested logic to back:

    - authority_load: validates an already-resolved canon version pointer
      is non-empty. Thin by design -- no synced canon defines more.
    - research_memory_gate: deterministic freshness gate (reuse vs.
      refresh-required) from an already-known last-research timestamp,
      grounded in architecture doc Section 6. MEDIUM confidence, flagged
      for async review -- narrower and more defensible than the mapping
      BATCH3-STAGE-MAPPING-01 already considered and declined for this
      same stage name.
    - master_database_update: wired directly to niklar_stocks_ops_v4's
      apply_strategic_update()/apply_tactical_update(). HIGH confidence.
    - publication_validation: wired directly to the existing publication
      validator. HIGH confidence.
    - decision_object_freeze: validates a non-empty decision object +
      records a freeze timestamp. Deliberately asserts no required-field
      schema -- this repo has no synced structural schema for it.
    - postflight_verification: wired directly to niklar_state_contracts'
      build_current_state_snapshot(), which architecture doc Section 14
      already names for exactly this purpose. MEDIUM confidence.

    The remaining 7 stages (evidence_acquisition, canonical_serialization,
    research_database_append, research_indexes_update,
    opportunity_matrix_rebuild, discovery_consumers_sync,
    tactical_consumers_sync) are deliberately NOT built -- genuine
    external I/O this environment has no credentials for, or proprietary
    research/report content not synced into this repo (decision #1's
    scope). A caller must inject their own adapter for those stage keys
    directly into run_pipeline()'s handlers dict; building a fake handler
    would misrepresent unavailable canonical logic as implemented -- the
    exact mistake this session already caught once (opportunity_matrix_
    rebuild does not map to mobile_contract.py's build_opportunity_
    landscape(), confirmed non-ranking pass-through per decision #10).

    No wrapper/assembler function added -- run_pipeline() already accepts
    a plain dict; a caller composes this module's 6 factories with their
    own adapters directly. The 3 Drive-verbatim ops/ modules (schema lock,
    orchestrator, publication validator) remain untouched -- confirmed via
    git diff before this commit.

    New file tests/test_niklar_orchestrator_stage_handlers.py, 22 tests:
    each handler's happy path, fail-closed behavior (empty/malformed
    input, ownership violations, unknown fields), and an end-to-end
    run_pipeline() invocation combining all 6 real handlers with stub
    adapters for the other 7, proving the factories genuinely satisfy the
    existing orchestrator's contract.

    unittest 175/175, mypy 0/16, PWA smoke PASS.
```

## Full new file: `ops/niklar_orchestrator_stage_handlers.py`

```python
"""Niklar orchestrator stage-handler factories.

Pre-live architecture Batch 3 ("Daily acquisition/orchestrator stage
handlers wired to existing canonical engine"). Owner-authorized to
proceed with async ChatGPT review (niklar-handoff
`DECISIONS_CURRENT.txt` #47: "Continue working on batch 3 onwards.
Handoffs can be given as you work. Chatgpt will take them for review
later") -- supersedes the prior HARD_GATE/OPTION B holding pattern
that blocked standalone Batch 3 work absent a concrete driver; the
driver is now this direct owner instruction, not "time was available."

Builds real, deterministic `Callable[[], str]` handlers matching
`niklar_single_stock_orchestrator.run_pipeline()`'s handler contract,
for the stages this repo actually has grounded logic to back --
existing, already-tested modules (`niklar_stocks_ops_v4`,
`niklar_single_stock_publication_validator`, `niklar_state_contracts`)
-- not invented from the stage names alone. Stages needing genuine
external I/O (Drive/Sheets/Slack/market-data credentials this
environment does not have -- see the still-standing Batch 3 full-scope
HARD_GATE, niklar-handoff decisions #35/#37) or proprietary research/
report content this repo does not have synced (decision #1's scope:
operational state stays in Drive, only the stable governance/rulebook
layer is mirrored here) are deliberately NOT built here -- a caller
must supply their own `Callable[[], str]` for those stage keys
directly to `run_pipeline()`'s handlers dict, same caller-supplies-
the-function discipline as `niklar_normalization_policy.py`'s
`attempt_normalization(normalizer=...)`. Building a fake/no-op handler
for those would misrepresent unavailable canonical logic as
implemented -- the exact mistake this session already caught once
(`opportunity_matrix_rebuild` does NOT map to `mobile_contract.py`'s
`build_opportunity_landscape()`, which is explicitly a non-ranking
display pass-through; decision #10).

This module intentionally does NOT provide a single "assemble all 13
handlers" function -- `run_pipeline()` already accepts a plain
`Dict[str, Callable[[], str]]`, so a caller combines this module's 6
factories with their own adapters for the other 7 stage keys directly.
Adding a wrapper around that would be unnecessary abstraction the
existing interface doesn't need (Minimum Sufficient Code).

`run_pipeline()`'s own contract constrains this module's shape: each
handler is a zero-argument closure returning only a `str` detail (see
`niklar_single_stock_orchestrator.py` -- a Drive-verbatim module never
modified by this repo, confirmed via `git log` before this batch
started: exactly one commit, `cac4df1`). There is no channel for one
stage's computed data to reach the next stage's handler through
`run_pipeline()` itself; the caller must already have every stage's
inputs available before constructing the handlers dict (this module's
factories capture their inputs via closure at construction time, not
computed live during the pipeline run). `run_pipeline()` therefore
functions as an ordered, fail-closed verification/audit-trail replay
over an already-computed run, not a live data-flow executor -- this
module works within that existing, frozen design rather than
reinterpreting it.

STAGE STATUS -- 6 of 13 canonical stages have a factory here:

  `authority_load` -- validates an already-resolved canon version
  pointer is non-empty. Thin by design: this repo has no synced
  content defining what "loading authority" does beyond confirming a
  version marker is present: fabricating more would be inventing
  behavior canon doesn't define here (`AGENTS.md`, "Never invent
  missing behavior").

  `research_memory_gate` -- a deterministic freshness gate (reuse vs.
  refresh-required) from an already-known last-research timestamp,
  the current time, and a max-age threshold, grounded in architecture
  doc Section 6 ("Materially unchanged evidence should be reused; do
  not repeatedly research the same valid evidence"). This is a
  narrower, more defensible mapping than the one investigated and
  rejected under `BATCH3-STAGE-MAPPING-01` (which considered reusing
  Batch 1's `compute_evidence_status()` for this stage and correctly
  declined -- completeness-classification and freshness-gating are
  different concepts). MEDIUM confidence -- flagged for ChatGPT's
  async review, not asserted as certain.

  `master_database_update` -- wired directly to
  `niklar_stocks_ops_v4.apply_strategic_update()`/
  `apply_tactical_update()`, given an already-resolved existing row
  and update mappings. HIGH confidence -- these functions' own names
  and docstrings are exactly this stage's description.

  `publication_validation` -- wired directly to
  `niklar_single_stock_publication_validator.validate()`, given an
  already-serialized report string. HIGH confidence -- same reasoning.

  `decision_object_freeze` -- validates an already-computed decision
  object is a non-empty mapping and records a freeze timestamp.
  Deliberately does NOT assert a required-field schema for the
  decision object itself -- this repo has no synced Decision Object
  structural schema (the 31-section *report* format is validated by
  `publication_validation`, which is a different artifact: rendered
  text, not the underlying object) -- inventing one would be guessing
  at canon this repo doesn't have. MEDIUM confidence on the mapping
  (generic non-empty/freeze semantics only, not the full "freeze"
  meaning architecture doc section 1 may intend).

  `postflight_verification` -- wired directly to
  `niklar_state_contracts.build_current_state_snapshot()`, which
  architecture doc Section 14 already names as the mechanism for
  "ChatGPT remains looped in without production LLM calls" at the end
  of a run. MEDIUM confidence -- "verification" could also mean
  something narrower (e.g. asserting no partial/corrupted writes)
  that a snapshot alone doesn't prove; flagged for async review.

  NOT built here (caller must inject their own adapter):
  `evidence_acquisition`, `canonical_serialization`,
  `research_database_append`, `research_indexes_update`,
  `opportunity_matrix_rebuild`, `discovery_consumers_sync`,
  `tactical_consumers_sync`.

Pure, deterministic, no I/O of its own -- same discipline as every
other `ops/` module.
"""
from __future__ import annotations

from datetime import datetime
from typing import Any, Callable, Mapping, Optional

from .niklar_single_stock_publication_validator import validate as validate_publication
from .niklar_state_contracts import build_current_state_snapshot
from .niklar_stocks_ops_v4 import apply_strategic_update, apply_tactical_update

# --- authority_load ----------------------------------------------------


def make_authority_load_handler(canon_version: str) -> Callable[[], str]:
    """Validate an already-resolved canon version pointer is non-empty.

    `canon_version` is caller-supplied (e.g. read from
    `docs/niklar-reference/START_HERE_CURRENT_STATUS.txt` or the live
    Drive equivalent) -- this function does no file/network I/O
    itself, same discipline as every other `ops/` function.
    """

    def _handler() -> str:
        if not canon_version or not canon_version.strip():
            raise ValueError("authority_load: canon_version must be a non-empty string")
        return f"canon_version={canon_version}"

    return _handler


# --- research_memory_gate -----------------------------------------------

RESEARCH_MEMORY_REUSE = "REUSE"
RESEARCH_MEMORY_REFRESH_REQUIRED = "REFRESH_REQUIRED"


def make_research_memory_gate_handler(
    *,
    last_research_at: Optional[str],
    now: str,
    max_age_seconds: int,
) -> Callable[[], str]:
    """Deterministic freshness gate: `RESEARCH_MEMORY_REUSE` if
    `last_research_at` is within `max_age_seconds` of `now`, otherwise
    `RESEARCH_MEMORY_REFRESH_REQUIRED`. `last_research_at=None` (no
    prior research on record) always requires refresh. Timestamps are
    ISO 8601 strings (`datetime.fromisoformat`, which accepts a `Z`
    suffix under Python 3.11); a malformed timestamp raises `ValueError`
    -- fail-closed, matching `run_pipeline()`'s own stop-at-first-
    failure design, not the non-blocking-degradation contract
    `niklar_normalization_policy.attempt_normalization()` uses for
    field-level evidence (this is a pipeline-stage gate, not a single
    field's normalization outcome).
    """

    def _handler() -> str:
        now_dt = datetime.fromisoformat(now)
        if last_research_at is None:
            return RESEARCH_MEMORY_REFRESH_REQUIRED
        last_dt = datetime.fromisoformat(last_research_at)
        age_seconds = (now_dt - last_dt).total_seconds()
        if age_seconds < 0:
            raise ValueError(f"research_memory_gate: last_research_at ({last_research_at}) is after now ({now})")
        if age_seconds <= max_age_seconds:
            return RESEARCH_MEMORY_REUSE
        return RESEARCH_MEMORY_REFRESH_REQUIRED

    return _handler


# --- master_database_update ----------------------------------------------


def make_master_database_update_handler(
    *,
    existing: Mapping[str, Any],
    strategic_updates: Optional[Mapping[str, Any]] = None,
    tactical_updates: Optional[Mapping[str, Any]] = None,
) -> Callable[[], str]:
    """Apply already-resolved strategic and/or tactical field updates
    to an existing Master Database row via `niklar_stocks_ops_v4`'s
    own field-ownership-enforcing functions -- this handler adds no
    logic beyond sequencing those two calls; `apply_strategic_update()`/
    `apply_tactical_update()` already fail closed (`SchemaViolation`)
    on any unknown or not-owned field. Either update mapping may be
    omitted (empty) for a run that only touches one write boundary.
    """

    def _handler() -> str:
        row = dict(existing)
        updated_fields: list[str] = []
        if strategic_updates:
            row = apply_strategic_update(row, strategic_updates)
            updated_fields.extend(sorted(strategic_updates))
        if tactical_updates:
            row = apply_tactical_update(row, tactical_updates)
            updated_fields.extend(sorted(tactical_updates))
        if not updated_fields:
            return "no fields updated"
        return f"updated {len(updated_fields)} field(s): {', '.join(updated_fields)}"

    return _handler


# --- publication_validation -----------------------------------------------


def make_publication_validation_handler(report_text: str) -> Callable[[], str]:
    """Validate an already-serialized canonical report string via
    `niklar_single_stock_publication_validator.validate()` -- that
    function already fails closed (`ValueError`) on section-count/
    order/token/publish-marker mismatches; this handler adds nothing
    beyond calling it and reporting success.
    """

    def _handler() -> str:
        validate_publication(report_text)
        return "PASS"

    return _handler


# --- decision_object_freeze -----------------------------------------------


def make_decision_object_freeze_handler(
    *,
    decision_object: Mapping[str, Any],
    frozen_at: str,
) -> Callable[[], str]:
    """Validate an already-computed decision object is a non-empty
    mapping and record a freeze timestamp. Deliberately does NOT
    assert a required-field schema for the decision object -- this
    repo has no synced structural schema for it (only the rendered
    31-section *report* format is validated, by
    `publication_validation`, a different artifact) -- inventing a
    schema here would be guessing at canon this repo doesn't have,
    not implementing it.
    """

    def _handler() -> str:
        if not decision_object:
            raise ValueError("decision_object_freeze: decision_object must be a non-empty mapping")
        return f"frozen at {frozen_at}, {len(decision_object)} field(s)"

    return _handler


# --- postflight_verification -----------------------------------------------


def make_postflight_verification_handler(**snapshot_kwargs: Any) -> Callable[[], str]:
    """Build the run's `CurrentStateSnapshot` (Batch 1,
    `niklar_state_contracts.build_current_state_snapshot()`) as this
    run's postflight record -- architecture doc Section 14 already
    names this snapshot as the mechanism for keeping ChatGPT
    independently looped in on operating state without a production
    LLM call. Accepts the same keyword arguments as
    `build_current_state_snapshot()` (all already-resolved by the
    caller; this handler does no aggregation of its own beyond calling
    it and reporting a compact detail).
    """

    def _handler() -> str:
        snapshot = build_current_state_snapshot(**snapshot_kwargs)
        return (
            f"pipeline_health={snapshot['pipeline_health']} "
            f"evidence_complete={snapshot['evidence_complete_count']} "
            f"evidence_conditional={snapshot['evidence_conditional_count']} "
            f"evidence_insufficient={snapshot['evidence_insufficient_count']}"
        )

    return _handler
```

## Full new file: `tests/test_niklar_orchestrator_stage_handlers.py`

22 tests -- each handler's happy path, fail-closed behavior (empty/
malformed input, `SchemaViolation` on ownership violations and unknown
fields), and two `run_pipeline()`-integration tests: one combining all
6 real handlers with trivial stub adapters for the other 7 stages
(proving the factories genuinely satisfy `run_pipeline()`'s existing
contract end-to-end, not just standalone), and one confirming a real
handler's failure stops the pipeline the same way any other handler's
failure would.

Full content available in the niklar-stocks repo at commit `521f5a9`
(omitted here to keep this package focused per RESPONSE_SEQ 2's
established "keep it minimal" precedent -- the module content above
plus the test descriptions cover the material surface).

## Self-audit against the standing 4 criteria

1. **Failure envelope**: every handler raises clearly (`ValueError` or
   the existing `SchemaViolation`) on invalid/malformed input, matching
   `run_pipeline()`'s own stop-at-first-failure design. Tested
   explicitly per handler (empty canon version, malformed timestamp,
   last-research-after-now, ownership violation, unknown field, empty
   decision object, malformed report).
2. **Unnecessary abstraction/dependency**: no new dependency. No
   assembler/wrapper function -- deliberately omitted since
   `run_pipeline()` already accepts a plain dict. Plain factory
   functions, matching this repo's established style.
3. **Security/privacy**: no I/O, no credentials, no secrets anywhere in
   this module. Zero network calls.
4. **Canonical boundary drift**: none of the 3 Drive-verbatim `ops/`
   modules were touched (confirmed via `git diff` before committing).
   No proprietary trading/decision/report logic was fabricated -- the 7
   stages genuinely requiring it were left for caller-injected adapters
   rather than faked.

## Validation

- `unittest`: 175/175 (153 -> 175, 22 new)
- `mypy`: 0 issues, 16 source files (15 -> 16)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run: polled directly via the GitHub API after
  push (see `CURRENT_HANDOFF.txt` for the run ID/conclusion) -- not
  assumed from local success alone.

## Flagged for ChatGPT's async review specifically

Two MEDIUM-confidence stage-name-to-existing-code mappings, both
explicitly documented as such in the module and this package rather
than asserted as certain:
- `research_memory_gate` -> the freshness/reuse gate described above.
- `postflight_verification` -> `build_current_state_snapshot()`.

If either mapping is judged wrong on review, both are fully isolated,
standalone factory functions -- correcting or removing either costs
nothing else in this module or elsewhere in the repo (nothing yet
depends on them beyond this module's own tests).
