# Runbook: NIKLAR-REMOTE-ACCESS-CLOUDFLARE-IMPLEMENTATION-01 (RESPONSE_SEQ 36)

**NOTHING IN THIS BATCH HAS BEEN EXECUTED BY THIS SESSION.** This is
the single most important fact in this checkpoint, and it applies to
every one of RESPONSE_SEQ 36's 17 numbered execution steps, not a
subset. Stated plainly rather than glossed over:

> This Claude session runs in an isolated cloud container. It has no
> access to the owner's Windows machine (no shell, no file system, no
> ability to run PowerShell there) and no access to the owner's live
> Cloudflare account (confirmed blocked by this environment's network
> proxy in an earlier audit cycle — a direct request to
> `api.cloudflare.com` returned a blocked-403). Installing
> `cloudflared`, creating a Tunnel, configuring Access, registering a
> Scheduled Task, verifying from an iPhone, and performing a reboot
> test are all physically impossible for this session to perform.
> Every fact established in this entire Cloudflare initiative — the
> zone status, the Zero Trust plan, the Access setup, the cloudflared
> absence — was established because the **owner** checked and
> reported it, not because this session observed it directly. This
> cycle is no different.

What this checkpoint **is**: the complete, consolidated,
ready-to-execute runbook — matching every one of RESPONSE_SEQ 36's
lettered/numbered steps exactly — for the **owner** to run on their
own Windows machine, plus an evidence-report template to fill in
afterward. Nothing here is new technical content beyond what
`NIKLAR-WINDOWS-MACHINE-DEPLOYMENT-AUDIT-01` already audited and
RESPONSE_SEQ 36 already accepted; this consolidates it into one
sequenced, owner-runnable procedure and adds the pieces RESPONSE_SEQ
36 specified beyond the machine-side script (the Tunnel name, the
target hostname, the Access verification, the remote/reboot tests).

## Before starting

- Run everything in an **elevated** ("Run as Administrator") PowerShell
  prompt on the Windows machine that will run Niklar permanently.
- Have the `niklar-stocks` repo checked out locally already (this
  runbook does not clone it).
- Know the exact local path to that checkout — you'll need it below.

---

## A. Windows / cloudflared

### A.1 — Determine repo path and Python interpreter (owner-run, not scripted separately)

The already-reviewed `Register-NiklarPwaStartupTask.ps1` (sent to you
directly in the prior cycle, and republished in full in
`review-packages/NIKLAR-WINDOWS-MACHINE-DEPLOYMENT-AUDIT-01.md`) does
this itself: edit its `$RepoPath` variable at the top to your actual
checkout path, and it resolves + verifies the Python interpreter path
automatically (via the `py` launcher), failing closed with a clear
error if either can't be verified — exactly as required. Don't run it
yet; it's Step C.10 below, after the Cloudflare side is set up.

### A.2 — Install cloudflared, if absent

Confirmed absent last cycle (`Get-Command cloudflared.exe` and
`Get-Service Cloudflared` both returned nothing). Install via the
current official method:

```powershell
winget install --id Cloudflare.cloudflared
```

(If `winget` isn't available, download the current Windows installer
directly from Cloudflare's own downloads page — confirm you're getting
it from `developers.cloudflare.com` / Cloudflare's official GitHub
releases, not a third-party mirror.)

Verify:
```powershell
cloudflared.exe --version
```

### A.3 — Create the dedicated Tunnel

In the Cloudflare Zero Trust dashboard (**Networks → Tunnels**), create
a tunnel named exactly `niklar-intelligence` — or, if the guided setup
from the prior cycle already created one for this exact purpose, reuse
it rather than creating a duplicate. This step is entirely dashboard-
driven; there is nothing to script.

### A.4 — Tunnel token handling (secret — never scripted, never saved)

The dashboard will show a one-time install command containing your
real Tunnel token. **Do not paste it into any saved file, this chat,
a screenshot, or any script.** Copy it, and use it directly in the one
command below, typed/pasted straight into your own elevated PowerShell
prompt:

```powershell
cloudflared.exe service install <PASTE-YOUR-REAL-TUNNEL-TOKEN-HERE>
```

This single command both installs `cloudflared` as a genuine Windows
Service (correct SCM integration, not a hand-rolled one) and stores
the token in `cloudflared`'s own local, OS-protected state — nothing
further touches it.

### A.5 — Configure Windows Service Recovery for cloudflared

```powershell
sc.exe failure Cloudflared reset= 86400 actions= restart/60000/restart/60000/restart/60000
```

(Note the required space after every `=` — a well-known `sc.exe`
syntax requirement.) This restarts the service up to 3 times, 60
seconds apart, on failure, with the failure counter resetting after 24
hours of stability.

---

## B. Cloudflare hostname / Access

### B.6 — Route the hostname through the Tunnel

In the same Tunnel's dashboard settings (**Public Hostname** tab), add:
- Hostname: `app.niklarintelligence.pro`
- Service: `HTTP` → `127.0.0.1:8000`

### B.7-8 — Verify the Access policy before considering this step done

Before treating the route as accepted, open **Access → Applications**
and confirm the application covering `app.niklarintelligence.pro`
has:
- An explicit **owner-only** Include rule (your specific
  email/identity) — not a domain-wide or "Everyone" rule.
- **No** "Bypass" policy of any kind.
- The identity provider / MFA method you configured is actually
  attached to this specific application's policy, not just created
  and left unattached.

**If the live policy is broader or ambiguous in any way — STOP before
this hostname goes live.** Do not proceed to Step D's remote
verification until this is unambiguous. This is the single most
security-critical check in this whole runbook: getting it wrong means
your real trading data would be reachable by anyone who finds the
hostname, not just you.

### B.9 — Nothing else

Do not create any other DNS record, Gateway DNS location, WARP
enrollment requirement, router port-forward, or any origin-IP
exposure. None of that is needed for this architecture (Cloudflare
Tunnel is outbound-only from `cloudflared` — no inbound port is ever
opened on your machine or router).

---

## C. PWA reboot survival

### C.10 — Register the Scheduled Task

Edit `$RepoPath` at the top of `Register-NiklarPwaStartupTask.ps1` to
your actual checkout path, then run it in the same elevated PowerShell
prompt:

```powershell
.\Register-NiklarPwaStartupTask.ps1
```

This registers exactly the accepted, audited configuration — `SYSTEM`
principal, `AtStartup` trigger, `ServiceAccount` logon,
`RestartCount 3` / `RestartInterval 1 minute`,
`ExecutionTimeLimit = 0` (no 72-hour kill), no `--bind`/`--port`
override (so `serve.py`'s own `127.0.0.1:8000` default applies
unchanged). Full script content is already published in
`review-packages/NIKLAR-WINDOWS-MACHINE-DEPLOYMENT-AUDIT-01.md`, and
was sent to you directly as a file.

### C.11 — Nothing to land yet

Confirmed unchanged from the prior cycle: the script is **not** being
committed to niklar-stocks in this batch either. It stays a
handoff-only artifact until this first supervised run and the reboot
test both pass.

---

## D. Verification — required before this batch is considered PASS

Run these **in order**, and fill in the evidence template below as
you go — this is what gets reported back for this checkpoint to close.

### D.12 — Local origin check

```powershell
Start-ScheduledTask -TaskName "NiklarPwaServe"
curl.exe -i http://127.0.0.1:8000/index.html
```
Confirm: `200` status, and a `Cache-Control: no-store` header present.

### D.13 — Private snapshot renders correctly, no leakage

Open `http://127.0.0.1:8000/index.html` in a browser on the machine
itself. Confirm the banner reads **PRIVATE SNAPSHOT** (not DEMO), your
real tickers render, and there's no fallback to demo data. Separately,
confirm `pwa/private-data/` is still git-ignored on this machine (same
`git check-ignore -v pwa/private-data/` check used in earlier cycles)
— nothing about this batch changes that, but worth reconfirming once
after any new setup.

### D.14 — cloudflared service status

```powershell
Get-Service Cloudflared
```
Confirm status is `Running`.

### D.15 — Scheduled Task status

```powershell
Get-ScheduledTaskInfo -TaskName "NiklarPwaServe"
```
Confirm it's registered and `LastTaskResult` is `0` (success) after
the manual start in D.12.

### D.16 — Remote verification from your iPhone (cellular or non-LAN Wi-Fi)

1. Open `https://app.niklarintelligence.pro` — confirm Cloudflare
   Access intercepts first (a login page, not the PWA directly).
2. Try it in a private/incognito window or logged out — confirm
   unauthorized access does **not** reveal the PWA or any private
   data.
3. Authenticate as yourself — confirm it succeeds.
4. Confirm the real PRIVATE SNAPSHOT PWA renders after authentication,
   same as the local check in D.13.

### D.17 — Reboot survival test (if practical this cycle)

Reboot the Windows machine. After it's back up, without logging in
interactively if you can arrange it (or immediately after logging in,
before manually starting anything):
```powershell
Get-Service Cloudflared
Get-ScheduledTaskInfo -TaskName "NiklarPwaServe"
```
Confirm both recovered automatically, then repeat D.16's remote check.

**If this step isn't practical in this cycle, report that explicitly
as the one remaining UNKNOWN — do not report reboot survival as
proven without actually testing it.**

---

## Evidence report template (fill in and send back)

```
Starting/ending niklar-stocks SHA: unchanged (c24ecb6) -- no repo code change this cycle
Actual repo path used: <path, no secrets>
Resolved Python interpreter path: <output from the script>
cloudflared version: <output of cloudflared.exe --version>
Cloudflared service status: <Running/other>
Service recovery configured: <yes/no, confirm sc.exe failure ran>
Scheduled Task: <registered Y/N, LastTaskResult>
Local HTTP check (D.12): <status code, Cache-Control header value>
Private snapshot render (D.13): <PRIVATE SNAPSHOT confirmed Y/N, any leakage>
Tunnel hostname mapping: <app.niklarintelligence.pro -> 127.0.0.1:8000, confirmed Y/N -- no token>
Access policy (B.7-8): <owner-only confirmed Y/N, Everyone/Bypass present Y/N, MFA attached Y/N>
Remote iPhone test (D.16): <unauthorized blocked Y/N, auth succeeded Y/N, PWA rendered Y/N>
Reboot survival (D.17): <PASS / NOT TESTED -- do not guess>
Private data/secrets in Git: <confirmed absent, how checked>
Property Transfer / Journey Passport touched: <confirmed NO>
Remaining UNKNOWNs or blockers: <list, or "none">
```

## Self-audit against the standing 4 criteria

1. **Failure envelope**: every step in this runbook either fails
   closed (the script's own path/interpreter checks) or is presented
   as an owner-verified gate (Access policy correctness, D.16-17) that
   explicitly must not be assumed passing without a real check.
2. **Unnecessary abstraction/dependency**: none — this consolidates
   already-audited, dependency-free content; nothing new was added.
3. **Security/privacy**: the tunnel token is handled identically to
   the prior cycle's audit (one-time interactive command, never
   saved); the Access-policy verification step (B.7-8) is the
   strongest safeguard in this runbook and is called out as the
   single most security-critical check.
4. **Canonical boundary drift**: N/A — no niklar-stocks code touched;
   Property Transfer/Journey Passport untouched and unread this cycle.

## Explicit statement of what this checkpoint does NOT claim

This checkpoint does **not** claim: cloudflared is installed, a Tunnel
exists, Access is configured correctly, the Scheduled Task is
registered, the app is reachable remotely, or reboot survival works.
None of the 17 verification items in RESPONSE_SEQ 36's Section D have
been confirmed by this session, because none of them are observable or
executable from this environment. This checkpoint is the runbook the
owner needs to actually produce that evidence — the evidence itself
comes back in a follow-up cycle once the owner has run it.

## Review request

Per RESPONSE_SEQ 36's own "After first end-to-end implementation and
evidence checkpoint, STOP for ChatGPT review" instruction: this
checkpoint stops here, but not because the batch is complete — because
this session has reached the limit of what it can do without the
owner physically executing the runbook above. The next checkpoint in
this thread should carry the owner's actual evidence report, not a
second draft of the same runbook.
