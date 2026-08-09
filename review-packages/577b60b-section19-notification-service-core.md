# Review package: niklar-stocks commit 577b60b (Section 19 notification-service core)

Continues the owner-authorized "batch 3 onwards" work (decision #47)
into Section 19's shared deterministic notification service. Full new-
file content published in full so ChatGPT's GitHub connector -- which
cannot reach the private `niklar-stocks` repo directly -- can
independently inspect the exact code. Scanned for secrets and Niklar
canonical/proprietary content before publishing (clean).

## Why this is in scope now, not just Batch 4

Section 19 was previously scoped to Batch 4 (decision #31: "belongs
when the mobile delivery/PWA interaction + notification service layers
are implemented"). Its own text requires: "Automatic and manual Slack
sends must share the SAME deterministic notification service,
freshness/eligibility gates, canonical renderer, idempotency rules,
retry behavior, and Change Ledger delivery records." That shared
deterministic core needs neither Slack credentials nor a PWA to exist
first -- unlike the actual delivery/UI work, which genuinely does
belong to a later batch. The owner's "Continue working on batch 3
**onwards**" (decision #47) explicitly widens scope past Batch 3 alone;
this module is exactly the kind of later-batch-adjacent, fully
groundable, credential-free core that instruction was read to
authorize. The actual Slack HTTP delivery, PWA UI wiring, and
consolidated-brief ticker-eligibility selection remain separate,
explicitly out of scope here.

## Commit message (verbatim)

```
commit 577b60b...
Section 19 deterministic notification-service core

Continues the owner-authorized "batch 3 onwards" work (decision #47)
into Section 19's shared deterministic notification service --
previously scoped to Batch 4 (decision #31), but Section 19's own
text requires automatic and manual Slack sends to share the SAME
deterministic notification service/eligibility gates/idempotency
rules, and that shared core needs neither Slack credentials nor a
PWA to exist first. The owner's "onwards" instruction explicitly
widens scope past Batch 3 alone; the actual Slack HTTP delivery, PWA
UI wiring, and consolidated-brief ticker selection remain separate,
later work this module doesn't attempt.

New file ops/niklar_notification_service.py, four pieces, each a
direct implementation of Section 19's own text (not invented from the
section name alone):

- derive_notification_id(): deterministic id from exactly the stable
  fields Section 19 names (run/checkpoint/channel/scope/Decision
  Object version), via sha256 -- same determinism convention as
  niklar_stocks_ops_v4.schema_hash().
- check_duplicate_send(): the DUPLICATE/RESEND gate -- a prior
  successful send for the same notification_id requires an explicit
  resend action; a prior failed attempt does not block a fresh send.
- decide_notification_eligibility(): the STALE/CAVEAT GATE -- reuses
  Batch 1's EVIDENCE_COMPLETE/CONDITIONAL/INSUFFICIENT contract
  directly rather than inventing a parallel evidence-quality concept;
  execution_sensitive and staleness thresholds are caller-supplied,
  not guessed at (this repo has no synced definition of which
  conclusions count as execution-sensitive).
- classify_delivery_attempt(): the DELIVERY STATUS classification
  against a real retry budget (succeeded -> SENT; failed with
  retries remaining -> PENDING_RETRY; failed with none -> DELIVERY_
  FAILED).

Preview content deliberately NOT reimplemented -- Section 19's
"PREVIEW BEFORE SEND: show exactly what Slack will receive" is
already satisfied by the existing render_preopen/render_intraday/
render_near_close renderers; wrapping them here would duplicate them,
not implement anything new.

New file tests/test_niklar_notification_service.py, 24 tests covering
determinism/collision-avoidance for the id derivation, every gate
ordering case for both the duplicate-send and eligibility gates
(including execution-sensitive-stale-but-conditional-evidence
blocking rather than merely caveating), and the retry-budget
classification.

No I/O, no Slack SDK, no credentials, no dependency added. unittest
202/202, mypy 0/17, PWA smoke PASS. The 3 Drive-verbatim ops/ modules
remain untouched.
```

## Full new file: `ops/niklar_notification_service.py`

```python
"""Niklar deterministic notification-service core (PWA<->Slack, architecture
doc Section 19).

Continues the owner-authorized "batch 3 onwards" work (niklar-handoff
`DECISIONS_CURRENT.txt` #47: "Continue working on batch 3 onwards.
Handoffs can be given as you work. Chatgpt will take them for review
later") into the one piece of Section 19 that's genuinely groundable
without Slack credentials or PWA UI work -- Section 19 was previously
scoped to Batch 4 (decision #31: "belongs when the mobile delivery/PWA
interaction + notification service layers are implemented"), but its
own text requires "Automatic and manual Slack sends must share the
SAME deterministic notification service, freshness/eligibility gates,
canonical renderer, idempotency rules, retry behavior" -- that shared
deterministic core doesn't need Slack credentials or a PWA to exist
first, and the owner's "onwards" explicitly widens scope past Batch 3
alone. The actual Slack HTTP delivery, PWA UI wiring, and consolidated-
brief ticker-eligibility selection remain separate, later work --
this module is the shared core those will eventually call into, not
those things themselves.

Four pieces, each grounded in Section 19's own text or already-existing
repo contracts, no fabrication:

1. `derive_notification_id()` -- "derive a deterministic notification_id
   from run/checkpoint/channel/ticker-or-brief scope/Decision Object
   version (or equivalent stable fields)" (Section 19, DUPLICATE/RESEND)
   -- a direct, literal implementation of that sentence.
2. `check_duplicate_send()` -- "If the exact state was already sent
   successfully, show when it was sent and require an explicit resend
   action rather than silently duplicating it" -- same section, direct
   implementation.
3. `decide_notification_eligibility()` -- "manual send remains possible
   only under canonical safety rules. If an execution-sensitive
   observation is stale or required evidence is insufficient, warn or
   suppress" (Section 19, STALE/CAVEAT GATE) -- reuses Batch 1's
   `EVIDENCE_COMPLETE`/`CONDITIONAL`/`INSUFFICIENT` contract
   (`niklar_state_contracts.py`) rather than inventing a parallel
   evidence-quality concept; `execution_sensitive` and the staleness
   threshold are caller-supplied, not guessed at.
4. `classify_delivery_attempt()` -- "PWA should expose SENT /
   PENDING_RETRY / DELIVERY_FAILED" with "bounded retry/backoff"
   (Section 19, DELIVERY STATUS + Reliability requirements) -- a small,
   real retry-budget policy (succeeded -> SENT; failed with retries
   remaining -> PENDING_RETRY; failed with none remaining ->
   DELIVERY_FAILED), not a bare relabeling.

**Preview content is deliberately NOT reimplemented here.** Section
19's "PREVIEW BEFORE SEND: show exactly what Slack will receive" is
already satisfied by this repo's existing tactical-card renderers
(`niklar_stocks_ops_v4.render_preopen`/`render_intraday`/
`render_near_close`) -- the Slack message body and the PWA preview
should be the SAME canonical rendering, per Section 19's own "converge
on the SAME renderer and send path" requirement. Wrapping those
renderers here would duplicate them, not implement anything new.

Pure, deterministic, no I/O of its own -- same discipline as every
other `ops/` module. No Slack SDK, no HTTP, no credentials.
"""
from __future__ import annotations

import hashlib
from typing import Any, Mapping, Optional, Sequence

from .niklar_state_contracts import EVIDENCE_CONDITIONAL, EVIDENCE_INSUFFICIENT

# --- notification_id derivation (Section 19, DUPLICATE/RESEND) -----------


def derive_notification_id(
    *,
    run_id: str,
    checkpoint: str,
    channel: str,
    scope: str,
    decision_object_version: str,
) -> str:
    """Deterministic `notification_id` from exactly the stable fields
    Section 19 names: run/checkpoint/channel/ticker-or-brief scope/
    Decision Object version. `scope` is a single ticker symbol for a
    SINGLE TICKER send or a stable brief-scope label (e.g. `"BRIEF"`,
    caller-defined) for an ALL/CURRENT BRIEF send -- this function
    doesn't distinguish the two, it just hashes whatever stable scope
    string it's given. Same inputs always produce the same id
    (`hashlib.sha256`, same determinism convention as
    `niklar_stocks_ops_v4.schema_hash()`); any field changing (a new
    Decision Object version, a different channel) produces a different
    id, which is exactly the duplicate-detection property Section 19
    needs. Fails closed (`ValueError`) if any field is empty -- an
    unstable/missing field would make the id meaningless.
    """
    fields = {
        "run_id": run_id,
        "checkpoint": checkpoint,
        "channel": channel,
        "scope": scope,
        "decision_object_version": decision_object_version,
    }
    empty = [name for name, value in fields.items() if not value]
    if empty:
        raise ValueError(f"derive_notification_id: field(s) must be non-empty: {', '.join(empty)}")
    stable = "|".join([run_id, checkpoint, channel, scope, decision_object_version])
    return hashlib.sha256(stable.encode("utf-8")).hexdigest()


# --- duplicate/resend gate (Section 19, DUPLICATE/RESEND) ----------------

NOTIFICATION_SEND_NEW = "NEW_SEND"
NOTIFICATION_SEND_ALREADY_SENT_REQUIRES_EXPLICIT_RESEND = "ALREADY_SENT_REQUIRES_EXPLICIT_RESEND"


def check_duplicate_send(
    *,
    notification_id: str,
    prior_sends: Sequence[Mapping[str, Any]],
    explicit_resend: bool = False,
) -> dict:
    """"If the exact state was already sent successfully, show when it
    was sent and require an explicit resend action rather than
    silently duplicating it" (Section 19). `prior_sends` is a
    caller-supplied record of past attempts -- this function does no
    lookup/storage of its own, same accept-already-resolved-data
    discipline as every other `ops/` function. Only entries with
    `status == "SENT"` count as a prior successful send; a prior
    `DELIVERY_FAILED`/`PENDING_RETRY` attempt for the same
    `notification_id` does not block a fresh attempt.

    Returns `{"decision": NOTIFICATION_SEND_NEW | NOTIFICATION_SEND_
    ALREADY_SENT_REQUIRES_EXPLICIT_RESEND, "prior_sent_at": <timestamp
    of the most recent matching successful send, or None>}`.
    `explicit_resend=True` (the caller's explicit resend action, per
    Section 19) allows a fresh `NEW_SEND` decision even with a matching
    prior send, while still reporting `prior_sent_at` so the caller can
    show when it was sent, as Section 19 requires.
    """
    matching = tuple(s for s in prior_sends if s.get("notification_id") == notification_id and s.get("status") == "SENT")
    prior_sent_at = matching[-1].get("sent_at") if matching else None
    if not matching or explicit_resend:
        return {"decision": NOTIFICATION_SEND_NEW, "prior_sent_at": prior_sent_at}
    return {"decision": NOTIFICATION_SEND_ALREADY_SENT_REQUIRES_EXPLICIT_RESEND, "prior_sent_at": prior_sent_at}


# --- stale/caveat eligibility gate (Section 19, STALE/CAVEAT GATE) -------

NOTIFICATION_ELIGIBLE = "ELIGIBLE"
NOTIFICATION_ELIGIBLE_WITH_CAVEAT = "ELIGIBLE_WITH_CAVEAT"
NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE = "BLOCKED_INSUFFICIENT_EVIDENCE"
NOTIFICATION_BLOCKED_STALE_OBSERVATION = "BLOCKED_STALE_OBSERVATION"


def decide_notification_eligibility(
    *,
    evidence_status: str,
    execution_sensitive: bool,
    observation_age_seconds: Optional[float] = None,
    max_observation_age_seconds: Optional[float] = None,
) -> str:
    """"Manual send remains possible only under canonical safety rules.
    If an execution-sensitive observation is stale or required evidence
    is insufficient, warn or suppress the affected conclusion rather
    than presenting it as current. EVIDENCE_CONDITIONAL caveats must
    travel with the card" (Section 19, STALE/CAVEAT GATE).

    `evidence_status` is one of Batch 1's `EVIDENCE_COMPLETE`/
    `EVIDENCE_CONDITIONAL`/`EVIDENCE_INSUFFICIENT`
    (`niklar_state_contracts.py`) -- reused directly, not a parallel
    concept. `execution_sensitive` (is this conclusion the kind Section
    19 means by "execution-sensitive," e.g. an entry/breakout trigger
    vs. a general note) and the staleness inputs are caller-supplied,
    not guessed at here -- this repo has no synced definition of which
    conclusions count as execution-sensitive.

    Order: `EVIDENCE_INSUFFICIENT` blocks outright regardless of
    staleness (insufficient evidence is the stronger reason). Then, for
    execution-sensitive conclusions with both age inputs supplied, an
    observation older than the threshold blocks. Otherwise
    `EVIDENCE_CONDITIONAL` is eligible-with-caveat (the caveat "must
    travel with the card" per Section 19 -- this function only signals
    that requirement via its return value, it doesn't attach the caveat
    text itself). `EVIDENCE_COMPLETE` and fresh/non-execution-sensitive
    otherwise: plain eligible.
    """
    if evidence_status == EVIDENCE_INSUFFICIENT:
        return NOTIFICATION_BLOCKED_INSUFFICIENT_EVIDENCE
    if execution_sensitive and observation_age_seconds is not None and max_observation_age_seconds is not None:
        if observation_age_seconds > max_observation_age_seconds:
            return NOTIFICATION_BLOCKED_STALE_OBSERVATION
    if evidence_status == EVIDENCE_CONDITIONAL:
        return NOTIFICATION_ELIGIBLE_WITH_CAVEAT
    return NOTIFICATION_ELIGIBLE


# --- delivery-status classification (Section 19, DELIVERY STATUS) --------

DELIVERY_STATUS_SENT = "SENT"
DELIVERY_STATUS_PENDING_RETRY = "PENDING_RETRY"
DELIVERY_STATUS_DELIVERY_FAILED = "DELIVERY_FAILED"


def classify_delivery_attempt(*, succeeded: bool, retries_remaining: int) -> str:
    """"PWA should expose SENT / PENDING_RETRY / DELIVERY_FAILED" with
    "bounded retry/backoff" (Section 19). `succeeded`/`retries_remaining`
    are supplied by the caller after an actual send attempt (the real
    Slack HTTP call itself is credentialed I/O this module doesn't do)
    -- this function only classifies the outcome against the retry
    budget, it does not perform or schedule the retry itself.
    """
    if succeeded:
        return DELIVERY_STATUS_SENT
    if retries_remaining > 0:
        return DELIVERY_STATUS_PENDING_RETRY
    return DELIVERY_STATUS_DELIVERY_FAILED
```

## Self-audit against the standing 4 criteria

1. **Failure envelope**: `derive_notification_id()` fails closed on any
   empty stable field; the other three functions are pure
   classification with no exception paths needed (they classify
   already-resolved inputs, they don't fetch/parse anything that could
   fail).
2. **Unnecessary abstraction/dependency**: no new dependency (stdlib
   `hashlib` only). No renderer wrapper -- explicitly declined, reusing
   the existing `render_*` functions directly per the module docstring.
3. **Security/privacy**: no I/O, no Slack SDK, no HTTP, no credentials
   anywhere in this module.
4. **Canonical boundary drift**: none of the 3 Drive-verbatim `ops/`
   modules were touched. No proprietary Slack-channel-mapping or
   ticker-eligibility logic was fabricated -- both remain explicitly
   out of scope, left for later batches with real specification.

## Validation

- `unittest`: 202/202 (178 -> 202, 24 new)
- `mypy`: 0 issues, 17 source files (16 -> 17)
- PWA smoke test: unaffected, still PASS
- Real GitHub Actions run: polled directly via the GitHub API after
  push (see `CURRENT_HANDOFF.txt` for the run ID/conclusion).
