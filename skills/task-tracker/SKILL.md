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
          "branch": "task/T001-short-slug",
          "upstream": "vela/dev-system"
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

### Field Reference

| Field | Description |
|-------|-------------|
| `repos[].path` | Relative path from project root to the git repo |
| `repos[].branch` | Task-specific local branch name |
| `repos[].upstream` | Remote tracking branch (e.g. `vela/dev-system`, `origin/main`) |
| `active_task` | ID of the currently active task, or `null` if no task is active |

### Task ID Generation

Generate the next ID by finding the maximum existing numeric suffix and incrementing by 1. Example: if T003 is the highest, next is T004.

## Branch Management

### Naming Convention

```
task/<TASK_ID>-<short-slug>
```

Examples: `task/T001-mcal-pwm`, `task/T003-dfx-poll`

### Branch Creation

When a task first needs a working branch in a repo:

1. **Detect the upstream branch** (in priority order):
   - If a Gerrit patch URL is associated, extract the target branch from the change metadata (`branch` field)
   - If the user specifies explicitly, use that
   - Otherwise, ask the user — do NOT blindly use the current branch's upstream (it may belong to a different task)
2. **Create with tracking**:
   ```bash
   git checkout -b task/<ID>-<slug> --track <upstream>
   ```
3. **Record in tasks.json**: Store both `branch` and `upstream` in `repos[]`

### Picking a Patch to Local Branch

When working on a Gerrit patch locally:

1. **Always fetch the latest patchset**: Query Gerrit for the current patchset number (via MCP or `git ls-remote`), then fetch it:
   ```bash
   git fetch <remote> refs/changes/XX/<change-num>/<latest-patchset>
   ```
2. **Cherry-pick onto the task branch**:
   ```bash
   git checkout task/<ID>-<slug>
   git cherry-pick FETCH_HEAD
   ```
   **If conflicts arise:**
   - Inform the user which files conflict
   - Ask whether to resolve interactively or abort (`git cherry-pick --abort`)
   - If aborting, record in the task that the patch could not be cleanly applied
   - Never silently leave the repo in a conflicted state
3. **For amend-and-push workflows** (fixing an existing patch):
   ```bash
   git fetch <remote> refs/changes/XX/<change-num>/<latest-patchset>
   git checkout FETCH_HEAD
   # make fixes, amend commit (preserve Change-Id)
   git push <remote> HEAD:refs/for/<target-branch>
   git checkout task/<ID>-<slug>
   ```
   **If the push fails:**
   - Do NOT immediately checkout the task branch (the amended commit would be lost)
   - Inform the user of the failure and the current detached HEAD state
   - Offer: retry push, create a temporary branch to preserve work, or abort

### Determining Latest Patchset

Use one of these methods (in priority order):
1. **Gerrit MCP** (`gerrit_get_change`): Response contains the current revision's `_number`
2. **git ls-remote**: `git ls-remote <remote> "refs/changes/XX/<change-num>/*"` — pick the highest number. Note: some Gerrit instances restrict wildcard queries; if empty result, fall back to other methods.
3. **Ask the user** if neither method is available

## Operations

### Create Task

Trigger: "新建任务" / "create task"

1. Ask user for:
   - Title (one line)
   - Description (optional, requirement context)
   - Dependencies (optional, other task IDs)
   - Repo path(s) and upstream branch (optional — can be added later when work begins)
2. Generate next task ID (find max existing + 1)
3. Set status to `pending`
4. Write to `.claude/tasks.json`

### List Tasks

Trigger: "任务列表" / "list tasks" / "show tasks"

Display all tasks in table format:

```
| ID   | Title              | Status      | Branch               | Patch           |
|------|--------------------|-------------|----------------------|-----------------|
| T001 | Fix ISR deadlock   | in_progress | task/T001-isr-dead   | gerrit/+/123 ○  |
| T002 | Add sensor driver  | pending     | —                    | —               |
| T003 | Optimize mempool   | done        | task/T003-mempool    | gerrit/+/456 ✓  |
```

Patch status symbols: ✓ = merged, ○ = open/review, ✗ = abandoned/closed, — = none

### Update Task

Trigger: "更新任务" / "update task"

Updatable fields:
- `status`: pending → in_progress → review → done (or blocked)
- `description`: additional context
- `dependencies`: add/remove
- `repos[].branch`: record the working branch
- `repos[].upstream`: record the remote tracking branch
- `patches[]`: add patch URL and status

### Switch Task

Trigger: "切换任务 T00X" / "switch task T00X"

This is the core workflow:

1. **Save current context**: Record the active task's current state
2. **Check for uncommitted changes** (across ALL repos of the current task):
   - For each repo in the current active task, run `git status`
   - If dirty, offer to stash with message: `"task/<current-task-id>: WIP"`
   - If user declines to stash and checkout would conflict, abort the switch
3. **Identify target task's branches**: Read `repos[].branch` for the target task
4. **Switch branches**: For each repo in the target task:
   - If branch exists locally: `git checkout <branch>`
   - If branch doesn't exist but `upstream` is set: create it with `git checkout -b <branch> --track <upstream>`
   - If neither: inform user, ask for upstream branch to create
5. **Restore context**: Display target task's full info:
   - Title, description, status
   - Patch links and their status
   - Dependencies and their status
   - Recent commits on the branch (`git log --oneline -10`)
6. **Set active_task** to the target task ID

### Resume Task

Trigger: "恢复任务" / "resume task" (resumes the active task)

1. Read `active_task` from tasks.json (`null` means no active task — prompt user to switch)
2. Verify branches are still checked out
3. Show task summary + recent activity
4. If branch has drifted from upstream, inform user and offer to rebase
5. Check if the task has stashed work (`git stash list | grep "task/<ID>"`); if found, offer to pop

### Current Task

Trigger: "当前任务" / "current task"

Show the active task's full detail including:
- All metadata
- Branch status (`git status` per repo)
- Upstream sync status (`git log --oneline <upstream>..<branch>`)
- Patch review status (if URL provided, try to query status)

### Close Task

Trigger: "关闭任务" / "close task"

1. Set status to `done`
2. Record final patch status
3. Set `active_task` to `null` if this was the active task
4. Branches are preserved (not deleted) — user can clean up manually later

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

### Gerrit-Specific: Fetching Patches

When fetching a Gerrit change for local work:
1. Extract change number from URL (e.g. `7533703` from `+/7533703`)
2. Query current patchset via Gerrit MCP or ls-remote
3. Fetch: `git fetch <remote> refs/changes/<last-2-digits>/<change-num>/<patchset>`
4. The `<last-2-digits>` is the last 2 digits of the change number (e.g. `03` for `7533703`)

## Important Rules

- Never delete tasks — only mark as `done` or `abandoned`
- Never force-switch branches with uncommitted changes without user consent
- Never silently leave a repo in a conflicted or detached HEAD state without informing the user
- Always show context summary after switching (user should immediately know where they are)
- Always fetch the **latest** patchset when picking a patch to local — never use a stale patchset number
- Branch upstream is not hardcoded — detect it from Gerrit change metadata or ask the user
- Task file `.claude/tasks.json` should be committed to repo for team sharing (consider `.gitignore` for `active_task` if multi-developer conflicts arise)
