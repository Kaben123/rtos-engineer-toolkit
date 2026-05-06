---
name: project-init
description: "Initialize RTOS project knowledge for this plugin. Use when user says: init rtos, initialize project, 初始化项目, 初始化 RTOS, setup rtos project, configure rtos, 配置 RTOS 项目, generate rtos knowledge, or when .claude/rtos-knowledge/ does not exist and any other plugin skill is triggered."
---

# Project Init — RTOS Knowledge Generation

Generate project-specific knowledge documents in `.claude/rtos-knowledge/` so other skills can work without hardcoding any RTOS-specific rules.

## Step 1: Ask RTOS Type

Ask the user:

> What RTOS does this project use?
> (e.g., NuttX, Zephyr, FreeRTOS, RT-Thread, ThreadX, LiteOS, RTEMS, or other)

Record the answer as `RTOS_TYPE`.

## Step 2: Confirm Project Root

Identify the current working directory as the project root. Ask the user to confirm:

> Project root: `<cwd>` — is this correct?

## Step 3: Auto-Scan Project

Based on `RTOS_TYPE` and the project directory, scan for:

| Information | How to detect |
|-------------|---------------|
| Build system | Look for Makefile, CMakeLists.txt, build.ninja, envsetup.sh, west.yml, SConstruct |
| Build commands | Infer from build scripts (e.g., `make`, `cmake --build`, `west build`, `source envsetup.sh && lunch && m`) |
| Toolchain | Look for cross-compiler prefixes in Makefiles/CMake (arm-none-eabi-, riscv64-, xtensa-) |
| Style checker | Look for checkpatch.sh, .clang-format, .astyle, .editorconfig |
| Coding style | Read .clang-format or infer from existing source (indent, line length, brace style, comment style) |
| Commit conventions | Read recent `git log --oneline -20` to detect message format pattern |
| Branch/push model | Check for `.gitreview` (Gerrit), detect remotes |

## Step 4: Present Results for Confirmation

Show the user a summary of detected information, organized by file:

```
## Detected Project Profile

- RTOS: <type>
- Build: <command>
- Toolchain: <prefix>
- Style: <rules>
- Commit format: <pattern>
- Push model: <gerrit/github/gitlab>

Does this look correct? What needs to be changed?
```

Wait for user confirmation. Apply any corrections they provide.

## Step 5: Generate Knowledge Files

Create `.claude/rtos-knowledge/` directory with the following files:

### profile.md
```markdown
# Project Profile

- RTOS: <type>
- Architecture: <arch>
- Project root: <path>
- Description: <one-line from user or inferred>
```

### build.md
```markdown
# Build

## Commands
- Full build: <command>
- Clean build: <command>
- Single module: <command if applicable>

## Output
- Build artifacts: <path>

## Notes
- <any special build requirements>
```

### coding-style.md
```markdown
# Coding Style

## Rules
- Indent: <spaces/tabs, count>
- Line length: <max chars>
- Brace style: <same-line/next-line>
- Comment style: <// or /* */>
- Naming: <convention>

## Checker
- Tool: <checkpatch/clang-format/none>
- Command: <how to run>

## Key Conventions
- <additional rules observed>
```

### toolchain.md
```markdown
# Toolchain

- Compiler prefix: <prefix>
- addr2line: <prefix>addr2line
- objdump: <prefix>objdump
- GDB: <prefix>gdb
- ELF location: <typical path to output ELF>
```

### conventions.md
```markdown
# Commit & Collaboration Conventions

## Commit Message Format
<detected format with example>

## Push Model
- Type: <Gerrit/GitHub PR/GitLab MR/direct>
- Push command: <e.g., git push origin HEAD:refs/for/main>

## Branch Strategy
- Main branch: <name>
- Feature branches: <pattern if any>

## Required Checks
- <style check, CI, tests before push>
```

## Step 6: Confirm Completion

Tell the user:

> Knowledge files generated in `.claude/rtos-knowledge/`:
> - profile.md
> - build.md
> - coding-style.md
> - toolchain.md
> - conventions.md
>
> Other plugin skills (kernel-review, commit-helper, crash-guide, rtos-workflow) will now read these files for project-specific rules.
>
> You can edit these files anytime to update the knowledge. Run this skill again to regenerate.

## Important Rules

- Keep each file under 50 lines — only essential information that skills need
- Do NOT invent or guess information — if detection fails, ask the user
- Do NOT hardcode RTOS-specific assumptions — let the scan results drive the content
- Generated files should be committed to the repo so the team shares them
