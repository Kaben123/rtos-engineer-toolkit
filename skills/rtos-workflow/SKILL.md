---
name: rtos-workflow
description: "RTOS kernel engineer daily workflow router. Use when user mentions: requirements analysis, design review, code implementation, driver development, build firmware, debug crash, memory leak, performance profiling, patch review, code review, commit, push, cherry-pick, rebase, 需求分析, 方案设计, 写驱动, 编译, 调试, 崩溃分析, 内存泄漏, 性能分析, 代码审查, 提交代码, or any embedded/RTOS development task."
---

# RTOS Daily Workflow Router

Route the user's task to the correct approach based on their intent.

## Prerequisites

Read `.claude/rtos-knowledge/profile.md` to identify the current RTOS type and project context. If this file does not exist, invoke the `project-init` skill first.

## Identify Intent

Classify the user's request into one of five workflows:

| Workflow | Trigger signals |
|----------|----------------|
| Requirements & Design | issue tracker, design doc, requirement, architecture, 需求, 方案 |
| Code Implementation | write driver, new feature, fix bug, build, compile, config, 写代码, 编译 |
| Debug & Analysis | crash, coredump, memory leak, performance, trace, hang, 崩溃, 内存, 性能 |
| Patch Review | review, audit, diff, check style, 审查, 评审 |
| Commit & Collaborate | commit, push, cherry-pick, rebase, 提交, 推送 |

## Workflow: Requirements & Design

1. Read the requirement source (issue tracker / doc / user description)
2. Identify affected code modules in the current repo
3. Output:
   - Task breakdown (each item < 1 day)
   - Risk & dependency list
   - Acceptance checklist
4. Ask user to confirm before proceeding

## Workflow: Code Implementation

1. Read `.claude/rtos-knowledge/build.md` for build commands
2. Read `.claude/rtos-knowledge/coding-style.md` for style rules
3. Clarify target: which subsystem, which board/target
4. Follow existing coding patterns in the repo
5. Mandatory checks before declaring done:
   - Compiles without error (using build command from knowledge)
   - Passes style checker (from knowledge)
   - Matches existing code patterns
6. Never generate APIs that don't exist — always grep to verify

## Workflow: Debug & Analysis

1. Read `.claude/rtos-knowledge/toolchain.md` for debug tools
2. Follow the 3-hypothesis method:
   - Parse the crash/log/trace data
   - Locate source code lines using project toolchain
   - Propose exactly 3 most likely root cause hypotheses
   - For each, state how to verify (specific command or check)
3. Ask user which hypothesis to investigate first

## Workflow: Patch Review

1. Read `.claude/rtos-knowledge/coding-style.md` for style rules
2. Review the diff against these dimensions (report by file + line + severity):

| Severity | Meaning |
|----------|---------|
| Block | Must fix before merge — correctness, security, data loss |
| Major | Should fix — resource leak, missing error path, race condition |
| Minor | Improvement — naming, redundancy, readability |
| Nit | Style only — formatting, comment wording |

Kernel-specific checks:
- ISR context: no blocking calls in interrupt handlers
- Synchronization: correct primitive for the scenario
- Resource lifecycle: every alloc path has a corresponding free path
- Error paths: all fallible ops checked, resources freed on error

## Workflow: Commit & Collaborate

1. Read `.claude/rtos-knowledge/conventions.md` for commit format and push model
2. Stage relevant files (never stage secrets or binaries)
3. Generate commit message following the project's convention
4. Run `pre-submit-review` as a quality gate before push
5. If review passes: push using the project's push model (from knowledge)
6. If review finds blocking issues: fix first, then re-review
7. Never force-push; never push directly without user confirmation

Note: Use `task-tracker` to record the patch link after successful push.

## Quality Guardrails (Always Apply)

- **Verify before trust**: Claude-generated APIs may not exist — grep to confirm
- **No secrets in context**: reject if user pastes keys/tokens
- **Every change must be verifiable**: answer "how do I verify this is correct?"
- **Human decides architecture**: propose, don't decide
- **Build → Static check → Test → Device verification**: this pipeline is non-negotiable
