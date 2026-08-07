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
- `snapshots/` — older handoff states, kept only when useful for
  recovery/comparison

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
