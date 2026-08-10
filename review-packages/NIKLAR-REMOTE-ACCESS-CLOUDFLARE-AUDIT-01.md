# Audit checkpoint: NIKLAR-REMOTE-ACCESS-CLOUDFLARE-AUDIT-01 (RESPONSE_SEQ 29)

READ-ONLY / NO CONFIGURATION OR CODE MUTATION, per the audit task's
own mode. Responds to RESPONSE_SEQ 29 (correcting RESPONSE_SEQ 28,
which authorized `CLOUDFLARE-PRIVATE-REMOTE-ACCESS-01` directly before
this audit-first sequence — that authorization is superseded and was
never acted on by this session; nothing from RESPONSE_SEQ 28 was
implemented).

**Disposition up front: this audit is PARTIAL, not complete.** Two of
the five required read-set items are genuinely out of this session's
reach, for reasons explained below rather than guessed past. Everything
this session *can* verify from within the niklar-stocks repo itself is
covered fully.

## 1. DISPOSITION

**NOT_READY.** Real, unresolved UNKNOWNs remain in sections A, D, E
(below) — none fabricated or approximated. See section 7 for the exact
two gaps and what would close each.

## 2. Recommended exact hostname and routing pattern, with collision analysis

**Not determinable from this session.** The audit's own read-set item
4 calls for "Property Transfer CURRENT handoff... especially the
known-good production edge state and wildcard/subdomain routing" —
that repository/canon is not attached to this session (this session's
GitHub scope is `niklar-stocks` + `niklar-handoff` only), and this
session was separately, directly instructed earlier by the owner to
"stay in niklar-stocks lane" regarding `niklarproperty.com` when it
surfaced as an apparently-unrelated screenshot. RESPONSE_SEQ 29 now
asks for exactly the cross-project read that instruction told this
session not to pursue — a real conflict, flagged to the owner rather
than resolved unilaterally in either direction (see section 7).

In principle (general Cloudflare practice, not specific to this
account's actual config, so not to be treated as a finding): a
dedicated subdomain (e.g. `app.niklarproperty.com` or
`stocks.niklarproperty.com`) with its own explicit Worker route/DNS
record would need to be checked against the existing
`*.niklarproperty.com/*` wildcard's actual route priority in the real
Cloudflare dashboard — Cloudflare evaluates routes by specificity, but
confirming that requires seeing the real route table, not assuming it.

## 3. Security controls required and current evidence for each

App-layer controls this session CAN verify directly from the repo:

| Control | Evidence |
|---|---|
| No public hosting / no public endpoint currently | Confirmed: `pwa/README.md` states this explicitly; nothing in the repo starts a network listener beyond the documented `127.0.0.1`-bound local server. |
| `private-data/` never committed | Confirmed via `git check-ignore` (re-verified this cycle, unchanged from the accepted batch). |
| No-store fetches for data endpoints | Confirmed: `pwa/app.js` fetches both `private-data/meta.json` and the demo/private data endpoints with `{cache: "no-store"}` (2 call sites, grepped directly). |
| Service worker never caches `private-data/` | Confirmed: `pwa/service-worker.js`'s `fetch` handler explicitly network-only's any path containing `/private-data/` or the three demo data filenames; only app-shell assets (`index.html`/`styles.css`/`app.js`/`manifest.json`/`icons/icon.svg`) are cache-first. Read the full file this cycle to confirm, not assumed from memory. |
| No client-side persistent storage of private data | Confirmed via a direct grep of `app.js` for `localStorage`/`sessionStorage`/`indexedDB`/`document.cookie` — zero matches. All private-data handling is in-memory fetch + DOM render only; nothing persists across a tab close. |

**NEW FINDING this cycle (not previously documented) — bfcache logout-bypass risk**:
the documented local server (`python3 -m http.server`) sends **no**
`Cache-Control` header on `index.html` or any asset (confirmed by
directly inspecting real HTTP response headers from the documented
command this cycle). Modern browsers' back-forward cache (bfcache) can
restore a fully-rendered page — including any real trading data
already in the DOM — via the Back button, without a network request,
unless the document response includes `Cache-Control: no-store`. Since
Cloudflare Access enforces authorization at the network edge (in front
of the origin), a bfcache restoration that doesn't touch the network
would not be re-checked by Access at all. **This is a real, currently
unaddressed gap for the "logout then browser back/cache behavior"
threat (section F below) — the app currently has no defense against
it.** Not something Cloudflare Access can be assumed to close on its
own without confirming its specific logout/session-invalidation
behavior (see section 7's second gap).

Controls this session CANNOT verify (require live Cloudflare account
state, unobservable from this environment): owner-only identity allow
rule presence/scope, MFA enforcement, Everyone/Bypass policy absence,
current Access session/logout behavior, tunnel configuration.

## 4. Existing Niklar settings/contracts confirmed preserved

**Yes, confirmed, and none would need to change for a Tunnel+Access
front-end as such.** Cloudflare Tunnel + Access operate entirely at
the network/transport/auth layer, in front of the origin process — they
have no visibility into or interaction with: the 63-column
`FROZEN_SCHEMA` mobile contract, the private-snapshot export flow
(`ops/mobile_snapshot_export.py`), the DEMO/PRIVATE mode resolution
logic (`app.js`'s `resolveMode()`), git-ignore protections, or any
trading/scoring/tactical/invalidation/Decision Object boundary. Adding
a Tunnel in front of the existing `python3 -m http.server` origin (or
an equivalent static server) changes *how the origin is reached*, not
*what the origin serves or how it's built*. The one exception is the
bfcache finding above — that's an origin-response-header change
(`Cache-Control`), not a contract change.

## 5. Exact implementation delta

- **Repo/code changes**: one recommended, not yet made (audit is
  read-only): add explicit `Cache-Control: no-store` (or equivalent)
  response headers to the app shell's HTML response — closes the
  bfcache gap in section 3. This is independent of which Cloudflare
  routing pattern is eventually chosen, and safe to consider even
  before the two blockers below resolve, since it strictly improves
  the existing local-only posture too.
- **Machine-local configuration**: `cloudflared` installation/service
  setup on the owner's machine, tunnel credentials file — entirely
  outside this repo, owner-only.
- **Cloudflare provider configuration**: DNS record, Worker route
  (pending collision analysis — blocked, section 7), Access
  application + policy + identity provider/MFA setup — owner-only,
  interactive, outside this session's reach.
- **Owner-only actions**: everything in the two bullets above; also
  confirming the current Cloudflare account's plan/tier for cost
  implications (section on cost below).

## 6. Required tests/smokes for implementation acceptance

App-side (this session can author and run these): a smoke test
asserting the app shell response carries `Cache-Control: no-store` (or
the chosen equivalent) once that fix lands; the existing
`pwa/smoke_test.py` and the 3-scenario Playwright pass already cover
DEMO/PRIVATE mode correctness and would need re-running after any
origin change, not because this audit found a regression risk there.

Cloudflare-side (cannot be authored without the real environment):
unauthenticated-hostname-request test, wrong-identity test, MFA-failure
test, Access-bypass-policy test, tunnel-down test, DNS/Worker-route
collision test against the real `*.niklarproperty.com/*` route.

## 7. Risks/UNKNOWNs/blockers

**Two real blockers, both requiring the owner's direct decision — not
something this session should resolve silently in either direction:**

1. **Cross-project scope conflict.** RESPONSE_SEQ 29's read-set item 4
   asks this session to read Property Transfer's Drive canon and
   current handoff for niklarproperty.com's live routing facts. This
   session was earlier told directly by the owner, in this same
   conversation, to "stay in niklar-stocks lane" regarding
   `niklarproperty.com` — at the time, in a context where it looked
   like unrelated name-sharing. RESPONSE_SEQ 29 makes clear the two
   are not infrastructure-independent if Niklar Stocks is hosted on a
   `niklarproperty.com` subdomain: a real collision risk against
   Property Transfer's live Worker route is explicitly named. This
   session will not add the Property Transfer repository or read its
   canon without the owner's explicit go-ahead for this specific,
   bounded, read-only purpose — that's the owner's call, not
   ChatGPT's or Claude's to make on their behalf, and the earlier
   instruction is the more recent, more specific signal until told
   otherwise.
2. **Live-infrastructure/local-machine state is unobservable from this
   environment.** This session runs in an isolated cloud container —
   it has no access to the owner's actual Cloudflare account
   (dashboard, API, DNS records, Worker routes, Access policies,
   account plan/tier) or their local Windows laptop (cloudflared
   service state, PWA process lifecycle, restart/reboot behavior).
   Sections A, D, E, and part of G of the audit task depend entirely
   on that state. Closing this gap requires either the owner reporting
   the specific facts directly, or a different mechanism this session
   doesn't currently have (e.g., a Cloudflare API token deliberately
   granted for read-only inspection — itself a credentials decision,
   OWNER_ONLY).

One real, verified finding (not a blocker, a fix candidate): the
bfcache logout-bypass gap in section 3.

## 8. Minimum authoritative read-set actually used

- This Drive sync doc, RESPONSE_SEQ 29 in full; RESPONSE_SEQ 28 only
  as characterized within RESPONSE_SEQ 29's own "CORRECTION" section
  (RESPONSE_SEQ 28's full original text was not separately fetched —
  not needed, since RESPONSE_SEQ 29 supersedes it for execution
  purposes and nothing from it was ever implemented).
- niklar-handoff `CURRENT_HANDOFF.txt` (HANDOFF_SEQ 31) to establish
  current niklar-stocks HEAD and private-PWA owner-use state.
- niklar-stocks current HEAD (`9b2c78c`): `pwa/app.js`,
  `pwa/service-worker.js`, `pwa/README.md`, `.gitignore`, and a direct
  HTTP header check against the documented local server command (run
  and killed within this session, no persistent process left).
- **NOT read**: Property Transfer Drive canon or GitHub repo/handoff
  (cross-project scope conflict, see section 7); the owner's live
  Cloudflare account or local machine (not observable from this
  environment).

## 9. Explicit confirmation: NO configuration/code/provider mutation performed during audit

Confirmed. This cycle's only actions were: reading existing repo files
(no edits), and running the *already-documented, already-accepted*
local server command briefly to inspect its real HTTP response headers
— the process was killed immediately after, no files were created or
changed, `git status --short` is empty. No Cloudflare, DNS, Access,
Tunnel, or any provider was touched or contacted.

## 10. SAFE TO PREPARE IMPLEMENTATION PROMPT: NO

Not until the two blockers in section 7 resolve. The app-layer
evidence (sections 3–6) is solid and reusable once they do.
