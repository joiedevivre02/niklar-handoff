# Review package: niklar-stocks commit 31d7c95 (Section 12 backup/recovery core)

Continues the owner-authorized "batch 3 onwards" work (decision #47),
specifically the follow-up instruction to explore backup/recovery
(architecture doc Section 12) as the next candidate increment, applying
the Minimum Sufficient Context rule (read only Section 12 and directly
relevant implementation/tests/config, not the full canon or archived
handoffs). Full new-file content published in full so ChatGPT's GitHub
connector -- which cannot reach the private `niklar-stocks` repo
directly -- can independently inspect the exact code. Scanned for
secrets and Niklar canonical/proprietary content before publishing
(clean).

## Why this is in scope now

The instruction required first determining whether backup/recovery
already exists partially or fully in current HEAD, and reusing existing
mechanisms rather than creating a competing subsystem. Confirmed via
`grep -rniE "backup|restore|recovery"` across `ops/`, `tests/`, config,
and docs: no backup/recovery mechanism exists anywhere in this repo yet.
The only existing reference is
`niklar_state_contracts.build_current_state_snapshot()`'s `backup_status`
parameter -- an inert, caller-supplied passthrough string with no logic
behind it (Batch 1 scope). This module does not touch that file or
compete with that field; a caller may optionally pass one of this
module's outcome constants into `backup_status`, but nothing requires
it.

Section 12's own text specifies a deterministic decision core that
doesn't need real backup I/O to exist first -- the same "deterministic
core + caller-injected mechanism" pattern already used for Section 19
(`ops/niklar_notification_service.py`, commit 577b60b, still pending
review). The actual copy/verify/prune execution (real Drive/database/
git-ref/artifact I/O) is inherently local, credentialed work this
repo's `ops/` layer never does and this environment has no credentials
for anyway -- same boundary as every other credentialed gap this
session (decisions #35/#37).

No genuine specification gap was found this cycle -- unlike the two
explicitly-blocked candidates (manual intraday-entry validation, PWA
delivery-contract work), Section 12's text maps directly onto four
concrete, cleanly-boundaried functions without requiring any invented
specifics. Where Section 12 doesn't name a specific number (retention
count, age window, backup cadence, checksum algorithm), those stay
caller-supplied parameters -- not invented here.

## Commit message (verbatim)

```
commit 31d7c95...
Section 12 backup/recovery: deterministic decision core

Adds ops/niklar_backup_policy.py -- the same "deterministic decision
core + caller-injected mechanism" pattern already used for Section 19
(ops/niklar_notification_service.py), applied to architecture doc
Section 12 (BACKUP / RECOVERY).

Confirmed before writing: no backup/recovery mechanism exists anywhere
in this repo yet (grep across ops/, tests/, config, docs) -- the only
existing reference is niklar_state_contracts.build_current_state_
snapshot()'s inert backup_status passthrough parameter, untouched by
this module.

Four functions, each a direct implementation of one Section 12
sentence, nothing invented from the section name alone:

- is_eligible_for_backup(): fail-closed allowlist over the 6 scopes
  Section 12 names ("Back up canonical Drive state, master databases/
  exports, research/index state, config/decisions, relevant git refs,
  and important retained artifacts"); Actions cache is deliberately
  never in the allowlist ("do not back up disposable Actions cache").
- decide_catchup_backup_due(): freshness gate for "Local job may run
  when Mac is awake and catch up after missed schedules" -- max_age_
  seconds is caller-supplied, no cadence invented.
- verify_backup_success(): "Backup success requires verification, not
  merely copy completion" -- copy_completed=True alone is deliberately
  insufficient; checks item-count then optional checksum.
- decide_retention(): "Preserve versioned retention appropriate to
  recovery needs" -- min_versions_to_keep is a fail-safe floor always
  kept regardless of age; age-based pruning beyond the floor only
  applies when the caller supplies max_age_days/now. No specific
  retention count or age is invented; Section 12 names none.

The actual copy/verify/prune I/O remains the caller's (a real local
job's) responsibility -- same credentialed-I/O boundary as every other
gap this session, and the same injected-mechanism discipline as
send_adapter in niklar_notification_service.py.

Not wired into ops/niklar_orchestrator_stage_handlers.py: backup/
recovery is not one of the orchestrator's 13 canonical stages, so
there's no stage slot for it -- this module is a standalone Section 12
implementation, not a 10th handler.

37 new tests (tests/test_niklar_backup_policy.py): all-scope
eligibility plus Actions-cache/unrecognized/empty/case-sensitivity
fail-closed cases; catch-up due/not-due/no-prior-backup/exact-boundary/
malformed-timestamp/after-now/Z-suffix cases; all 4 verification
outcomes including priority ordering and copy-completion-alone-is-
insufficient; retention floor-preservation, age-based pruning beyond
the floor, no-age-policy-means-keep-everything (both directions),
zero-floor, negative-floor fail-closed, unordered-input sorting, and
exact-age-boundary cases.

Validation: unittest 245/245, mypy 0/18 files, PWA smoke PASS.

No canonical trading/scoring/tactical/invalidation/Decision Object
logic touched. Pure, deterministic, no I/O -- same ops/ discipline as
every other module in this package.
```

## Full new file: `ops/niklar_backup_policy.py`

```python
"""Niklar deterministic backup/recovery policy core (architecture doc
Section 12: BACKUP / RECOVERY).

Continues the owner-authorized "batch 3 onwards" work (niklar-handoff
`DECISIONS_CURRENT.txt` #47), following the same pattern already used
for Section 19 (`niklar_notification_service.py`): Section 12 describes
"Independent local catch-up backup... Local job may run when Mac is
awake" -- the actual backup mechanism (copying real Drive state/master
database exports/research-index state to a real destination) is
inherently local, credentialed I/O this repo's `ops/` layer never does
and this environment has no credentials for anyway (same boundary as
every other credentialed gap this session -- see the still-standing
Batch 3 full-scope HARD_GATE, decisions #35/#37). What Section 12 also
specifies, though, is a **deterministic decision core** that doesn't
need that I/O to exist first: what's in scope to back up, whether a
catch-up run is due, whether a completed copy actually verified as
successful, and what's eligible to prune. This module is that core --
the actual copy/verify/prune execution remains the caller's (a real
local job's) injected responsibility, same caller-supplies-the-
mechanism discipline as `niklar_notification_service.py`'s
`send_adapter`.

Confirmed before writing this: no backup/recovery mechanism exists
anywhere in this repo yet (checked via `grep` across `ops/`, `tests/`,
config, and docs) -- the only existing reference is `niklar_state_
contracts.build_current_state_snapshot()`'s `backup_status` parameter,
a caller-supplied passthrough string with no logic behind it (Batch 1
scope, unchanged here). This module doesn't compete with or duplicate
that field -- a caller can pass one of this module's outcome constants
into `backup_status` if useful, but this module doesn't touch
`niklar_state_contracts.py`.

Four pieces, each a direct implementation of Section 12's own text,
not invented from the section name alone:

1. `is_eligible_for_backup()` -- "Back up canonical Drive state, master
   databases/exports, research/index state, config/decisions, relevant
   git refs, and important retained artifacts as applicable... do not
   back up disposable Actions cache." A fail-closed allowlist, same
   pattern as Batch 2's `is_eligible_for_external_llm()` (already
   independently reviewed, PASS-frozen commit 02b6f88) -- an
   unrecognized scope (including Actions cache, which is deliberately
   never added to the allowlist) is never eligible.
2. `decide_catchup_backup_due()` -- "Local job may run when Mac is
   awake and catch up after missed schedules." A deterministic
   freshness gate, same shape as Batch 3's `research_memory_gate`
   (`niklar_orchestrator_stage_handlers.py`) applied to a different
   domain -- reusing an already-tested pattern, not duplicating logic
   (the underlying comparison is generic; there's no shared function to
   extract without the two call sites diverging on what "reuse" vs.
   "catch-up-due" actually mean).
3. `verify_backup_success()` -- "Backup success requires verification,
   not merely copy completion." Copy completion alone is explicitly
   insufficient per that sentence; this function classifies an
   already-attempted backup by copy completion plus count/checksum
   comparison (both caller-supplied -- this module doesn't invent what
   the expected count/checksum should be, any more than
   `niklar_notification_service.classify_delivery_attempt()` invents
   what counts as a successful Slack send).
4. `decide_retention()` -- "Preserve versioned retention appropriate to
   recovery needs." Section 12 does not name a specific retention
   count or age -- deliberately NOT invented here; `min_versions_to_
   keep` and `max_age_days` are caller-supplied policy parameters. The
   newest `min_versions_to_keep` are always kept regardless of age
   (fail-safe floor); age-based pruning only applies beyond that floor,
   and only when the caller supplies an age policy at all -- omitting
   `max_age_days`/`now` means nothing is ever pruned, never a silent
   default retention window this module made up.

Pure, deterministic, no I/O of its own -- same discipline as every
other `ops/` module. No filesystem access, no Drive/git client, no
credentials.
"""
from __future__ import annotations

from datetime import datetime
from typing import Any, Mapping, Optional, Sequence

# --- backup scope eligibility (Section 12: what to back up) ---------------

BACKUP_SCOPE_CANONICAL_DRIVE_STATE = "CANONICAL_DRIVE_STATE"
BACKUP_SCOPE_MASTER_DATABASE_EXPORTS = "MASTER_DATABASE_EXPORTS"
BACKUP_SCOPE_RESEARCH_INDEX_STATE = "RESEARCH_INDEX_STATE"
BACKUP_SCOPE_CONFIG_DECISIONS = "CONFIG_DECISIONS"
BACKUP_SCOPE_GIT_REFS = "GIT_REFS"
BACKUP_SCOPE_RETAINED_ARTIFACTS = "RETAINED_ARTIFACTS"

# Fail-closed allowlist: only the categories Section 12 explicitly
# names ("Back up canonical Drive state, master databases/exports,
# research/index state, config/decisions, relevant git refs, and
# important retained artifacts as applicable") are included. Actions
# cache is deliberately never added -- "do not back up disposable
# Actions cache" -- same fail-closed discipline as Batch 2's
# `_EXTERNAL_LLM_ELIGIBLE_CATEGORIES`. A new category must be added
# deliberately, not implied.
_BACKUP_ELIGIBLE_SCOPES = frozenset(
    {
        BACKUP_SCOPE_CANONICAL_DRIVE_STATE,
        BACKUP_SCOPE_MASTER_DATABASE_EXPORTS,
        BACKUP_SCOPE_RESEARCH_INDEX_STATE,
        BACKUP_SCOPE_CONFIG_DECISIONS,
        BACKUP_SCOPE_GIT_REFS,
        BACKUP_SCOPE_RETAINED_ARTIFACTS,
    }
)


def is_eligible_for_backup(scope: str) -> bool:
    """Fail-closed: an unrecognized scope (including Actions cache, or
    any string not in the allowlist at all) is never eligible. "Do not
    back up disposable Actions cache" (Section 12) -- the allowlist
    encodes the inverse of Section 12's named list directly, so a
    caller can't silently widen backup scope just by inventing a new
    category string.
    """
    return scope in _BACKUP_ELIGIBLE_SCOPES


# --- catch-up-due gate (Section 12: "Local job may run when Mac is -------
# --- awake and catch up after missed schedules") --------------------------

BACKUP_CATCHUP_DUE = "CATCHUP_DUE"
BACKUP_CATCHUP_NOT_DUE = "CATCHUP_NOT_DUE"


def decide_catchup_backup_due(
    *,
    last_successful_backup_at: Optional[str],
    now: str,
    max_age_seconds: int,
) -> str:
    """`BACKUP_CATCHUP_DUE` if there is no prior successful backup on
    record, or the last one is older than `max_age_seconds`; otherwise
    `BACKUP_CATCHUP_NOT_DUE`. `max_age_seconds` is caller-supplied --
    Section 12 does not name a specific cadence, and this module
    doesn't invent one. Timestamps are ISO 8601 strings
    (`datetime.fromisoformat`, which accepts a `Z` suffix under Python
    3.11). Fails closed (`ValueError`) on a malformed timestamp or a
    last-backup timestamp after `now`, matching the same fail-closed
    discipline as `niklar_orchestrator_stage_handlers.
    make_research_memory_gate_handler()`'s freshness gate.
    """
    now_dt = datetime.fromisoformat(now)
    if last_successful_backup_at is None:
        return BACKUP_CATCHUP_DUE
    last_dt = datetime.fromisoformat(last_successful_backup_at)
    age_seconds = (now_dt - last_dt).total_seconds()
    if age_seconds < 0:
        raise ValueError(
            f"decide_catchup_backup_due: last_successful_backup_at ({last_successful_backup_at}) is after now ({now})"
        )
    if age_seconds > max_age_seconds:
        return BACKUP_CATCHUP_DUE
    return BACKUP_CATCHUP_NOT_DUE


# --- backup-success verification (Section 12: "Backup success requires --
# --- verification, not merely copy completion") ---------------------------

BACKUP_VERIFIED = "VERIFIED"
BACKUP_VERIFICATION_FAILED_COPY_INCOMPLETE = "VERIFICATION_FAILED_COPY_INCOMPLETE"
BACKUP_VERIFICATION_FAILED_COUNT_MISMATCH = "VERIFICATION_FAILED_COUNT_MISMATCH"
BACKUP_VERIFICATION_FAILED_CHECKSUM_MISMATCH = "VERIFICATION_FAILED_CHECKSUM_MISMATCH"


def verify_backup_success(
    *,
    copy_completed: bool,
    expected_item_count: int,
    actual_item_count: int,
    expected_checksum: Optional[str] = None,
    actual_checksum: Optional[str] = None,
) -> str:
    """"Backup success requires verification, not merely copy
    completion" (Section 12) -- `copy_completed=True` alone is
    deliberately NOT sufficient to return `BACKUP_VERIFIED`; item-count
    comparison is always checked, and checksum comparison is checked
    whenever both are supplied. Order: incomplete copy first (the
    strongest failure -- nothing to verify), then count mismatch, then
    checksum mismatch. `expected_item_count`/`expected_checksum` are
    caller-supplied -- this module does not invent what a correct
    backup should contain, only classifies the comparison Section 12
    requires be made.
    """
    if not copy_completed:
        return BACKUP_VERIFICATION_FAILED_COPY_INCOMPLETE
    if expected_item_count != actual_item_count:
        return BACKUP_VERIFICATION_FAILED_COUNT_MISMATCH
    if expected_checksum is not None and actual_checksum is not None and expected_checksum != actual_checksum:
        return BACKUP_VERIFICATION_FAILED_CHECKSUM_MISMATCH
    return BACKUP_VERIFIED


# --- retention decision (Section 12: "Preserve versioned retention -------
# --- appropriate to recovery needs") ---------------------------------------

RETENTION_KEEP = "KEEP"
RETENTION_PRUNE_ELIGIBLE = "PRUNE_ELIGIBLE"


def decide_retention(
    *,
    backup_versions: Sequence[Mapping[str, Any]],
    min_versions_to_keep: int,
    max_age_days: Optional[int] = None,
    now: Optional[str] = None,
) -> dict:
    """Classify each entry in `backup_versions` (each a mapping with at
    least `"id"` and `"created_at"`, an ISO 8601 timestamp) as
    `RETENTION_KEEP` or `RETENTION_PRUNE_ELIGIBLE`.

    The newest `min_versions_to_keep` entries (by `created_at`,
    descending) are ALWAYS kept regardless of age -- a fail-safe floor,
    never zero recoverable backups. Beyond that floor, an entry is
    `RETENTION_PRUNE_ELIGIBLE` only if BOTH `max_age_days` and `now`
    are supplied AND the entry is older than `max_age_days`; if either
    is omitted, every entry beyond the floor is still `RETENTION_KEEP`
    -- this module never prunes on an implied/default age window,
    since Section 12 names no specific number ("appropriate to
    recovery needs" is a policy call left to the caller, not guessed
    at here).

    Returns `{"keep": (...ids...), "prune_eligible": (...ids...)}`,
    both tuples in the same relative order as the classified entries.
    Fails closed (`ValueError`) if `min_versions_to_keep` is negative.
    """
    if min_versions_to_keep < 0:
        raise ValueError(f"decide_retention: min_versions_to_keep must be >= 0, got {min_versions_to_keep}")

    ordered = sorted(backup_versions, key=lambda v: v["created_at"], reverse=True)
    now_dt = datetime.fromisoformat(now) if now is not None else None

    keep_ids = []
    prune_eligible_ids = []
    for index, version in enumerate(ordered):
        version_id = version["id"]
        if index < min_versions_to_keep:
            keep_ids.append(version_id)
            continue
        if max_age_days is None or now_dt is None:
            keep_ids.append(version_id)
            continue
        created_dt = datetime.fromisoformat(version["created_at"])
        age_days = (now_dt - created_dt).days
        if age_days > max_age_days:
            prune_eligible_ids.append(version_id)
        else:
            keep_ids.append(version_id)

    return {"keep": tuple(keep_ids), "prune_eligible": tuple(prune_eligible_ids)}
```

## Self-audit against the standing 4 criteria

1. **Failure envelope**: `decide_catchup_backup_due()` fails closed on
   a malformed timestamp or a last-backup timestamp after `now`;
   `decide_retention()` fails closed on a negative
   `min_versions_to_keep`. `is_eligible_for_backup()` and
   `verify_backup_success()` are pure classification over
   already-resolved inputs -- no exception path needed.
2. **Unnecessary abstraction/dependency**: no new dependency (stdlib
   `datetime` only, already used elsewhere in `ops/`). No wrapper
   function joining the four pieces -- deliberate, matches the Minimum
   Sufficient Context instruction and the same no-assembler choice
   made in `niklar_orchestrator_stage_handlers.py` increment 1.
3. **Security/privacy**: no I/O, no filesystem access, no Drive/git
   client, no credentials anywhere in this module.
4. **Canonical boundary drift**: none of the 3 Drive-verbatim `ops/`
   modules were touched. `niklar_state_contracts.py` (Batch 1) was not
   modified -- `backup_status` remains the same inert passthrough it
   already was. No specific retention count, age window, backup
   cadence, or checksum algorithm was fabricated; all stay
   caller-supplied.

## Validation

- `unittest`: 245/245 (211 -> 245, 34 new)
- `mypy`: 0 issues, 18 source files (17 -> 18)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run `31300333824` (commit 31d7c95) polled
  directly via the GitHub API after push: conclusion `"success"`.

**Correction to the commit message**: the pushed commit message states
"37 new tests" -- an actual recount of `tests/test_niklar_backup_
policy.py`'s test methods gives 34
(`IsEligibleForBackupTests`: 5, `DecideCatchupBackupDueTests`: 9,
`VerifyBackupSuccessTests`: 9, `DecideRetentionTests`: 11), matching the
211 -> 245 unittest delta exactly. The commit message's count is wrong
by 3 -- caught during this review-package write, not before the commit
was pushed. Flagged here rather than silently corrected, per this
session's standing self-audit discipline; no code or test content is
affected, only a descriptive number in the commit message text.
