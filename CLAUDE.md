# RTOS Engineer Toolkit — Contributor Guide

## Plugin 定位

本 plugin 封装 RTOS 内核工程师的**通用**工作方法论，不绑定任何特定 RTOS。项目特定的规则通过 `.claude/rtos-knowledge/` 知识文档提供。

## 核心原则

1. **Plugin 不含 RTOS 特有规则** — 所有 RTOS 特定内容（coding style、build 命令、toolchain）由 `project-init` skill 生成到项目本地
2. **Skill 运行时读取知识文档** — 不硬编码路径、命令、规范
3. **检测失败则询问** — 自动扫描推断不出来的信息，必须问用户，不能编造

## 贡献规则

### 什么适合放进来

- 通用的 RTOS 开发方法论（如三假设法排查 crash）
- 不依赖特定 RTOS 的审查维度（如 ISR 安全、内存生命周期）
- 通用的工作流程（如 commit 前检查、review 输出格式）

### 什么不该放进来

- 特定 RTOS 的 API 名称或规范（放知识文档）
- 特定项目的路径或构建命令（放知识文档）
- 特定工具链的固定路径（放知识文档）
- 未经验证的实验性流程

## Skill 编写规范

1. 每个 skill 一个目录：`skills/<name>/SKILL.md`
2. frontmatter 必须包含 `name` 和 `description`
3. `description` 写清楚所有可能的触发词（中英文）
4. Skill 正文中引用项目规则时，写 "Read `.claude/rtos-knowledge/<file>.md`"
5. 如果知识文件不存在，引导用户执行 `project-init`
6. 不硬编码任何 RTOS 特有内容

## 知识文档约定

`project-init` 生成的文件：
- `profile.md` — RTOS 类型、架构（skill: rtos-workflow）
- `build.md` — 构建命令（skill: rtos-workflow, commit-helper）
- `coding-style.md` — 编码规范（skill: kernel-review, commit-helper）
- `toolchain.md` — 调试工具（skill: crash-guide）
- `conventions.md` — 提交和协作规范（skill: commit-helper）

每个文件 < 50 行，只含 skill 运行必需的信息。

## 版本管理

- `package.json` 和 `.claude-plugin/plugin.json` 的 version 保持一致
- 每次有意义的更新递增版本号（semver）

## 测试方式

```bash
claude --plugin-dir /path/to/this/plugin/
```

验证：
1. 会话启动时看到质量红线提醒
2. 说 "初始化 RTOS 项目" → 触发 project-init
3. 说 "review 代码" → 触发 kernel-review（读取知识文档）
4. 说 "提交" → 触发 commit-helper → 自动调用 pre-submit-review gate
5. 贴 crash 日志 → 触发 crash-guide（读取知识文档）
6. 说 "新建任务" → 触发 task-tracker
7. 说 "切换任务 T002" → task-tracker 切换分支并恢复上下文
8. 说 "提交前检查" → 触发 pre-submit-review（独立使用）
