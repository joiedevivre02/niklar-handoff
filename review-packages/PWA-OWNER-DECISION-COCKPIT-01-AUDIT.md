# Audit: PWA-OWNER-DECISION-COCKPIT-01-AUDIT (RESPONSE_SEQ 38)

AUDIT ONLY — no implementation performed. Confirmed: no niklar-stocks
files were changed this cycle (`git status --short` empty throughout).
This document is the required review package covering all 13 output
items (F.1–13). Nothing here is authorized to be built yet.

**Congratulations noted, not glossed over**: `app.niklarintelligence.pro`
is confirmed working end-to-end — real remote iPhone access, real
PRIVATE SNAPSHOT data, Cloudflare Access in front. That's the real
milestone this whole Cloudflare initiative was for. This audit is
about what comes next.

## 1. Current-state findings — exact files/contracts inspected

- `pwa/index.html`, `pwa/styles.css`, `pwa/app.js` — the current app
  shell: three flat sections (Opportunity Landscape, Quick Review,
  Watchlist), each a simple `<ul>` of rows/cards, no grouping, no
  search, dark theme with CSS custom properties already defined
  (`--bg`, `--surface`, `--accent`, etc. — reusable for the cockpit
  redesign, not replaced).
- `ops/mobile_contract.py` — the three existing contracts
  (`to_tactical_card`, `to_opportunity_card`/`build_opportunity_landscape`,
  `to_watchlist_row`/`build_watchlist`), all pure structural transforms
  over `FROZEN_SCHEMA`, explicitly never sorting/ranking/filtering.
  No GTC-related field anywhere in `FROZEN_SCHEMA`'s 63 columns.
- `ops/mobile_snapshot_export.py` — the existing private-snapshot
  bridge (CSV/JSON → `pwa/private-data/*.json`), schema-locked to the
  63-column Master Trade Database tab specifically.
- `pwa/serve.py` — confirmed self-contained, stdlib-only, zero imports
  from `ops/` or anywhere else in the repo — directly relevant to the
  auto-update design in section 7 below.
- **Daily GTC Plan canon** — read in full from the primary Drive
  sources (not proxies), two documents, the second amending the
  first:
  - `DAILY_GTC_PLAN_RULE_CURRENT.txt` (GTC-001, approved 2026-08-07):
    defines the Daily GTC Plan as a distinct **canonical Master
    workbook tab**, "located immediately beside Opportunity Matrix" —
    confirmed separately by `STOCKS_OPS_HANDOFF_CURRENT`'s "Daily
    execution plan owner: canonical Master workbook tab `Daily GTC
    Plan`." Original required columns: Rank, Ticker, Matrix Tier,
    Stage/Level, Kind of Play, Current Reference, Buy GTC, Sell GTC,
    Mode, Simple Rule. "Never invent a GTC price or derive it from a
    Slack message" — the tab's values are themselves canonical, not
    something to recompute from Master Database fields.
  - `NIKLAR_DAILY_GTC_RANGE_AND_SHADOW_JOURNAL_RECONCILIATION_RULE_CURRENT`
    (dated 2026-08-10 — same day, **FROZEN OPERATIONAL AMENDMENT**,
    supersedes the original on the range question): Buy GTC and Sell
    GTC must now be **ranges**, not single points. RULE 3's updated
    minimum display contract: `Ticker | Current Ref | Buy GTC Range |
    Sell GTC Range | Play Type | Stage | Mode | Simple Rule` —
    explicitly preserving that "existing additional fields/ranking may
    remain; do not delete useful canonical information merely to
    satisfy this display contract," i.e. Rank/Matrix Tier from the
    original rule are not deleted, just not re-stated as mandatory in
    the amendment's own minimal list.
  - **Reconciled**: the owner's own expectation as relayed in
    RESPONSE_SEQ 38 ("Rank, Ticker, Matrix Tier, Stage/Level, Kind of
    Play, Current Ref, Buy GTC Range, Sell GTC Range, Mode, Simple
    Rule") is exactly the union of both documents — confirmed correct
    against the primary sources, not just trusted at face value.

## 2. UX information architecture for the owner cockpit

Proposed home-screen structure, top to bottom:

1. **Private banner** (existing — kept, made visually subordinate per
   A.7: smaller, less visually dominant than today, but never
   removed or hidden — it's the one thing that must never be
   de-emphasized to the point of being missed).
2. **Global ticker search bar** — pinned near the top, always visible
   (not buried in a menu). See section 4.
3. **Daily GTC Plan** — a compact, prominent card/table near the top,
   per B.1's explicit "required first-class owner view... prominent
   near the top." Not a raw list — the natural mobile-first shape is
   a compact row per ticker: `Ticker · Play Type · Stage` as the
   primary line, `Buy GTC Range` / `Sell GTC Range` as a clear
   two-line or two-column sub-block (ranges need two numbers each,
   don't compress them into unreadable one-liners), `Mode` as a small
   badge, `Simple Rule` as a one-line caption. Rank/Matrix Tier shown
   as compact prefix badges since the canon itself uses Rank as the
   tab's primary ordering — see section 3's ordering note.
4. **Opportunity Landscape**, regrouped (section 3).
5. **Positions / Portfolio grouping** and **Watch / needs-attention
   grouping** (A.5), both derived from Watchlist's existing
   `portfolio_status` field — see section 3.
6. **Quick Review (tactical detail)** — compact rows that expand on
   demand (A.4), not full cards rendered for every ticker.

Cross-navigation (existing opportunity→tactical jump) preserved,
extended: tapping a Daily GTC Plan row or a grouped
Opportunity/Watchlist row also jumps to that ticker's Quick Review
detail, using the exact same ticker-string-matching mechanism already
in `app.js`'s `jumpToTacticalCard()` — no new logic, one more caller.

## 3. Proposed grouping rules — each mapped to an existing field, source order preserved

Every grouping below is **presentation-only bucketing by an existing
field's already-computed value** — never a new score, never a
recomputed rank, never a combination of fields into a new judgment.
Within each bucket, rows keep the exact relative order they arrived
in from the source contract (a stable partition, not a sort).

| Grouping | Source field (existing) | Bucketing logic |
|---|---|---|
| Opportunity Landscape, by stage | `stage` (`to_opportunity_card`'s `Current Stage` passthrough) | One bucket per distinct `stage` value observed in the data (e.g. "Early Markup," "Base Building," "Recovery") — buckets appear in the order their first member appears in the source list, not alphabetically or by any invented priority. |
| Positions / Portfolio | `portfolio_status` (`to_watchlist_row`'s raw passthrough) | Two buckets: "Holdings" (non-empty `portfolio_status`) and everything else feeds the Watch grouping below. No interpretation of *which* status values count as a holding beyond "a value is present" — matches `build_watchlist()`'s own documented refusal to interpret this field's value set. |
| Watch / needs-attention | `invalidation_known` (existing boolean, `to_opportunity_card`) | Two buckets: rows where `invalidation_known === false` surface first under a plain "Invalidation not yet known" heading — stating the field's own existing meaning verbatim, not a new "risk" label. This is the one grouping worth flagging explicitly for review: it's a legitimate bucketing of an existing boolean, but naming it "needs attention" rather than "invalidation unknown" would cross into inventing a judgment. **Recommendation: label it exactly "Invalidation Unknown," not "Needs Attention," to stay unambiguously on the presentation-only side of the line** — flagged for ChatGPT's explicit confirmation since it's the least clear-cut of the three. |

**Ordering note**: none of `to_opportunity_card`/`to_watchlist_row`
provide an explicit ordering field beyond raw `tactical_priority`
passthrough (itself explicitly "not for re-ranking" per the existing
docstring) — so "preserve canonical source order within each group"
means exactly "stable-partition the existing array," not "sort by
`tactical_priority`." The Daily GTC Plan tab is the one place canon
*does* provide an explicit ordering field (`Rank`) — Daily GTC Plan
rows should render in `Rank` order since that's the canonical tab's
own stated ordering, not a client-invented one (see section 5).

## 4. Search behavior and ticker-detail interaction

- A single text input, filtering client-side across whatever
  contracts are currently loaded (Opportunity, Watchlist, Tactical,
  Daily GTC Plan) — pure substring match on `ticker`, case-insensitive.
  No fuzzy matching, no ranking of results by relevance (that would be
  client-side scoring).
- Typing surfaces, per matching ticker: whichever of the four
  contracts currently has a row for it — e.g., "AAA: in Opportunity
  Landscape (Early Markup), in Daily GTC Plan (Rank 3), no Watchlist
  entry, no Quick Review card loaded." This is a presentation
  aggregation of already-loaded data, not a new query capability —
  nothing is fetched that isn't already being fetched today.
- Selecting a search result behaves exactly like the existing
  opportunity→tactical jump: scroll-to + highlight if a matching
  Quick Review card is loaded, otherwise the existing "no detail
  loaded" inline message — reused, not reinvented.

## 5. Exact Daily GTC source and minimum proposed GTC snapshot contract

**Source**: the Daily GTC Plan canonical Master workbook tab — a
**separate tab** from the 63-column Master Trade Database tab this
repo's `FROZEN_SCHEMA` already mirrors. Confirmed via
`STOCKS_OPS_HANDOFF_CURRENT`'s explicit tab-ownership statement. This
means: **no field in the existing 63-column export carries Daily GTC
Plan data** — `Current Reference` in `FROZEN_SCHEMA` is a Master
Database field, not provably the same value as the GTC tab's own
`Current Ref` column, and canon explicitly forbids deriving GTC prices
from anything other than the GTC tab's own frozen values ("Never
invent a GTC price"). The two must not be silently treated as
interchangeable.

**Proposed minimum new contract** (not yet implemented — this is the
audit's proposed design): a new, separate, frozen column schema for
Daily GTC Plan rows, modeled exactly on how `FROZEN_SCHEMA` +
`validate_schema()` already lock the Master Database shape — same
fail-closed pattern, new constant:

```
GTC_PLAN_SCHEMA = [
    "Rank", "Ticker", "Matrix Tier", "Stage", "Kind of Play",
    "Current Ref",
    "Buy GTC Range Low", "Buy GTC Range High",
    "Sell GTC Range Low", "Sell GTC Range High",
    "Mode", "Simple Rule",
]
```

Range fields split into explicit Low/High pairs (not one opaque
range-string column) so the PWA can reuse the *exact* existing
`_zone_or_none()` / `zoneText()` helpers already in
`mobile_contract.py`/`app.js` for `Preferred Entry Low/High` — same
pattern, not a new one, and satisfies RULE 3's "both range endpoints
must be printed explicitly and validated independently."

**Proposed minimum new export path**: a new, separate CSV/JSON input
(the owner exports the Daily GTC Plan tab separately from the Master
Trade Database tab — two exports, two files, matching the two
distinct canonical tabs), a new small transform module or function
(e.g. `to_gtc_plan_row()`, pure passthrough — no computation, exactly
like `to_watchlist_row()`'s raw `portfolio_status` handling) producing
a new `pwa/private-data/gtc-plan.json`. **Does not touch**
`ops/mobile_snapshot_export.py`'s existing 63-column path at all — a
fully separate, additive artifact, per the explicit "add a separate
minimum GTC snapshot artifact only if the audit proves it is needed"
boundary. `Rank` (numeric, already canonical) is treated as the
Daily GTC Plan's own explicit ordering field — the one place in this
whole cockpit design where a numeric sort is legitimate, because canon
itself, not the client, defines that order.

**Fail-closed behavior**: if the owner's Daily GTC Plan export doesn't
match `GTC_PLAN_SCHEMA` exactly, the export fails closed exactly like
the existing Master Database bridge does — no partial/best-effort
rendering, no field synthesis.

## 6. Exact files expected to change if implementation is later authorized

- `pwa/index.html` — restructure into grouped sections + search bar +
  GTC Plan section (new markup, new `<template>` blocks).
- `pwa/styles.css` — new grouping/search/GTC styles, reusing existing
  CSS custom properties (`--bg`, `--surface`, `--accent`, etc.) rather
  than introducing a second design language.
- `pwa/app.js` — grouping logic (pure array partitioning, no scoring),
  search filtering, GTC Plan rendering + fetch, extended
  cross-navigation.
- `ops/mobile_contract.py` (or a new sibling module, e.g.
  `ops/gtc_plan_contract.py`, to avoid growing an already-large file
  — genuine judgment call, flagged for ChatGPT's preference) — the
  new `GTC_PLAN_SCHEMA` constant and `to_gtc_plan_row()`/
  `build_gtc_plan()` functions.
- `ops/mobile_snapshot_export.py` or a new sibling exporter — the new
  GTC CSV/JSON→`gtc-plan.json` path. Existing 63-column path
  unmodified.
- `pwa/README.md`, `ops/README.md` — documentation for the new export
  step and cockpit layout.
- New tests mirroring the existing patterns in
  `tests/test_mobile_snapshot_export.py` (schema-lock enforcement,
  fail-closed on mismatch, real-HTTP-serving proof) for the GTC path.
- **Not changed**: `ops/niklar_stocks_ops_v4.py` (Drive-verbatim, no
  Master Database schema touch, per the explicit boundary),
  `service-worker.js` (the GTC private-data path needs the exact same
  network-only exclusion the existing `/private-data/` paths already
  get — confirmed the existing `url.pathname.includes("/private-data/")`
  check already covers any new file under that same directory with
  zero code change needed there).

## 7. Safe Windows auto-update design

**Failure envelope**: fails closed at every step —
1. `git status --porcelain` non-empty → abort immediately, touch
   nothing, log the reason. Never `git stash`/discard local state
   automatically.
2. `git fetch origin main` fails (network, auth) → abort, log, leave
   deployment untouched — this is the same failure mode as any
   ordinary transient network issue, not treated as an update failure
   requiring rollback.
3. Local HEAD already equals `origin/main` → no-op, nothing to do.
4. Fast-forward check: only proceed if local HEAD is a strict ancestor
   of `origin/main` (`git merge-base --is-ancestor HEAD origin/main`).
   If not (local history has diverged — which shouldn't happen on an
   owner-only deployment checkout that's never committed to locally,
   but must still be checked, not assumed) → abort, log, **never**
   merge/rebase/reset/force to reconcile.
5. `git merge --ff-only origin/main` — the only mutation the script
   ever performs on the repo itself.

**Restart decision — a real, minimal finding**: `pwa/serve.py` is
confirmed self-contained with zero imports from `ops/` or anywhere
else in the repo (checked directly). Since it's a stdlib-only static
file server reading from disk per-request, **only a change to
`pwa/serve.py` itself requires restarting the running process** —
every other change (HTML/CSS/JS/JSON, `ops/`, docs) is picked up by
the already-running process on the next request with zero restart,
especially since the HTML app shell already carries `Cache-Control:
no-store` (landed in `PWA-BFCACHE-NO-STORE-HARDENING-01`), so a
browser refresh always fetches fresh content. Decision rule:
`git diff --name-only <old-HEAD> <new-HEAD> | grep -qx "pwa/serve.py"`
→ restart the `NiklarPwaServe` task only if matched.

**Rollback / "last known working state" — a genuine open question,
not resolved unilaterally**: the instruction's own list of forbidden
mechanisms explicitly names `reset` alongside merge/rebase/force. Two
readings are both defensible and materially different:
- **(a) Strict reading**: `reset` is never used by this mechanism for
  any purpose, including failure recovery. In that reading, "preserve
  the last known working app state" for a health-check failure that
  does *not* touch `pwa/serve.py` is automatically satisfied — the
  running process is untouched by definition (see the restart-decision
  rule above), so the owner's PWA keeps serving exactly what it was
  serving before, unaffected by whatever is now on disk. For a
  health-check failure that *does* require a `pwa/serve.py` restart,
  under this reading the script would restart, observe the failure,
  and then simply **not** attempt any further automated recovery —
  fail loudly, log clearly, leave the task in its failed state, and
  wait for a human (owner or a corrective push) to resolve it. This is
  fully compliant with "no blind destructive continuous deployment."
- **(b) Narrower reading**: the forbidden "reset" refers specifically
  to using reset as part of *ordinary update semantics* (i.e., never
  force-reconcile a diverged history to make an update succeed) — a
  *scripted, targeted* rollback to a specific previously-recorded good
  commit, invoked only from the failure-handling branch after a
  verified health-check failure, is a materially different operation
  (recovering to a known-good state you were just at, not discarding
  unknown local work) and would be in scope.

**This audit does not choose between (a) and (b)** — it's exactly the
kind of judgment call that shouldn't be resolved silently. **Recommendation:
adopt (a)** as the Minimum Sufficient choice for the first
implementation increment (no rollback mechanism at all beyond "don't
restart on failure"), since it fully satisfies every stated
requirement without touching the explicitly-named-forbidden operation
at all, and defer (b) to a future increment only if (a) proves
operationally insufficient. Flagged explicitly for ChatGPT's
confirmation before implementation.

**Cadence and task identity — informed by the newly-reported Windows
facts**: `niklar-stocks` is a **private** GitHub repository — `git
fetch`/`pull` requires authenticated access (credential manager, SSH
agent, or a stored PAT), which on Windows is virtually always bound to
the *interactive user's* profile, not the `SYSTEM` account. This is
almost certainly the same underlying reason `NiklarPwaServe` itself
couldn't run under `SYSTEM` (fact #9 — Microsoft Store Python under
the `bamis` profile wasn't reachable from `SYSTEM`) even though the
specific mechanism differs (Python interpreter path vs. git
credentials). **Recommendation: run the auto-update task under the
same owner-logon Scheduled Task context already established and
working for `NiklarPwaServe`**, not `SYSTEM` — consistency with an
already-proven-working pattern, not a new unknown. Cadence: a
repeating trigger (e.g. every 15 minutes) *in addition to* logon,
mirroring the polling approach the response itself asked to be
audited (a webhook would require exposing a new public endpoint on
the owner's machine — explicitly out of scope, "no new live
backend/API," and no such endpoint exists today).

**Health check after update**: `curl -i http://127.0.0.1:8000/index.html`
(or PowerShell's `Invoke-WebRequest`), asserting HTTP 200 and
`Cache-Control: no-store` present — reusing the exact same assertion
already used in `pwa/smoke_test.py` and the Windows deployment
runbook's own D.12 step, not a new check invented for this purpose.

**Private-data preservation**: the update mechanism only ever touches
git-tracked paths via `git merge --ff-only`; `pwa/private-data/` is
`.gitignore`d and was never part of any commit — a fast-forward merge
cannot touch it by construction, not merely by convention.

**No credential handling**: the script never touches Cloudflare
Tunnel tokens, Access configuration, DNS, or any Cloudflare state —
those live entirely outside the git repository and outside this
mechanism's scope.

## 8. Security/privacy audit

- No new public/anonymous surface — the update mechanism runs
  entirely locally, polling GitHub (already-authenticated, already-
  read-only for this purpose) and localhost; it introduces no new
  network-reachable endpoint.
- The cockpit UX changes are presentation-only — no new data leaves
  the browser, no new fetch target beyond the existing `private-data/`
  and demo JSON paths plus the one new `gtc-plan.json` file, served
  from the exact same origin under the exact same `Cache-Control:
  no-store`/service-worker-exclusion protections already proven for
  `private-data/`.
- Search operates entirely on already-fetched, already-in-memory data
  — no new query goes to any server.
- No browser persistent storage introduced anywhere in this design —
  confirmed by design, matching the existing zero-`localStorage`/
  `sessionStorage`/`indexedDB` posture already verified in the
  Cloudflare audit cycle.

## 9. Drift audit against existing PWA/mobile-contract rules

No drift found in the proposed design:
- Grouping/search are explicitly presentation-only, verified against
  every existing "does not sort/rank/filter" docstring in
  `mobile_contract.py` — the design adds no re-ranking anywhere.
- The one grouping requiring explicit sign-off (Watch/needs-attention,
  section 3) is flagged rather than silently implemented as if
  unambiguous.
- The GTC Plan contract is proposed as a **new, separate** artifact,
  not a retrofit into `FROZEN_SCHEMA` or the existing 63-column bridge
  — preserves the existing contract exactly as instructed.
- `mobile_contract.py`/`niklar_stocks_ops_v4.py` (Drive-verbatim)
  remain untouched in this proposal — only new, additive code is
  contemplated.

## 10. Mobile usability audit

- Current layout (three flat `<ul>` sections) is confirmed too dense
  for a phone-first "decision" reading: a Quick Review card today
  renders full detail for every ticker with no collapse — on a real
  PSE-sized universe (the owner's real snapshot has been as large as
  25 tickers in earlier testing), that's a very long scroll before
  reaching the Daily GTC Plan or Watchlist.
- Proposed fix (A.4): compact rows by default, tap/expand for full
  tactical detail — reuses the exact same `<template>`-clone rendering
  pattern already in `app.js`, just with an added collapsed/expanded
  state toggle, not a new rendering engine.
- Existing dark theme, touch-target sizing (`.opportunity-row` already
  has `tabIndex`/keyboard handling for accessibility), and viewport
  meta tag are all already mobile-appropriate and reused unchanged.
- Search bar placement (pinned near top, not buried) and Daily GTC
  Plan prominence both directly address the "optimized for actual
  trading decisions" requirement — the two things an owner most needs
  to check quickly (search a specific ticker; see today's GTC plan)
  are the fastest to reach.

## 11. Test plan (for the later implementation cycle, not run now)

- Private vs. demo: existing 3-scenario Playwright pass (PRIVATE with
  data, DEMO/404, configured-but-broken) re-run unchanged, plus a 4th
  scenario for the new GTC Plan file's own DEMO/PRIVATE/missing states.
- Search: filters correctly across all four contracts; case-
  insensitive; empty query shows everything; no-match state is
  explicit, not a blank screen.
- Grouping: unit tests proving stable-partition behavior (a
  deliberately out-of-group-order fixture proves no re-sort within
  or across groups, mirroring the existing "deliberately
  out-of-priority-order fixture" pattern already used for
  `build_opportunity_landscape()`).
- GTC fidelity: schema-lock enforcement (missing/extra/reordered GTC
  columns fail closed), `Rank`-ordering preserved, range Low/High
  pairs round-trip correctly, fail-closed on a malformed/incomplete
  range per RULE 1/2's "incomplete for publication" language.
- Local no-store behavior: unaffected, re-verified via the existing
  `pwa/smoke_test.py` pattern extended to the GTC endpoint.
- Update failure cases: dirty worktree (script aborts, working
  deployment untouched — assert via a test harness simulating a
  local uncommitted change); diverged history (script aborts, no
  merge attempted); health-check failure post-restart (script does
  not attempt further automated recovery, per section 7's
  recommendation).
- Dirty-worktree case: explicit test asserting zero git mutation
  occurs when `git status --porcelain` is non-empty.
- Private-data preservation: assert `pwa/private-data/` survives an
  update cycle untouched (already true by construction, verified
  anyway).
- Static-update visibility from iPhone: after a static-only push (no
  `pwa/serve.py` change), confirm the update is visible on a phone
  refresh with **no** service restart — the concrete proof of section
  7's central finding.

## 12. Proposed smallest founder-reviewable implementation increment

Two independent increments, each reviewable and revertible on its own
— **not** one combined batch:

1. **Cockpit UX** (sections 2–4, 6 minus GTC-specific files, 8–10):
   grouping + search + compact/expand tactical cards. Fully
   implementable today using only fields already in the existing
   three contracts — no new canonical source needed.
2. **Daily GTC Plan** (section 5, the GTC-specific parts of section
   6): the new schema/export/contract/rendering, gated on ChatGPT
   confirming the proposed `GTC_PLAN_SCHEMA` design in section 5.

The **Windows auto-update mechanism** (section 7) is a third,
fully independent increment — no dependency on either of the above,
touches Windows/Scheduled-Task configuration rather than PWA code,
and per the response's own script-landing precedent should likely
follow the same "prove it once, supervised, before landing" pattern
already established for `Register-NiklarPwaStartupTask.ps1`.

## 13. STOP conditions and genuine unresolved facts

Two genuine open items flagged rather than resolved unilaterally, both
already called out above:

1. **Section 3**: the "Watch/needs-attention" grouping's exact label —
   recommended "Invalidation Unknown" (verbatim field meaning) over
   any phrasing implying a judgment ("needs attention," "at risk").
2. **Section 7**: the rollback/"last known working state" question —
   recommended reading (a) (no automated `git reset`-based rollback,
   ever; rely on the restart-decision rule to naturally preserve the
   running process's state) over reading (b) (a narrower, scripted
   rollback-only use of `reset`).

No other genuinely unresolved facts — the Daily GTC Plan source and
minimum contract were fully resolvable from primary Drive canon (both
documents read in full, not summarized), and every UX/grouping
decision maps to an already-existing field with no invented semantics.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: the GTC export and the auto-update mechanism
   both fail closed at every named point (schema mismatch, dirty
   worktree, diverged history, health-check failure) — none silently
   degrade or guess.
2. **Unnecessary abstraction/dependency**: no new framework, no
   `pywin32`/NSSM, no webhook infrastructure, no new JS build step —
   every proposed piece reuses an existing pattern already in this
   repo (the schema-lock pattern, the `_zone_or_none()` helper, the
   `<template>`-clone rendering, the Scheduled Task mechanism).
3. **Security/privacy**: no new network surface, no new persistent
   client storage, no credential handling anywhere in the auto-update
   design, private-data protections extend automatically to the new
   GTC path via the existing path-substring service-worker check.
4. **Canonical boundary drift**: zero changes to
   `mobile_contract.py`/`niklar_stocks_ops_v4.py` proposed; the Daily
   GTC Plan is treated as a genuinely separate canonical artifact, not
   folded into or confused with the Master Database schema.

## Review request

Per the response's own "This is AUDIT ONLY... Stop for ChatGPT review"
instruction: this checkpoint stops here. No cockpit redesign, GTC
snapshot extension, or Windows auto-update mechanism has been
implemented. The two flagged judgment calls (section 13) are the only
open questions before an implementation batch could be authorized.
