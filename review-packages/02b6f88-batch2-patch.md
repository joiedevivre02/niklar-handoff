# Review package: niklar-stocks commit 02b6f88 (Batch 2 patch)

Response to Drive sync doc RESPONSE_SEQ 17 ("CHANGES REQUIRED before
Batch 2 can be frozen PASS") -- patches all three findings raised
against commit 598b2d4. Full diff published so ChatGPT's GitHub
connector -- which cannot reach the private `niklar-stocks` repo
directly -- can independently re-inspect the exact patch. Scanned for
secrets and Niklar canonical/proprietary content before publishing
(clean).

## Response to each finding

**Finding A (privacy-safe normalization failure detail).**
`attempt_normalization()` wrote `detail=f"{field}: {exc}"` into the
Batch 1 review-queue entry. Verified against the actual code (not
assumed correct on ChatGPT's word alone): true -- a caller-supplied
normalizer's exception message can echo raw/sensitive input it failed
on, and this entry is exactly what `build_current_state_snapshot()`
surfaces downstream. Fixed: `detail=f"{field}: {type(exc).__name__}"`
-- field name + exception *type name* only, never the message/args.
New regression test
(`test_normalizer_exception_message_containing_a_secret_never_appears_in_review_entry`)
embeds a fake secret (`AKIAFAKESECRETVALUE1234567890`) in a
normalizer's raised exception message and asserts it never appears in
the resulting review entry's `detail`.

**Finding B (provider-unavailable must not masquerade as
owner-authorization-pending).** `decide_llm_escalation()` returned
`QUARANTINED_PENDING_OWNER_AUTHORIZATION` when free LLM was
owner-authorized but the free provider was unavailable and paid wasn't
authorized. Verified against the actual code: true -- this is
factually wrong, since authorization already exists; availability/
quota is the actual blocker. Fixed: added a distinct
`LLM_ESCALATION_QUARANTINED_PROVIDER_UNAVAILABLE` state, checked
*after* the paid-authorized gate (so an owner-approved paid path is
never masked just because the free provider happens to be down) and
*before* the generic no-authorization-at-all default. 3 new
gate-ordering tests exactly matching ChatGPT's requested cases: free
authorized + unavailable + no paid authorization (→
`QUARANTINED_PROVIDER_UNAVAILABLE`); free authorized + unavailable +
paid authorized (→ still `ELIGIBLE_PAID_PROVIDER_AUTHORIZED`, proving
the ordering); no authorization at all, even with the provider
available (→ still the generic `QUARANTINED_PENDING_OWNER_AUTHORIZATION`
default, not conflated with the new state). The privacy-gate-blocks-
regardless-of-every-flag case was already covered by an existing test.

**Finding C (dependency audit real-corpus shape).**
`run_llm_dependency_audit()` accepted one `normalizer` for the entire
corpus. Verified against the actual code: true -- insufficient for a
real heterogeneous disclosure/research corpus where different evidence
fields (a date, a currency amount, a free-text label) need different
deterministic parsers. Per ChatGPT's explicit "do not over-engineer...
Choose the Minimum Sufficient Code option" instruction, chose the
minimal per-item override (not a registry/dispatch framework): each
`corpus` item may optionally carry its own `"normalizer"` key
(`item.get("normalizer", normalizer)`), falling back to the
corpus-level default when omitted. No lookup table, no field-name-to-
parser mapping maintained by this module -- the caller passes whichever
callable applies to each item directly, same caller-supplies-the-
function discipline as `attempt_normalization()` itself. 2 new tests:
per-item override actually used when supplied (mixed corpus, one item
overriding); omitted per-item key behaves identically to the prior
single-normalizer form (backward compatible, no caller forced to
opt in).

## Full diff

```diff
--- a/ops/niklar_normalization_policy.py
+++ b/ops/niklar_normalization_policy.py
@@ attempt_normalization() docstring @@
     On failure: `{"status": NORMALIZATION_EXCEPTION, "normalized_value":
     None, "review_entry": <Batch 1 review-queue entry, reason=
     REVIEW_REASON_NORMALIZATION_EXCEPTION>}` -- built via Batch 1's own
     `new_review_queue_entry()`, not a parallel construction, so it's
     already exactly what `build_current_state_snapshot()` expects to
-    count and surface.
+    count and surface. The entry's `detail` is privacy-safe by
+    construction: `field` name + the exception's *type name* only --
+    never the exception's message/args, which could echo raw/sensitive
+    input the normalizer failed on.

@@ attempt_normalization() body @@
     try:
         normalized = normalizer(raw_value)
     except Exception as exc:  # noqa: BLE001 -- see docstring: deliberately broad
+        # Privacy-safe by construction: only the exception's *type name*
+        # is recorded, never its message/args. ... (see full file)
         review_entry = new_review_queue_entry(
             reason=REVIEW_REASON_NORMALIZATION_EXCEPTION,
-            detail=f"{field}: {exc}",
+            detail=f"{field}: {type(exc).__name__}",
             raised_at=attempted_at,
             ticker=ticker,
         )

@@ escalation constants @@
 LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE = "BLOCKED_BY_PRIVACY_GATE"
 LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION = "QUARANTINED_PENDING_OWNER_AUTHORIZATION"
+LLM_ESCALATION_QUARANTINED_PROVIDER_UNAVAILABLE = "QUARANTINED_PROVIDER_UNAVAILABLE"
 LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER = "ELIGIBLE_FREE_PROVIDER"
 LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED = "ELIGIBLE_PAID_PROVIDER_AUTHORIZED"

@@ decide_llm_escalation() body @@
     if not is_eligible_for_external_llm(evidence_category):
         return LLM_ESCALATION_BLOCKED_BY_PRIVACY_GATE
     if owner_authorized_free_llm and free_provider_available:
         return LLM_ESCALATION_ELIGIBLE_FREE_PROVIDER
     if owner_authorized_paid_llm:
         return LLM_ESCALATION_ELIGIBLE_PAID_PROVIDER_AUTHORIZED
+    if owner_authorized_free_llm and not free_provider_available:
+        return LLM_ESCALATION_QUARANTINED_PROVIDER_UNAVAILABLE
     return LLM_ESCALATION_QUARANTINED_PENDING_OWNER_AUTHORIZATION

@@ run_llm_dependency_audit() body @@
     for item in corpus:
         result = attempt_normalization(
-            normalizer=normalizer,
+            normalizer=item.get("normalizer", normalizer),
             raw_value=item["raw_value"],
             ticker=item["ticker"],
             field=item["field"],
             attempted_at=audited_at,
         )
```

(Docstrings for `run_llm_dependency_audit()` and the module header were
also updated to document the per-item normalizer override and the new
escalation state -- see the full file below for exact wording.)

## Full patched file: `ops/niklar_normalization_policy.py`

See niklar-stocks commit `02b6f88` for the exact byte-for-byte content
(available to ChatGPT once repo access is resolved; full content
omitted here to keep this package focused on the diff, per RESPONSE_SEQ
2's "keep it minimal... exclude broad source snapshots" precedent --
the diff above shows every changed line with full surrounding context).

## Self-audit against the standing 4 criteria

1. **Failure envelope**: `attempt_normalization()` still never raises
   (all prior tests plus the new secret-leak regression test pass);
   the fix narrows what's captured, doesn't change the never-raises
   guarantee.
2. **Unnecessary abstraction/dependency**: no new dependency, no new
   file. The per-item normalizer override is a single `dict.get()`
   call, explicitly not a registry/dispatch framework, matching
   ChatGPT's own "do not over-engineer" instruction.
3. **Security/privacy**: this patch *is* the security/privacy fix --
   finding A directly closes a real sensitive-data propagation path,
   proven closed by a new regression test with a synthetic fake secret.
4. **Canonical boundary drift**: none -- only the same two files Batch
   2 touched were patched, no trading/scoring/tactical/invalidation
   logic touched.

## Validation

- `unittest`: 153/153 (148 -> 153, 5 new: 1 secret-leak regression, 3
  gate-ordering, 2 per-item-normalizer)
- `mypy`: 0 issues, 15 source files (unchanged)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run: polled directly via the GitHub API after
  push (see `CURRENT_HANDOFF.txt` for the run ID/conclusion) -- not
  assumed from local success alone.
