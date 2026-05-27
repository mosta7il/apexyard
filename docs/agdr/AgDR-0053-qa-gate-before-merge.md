---
id: AgDR-0053
timestamp: 2026-05-27T00:00:00Z
agent: claude
model: claude-sonnet-4-6
trigger: user-prompt
status: executed
---

# QA approval gate — pre-merge, structured marker, framework-wide

> In the context of the apexyard merge gate, facing QA verification happening post-merge (defects reaching main before QA catches them), I decided to add a third required marker (`<pr>-qa.approved`) between Rex and CEO in `block-unreviewed-merge.sh` to achieve a framework-wide pre-merge QA gate, accepting that in-flight PRs will be blocked until `/qa-approve` is invoked.

## Context

- `block-unreviewed-merge.sh` enforced two markers: Rex (bare SHA) + CEO (structured key/value)
- QA phase was Phase 5 in the SDLC — after merge, verifying ACs on main
- User policy change: QA must sign off **before** merge, not after, so defects never land on main
- Framework-wide: applies to all managed projects, no per-project opt-out

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Third marker in merge gate (chosen)** — `<pr>-qa.approved` written by `/qa-approve` skill, checked between Rex and CEO in `block-unreviewed-merge.sh` | Mechanical enforcement at merge boundary; same pattern as CEO marker; clear error message on missing/stale QA approval | In-flight PRs blocked until `/qa-approve` invoked; adds one human step per PR |
| **Post-merge ticket-closure gate** — QA approval required to close the ticket (Done state), not to merge | No disruption to in-flight PRs; lower friction | Defects still reach main before QA catches them; doesn't satisfy the stated policy |
| **GitHub branch protection required reviewer** — add QA team as required reviewer via GitHub settings | Native GitHub enforcement | Requires GitHub team configuration; not self-contained in the framework; doesn't integrate with marker-based audit trail |

## Decision

Chosen: **Third marker in merge gate**, because it satisfies the explicit policy ("before merge") mechanically, reuses the proven marker pattern, and keeps enforcement inside the framework without GitHub org-level configuration. The in-flight PR disruption is a one-time rollout concern.

## Consequences

- `block-unreviewed-merge.sh`: three marker checks (Rex → QA → CEO)
- New skill `.claude/skills/qa-approve/SKILL.md` mirrors `/approve-merge` but writes `approved_by=qa-engineer` and does not run the merge itself
- SDLC Phase 5 moves to pre-merge; Phase 6 becomes Deploy
- `auto-code-review.sh` PostToolUse hook reminds about QA requirement after PR creation

## Artifacts

- Refs me2resh/apexyard#420
