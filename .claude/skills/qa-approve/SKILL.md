---
name: qa-approve
description: Record QA engineer sign-off on a PR after verifying all acceptance criteria. Writes a structured marker that the merge gate requires before gh pr merge is allowed.
---

# /qa-approve — Record QA Approval

Writes a structured `<pr>-qa.approved` marker at `.claude/session/reviews/`. The merge gate
(`block-unreviewed-merge.sh`) requires this marker — alongside Rex and CEO markers — before
`gh pr merge` is allowed. This skill does **not** merge; CEO `/approve-merge` is still required.

Decision record: AgDR-0053-qa-gate-before-merge.

## The one rule you must not break

**INVOKE THIS SKILL ONLY AFTER YOU HAVE ACTUALLY VERIFIED THE ACCEPTANCE CRITERIA.**

Valid invocation: QA engineer has deployed the PR branch to staging (or run it locally), stepped
through every acceptance criterion, confirmed all pass, and is now recording that sign-off.

Invalid invocation: rubber-stamping without testing, approving from a code read alone, or
approving speculatively "so the merge can proceed".

## Process

### 1. Parse the PR number

Extract from the argument (e.g. `42`, `owner/repo#42`). If empty:
- Try `gh pr view --json number --jq '.number'` from the current branch
- If still ambiguous, ask the user which PR

### 2. Verify the PR state

```bash
gh pr view <pr> --repo <owner/repo> --json state,isDraft,mergeable,headRefOid
```

- `state` must be `OPEN` — refuse on `MERGED`, `CLOSED`, `DRAFT`
- `mergeable` must be `MERGEABLE` or `UNKNOWN` — refuse on `CONFLICTING`
- Capture `headRefOid` — this is the SHA the marker must record (NOT local `git rev-parse HEAD`)

### 3. Resolve the ops root

Markers live in the ops fork, not the project workspace clone:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
HOOK_DIR="$(git rev-parse --show-toplevel 2>/dev/null)/.claude/hooks"
OPS_ROOT=""
if [ -f "$HOOK_DIR/_lib-ops-root.sh" ]; then
  . "$HOOK_DIR/_lib-ops-root.sh"
  OPS_ROOT=$(resolve_ops_root "$REPO_ROOT")
fi
MARKER_HOME="${OPS_ROOT:-${REPO_ROOT:-.}}"
if [ -n "$OPS_ROOT" ] && [ -f "$HOOK_DIR/_lib-read-config.sh" ] && [ -f "$HOOK_DIR/_lib-portfolio-paths.sh" ]; then
  . "$HOOK_DIR/_lib-read-config.sh"
  . "$HOOK_DIR/_lib-portfolio-paths.sh"
  _sh=$(portfolio_session_home 2>/dev/null)
  [ -n "$_sh" ] && MARKER_HOME="$_sh"
fi
```

### 4. Verify the Rex marker exists at the PR's HEAD

QA stamps code that has already been code-reviewed:

```bash
REX="$MARKER_HOME/.claude/session/reviews/<pr>-rex.approved"
REX_SHA=$(tr -d '[:space:]' < "$REX" 2>/dev/null)
```

If Rex marker is missing or its SHA doesn't match `headRefOid`, stop and tell the user to
invoke the code-reviewer first.

### 5. Write the structured QA marker

```bash
mkdir -p "$MARKER_HOME/.claude/session/reviews"
ts=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
notes=$(echo "<any notes the QA engineer provided>" | tr '\n' ' ' | tr -d '"`$\\' | cut -c1-200)

cat > "$MARKER_HOME/.claude/session/reviews/<pr>-qa.approved" <<EOF
sha=<headRefOid>
approved_by=qa-engineer
approved_at=${ts}
skill_version=1
approval_summary="${notes}"
EOF
```

**Required fields** (the merge gate validates all three):

| Field | Value | Why |
|-------|-------|-----|
| `sha` | 40-char hex PR HEAD from GitHub | Binds approval to a specific commit |
| `approved_by` | `qa-engineer` (literal) | Distinguishes skill-written marker from a forged one |
| `skill_version` | `1` | Format version; gate requires >= 1 |

**Optional fields** (stored, not validated):

| Field | Use |
|-------|-----|
| `approved_at` | Audit timestamp |
| `approval_summary` | QA notes (ACs passed, environment tested, caveats) |

### 6. Report

```
✓ QA approval recorded for PR #<pr> at commit <sha:0:7>.
Marker: <MARKER_HOME>/.claude/session/reviews/<pr>-qa.approved

Next: CEO approval required to merge.
  /approve-merge <pr>
```

## Notes

- The marker is gitignored (`.claude/session/` is in `.gitignore`) — session state, not code.
- Running `/qa-approve <pr>` again on the same PR overwrites the marker (useful after a rebase).
- New commits after the marker is written invalidate it — the gate blocks because `sha=` no longer
  matches PR HEAD. Re-invoke `/qa-approve` after any force-push or rebase.
- If only the CEO marker exists (QA was skipped from an older session), the gate will block at the
  QA check. Invoke `/qa-approve` first, then the CEO can run `/approve-merge` again.

---

*Part of [ApexYard](https://github.com/me2resh/apexyard) — multi-project SDLC framework for Claude Code · MIT.*
