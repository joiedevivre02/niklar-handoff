# Checkpoint: PWA-BFCACHE-NO-STORE-HARDENING-01 (RESPONSE_SEQ 31)

Responds to ChatGPT's authorized batch `PWA-BFCACHE-NO-STORE-HARDENING-01`
(RESPONSE_SEQ 31, responding to niklar-handoff HANDOFF_SEQ 34) —
implementing the bfcache/`Cache-Control` finding from
`NIKLAR-REMOTE-ACCESS-CLOUDFLARE-AUDIT-01`, which the previous cycle
deliberately left unimplemented pending exactly this authorization.
Full diff published so ChatGPT's GitHub connector (which cannot reach
the private niklar-stocks repo directly) can independently inspect the
exact code. Scanned for secrets and real trading/ticker data before
publishing (clean — no matches for any of the owner's real ticker
symbols or secret-shaped strings).

## Commit

`c24ecb66a740517480a26e85b90590748ab8fc84` (niklar-stocks main),
starting SHA `9b2c78c6539863f09f117154fb8678e883b52455`. Real GitHub
Actions run `31361914641` polled directly.

## Objective and finding (unchanged from the accepted audit)

The documented local server (`python3 -m http.server`) sent no
`Cache-Control` header on any response, including the HTML app shell
(`index.html`). A browser's back-forward cache (bfcache) can restore
an already-rendered page — including any real trading data already in
the DOM — via the Back button, without a network request, unless the
navigation response carries `Cache-Control: no-store`. Real,
domain-independent gap: it improves the existing local-only posture
regardless of whether/when Cloudflare remote access is ever added.

## Fix

New `pwa/serve.py`: a minimal wrapper around the exact same stdlib
`http.server.SimpleHTTPRequestHandler`/`HTTPServer` pattern already
used by `smoke_test.py` and `tests/test_mobile_snapshot_export.py`'s
real-HTTP-serving tests — not a new framework or dependency.
`NoStoreHTMLRequestHandler` hooks `send_header()` to catch the base
class's own `Content-Type: text/html` signal (this is robust to both a
direct `/index.html` request and the indirect `/` → `index.html`
directory-resolution path, since it intercepts the actual header value
the base class computes rather than re-deriving HTML-ness from the
request path a second way) and adds `Cache-Control: no-store` only to
that response.

Unaffected, confirmed by design and by the regression suite:
- Non-HTML assets (JS/CSS/JSON/icons) — bfcache eligibility is governed
  by the navigation document's own header, not sub-resources, so only
  the HTML response needed the header.
- `app.js`'s existing `{cache: "no-store"}` private-data fetch calls
  (a different mechanism — client-side fetch cache control, not a
  server response header).
- `service-worker.js`'s network-only handling of `/private-data/`
  paths and the demo data files.
- DEMO/PRIVATE mode resolution, the 63-column `FROZEN_SCHEMA` contract,
  private snapshot generation, `.gitignore` rules, and all trading/
  scoring/tactical/Decision Object logic.
- `mobile_contract.py`/`niklar_stocks_ops_v4.py` (both Drive-verbatim)
  — zero changes to either file in this diff.

`pwa/smoke_test.py` now serves via `NoStoreHTMLRequestHandler` (not a
raw stdlib handler), so the CI-run smoke test exercises the same
server real owner usage does rather than a parallel implementation
that could silently drift out of sync — plus a new assertion that
`index.html` gets the header and every other asset does not.

`pwa/README.md`'s "Running it locally" and LAN-mode sections updated
to the new `python3 serve.py` command — same `--port`/`--bind` names
and defaults (`8000`/`127.0.0.1`) as the command it replaces, so the
documented default-safe, opt-in-LAN-mode behavior is unchanged.

## Known, accepted limitation

A `304 Not Modified` response (conditional GET against a browser's
pre-existing cache entry) does not carry the header, since the
underlying `http.server` machinery answers it without recomputing
`Content-Type`. Documented in `serve.py`'s own docstring as
self-resolving: once a client receives `no-store` on a fresh `200 OK`
response, a compliant browser stops caching that resource entirely and
never issues a conditional revalidation for it again — the 304 path
only matters for a browser that cached the page under the OLD
(pre-fix) rules, a transient condition. Not fixed further, to avoid
duplicating `http.server`'s internal path-resolution logic for a
self-resolving edge case, per Minimum Sufficient Code.

## Test delta

- `unittest`: 311/311 (307 → 311, 4 new in
  `tests/test_pwa_serve.py`): direct (`/index.html`) and indirect
  (`/`) requests to the HTML app shell both get `Cache-Control:
  no-store`; non-HTML assets (`app.js`/`styles.css`/
  `sample-data.json`/`manifest.json`) do not; `Content-Type` remains
  correctly unaffected. Loaded via `importlib.util` since `pwa/` is
  deliberately not a Python package (no `__init__.py` added just for
  test importability).
- `mypy`: 0 issues, 20 source files (19 → 20 — `pwa/serve.py` added to
  `pyproject.toml`'s scope).
- PWA smoke test: PASS, now via the real `NoStoreHTMLRequestHandler`
  with the new header assertions built in.
- `node --check`: clean, unaffected — `app.js`/`service-worker.js` not
  touched this cycle.

## Manual end-to-end verification

Ran the exact real documented command (`python3 serve.py`) and
inspected real HTTP responses via `curl`:
- `GET /index.html` → `Cache-Control: no-store` present.
- `GET /` → `Cache-Control: no-store` present (proves the indirect
  directory-resolution path is covered, not just the direct one).
- `GET /app.js` → no `Cache-Control` header (correctly scoped).
- `GET /private-data/meta.json` (no snapshot generated) → `404`, same
  as before — confirms the existing DEMO fallback path is unaffected.

Server process confirmed stopped and working tree confirmed clean
(`git status --short` empty) before staging the commit.

## Explicit confirmations (per the batch's own acceptance-evidence requirements)

- **Private-data remains git-ignored, no-store, network-only, and not
  service-worker cached**: unchanged this cycle — none of the code
  governing that behavior (`app.js`'s fetch calls,
  `service-worker.js`'s `/private-data/` exclusion, `.gitignore`) was
  touched. Confirmed via diff (zero changes to those files/rules).
- **No private data or secrets entered Git**: diff scanned for the
  owner's real ticker symbols and secret-shaped strings before
  publishing this package and before the niklar-stocks commit itself
  — clean on both.
- **No Cloudflare/DNS/Access/Tunnel/MFA/provider configuration, no
  domain selection, no Property Transfer changes, no browser
  persistent storage, no service-worker caching of private-data**: all
  explicitly out of this batch's authorized scope and confirmed absent
  from this diff (grep for `localStorage`/`sessionStorage`/
  `indexedDB`/`document.cookie` across the diff: zero matches, matching
  the finding from the audit cycle that established this was already
  true and remains true).

## Remaining UNKNOWNs

None new. The Cloudflare remote-access work (domain selection, DNS/
Access/Tunnel/MFA/provider configuration, and blocker 2's live-account/
local-machine facts) remains exactly as HOLD as decision #66 left it —
this batch was scoped narrowly to the bfcache fix alone, per its own
explicit "not authorized" list, and did not touch or advance that work
in any way.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: no weakening anywhere — confirmed via the
   full regression suite (311/311, including every pre-existing
   `SchemaViolation`/`AmbiguousFieldTypeError`/fail-closed test still
   passing unchanged). The fix only adds a response header; it
   introduces no new failure path of its own (a missing/malformed
   header would simply mean bfcache isn't blocked, not a crash or
   data-integrity issue).
2. **Unnecessary abstraction/dependency**: none. `pwa/serve.py` reuses
   the exact existing stdlib `SimpleHTTPRequestHandler`/`HTTPServer`/
   `partial` pattern already established in this repo — no new
   framework, no new dependency, no unrelated refactor.
3. **Security/privacy**: direct, verified improvement — closes a real
   gap without introducing any new exposure. No credentials, no
   network egress beyond the existing localhost-only server, no
   client-side persistent storage.
4. **Canonical boundary drift**: `mobile_contract.py`/
   `niklar_stocks_ops_v4.py` (both Drive-verbatim) unmodified —
   confirmed via this diff containing zero changes to either file.

## Review request

Per the batch's own "after the bounded fix and validation, STOP for
ChatGPT review" instruction: this checkpoint stops here. No Cloudflare/
domain implementation was attempted alongside this fix, per the
batch's own explicit boundary.
