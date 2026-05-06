---
name: task-tracker
description: "Track development tasks with status, dependencies, branches, and patch links. Use when user says: 新建任务, 创建任务, new task, create task, 任务列表, list tasks, 切换任务, switch task, 恢复任务, resume task, 任务状态, task status, 更新任务, update task, 关闭任务, close task, 当前任务, current task, show tasks, 查看任务, track task, 跟踪任务, or any request to manage development task state."
---

# Task Tracker

Track development tasks across branches and patches, enabling seamless context switching.

## Storage

Tasks are persisted in `.claude/tasks.json` at the project root. Format:

```json
{
  "version": 1,
  "tasks": [
    {
      "id": "T001",
      "title": "Short description",
      "status": "in_progress",
      "description": "Detailed requirement or context",
      "dependencies": ["T002"],
      "repos": [
        {
          "path": "relative/path/to/repo",
          "branch": "feature/task-001-xxx"
        }
      ],
      "patches": [
        {
          "url": "https://gerrit.example.com/c/project/+/12345",
          "status": "open"
        }
      ],
      "created": "2026-05-06",
      "updated": "2026-05-06"
    }
  ],
  "active_task": "T001"
}
```

## Operations

### Create Task

Trigger: "新建任务" / "create task"

1. Ask user for:
   - Title (one line)
   - Description (optional, requirement context)
   - Dependencies (optional, other task IDs)
2. Generate next task ID (T001, T002, ...)
3. Set status to `pending`
4. Write to `.claude/tasks.json`

### List Tasks

Trigger: "任务列表" / "list tasks" / "show tasks"

Display all tasks in table format:

```
| ID   | Title              | Status      | Branch          | Patch           |
|------|--------------------|-------------|-----------------|-----------------|
| T001 | Fix ISR deadlock   | in_progress | fix/isr-dead    | gerrit/+/123 ✓  |
| T002 | Add sensor driver  | pending     | —               | —               |
| T003 | Optimize mempool   | done        | opt/mempool     | gerrit/+/456 M  |
```

Patch status symbols: ✓ = merged, ○ = open/review, ✗ = abandoned/closed, — = none

### Update Task

Trigger: "更新任务" / "update task"

Updatable fields:
- `status`: pending → in_progress → review → done (or blocked)
- `description`: additional context
- `dependencies`: add/remove
- `repos[].branch`: record the working branch
- `patches[]`: add patch URL and status

### Switch Task

Trigger: "切换任务 T00X" / "switch task T00X"

This is the core workflow:

1. **Save current context**: Record the active task's current state
2. **Identify target task's branches**: Read `repos[].branch` for the target task
3. **Switch branches**: For each repo in the target task:
   - If branch exists locally: `git checkout <branch>`
   - If branch doesn't exist: inform user, offer to create
4. **Restore context**: Display target task's full info:
   - Title, description, status
   - Patch links and their status
   - Dependencies and their status
   - Recent commits on the branch (`git log --oneline -10`)
5. **Set active_task** to the target task ID

### Resume Task

Trigger: "恢复任务" / "resume task" (resumes the active task)

1. Read `active_task` from tasks.json
2. Verify branches are still checked out
3. Show task summary + recent activity
4. If branch has drifted (someone else pushed), inform user

### Current Task

Trigger: "当前任务" / "current task"

Show the active task's full detail including:
- All metadata
- Branch status (`git status` per repo)
- Patch review status (if URL provided, try to query status)

### Close Task

Trigger: "关闭任务" / "close task"

1. Set status to `done`
2. Record final patch status
3. Clear from `active_task` if it was active

## Branch Isolation Principle

Each task has its own branch(es). When switching tasks:
- The target task's branch contains ONLY commits related to that task
- This ensures no cross-contamination between tasks
- If user has uncommitted changes when switching, warn and offer to stash

## Patch Link Handling

Support both Gerrit and GitHub/GitLab:

| Platform | URL pattern | Status query |
|----------|-------------|--------------|
| Gerrit | `*/c/project/+/12345` or change-id | Use Gerrit MCP if available |
| GitHub | `*/pull/123` | Use `gh` CLI if available |
| GitLab | `*/merge_requests/123` | Use `glab` CLI if available |

If no tool is available to query status, ask user to update manually.

## Important Rules

- Never delete tasks — only mark as `done` or `abandoned`
- Never force-switch branches with uncommitted changes without user consent
- Always show context summary after switching (user should immediately know where they are)
- Task file `.claude/tasks.json` should be committed to repo for team sharing
