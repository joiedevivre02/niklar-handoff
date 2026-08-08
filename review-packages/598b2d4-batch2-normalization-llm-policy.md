# Review package: niklar-stocks commit 598b2d4 (Batch 2)

Released by Drive sync doc RESPONSE_SEQ 14, after Batch 1's patch
(commit 221020b) passed independent re-review (RESPONSE_SEQ 10's
findings, checkpointed at niklar-handoff HANDOFF_SEQ 8). Full new-file
content published in full so ChatGPT's GitHub connector -- which
cannot reach the private `niklar-stocks` repo directly -- can
independently inspect the exact code. Scanned for secrets and Niklar
canonical/proprietary content before publishing (clean).

Per RESPONSE_SEQ 14's explicit scope boundary: "non-blocking
normalization exception path, free-LLM/owner-gating policy
foundations, and deterministic LLM-dependency audit tooling...
incorporate the later approved Section 20 direction where Batch 2
naturally owns it... do not jump ahead into PWA Search UX, Apple
Vision, ScreenCaptureKit, Slack/PWA controls, or other later batches
except for the minimum contracts/interfaces required to avoid
incompatibility." This package covers exactly that scope -- no more.

## Commit message (verbatim)

```
commit 598b2d4d55a9cc8a34891df5ef1bdbab3132e347
Author: Claude <noreply@anthropic.com>
Date:   Sat Aug 8 20:59:27 2026 +0000

    Batch 2: normalization exception path + owner-gated LLM policy foundations

    Released by Drive sync doc RESPONSE_SEQ 14 after Batch 1's patch
    (commit 221020b) passed independent re-review. Per RESPONSE_SEQ 14's
    explicit scope: "non-blocking normalization exception path, free-LLM/
    owner-gating policy foundations, and deterministic LLM-dependency
    audit tooling... incorporate the later approved Section 20 direction
    where Batch 2 naturally owns it... do not jump ahead" -- no PWA
    Search UX, Apple Vision, ScreenCaptureKit, or Slack/PWA controls
    included here.

    New file ops/niklar_normalization_policy.py, three pieces:
    - attempt_normalization(): non-blocking wrapper, deliberately broad
      except Exception, degrades a normalizer failure to a Batch 1
      REVIEW_REASON_NORMALIZATION_EXCEPTION review-queue entry (built via
      Batch 1's own new_review_queue_entry(), not a parallel
      construction) instead of raising past itself.
    - is_eligible_for_external_llm() / decide_llm_escalation(): pure,
      no-I/O owner-gated LLM escalation policy. Fail-closed privacy
      allowlist (only PUBLIC_DISCLOSURE/PUBLIC_NEWS/PUBLIC_MARKET_DATA
      are eligible); free-tier path requires owner authorization AND
      provider availability, paid path requires its own separate
      authorization: default outcome with no gate satisfied is always
      QUARANTINED_PENDING_OWNER_AUTHORIZATION, matching the architecture
      doc's routine-target-is-zero-LLM-calls stance. Incorporates Section
      20's free-provider-eligible/paid-requires-explicit-authorization
      shape without any provider SDK or network call -- foundations only.
    - run_llm_dependency_audit(): replays a caller-supplied corpus
      through attempt_normalization(), reports deterministic pass/fail
      counts and rates plus per-failure ticker/field/reason/materiality/
      affected-dimensions. None (not fabricated 0.0/1.0) for an empty
      corpus. Deliberately does not classify "genuine semantic reasoning
      need" vs. "parser defect" -- documented as out of scope for a
      deterministic function.

    New file tests/test_niklar_normalization_policy.py, 18 tests
    including one proving attempt_normalization() never raises even for
    an exception type unrelated to normal parsing failures.

    No existing file modified, no new dependency -- Minimum Sufficient
    Code. No canonical trading/scoring/tactical/invalidation logic
    touched; pure policy/state scaffolding, same discipline as Batch 1.

    Validation: unittest 148/148 (18 new), mypy 0/15, PWA smoke PASS.
```

## Full new file: `ops/niklar_normalization_policy.py`

```python
"""Niklar pre-live normalization/LLM-escalation policy layer.

Batch 2 of the owner-approved pre-live architecture (Google Drive
`NIKLAR_PRELIVE_ARCHITECTURE_DECISIONS_CURRENT`, frozen 2026-08-09,
released for implementation by RESPONSE_SEQ 14 after Batch 1's
independent re-review passed; see niklar-handoff `DECISIONS_CURRENT.txt`
for the checkpoints recording both batches). Pure, deterministic
functions only -- no I/O, no credentials, no actual LLM provider call,
same discipline as `niklar_state_contracts.py` (Batch 1). Builds on top
of Batch 1's contracts (`new_review_queue_entry`,
`REVIEW_REASON_NORMALIZATION_EXCEPTION`) rather than duplicating them.

Per RESPONSE_SEQ 14's explicit scope: "free-LLM/owner-gating policy
FOUNDATIONS" -- the decision logic for whether/when an LLM escalation
would be *eligible*, not the actual provider wiring. Section 20's
GPT-OSS-120B/Groq direction and Gemini's CI/QA-diagnosis-only role
inform the shape of these functions (a provider-independent boundary,
free tier gated the same as paid) but this module makes zero network
calls and imports no provider SDK -- "do not jump ahead ... except for
the minimum contracts/interfaces required to avoid incompatibility"
(RESPONSE_SEQ 14).

Three pieces, matching Batch 2's scope exactly:
  1. `attempt_normalization()` -- the non-blocking normalization
     exception path (architecture doc sections 2, 3). Never raises past
     this function -- a normalizer failure degrades to a Batch 1
     `REVIEW_REASON_NORMALIZATION_EXCEPTION` review-queue entry, never
     an unhandled exception that would stop Niklar.
  2. `is_eligible_for_external_llm()` / `decide_llm_escalation()` -- the
     owner-gated LLM policy (sections 2, 4; Section 20's routing
     policy). Routine production target is ZERO LLM calls
     (architecture doc section 2) -- `decide_llm_escalation()` only
     ever classifies whether an escalation *would be* eligible; it
     never invokes anything itself, and the default outcome whenever
     owner authorization is absent is always the safe, non-escalating
     one (`LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION`).
  3. `run_llm_dependency_audit()` -- the pre-live LLM dependency audit
     (section 4): replay a corpus through `attempt_normalization()` and
     report pass/fail counts, exact failing tickers/fields/reasons, and
     materiality/affected-dimensions context. Deliberately does NOT
     attempt to classify a failure as "genuine semantic reasoning need"
     vs. "parser/normalizer defect" -- section 4 asks for that
     distinction, but making it requires judgment about the actual
     failure's substance that a generic deterministic function cannot
     supply; this function reports the structured failure list a human/
     engineer needs to make that call, it doesn't guess at it.
"""
from __future__ import annotations

from typing import Any, Callable, Mapping, Sequence

from .niklar_state_contracts import REVIEW_REASON_NORMALIZATION_EXCEPTION, new_review_queue_entry

# --- Non-blocking normalization exception path (architecture doc sections 2, 3) ---

NORMALIZATION_SUCCESS = "SUCCESS"
NORMALIZATION_EXCEPTION = "EXCEPTION"


def attempt_normalization(
    *,
    normalizer: Callable[[Any], Any],
    raw_value: Any,
    ticker: str,
    field: str,
    attempted_at: str,
) -> dict:
    """Try `normalizer(raw_value)`. Never raises past this function --
    the deliberately broad `except Exception` below is the entire point
    of this function: a normalization failure must degrade to a visible
    review-queue entry, not stop Niklar (architecture doc section 3,
    "non-blocking evidence degradation"). This is the same fail-safe
    pattern as `scripts/ai_qa/gemini_client.py`'s broad `except
    (OSError, http.client.HTTPException)` -- catching broadly is
    correct here, not sloppy, because the caller has no way to
    enumerate every failure mode a caller-supplied normalizer might
    raise, and *any* of them must degrade the same way.

    On success: `{"status": NORMALIZATION_SUCCESS, "normalized_value":
    <result>, "review_entry": None}`.

    On failure: `{"status": NORMALIZATION_EXCEPTION, "normalized_value":
    None, "review_entry": <Batch 1 review-queue entry, reason=
    REVIEW_REASON_NORMALIZATION_EXCEPTION>}` -- built via Batch 1's own
    `new_review_queue_entry()`, not a parallel construction, so it's
    already exactly what `build_current_state_snapshot()` expects to
    count and surface.

    Deliberately does NOT decide `EVIDENCE_CONDITIONAL` vs.
    `EVIDENCE_INSUFFICIENT` -- that requires knowing every other
    required field's status too (Batch 1's `compute_evidence_status()`
    already owns that aggregate judgment); a single field's
    normalization outcome doesn't have that visibility on its own, so
    this function reports only the outcome, and the caller composes it
    with `compute_evidence_status()` once every field for a computation
    has been attempted.

    `normalizer` is expected to be a pure, deterministic parsing
    function (e.g. a date/number parser) -- not a network call. This
    module makes no I/O itself; whether a caller-supplied `normalizer`
    does is the caller's responsibility, same as `mobile_contract.py`
    accepting already-resolved rows rather than fetching them itself.
    """
    try:
        normalized = normalizer(raw_value)
    except Exception as exc:  # noqa: BLE001 -- see docstring: deliberately broad
        review_entry = new_review_queue_entry(
            reason=REVIEW_REASON_NORMALIZATION_EXCEPTION,
            detail=f"{field}: {exc}",
            raised_at=attempted_at,
            ticker=ticker,
        )
        return {"status": NORMALIZATION_EXCEPTION, "normalized_value": None, "review_entry": review_entry}
    return {"status": NORMALIZATION_SUCCESS, "normalized_value": normalized, "review_entry": None}


# --- Privacy/public-data gate (architecture doc section 2; Section 20 "PRIVACY GATE") ---

EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE = "PUBLIC_DISCLOSURE"
EVIDENCE_CATEGORY_PUBLIC_NEWS = "PUBLIC_NEWS"
EVIDENCE_CATEGORY_PUBLIC_MARKET_DATA = "PUBLIC_MARKET_DATA"
EVIDENCE_CATEGORY_PRIVATE_PORTFOLIO = "PRIVATE_PORTFOLIO"
EVIDENCE_CATEGORY_CREDENTIAL_OR_SECRET = "CREDENTIAL_OR_SECRET"
EVIDENCE_CATEGORY_PROPRIETARY_RESEARCH = "PROPRIETARY_RESEARCH"

# Fail-closed allowlist: only categories Section 20 explicitly names as
# "may be eligible" ("Public issuer/PSE/news evidence") are included.
# Everything else -- including any category not listed here at all --
# is NOT eligible. A new category must be added deliberately, not
# implied, same fail-closed discipline as Batch 1's
# `_VALID_OBSERVATION_SOURCES`/`_VALID_REVIEW_REASONS`.
_EXTERNAL_LLM_ELIGIBLE_CATEGORIES = frozenset(
    {
        EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE,
        EVIDENCE_CATEGORY_PUBLIC_NEWS,
        EVIDENCE_CATEGORY_PUBLIC_MARKET_DATA,
    }
)


def is_eligible_for_external_llm(evidence_category: str) -> bool:
    """Fail-closed: an unrecognized category (including the explicitly
    private ones defined above, or any string not in the allowlist at
    all) is never eligible. "Never automatically send private
    portfolio details, credentials, secrets, proprietary internal
    notes/research, or sensitive user data to a free external model"
    (Section 20, PRIVACY GATE) -- the allowlist encodes the inverse of
    that sentence directly, so a caller can't silently widen eligibility
    just by inventing a new category string.
    """
    return evidence_category in _EXTERNAL_LLM_ELIGIBLE_CATEGORIES


# --- Owner-gated LLM escalation policy (architecture doc section 2; Section 20 ROUTING POLICY) ---

LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE = "BLOCKED_BY_PRIVACY_GATE"
LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION = "QUARANTINED_PENDING_OWNER_AUTHORIZATION"
LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER = "ELIGIBLE_FREE_PROVIDER"
LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED = "ELIGIBLE_PAID_PROVIDER_AUTHORIZED"


def decide_llm_escalation(
    *,
    evidence_category: str,
    owner_authorized_free_llm: bool,
    owner_authorized_paid_llm: bool,
    free_provider_available: bool,
) -> str:
    """Pure decision, no I/O, no actual LLM call -- this function only
    ever classifies whether an escalation *would be* eligible. It never
    invokes anything itself, and nothing that calls this function is
    obligated to act on an eligible result; the routine production
    target remains ZERO LLM calls (architecture doc section 2).

    Fail-closed order, each gate independent of the others:
    1. Privacy gate first -- an ineligible `evidence_category` blocks
       escalation outright, regardless of every other flag. Checked via
       `is_eligible_for_external_llm()`, not duplicated here.
    2. Free-LLM path requires BOTH `owner_authorized_free_llm` AND
       `free_provider_available` -- "owner authorization is required
       before any production LLM resolution call" (section 2) applies
       to the free tier too, not just paid; a free provider being
       merely configured/reachable is not itself authorization.
    3. Paid-LLM path requires `owner_authorized_paid_llm` explicitly --
       never implied by free-tier authorization, matching Section 20's
       "Paid model escalation only after explicit owner authorization"
       as a genuinely separate gate.
    4. Otherwise: quarantined pending owner authorization -- the
       default, safe outcome when no gate above is satisfied. This is
       the same outcome as a provider being unavailable/quota-limited,
       per Section 20's "continue unaffected computation with visible
       evidence caveats + manual review. Do not block the workflow" --
       this function's default already IS that safe outcome, not an
       error case.
    """
    if not is_eligible_for_external_llm(evidence_category):
        return LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE
    if owner_authorized_free_llm and free_provider_available:
        return LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER
    if owner_authorized_paid_llm:
        return LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED
    return LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION


# --- Deterministic LLM dependency audit tooling (architecture doc section 4) ---


def run_llm_dependency_audit(
    *,
    corpus: Sequence[Mapping[str, Any]],
    normalizer: Callable[[Any], Any],
    audited_at: str,
) -> dict:
    """Replay `corpus` through `attempt_normalization()` and report
    exactly the fields architecture doc section 4 asks for: total
    evidence/disclosures tested, deterministic PASS/FAIL count and rate,
    and -- for each failure -- the exact ticker/field, failure reason,
    and materiality/affected-dimensions context (passed through from
    the corpus item, not invented). Pure aggregation, no I/O of its
    own; `corpus` is caller-supplied (a real pre-live run would replay
    the actual disclosure/research corpus -- this function doesn't
    fetch one).

    Each `corpus` item: `{"ticker": str, "field": str, "raw_value":
    Any, "materiality": str (optional), "affected_dimensions":
    Sequence[str] (optional)}`.

    Does NOT classify a failure as "genuine semantic reasoning need"
    vs. "parser/normalizer defect" (section 4 asks for that
    distinction) -- see this module's docstring for why that judgment
    call is out of scope for a generic deterministic function; this
    returns the structured failure list a human/engineer needs to make
    it, not a guess.

    `deterministic_pass_rate`/`deterministic_fail_rate` are `None` for
    an empty corpus (never a fabricated `0.0`/`1.0` for zero
    denominator).
    """
    total = len(corpus)
    failed_items = []
    passed = 0
    for item in corpus:
        result = attempt_normalization(
            normalizer=normalizer,
            raw_value=item["raw_value"],
            ticker=item["ticker"],
            field=item["field"],
            attempted_at=audited_at,
        )
        if result["status"] == NORMALIZATION_SUCCESS:
            passed += 1
        else:
            failed_items.append(
                {
                    "ticker": item["ticker"],
                    "field": item["field"],
                    "failure_reason": result["review_entry"]["detail"],
                    "materiality": item.get("materiality"),
                    "affected_dimensions": tuple(item.get("affected_dimensions", ())),
                }
            )
    failed = total - passed

    return {
        "audited_at": audited_at,
        "total_tested": total,
        "deterministic_pass_count": passed,
        "deterministic_fail_count": failed,
        "deterministic_pass_rate": (passed / total) if total else None,
        "deterministic_fail_rate": (failed / total) if total else None,
        "failed_items": tuple(failed_items),
    }
```

## Full new file: `tests/test_niklar_normalization_policy.py`

18 tests -- happy path, fail-closed edge cases for the privacy
allowlist and escalation gate ordering, and (per the standing test
rigor established in Batch 1) a dedicated test proving
`attempt_normalization()` never raises even for an exception type
unrelated to ordinary parsing failures (`RuntimeError`, not just
`ValueError`).

```python
"""Tests for ops/niklar_normalization_policy.py -- the pre-live
architecture's Batch 2 normalization-exception path, owner-gated LLM
escalation policy, and deterministic LLM dependency audit tooling.

Uses synthetic fixture data only (no real tickers/prices/research).
"""
import unittest

from ops.niklar_normalization_policy import (
    EVIDENCE_CATEGORY_CREDENTIAL_OR_SECRET,
    EVIDENCE_CATEGORY_PRIVATE_PORTFOLIO,
    EVIDENCE_CATEGORY_PROPRIETARY_RESEARCH,
    EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE,
    EVIDENCE_CATEGORY_PUBLIC_MARKET_DATA,
    EVIDENCE_CATEGORY_PUBLIC_NEWS,
    LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE,
    LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER,
    LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED,
    LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION,
    NORMALIZATION_EXCEPTION,
    NORMALIZATION_SUCCESS,
    attempt_normalization,
    decide_llm_escalation,
    is_eligible_for_external_llm,
    run_llm_dependency_audit,
)
from ops.niklar_state_contracts import REVIEW_REASON_NORMALIZATION_EXCEPTION


def _upper(value):
    return str(value).upper()


def _boom(value):
    raise ValueError("bad value")


class AttemptNormalizationTests(unittest.TestCase):
    def test_success_returns_normalized_value_and_no_review_entry(self):
        result = attempt_normalization(
            normalizer=_upper,
            raw_value="abc",
            ticker="ABC",
            field="ticker_symbol",
            attempted_at="2026-08-09T00:00:00Z",
        )
        self.assertEqual(result["status"], NORMALIZATION_SUCCESS)
        self.assertEqual(result["normalized_value"], "ABC")
        self.assertIsNone(result["review_entry"])

    def test_normalizer_exception_degrades_to_review_entry_not_raised(self):
        result = attempt_normalization(
            normalizer=_boom,
            raw_value="anything",
            ticker="XYZ",
            field="market_cap",
            attempted_at="2026-08-09T00:00:00Z",
        )
        self.assertEqual(result["status"], NORMALIZATION_EXCEPTION)
        self.assertIsNone(result["normalized_value"])
        entry = result["review_entry"]
        self.assertEqual(entry["reason"], REVIEW_REASON_NORMALIZATION_EXCEPTION)
        self.assertEqual(entry["ticker"], "XYZ")
        self.assertEqual(entry["raised_at"], "2026-08-09T00:00:00Z")
        self.assertIn("market_cap", entry["detail"])
        self.assertIn("bad value", entry["detail"])

    def test_never_raises_even_for_an_unexpected_exception_type(self):
        def _raises_keyboard_style(value):
            # Not a "normal" parsing failure (ValueError/TypeError) --
            # proves the except clause is genuinely broad, not narrowly
            # tuned to the cases above.
            raise RuntimeError("unexpected internal failure")

        try:
            result = attempt_normalization(
                normalizer=_raises_keyboard_style,
                raw_value="x",
                ticker="QQQ",
                field="some_field",
                attempted_at="2026-08-09T00:00:00Z",
            )
        except Exception as exc:  # pragma: no cover -- the assertion below is the real check
            self.fail(f"attempt_normalization() must never raise, but raised {exc!r}")
        self.assertEqual(result["status"], NORMALIZATION_EXCEPTION)
        self.assertIn("unexpected internal failure", result["review_entry"]["detail"])

    def test_review_entry_built_via_batch1_constructor_is_snapshot_compatible(self):
        result = attempt_normalization(
            normalizer=_boom,
            raw_value="x",
            ticker="AAA",
            field="field_a",
            attempted_at="2026-08-09T00:00:00Z",
        )
        entry = result["review_entry"]
        # Exactly the shape new_review_queue_entry() produces -- proves
        # this isn't a parallel/ad-hoc construction that could drift
        # from what build_current_state_snapshot() expects to count.
        self.assertEqual(set(entry.keys()), {"reason", "detail", "raised_at", "ticker"})


class IsEligibleForExternalLlmTests(unittest.TestCase):
    def test_public_categories_are_eligible(self):
        for category in (
            EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE,
            EVIDENCE_CATEGORY_PUBLIC_NEWS,
            EVIDENCE_CATEGORY_PUBLIC_MARKET_DATA,
        ):
            self.assertTrue(is_eligible_for_external_llm(category))

    def test_private_categories_are_not_eligible(self):
        for category in (
            EVIDENCE_CATEGORY_PRIVATE_PORTFOLIO,
            EVIDENCE_CATEGORY_CREDENTIAL_OR_SECRET,
            EVIDENCE_CATEGORY_PROPRIETARY_RESEARCH,
        ):
            self.assertFalse(is_eligible_for_external_llm(category))

    def test_unrecognized_category_fails_closed_not_eligible(self):
        self.assertFalse(is_eligible_for_external_llm("SOME_NEW_CATEGORY_NOBODY_DECLARED"))


class DecideLlmEscalationTests(unittest.TestCase):
    def test_privacy_gate_blocks_regardless_of_every_other_flag(self):
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PRIVATE_PORTFOLIO,
            owner_authorized_free_llm=True,
            owner_authorized_paid_llm=True,
            free_provider_available=True,
        )
        self.assertEqual(outcome, LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE)

    def test_free_provider_eligible_only_when_authorized_and_available(self):
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PUBLIC_NEWS,
            owner_authorized_free_llm=True,
            owner_authorized_paid_llm=False,
            free_provider_available=True,
        )
        self.assertEqual(outcome, LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER)

    def test_free_authorized_but_provider_unavailable_is_not_eligible_free(self):
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PUBLIC_NEWS,
            owner_authorized_free_llm=True,
            owner_authorized_paid_llm=False,
            free_provider_available=False,
        )
        self.assertEqual(outcome, LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION)

    def test_paid_path_requires_its_own_explicit_authorization(self):
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE,
            owner_authorized_free_llm=False,
            owner_authorized_paid_llm=True,
            free_provider_available=False,
        )
        self.assertEqual(outcome, LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED)

    def test_free_tier_authorization_does_not_imply_paid_eligibility(self):
        # free_provider_available False and only free authorized (not
        # paid) -- must NOT fall through to the paid-eligible outcome.
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PUBLIC_DISCLOSURE,
            owner_authorized_free_llm=True,
            owner_authorized_paid_llm=False,
            free_provider_available=False,
        )
        self.assertEqual(outcome, LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION)

    def test_no_authorization_at_all_is_quarantined_default(self):
        outcome = decide_llm_escalation(
            evidence_category=EVIDENCE_CATEGORY_PUBLIC_MARKET_DATA,
            owner_authorized_free_llm=False,
            owner_authorized_paid_llm=False,
            free_provider_available=False,
        )
        self.assertEqual(outcome, LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION)


class RunLlmDependencyAuditTests(unittest.TestCase):
    def test_empty_corpus_reports_none_rates_not_fabricated_zero_or_one(self):
        report = run_llm_dependency_audit(corpus=(), normalizer=_upper, audited_at="2026-08-09T00:00:00Z")
        self.assertEqual(report["total_tested"], 0)
        self.assertEqual(report["deterministic_pass_count"], 0)
        self.assertEqual(report["deterministic_fail_count"], 0)
        self.assertIsNone(report["deterministic_pass_rate"])
        self.assertIsNone(report["deterministic_fail_rate"])
        self.assertEqual(report["failed_items"], ())

    def test_all_pass_reports_full_pass_rate_and_no_failed_items(self):
        corpus = [
            {"ticker": "AAA", "field": "f1", "raw_value": "a"},
            {"ticker": "BBB", "field": "f2", "raw_value": "b"},
        ]
        report = run_llm_dependency_audit(corpus=corpus, normalizer=_upper, audited_at="2026-08-09T00:00:00Z")
        self.assertEqual(report["total_tested"], 2)
        self.assertEqual(report["deterministic_pass_count"], 2)
        self.assertEqual(report["deterministic_fail_count"], 0)
        self.assertEqual(report["deterministic_pass_rate"], 1.0)
        self.assertEqual(report["deterministic_fail_rate"], 0.0)
        self.assertEqual(report["failed_items"], ())

    def test_mixed_corpus_reports_exact_failing_ticker_field_and_context(self):
        corpus = [
            {"ticker": "AAA", "field": "f1", "raw_value": "a"},
            {
                "ticker": "BBB",
                "field": "f2",
                "raw_value": "b",
                "materiality": "HIGH",
                "affected_dimensions": ["valuation", "catalyst"],
            },
        ]

        def _pass_a_fail_b(value):
            if value == "b":
                raise ValueError("cannot parse")
            return value.upper()

        report = run_llm_dependency_audit(
            corpus=corpus, normalizer=_pass_a_fail_b, audited_at="2026-08-09T00:00:00Z"
        )
        self.assertEqual(report["total_tested"], 2)
        self.assertEqual(report["deterministic_pass_count"], 1)
        self.assertEqual(report["deterministic_fail_count"], 1)
        self.assertEqual(report["deterministic_pass_rate"], 0.5)
        self.assertEqual(report["deterministic_fail_rate"], 0.5)
        self.assertEqual(len(report["failed_items"]), 1)
        failure = report["failed_items"][0]
        self.assertEqual(failure["ticker"], "BBB")
        self.assertEqual(failure["field"], "f2")
        self.assertIn("cannot parse", failure["failure_reason"])
        self.assertEqual(failure["materiality"], "HIGH")
        self.assertEqual(failure["affected_dimensions"], ("valuation", "catalyst"))

    def test_failed_item_without_optional_context_defaults_are_none_and_empty(self):
        corpus = [{"ticker": "CCC", "field": "f3", "raw_value": "c"}]
        report = run_llm_dependency_audit(corpus=corpus, normalizer=_boom, audited_at="2026-08-09T00:00:00Z")
        failure = report["failed_items"][0]
        self.assertIsNone(failure["materiality"])
        self.assertEqual(failure["affected_dimensions"], ())

    def test_does_not_raise_when_normalizer_fails_for_every_item(self):
        corpus = [{"ticker": "DDD", "field": "f4", "raw_value": "d"}]
        report = run_llm_dependency_audit(corpus=corpus, normalizer=_boom, audited_at="2026-08-09T00:00:00Z")
        self.assertEqual(report["deterministic_fail_count"], 1)
        self.assertEqual(report["deterministic_pass_rate"], 0.0)


if __name__ == "__main__":
    unittest.main()
```

## Self-audit against the standing 4 criteria

1. **Failure envelope**: `attempt_normalization()`'s `except Exception`
   is deliberately broad -- the caller supplies an arbitrary
   normalizer, so any failure mode must degrade to a review-queue
   entry, never propagate. Verified by
   `test_never_raises_even_for_an_unexpected_exception_type`, which
   raises `RuntimeError` (not `ValueError`/`TypeError`, the "obvious"
   parsing exceptions) to confirm the catch isn't narrowly tuned to
   the happy-path test cases.
2. **Unnecessary abstraction/dependency**: no new dependency, no
   dataclass/Enum -- plain dicts and bare string constants, matching
   `mobile_contract.py`'s and Batch 1's established convention. Two
   files added, zero existing files touched.
3. **Security/privacy**: `is_eligible_for_external_llm()` is a
   fail-closed allowlist -- an unrecognized category (or any of the
   three explicitly private ones) is never eligible. No network call,
   no provider SDK, no credential handling anywhere in this module.
4. **Canonical boundary drift**: no trading/scoring/tactical/
   invalidation/Decision Object logic touched. Pure policy/state
   scaffolding, same as Batch 1. The three Drive-verbatim `ops/`
   modules are untouched.

## Validation

- `unittest`: 148/148 (130 -> 148, 18 new)
- `mypy`: 0 issues, 15 source files (14 -> 15)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run `31278234207` (workflow `ai-qa.yml`, commit
  `598b2d4`) polled directly via the GitHub API: `status: completed`,
  `conclusion: success` -- not assumed from local success alone.
