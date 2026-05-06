---
name: commit-helper
description: "Generate standardized commit messages and manage code submission workflow. Generic fallback for repos without a project-specific git-commit skill. Use when user says: commit, 提交, git commit, push, 推送, generate commit message, 生成commit message, commit msg, 写commit, or any request to create a git commit or push code for review. NOTE: If the project has a `git-commit` skill (e.g. Gerrit/Vela projects), defer to that instead."
---

# Commit Helper

Generate standardized commit messages and guide the code submission workflow.

**Scope**: This is a generic commit helper. If the project provides a dedicated `git-commit` skill (common in Gerrit/Vela/NuttX projects), that skill takes precedence over this one — it has deeper integration with project-specific tooling (checkpatch, JIRA, Change-Id management).

## Prerequisites

Read `.claude/rtos-knowledge/conventions.md` for commit message format and push model. If not found, invoke `project-init` first.

## Step 1: Gather Context and Validate State (parallel)

Run these commands simultaneously:
- `git symbolic-ref HEAD 2>/dev/null` — verify we are on a named branch (not detached HEAD)
- `git status` — verify there are changes to commit; also detect rebase/merge state
- `git diff --cached` — staged changes (primary)
- `git diff` — unstaged changes (for staging decisions)
- `git log --oneline -5 2>/dev/null` — confirm commit style (may fail on empty repo)

**Abort conditions** (inform user and stop):
- No changes exist (clean working tree)
- Repository is in rebase/merge/cherry-pick state (`git status` shows "rebase in progress", or `.git/rebase-merge`/`.git/MERGE_HEAD` exists) — tell user to resolve first
- Detached HEAD — prompt user to create a named branch: `git checkout -b <branch-name>`

**Initial commit** (no HEAD yet): `git diff --cached` still works; `git log` will fail — skip log silently and infer format from conventions.md only.

## Step 2: Determine Commit Message Format

Use the format defined in `.claude/rtos-knowledge/conventions.md`. If not available, infer from `git log` output.

Common patterns to detect:
- `<module>: <subject>` (NuttX, Linux kernel style)
- `<type>(<scope>): <subject>` (Conventional Commits)
- `[MODULE] <subject>` (bracket prefix style)
- Free-form with ticket ID

Rules (universal):
- **ASCII-only**: Commit message MUST NOT contain non-ASCII characters (no Chinese, Japanese, emoji, etc.). All text must be in English.
- Subject line: imperative mood, concise, max 72 characters
- Body (wrapped at 72 chars): explain what and why, not how
- Include issue tracker ID if the project requires it (prompt user if not provided)
- Add sign-off if the project convention requires it (detect from git log)
- Gerrit Change-Id: if amending an existing change, extract from previous commit and place as the LAST trailer line (after Signed-off-by)

### ASCII Validation

Before creating the commit, verify the message contains only printable ASCII plus whitespace:
```bash
printf '%s\n' "$COMMIT_MSG" | LC_ALL=C tr -d '\t\n\r -~' | wc -c
```
If the byte count is > 0, non-ASCII characters are present. Rewrite the message in English before proceeding.

Common violations:
- Chinese/Japanese descriptions → translate to English
- Non-ASCII punctuation (。，：「」） → replace with ASCII equivalents (. , : "")
- Smart quotes (" " ' ') → plain quotes (" ')
- Em-dash (—) → double hyphen (--)
- Emoji → remove

## Step 3: Stage Files

- Stage specific files by name (never `git add -A` or `git add .` blindly)
- **Never stage** (warn user if they request these):
  - `.env`, `.env.*` — environment secrets
  - `*.key`, `*.pem`, `*.p12`, `id_rsa*` — private keys
  - `credentials.json`, `token.json`, `*.keystore` — auth tokens
  - Large binaries (check with `git diff --cached --stat` for unexpectedly large files)
  - `.claude/` internal state files (unless explicitly project-shared like `tasks.json`)
- For RTOS projects: `*.bin`, `*.elf`, `*.hex` should NOT be staged unless the project explicitly versions firmware binaries

## Step 4: Style Check (if configured)

Read `.claude/rtos-knowledge/coding-style.md` (search from workspace root, not git sub-repo root) for the style checker command.

If a checker is specified, run it on the staged files:
```bash
<checker-command> <staged-files>
```

Fix any issues, re-stage, then proceed. If no checker is configured, skip this step.

## Step 5: Create Commit

Create the commit with the validated, ASCII-only message:
```bash
git commit -m "$(cat <<'EOF'
<formatted-message>
EOF
)"
```

For multi-line messages, use a heredoc to preserve formatting. Verify the commit succeeded (exit code 0).

## Step 6: Pre-Submit Review Gate (before push)

Before pushing, run the `pre-submit-review` skill on the committed diff:
- If **PASS**: proceed to push
- If **NEEDS FIX**: stop, show issues, wait for user to fix and re-commit
- If **WAIVABLE**: ask user whether to fix or waive

**Fallback**: If `pre-submit-review` skill is not available, skip this step and proceed directly to push (with user confirmation).

## Step 7: Push (only if user requests and review passes)

Read push model from `.claude/rtos-knowledge/conventions.md`:

| Model | Command |
|-------|---------|
| Gerrit | `git push <remote> HEAD:refs/for/<branch>` |
| GitHub/GitLab PR/MR | Push branch, then create PR/MR |
| Direct push | `git push <remote> <branch>` |

Rules:
- Confirm remote and branch with user before pushing
- For Gerrit (`refs/for/`): each push creates a new patchset — safe to repeat, not a force-push
- For direct push: never force-push without explicit user permission
- If amending a Gerrit change, preserve the existing `Change-Id` footer:
  ```bash
  # Extract from previous commit
  git log --format="%B" -1 | grep "^Change-Id:"
  ```
  Place Change-Id as the last trailer (after Signed-off-by). If `.git/hooks/commit-msg` exists and is executable, Change-Id is auto-generated for new commits — do not manually add one in that case.

## Error Recovery

| Failure | Action |
|---------|--------|
| Style check fails before first commit | Fix, re-stage, create new commit |
| Style check fails after push (Gerrit CI) | Fix, re-stage, amend (preserve Change-Id) |
| Push fails with "not a branch" | Create a named branch first |
| Push fails with "no new changes" | Commit was already submitted — inform user |
| Push fails with auth error | Inform user to check credentials |
| Push fails with hook rejection | Show hook output, do not retry with `--no-verify` |
| Commit hook fails | Fix the issue, re-stage, create NEW commit (don't amend — the original commit didn't happen) |
