# Review package: niklar-stocks commit 05657ef (Batch 3 increment 3)

Continues the owner-authorized "batch 3 onwards" work (decision #47),
wiring the Section 19 deterministic notification-service core
(decision #50, commit 577b60b) into the two remaining orchestrator
stages it was built for. Full diff published so ChatGPT's GitHub
connector -- which cannot reach the private `niklar-stocks` repo
directly -- can independently inspect the exact code. Scanned for
secrets and Niklar canonical/proprietary content before publishing
(clean).

## What this adds

`make_consumer_sync_handler()` -- ONE shared factory used for BOTH
`discovery_consumers_sync` and `tactical_consumers_sync` stage keys
(9th and 10th of 13 canonical stages to get a real handler -- actually
both land in this one increment, bringing the module total to 9 real
handlers). Sequences Section 19's own eligibility -> duplicate/resend
-> send -> delivery-status-classify order, using the
`niklar_notification_service` functions built in the prior increment
directly (no reimplementation).

**MEDIUM confidence, explicitly flagged for this review**: treating
both stage names as routing through this one shared mechanism (scope-
differentiated only -- e.g. a per-ticker `notification_id` for
`tactical_consumers_sync` vs. a brief-scoped one for
`discovery_consumers_sync`) is a genuine interpretive judgment call.
Grounding: Section 19 requires "Automatic and manual Slack sends must
share the SAME deterministic notification service." Gap: this repo has
no further specification distinguishing what "discovery" vs.
"tactical" sync concretely differ in beyond scope -- if that
distinction should instead route through materially different logic,
please flag it.

**Non-blocking Slack failure**: `send_adapter`'s failure (raised
exception) is caught and classified as a failed delivery attempt,
never propagated to `run_pipeline()` -- "Slack outage/failure is
non-blocking to... canonical engine completion" (Section 19). This
mirrors `evidence_acquisition`'s (increment 2) non-blocking
normalization-degradation pattern, applied here for the same
architecture-doc-mandated reason.

## Diff

```diff
--- a/ops/niklar_orchestrator_stage_handlers.py
+++ b/ops/niklar_orchestrator_stage_handlers.py
@@ imports @@
 from .niklar_normalization_policy import NORMALIZATION_SUCCESS, attempt_normalization
+from .niklar_notification_service import (
+    NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE as _NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE,
+    NOTIFICATION_BLOCKED_STALE_OBSERVATION as _NOTIFICATION_BLOCKED_STALE_OBSERVATION,
+    NOTIFICATION_SEND_ALREADY_SENT_REQUIRES_EXPLICIT_RESEND as _NOTIFICATION_SEND_ALREADY_SENT_REQUIRES_EXPLICIT_RESEND,
+    check_duplicate_send as _check_duplicate_send,
+    classify_delivery_attempt as _classify_delivery_attempt,
+    decide_notification_eligibility as _decide_notification_eligibility,
+)
 from .niklar_single_stock_publication_validator import validate as validate_publication

@@ new section, between decision_object_freeze and postflight_verification @@
+# --- discovery_consumers_sync / tactical_consumers_sync -------------------
+
+
+def make_consumer_sync_handler(
+    *,
+    evidence_status: str,
+    execution_sensitive: bool,
+    notification_id: str,
+    observation_age_seconds: Optional[float] = None,
+    max_observation_age_seconds: Optional[float] = None,
+    prior_sends: Sequence[Mapping[str, Any]] = (),
+    explicit_resend: bool = False,
+    send_adapter: Optional[Callable[[], bool]] = None,
+    max_retries: int = 0,
+) -> Callable[[], str]:
+    """... (see module docstring / full file for the complete rationale) ..."""
+
+    def _handler() -> str:
+        eligibility = _decide_notification_eligibility(
+            evidence_status=evidence_status,
+            execution_sensitive=execution_sensitive,
+            observation_age_seconds=observation_age_seconds,
+            max_observation_age_seconds=max_observation_age_seconds,
+        )
+        if eligibility in (_NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE, _NOTIFICATION_BLOCKED_STALE_OBSERVATION):
+            return f"blocked: {eligibility}"
+
+        dup = _check_duplicate_send(
+            notification_id=notification_id, prior_sends=prior_sends, explicit_resend=explicit_resend
+        )
+        if dup["decision"] == _NOTIFICATION_SEND_ALREADY_SENT_REQUIRES_EXPLICIT_RESEND:
+            return f"skipped: already sent at {dup['prior_sent_at']}, explicit resend required"
+
+        if send_adapter is None:
+            return f"eligible ({eligibility}), no send_adapter supplied -- not sent"
+
+        try:
+            succeeded = send_adapter()
+        except Exception:  # noqa: BLE001 -- deliberately broad, Slack failure is non-blocking
+            succeeded = False
+        status = _classify_delivery_attempt(succeeded=succeeded, retries_remaining=max_retries)
+        return f"{eligibility}: delivery {status}"
+
+    return _handler
```

(Module docstring's STAGE STATUS updated: "7 of 13" -> "9 of 13,"
these two stages moved from the "NOT built" list into the real-handler
list with their own rationale paragraph -- full text in the
niklar-stocks repo at this commit.)

## New tests (9, in `tests/test_niklar_orchestrator_stage_handlers.py`)

- `test_insufficient_evidence_blocks_and_never_calls_send_adapter` --
  proves the eligibility gate short-circuits before any send attempt.
- `test_no_send_adapter_reports_eligibility_without_sending` --
  dry-run/preview-only mode.
- `test_successful_send_reports_sent`
- `test_failed_send_with_retries_remaining_reports_pending_retry`
- `test_failed_send_with_no_retries_reports_delivery_failed`
- `test_send_adapter_exception_is_caught_not_propagated_and_classified_as_failure`
  -- proves the handler never raises past itself on a Slack-side
  error, unlike a genuine `evidence_acquisition`-style fetch failure.
- `test_duplicate_send_without_explicit_resend_is_skipped` -- proves
  the send adapter is never called for an already-sent notification.
- `test_explicit_resend_allows_a_fresh_send_attempt`
- `test_conditional_evidence_is_eligible_with_caveat_in_detail`

The end-to-end `run_pipeline()` integration test now uses both real
handlers instead of stubs -- 9 real handlers wired together, proving
the full set genuinely satisfies the existing orchestrator's contract.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: eligibility/duplicate gates short-circuit
   cleanly (tested); `send_adapter` failure is deliberately caught, not
   propagated, per Section 19's explicit non-blocking requirement
   (tested with a raised `ConnectionError`).
2. **Unnecessary abstraction/dependency**: no new dependency, no new
   file -- one function reusing the prior increment's module directly.
   ONE shared factory for both stage keys, not two near-duplicate ones.
3. **Security/privacy**: no I/O, no Slack SDK, no credentials anywhere
   in this addition. The actual send mechanism remains entirely the
   caller's injected responsibility.
4. **Canonical boundary drift**: none of the 3 Drive-verbatim `ops/`
   modules were touched.

## Validation

- `unittest`: 211/211 (202 -> 211, 9 new)
- `mypy`: 0 issues, 17 source files (unchanged)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run: polled directly via the GitHub API after
  push (see `CURRENT_HANDOFF.txt` for the run ID/conclusion).
