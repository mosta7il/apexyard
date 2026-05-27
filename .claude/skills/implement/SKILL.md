---
name: implement
description: Full Build + PR flow for the active ticket — decide gate, branch, implement, pre-push checks, commit, push, PR. Requires /start-ticket to have been run first.
argument-hint: "[optional implementation notes]"
---

# /implement — Build + PR in One Flow

Executes the Build phase (steps 2–4 of the standard workflow) for the currently active ticket:
decide gate → branch → implement → pre-push gate → commit → push → PR.

Requires `/start-ticket` to have been run first. Does **not** replace `/code-review`, `/qa-approve`,
or `/approve-merge` — those run after this skill completes.

---

## Fail-fast contract

**Every step below is a hard gate.** If a step fails or produces an ambiguous result, STOP
immediately — do not proceed to the next step. Print a clear STOPPED message that names:

1. Which step failed
2. What the exact error or blocker is
3. What the user must do to unblock it

Never silently skip a failing step. Never paper over a failure by guessing or continuing anyway.
The skill is designed to be re-run after the user fixes the blocker — each step is idempotent
enough that re-running from the beginning is safe.

---

## Step 1 — Load the active ticket

Resolve the ops root (walk up from CWD looking for `onboarding.yaml` + `apexyard.projects.yaml`).
Read the ticket marker:

```bash
source "$OPS_ROOT/.claude/hooks/_lib-read-config.sh"
source "$OPS_ROOT/.claude/hooks/_lib-portfolio-paths.sh"
session_home=$(portfolio_session_home)

# Per-project marker: $session_home/.claude/session/tickets/<project>
# Ops fallback:       $session_home/.claude/session/current-ticket
```

Parse fields: `repo`, `number`, `title`, `url`, `suggested_branch`.

**STOP if:**
- No marker file exists at either path:
  ```
  STOPPED (Step 1): No active ticket.
  Fix: run /start-ticket <N> to declare a ticket before implementing.
  ```
- Any required field (`repo`, `number`, `title`, `suggested_branch`) is missing from the marker:
  ```
  STOPPED (Step 1): Marker at <path> is missing required field: <field>.
  Fix: re-run /start-ticket <N> to rewrite a valid marker.
  ```

---

## Step 2 — Read the issue

Fetch the full issue body:

```bash
gh issue view <number> --repo <owner/repo> --json title,body,labels,state
```

**STOP if:**
- `gh` exits non-zero (network error, auth failure, issue not found):
  ```
  STOPPED (Step 2): Could not fetch issue <owner/repo>#<number>.
  Error: <exact gh error output>
  Fix: check network/auth, verify the issue exists, then re-run /implement.
  ```
- Issue `state` is `CLOSED`:
  ```
  STOPPED (Step 2): Issue #<number> is already closed.
  Fix: confirm you have the right ticket, reopen it if needed, then re-run /implement.
  ```

Read the ACs carefully — they define what "done" means and what tests must cover.

---

## Step 3 — Decide gate

Before writing any code, assess whether 2+ meaningfully different implementation approaches exist.

**Run `/decide` when:**
- The ticket body describes a problem but leaves the implementation approach open
- Two or more architectural choices would lead to different file structures, dependencies, or patterns
- A new library, design pattern, or data-model change is involved

**Skip `/decide` when:**
- The ticket body already prescribes the implementation (e.g. "add X to function Y in file Z")
- The change is a single clearly-scoped addition with one obvious path
- A prior AgDR already covers this decision

If `/decide` is needed: run it inline (same session), produce the AgDR, then continue.
If skipped: state in one sentence why the approach is unambiguous before proceeding.

**STOP if:**
- `/decide` is needed but cannot produce a clear recommendation (conflicting constraints,
  missing information, external dependencies not yet resolved):
  ```
  STOPPED (Step 3): Cannot proceed without a decision on <topic>.
  The decide flow surfaced conflicting options with no clear winner.
  Fix: resolve the open question (linked in the AgDR draft), then re-run /implement.
  ```

---

## Step 4 — Create the branch

Use the `suggested_branch` from the marker. If already on that branch, skip checkout.

```bash
git checkout -b <suggested_branch>
```

Branch format: `{type}/GH-{N}-{slug}` per `.claude/rules/git-conventions.md`.

**STOP if:**
- The branch already exists **locally** and is not the current branch:
  ```
  STOPPED (Step 4): Branch <suggested_branch> already exists locally.
  Fix: run `git checkout <suggested_branch>` manually if you want to resume,
       or delete it with `git branch -d <suggested_branch>` to start fresh.
  ```
- The branch already exists **on the remote**:
  ```
  STOPPED (Step 4): Branch <suggested_branch> already exists on origin.
  Fix: decide whether to resume that branch or start fresh. Do NOT force-push.
       Check `gh pr list` to see if a PR is already open for this branch.
  ```
- `git checkout -b` fails for any other reason (dirty tree, detached HEAD, etc.):
  ```
  STOPPED (Step 4): Could not create branch <suggested_branch>.
  Error: <exact git error>
  Fix: resolve the git state issue, then re-run /implement.
  ```

---

## Step 5 — Implement

Read the relevant source files before editing. Follow the project's existing patterns.

- Edit only files needed to satisfy the ACs
- Write or update tests alongside the implementation — do not defer tests to a follow-up
- No comments unless the WHY is genuinely non-obvious
- No features beyond what the ticket requires

**Test-first for the failing case** (mandatory for bug fixes and features, optional for chores):
write the test that captures the AC, confirm it fails, then implement until it passes.

**STOP if:**
- A required source file cannot be found and its location is genuinely unclear:
  ```
  STOPPED (Step 5): Cannot locate <file> needed for this change.
  Fix: identify the correct file path, then re-run /implement.
  ```
- The implementation reveals the ticket's scope is larger than the ACs describe
  (e.g. a dependency needs changing that touches unrelated code):
  ```
  STOPPED (Step 5): Scope creep detected — <description of the unexpected dependency>.
  Fix: clarify the ticket scope with the user before continuing. Do not implement
       beyond what the ACs require.
  ```

---

## Step 6 — Pre-push gate

Run all checks locally before pushing. **All must pass — no exceptions.**

Detect the project's package manager and run its standard gate:

```bash
# pnpm (Next.js / Node)
pnpm lint && pnpm typecheck && pnpm test && pnpm build

# npm
npm run lint && npm run typecheck && npm test && npm run build

# Python
ruff check . && mypy . && pytest

# Go
go vet ./... && go test ./...
```

**STOP if any check fails:**
```
STOPPED (Step 6): Pre-push gate failed at <lint|typecheck|test|build>.
Error output:
  <exact failure output>
Fix: resolve the failure above, then re-run /implement.
     Do NOT push. Do NOT create the PR.
```

Do not attempt to suppress or work around the failure (e.g. `--no-verify`, `// eslint-disable`).
Fix the root cause.

---

## Step 7 — Commit

Stage **specific files only** (never `git add -A` or `git add .`):

```bash
git add <file1> <file2> ...
```

Commit message format:

```
type(#N): subject line

- Detailed change 1 (what + why)
- Detailed change 2

Closes #N

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
```

`type` must match the branch type: `feat`, `fix`, `chore`, `test`, `docs`, `refactor`, `perf`.

**STOP if:**
- Nothing was staged (working tree clean — implementation produced no changes):
  ```
  STOPPED (Step 7): Nothing to commit — no files were changed.
  Fix: verify the implementation in Step 5 actually modified files, then re-run /implement.
  ```
- A pre-commit hook rejects the commit:
  ```
  STOPPED (Step 7): Pre-commit hook blocked the commit.
  Hook output:
    <exact hook output>
  Fix: resolve the hook failure (do NOT use --no-verify), then re-run /implement.
  ```

---

## Step 8 — Push

```bash
git push -u origin <branch>
```

**STOP if:**
- Push is rejected for any reason (remote ahead, auth failure, protected branch):
  ```
  STOPPED (Step 8): Push failed.
  Error: <exact git push output>
  Fix: resolve the push error. Do NOT force-push without explicit user instruction.
  ```

---

## Step 9 — Create the PR

```bash
gh pr create --repo <owner/repo> \
  --title "type(#N): description" \
  --body "..."
```

### PR body template

```markdown
## Summary
- **<what changed>** — <why it matters / what problem it solves>
- **<what changed>** — <why it matters>

## Testing
1. <Step to verify the change works>
2. <Step to verify edge cases>

Closes #N

---

## Glossary
| Term | Definition |
|------|------------|
| <term> | <definition relevant to this PR> |
```

**Summary bullets:** every bullet must answer *what changed* AND *why it matters*.
No label-only bullets ("State fix", "Add tests") — those force reviewers into diff archaeology.

**Glossary is mandatory.** At minimum define the most domain-specific term in the diff.

**STOP if:**
- `gh pr create` exits non-zero:
  ```
  STOPPED (Step 9): PR creation failed.
  Error: <exact gh error output>
  Fix: resolve the error (auth, duplicate PR, branch not pushed, etc.), then re-run /implement.
  ```
- A pre-PR hook (e.g. `validate-pr-create.sh`) rejects the title or body:
  ```
  STOPPED (Step 9): PR validation hook blocked creation.
  Hook output:
    <exact hook output>
  Fix: correct the PR title/body format, then re-run /implement.
  ```

---

## Step 10 — Hand off to code review

After `gh pr create` succeeds, the `auto-code-review.sh` PostToolUse hook fires automatically.
Invoke Rex immediately:

```
/code-review <pr-number>
```

Rex must approve before the merge gate allows `/approve-merge`.

---

## What this skill does NOT do

| Step | Skill |
|------|-------|
| Declare the active ticket | `/start-ticket` |
| Code review | `/code-review` (Rex) |
| QA verification | `/qa-approve` |
| Merge | `/approve-merge` |

---

*Part of [ApexYard](https://github.com/me2resh/apexyard) — multi-project SDLC framework for Claude Code · MIT.*
