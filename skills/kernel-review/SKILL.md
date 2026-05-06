---
name: kernel-review
description: "RTOS kernel code review specialist. Use when user asks to review kernel code, audit driver implementation, check concurrency safety, review patch, 审查内核代码, 检查并发安全, review driver, audit ISR safety, check synchronization, or any request to review embedded/RTOS C code for correctness and safety."
---

# Kernel Code Review

Perform systematic review of RTOS kernel / driver code with focus on correctness and safety.

## Prerequisites

Read `.claude/rtos-knowledge/coding-style.md` for project-specific style rules. If not found, invoke `project-init` first.

## Review Dimensions

For each file in the diff, check ALL of the following:

### 1. ISR Context Safety

- No blocking calls in interrupt handlers (typical blockers: malloc, mutex_lock, sem_wait, sleep, any call that may schedule)
- No unbounded loops in ISR context
- Proper use of critical sections for shared state
- DMA buffers aligned and cache-coherent if applicable

### 2. Synchronization Correctness

| Scenario | Correct approach |
|----------|-----------------|
| ISR ↔ task shared data | Disable interrupts or spinlock with IRQ save |
| Task ↔ task (short hold, no blocking) | Spinlock (SMP) or scheduler lock (UP) |
| Task ↔ task (may block) | Mutex or semaphore |
| Priority-sensitive | Priority-inheritance mutex |
| One-shot signal | Binary semaphore or event flags |
| Producer-consumer | Message queue or ring buffer |

Red flags:
- Mutex operations in ISR context
- Nested lock ordering inconsistency (deadlock risk)
- Missing unlock on error return path
- Semaphore used where mutex semantics are needed (ownership)

### 3. Memory Safety

- Every allocation has a corresponding free on all paths (including error paths)
- No use-after-free (especially in async callbacks and deferred work)
- Buffer size validated before copy operations
- Stack allocation size reasonable (check project convention)
- No variable-length arrays in kernel code

### 4. Error Handling

- All fallible operations checked (return value, error code)
- Resources freed in reverse allocation order on error
- Consistent cleanup pattern (goto-cleanup or early-return with proper teardown)
- No silent error swallowing without documented reason

### 5. Multi-core / SMP (if applicable)

- Per-CPU data accessed with preemption disabled
- CPU-local results not stale across schedule points
- Proper memory barriers for lock-free shared data
- No assumption about execution order across cores

### 6. Project Coding Style

Read rules from `.claude/rtos-knowledge/coding-style.md` and check compliance. If no knowledge file exists, infer from surrounding code in the same file.

## Output Format

```
## <filename>

### Line <N> [Block/Major/Minor/Nit]
<description of issue>
**Suggestion:** <fix or improvement>
```

## Final Summary

After per-file review, provide:
1. Overall risk assessment (Safe / Low Risk / Medium Risk / High Risk)
2. Top 3 most critical issues
3. Suggested test scenarios to verify correctness
