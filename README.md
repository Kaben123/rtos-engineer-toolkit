# RTOS Engineer Toolkit — Claude Code Plugin

面向 RTOS 内核工程师的通用 Claude Code 插件。不绑定任何特定 RTOS，通过知识文档驱动工作流。

## 工作原理

```
┌─────────────────────────────────────────────────┐
│  Plugin (通用，不含 RTOS 特有规则)                │
│  ┌───────────┐ ┌──────────────┐ ┌────────────┐  │
│  │ Skills    │ │ Agent        │ │ Hooks      │  │
│  │ (通用流程) │ │ (通用审查)    │ │ (质量红线)  │  │
│  └─────┬─────┘ └──────┬───────┘ └────────────┘  │
│        │               │                         │
│        ▼               ▼                         │
│  ┌─────────────────────────────────────┐         │
│  │ .claude/rtos-knowledge/ (项目级)     │         │
│  │ profile.md | build.md | coding-     │         │
│  │ style.md | toolchain.md |           │         │
│  │ conventions.md                      │         │
│  └─────────────────────────────────────┘         │
└─────────────────────────────────────────────────┘
```

- **Plugin** 只包含通用工作流程和方法论
- **知识文档** 由 `project-init` skill 在具体项目中生成，记录该项目的 RTOS 类型、构建命令、编码规范等
- **Skills/Agent** 运行时读取知识文档，获取项目特定规则

## 功能概览

| 组件 | 说明 |
|------|------|
| `skills/project-init` | 初始化项目：扫描 → 生成 `.claude/rtos-knowledge/` |
| `skills/rtos-workflow` | 日常工作流路由器（需求→实现→调试→Review→提交） |
| `skills/kernel-review` | 内核代码审查（ISR/同步/内存/SMP/风格） |
| `skills/commit-helper` | 标准化提交（读取项目提交规范） |
| `skills/crash-guide` | Crash 排查指导（三假设法） |
| `skills/task-tracker` | 任务跟踪（状态/分支/patch/上下文切换） |
| `skills/pre-submit-review` | 提交前自动审查（正确性/兼容性/风格/commit msg） |
| `agents/kernel-reviewer` | 内核审阅子 Agent |
| `hooks/session-start` | 会话启动注入质量红线提醒 |

## 安装方式

### 方式 1：Marketplace 安装（推荐）

本仓库自包含 marketplace 元数据，可直接作为 marketplace 注册并安装：

```bash
# 1. 注册 marketplace（GitHub 仓库方式）
claude plugin marketplace add Kaben123/rtos-engineer-toolkit

# 2. 安装插件
claude plugin install rtos-engineer-toolkit@rtos-engineer-toolkit-marketplace

# 3. 验证
claude plugin list
```

安装成功后显示：

```
  ❯ rtos-engineer-toolkit@rtos-engineer-toolkit-marketplace
    Version: 1.1.0
    Status: ✔ enabled
```

### 方式 2：开发模式（本地调试）

克隆仓库后直接指定路径启动，无需注册 marketplace：

```bash
git clone https://github.com/Kaben123/rtos-engineer-toolkit.git
claude --plugin-dir ./rtos-engineer-toolkit
```

### 卸载

```bash
claude plugin uninstall rtos-engineer-toolkit@rtos-engineer-toolkit-marketplace
claude plugin marketplace remove rtos-engineer-toolkit-marketplace
```

### 更新

```bash
claude plugin update rtos-engineer-toolkit@rtos-engineer-toolkit-marketplace
```

## 首次使用

安装 plugin 后，在项目中首次使用时：

```
初始化 RTOS 项目
```

Plugin 会引导你：
1. 确认 RTOS 类型（NuttX / Zephyr / FreeRTOS / RT-Thread / 其他）
2. 自动扫描项目目录，推断构建命令、工具链、编码规范
3. 展示检测结果让你确认和修正
4. 生成 `.claude/rtos-knowledge/` 知识文档

之后所有 skill 自动读取这些知识文档工作。

## 支持的 RTOS

Plugin 本身不限制 RTOS 类型。只要通过 `project-init` 生成了知识文档，即可用于：

- NuttX / OpenVela
- Zephyr
- FreeRTOS
- RT-Thread
- ThreadX / Azure RTOS
- LiteOS
- RTEMS
- 其他任何 RTOS

## 知识文档结构

```
<项目>/.claude/rtos-knowledge/
├── profile.md        # RTOS 类型、架构、项目描述
├── build.md          # 构建命令、输出路径
├── coding-style.md   # 编码规范、检查工具
├── toolchain.md      # 工具链前缀、调试工具路径
└── conventions.md    # 提交格式、push 模型、分支策略
```

每个文件 < 50 行，只记录 skill 运行所需的关键信息。

## 持续迭代

### 补充经验到 plugin

新增通用工作流 → 在 `skills/` 下加目录：
```
skills/<new-skill>/SKILL.md
```

### 补充项目特定知识

直接编辑 `.claude/rtos-knowledge/` 中的对应文件。

### 重新初始化

项目构建系统变化时，重新运行：
```
初始化 RTOS 项目
```

## 目录结构

```
rtos-engineer-toolkit/
├── .claude-plugin/
│   ├── plugin.json             # 插件元数据
│   └── marketplace.json        # Marketplace 注册信息（自包含）
├── skills/
│   ├── project-init/SKILL.md       # 项目初始化（生成知识文档）
│   ├── rtos-workflow/SKILL.md      # 工作流路由
│   ├── kernel-review/SKILL.md      # 代码审查
│   ├── commit-helper/SKILL.md      # 提交助手
│   ├── crash-guide/SKILL.md        # Crash 分析
│   ├── task-tracker/SKILL.md       # 任务跟踪
│   └── pre-submit-review/SKILL.md  # 提交前审查
├── agents/
│   └── kernel-reviewer.md          # 内核审阅子 Agent
├── hooks/
│   ├── hooks.json                  # Hook 注册
│   ├── run-hook.cmd                # 跨平台 hook 启动器
│   └── session-start               # 会话启动脚本
├── CLAUDE.md                       # 贡献指南
├── README.md
└── package.json
```
