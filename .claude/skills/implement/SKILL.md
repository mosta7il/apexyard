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

## Step 1 — Load the active ticket

Resolve the ops root (walk up from CWD looking for `onboarding.yaml` + `apexyard.projects.yaml`).
Read the ticket marker:

```bash
source "$OPS_ROOT/.claude/hooks/_lib-read-config.sh"
source "$OPS_ROOT/.claude/hooks/_lib-portfolio-paths.sh"
session_home=$(portfolio_session_home)

# Per-project marker takes precedence over the ops fallback.
# Determine the project name from the registry by matching the ticket's repo.
# Per-project marker path: $session_home/.claude/session/tickets/<project>
# Ops fallback: $session_home/.claude/session/current-ticket
```

Parse fields: `repo`, `number`, `title`, `url`, `suggested_branch`.

**If no marker exists → STOP.** Print:
```
No active ticket. Run /start-ticket <N> first.
```

---

## Step 2 — Read the issue

Fetch the full issue body to understand requirements and acceptance criteria:

```bash
gh issue view <number> --repo <owner/repo> --json title,body,labels
```

Read the ACs carefully — they define what "done" means for this ticket and what tests must cover.

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

If `/decide` is needed: run it now (inline — same session), produce the AgDR, then continue.
If skipped: note in one sentence why the approach is unambiguous before proceeding.

---

## Step 4 — Create the branch

Use the `suggested_branch` from the marker. If the current branch already matches, skip.

```bash
git checkout -b <suggested_branch>
```

Branch format: `{type}/GH-{N}-{slug}` per `.claude/rules/git-conventions.md`.

---

## Step 5 — Implement

Read the relevant source files before editing. Follow the project's existing patterns.

- Edit only files needed to satisfy the ACs
- Write or update tests alongside the implementation — do not defer tests to a follow-up
- No comments unless the WHY is genuinely non-obvious
- No features beyond what the ticket requires

**Test first for the failing case** (if applicable): write the test that captures the AC, confirm it fails, then implement until it passes. This is optional for chores/refactors but mandatory for bug fixes and features.

---

## Step 6 — Pre-push gate

Run all checks locally before pushing. **All must pass — no exceptions.**

Detect the project's package manager and run its standard gate. Common patterns:

```bash
# pnpm (Next.js / Node projects)
pnpm lint && pnpm typecheck && pnpm test && pnpm build

# npm
npm run lint && npm run typecheck && npm test && npm run build

# Python
ruff check . && mypy . && pytest

# Go
go vet ./... && go test ./...
```

If any check fails → fix it before proceeding. Do not push red.

---

## Step 7 — Commit

Stage **specific files only** (never `git add -A` or `git add .`):

```bash
git add <file1> <file2> ...
```

Commit message format (`.claude/rules/git-conventions.md`):

```
type(#N): subject line

- Detailed change 1 (what + why)
- Detailed change 2

Closes #N

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
```

`type` must match the branch type: `feat`, `fix`, `chore`, `test`, `docs`, `refactor`, `perf`.

---

## Step 8 — Push

```bash
git push -u origin <branch>
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

**Rules for Summary bullets:**
- Every bullet must answer *what changed* AND *why it matters*
- No label-only bullets ("State fix", "Add tests") — those force reviewers into diff archaeology
- Minimum 1 bullet, maximum 6

**Glossary is mandatory.** If the PR introduces no new terms, add at least one definition for the
most domain-specific term already in the diff. Reviewers should never have to ask "what is X?"

---

## Step 10 — Hand off to code review

After `gh pr create` succeeds, the `auto-code-review.sh` PostToolUse hook fires and reminds you
to invoke Rex. Do it immediately:

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

## Error conditions

| Condition | Action |
|-----------|--------|
| No active ticket marker | STOP — print `No active ticket. Run /start-ticket <N> first.` |
| Pre-push gate fails | Fix the failure. Do not push. Do not create the PR. |
| Branch already exists remotely | Stop and ask the user — don't force-push |
| Ticket is already closed | Warn and ask for confirmation before proceeding |

---

*Part of [ApexYard](https://github.com/me2resh/apexyard) — multi-project SDLC framework for Claude Code · MIT.*
