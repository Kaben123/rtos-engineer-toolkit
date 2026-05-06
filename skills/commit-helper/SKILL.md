---
name: commit-helper
description: "Generate standardized commit messages and manage code submission workflow. Use when user says: commit, 提交, git commit, push, 推送, generate commit message, 生成commit message, commit msg, 写commit, or any request to create a git commit or push code for review."
---

# Commit Helper

Generate standardized commit messages and guide the code submission workflow.

## Prerequisites

Read `.claude/rtos-knowledge/conventions.md` for commit message format and push model. If not found, invoke `project-init` first.

## Step 1: Gather Context (parallel)

Run these commands simultaneously:
- `git status` — verify there are changes to commit
- `git diff HEAD` — understand what changed
- `git log --oneline -5` — confirm commit style matches knowledge

If no changes exist, inform user and stop.

## Step 2: Determine Commit Message Format

Use the format defined in `.claude/rtos-knowledge/conventions.md`. If not available, infer from `git log` output.

Common patterns to detect:
- `<module>: <subject>` (NuttX, Linux kernel style)
- `<type>(<scope>): <subject>` (Conventional Commits)
- `[MODULE] <subject>` (bracket prefix style)
- Free-form with ticket ID

Rules (universal):
- Subject line: imperative mood, concise
- Body: explain what and why, not how
- Include issue tracker ID if the project requires it
- Add sign-off if the project convention requires it (detect from git log)

## Step 3: Style Check (if configured)

Read `.claude/rtos-knowledge/coding-style.md` for the style checker command.

If a checker is specified:
```bash
# Run the checker specified in knowledge
<checker-command> <staged-files-or-diff>
```

Fix any issues before proceeding. If no checker is specified, skip this step.

## Step 4: Stage and Commit

- Stage specific files (never `git add -A` blindly)
- Never stage: `.env`, credentials, large binaries, prebuilt artifacts
- Create commit with the formatted message

## Step 5: Pre-Submit Review Gate (before push)

Before pushing, run the `pre-submit-review` skill on the committed diff:
- If **PASS**: proceed to push
- If **NEEDS FIX**: stop, show issues, wait for user to fix and re-commit
- If **WAIVABLE**: ask user whether to fix or waive

## Step 6: Push (only if user requests and review passes)

Read push model from `.claude/rtos-knowledge/conventions.md`:

| Model | Command |
|-------|---------|
| Gerrit | `git push <remote> HEAD:refs/for/<branch>` |
| GitHub/GitLab PR/MR | Push branch, then create PR/MR |
| Direct push | `git push <remote> <branch>` |

Rules:
- Never force-push without explicit user permission
- If amending a Gerrit change, preserve the existing `Change-Id` footer
- Confirm remote and branch with user before pushing

## Error Recovery

- If style check fails: fix issues, re-stage, create new commit (don't amend previous)
- If push fails with "not a branch": create a named branch first
- If push fails with "no new changes": commit was already submitted
- If push fails with auth error: inform user to check credentials
