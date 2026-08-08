# Review package: niklar-stocks commit 53d2b3f

Pre-live architecture Batch 1 (contracts/state layer), published in
full so ChatGPT's GitHub connector -- which cannot reach the private
`niklar-stocks` repo directly (confirmed 404 on an earlier commit) --
can independently inspect the exact code/diff without needing
private-repo access. This is a full-diff package (not a trimmed
function excerpt) since both new files are new and already small.

**Scope**: exactly the 2 files this commit added -- both entirely new,
nothing modified elsewhere. Scanned for secrets and Niklar canonical/
proprietary content before publishing to this public repo (clean).
`ops/niklar_state_contracts.py` is pure, deterministic Python (stdlib
`typing` only) -- no I/O, no credentials, no network, no trading/
scoring/tactical logic; it implements only the 6 contracts/state-layer
pieces the pre-live architecture doc's Batch 1 names.

## Commit message (verbatim)

```
commit 53d2b3f01d83f7c664d9652a1dbbeb8b3f3218ce
Author: Claude <noreply@anthropic.com>
Date:   Sat Aug 8 20:05:04 2026 +0000

    Pre-live architecture Batch 1: contracts/state layer
    
    Per the owner-approved pre-live architecture (Google Drive
    NIKLAR_PRELIVE_ARCHITECTURE_DECISIONS_CURRENT, frozen 2026-08-09;
    Drive sync doc RESPONSE_SEQ 8, IMPLEMENTATION AUTHORIZATION: RELEASED),
    implementing Batch 1 only per the explicit staged-batch discipline
    ("Do not attempt one giant refactor"): the contracts/state layer.
    
    New module ops/niklar_state_contracts.py -- pure, deterministic data
    shapes and pure builder/classification functions only, no I/O, no
    credentials, no Drive/Sheets/Slack/market-data connectivity, matching
    ops/'s existing "deterministic, non-networked" discipline. Follows
    mobile_contract.py's established style (plain dicts, bare string
    constants for closed value sets) rather than introducing a new
    dataclass/Enum pattern for this batch alone.
    
    Six pieces, matching the architecture doc's Batch 1 scope exactly:
    1. compute_evidence_status() -- doc section 3 (EVIDENCE_COMPLETE /
       EVIDENCE_CONDITIONAL / EVIDENCE_INSUFFICIENT classification,
       non-blocking degradation).
    2. new_intraday_observation() -- doc section 7, the one contract every
       intraday input adapter (manual, screenshot OCR, future feed) must
       emit. Fail-closed on an unrecognized source.
    3. new_change_ledger_entry() / append_change() -- doc section 14,
       append-only, never mutates input.
    4. new_review_queue_entry() / append_review_entry() -- doc section 14's
       8 named trigger conditions. Fail-closed on an unrecognized reason.
    5. new_llm_review_request() -- doc sections 2 and 4. Always starts
       PENDING; no keyword argument exists to bypass that (owner
       authorization must be a separate, later, explicit action).
    6. build_current_state_snapshot() -- doc section 14. Pure aggregation
       over already-supplied collections; computes its own counts so they
       can never drift from what was actually passed in.
    
    market_state (doc section 14's "market state/rankings/watchlist/
    positions as applicable") is a deliberate generic optional passthrough,
    not a rigid invented sub-schema -- marked as an explicit INTERFACE GAP,
    matching NIKLAR_APP_CONTRACT.md's existing real-vs-simulated-position
    gap-handling pattern, rather than guessing at semantics not yet defined
    anywhere in canon.
    
    27 new tests (tests/test_niklar_state_contracts.py): every function's
    happy path, fail-closed validation paths, non-mutation/append-only
    guarantees, JSON-serializability, and that counts/dedup logic in the
    snapshot builder are actually computed from inputs, not fabricated.
    
    Self-audit against the standing 4 criteria (failure envelope,
    unnecessary abstraction/dependency, security/privacy, canonical
    boundary drift): clean, no findings -- pure stdlib typing only, no new
    dependency, no I/O surface to leak from, no trading/scoring/tactical
    logic touched.
    
    Full validation: unittest 122/122 (95->122, 27 new), mypy 0 issues/14
    files, PWA smoke unaffected.
    
    Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
    Claude-Session: https://claude.ai/code/session_01Fd6zhPr8XrRzdMckvrXZQJ
```

## Full diff (both files are new)

```diff
diff --git a/ops/niklar_state_contracts.py b/ops/niklar_state_contracts.py
new file mode 100644
index 0000000..e1a354f
--- /dev/null
+++ b/ops/niklar_state_contracts.py
@@ -0,0 +1,341 @@
+"""Niklar pre-live state/contracts layer.
+
+Batch 1 of the owner-approved pre-live architecture (Google Drive
+`NIKLAR_PRELIVE_ARCHITECTURE_DECISIONS_CURRENT`, frozen 2026-08-09; see
+niklar-handoff `DECISIONS_CURRENT.txt` for the checkpoint recording
+this batch). Pure, deterministic data shapes and pure builder/
+classification functions only -- no I/O, no credentials, no Drive/
+Sheets/Slack/market-data connectivity, matching this package's existing
+"deterministic, non-networked" discipline (see `ops/README.md`).
+Wiring these into a real pipeline (reading/writing actual evidence,
+persisting a real ledger, scheduling real runs) is a later batch, not
+this one.
+
+Follows this package's existing style (`mobile_contract.py`): plain
+`dict` return values (directly JSON-serializable, no dataclass/Enum
+conversion step needed) and bare string constants for closed value
+sets, rather than introducing a new pattern for this batch alone.
+
+Six pieces, matching the architecture doc's Batch 1 scope exactly:
+  1. `compute_evidence_status()` -- doc section 3.
+  2. `new_intraday_observation()` -- doc section 7, the one contract
+     every intraday input adapter (manual, screenshot OCR, future feed)
+     must emit, regardless of source.
+  3. `new_change_ledger_entry()` / `append_change()` -- doc section 14.
+  4. `new_review_queue_entry()` / `append_review_entry()` -- doc section
+     14's 8 named trigger conditions.
+  5. `new_llm_review_request()` -- doc sections 2 and 4.
+  6. `build_current_state_snapshot()` -- doc section 14.
+
+Deliberately generic/optional where the architecture doc names a field
+category without yet defining its shape (`market_state` below covers
+"market state/rankings/watchlist/positions as applicable" from section
+14) -- inventing a rigid sub-schema for that now would be exactly the
+"silently become a confident recommendation" / reverse-authoring-the-
+contract failure this repo's discipline forbids elsewhere (see
+`docs/app/NIKLAR_APP_CONTRACT.md`, "Real positions vs. simulated/
+Shadow Journal state" for the same pattern applied to a different gap).
+Marked as an explicit INTERFACE GAP in `build_current_state_snapshot()`
+below, not guessed at.
+"""
+from __future__ import annotations
+
+from typing import Any, Mapping, Optional, Sequence
+
+# --- Evidence status (architecture doc section 3) --------------------------
+
+EVIDENCE_COMPLETE = "EVIDENCE_COMPLETE"
+EVIDENCE_CONDITIONAL = "EVIDENCE_CONDITIONAL"
+EVIDENCE_INSUFFICIENT = "EVIDENCE_INSUFFICIENT"
+
+
+def compute_evidence_status(
+    required_fields: Sequence[str],
+    resolved_fields: Sequence[str],
+    *,
+    materially_dependent_fields: Sequence[str] = (),
+) -> "tuple[str, tuple[str, ...]]":
+    """Pure classification, no I/O: given what evidence a computation
+    requires and what was actually resolved (normalized), return the
+    overall status plus the tuple of unresolved fields that caused it
+    (the "affected dimensions" the architecture doc requires be
+    identified -- section 3).
+
+    - All required fields resolved -> `EVIDENCE_COMPLETE`, no affected
+      fields.
+    - Some required fields unresolved, but none of them are in
+      `materially_dependent_fields` -> `EVIDENCE_CONDITIONAL`:
+      computation continues, affected fields are returned for the
+      caller to attach as a visible caveat.
+    - Any field in `materially_dependent_fields` is unresolved ->
+      `EVIDENCE_INSUFFICIENT`: the caller must suppress/mark-unknown
+      the conclusions that depend on it, per section 3's "only
+      conclusions materially dependent on missing/unresolved evidence
+      are suppressed."
+
+    Never raises on empty input; an empty `required_fields` is
+    trivially `EVIDENCE_COMPLETE` (nothing was required).
+    """
+    resolved = set(resolved_fields)
+    unresolved = tuple(f for f in required_fields if f not in resolved)
+    if not unresolved:
+        return EVIDENCE_COMPLETE, ()
+    material_unresolved = tuple(f for f in unresolved if f in materially_dependent_fields)
+    if material_unresolved:
+        return EVIDENCE_INSUFFICIENT, material_unresolved
+    return EVIDENCE_CONDITIONAL, unresolved
+
+
+# --- Intraday observation (architecture doc section 7) ----------------------
+
+OBSERVATION_SOURCE_MANUAL_ENTRY = "MANUAL_ENTRY"
+OBSERVATION_SOURCE_SCREENSHOT_OCR = "SCREENSHOT_OCR"
+OBSERVATION_SOURCE_SCREEN_CAPTURE = "SCREEN_CAPTURE"
+OBSERVATION_SOURCE_LIVE_FEED = "LIVE_FEED"
+
+_VALID_OBSERVATION_SOURCES = frozenset(
+    {
+        OBSERVATION_SOURCE_MANUAL_ENTRY,
+        OBSERVATION_SOURCE_SCREENSHOT_OCR,
+        OBSERVATION_SOURCE_SCREEN_CAPTURE,
+        OBSERVATION_SOURCE_LIVE_FEED,
+    }
+)
+
+
+def new_intraday_observation(
+    *,
+    ticker: str,
+    field: str,
+    value: str,
+    observed_at: str,
+    source: str,
+    confidence: Optional[str] = None,
+    source_detail: Optional[str] = None,
+) -> dict:
+    """One normalized intraday tactical input, regardless of source.
+
+    All adapters (manual entry, screenshot OCR, a future ScreenCaptureKit
+    pipeline, an eventual live feed) MUST emit this same shape so the
+    canonical tactical engine remains unchanged (architecture doc
+    section 7). `source` is provenance/audit metadata only -- never
+    branch tactical logic on it elsewhere.
+
+    `confidence` is a string, not a numeric score -- this layer never
+    fabricates a confidence number; adapters that can genuinely assess
+    it (e.g. OCR recognition quality) report their own label, passed
+    through verbatim, same discipline as `mobile_contract.py`'s
+    `as_of_status` passthrough. `observed_at` is adapter-supplied and
+    never fabricated by this function.
+
+    Raises `ValueError` on an unrecognized `source` -- fail closed
+    rather than silently accept an adapter type this layer doesn't
+    know about; a new adapter must be added to
+    `_VALID_OBSERVATION_SOURCES` deliberately, not implied.
+    """
+    if source not in _VALID_OBSERVATION_SOURCES:
+        raise ValueError(f"unrecognized observation source: {source!r}")
+    return {
+        "ticker": ticker,
+        "field": field,
+        "value": value,
+        "observed_at": observed_at,
+        "source": source,
+        "confidence": confidence,
+        "source_detail": source_detail,
+    }
+
+
+# --- Change ledger (architecture doc section 14) ----------------------------
+
+
+def new_change_ledger_entry(
+    *,
+    timestamp: str,
+    decision_object_id: str,
+    decision_object_version: str,
+    reason_code: str,
+    rule_fired: str,
+    relevant_inputs: Sequence[str] = (),
+) -> dict:
+    """One record of a meaningful Decision Object state transition."""
+    return {
+        "timestamp": timestamp,
+        "decision_object_id": decision_object_id,
+        "decision_object_version": decision_object_version,
+        "reason_code": reason_code,
+        "rule_fired": rule_fired,
+        "relevant_inputs": tuple(relevant_inputs),
+    }
+
+
+def append_change(
+    ledger: Sequence[Mapping[str, Any]], entry: Mapping[str, Any]
+) -> "tuple[Mapping[str, Any], ...]":
+    """Pure append -- never mutates `ledger`, matching this package's
+    existing never-mutate-input discipline (`mobile_contract.py` never
+    mutates its input rows either). The ledger itself is append-only,
+    matching the canonical `research_database_append` stage's
+    append-only discipline
+    (`ops/niklar_single_stock_orchestrator.py`'s stage list).
+    """
+    return tuple(ledger) + (entry,)
+
+
+# --- Review queue (architecture doc section 14) -----------------------------
+
+REVIEW_REASON_NORMALIZATION_EXCEPTION = "NORMALIZATION_EXCEPTION"
+REVIEW_REASON_CONFLICTING_HIGH_AUTHORITY_EVIDENCE = "CONFLICTING_HIGH_AUTHORITY_EVIDENCE"
+REVIEW_REASON_UNEXPECTED_ENGINE_STATE = "UNEXPECTED_ENGINE_STATE"
+REVIEW_REASON_LARGE_RANKING_TRANSITION = "LARGE_RANKING_TRANSITION"
+REVIEW_REASON_TACTICAL_STATE_INCOMPLETE_EVIDENCE = "TACTICAL_STATE_INCOMPLETE_EVIDENCE"
+REVIEW_REASON_REPEATED_PARSER_FAILURE = "REPEATED_PARSER_FAILURE"
+REVIEW_REASON_DATA_QUALITY_ANOMALY = "DATA_QUALITY_ANOMALY"
+REVIEW_REASON_CANONICAL_INVARIANT_WARNING = "CANONICAL_INVARIANT_WARNING"
+
+# The 8 trigger conditions the architecture doc names verbatim (section
+# 14) as warranting ChatGPT review. Not an exhaustive taxonomy invented
+# here -- only what the doc specifies.
+_VALID_REVIEW_REASONS = frozenset(
+    {
+        REVIEW_REASON_NORMALIZATION_EXCEPTION,
+        REVIEW_REASON_CONFLICTING_HIGH_AUTHORITY_EVIDENCE,
+        REVIEW_REASON_UNEXPECTED_ENGINE_STATE,
+        REVIEW_REASON_LARGE_RANKING_TRANSITION,
+        REVIEW_REASON_TACTICAL_STATE_INCOMPLETE_EVIDENCE,
+        REVIEW_REASON_REPEATED_PARSER_FAILURE,
+        REVIEW_REASON_DATA_QUALITY_ANOMALY,
+        REVIEW_REASON_CANONICAL_INVARIANT_WARNING,
+    }
+)
+
+
+def new_review_queue_entry(
+    *, reason: str, detail: str, raised_at: str, ticker: Optional[str] = None
+) -> dict:
+    """Raises `ValueError` on an unrecognized `reason` -- same
+    fail-closed discipline as `new_intraday_observation()`.
+    """
+    if reason not in _VALID_REVIEW_REASONS:
+        raise ValueError(f"unrecognized review queue reason: {reason!r}")
+    return {"reason": reason, "detail": detail, "raised_at": raised_at, "ticker": ticker}
+
+
+def append_review_entry(
+    queue: Sequence[Mapping[str, Any]], entry: Mapping[str, Any]
+) -> "tuple[Mapping[str, Any], ...]":
+    """Pure append -- see `append_change()`'s docstring for the same
+    never-mutate-input discipline.
+    """
+    return tuple(queue) + (entry,)
+
+
+# --- LLM/manual-review request ledger (architecture doc sections 2, 4) -----
+
+LLM_REQUEST_PENDING = "PENDING"
+LLM_REQUEST_OWNER_AUTHORIZED = "OWNER_AUTHORIZED"
+LLM_REQUEST_RESOLVED = "RESOLVED"
+LLM_REQUEST_REJECTED = "REJECTED"
+
+
+def new_llm_review_request(
+    *,
+    ticker: str,
+    evidence_description: str,
+    failure_reason: str,
+    materiality: str,
+    affected_dimensions: Sequence[str],
+    deterministic_attempts_made: Sequence[str],
+    requested_at: str,
+) -> dict:
+    """One quarantined unresolved-evidence case (architecture doc
+    section 2: "entered into an LLM/manual-review request ledger with
+    ticker, evidence, failure reason, materiality, affected dimensions,
+    and deterministic attempts made").
+
+    `status` always starts `LLM_REQUEST_PENDING` -- there is no
+    parameter to set it otherwise here, deliberately: owner
+    authorization is required before any production LLM resolution
+    call (architecture doc section 2), and that must be a separate,
+    explicit later action performed elsewhere, never implied by
+    construction.
+    """
+    return {
+        "ticker": ticker,
+        "evidence_description": evidence_description,
+        "failure_reason": failure_reason,
+        "materiality": materiality,
+        "affected_dimensions": tuple(affected_dimensions),
+        "deterministic_attempts_made": tuple(deterministic_attempts_made),
+        "requested_at": requested_at,
+        "status": LLM_REQUEST_PENDING,
+    }
+
+
+# --- Current state snapshot (architecture doc section 14) ------------------
+
+
+def build_current_state_snapshot(
+    *,
+    run_timestamp: str,
+    engine_version: str,
+    canon_version: str,
+    change_ledger: Sequence[Mapping[str, Any]],
+    review_queue: Sequence[Mapping[str, Any]],
+    llm_requests: Sequence[Mapping[str, Any]],
+    evidence_statuses: Sequence[str],
+    active_tactical_states: Sequence[str] = (),
+    pipeline_health: str = "UNKNOWN",
+    last_successful_run: Optional[str] = None,
+    data_timestamp: Optional[str] = None,
+    research_timestamp: Optional[str] = None,
+    cost_usage_status: Optional[str] = None,
+    backup_status: Optional[str] = None,
+    market_state: Optional[Mapping[str, Any]] = None,
+) -> dict:
+    """Compact deterministic summary of Niklar's operating state,
+    assembled from already-computed inputs -- this function never
+    fetches or reads anything itself (no I/O). Wiring a real builder
+    that pulls live ledger/queue/pipeline state together from an actual
+    running system is a later batch, not this one.
+
+    Counts (`evidence_*_count`, `manual_review_queue_size`,
+    `pending_llm_requests_count`, `normalization_exception_count`) are
+    computed here, not supplied by the caller, so they can never drift
+    from the collections actually passed in. `changed_decision_objects`
+    is deduped while preserving first-occurrence order (never silently
+    re-sorted, matching `mobile_contract.py`'s never-reorder
+    discipline for its own list-shaped output).
+
+    `market_state` is deliberately a generic, optional passthrough
+    mapping rather than a rigid sub-schema: the architecture doc names
+    "market state/rankings/watchlist/positions as applicable" (section
+    14) without defining their shape. See this module's docstring for
+    why that's marked an INTERFACE GAP here rather than guessed at.
+    """
+    complete = sum(1 for s in evidence_statuses if s == EVIDENCE_COMPLETE)
+    conditional = sum(1 for s in evidence_statuses if s == EVIDENCE_CONDITIONAL)
+    insufficient = sum(1 for s in evidence_statuses if s == EVIDENCE_INSUFFICIENT)
+    pending_llm = sum(1 for r in llm_requests if r.get("status") == LLM_REQUEST_PENDING)
+    changed_decision_objects = tuple(dict.fromkeys(e["decision_object_id"] for e in change_ledger))
+
+    return {
+        "run_timestamp": run_timestamp,
+        "engine_version": engine_version,
+        "canon_version": canon_version,
+        "data_timestamp": data_timestamp,
+        "research_timestamp": research_timestamp,
+        "active_tactical_states": tuple(active_tactical_states),
+        "changed_decision_objects": changed_decision_objects,
+        "evidence_complete_count": complete,
+        "evidence_conditional_count": conditional,
+        "evidence_insufficient_count": insufficient,
+        "normalization_exception_count": conditional + insufficient,
+        "manual_review_queue_size": len(review_queue),
+        "pending_llm_requests_count": pending_llm,
+        "pipeline_health": pipeline_health,
+        "last_successful_run": last_successful_run,
+        "cost_usage_status": cost_usage_status,
+        "backup_status": backup_status,
+        "market_state": market_state,  # INTERFACE GAP -- see module docstring
+    }
diff --git a/tests/test_niklar_state_contracts.py b/tests/test_niklar_state_contracts.py
new file mode 100644
index 0000000..09e6d1a
--- /dev/null
+++ b/tests/test_niklar_state_contracts.py
@@ -0,0 +1,324 @@
+"""Tests for ops/niklar_state_contracts.py — the pre-live architecture's
+Batch 1 contracts/state layer (evidence status, intraday observation,
+change ledger, review queue, LLM review request, current state snapshot).
+
+Uses synthetic fixture data only (no real tickers/prices/research).
+"""
+import json
+import unittest
+
+from ops.niklar_state_contracts import (
+    EVIDENCE_COMPLETE,
+    EVIDENCE_CONDITIONAL,
+    EVIDENCE_INSUFFICIENT,
+    LLM_REQUEST_PENDING,
+    OBSERVATION_SOURCE_MANUAL_ENTRY,
+    OBSERVATION_SOURCE_SCREENSHOT_OCR,
+    REVIEW_REASON_DATA_QUALITY_ANOMALY,
+    REVIEW_REASON_NORMALIZATION_EXCEPTION,
+    append_change,
+    append_review_entry,
+    build_current_state_snapshot,
+    compute_evidence_status,
+    new_change_ledger_entry,
+    new_intraday_observation,
+    new_llm_review_request,
+    new_review_queue_entry,
+)
+
+
+class ComputeEvidenceStatusTests(unittest.TestCase):
+    def test_all_resolved_is_complete_with_no_affected_fields(self):
+        status, affected = compute_evidence_status(["a", "b"], ["a", "b", "c"])
+        self.assertEqual(status, EVIDENCE_COMPLETE)
+        self.assertEqual(affected, ())
+
+    def test_empty_required_fields_is_trivially_complete(self):
+        status, affected = compute_evidence_status([], [])
+        self.assertEqual(status, EVIDENCE_COMPLETE)
+        self.assertEqual(affected, ())
+
+    def test_missing_non_material_field_is_conditional(self):
+        status, affected = compute_evidence_status(
+            ["a", "b"], ["a"], materially_dependent_fields=["c"]
+        )
+        self.assertEqual(status, EVIDENCE_CONDITIONAL)
+        self.assertEqual(affected, ("b",))
+
+    def test_missing_material_field_is_insufficient(self):
+        status, affected = compute_evidence_status(
+            ["a", "b"], ["a"], materially_dependent_fields=["b"]
+        )
+        self.assertEqual(status, EVIDENCE_INSUFFICIENT)
+        self.assertEqual(affected, ("b",))
+
+    def test_mixed_material_and_non_material_missing_reports_only_material_as_insufficient_cause(self):
+        status, affected = compute_evidence_status(
+            ["a", "b", "c"], [], materially_dependent_fields=["b"]
+        )
+        self.assertEqual(status, EVIDENCE_INSUFFICIENT)
+        self.assertEqual(affected, ("b",))
+
+
+class NewIntradayObservationTests(unittest.TestCase):
+    def test_builds_expected_shape(self):
+        obs = new_intraday_observation(
+            ticker="TEST",
+            field="Current Reference",
+            value="10.50",
+            observed_at="2026-08-09T10:00:00+08:00",
+            source=OBSERVATION_SOURCE_MANUAL_ENTRY,
+        )
+        self.assertEqual(obs["ticker"], "TEST")
+        self.assertEqual(obs["source"], OBSERVATION_SOURCE_MANUAL_ENTRY)
+        self.assertIsNone(obs["confidence"])
+        self.assertIsNone(obs["source_detail"])
+
+    def test_accepts_optional_confidence_and_source_detail(self):
+        obs = new_intraday_observation(
+            ticker="TEST",
+            field="Current Reference",
+            value="10.50",
+            observed_at="2026-08-09T10:00:00+08:00",
+            source=OBSERVATION_SOURCE_SCREENSHOT_OCR,
+            confidence="HIGH",
+            source_detail="broker-app-screenshot-1",
+        )
+        self.assertEqual(obs["confidence"], "HIGH")
+        self.assertEqual(obs["source_detail"], "broker-app-screenshot-1")
+
+    def test_unrecognized_source_raises(self):
+        with self.assertRaises(ValueError):
+            new_intraday_observation(
+                ticker="TEST",
+                field="Current Reference",
+                value="10.50",
+                observed_at="2026-08-09T10:00:00+08:00",
+                source="TELEPATHY",
+            )
+
+    def test_is_json_serializable(self):
+        obs = new_intraday_observation(
+            ticker="TEST",
+            field="Current Reference",
+            value="10.50",
+            observed_at="2026-08-09T10:00:00+08:00",
+            source=OBSERVATION_SOURCE_MANUAL_ENTRY,
+        )
+        json.dumps(obs)  # must not raise
+
+
+class ChangeLedgerTests(unittest.TestCase):
+    def test_new_entry_builds_expected_shape_with_default_relevant_inputs(self):
+        entry = new_change_ledger_entry(
+            timestamp="2026-08-09T10:00:00+08:00",
+            decision_object_id="TEST-DO-1",
+            decision_object_version="v2",
+            reason_code="THESIS_UPGRADE",
+            rule_fired="momentum_confirmation",
+        )
+        self.assertEqual(entry["relevant_inputs"], ())
+
+    def test_append_does_not_mutate_original_ledger(self):
+        original = (new_change_ledger_entry(
+            timestamp="t1", decision_object_id="A", decision_object_version="v1",
+            reason_code="R", rule_fired="F",
+        ),)
+        entry2 = new_change_ledger_entry(
+            timestamp="t2", decision_object_id="B", decision_object_version="v1",
+            reason_code="R", rule_fired="F",
+        )
+        appended = append_change(original, entry2)
+        self.assertEqual(len(original), 1)
+        self.assertEqual(len(appended), 2)
+        self.assertIs(appended[1], entry2)
+
+    def test_is_json_serializable(self):
+        entry = new_change_ledger_entry(
+            timestamp="t1", decision_object_id="A", decision_object_version="v1",
+            reason_code="R", rule_fired="F", relevant_inputs=["x", "y"],
+        )
+        json.dumps(entry)
+
+
+class ReviewQueueTests(unittest.TestCase):
+    def test_new_entry_builds_expected_shape(self):
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_NORMALIZATION_EXCEPTION,
+            detail="unparseable disclosure date",
+            raised_at="2026-08-09T10:00:00+08:00",
+            ticker="TEST",
+        )
+        self.assertEqual(entry["reason"], REVIEW_REASON_NORMALIZATION_EXCEPTION)
+        self.assertEqual(entry["ticker"], "TEST")
+
+    def test_ticker_defaults_to_none_for_non_ticker_scoped_reasons(self):
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_DATA_QUALITY_ANOMALY,
+            detail="pipeline-wide anomaly",
+            raised_at="2026-08-09T10:00:00+08:00",
+        )
+        self.assertIsNone(entry["ticker"])
+
+    def test_unrecognized_reason_raises(self):
+        with self.assertRaises(ValueError):
+            new_review_queue_entry(reason="VIBES_ARE_OFF", detail="x", raised_at="t")
+
+    def test_append_does_not_mutate_original_queue(self):
+        original: tuple = ()
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_DATA_QUALITY_ANOMALY, detail="x", raised_at="t"
+        )
+        appended = append_review_entry(original, entry)
+        self.assertEqual(original, ())
+        self.assertEqual(len(appended), 1)
+
+    def test_is_json_serializable(self):
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_DATA_QUALITY_ANOMALY, detail="x", raised_at="t"
+        )
+        json.dumps(entry)
+
+
+class LlmReviewRequestTests(unittest.TestCase):
+    def test_always_starts_pending(self):
+        request = new_llm_review_request(
+            ticker="TEST",
+            evidence_description="ambiguous disclosure wording",
+            failure_reason="parser could not classify materiality",
+            materiality="HIGH",
+            affected_dimensions=["thesis_health"],
+            deterministic_attempts_made=["regex_parse", "keyword_match"],
+            requested_at="2026-08-09T10:00:00+08:00",
+        )
+        self.assertEqual(request["status"], LLM_REQUEST_PENDING)
+
+    def test_no_keyword_argument_exists_to_bypass_pending_status(self):
+        # A future maintainer adding a `status=` kwarg here would be
+        # exactly the regression this test guards against -- owner
+        # authorization must stay a separate explicit action.
+        with self.assertRaises(TypeError):
+            new_llm_review_request(  # type: ignore[call-arg]
+                ticker="TEST",
+                evidence_description="x",
+                failure_reason="x",
+                materiality="LOW",
+                affected_dimensions=[],
+                deterministic_attempts_made=[],
+                requested_at="t",
+                status="OWNER_AUTHORIZED",
+            )
+
+    def test_is_json_serializable(self):
+        request = new_llm_review_request(
+            ticker="TEST", evidence_description="x", failure_reason="x",
+            materiality="LOW", affected_dimensions=[], deterministic_attempts_made=[],
+            requested_at="t",
+        )
+        json.dumps(request)
+
+
+class BuildCurrentStateSnapshotTests(unittest.TestCase):
+    def test_counts_evidence_statuses_correctly(self):
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t",
+            engine_version="v1",
+            canon_version="c1",
+            change_ledger=(),
+            review_queue=(),
+            llm_requests=(),
+            evidence_statuses=[
+                EVIDENCE_COMPLETE, EVIDENCE_COMPLETE,
+                EVIDENCE_CONDITIONAL,
+                EVIDENCE_INSUFFICIENT, EVIDENCE_INSUFFICIENT, EVIDENCE_INSUFFICIENT,
+            ],
+        )
+        self.assertEqual(snapshot["evidence_complete_count"], 2)
+        self.assertEqual(snapshot["evidence_conditional_count"], 1)
+        self.assertEqual(snapshot["evidence_insufficient_count"], 3)
+        self.assertEqual(snapshot["normalization_exception_count"], 4)
+
+    def test_pending_llm_requests_count_only_counts_pending(self):
+        pending = new_llm_review_request(
+            ticker="A", evidence_description="x", failure_reason="x",
+            materiality="LOW", affected_dimensions=[], deterministic_attempts_made=[],
+            requested_at="t",
+        )
+        resolved = dict(pending)
+        resolved["status"] = "RESOLVED"
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(),
+            llm_requests=[pending, resolved],
+            evidence_statuses=[],
+        )
+        self.assertEqual(snapshot["pending_llm_requests_count"], 1)
+
+    def test_manual_review_queue_size_matches_queue_length(self):
+        entry = new_review_queue_entry(
+            reason=REVIEW_REASON_DATA_QUALITY_ANOMALY, detail="x", raised_at="t"
+        )
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=[entry, entry], llm_requests=(),
+            evidence_statuses=[],
+        )
+        self.assertEqual(snapshot["manual_review_queue_size"], 2)
+
+    def test_changed_decision_objects_deduped_preserving_first_occurrence_order(self):
+        ledger = [
+            new_change_ledger_entry(
+                timestamp="t1", decision_object_id="B", decision_object_version="v1",
+                reason_code="R", rule_fired="F",
+            ),
+            new_change_ledger_entry(
+                timestamp="t2", decision_object_id="A", decision_object_version="v1",
+                reason_code="R", rule_fired="F",
+            ),
+            new_change_ledger_entry(
+                timestamp="t3", decision_object_id="B", decision_object_version="v2",
+                reason_code="R", rule_fired="F",
+            ),
+        ]
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=ledger, review_queue=(), llm_requests=(),
+            evidence_statuses=[],
+        )
+        self.assertEqual(snapshot["changed_decision_objects"], ("B", "A"))
+
+    def test_defaults_are_not_fabricated(self):
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=(),
+            evidence_statuses=[],
+        )
+        self.assertEqual(snapshot["pipeline_health"], "UNKNOWN")
+        self.assertIsNone(snapshot["last_successful_run"])
+        self.assertIsNone(snapshot["data_timestamp"])
+        self.assertIsNone(snapshot["research_timestamp"])
+        self.assertIsNone(snapshot["cost_usage_status"])
+        self.assertIsNone(snapshot["backup_status"])
+        self.assertIsNone(snapshot["market_state"])
+
+    def test_market_state_passthrough_is_not_interpreted(self):
+        raw = {"opportunity_matrix": ["TEST1", "TEST2"]}
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=(),
+            evidence_statuses=[], market_state=raw,
+        )
+        self.assertEqual(snapshot["market_state"], raw)
+
+    def test_is_json_serializable(self):
+        snapshot = build_current_state_snapshot(
+            run_timestamp="t", engine_version="v1", canon_version="c1",
+            change_ledger=(), review_queue=(), llm_requests=(),
+            evidence_statuses=[EVIDENCE_COMPLETE],
+            active_tactical_states=["TEST"],
+        )
+        json.dumps(snapshot)
+
+
+if __name__ == "__main__":
+    unittest.main()
```

## Self-audit checklist (from the frozen review pattern)

- **Minimum Sufficient Code**: no dataclasses/Enum introduced -- follows
  `mobile_contract.py`'s existing plain-dict/bare-string-constant style
  rather than adding a new pattern for this batch alone. 6 small pure
  functions, one file.
- **Unnecessary abstraction/dependency**: none -- stdlib `typing` only,
  no new dependency.
- **Duplicate logic**: none -- each function implements exactly one
  piece the architecture doc names, no overlap.
- **Missing failure-envelope coverage**: fail-closed `ValueError` on an
  unrecognized `source` (`new_intraday_observation`) or `reason`
  (`new_review_queue_entry`) -- both tested. `new_llm_review_request`
  has no `status=` parameter at all, by design, so PENDING can't be
  bypassed at construction -- tested via `assertRaises(TypeError)` for
  the rejected kwarg.
- **Security/privacy leakage**: none possible -- zero I/O surface,
  synthetic-only test fixtures (`TEST`/`A`/`B`), no real
  tickers/prices/research content anywhere.
- **Architecture/canon drift**: none -- no trading, scoring, tactical,
  invalidation, or Decision Object *logic* implemented; `market_state`
  (doc section 14's rankings/watchlist/positions field) is a generic
  optional passthrough marked INTERFACE GAP, not a guessed sub-schema.

## Real-world verification (not just local)

Real GitHub Actions run for this commit: `31276045591`
(`joiedevivre02/niklar-stocks`, workflow `ai-qa.yml`) --
`status: "completed"`, `conclusion: "success"`. Local: `unittest`
122/122 (95->122, the 27 new tests above), `mypy` 0 issues across 14
files, PWA smoke test unaffected.
