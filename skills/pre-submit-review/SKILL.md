---
name: pre-submit-review
description: "Automated code review before patch submission. Use when user says: 提交前检查, pre-submit review, submit review, 提交检查, review before push, 自动审查, auto review, check before commit, 检查后提交, submit with review, push with review, or when commit-helper skill is about to push a patch."
---

# Pre-Submit Review

Automatically review code quality before patch submission. Runs as a gate — issues must be resolved or explicitly waived before push.

## Prerequisites

Read `.claude/rtos-knowledge/coding-style.md` and `.claude/rtos-knowledge/conventions.md`. If not found, invoke `project-init` first.

## Trigger

This skill activates:
1. When user explicitly requests pre-submit review
2. When `commit-helper` skill is about to push (integrated gate)

## Step 1: Collect the Diff

Determine what will be submitted:

```bash
# If commit exists (reviewing before push)
git diff <base-branch>...HEAD

# If changes are staged (reviewing before commit)
git diff --cached
```

Ask user to confirm which commits/changes to review if ambiguous.

## Step 2: Run All Review Dimensions

### 2.1 Functional Correctness

- Does the logic match the stated intent (from commit message or task description)?
- Are edge cases handled (null, zero, overflow, empty, boundary)?
- Are return values from all fallible calls checked?
- Does error handling clean up properly (reverse allocation order)?
- Are there off-by-one errors in loops or array indexing?

### 2.2 Functional Compatibility

- Does this change break existing API contracts (signature, behavior, return semantics)?
- Are callers of modified functions still correct?
- If a struct/enum is modified, are all users updated?
- Binary compatibility: does this change ABI if relevant?
- Configuration: are new Kconfig/CMake options properly gated with defaults?

### 2.3 Concurrency & Safety (kernel code)

- No blocking calls in ISR context
- Correct synchronization primitive for the scenario
- No race conditions on shared data
- Lock ordering consistent (no deadlock potential)
- Resource lifecycle: no use-after-free, no double-free

### 2.4 Code Style

Read checker from `.claude/rtos-knowledge/coding-style.md`:
- Run the configured style checker if available
- If no automated checker, manually verify:
  - Indentation, line length, brace style (per project rules)
  - Naming conventions
  - Comment style
  - No trailing whitespace, no mixed tabs/spaces

### 2.5 Commit Message

Read format from `.claude/rtos-knowledge/conventions.md`:
- Matches required format (e.g., `<module>: <subject>`)
- Subject line is imperative mood, concise
- Body explains WHY, not just WHAT
- Issue tracker ID present if required
- Sign-off present if required
- No typos in subject line
- Line wrap at configured width

## Step 3: Generate Report

Output a structured report:

```
## Pre-Submit Review Report

### Summary
- Files changed: N
- Insertions: +X, Deletions: -Y
- Issues found: A Block / B Major / C Minor / D Nit

### Blocking Issues (must fix)
1. [file:line] <description>
   **Fix:** <suggestion>

### Major Issues (should fix)
1. [file:line] <description>
   **Fix:** <suggestion>

### Minor Issues (consider fixing)
...

### Style Issues
...

### Commit Message
- Format: ✓/✗
- Content: ✓/✗
- Issue ID: ✓/✗
- Sign-off: ✓/✗ (if required)

### Verdict: PASS / NEEDS FIX / WAIVABLE
```

## Step 4: Decision Gate

Based on the report:

| Verdict | Action |
|---------|--------|
| **PASS** | No blocking/major issues. Inform user they can proceed with push. |
| **NEEDS FIX** | Blocking issues exist. List them. Do NOT proceed until fixed. |
| **WAIVABLE** | Only major/minor issues. Ask user: fix now, or waive and submit? |

If user waives issues, add a note to the commit message or task record.

## Step 5: Proceed or Iterate

- If PASS or user waives: hand off to `commit-helper` for push
- If NEEDS FIX: offer to auto-fix what's possible (style, simple corrections), show remaining manual fixes
- After fixes: re-run the review on the updated diff

## Auto-Fix Capabilities

The following can be auto-fixed (with user confirmation):
- Trailing whitespace
- Indentation (if style checker available)
- Missing sign-off line
- Line length violations (safe reformatting only)
- Import/include ordering (if tool available)

Things that require manual fix:
- Logic errors
- Missing error handling
- Synchronization issues
- API compatibility breaks

## Integration with commit-helper

When `commit-helper` is about to push, it should invoke this skill as a gate:

```
commit-helper Step 5 (Push):
  → Run pre-submit-review
  → If PASS/WAIVED: proceed with push
  → If NEEDS FIX: stop, show issues, wait for user
```

## Important Rules

- Never skip blocking issues — they exist to prevent broken code from landing
- Auto-fix only with explicit user consent
- If project has no style checker configured, do best-effort manual review
- Report must be actionable: every issue needs a file:line and a fix suggestion
- Keep review focused on the diff — don't flag pre-existing issues in untouched code
