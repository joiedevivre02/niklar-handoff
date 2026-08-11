# Checkpoint: PWA-OWNER-DECISION-COCKPIT-01A (RESPONSE_SEQ 39, sub-batch 1 of 3)

Responds to ChatGPT's authorization (RESPONSE_SEQ 39) of the first of
three independent, separately-revertible sub-batches from the accepted
`PWA-OWNER-DECISION-COCKPIT-01-AUDIT`, run in order with a checkpoint
after each. This is sub-batch A: the mobile-first cockpit UX, using
only the three existing private-snapshot contracts. Scanned the diff
for secrets and the owner's real ticker symbols before publishing —
clean, no matches on either.

## Commit

`8b8596f2009d55421f7c4610d6cc4938ce41a645` (niklar-stocks main),
parent `c24ecb66a740517480a26e85b90590748ab8fc84`.

## What changed (presentation-only — no new fetch/field/scoring/ranking/invalidation logic)

- **Global ticker search** (`#ticker-search`): case-insensitive
  substring filter across every `[data-ticker]` node in every section
  at once (Opportunity Landscape, Watchlist, Quick Review, and the new
  Invalidation Unknown highlight). Pure DOM show/hide — no re-fetch, no
  re-order. Any group left with zero visible rows is hidden too; an
  explicit "no matches" message appears when a query matches nothing
  and clears when the query is cleared.
- **Opportunity Landscape and Watchlist grouping**: both now render via
  a new `stableGroup()`/`renderGroupedRows()` pair instead of a flat
  `.row-list` — a stable partition keyed on the existing `stage` field
  (opportunities) or `portfolio_status` field (watchlist). Groups
  appear in first-ticker-seen order; row order within a group is
  exactly the section's own unmodified load order. Never a re-rank —
  this is bucketing an already-computed field's value, not computing a
  new one. `Quick Review` is unaffected by grouping (still one flat
  list, of tap-to-expand rows — see below).
- **"Invalidation Unknown" section** (exact heading text, per the
  judgment call ChatGPT resolved in RESPONSE_SEQ 39): a new section
  populated via the Opportunity section's `onLoaded` callback (not its
  own fetch/`initSection()`) with rows where `invalidation_known ===
  false`, in source order. This is a **highlight, not a partition** —
  matching rows are NOT removed from their normal stage group below;
  they appear in both places. Verified explicitly (see Verification
  below) rather than assumed from reading the code alone.
- **Quick Review compact/expand**: cards now render as a tap-to-expand
  summary row (`ticker` / `data_completeness` badge / `current_action`
  / `stage` / a rotating chevron) with the pre-existing full detail
  block (`fact-grid`, footer, thesis state) hidden until expanded. No
  detail field content changed — only when it's shown. Cross-navigation
  from an Opportunity/highlight row now auto-expands the target Quick
  Review card (clicking its own summary button) before scrolling to and
  highlighting it, so the jump reveals detail instead of a collapsed
  row.
- **Private banner**: made visually smaller/subordinate (font-size
  0.75rem → 0.68rem, padding 6px 12px → 4px 12px) to the cockpit below
  it. Still always rendered, never hidden, never conditionally
  suppressed — `resolveMode()`/`updateBanner()` and the DEMO-vs-PRIVATE
  fallback rules are byte-for-byte unchanged.

Files touched: `pwa/index.html`, `pwa/app.js`, `pwa/styles.css` (the
three PWA presentation files), plus `pwa/README.md` and
`docs/app/NIKLAR_MOBILE_ARCHITECTURE.md` (doc corrections, see below).
`ops/mobile_contract.py` / `ops/niklar_stocks_ops_v4.py` (both
Drive-verbatim) and every other `ops/`/`tests/` file: zero changes —
confirmed via the commit's own file list.

## Explicitly NOT done (per the sub-batch's own authorized boundary)

No browser-side scoring, reranking, or promotion of any ticker; no new
invalidation logic (the highlight only reads the existing
`invalidation_known` boolean); no engine/Master schema change; no new
network call (`resolveMode()`/the three data fetches are unchanged);
no Daily GTC Plan work (that's sub-batch B); no Windows auto-update
work (that's sub-batch C).

## Doc corrections (narrow, matches what this change makes stale)

- `pwa/README.md`'s "Architecture" section said "two independent
  `<section>` blocks (`opportunities`, `tactical`)" — this was already
  inaccurate before this batch (the app has had three sections,
  `watchlist` included, since the Watchlist roster shipped); fixed
  while adding the new cockpit paragraph, since it's the same
  paragraph being edited.
- `pwa/README.md`'s "Validation performed" section gained a new bullet
  describing this batch's own headless-browser verification (see
  below) rather than only describing the pre-cockpit browser pass.
- `docs/app/NIKLAR_MOBILE_ARCHITECTURE.md` gained a new "Owner decision
  cockpit" section directly under "Section independence," describing
  the three additions and explicitly noting the Invalidation Unknown
  section is populated via a callback, not its own `initSection()`
  fetch — so a future reader diffing docs against code doesn't have to
  rediscover that distinction.

## Test delta

No Python file changed, so the unittest count is unchanged:
- `unittest`: 311/311 (no new/changed Python tests — this batch is
  HTML/CSS/JS/docs only).
- `mypy`: 0 issues, 20 source files (unchanged — no `.py` file in the
  diff).
- `pwa/smoke_test.py`: PASS (9 assets served correctly) — still fetches
  by filename and checks status/content-type/`Cache-Control`, all of
  which are unchanged by this batch.
- `node --check pwa/app.js` / `node --check pwa/service-worker.js`:
  both clean.

## Manual verification (real headless-browser, Playwright/Chromium)

Ran two full passes — one against the shipped DEMO fixtures
(`opportunity-data.json`/`sample-data.json`/`watchlist-data.json`,
tickers `DEMO1`-`DEMO4`/`DEMO5`), one against a synthetic PRIVATE
snapshot (the same fixtures copied under a temporary, git-ignored
`pwa/private-data/` with a synthetic `meta.json`, deleted immediately
after) served locally via `python3 serve.py`:

- Private banner: visible and reads `data-mode="DEMO"` /
  `data-mode="PRIVATE"` correctly in each mode; PRIVATE banner text
  includes the synthetic `generated_at`/ticker count. Confirmed visible
  at both the start and end of the DEMO-mode pass.
- Opportunity Landscape groups rendered in exactly `["ACCUMULATION",
  "DISTRIBUTION_WATCH", "Unknown", "MARKUP"]` order — the fixture's
  first-seen stage order, not alphabetical/priority order — with all
  four tickers (`DEMO1`-`DEMO4`) appearing in that same relative order
  inside their groups. Identical result in PRIVATE mode against the
  same underlying fixture data.
- Watchlist groups rendered in exactly `["WATCHING", "HOLDING", "No
  portfolio status reported"]` order (the fixture has one row with
  `portfolio_status: null` → confirms the fallback-key text, not a
  crash or a silently-dropped row).
- Invalidation Unknown highlight: the fixture's `DEMO3` (the only row
  with `invalidation_known: false`) appeared in the highlight section
  AND was independently confirmed still present in its own `"Unknown"`
  stage group afterward — proving the highlight is additive, not a
  filter, in an actual rendered DOM rather than by reading the code.
- Quick Review cards: confirmed `data-expanded="false"` and the detail
  block hidden on initial render; tapping the summary button set
  `data-expanded="true"` and revealed the detail block; tapping again
  collapsed it back.
- Cross-navigation: clicking the `DEMO3` Opportunity row (which has a
  loaded Quick Review card) auto-set that card to `data-expanded="true"`
  and made its detail block visible before scrolling — confirmed the
  auto-expand actually fires, not just the pre-existing scroll/
  highlight. Clicking the `DEMO4` Opportunity row (which has NO loaded
  Quick Review card, since the tactical fixture only has 3 rows) showed
  the existing inline "No Quick Review card is currently loaded for
  DEMO4" message instead of doing nothing.
- Search: typing `demo1` left exactly `DEMO1` visible across all
  sections and hid every group left with zero matches; typing a
  non-matching string surfaced the "No ticker matches your search"
  message; clearing the field restored every row.
- Zero console errors in either mode (the one network 404 logged by
  the browser — `private-data/meta.json` in DEMO mode — is the
  documented, expected DEMO-mode-detection signal in `app.js`'s own
  `resolveMode()`, not an app bug).

Synthetic `pwa/private-data/` deleted and the local server process
stopped before staging the commit; `git status --short` confirmed
clean except the intended 5 files before `git add`.

## Hosted CI: BLOCKED — repo-wide GitHub Actions infrastructure issue, not a code defect

This is a genuine open item, not silently omitted. After pushing
`8b8596f`, the `ai-qa` workflow run (`31546488832`) failed on all
**three** attempts (the original run plus two re-run triggers: `rerun
failed jobs`, then a full `rerun workflow run`) with the same
signature every time: job completes in 2-5 seconds, `runner_id: 0` /
`runner_name: ""` (no runner was ever actually assigned), zero billable
minutes (`get_workflow_run_usage` reports `total_ms: 0` for both job
attempts), and no log content at all (log download 404s, the check
run's own `output.text`/`summary` are both empty strings) — this is not
a test/assertion failure inside the job, it's the job never starting.

This is not isolated to my push: in the same time window, an unrelated
branch (`repair/pse-eod-freshness-interlock-20260812`, not touched by
this session) shows the identical failure signature on both its `push`
and `pull_request` workflow runs. Two independent branches, one of them
entirely outside this batch's diff, failing identically at the same
time strongly indicates a repo- or account-level GitHub Actions
condition (most plausibly the private repo's included Actions
minutes/spending limit, since private repos consume billed minutes
unlike public ones) — not a defect in this commit's code. This session
has no access to the repository's GitHub billing/Actions settings to
confirm or fix this; it's an owner-only fact, flagged rather than
guessed at or silently worked around.

Given exhaustive local verification (unittest/mypy/smoke/node --check/
two full Playwright passes, all green) and that the failure signature
is identical across an unrelated branch this session didn't touch,
this is being treated as a real, disclosed **external CI-infrastructure
gap** — not a STOP condition this batch itself triggered, and not
grounds to claim green hosted CI that doesn't exist. **Ask for the
owner:** please check the niklar-stocks repository's GitHub
Actions/billing status (Settings → Billing and plans → Actions, or the
Actions tab's own banner) for a spending-limit or included-minutes
message.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: no weakening. No new fetch/error path — the
   new code operates only on rows already successfully loaded by the
   existing `initSection()` fetch/error/empty state machine, which is
   untouched. A malformed `stage`/`portfolio_status` value degrades to
   an explicit fallback bucket name (`"Unknown stage"` / `"No portfolio
   status reported"`), never a silent drop or a crash.
2. **Unnecessary abstraction/dependency**: none added. `stableGroup()`/
   `renderGroupedRows()` is one small generic pair reused identically
   for both grouped sections (no per-section duplication); no new
   library, no new build step, no new CSS framework — new styles reuse
   the existing `:root` custom-property tokens throughout.
3. **Security/privacy**: no new network call, no new client-side
   storage (`localStorage`/`sessionStorage`/`indexedDB`/
   `document.cookie` — zero matches in this diff, same as every prior
   PWA batch), search runs entirely in already-loaded DOM/memory. The
   private banner's visibility and mode-resolution logic are untouched
   byte-for-byte.
4. **Canonical boundary drift**: `ops/mobile_contract.py` and
   `ops/niklar_stocks_ops_v4.py` (both Drive-verbatim) — zero changes,
   confirmed via the commit's own file list containing neither path.

## Remaining UNKNOWNs

- Hosted CI status for commit `8b8596f` (see above) — external,
  owner-actionable, not a code defect per the evidence gathered.
- Everything already open before this sub-batch and untouched by it:
  tunnel-token rotation housekeeping, reboot-survival evidence, and
  sub-batches B (Daily GTC Plan) and C (Windows auto-update), which
  RESPONSE_SEQ 39 authorized to follow this one if tests pass and no
  STOP condition is hit — local tests passed exhaustively; the CI
  outage above is an infrastructure condition, not one of the six named
  STOP conditions, so sub-batch B follows next per that instruction.

## Review request

Per RESPONSE_SEQ 39's "checkpoint after each, continue without owner
interruption if tests pass and no STOP condition is hit, stop after the
third for ChatGPT review": this checkpoint publishes sub-batch A and
proceeds directly to sub-batch B, consistent with that instruction.
Flagging the hosted-CI gap explicitly here rather than waiting for the
stop-after-three checkpoint, since it's a new fact ChatGPT and the
owner should see as soon as it's known.
