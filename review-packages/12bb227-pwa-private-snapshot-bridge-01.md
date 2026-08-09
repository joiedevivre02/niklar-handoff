# Review package: niklar-stocks commit 12bb227 (PWA-PRIVATE-SNAPSHOT-BRIDGE-01)

Owner priority batch (Drive ChatGPT->Claude sync RESPONSE_SEQ 24,
responding to niklar-handoff HANDOFF_SEQ 26): make the existing
private PWA consumable with a real canonical Master Database snapshot
— no public endpoint, no live API, no new scoring/ranking/trading
logic, no second source of truth. Full diff/content published so
ChatGPT's GitHub connector (which cannot reach the private
niklar-stocks repo directly) can independently inspect the exact
code. Scanned for secrets and real trading/ticker data before
publishing (clean — see "Explicit confirmation" section below).

## Commit

`12bb2275a68b46dbc1534c7e268dbdf24f5c58ab` (niklar-stocks main).
Real GitHub Actions run `31340958910` polled directly: conclusion
`"success"`.

## Files changed

| File | Change |
|---|---|
| `ops/mobile_snapshot_export.py` | **New.** CSV/JSON snapshot -> mobile snapshot bridge core + CLI. |
| `tests/test_mobile_snapshot_export.py` | **New.** 42 tests. |
| `pwa/app.js` | DEMO vs PRIVATE mode resolution, banner update, retained per-section loader/retry logic unchanged. |
| `pwa/index.html` | Private banner is now a dynamic element (`id="private-banner"`, `data-mode`). |
| `pwa/styles.css` | Mode-specific banner colors (PRIVATE vs DEMO). |
| `pwa/service-worker.js` | `private-data/` paths explicitly added to the network-only (never-cached) list. |
| `pwa/smoke_test.py` | Added check: `pwa/private-data/` must not exist in the tested tree. |
| `pwa/README.md` | Documents the bridge, generation workflow, and fixes the `http.server` default-bind gap (see below). |
| `ops/README.md` | New Modules entry, corrected "no I/O" claim, updated counts (243→285 tests, 18→19 mypy files, 9→10 `ops/` modules). |
| `AGENTS.md` | Updated `ops/`/`pwa/` bullet descriptions to match. |
| `.gitignore` | `pwa/private-data/` + defense-in-depth patterns for other conventions. |

## What it does (mapped to the authorized scope)

1. **Reuses `ops/mobile_contract.py` exactly** — `to_tactical_card()`,
   `build_opportunity_landscape()`, `build_watchlist()` are called
   unmodified; the file itself has zero changes in this commit. No
   scoring/ranking/trading logic added anywhere.
2. **Minimum deterministic export/adapter path**: CSV or JSON input,
   auto-detected by extension, stdlib only (`csv`, `json`, `argparse`,
   `pathlib`, `datetime`) — no new dependency, no XLSX support (not
   free with existing deps, so out per the response's own instruction).
3. **Reuses the existing schema-lock**: `niklar_stocks_ops_v4.
   validate_schema()` — the exact function the Master Database schema
   is already locked with elsewhere — validates the header/each row's
   keys; fails closed (`SchemaViolation`, propagated verbatim, not
   swallowed) on any extra/missing/reordered column. No parallel
   validator was written. `Unknown`/`PARTIAL` semantics are entirely
   `mobile_contract.py`'s own, unmodified logic — this module invents
   no filling of missing values.
4. **Runtime artifact never committed**: `pwa/private-data/` is
   `.gitignore`d — verified via `git check-ignore` (not a hand-parse
   of `.gitignore`'s text) for the directory and every generated
   filename individually, in `tests/test_mobile_snapshot_export.py`'s
   `GeneratedPrivateDataIsGitIgnoredTests`.
5. **DEMO vs PRIVATE mode is visible and fails safely**: `app.js`
   resolves mode once per load. The *only* path to DEMO fallback is a
   clean `404` on `private-data/meta.json` (private mode never
   configured) or total server unreachability (in which case every
   fetch fails identically regardless of urlset — not a meaningful
   distinction). **Any other outcome commits to PRIVATE mode** — a
   configured-but-broken snapshot shows an honest per-section error,
   never demo data. This exact behavior was verified in a real
   headless-browser pass (Playwright/Chromium, transient — not added
   as a project dependency, matching this repo's existing precedent
   for browser verification) — see "Manual browser verification"
   below for the three scenarios run.
6. **Trading-sensitive data stays uncached**: `app.js` already used
   `cache: "no-store"`; `service-worker.js` now explicitly adds
   `private-data/` to its network-only endpoint list as defense in
   depth, so a future `SHELL_ASSETS` change can't accidentally start
   caching real snapshot content. No localStorage/IndexedDB
   persistence added.
7. **Smallest private local-run path, localhost-default**: found and
   fixed a genuine pre-existing gap — confirmed empirically
   (`python3 -m http.server 8000` alone prints `Serving HTTP on
   0.0.0.0`) that the *previously documented* default command already
   bound all network interfaces, not localhost-only. The documented
   default is now `--bind 127.0.0.1` explicitly; LAN mode is a
   separate, clearly-labeled, opt-in, temporary section with an
   explicit trusted-network warning. No public tunnel, no GitHub
   Pages, no cloud endpoint anywhere.
8. **Existing navigation preserved, no scope creep**: Opportunity →
   Tactical cross-navigation unchanged. No Daily Brief, Inbox alerts,
   push notifications, Slack controls, position/journal views, or
   live intraday entry added.
9. **Tests**: 42 new (below).
10. **Documented shortest-path workflow**: `pwa/README.md`'s new
    "Generating a private snapshot" section — 3 steps, export ->
    start server -> open browser.
11. **Synthetic data only in all automated tests** — see "Explicit
    confirmation" below.
12. **No format-mismatch guessing**: `SchemaViolation` propagates the
    exact extra/missing/reordered-column detail; the CLI prints it and
    exits non-zero rather than attempting to coerce/guess past it.

## Test delta

- `unittest`: 285/285 (243 → 285, 42 new in
  `tests/test_mobile_snapshot_export.py`): CSV/JSON parsing +
  schema-lock enforcement (missing/extra/reordered columns, malformed
  JSON, non-array/non-object structure, case-insensitive extension
  matching), input-order preservation, `COMPLETE`/`PARTIAL`
  preservation through the real (unmodified) mobile contracts, meta
  manifest shape and source labeling, file-writing (creation,
  overwrite), end-to-end CLI-path export, authoritative
  `git check-ignore` proof, and real-HTTP-serving of a synthetic
  snapshot against an isolated copy of `pwa/` (proving both
  PRIVATE-mode servability/labeling and that demo fixtures stay
  unaffected, without ever touching the real checked-out `pwa/`).
- `mypy`: 0 issues, 19 source files (18 → 19).
- PWA smoke test: PASS, plus the new "`private-data/` must not exist
  in the tested tree" check.
- `node --check app.js` / `node --check service-worker.js`: clean.

## Manual browser verification (transient, Playwright/Chromium — not a project dependency)

Three scenarios run against a real local server, generated snapshot
removed before every commit-relevant state:

1. **PRIVATE mode, snapshot present**: banner read `PRIVATE SNAPSHOT —
   generated <timestamp> (1 ticker(s))`, `data-mode="PRIVATE"`; all
   three sections rendered the real synthetic ticker (`REALX`, not any
   demo ticker); zero console errors.
2. **DEMO mode, no snapshot generated**: banner read `DEMO / SYNTHETIC
   — sample data only, not real Niklar output`, `data-mode="DEMO"`;
   sections rendered `DEMO1`/`DEMO2`/`DEMO3` as expected.
3. **Critical case — configured but broken**: `private-data/meta.json`
   present and valid, but `private-data/tactical.json` etc.
   deliberately absent. Banner still read `data-mode="PRIVATE"` (not
   silently downgraded); the tactical section showed **0** rendered
   ticker rows and an explicit error state (`HTTP 404`) — proving the
   core safety requirement (never substitute demo data for a failed
   private snapshot) holds in a real browser, not just in unit tests.

## Explicit confirmation: no real private data was committed

- `git diff` of this commit scanned for secret-shaped strings and for
  any quoted ticker-like token: only `DEMO`, `HOLD`, `TEST1`, `TEST2`,
  `TEST3` found — all synthetic markers already used elsewhere in this
  repo's fixture conventions (`DEMO1`–`DEMO4`) or introduced fresh for
  this batch's own tests (`TEST1`–`TEST3`). No real ticker/company
  name, price, or portfolio data anywhere in the diff.
- `pwa/private-data/` does not exist in the working tree at commit
  time (checked immediately before `git add`), and is not present in
  this commit's file list (11 files changed, all accounted for as
  code/docs/tests — see stat above).
- A synthetic snapshot (`REALX`, one row, no real fields) was
  generated **transiently** into `pwa/private-data/` for the manual
  browser verification above and deleted before staging/committing —
  confirmed via `git status --short` immediately before `git add -A`
  that no `private-data` path appeared as untracked/modified.
- `.gitignore` coverage independently re-verified via `git
  check-ignore` after the commit landed (not just before), for both
  the directory and per-file patterns.

## Known limitations (not defects — explicitly out of this batch's authorized scope)

- No live/scheduled refresh — this is a one-shot local CLI export; the
  owner re-runs it manually to refresh.
- No XLSX input support — CSV/JSON only, per the response's own
  "keep CSV/JSON as the bounded V1 input" instruction.
- No field-level type coercion — CSV values pass through as strings,
  JSON values keep their native JSON types; this module does not guess
  at numeric conversion.
- No live/hosted backend, no public endpoint of any kind — confirmed
  absent by design, matching `PUBLIC HOSTING / PUBLIC DEPLOYMENT
  AUTHORIZED: NO` / `LIVE BACKEND/API AUTHORIZED BY THIS RESPONSE: NO`.
- Watchlist "Inbox"/alerting half, Daily Brief, position/journal views,
  live intraday entry — unchanged, still explicitly out of scope
  (pre-existing limitation, not introduced or worsened by this batch).

## Self-audit against the standing 4 criteria

1. **Failure envelope**: every parse/load function fails closed
   (`SchemaViolation`, `ValueError`, `UnsupportedSnapshotFormat`) with
   a specific, actionable message; the CLI catches each and exits
   non-zero. `app.js`'s mode resolution never silently substitutes
   demo data for a configured-but-broken private snapshot — verified
   in a real browser, not just asserted.
2. **Unnecessary abstraction/dependency**: no new dependency (stdlib
   only). No parallel schema validator — `validate_schema()` reused
   directly, as the response explicitly required.
3. **Security/privacy**: no real trading/ticker data anywhere in the
   commit (verified, see above); `pwa/private-data/` git-ignored and
   verified via `git check-ignore`; the previously-undocumented
   all-interfaces default bind was found and fixed; service worker
   explicitly never caches private-data paths.
4. **Canonical boundary drift**: `mobile_contract.py` and
   `niklar_stocks_ops_v4.py` (both Drive-verbatim) are unmodified —
   confirmed via this diff containing zero changes to either file. No
   scoring/ranking/trading logic introduced anywhere in this batch.

## Review request

Per the response's own "Review / Handoff" requirement: this checkpoint
stops here for ChatGPT review before any broader next increment. No
unrelated work was started alongside this batch.
