# Review package: niklar-stocks commit aca2a86 (Batch 3 increment 2)

Continues the owner-authorized Batch 3 work (decision #47) with async
review, per "Handoffs can be given as you work. Chatgpt will take them
for review later." Adds one more real stage handler
(`evidence_acquisition`) to the module published in increment 1's
package (`521f5a9-batch3-increment1-stage-handlers.md`). Full diff
published so ChatGPT's GitHub connector -- which cannot reach the
private `niklar-stocks` repo directly -- can independently inspect the
exact code. Scanned for secrets and Niklar canonical/proprietary
content before publishing (clean).

## What this adds

`make_evidence_acquisition_handler()` -- the 7th of 13 canonical
stages to get a real, grounded handler (was 6 after increment 1).
Acquires one evidence field via a caller-injected fetch adapter (the
actual Drive/news/market-data call, which this environment has no
credentials for -- honestly deferred, not fabricated), then normalizes
the fetched value through Batch 2's own `attempt_normalization()`
(already independently reviewed and PASS-frozen, commit 02b6f88).

**Why this is a real mapping, not just "acquisition alone with the
hard part deferred"**: architecture doc section 1 states "Research
evidence must be normalized before consumption by the deterministic
engine," and there is no separate named stage for normalization among
the 13 canonical stages -- the orchestrator's own pipeline docstring
reads "evidence acquisition -> Decision Object" with nothing named
"normalization" between them. `evidence_acquisition` is the stage that
owns both acquire and normalize; only the acquisition mechanism itself
(the actual network/Drive call) is genuinely unavailable here.

**Two distinct failure modes, handled per the architecture doc's own
distinction**:
- `fetch_adapter()` raising -- a genuine acquisition failure (network/
  credential/upstream problem). Propagates and stops the pipeline,
  matching `run_pipeline()`'s own fail-closed design -- a real stage
  failure, not a soft evidence-quality issue.
- The fetched value failing to normalize -- architecture doc section 3
  ("non-blocking evidence degradation... must degrade confidence, not
  stop Niklar") is explicit this must NOT stop the pipeline. The
  handler does not raise in that case; it returns successfully with a
  "degraded" detail, mirroring `attempt_normalization()`'s own
  non-blocking contract instead of overriding it with
  `run_pipeline()`'s stricter default.

## Diff

```diff
--- a/ops/niklar_orchestrator_stage_handlers.py
+++ b/ops/niklar_orchestrator_stage_handlers.py
@@ imports @@
+from .niklar_normalization_policy import NORMALIZATION_SUCCESS, attempt_normalization
 from .niklar_single_stock_publication_validator import validate as validate_publication
 from .niklar_state_contracts import build_current_state_snapshot
 from .niklar_stocks_ops_v4 import apply_strategic_update, apply_tactical_update

@@ new section, between research_memory_gate and master_database_update @@
+# --- evidence_acquisition -----------------------------------------------
+
+
+def make_evidence_acquisition_handler(
+    *,
+    fetch_adapter: Callable[[], Any],
+    normalizer: Callable[[Any], Any],
+    ticker: str,
+    field: str,
+    attempted_at: str,
+) -> Callable[[], str]:
+    """Acquire one evidence field via a caller-injected `fetch_adapter`
+    ... (see module docstring for full rationale) ...
+    """
+
+    def _handler() -> str:
+        raw_value = fetch_adapter()
+        result = attempt_normalization(
+            normalizer=normalizer,
+            raw_value=raw_value,
+            ticker=ticker,
+            field=field,
+            attempted_at=attempted_at,
+        )
+        if result["status"] == NORMALIZATION_SUCCESS:
+            return f"acquired+normalized {field} for {ticker}"
+        review_entry = result["review_entry"]
+        return f"acquired {field} for {ticker}, normalization degraded: {review_entry['detail']}"
+
+    return _handler
```

(Module docstring's STAGE STATUS section updated: "6 of 13" -> "7 of
13", `evidence_acquisition` moved from the "NOT built" list into the
real-handler list with its own rationale paragraph -- full text in the
niklar-stocks repo at this commit.)

## New tests (3, in `tests/test_niklar_orchestrator_stage_handlers.py`)

- `test_successful_fetch_and_normalization` -- happy path.
- `test_normalization_failure_degrades_non_blockingly_does_not_raise`
  -- proves a bad value from the fetch adapter does NOT raise past the
  handler; returns a "degraded" detail instead.
- `test_fetch_adapter_failure_propagates_and_stops_pipeline` -- proves
  the opposite: a genuine fetch failure (`ConnectionError`) DOES
  propagate, unlike the normalization case.

The increment 1 end-to-end `run_pipeline()` integration test was
updated to use this real handler instead of its prior stub, so the
"all real handlers wire into the existing orchestrator" proof now
covers 7 real handlers, not 6.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: two failure modes handled per architecture-
   doc-mandated distinction, both explicitly tested (propagate vs.
   degrade-non-blockingly) rather than picking one uniformly.
2. **Unnecessary abstraction/dependency**: no new dependency, no new
   file -- one function added to the same module, following the same
   factory pattern as increment 1's six.
3. **Security/privacy**: no I/O, no credentials, no secrets. The fetch
   mechanism remains entirely the caller's responsibility.
4. **Canonical boundary drift**: none -- same two files as increment 1,
   the 3 Drive-verbatim `ops/` modules remain untouched.

## Validation

- `unittest`: 178/178 (175 -> 178, 3 new)
- `mypy`: 0 issues, 16 source files (unchanged)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run: polled directly via the GitHub API after
  push (see `CURRENT_HANDOFF.txt` for the run ID/conclusion).
