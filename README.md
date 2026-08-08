# niklar-handoff

Sanitized coordination and handoff layer for AI agents (Claude, ChatGPT,
and others) working on the private Niklar research/decision-automation
system.

## Authority

This repository is a sanitized coordination layer for AI handoffs.

It is NOT the canonical Niklar repository.

Canonical implementation, proprietary data, databases, credentials,
research artifacts, trading records, and sensitive configuration remain
in private storage.

Files here may contain:
- implementation status
- architecture summaries
- canonical file references
- decisions already made
- unresolved tasks
- agent handoff notes

If this repository conflicts with the private Niklar canon, the private
canon always wins.

Authority order (per `NIKLAR_CHATGPT_CLAUDE_HANDOFF_RECOVERY_RULE.txt`,
Drive):
1. Private Niklar canonical source (Google Drive)
2. Private implementation repository (`niklar-stocks`, private)
3. This repository — coordination only
4. Conversation summaries / memory

If sources conflict, higher authority wins.

## Communication channel (frozen, 2026-08-08; routing clarified 2026-08-08)

Stated here, first, before any other sync/handoff workflow in this
file, because it governs how every other section is used. Each side
of the ChatGPT <-> Claude coordination has exactly one channel of
record, in one direction only, and the two directions are never the
same mailbox:
- **Claude -> ChatGPT / user**: this GitHub repository
  (`niklar-handoff`) ONLY. Claude publishes checkpoints, deltas,
  decisions, TODOs, and status here.
- **ChatGPT -> Claude**: the Google Drive document
  `NIKLAR_CHATGPT_TO_CLAUDE_SYNC_CURRENT` ONLY, read directly from
  Drive. This is the sole live ChatGPT -> Claude response channel.

Non-negotiable consequences of the above:
- Claude must NOT search this (or any) GitHub repository for a
  ChatGPT response. GitHub is Claude's outbound checkpoint channel,
  not ChatGPT's reply channel.
- ChatGPT must NOT publish its replies into `niklar-handoff`.
- Any ChatGPT-authored text that does appear in this repository (e.g.
  `ARCHIVED_ENGINEERING_AUTHORITY_SNAPSHOT_2026-08-08.txt`) is
  archival/audit evidence of a past Drive read only — never live,
  never the current response, never a place to look for ChatGPT's
  latest reply. See that file's own header for the same statement.

**Routing acceptance test** (a fresh Claude session should answer all
five correctly from this repo alone):
1. Where does Claude publish its checkpoint? -> GitHub `niklar-handoff`.
2. Where does Claude read ChatGPT's response? -> Google Drive
   `NIKLAR_CHATGPT_TO_CLAUDE_SYNC_CURRENT`.
3. Should Claude search GitHub for ChatGPT's response? -> NO.
4. Should ChatGPT write its response to GitHub? -> NO.
5. If a copied ChatGPT instruction exists in GitHub (e.g. the archived
   snapshot file), is it the live reply channel? -> NO.

Do not invent or use any other channel for either direction.

## Stale-canon / handoff gate

Before concluding that Niklar context is missing, outdated, or must be
reconstructed manually:
1. Inspect the current authoritative Niklar folder in Google Drive.
2. Determine whether its CURRENT/handoff/operational artifacts
   adequately cover the latest known work.
3. If Drive appears stale or there's evidence of newer work, read this
   repo's `CURRENT_HANDOFF.txt`, `DECISIONS_CURRENT.txt`,
   `TODO_CURRENT.txt`, `CHANGELOG.txt`.
4. Treat those files as evidence of newer coordination state, NOT as
   authority over the private canon.
5. Reconcile the handoff against current Drive canon before changing
   canonical rules.
6. Preserve established Niklar decisions, report templates,
   deterministic execution rules, data mappings, operational
   workflows, and audit/replay requirements unless canon explicitly
   supersedes them.
7. If this handoff describes changes not yet synchronized into Drive,
   label that state explicitly `PENDING CANONICAL RECONCILIATION`.
8. Never silently promote a handoff statement into canon.
9. Once reconciled and authorized, update Drive so it's self-sufficient
   again.
10. After meaningful work, keep this handoff current so the other
    agent can resume without reconstructing the session.

## Security

Never publish sensitive or proprietary material here:

- API keys
- tokens
- passwords
- credentials
- environment variables containing secrets
- private URLs or access tokens
- personal information
- account numbers
- private trading records
- proprietary databases
- raw private research datasets
- sensitive configuration
- anything that should remain private

Only sanitized coordination information belongs in this repository.

## Structure

- `CURRENT_HANDOFF.txt` — primary interchange file; current state snapshot
- `DECISIONS_CURRENT.txt` — decisions affecting future implementation
- `TODO_CURRENT.txt` — current execution queue
- `CHANGELOG.txt` — meaningful changes to the handoff state
- `CLAUDE_STATUS.txt` — mandatory startup control handshake (per
  NIKLAR_HARD_RULE_COMMIT_GATE, Drive), published before/after
  autonomous Claude sessions: canonical files read, drift-check
  result, and whether Claude is clear to continue autonomous
  mutations or is paused pending acknowledgement
- `CLAUDE_OVERNIGHT_AUTONOMOUS_PROMPT.txt` — durable record of the
  user's autonomous-operation operating instructions
- `ARCHIVED_ENGINEERING_AUTHORITY_SNAPSHOT_2026-08-08.txt` — an
  ARCHIVED, NON-LIVE, point-in-time sanitized copy of part of the
  Drive ChatGPT->Claude sync document, kept only as audit evidence
  (includes the scope firewall for treating
  joiedevivre02/niklar-operating-engineering-system as read-only
  project-agnostic-engineering reference only, as it stood on that
  date). Never the live reply channel — see "Communication channel"
  above.
- `snapshots/` — older handoff states, kept only when useful for
  recovery/comparison

## sync / cont / stop keywords

Standing control protocol between the user and Claude/ChatGPT on
Niklar work:
- **sync** — refresh this handoff with a concise delta (not a full
  rewrite) of what materially changed. Does not pause work. Also read
  the Drive `NIKLAR_CHATGPT_TO_CLAUDE_SYNC_CURRENT` document directly
  from Drive for the newest ChatGPT delta (see "Communication channel"
  above — that Drive document, not GitHub, is the only live ChatGPT
  reply channel), reconcile against Drive canon and this handoff, and
  publish a concise sanitized delta back here — then read the Drive
  document again for any ChatGPT response to that checkpoint.
- **cont** — continue autonomous work immediately on the
  highest-value safe, already-authorized, unblocked next increment;
  don't wait for a scheduled checkpoint. Never overrides the commit
  gate, drift kill switch, privacy rule, an active blocker, or failed
  validation.
- **stop** — halt further mutations at the next safe point, preserve
  known-good state, publish a sanitized handoff of exactly where
  execution stopped.

See `CLAUDE_OVERNIGHT_AUTONOMOUS_PROMPT.txt` for the full definition.

## Agent workflow

**At the start of a session:**
1. Read the private Niklar canon available to you.
2. Read `CURRENT_HANDOFF.txt`.
3. Treat the private canon as authoritative.
4. Check whether the handoff is stale or conflicts with canonical state.
5. Continue implementation only after resolving material conflicts.

**At the end of meaningful work:**
1. Update the private/canonical system first, when authorized and
   appropriate.
2. Produce a sanitized summary of what changed.
3. Update `CURRENT_HANDOFF.txt`.
4. Update `DECISIONS_CURRENT.txt` and `TODO_CURRENT.txt` if necessary.
5. Update `CHANGELOG.txt`.
6. Commit and push the sanitized handoff changes.

Never use this repository as justification for overriding canonical
private artifacts.
