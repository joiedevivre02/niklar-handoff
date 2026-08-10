# Corrective checkpoint: CSV/JSON UTF-8 BOM handling (owner-operability verification, HANDOFF_SEQ 30 cycle)

Owner "Cont": run the accepted private Niklar PWA with the real Master
Database. The owner supplied the real, canonical Master Database
export (CSV, 63 columns, 25 rows) directly to this session. Verifying
it against the already-accepted `PWA-PRIVATE-SNAPSHOT-BRIDGE-01`
pipeline (RESPONSE_SEQ 26 PASS) surfaced one real, narrow bug — fixed
in the same cycle, tested, and re-verified against the real file.
Full diff published so ChatGPT's GitHub connector (which cannot reach
the private niklar-stocks repo directly) can independently inspect the
exact code. Scanned for secrets and real trading/ticker data before
publishing (clean — only synthetic `TEST1`/`TEST2` markers; the real
owner data itself was never copied into any tracked or working-tree
path, never included in this package, and was removed from the
implementation environment immediately after use — see "Handling of
the real data" below).

## Commit

`9b2c78c6539863f09f117154fb8678e883b52455` (niklar-stocks main).
Real GitHub Actions run `31347338506` polled directly: conclusion
`"success"`.

## Finding — CSV/JSON files with a UTF-8 byte-order mark (BOM) failed schema validation — FIXED

**Finding**: `load_snapshot_rows()` read input files with plain
`"utf-8"`. A file saved as "UTF-8" by a spreadsheet tool (Excel,
Google Sheets — confirmed against the owner's actual export) commonly
carries a leading byte-order mark (BOM, bytes `EF BB BF`). Plain
`utf-8` decoding leaves that BOM as a literal `U+FEFF` character
prepended to the file's text, which corrupts the first column name
(`Ticker` becomes `﻿Ticker`) and fails `validate_schema()` for a
reason that has nothing to do with the actual data — a false-positive
rejection of a genuinely canon-compatible export.

**Why this doesn't conflict with the module's "never guess" discipline**
(the same discipline the prior F1 fix enforces for `No-Chase Flag`):
BOM handling is a well-defined, purely mechanical text-**decoding**
convention — "how are these bytes turned into text" — not a
business-**semantics** guess about what a value means. `AmbiguousFieldTypeError`
and its fail-closed behavior for genuinely ambiguous field values are
completely unaffected and unchanged.

**Fix**: `load_snapshot_rows()` now reads with `"utf-8-sig"` instead of
`"utf-8"`. This codec strips a leading BOM if present, and is exactly
equivalent to plain `utf-8` if one is absent — fully backward
compatible with every file this module already accepted; no other
behavior changes.

## Test delta

- `unittest`: 307/307 (303 → 307, 4 new in
  `tests/test_mobile_snapshot_export.py`, `BomHandlingTests`):
  a BOM-prefixed CSV parses correctly; a BOM-prefixed JSON parses
  correctly; a CSV without a BOM is unaffected (regression proof); a
  BOM-prefixed CSV survives `export_snapshot()` end-to-end. Each BOM
  test asserts the BOM bytes are actually present on disk first, so
  the test would fail before the fix rather than passing vacuously.
- `mypy`: 0 issues, 19 source files (unchanged).
- PWA smoke test: PASS (unaffected).

## Verification against the real owner data

Independently confirmed against the owner's actual Master Database
export (25 rows, real tickers, real strategic/tactical fields):

- Header now parses as `Ticker` (previously corrupted).
- Zero rows carry a non-empty `No-Chase Flag` — the CSV V1 restriction
  from the prior F1 fix / RESPONSE_SEQ 26's owner-use terms is
  satisfied by the real file as-is; no rejection needed.
- The documented export CLI (`python3 -m ops.mobile_snapshot_export
  --input <file> --output pwa/private-data`) produced all 4 output
  files, exit 0.
- `git check-ignore -v pwa/private-data/` confirmed the output path is
  protected.
- A real headless-browser render (Playwright/Chromium, transient)
  against the documented `127.0.0.1`-only local server confirmed:
  `data-mode="PRIVATE"`, banner text includes "PRIVATE SNAPSHOT" and
  the correct row count, all 25 real tickers render, zero DEMO-fixture
  leakage, zero console errors.

## Handling of the real data

The owner uploaded their real Master Database CSV directly to the
implementation session (outside the niklar-stocks repo entirely — it
was never copied into any repo-tracked path). It was read only via
the documented `--input` flag pointing at its original location. The
resulting private snapshot (`pwa/private-data/*.json`) was:

1. Confirmed `git check-ignore`d (defense already verified by the
   prior batch).
2. Delivered directly to the owner as a private file attachment (not
   published anywhere, not included in this package or any git
   history).
3. Removed from the implementation environment immediately after —
   `git status --short` confirmed empty (no generated artifacts, no
   modified files beyond the two source files this fix touches) before
   committing.

No real ticker symbol, price, or strategic/tactical field value from
the owner's data appears anywhere in this commit, this review package,
or niklar-handoff's git history — verified via an explicit scan of the
diff for the 25 real ticker symbols before publishing (clean).

## Self-audit against the standing 4 criteria

1. **Failure envelope**: unaffected for genuine mismatches — a file
   with actually-wrong columns, wrong order, or a hash mismatch still
   fails closed exactly as before (full regression suite green,
   including every pre-existing `SchemaViolation`/`AmbiguousFieldTypeError`
   test). The fix narrows *only* the BOM false-positive, nothing else.
2. **Unnecessary abstraction/dependency**: none. A one-parameter change
   (`"utf-8"` → `"utf-8-sig"`) in the single existing read call, no new
   dependency, no new code path.
3. **Security/privacy**: no security/privacy regression — the change
   is encoding-level only, doesn't touch what data is accepted/reads
   /writes. The real owner data itself was never committed or left in
   any tracked path (see above), consistent with every prior cycle.
4. **Canonical boundary drift**: `mobile_contract.py` and
   `niklar_stocks_ops_v4.py` (both Drive-verbatim) are unmodified —
   confirmed via this diff containing zero changes to either file. The
   fix lives entirely in `ops/mobile_snapshot_export.py`, the same
   module the prior F1 fix also lives in.

## Review request

Per the owner's "after a material checkpoint, publish the handoff and
stop for ChatGPT review" instruction: this checkpoint stops here for
ChatGPT review. No backend, hosting, Slack, live API, Inbox, journal,
or broader architecture work was attempted alongside this fix.
