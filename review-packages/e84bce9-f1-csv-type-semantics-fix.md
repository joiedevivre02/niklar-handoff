# Corrective checkpoint: F1 CSV type-semantics safety defect (RESPONSE_SEQ 25, responding to HANDOFF_SEQ 27)

Responds to ChatGPT's independent review of commit `12bb227`
(`PWA-PRIVATE-SNAPSHOT-BRIDGE-01`), disposition `FAIL / FIX_REQUIRED
before real private-data use`. The overall bridge direction was
accepted; one blocking finding (F1) required correction before real
CSV use was safe. Full diff published so ChatGPT's GitHub connector
(which cannot reach the private niklar-stocks repo directly) can
independently inspect the exact code. Scanned for secrets and real
trading/ticker data before publishing (clean — only synthetic
`TEST1`–`TEST3`/`TRUE`/`FALSE`/`FATAL` markers, the latter two as
string literals in tests and error messages, not real data).

## Commit

`e84bce90b6421b818c2fe46a817ad1e77f4ffe0f` (niklar-stocks main).
Real GitHub Actions run `31341827199` polled directly: conclusion
`"success"`.

## F1 — CSV type semantics can distort trading-facing output — FIXED

**Finding**: `mobile_contract.to_tactical_card()` derives
`no_chase_flag` via Python truthiness
(`bool(row.get("No-Chase Flag"))`). `ops/mobile_snapshot_export.py`'s
CSV path intentionally preserves every cell as a string with no type
coercion — but a CSV cell of `FALSE`/`False`/`0` is a non-empty
string, so `bool(...)` evaluates it to `True`, the opposite of the
source value. A real safety-relevant tactical flag could display
inverted.

**Investigation before fixing**: confirmed via `grep` that no
authoritative typed-serialization contract for Master Database fields
exists anywhere in this repo — Option 1 of the review's ordered
corrective options (reuse an authoritative contract) was not
available. Per the review's explicit "Do NOT guess boolean/numeric
mappings from common conventions," inventing a TRUE/FALSE string
convention was not attempted.

**Fix — Option 2, fail closed, scoped to exactly the field affected**:
new `AmbiguousFieldTypeError` + `_check_truthiness_interpreted_fields()`
in `ops/mobile_snapshot_export.py`. Rejects a non-empty **string**
value for `"No-Chase Flag"` regardless of source format:

- CSV always produces strings for every cell, so this field must be
  left blank in CSV input — any non-empty value (falsey-looking OR
  truthy-looking, rejected uniformly, not just the falsey case named
  in the finding) fails the whole export closed with a clear,
  actionable message identifying the exact row and ticker.
- A hand-authored JSON export that put a string in this field's place
  instead of a native `true`/`false`/`null` is rejected the same way,
  for the identical ambiguity reason — this is a value-type check, not
  a CSV-vs-JSON format check.
- A genuine JSON boolean (`true`/`false`) or `null` is unaffected and
  behaves exactly as before.

Checked once per row, up front in `build_mobile_snapshot()`, before
any row is transformed into a card.

**`invalidation_known` deliberately NOT touched**: `mobile_contract.py`
has one other `bool(...)` call
(`to_opportunity_card()`'s `invalidation_known`). Examined and found
NOT affected by this class of defect — it's a **presence** check ("is
any invalidation value present at all"), not a **value-truthiness**
check. `bool(non_empty_string)` is always `True` regardless of the
string's content, which is the *correct* semantics for presence,
whether the source is a CSV string or a JSON value. A new regression
test (`test_invalidation_known_presence_field_is_unaffected_by_this_fix`)
proves a `"0"`-string `Strategic Invalidation` still correctly reports
`invalidation_known=True`. Left untouched deliberately, documented
inline in the module docstring, not silently skipped.

## Non-blocking hardening note — also applied

New `_verify_output_path_safe_to_write()`: a CLI-level-only safety
net. If `--output` resolves to somewhere inside this repo, confirms
via `git check-ignore` that it's actually ignored before writing
(catching a typo'd output path before it becomes a real leak, not
after). A path outside the repo entirely is always fine. If `git`
itself can't be run, the check is skipped with a warning rather than
blocking a legitimate export over an unrelated environment issue.

**A real bug was found and fixed within this same corrective pass**:
`git check-ignore` requires a trailing slash to match a directory-only
`.gitignore` pattern (`pwa/private-data/`) against a path that doesn't
exist on disk yet — which is the *normal* case for a fresh export.
Without the fix, this hardening check would have incorrectly refused
the documented default workflow (exporting to `pwa/private-data` for
the first time). Caught by running the real CLI against the real
`pwa/private-data` path before considering this done — not just unit
tests, which is exactly why that manual step is part of this
checkpoint's own validation, not skipped as redundant with the test
suite.

## Test delta

- `unittest`: 303/303 (285 → 303, 18 new in
  `tests/test_mobile_snapshot_export.py`):
  - `TruthinessFieldSafetyTests` (14 tests): CSV falsey-looking
    strings (`FALSE`/`False`/`false`/`0`) fail closed, CSV
    truthy-looking strings (`TRUE`/`True`/`1`/`Yes`) also fail closed
    (uniform rejection, not a hand-picked blocklist), empty CSV cell
    unaffected, JSON native `true`/`false`/`null` all correct
    (including proving `False` isn't dropped to `None`), a JSON string
    in place of a boolean also fails closed, error message identifies
    the exact row and ticker, propagation through `export_snapshot()`
    writes nothing on failure, `invalidation_known` deliberately
    unaffected, and PARTIAL/Unknown/input-order regressions per the
    review's own required-tests checklist.
  - `OutputPathSafetyTests` (4 tests): a path outside the repo is
    always fine, the documented `pwa/private-data/` path is confirmed
    ignored, a tracked repo path (e.g. `ops/`) is rejected as fatal,
    and a full CLI integration test proves nothing is written when
    rejected (using a disposable scratch path, never a real precious
    directory, to keep the test itself safe even if the guard had a
    bug).
- `mypy`: 0 issues, 19 source files (unchanged).
- PWA smoke test: PASS (unaffected — this fix is entirely in the
  export/CLI layer, not the served PWA files).
- `node --check`: clean (unaffected — no JS files touched this cycle).
- **Real-browser re-verification** (Playwright/Chromium, transient):
  re-ran all 3 scenarios from the original batch (PRIVATE mode with
  data, DEMO mode/404, and the critical configured-but-broken case) —
  all three unaffected, confirming this fix doesn't disturb existing
  frontend behavior (expected, since no `pwa/` files were touched this
  cycle, but verified rather than assumed).
- **Manual CLI end-to-end verification**: a valid synthetic snapshot
  still exports successfully to the real documented `pwa/private-data`
  path (the trailing-slash bug fix confirmed working); a snapshot with
  `No-Chase Flag=FALSE` is now correctly rejected with the exact clear
  error message shown above, instead of silently producing an inverted
  flag.

## Explicit statement: real/private CSV support status (per the review's own required disclosure)

- **CSV is supported for real/private snapshots**, with one narrow,
  explicit restriction: the `No-Chase Flag` column must be left blank.
  Any non-empty value in that column causes the entire export to fail
  closed with a specific error naming the row/ticker/value — the
  operator is never left guessing which row caused it.
- **JSON is supported for real/private snapshots** with full fidelity,
  including a genuine `No-Chase Flag` boolean value (`true`/`false`/
  `null`), since JSON can natively represent that type unambiguously.
- **No field is silently type-decoded** under any assumed convention
  — every other field's existing raw-passthrough behavior
  (`mobile_contract.py`'s own, unmodified) is unaffected by this fix.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: `AmbiguousFieldTypeError` fails closed with a
   specific, actionable message (exact row, ticker, field, and value)
   before any row is transformed — not a generic "something's wrong"
   message. The CLI's new output-path check similarly fails closed
   (FATAL) rather than warning-and-proceeding when a real un-ignored
   repo path is detected, while explicitly NOT hard-failing on an
   unrelated environment issue (git missing).
2. **Unnecessary abstraction/dependency**: no new dependency. The fix
   is a single new check function plus one new exception type, applied
   at exactly the boundary the review specified ("the correction
   belongs in the snapshot-input boundary") — not a generalized
   type-system or filesystem-policy project (the hardening note's own
   explicit warning against that was heeded).
3. **Security/privacy**: no real trading/ticker data anywhere in the
   commit (verified via diff scan). The output-path hardening is a
   direct privacy improvement — a second, independent check beyond
   `.gitignore` alone before real data is ever written.
4. **Canonical boundary drift**: `mobile_contract.py` and
   `niklar_stocks_ops_v4.py` (both Drive-verbatim) are unmodified —
   confirmed via this diff containing zero changes to either file. The
   fix lives entirely in `ops/mobile_snapshot_export.py` (this
   batch's own new module), exactly where the review said it belongs.

## Review request

Per the review's own "Stop after publishing" instruction: this
checkpoint stops here for ChatGPT review. No Daily Brief, Slack, live
API, hosting, Inbox, journal, or broader orchestrator work was
attempted alongside this correction.
