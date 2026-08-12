# Checkpoint: PWA-OWNER-SAFE-AUTOUPDATE-01 (RESPONSE_SEQ 39, sub-batch 3 of 3 — FINAL)

Responds to ChatGPT's authorization (RESPONSE_SEQ 39) of the third and
final of three independent, separately-revertible sub-batches from
`PWA-OWNER-DECISION-COCKPIT-01-AUDIT`. Sub-batch C: a safe Windows
auto-update mechanism, applying the STRICT NO-RESET resolution
RESPONSE_SEQ 39 confirmed. Scanned the diff for secrets and the
owner's real ticker symbols before publishing — clean, no matches.
**Per RESPONSE_SEQ 39's own "stop after the third for ChatGPT review"
instruction, this is where this session stops.**

## Commit

`f71fbd6756e40984f16d5da70915e61f9786d99a` (niklar-stocks main),
parent `7b68d74cf4c2a12655d3acb5e4f420d53e8ef180`.

## What changed

- **`ops/safe_auto_update.py`** (new, committed — portable Python,
  tested here): `run_once()` orchestrates the five fail-closed steps
  from the accepted audit design (section 7), unchanged:
  1. Dirty working tree (`git status --porcelain` non-empty) → abort,
     touch nothing. Never `git stash`/discard.
  2. `git fetch origin <branch>` fails → abort, leave the deployment
     untouched.
  3. Local HEAD already equals `origin/<branch>` → no-op.
  4. Fast-forward check (`git merge-base --is-ancestor`) — treats ANY
     non-zero exit (not just "definitely not an ancestor") as unsafe
     to proceed, fail-closed on an ambiguous git result too.
  5. `git merge --ff-only origin/<branch>` — the **only** mutation
     this module ever performs.

  Restart decision: `git diff --name-only <old> <new>` checked for
  literal `pwa/serve.py` — restarts the `NiklarPwaServe` Scheduled
  Task only then, exactly matching the audit's own finding that
  `serve.py` has zero imports elsewhere in the repo.

  **STRICT NO-RESET applied**: on a failed restart command or a failed
  post-restart health check, `run_once()` returns a distinct outcome
  (`UPDATED_RESTART_COMMAND_FAILED` / `UPDATED_RESTARTED_UNHEALTHY`)
  and does **nothing further** — no automated rollback, no `git
  reset`, ever. The fast-forwarded commit stays on disk; a human
  resolves it. This was verified behaviorally (see Test delta) AND
  structurally — see `StrictNoResetSourceAuditTests` below.

  The one Windows-specific call, `restart_service()` (`Stop-
  ScheduledTask`/`Start-ScheduledTask` via `powershell.exe`), is
  isolated behind an injectable `runner` parameter specifically so
  every other function — the git-orchestration logic, which carries
  the real correctness risk — is testable against real local git
  repositories on any platform, including this repo's own Linux CI.
  Health check reuses the exact HTTP-200-plus-`Cache-Control:
  no-store` assertion already used in `pwa/smoke_test.py` and the
  Windows deployment runbook.

- **`tests/test_safe_auto_update.py`** (new, 34 tests) — see Test
  delta below.

- **`ops/README.md`**: updated intro to honestly disclose this is the
  first `ops/` module doing real network I/O (`git fetch` to GitHub,
  an HTTP health check to `localhost`) — while still no credentials of
  its own and no market/broker/research-data connectivity, this is a
  genuine, narrow expansion of the directory's originally-stated
  scope, stated plainly rather than silently redefined. New module
  documentation bullet added.

- **`pwa/README.md`**: new "Keeping a long-running deployment up to
  date" subsection, explicitly noting this is optional (manual
  `git pull` remains fully supported and is still the fresh-clone
  default) and pointing to the handoff-only registration script below.

## The Windows Scheduled Task registration script (NOT committed — handoff-only, per the established script-landing decision)

Confirmed the precedent explicitly before writing this: the prior
Cloudflare/PWA deployment cycle's own review package
(`NIKLAR-REMOTE-ACCESS-CLOUDFLARE-IMPLEMENTATION-01-runbook.md`, "C.11
— Nothing to land yet") states outright that
`Register-NiklarPwaStartupTask.ps1` is **not** committed to
niklar-stocks and "stays a handoff-only artifact until this first
supervised run and the reboot test both pass." Sub-batch C's own
registration script follows the identical rule — it is **not**
committed to niklar-stocks. Full content below, and delivered directly
to the owner via file transfer, exactly as the prior script was:

```powershell
<#
.SYNOPSIS
  Registers a Windows Scheduled Task that periodically runs
  ops/safe_auto_update.py against your niklar-stocks checkout, so
  approved pushes to main reach this machine without you manually
  running `git pull`.

.DESCRIPTION
  Batch PWA-OWNER-SAFE-AUTOUPDATE-01 (RESPONSE_SEQ 39, sub-batch 3 of
  3). This script is NOT committed to the niklar-stocks repository --
  per the same "script-landing decision" already established for
  Register-NiklarPwaStartupTask.ps1, it stays a handoff-only artifact
  until a first supervised run on your real machine proves it. The
  Python logic it invokes (ops/safe_auto_update.py) IS committed and
  has a full test suite exercising real git repositories -- this
  script is only a thin Windows Task Scheduler wrapper around it.

  What the underlying Python script does on every run (see its own
  module docstring for the full design): checks the working tree is
  clean (aborts, touches nothing, if not); fetches origin/main; if
  already up to date, does nothing; if local HEAD is not a strict
  ancestor of origin/main (history has diverged), aborts without
  merging/rebasing/resetting/forcing; otherwise fast-forwards
  (`git merge --ff-only`) -- the ONLY mutation it ever performs;
  restarts the NiklarPwaServe task ONLY if pwa/serve.py itself changed
  in that update; runs a health check (HTTP 200 + Cache-Control:
  no-store on http://127.0.0.1:8000/index.html) after any restart.
  STRICT NO-RESET (RESPONSE_SEQ 39's resolved judgment call): if the
  post-restart health check fails, it does NOT attempt any automated
  rollback -- it fails loudly and stops, leaving the Scheduled Task's
  own "last run result" showing a real problem for you to look at.

  This task's own -Force flag below is Task Scheduler's ordinary
  "overwrite an existing task registration of the same name" flag --
  unrelated to, and not a violation of, the git-level STRICT NO-RESET
  rule the Python script itself follows. No git `--force` of any kind
  is ever used by the underlying script (see its own AST-audited test:
  tests/test_safe_auto_update.py's StrictNoResetSourceAuditTests).

  NOT DONE BY THIS SCRIPT OR THE PYTHON IT RUNS: no Cloudflare Tunnel/
  Access/DNS configuration, no credential storage of any kind (git
  authentication reuses whatever credential helper your normal `git
  pull` already relies on -- nothing new is introduced), no changes to
  pwa/private-data/ (.gitignore'd, never part of any commit -- a
  fast-forward merge cannot touch it by construction).

.NOTES
  Registered under YOUR OWN Windows logon (matching NiklarPwaServe's
  already-proven, actually-deployed configuration -- NOT SYSTEM, per
  the same Microsoft Store Python interpreter reachability issue that
  forced that same correction there). Because this runs under your own
  logon rather than SYSTEM/AtStartup, registering it should NOT require
  an elevated ("Run as Administrator") PowerShell window -- unlike the
  original NiklarPwaServe runbook step, which was written for the
  SYSTEM/AtStartup design before that correction. If registration
  fails with an access-denied error in a normal window, try an
  elevated one as a fallback, but it should not be necessary.

  This script has NOT been executed against your real machine by this
  session -- there is no access to it from here. Everything below is
  reviewed and tested (the Python logic, via tests/test_safe_auto_update.py's
  34 tests against real local git repositories and a real local HTTP
  server), but genuinely UNKNOWN until you run it: whether
  Stop-ScheduledTask/Start-ScheduledTask against NiklarPwaServe
  actually requires elevation on your machine, and whether the
  15-minute repeating trigger behaves as expected across a sleep/wake
  cycle. Do not report this as fully proven until you've watched it
  fire at least once.
#>

# ---- EDIT THIS before running ------------------------------------
$RepoPath = "D:\Niklar\niklar-stocks"
# --------------------------------------------------------------------

$TaskName = "NiklarSafeAutoUpdate"
$ServiceTaskName = "NiklarPwaServe"

if (-not (Test-Path $RepoPath)) {
    throw "RepoPath '$RepoPath' does not exist. Edit `$RepoPath above and retry."
}

$ScriptPath = Join-Path $RepoPath "ops\safe_auto_update.py"
if (-not (Test-Path $ScriptPath)) {
    throw "ops\safe_auto_update.py not found under RepoPath = '$RepoPath'. Fix `$RepoPath above and retry."
}

# Resolve the Python interpreter the same way
# Register-NiklarPwaStartupTask.ps1 already does (via the `py`
# launcher), so this task runs against the exact same interpreter
# NiklarPwaServe already proved reachable under this logon -- not a
# second, unverified interpreter resolution.
try {
    $PythonPath = (& py -3 -c "import sys; print(sys.executable)").Trim()
} catch {
    throw "Could not resolve the Python interpreter via the 'py' launcher. Run 'py -3 -c ""import sys; print(sys.executable)""' manually to diagnose, or hardcode `$PythonPath below and remove this block."
}
if (-not (Test-Path $PythonPath)) {
    throw "Resolved Python interpreter path '$PythonPath' does not exist. Something is wrong with the 'py' launcher resolution -- fix before continuing."
}
Write-Host "Using Python interpreter: $PythonPath"

$Action = New-ScheduledTaskAction `
    -Execute $PythonPath `
    -Argument "-m ops.safe_auto_update --repo-root `"$RepoPath`" --service-task-name `"$ServiceTaskName`"" `
    -WorkingDirectory $RepoPath

# Two triggers: once at logon (so an update check happens promptly
# after you sign in, same as NiklarPwaServe itself starting), plus a
# repeating trigger every 15 minutes indefinitely (the cadence the
# accepted audit design recommended -- a webhook would need a new
# public endpoint on this machine, explicitly out of scope).
$LogonTrigger = New-ScheduledTaskTrigger -AtLogOn
$RepeatingTrigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Minutes 15) `
    -RepetitionDuration ([TimeSpan]::MaxValue)

# ExecutionTimeLimit here is intentionally SHORT (10 minutes), unlike
# NiklarPwaServe's ExecutionTimeLimit=Zero -- that task runs forever by
# design (a long-lived HTTP server); this one is meant to run to
# completion quickly and exit every 15 minutes. A short limit means a
# hung run (e.g. a stuck network call) gets killed before it could pile
# up against the next scheduled run. MultipleInstances=IgnoreNew is the
# same protection from the other direction: if a run is still going
# when the next trigger fires, skip the new one rather than running two
# at once.
$Settings = New-ScheduledTaskSettingsSet `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 10) `
    -MultipleInstances IgnoreNew `
    -StartWhenAvailable `
    -DontStopOnIdleEnd

# Owner-logon context, matching NiklarPwaServe's own actually-deployed
# (not the originally-audited SYSTEM/AtStartup) configuration -- edit
# -UserId if your Windows username differs from $env:USERNAME.
$Principal = New-ScheduledTaskPrincipal `
    -UserId $env:USERNAME `
    -LogonType Interactive `
    -RunLevel Limited

Register-ScheduledTask `
    -TaskName $TaskName `
    -Action $Action `
    -Trigger @($LogonTrigger, $RepeatingTrigger) `
    -Settings $Settings `
    -Principal $Principal `
    -Description "Niklar PWA safe auto-update: fast-forward-only git pull + conditional NiklarPwaServe restart. See ops/safe_auto_update.py in the repo for the full design." `
    -Force

Write-Host ""
Write-Host "Registered Scheduled Task '$TaskName'."
Write-Host "To run it once immediately (recommended, to prove it works before trusting the 15-minute schedule):"
Write-Host "  Start-ScheduledTask -TaskName '$TaskName'"
Write-Host "Then check the result:"
Write-Host "  (Get-ScheduledTaskInfo -TaskName '$TaskName').LastTaskResult   # 0 = success"
Write-Host ""
Write-Host "NOT YET VERIFIED on this machine -- confirm all of these before considering this batch done:"
Write-Host "  1. A manual Start-ScheduledTask run completes with LastTaskResult = 0 when already up to date."
Write-Host "  2. Pushing a small, non-serve.py change to main gets picked up within 15 minutes (or via a manual"
Write-Host "     Start-ScheduledTask run) with NO NiklarPwaServe restart."
Write-Host "  3. Pushing a change to pwa/serve.py itself triggers a NiklarPwaServe restart, and the app is still"
Write-Host "     reachable afterward (curl -i http://127.0.0.1:8000/index.html -> 200, Cache-Control: no-store)."
Write-Host "  4. Whether Stop-ScheduledTask/Start-ScheduledTask against NiklarPwaServe needed elevation on this"
Write-Host "     machine specifically (see .NOTES above)."
```

## Explicitly NOT done (capability boundary, disclosed not glossed over)

This session has no access to the owner's Windows machine — the same
capability boundary already disclosed for the Cloudflare/PWA
deployment work. This batch does **not** claim: that the registration
script runs without error on the real machine; that
`Stop-ScheduledTask`/`Start-ScheduledTask` against `NiklarPwaServe`
works without elevation; that the 15-minute repeating trigger survives
a real sleep/wake cycle; or that a real end-to-end update-and-restart
has ever actually happened. Everything claimed below is either (a)
verified by this session's own tests against real local git
repositories and a real local HTTP server, or (b) explicitly marked
UNKNOWN pending the owner's own supervised run — the same two-tier
distinction already used throughout this session's Cloudflare/Windows
work.

## Test delta

- `unittest`: 390/390 (356 → 390, 34 new tests in
  `tests/test_safe_auto_update.py`, against REAL local git
  repositories — a synthetic "origin" plus a cloned "local checkout"
  — not mocked git calls):
  - `IsWorktreeCleanTests` / `FetchAndAncestryTests` /
    `ChangedFilesAndRestartDecisionTests`: each of the five
    fail-closed decision functions tested directly, including that an
    unknown/erroring `git merge-base` call is treated as "not safe,"
    not silently permissive.
  - `RestartServiceTests`: the injected-runner behavior (success/
    failure, correct task-name targeting) — real `powershell.exe`
    doesn't exist on this repo's own Linux test environment, this is
    the one deliberately-mocked boundary (see module docstring).
  - `HealthCheckTests`: a real local `HTTPServer` proving the
    200-plus-`Cache-Control:-no-store` assertion, the negative cases
    (wrong status, missing/wrong header), and that connection-refused
    degrades to `False` rather than crashing the caller.
  - `RunOnceEndToEndTests`: full end-to-end scenarios — up-to-date
    no-op; dirty worktree aborts leaving the uncommitted edit's content
    byte-for-byte unchanged; fetch failure aborts with local HEAD
    unchanged; **diverged history aborts with zero merge attempt**
    (local HEAD provably unchanged, confirmed distinct from origin's
    new HEAD); fast-forward with no `serve.py` change updates without
    invoking the restart runner at all; fast-forward with a `serve.py`
    change restarts and health-checks; **a failed restart command, and
    separately a failed post-restart health check, each leave the
    fast-forwarded commit in place** — the concrete behavioral proof of
    STRICT NO-RESET, not merely an absence-of-a-call assertion.
  - `CliMainTests`: exit-code mapping (0 for up-to-date/successful
    update, 1 for dirty-worktree/diverged-history) via the real `main()`
    entrypoint.
  - `StrictNoResetSourceAuditTests`: parses this module's own AST and
    extracts every literal argument list passed to `_run_git(...)`,
    asserting none contains `reset`/`rebase`/`stash`/`clean`/`push`/
    `--force`/`-f`/`--hard`/`--mixed`/`--soft`, and that the module's
    one `git merge` call site always includes `--ff-only`. A
    **structural** proof, independent of whether any given behavioral
    test happens to exercise every code path — a future edit adding a
    forbidden call trips this immediately.
- `mypy`: 0 issues, 23 source files (22 → 23).
- `pwa/smoke_test.py`: PASS, unaffected (no PWA-facing file touched
  this sub-batch).
- `node --check`: clean, unaffected (no JS touched this sub-batch —
  this is deployment tooling only).
- Manual: ran the real CLI (`python3 -m ops.safe_auto_update
  --repo-root <path>`) against a freshly-created real local git
  checkout/origin pair, confirming the exact documented output format
  and a `0` exit code for a genuine fast-forward update.

A deliberate bug caught and fixed during this batch's own
verification, not left in the delivered test suite: the local HTTP
server helper in `HealthCheckTests` initially registered
`addCleanup(server.shutdown)` before `addCleanup(thread.join)` —
`unittest`'s `addCleanup` runs LIFO, so `join()` (registered second)
ran first and deadlocked waiting for a `shutdown()` that hadn't
happened yet. Caught because the full suite visibly hung rather than
completing; fixed by consolidating both calls (plus `server_close()`,
to avoid a `ResourceWarning` for an unclosed socket) into one ordered
cleanup function, matching `tests/test_mobile_snapshot_export.py`'s
own established `_shutdown_server()` pattern. Not a defect in the
module under test — a defect in the test's own setup, caught before
this checkpoint, not after.

## Hosted CI: still BLOCKED, same infrastructure signature — unchanged, not re-investigated at length again

The `ai-qa` workflow run for this commit failed identically to the
signature already reported for sub-batches A and B (job never starts,
zero billable minutes, no log content). Same persistent condition,
not a new or worse one. Still flagged for the owner to check the
niklar-stocks repo's GitHub billing/Actions settings — **this remains
unresolved across all three sub-batches of this checkpoint sequence**,
and is the one concrete, owner-actionable item this whole authorized
implementation run is leaving open.

## Self-audit against the standing 4 criteria

1. **Failure envelope**: fails closed at every named point (dirty
   worktree, fetch failure, diverged history, merge failure, restart
   failure, health-check failure) — each returns a distinct,
   human-readable outcome rather than a generic exception, and none
   silently continues past a failure into a state that looks like
   success.
2. **Unnecessary abstraction/dependency**: none. Stdlib only
   (`subprocess`, `urllib`, `dataclasses`, `argparse`; `ast` in the
   test file only, for the structural audit). The one OS-specific call
   is isolated behind a plain injectable parameter, not a new
   abstraction layer.
3. **Security/privacy**: no new credential storage — git auth reuses
   whatever credential helper the machine's own `git` already has
   configured (unchanged by this batch). No Cloudflare Tunnel/Access/
   DNS state touched anywhere. `pwa/private-data/` is untouchable by a
   fast-forward merge by construction (`.gitignore`d, never committed).
4. **Canonical boundary drift**: `ops/niklar_stocks_ops_v4.py`,
   `ops/mobile_contract.py`, and every `pwa/*.html|js|css` file — zero
   changes, confirmed via diff. This sub-batch is deployment tooling
   only; no UI, contract, or canonical logic touched.

## This is the final sub-batch — session stops here per RESPONSE_SEQ 39

All three sub-batches ChatGPT authorized in RESPONSE_SEQ 39 are now
shipped:
- Sub-batch A (`PWA-OWNER-DECISION-COCKPIT-01A`, commit `8b8596f`) —
  mobile-first cockpit UX.
- Sub-batch B (`PWA-OWNER-DECISION-COCKPIT-01B-GTC`, commit `7b68d74`)
  — Daily GTC Plan first-class view.
- Sub-batch C (`PWA-OWNER-SAFE-AUTOUPDATE-01`, commit `f71fbd6`) —
  safe Windows auto-update mechanism (this checkpoint).

Local tests passed exhaustively at every step (390/390 unittest, 0
mypy issues, smoke test PASS, real headless-browser and real-git-repo
verification throughout); none of the six named STOP conditions were
hit at any point. The one open, disclosed, owner-actionable item
carried across all three checkpoints is the hosted GitHub Actions CI
blocker (see above) — not a code defect, not self-resolved, not
silently dropped.

## Review request

Per RESPONSE_SEQ 39's own "stop after the third for ChatGPT review"
instruction: this session stops here, awaiting ChatGPT's review of
all three sub-batches together — in particular, independent
confirmation that STRICT NO-RESET was actually implemented as
resolved (not just described), and awareness that the Windows
Scheduled Task registration script for this sub-batch (like the one
before it) remains unexecuted against the owner's real machine and
needs the owner's own supervised run before this batch can be
considered fully proven.
