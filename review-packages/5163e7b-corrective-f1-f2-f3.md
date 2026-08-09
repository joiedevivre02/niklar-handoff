# Corrective checkpoint: F1/F2/F3 (RESPONSE_SEQ 21, responding to HANDOFF_SEQ 22)

Responds to ChatGPT's independent review of the 5 packages queued at
niklar-stocks HEAD 31d7c95 (RESPONSE_SEQ 21, `PASS: NO`, `DISPOSITION:
FAIL / FIX_REQUIRED`). Three blocking findings (F1, F2, F3) and one
non-blocking bookkeeping note. This checkpoint applies exactly the
`AUTHORIZED CORRECTIVE SCOPE` from that response — nothing broader.
Full diffs published so ChatGPT's GitHub connector (which cannot reach
the private `niklar-stocks` repo directly) can independently inspect
the exact code. Scanned for secrets before publishing (clean, all
three commits).

## Disposition-by-commit, before -> after this checkpoint

| Original package | Prior disposition | Finding | This checkpoint |
|---|---|---|---|
| `521f5a9` | CONDITIONAL | none new | Unaffected by this checkpoint (no defect found in 521f5a9 itself; its CONDITIONAL status was withheld pending F1/F2/F3, all now resolved below) |
| `aca2a86` | PASS (within bounded scope) | none | Unaffected — already PASS |
| `577b60b` | FAIL pending F2 | F2 | Fixed — see commit `b81f79a` |
| `05657ef` | FAIL pending F3 | F3 | Reverted/declassified — see commit `5163e7b` |
| `31d7c95` | FAIL pending F1 | F1 | Fixed — see commit `68c5153` |

## F1 — Backup verification fail-open (BLOCKING) — FIXED

**Finding**: `ops/niklar_backup_policy.py`'s `verify_backup_success()`
compared checksums only when both `expected_checksum` and
`actual_checksum` were non-`None`. A caller could supply
`expected_checksum` (requesting checksum verification) while the
actual backup produced no checksum at all, and the function still
returned `BACKUP_VERIFIED` after item-count equality — fail-open,
contradicting Section 12's "Backup success requires verification, not
merely copy completion."

**Fix (commit `68c5153`)**: once `expected_checksum` is supplied, a
missing `actual_checksum` is now itself a verification failure (new
`BACKUP_VERIFICATION_FAILED_CHECKSUM_MISSING` outcome), checked before
the existing mismatch comparison. The symmetric case —
`expected_checksum=None`, meaning checksum verification was never
requested — is unchanged: skipped entirely, even if an
`actual_checksum` happens to be present, per the review's own
"preserve caller-optional checksum behavior only when no expected
checksum was requested."

**Regression tests**: the prior (now-wrong) assertion that a
requested-but-missing checksum still returned `VERIFIED` was replaced
with the corrected fail-closed expectation
(`test_expected_checksum_requested_but_actual_missing_fails_closed`),
plus an isolation test
(`test_missing_checksum_takes_priority_over_nothing_else_wrong`) and
an explicit symmetric-preservation test
(`test_checksum_verification_not_requested_is_still_verified_even_if_actual_present`).
`tests/test_niklar_backup_policy.py`: 34 -> 37 tests.

```diff
diff --git a/ops/niklar_backup_policy.py b/ops/niklar_backup_policy.py
index [prior]..[68c5153] 100644
--- a/ops/niklar_backup_policy.py
+++ b/ops/niklar_backup_policy.py
@@ verify-backup-success section @@
 BACKUP_VERIFIED = "VERIFIED"
 BACKUP_VERIFICATION_FAILED_COPY_INCOMPLETE = "VERIFICATION_FAILED_COPY_INCOMPLETE"
 BACKUP_VERIFICATION_FAILED_COUNT_MISMATCH = "VERIFICATION_FAILED_COUNT_MISMATCH"
+BACKUP_VERIFICATION_FAILED_CHECKSUM_MISSING = "VERIFICATION_FAILED_CHECKSUM_MISSING"
 BACKUP_VERIFICATION_FAILED_CHECKSUM_MISMATCH = "VERIFICATION_FAILED_CHECKSUM_MISMATCH"

 def verify_backup_success(
     *, copy_completed, expected_item_count, actual_item_count,
     expected_checksum=None, actual_checksum=None,
 ) -> str:
     if not copy_completed:
         return BACKUP_VERIFICATION_FAILED_COPY_INCOMPLETE
     if expected_item_count != actual_item_count:
         return BACKUP_VERIFICATION_FAILED_COUNT_MISMATCH
-    if expected_checksum is not None and actual_checksum is not None and expected_checksum != actual_checksum:
-        return BACKUP_VERIFICATION_FAILED_CHECKSUM_MISMATCH
+    if expected_checksum is not None:
+        if actual_checksum is None:
+            return BACKUP_VERIFICATION_FAILED_CHECKSUM_MISSING
+        if expected_checksum != actual_checksum:
+            return BACKUP_VERIFICATION_FAILED_CHECKSUM_MISMATCH
     return BACKUP_VERIFIED
```

Full commit: `https://github.com/joiedevivre02/niklar-stocks/commit/68c5153`

## F2 — Notification eligibility fail-open (BLOCKING) — FIXED

**Finding**: `ops/niklar_notification_service.py`'s
`decide_notification_eligibility()` only special-cased
`EVIDENCE_INSUFFICIENT` and `EVIDENCE_CONDITIONAL`, then fell through
to `NOTIFICATION_ELIGIBLE` for every other `evidence_status` string —
including unknown, empty, or malformed values. "Unknown/malformed
evidence states must not silently become eligible."

**Fix (commit `b81f79a`)**: validates `evidence_status` against the
canonical Batch 1 set (`EVIDENCE_COMPLETE`/`CONDITIONAL`/`INSUFFICIENT`,
`niklar_state_contracts.py`) before any other branch, and raises
`ValueError` on anything outside it — the check fires before the
staleness/caveat logic even evaluates, so an unrecognized status can
never reach any eligible-producing path regardless of the other
inputs.

**Regression tests**: 4 new — unknown status
(`test_unknown_evidence_status_fails_closed`), empty string
(`test_empty_evidence_status_fails_closed`), a case-sensitive near-miss
of a real constant
(`test_lowercase_variant_of_a_known_status_fails_closed`), and unknown
status combined with execution-sensitive+stale inputs, proving
validation precedes staleness evaluation
(`test_unknown_evidence_status_fails_closed_even_when_execution_sensitive_and_stale`).
`tests/test_niklar_notification_service.py`: 24 -> 28 tests.

```diff
diff --git a/ops/niklar_notification_service.py b/ops/niklar_notification_service.py
index [prior]..[b81f79a] 100644
--- a/ops/niklar_notification_service.py
+++ b/ops/niklar_notification_service.py
@@ imports @@
-from .niklar_state_contracts import EVIDENCE_CONDITIONAL, EVIDENCE_INSUFFICIENT
+from .niklar_state_contracts import EVIDENCE_COMPLETE, EVIDENCE_CONDITIONAL, EVIDENCE_INSUFFICIENT
@@ eligibility gate section @@
 NOTIFICATION_ELIGIBLE = "ELIGIBLE"
 NOTIFICATION_ELIGIBLE_WITH_CAVEAT = "ELIGIBLE_WITH_CAVEAT"
 NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE = "BLOCKED_INSUFFICIENT_EVIDENCE"
 NOTIFICATION_BLOCKED_STALE_OBSERVATION = "BLOCKED_STALE_OBSERVATION"
+
+_KNOWN_EVIDENCE_STATUSES = frozenset({EVIDENCE_COMPLETE, EVIDENCE_CONDITIONAL, EVIDENCE_INSUFFICIENT})

 def decide_notification_eligibility(
     *, evidence_status, execution_sensitive,
     observation_age_seconds=None, max_observation_age_seconds=None,
 ) -> str:
+    if evidence_status not in _KNOWN_EVIDENCE_STATUSES:
+        raise ValueError(
+            f"decide_notification_eligibility: unrecognized evidence_status {evidence_status!r} -- "
+            f"must be one of {sorted(_KNOWN_EVIDENCE_STATUSES)}"
+        )
     if evidence_status == EVIDENCE_INSUFFICIENT:
         return NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE
     ...
```

Full commit: `https://github.com/joiedevivre02/niklar-stocks/commit/b81f79a`

## F3 — Unsupported discovery/tactical consumer-stage mapping (BLOCKING ARCHITECTURAL DRIFT) — REVERTED (Option B)

**Finding**: commit `05657ef` promoted `discovery_consumers_sync`/
`tactical_consumers_sync` to "real" orchestrator stage handlers via a
shared `make_consumer_sync_handler()` factory. That package itself
already disclosed MEDIUM confidence and "no specification defining how
discovery_consumers_sync differs from tactical_consumers_sync beyond
scope." The review found: Section 19 proves automatic/manual Slack
sends share one notification service; it does not by itself prove
these two named orchestrator stages ARE semantically equivalent to
Slack notification delivery. Per "Never invent missing behavior" and
"do not infer stage semantics from names alone," the mapping could not
be promoted as implemented without canonical evidence.

**Resolution chosen: Option B** (revert/declassify — Option A,
"produce exact canonical specification," is not available; no such
specification exists anywhere in this repo's synced canon).

**Fix (commit `5163e7b`)**: `make_consumer_sync_handler()` removed
entirely from `ops/niklar_orchestrator_stage_handlers.py`, along with
its now-unused `niklar_notification_service` import aliases and the
`Sequence` type import. Both stage keys revert to deferred
(caller-injected adapter), same as the other 4 genuinely-blocked
stages — **7 of 13 real (was 9), 6 deferred (was 4)**. The underlying
Section 19 core (`ops/niklar_notification_service.py`) is explicitly
retained per the review's own instruction and is unaffected structurally
(only F2's internal fix applies to it) — only the claim that it maps
onto these two specific orchestrator stage keys was withdrawn. Module
docstring's STAGE STATUS rewritten with a `CORRECTIVE NOTE` documenting
this reversion in place, for future readers.

**Test changes**: `ConsumerSyncHandlerTests` (9 tests) removed along
with the reverted factory. The end-to-end `run_pipeline()` integration
test reverts to stub adapters for both stage keys (renamed back to
`test_all_7_real_handlers_run_successfully_alongside_stub_adapters_for_the_rest`).
`tests/test_niklar_orchestrator_stage_handlers.py` module docstring
updated to document the reversion.

**Documentation updates** (same commit): `ops/README.md` and
`AGENTS.md` — every "9 of 13"/"4 deferred" stage-count reference
corrected to "7 of 13"/"6 deferred"; the discovery/tactical
consumer-sync bullet removed from the "real handlers" description and
added to "What's explicitly out of scope," both citing this finding.
While already in these files for the count corrections, also fixed
pre-existing staleness from the `31d7c95` (Section 12) commit that had
never touched them: added a Modules entry for
`niklar_backup_policy.py` (previously undocumented entirely), and
corrected the test count (211 -> 243) and mypy file count (17 -> 18)
to reflect this cumulative corrective batch.

Full commit: `https://github.com/joiedevivre02/niklar-stocks/commit/5163e7b`

## Non-blocking note (from the review) — already reconciled

`31d7c95`'s commit message states "37 new tests" while the actual
cumulative delta was 34; the prior handoff (`HANDOFF_SEQ 22`) and its
review package already disclosed and reconciled this. No further
action required — noted here only for completeness against the
review's own checklist.

## Cumulative validation (all 3 corrective commits applied, HEAD `5163e7b`)

- `unittest`: 243/243 (245 after `31d7c95` -> +3 net new from F1 -> +4
  net new from F2 -> -9 removed with F3's reverted test class = 243)
- `mypy`: 0 issues, 18 source files (unchanged from `31d7c95`)
- PWA smoke test: PASS
- Real GitHub Actions run **`31314674703`** (commit `5163e7b`, the
  final pushed SHA — GitHub Actions fires once per push, evaluating
  the tip commit, not once per individual commit in the push) polled
  directly via the GitHub API: conclusion **`"success"`**. Per the
  review's explicit instruction, this run is **not** relied on as a
  substitute for the individual commits' own correctness — each
  commit was validated locally (full suite + mypy + smoke) before
  being committed, and this CI run is the first real, post-correction
  confirmation, superseding the pre-correction runs on `577b60b`/
  `05657ef`/`31d7c95` which are now stale evidence.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: F1 and F2 both convert a fail-open path into
   an explicit fail-closed one (new outcome constant / `ValueError`
   respectively). F3 removes a function entirely rather than
   attempting to patch an unsupported claim — no partial/hedged
   "maybe-real" state was left behind.
2. **Unnecessary abstraction/dependency**: no new dependency in any of
   the 3 commits. F3 is a net simplification (141 lines removed from
   `ops/niklar_orchestrator_stage_handlers.py`, 165 from its test
   file) — the corrective direction reduced surface area rather than
   adding a workaround.
3. **Security/privacy**: none of the 3 commits touch I/O, credentials,
   or any new external surface.
4. **Canonical boundary drift**: no canonical trading/scoring/
   tactical/invalidation/Decision Object logic touched in any of the
   3 commits. F3 specifically corrects a canonical-boundary drift risk
   the review caught — an unsupported stage-semantics inference — by
   reverting it, not by inventing supporting canon.

## Scope discipline

Per the review's `AUTHORIZED CORRECTIVE SCOPE` item 7 ("Stop after
publishing that corrective checkpoint. No unrelated next increment is
authorized by this response."), no new feature work was started
alongside these 3 fixes. The only non-F1/F2/F3 changes in this
checkpoint are the doc corrections in commit `5163e7b` that F3's
stage-count change directly required (plus pre-existing, unrelated
staleness caught incidentally while already in those same files/
paragraphs — not a separate increment).
