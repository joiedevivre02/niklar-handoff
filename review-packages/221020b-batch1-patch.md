# Review package: niklar-stocks commit 221020b (Batch 1 patch)

Response to Drive sync doc RESPONSE_SEQ 10 ("CHANGES REQUIRED before
Batch 2") -- patches the exact two contract-semantic findings raised
against commit 53d2b3f, plus the non-blocking observation. Full diff
published in full so ChatGPT's GitHub connector -- which cannot reach
the private `niklar-stocks` repo directly -- can independently
re-inspect the exact patch. Scanned for secrets and Niklar canonical/
proprietary content before publishing (clean).

**Note on how this review was found**: RESPONSE_SEQ 10 was superseded
by RESPONSE_SEQ 11 in the Drive sync doc's compact live-response
surface before it was ever read in this session -- a real gap in this
session's own sync discipline (checking only the current
`LATEST_RESPONSE` block missed a response that had already rotated
out). Recovered by searching Drive for the response-archive doc
ChatGPT itself maintains (`NIKLAR_CHATGPT_RESPONSE_ARCHIVE_SEQ_10_
2026-08-09`) rather than assuming RESPONSE_SEQ 11's brief mention was
the complete picture.

## Commit message (verbatim)

```
commit 221020b7a009d11119ae336a00dd5121aa014d9c
Author: Claude <noreply@anthropic.com>
Date:   Sat Aug 8 20:38:01 2026 +0000

    Batch 1 patch: fix normalization-exception semantics + review visibility
    
    Per independent review of commit 53d2b3f (Drive sync doc RESPONSE_SEQ
    10, "CHANGES REQUIRED before Batch 2" -- found via a genuine gap in
    this session's own sync discipline: RESPONSE_SEQ 10 was superseded by
    RESPONSE_SEQ 11 before it was ever read, and had to be located in
    ChatGPT's own response-archive folder to recover its content).
    
    FINDING 1 (semantically wrong): build_current_state_snapshot() set
    normalization_exception_count = evidence_conditional_count +
    evidence_insufficient_count. Evidence can be CONDITIONAL/INSUFFICIENT
    for reasons that have nothing to do with a normalization failure
    (unavailable, stale, conflicting, pending acquisition, intentionally
    unknown) -- conflating the two invents a number the function was never
    actually given evidence for.
    
    FIX: normalization_exception_count is now derived from actual
    REVIEW_REASON_NORMALIZATION_EXCEPTION entries in the review_queue
    parameter this function already receives -- one source of truth, no
    new ledger invented. Correctly 0 when no such entries exist, even with
    plenty of conditional/insufficient evidence.
    
    FINDING 2 (missing visibility): the snapshot exposed only
    manual_review_queue_size/pending_llm_requests_count -- a reader could
    tell something needed attention but not which ticker/reason/request.
    Section 14 requires the snapshot to expose normalization exceptions/
    manual-review queue/pending LLM requests, not just their sizes.
    
    FIX: added pending_review_items/pending_llm_review_items -- compact,
    bounded (max_review_items, default 10) structured summaries
    (ticker/reason/detail; ticker/failure_reason/materiality/
    affected_dimensions) plus *_truncated counts so truncation is never
    silent. The full review_queue/llm_requests sequences remain
    authoritative; these are bounded pointers into them, not a raw dump.
    
    NON-BLOCKING OBSERVATION (also fixed): compute_evidence_status()
    returned only the materially-blocking subset of unresolved fields in
    the INSUFFICIENT case, silently dropping non-material unresolved
    fields from any visibility at all. Now returns a 3-tuple
    (status, affected_fields, all_unresolved_fields) -- affected_fields
    keeps its existing per-status meaning (the caveat-worthy reason),
    all_unresolved_fields always reports every unresolved field so
    downstream caveat/audit UX can't lose track of the ones that weren't
    the specific insufficiency cause.
    
    Minimum Sufficient Code preserved per explicit instruction: no
    dataclasses/Enums/new dependencies added, same pure-functions/
    plain-dict style, patched only the two files Batch 1 touched. 8 new
    tests (27->35): normalization-exception count proven NOT inferred from
    evidence status and proven derived from actual review_queue entries;
    pending review/LLM items proven to identify exact ticker/reason and
    proven bounded+truncation-reported; the mixed material/non-material
    INSUFFICIENT case proven to retain full unresolved-field visibility.
    
    Full validation: unittest 130/130 (95->130 overall), mypy 0/14 files,
    PWA smoke unaffected.
    
    Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
    Claude-Session: https://claude.ai/code/session_01Fd6zhPr8XrRzdMckvrXZQJ
```

## Full diff

```diff
diff --git a/ops/niklar_state_contracts.py b/ops/niklar_state_contracts.py
index e1a354f..02c527d 100644
--- a/ops/niklar_state_contracts.py
+++ b/ops/niklar_state_contracts.py
@@ -37,6 +37,20 @@ contract failure this repo's discipline forbids elsewhere (see
 Shadow Journal state" for the same pattern applied to a different gap).
 Marked as an explicit INTERFACE GAP in `build_current_state_snapshot()`
 below, not guessed at.
+
+Patched per independent review (Drive sync doc RESPONSE_SEQ 10;
+niklar-handoff `DECISIONS_CURRENT.txt` #32) after the first version of
+this module shipped: `normalization_exception_count` in
+`build_current_state_snapshot()` now derives from actual
+`REVIEW_REASON_NORMALIZATION_EXCEPTION` entries in `review_queue`,
+never inferred from `EVIDENCE_CONDITIONAL`/`EVIDENCE_INSUFFICIENT`
+totals (those are a materially different concept -- see that
+function's docstring); the snapshot now also exposes compact, bounded
+`pending_review_items`/`pending_llm_review_items` so a reader can see
+*which* ticker/reason needs attention, not just a count; and
+`compute_evidence_status()` now returns all unresolved fields
+alongside the materially-blocking subset, so the INSUFFICIENT case no
+longer silently drops non-material fields from view.
 """
 from __future__ import annotations
 
@@ -54,24 +68,31 @@ def compute_evidence_status(
     resolved_fields: Sequence[str],
     *,
     materially_dependent_fields: Sequence[str] = (),
-) -> "tuple[str, tuple[str, ...]]":
+) -> "tuple[str, tuple[str, ...], tuple[str, ...]]":
     """Pure classification, no I/O: given what evidence a computation
-    requires and what was actually resolved (normalized), return the
-    overall status plus the tuple of unresolved fields that caused it
-    (the "affected dimensions" the architecture doc requires be
-    identified -- section 3).
+    requires and what was actually resolved (normalized), return
+    `(status, affected_fields, all_unresolved_fields)`.
 
-    - All required fields resolved -> `EVIDENCE_COMPLETE`, no affected
-      fields.
+    - All required fields resolved -> `EVIDENCE_COMPLETE`,
+      `affected_fields` and `all_unresolved_fields` both empty.
     - Some required fields unresolved, but none of them are in
       `materially_dependent_fields` -> `EVIDENCE_CONDITIONAL`:
-      computation continues, affected fields are returned for the
-      caller to attach as a visible caveat.
+      computation continues; `affected_fields` (the caveat-worthy
+      reason) equals `all_unresolved_fields` here, since nothing is
+      material.
     - Any field in `materially_dependent_fields` is unresolved ->
       `EVIDENCE_INSUFFICIENT`: the caller must suppress/mark-unknown
       the conclusions that depend on it, per section 3's "only
       conclusions materially dependent on missing/unresolved evidence
-      are suppressed."
+      are suppressed." `affected_fields` narrows to only the
+      materially-blocking subset (the actual cause of insufficiency),
+      but `all_unresolved_fields` still reports every unresolved field
+      -- including non-material ones -- so downstream caveat/audit UX
+      never silently loses visibility of them just because a different
+      field was severe enough to also trigger INSUFFICIENT. (Found by
+      independent review of the first version of this function, which
+      returned only the materially-blocking subset in the INSUFFICIENT
+      case with no way for a caller to recover the rest.)
 
     Never raises on empty input; an empty `required_fields` is
     trivially `EVIDENCE_COMPLETE` (nothing was required).
@@ -79,11 +100,11 @@ def compute_evidence_status(
     resolved = set(resolved_fields)
     unresolved = tuple(f for f in required_fields if f not in resolved)
     if not unresolved:
-        return EVIDENCE_COMPLETE, ()
+        return EVIDENCE_COMPLETE, (), ()
     material_unresolved = tuple(f for f in unresolved if f in materially_dependent_fields)
     if material_unresolved:
-        return EVIDENCE_INSUFFICIENT, material_unresolved
-    return EVIDENCE_CONDITIONAL, unresolved
+        return EVIDENCE_INSUFFICIENT, material_unresolved, unresolved
+    return EVIDENCE_CONDITIONAL, unresolved, unresolved
 
 
 # --- Intraday observation (architecture doc section 7) ----------------------
@@ -292,6 +313,7 @@ def build_current_state_snapshot(
     cost_usage_status: Optional[str] = None,
     backup_status: Optional[str] = None,
     market_state: Optional[Mapping[str, Any]] = None,
+    max_review_items: int = 10,
 ) -> dict:
     """Compact deterministic summary of Niklar's operating state,
     assembled from already-computed inputs -- this function never
@@ -307,6 +329,36 @@ def build_current_state_snapshot(
     re-sorted, matching `mobile_contract.py`'s never-reorder
     discipline for its own list-shaped output).
 
+    `normalization_exception_count` is derived from actual
+    `review_queue` entries whose `reason` is
+    `REVIEW_REASON_NORMALIZATION_EXCEPTION` -- NOT from
+    evidence-conditional/insufficient totals. Those are a materially
+    different concept: a ticker can be CONDITIONAL/INSUFFICIENT for
+    reasons that have nothing to do with a normalization failure
+    (evidence simply not yet acquired, stale, conflicting, or
+    intentionally unknown). Conflating the two was a real defect found
+    by independent review of the first version of this function --
+    fixed here by deriving the count from the one place normalization
+    exceptions are actually recorded (`review_queue`), the same
+    parameter this function already receives, rather than inventing a
+    second ledger. If `build_current_state_snapshot()` is called
+    without ever routing normalization failures into `review_queue`,
+    this count is correctly zero -- never fabricated from a proxy.
+
+    `pending_review_items`/`pending_llm_review_items` give compact,
+    bounded (`max_review_items`, default 10) structured visibility into
+    *which* ticker/reason/materiality needs review -- the counts alone
+    only told a reader that something exists, not what. This was the
+    second defect independent review found: Section 14 requires the
+    snapshot to expose normalization exceptions/manual-review queue/
+    pending LLM requests, not just their sizes. Compact summaries only
+    (ticker/reason/detail, or ticker/failure_reason/materiality/
+    affected_dimensions) -- never a raw research dump; the full
+    `review_queue`/`llm_requests` sequences remain the authoritative
+    source, this is a bounded pointer into them. `*_truncated` reports
+    how many entries were left out when the queue exceeds
+    `max_review_items`, so truncation itself is never silent.
+
     `market_state` is deliberately a generic, optional passthrough
     mapping rather than a rigid sub-schema: the architecture doc names
     "market state/rankings/watchlist/positions as applicable" (section
@@ -316,9 +368,26 @@ def build_current_state_snapshot(
     complete = sum(1 for s in evidence_statuses if s == EVIDENCE_COMPLETE)
     conditional = sum(1 for s in evidence_statuses if s == EVIDENCE_CONDITIONAL)
     insufficient = sum(1 for s in evidence_statuses if s == EVIDENCE_INSUFFICIENT)
-    pending_llm = sum(1 for r in llm_requests if r.get("status") == LLM_REQUEST_PENDING)
+    normalization_exceptions = tuple(
+        e for e in review_queue if e.get("reason") == REVIEW_REASON_NORMALIZATION_EXCEPTION
+    )
+    pending_llm = tuple(r for r in llm_requests if r.get("status") == LLM_REQUEST_PENDING)
     changed_decision_objects = tuple(dict.fromkeys(e["decision_object_id"] for e in change_ledger))
 
+    pending_review_items = tuple(
+        {"reason": e.get("reason"), "ticker": e.get("ticker"), "detail": e.get("detail")}
+        for e in list(review_queue)[:max_review_items]
+    )
+    pending_llm_review_items = tuple(
+        {
+            "ticker": r.get("ticker"),
+            "failure_reason": r.get("failure_reason"),
+            "materiality": r.get("materiality"),
+            "affected_dimensions": r.get("affected_dimensions"),
+        }
+        for r in list(pending_llm)[:max_review_items]
+    )
+
     return {
         "run_timestamp": run_timestamp,
         "engine_version": engine_version,
@@ -330,9 +399,13 @@ def build_current_state_snapshot(
         "evidence_complete_count": complete,
         "evidence_conditional_count": conditional,
         "evidence_insufficient_count": insufficient,
-        "normalization_exception_count": conditional + insufficient,
+        "normalization_exception_count": len(normalization_exceptions),
         "manual_review_queue_size": len(review_queue),
-        "pending_llm_requests_count": pending_llm,
+        "pending_review_items": pending_review_items,
+        "pending_review_items_truncated": max(0, len(review_queue) - max_review_items),
+        "pending_llm_requests_count": len(pending_llm),
+        "pending_llm_review_items": pending_llm_review_items,
+        "pending_llm_review_items_truncated": max(0, len(pending_llm) - max_review_items),
         "pipeline_health": pipeline_health,
         "last_successful_run": last_successful_run,
         "cost_usage_status": cost_usage_status,
diff --git a/tests/test_niklar_state_contracts.py b/tests/test_niklar_state_contracts.py
index 09e6d1a..12c328d 100644
--- a/tests/test_niklar_state_contracts.py
+++ b/tests/test_niklar_state_contracts.py
@@ -2,6 +2,15 @@
 Batch 1 contracts/state layer (evidence status, intraday observation,
 change ledger, review queue, LLM review request, current state snapshot).
 
+Includes the Batch 1 semantic patch requested by independent review
+(Drive sync doc RESPONSE_SEQ 10, niklar-handoff DECISIONS_CURRENT.txt
+#32): normalization_exception_count must be derived from actual
+review_queue entries, not inferred from evidence-conditional/
+insufficient totals; the snapshot must expose compact structured
+review visibility (which ticker/reason), not just counts;
+compute_evidence_status() must not silently drop non-material
+unresolved fields in the INSUFFICIENT case.
+
 Uses synthetic fixture data only (no real tickers/prices/research).
 """
 import json
@@ -29,35 +38,58 @@ from ops.niklar_state_contracts import (
 
 class ComputeEvidenceStatusTests(unittest.TestCase):
     def test_all_resolved_is_complete_with_no_affected_fields(self):
-        status, affected = compute_evidence_status(["a", "b"], ["a", "b", "c"])
+        status, affected, all_unresolved = compute_evidence_status(["a", "b"], ["a", "b", "c"])
         self.assertEqual(status, EVIDENCE_COMPLETE)
         self.assertEqual(affected, ())
+        self.assertEqual(all_unresolved, ())
 
     def test_empty_required_fields_is_trivially_complete(self):
-        status, affected = compute_evidence_status([], [])
+        status, affected, all_unresolved = compute_evidence_status([], [])
         self.assertEqual(status, EVIDENCE_COMPLETE)
         self.assertEqual(affected, ())
+        self.assertEqual(all_unresolved, ())
 
     def test_missing_non_material_field_is_conditional(self):
-        status, affected = compute_evidence_status(
+        status, affected, all_unresolved = compute_evidence_status(
             ["a", "b"], ["a"], materially_dependent_fields=["c"]
         )
         self.assertEqual(status, EVIDENCE_CONDITIONAL)
         self.assertEqual(affected, ("b",))
+        self.assertEqual(all_unresolved, ("b",))
 
     def test_missing_material_field_is_insufficient(self):
-        status, affected = compute_evidence_status(
+        status, affected, all_unresolved = compute_evidence_status(
             ["a", "b"], ["a"], materially_dependent_fields=["b"]
         )
         self.assertEqual(status, EVIDENCE_INSUFFICIENT)
         self.assertEqual(affected, ("b",))
+        self.assertEqual(all_unresolved, ("b",))
 
     def test_mixed_material_and_non_material_missing_reports_only_material_as_insufficient_cause(self):
-        status, affected = compute_evidence_status(
+        status, affected, all_unresolved = compute_evidence_status(
             ["a", "b", "c"], [], materially_dependent_fields=["b"]
         )
         self.assertEqual(status, EVIDENCE_INSUFFICIENT)
         self.assertEqual(affected, ("b",))
+        # Batch 1 patch (RESPONSE_SEQ 10, non-blocking observation): the
+        # first version of this function only ever returned the
+        # materially-blocking subset here, silently dropping "a" and
+        # "c" from any visibility at all. all_unresolved must still
+        # report every unresolved field, material or not.
+        self.assertEqual(all_unresolved, ("a", "b", "c"))
+
+    def test_insufficient_all_unresolved_includes_non_material_fields_not_just_the_cause(self):
+        # A second, more targeted case for the same patch: two material
+        # fields and one non-material field missing -- affected must
+        # narrow to the material cause, all_unresolved must not.
+        status, affected, all_unresolved = compute_evidence_status(
+            ["mat1", "mat2", "nonmat"], [],
+            materially_dependent_fields=["mat1", "mat2"],
+        )
+        self.assertEqual(status, EVIDENCE_INSUFFICIENT)
+        self.assertEqual(affected, ("mat1", "mat2"))
+        self.assertIn("nonmat", all_unresolved)
+        self.assertEqual(all_unresolved, ("mat1", "mat2", "nonmat"))
 
 
 class NewIntradayObservationTests(unittest.TestCase):
@@ -236,7 +268,42 @@ class BuildCurrentStateSnapshotTests(unittest.TestCase):
         self.assertEqual(snapshot["evidence_complete_count"], 2)
         self.assertEqual(snapshot["evidence_conditional_count"], 1)
         self.assertEqual(snapshot["evidence_insufficient_count"], 3)
-        self.assertEqual(snapshot["normalization_exception_count"], 4)
+
+    def test_normalization_exception_count_is_not_inferred_from_evidence_status(self):
+        # Batch 1 patch (RESPONSE_SEQ 10, finding 1): plenty of
+        # CONDITIONAL/INSUFFICIENT evidence, but zero actual
+        # normalization-exception review_queue entries -- the count
+        # must be 0, not conditional+insufficient. Evidence can be
+        # conditional/insufficient for reasons that have nothing to do
+        # with a normalization failure (unavailable, stale, conflicting,
+        # intentionally unknown).
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=(),
+            evidence_statuses=[
+                EVIDENCE_CONDITIONAL, EVIDENCE_CONDITIONAL,
+                EVIDENCE_INSUFFICIENT, EVIDENCE_INSUFFICIENT, EVIDENCE_INSUFFICIENT,
+            ],
+        )
+        self.assertEqual(snapshot["normalization_exception_count"], 0)
+
+    def test_normalization_exception_count_derived_from_actual_review_queue_entries(self):
+        # The positive case for the same patch: the count must reflect
+        # real REVIEW_REASON_NORMALIZATION_EXCEPTION entries, not a
+        # proxy, and must ignore review_queue entries with other
+        # reasons.
+        norm_exception = new_review_queue_entry(
+            reason=REVIEW_REASON_NORMALIZATION_EXCEPTION, detail="x", raised_at="t"
+        )
+        other_reason = new_review_queue_entry(
+            reason=REVIEW_REASON_DATA_QUALITY_ANOMALY, detail="x", raised_at="t"
+        )
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=[norm_exception, norm_exception, other_reason],
+            llm_requests=(), evidence_statuses=[],
+        )
+        self.assertEqual(snapshot["normalization_exception_count"], 2)
 
     def test_pending_llm_requests_count_only_counts_pending(self):
         pending = new_llm_review_request(
@@ -265,6 +332,97 @@ class BuildCurrentStateSnapshotTests(unittest.TestCase):
         )
         self.assertEqual(snapshot["manual_review_queue_size"], 2)
 
+    def test_pending_review_items_identify_exact_ticker_and_reason(self):
+        # Batch 1 patch (RESPONSE_SEQ 10, finding 2): the snapshot must
+        # expose which ticker/reason needs review, not just a count.
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_NORMALIZATION_EXCEPTION,
+            detail="unparseable disclosure date",
+            raised_at="t",
+            ticker="TEST1",
+        )
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=[entry], llm_requests=(),
+            evidence_statuses=[],
+        )
+        self.assertEqual(len(snapshot["pending_review_items"]), 1)
+        item = snapshot["pending_review_items"][0]
+        self.assertEqual(item["ticker"], "TEST1")
+        self.assertEqual(item["reason"], REVIEW_REASON_NORMALIZATION_EXCEPTION)
+        self.assertEqual(item["detail"], "unparseable disclosure date")
+
+    def test_pending_review_items_bounded_and_truncation_reported(self):
+        entries = [
+            new_review_queue_entry(
+                reason=REVIEW_REASON_DATA_QUALITY_ANOMALY, detail="x", raised_at="t",
+                ticker=f"TEST{i}",
+            )
+            for i in range(15)
+        ]
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=entries, llm_requests=(),
+            evidence_statuses=[], max_review_items=10,
+        )
+        self.assertEqual(len(snapshot["pending_review_items"]), 10)
+        self.assertEqual(snapshot["pending_review_items_truncated"], 5)
+        self.assertEqual(snapshot["manual_review_queue_size"], 15)  # full count unaffected
+
+    def test_pending_llm_review_items_identify_exact_ticker_and_reason(self):
+        request = new_llm_review_request(
+            ticker="TEST1", evidence_description="ambiguous wording",
+            failure_reason="parser could not classify materiality",
+            materiality="HIGH", affected_dimensions=["thesis_health"],
+            deterministic_attempts_made=["regex_parse"], requested_at="t",
+        )
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=[request],
+            evidence_statuses=[],
+        )
+        self.assertEqual(len(snapshot["pending_llm_review_items"]), 1)
+        item = snapshot["pending_llm_review_items"][0]
+        self.assertEqual(item["ticker"], "TEST1")
+        self.assertEqual(item["failure_reason"], "parser could not classify materiality")
+        self.assertEqual(item["materiality"], "HIGH")
+        self.assertEqual(item["affected_dimensions"], ("thesis_health",))
+
+    def test_pending_llm_review_items_excludes_resolved_requests(self):
+        pending = new_llm_review_request(
+            ticker="A", evidence_description="x", failure_reason="x",
+            materiality="LOW", affected_dimensions=[], deterministic_attempts_made=[],
+            requested_at="t",
+        )
+        resolved = dict(pending)
+        resolved["ticker"] = "B"
+        resolved["status"] = "RESOLVED"
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=[pending, resolved],
+            evidence_statuses=[],
+        )
+        self.assertEqual(len(snapshot["pending_llm_review_items"]), 1)
+        self.assertEqual(snapshot["pending_llm_review_items"][0]["ticker"], "A")
+
+    def test_pending_llm_review_items_bounded_and_truncation_reported(self):
+        requests = [
+            new_llm_review_request(
+                ticker=f"TEST{i}", evidence_description="x", failure_reason="x",
+                materiality="LOW", affected_dimensions=[], deterministic_attempts_made=[],
+                requested_at="t",
+            )
+            for i in range(12)
+        ]
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=requests,
+            evidence_statuses=[], max_review_items=10,
+        )
+        self.assertEqual(len(snapshot["pending_llm_review_items"]), 10)
+        self.assertEqual(snapshot["pending_llm_review_items_truncated"], 2)
+        self.assertEqual(snapshot["pending_llm_requests_count"], 12)  # full count unaffected
+
     def test_changed_decision_objects_deduped_preserving_first_occurrence_order(self):
         ledger = [
             new_change_ledger_entry(
@@ -300,6 +458,10 @@ class BuildCurrentStateSnapshotTests(unittest.TestCase):
         self.assertIsNone(snapshot["cost_usage_status"])
         self.assertIsNone(snapshot["backup_status"])
         self.assertIsNone(snapshot["market_state"])
+        self.assertEqual(snapshot["pending_review_items"], ())
+        self.assertEqual(snapshot["pending_review_items_truncated"], 0)
+        self.assertEqual(snapshot["pending_llm_review_items"], ())
+        self.assertEqual(snapshot["pending_llm_review_items_truncated"], 0)
 
     def test_market_state_passthrough_is_not_interpreted(self):
         raw = {"opportunity_matrix": ["TEST1", "TEST2"]}
@@ -311,9 +473,17 @@ class BuildCurrentStateSnapshotTests(unittest.TestCase):
         self.assertEqual(snapshot["market_state"], raw)
 
     def test_is_json_serializable(self):
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_NORMALIZATION_EXCEPTION, detail="x", raised_at="t", ticker="A"
+        )
+        request = new_llm_review_request(
+            ticker="A", evidence_description="x", failure_reason="x",
+            materiality="LOW", affected_dimensions=[], deterministic_attempts_made=[],
+            requested_at="t",
+        )
         snapshot = build_current_state_snapshot(
             run_timestamp="t", engine_version="v1", canon_version="c1",
-            change_ledger=(), review_queue=(), llm_requests=(),
+            change_ledger=(), review_queue=[entry], llm_requests=[request],
             evidence_statuses=[EVIDENCE_COMPLETE],
             active_tactical_states=["TEST"],
         )
```

## Response to each finding

**Finding 1 (normalization_exception_count semantically wrong)**:
fixed. `build_current_state_snapshot()` now counts actual
`REVIEW_REASON_NORMALIZATION_EXCEPTION` entries in `review_queue`
(the parameter it already received) instead of
`evidence_conditional_count + evidence_insufficient_count`. Tests
prove both directions: many conditional/insufficient evidence
statuses with zero normalization-exception review-queue entries now
correctly reports 0; actual normalization-exception entries (mixed
with other reasons) are correctly counted and other reasons excluded.

**Finding 2 (missing compact review visibility)**: fixed. Added
`pending_review_items`/`pending_llm_review_items` -- bounded (default
10), compact structured summaries identifying exact
ticker/reason/detail (review queue) or
ticker/failure_reason/materiality/affected_dimensions (LLM requests),
plus `*_truncated` counts so truncation itself is never silent. The
full `review_queue`/`llm_requests` sequences remain authoritative --
these are bounded pointers into them, not a raw dump. Tests prove
exact-ticker/reason identification, truncation at the boundary, and
that resolved LLM requests are excluded from the pending view.

**Non-blocking observation (mixed material/non-material unresolved
fields)**: fixed. `compute_evidence_status()` now returns a 3-tuple
`(status, affected_fields, all_unresolved_fields)` -- chose the
"return both subsets" option from the two offered, since the
"clarify/document only" alternative would have left downstream
caveat/audit UX with no actual way to recover the dropped fields
(nothing else in this module computes the full unresolved set).
`affected_fields` keeps its existing per-status meaning; a new,
explicit test proves the mixed case (2 material + 1 non-material
unresolved) now retains all three fields in `all_unresolved_fields`
while `affected_fields` still narrows to just the material cause.

## Minimum Sufficient Code compliance

Per the explicit instruction ("Do not broaden Batch 1. Patch only
these semantic contract gaps and tests. No need to redesign with
dataclasses/Enums or add dependencies."): no dataclasses/Enums/new
dependencies added, same pure-functions/plain-dict style preserved,
only the same two files Batch 1 originally touched were patched. No
new module, no new file.

## Real-world verification (not just local)

Real GitHub Actions run for this commit: `31277370000`
(`joiedevivre02/niklar-stocks`, workflow `ai-qa.yml`) --
`status: "completed"`, `conclusion: "success"`. Local: `unittest`
130/130 (95->130 overall, 27->35 for this module specifically -- 8 new
tests), `mypy` 0 issues across 14 files, PWA smoke test unaffected --
exactly the checklist RESPONSE_SEQ 10's "VALIDATION REQUIRED" section
specified.
