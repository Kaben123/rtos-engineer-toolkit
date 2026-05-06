---
name: crash-guide
description: "Guide systematic crash/fault analysis for RTOS systems. Use when user mentions: crash, coredump, hardfault, assert, panic, memory corruption, use-after-free, stack overflow, watchdog timeout, deadlock, hang, OOM, signal 11, bus error, 崩溃, 死机, 卡死, 内存踩踏, 看门狗超时, or pastes a crash log/dump/backtrace."
---

# Crash Analysis Guide

Systematic approach to RTOS crash/fault root cause analysis.

## Prerequisites

Read `.claude/rtos-knowledge/toolchain.md` for debug tool paths (addr2line, GDB, objdump, ELF location). If not found, invoke `project-init` first.

## Step 1: Classify the Fault Type

From the crash log, identify which category:

| Fault Type | Indicators |
|-----------|------------|
| Hard Fault | Fault status registers, fault address |
| Assert / Panic | `ASSERT`, `panic`, `abort` in log |
| Stack Overflow | SP near stack boundary, stack guard triggered |
| Use-After-Free | Access to freed memory, memory sanitizer report |
| Heap Corruption | malloc/free crash, double-free, guard pattern broken |
| Deadlock | Two+ tasks blocked on each other's locks |
| Watchdog Timeout | WDT reset, no explicit fault |
| OOM | Allocation failure, out-of-memory error |
| Bus Error | Unaligned access, invalid address region |
| Null Pointer | Fault address at 0x00000000 or near-zero |

## Step 2: Extract Key Information

From the crash dump, extract:
1. **Faulting address** (PC / LR / fault address)
2. **Call stack** (backtrace with addresses)
3. **Faulting task** (name, PID, priority, state)
4. **Register dump** (SP, LR, PC, status register)
5. **Memory state** (heap free, stack remaining)

Resolve addresses using toolchain from `.claude/rtos-knowledge/toolchain.md`:
```bash
<toolchain-prefix>addr2line -e <elf-path> -f -C <address>
```

Or with GDB:
```bash
<toolchain-prefix>gdb <elf-path> -batch -ex "info line *<address>"
```

## Step 3: Three-Hypothesis Method

Generate exactly 3 root cause hypotheses ranked by likelihood.

For each hypothesis provide:
- **Hypothesis**: One-line description
- **Evidence**: What in the log supports this
- **Counter-evidence**: What might disprove this
- **Verification**: Specific command or check to confirm/deny

## Step 4: Triage Decision

Based on analysis, recommend ONE of:
- **Immediate fix**: Root cause is clear, propose the patch
- **Need more data**: Specify what to enable/collect (sanitizer, trace, memory dump)
- **Need reproduction**: Suggest minimal reproduction steps
- **Escalate**: Multiple subsystems involved, need broader investigation

## Common Patterns (Quick Reference)

| Pattern | Typical Root Cause |
|---------|-------------------|
| Crash in free/dealloc | Double-free or heap corruption |
| Crash at 0x00000000 | Null function pointer call |
| Crash after deferred/async callback | Object freed before callback fires |
| Two tasks deadlocked | Lock ordering violation |
| Periodic watchdog reset | Priority inversion starving critical task |
| Crash only under load | Race condition in shared resource |
| Crash in memcpy/memset | Buffer overflow or unaligned access |
| Assert in semaphore/mutex op | Synchronization object destroyed while in use |

## Important Reminders

- Assert/panic is a **symptom**, not the root cause — dig deeper
- Always check: was the faulting code recently changed? (`git log --oneline <file>`)
- Verify your fix addresses the root cause, not just the crash location
- If toolchain info is missing, ask the user rather than guessing paths
