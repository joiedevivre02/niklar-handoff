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

Authority order:
1. Private Niklar canonical source (Google Drive)
2. Private implementation repository (`niklar-stocks`, private)
3. This repository — coordination only

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
