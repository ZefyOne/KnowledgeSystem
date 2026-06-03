# Claude Code 使用手册

> 按使用频率分三层：基础层（日常必备）、进阶层（跟着教程能配）、高级层（懂行人专用）。

---

## 基础层 — 日常必备

### 安装

```bash
# npm（推荐，跨平台）
npm install -g @anthropic-ai/claude-code

# macOS Homebrew
brew install claude-code

# 升级
claude update
```

### 启动与认证

```bash
claude              # 首次启动会引导 API Key 或 OAuth 登录
claude -c           # 恢复上一次会话（需在同一目录）
claude "帮我解释这个项目"   # 直接带问题启动
```

认证方式二选一：
- **API Key**：在 [console.anthropic.com](https://console.anthropic.com) 生成，适合按量付费
- **OAuth**：通过 Claude Max 订阅登录，适合订阅用户

### 三种核心模式

| 模式 | 触发 | 用途 |
|------|------|------|
| 默认模式 | 直接聊天 | 日常使用，AI 自主判断简单直接做、复杂先问 |
| Plan Mode（计划模式）| `/plan` 或 `Shift+Tab` | 先出方案，你审批后再动手 |
| Accept Edits On | `Shift+Tab` 切换 | 编辑自动写入，不再逐条确认 |

详见 [[Claude各模式]]。

### 日常对话中的关键操作

- **提需求**：直接用自然语言描述即可 — "修复这个 bug"、"给 User 类加个 email 字段"、"这段代码是什么意思"
- **@ 引用文件**：输入 `@` 可以快速引用文件或文件夹作为上下文
- **拖拽文件**：把文件/图片拖进终端即可让 Claude 读取
- **粘贴截图**：直接粘贴图片到终端，Claude 能识别图片内容
- **! 前缀**：在 prompt 前加 `!` 可运行 shell 命令，输出直接进对话（如 `! cat error.log`）
- **权限弹窗**：每次文件读写/命令执行会弹出确认，选 Allow 放行，Deny 拒绝

### 常用斜杠命令

详见 [[命令]]，这里列最高频的：

| 命令 | 用途 |
|------|------|
| `/help` | 列出所有命令 |
| `/clear` | 清除当前对话上下文 |
| `/compact` | 压缩对话历史，释放上下文窗口 |
| `/init` | 在当前项目生成 CLAUDE.md |
| `/config` | 查看/修改配置 |
| `/cost` | 查看 token 用量和费用 |
| `/undo` | 撤销最近一次文件修改 |

### 代码操作速查

```bash
# 让 Claude 帮你做这些事，直接说就行：
"帮我提交代码"          # Claude 会 git status → diff → 写 commit message → 提交
"创建一个 PR"           # 会自动 gh pr create
"运行测试看看有没有问题"
"解释 src/auth.ts 的逻辑"
"这个函数有没有 bug"
```

### 三种权限模式

在输入框按 `Shift+Tab` 循环切换：

1. **Ask before editing**（默认）— 每次编辑弹窗确认
2. **Accept edits automatically** — 编辑自动生效，不弹确认
3. **Plan mode** — 只规划不动手

---

## 进阶层 — 跟着教程能配

### CLAUDE.md — 项目记忆文件

在项目根目录创建 `CLAUDE.md`，Claude 每次启动自动加载。相当于项目的"使用说明书"。

```markdown
# CLAUDE.md

## 项目概述
这是一个 React + Express 的全栈博客应用。

## 常用命令
- `npm run dev` — 启动开发服务器
- `npm test` — 运行测试

## 代码风格
- 使用 TypeScript 严格模式
- 组件放在 src/components/ 下
- 测试文件与源文件同目录
```

**进阶用法**：
- `CLAUDE.local.md` — 本地覆盖，不提交 git，适合个人偏好
- 子目录中可以放自己的 `CLAUDE.md`，进入该目录时自动加载
- 支持在 CLAUDE.md 中引用其他文件

### settings.json — 全局/项目级配置

```
~/.claude/settings.json          # 全局配置（用户级）
项目根目录/.claude/settings.json  # 项目配置（会覆盖全局）
项目根目录/.claude/settings.local.json  # 本地覆盖，不提交 git
```

**项目级 settings.json 示例**：

```json
{
  "permissions": {
    "allow": [
      "npm run dev",
      "npm run build",
      "npm test"
    ],
    "deny": [
      "rm -rf *",
      "git push --force origin main"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

**关键配置项**：

| 配置 | 说明 |
|------|------|
| `permissions.allow` | 自动放行的命令列表（支持 glob） |
| `permissions.deny` | 永远禁止的操作 |
| `permissions.defaultMode` | 默认权限模式 |
| `hooks` | 事件钩子（见下方） |
| `model` | 指定使用的模型 |
| `env` | 设置环境变量 |

### 权限系统 — 精细控制

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run *)",
      "Read(/home/user/projects/**)",
      "Edit(/home/user/projects/**)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(git push *)"
    ]
  }
}
```

工具类型：`Bash`、`Read`、`Write`、`Edit`、`WebFetch`、`WebSearch`、`NotebookEdit`

权限作用域：项目 settings.json < 全局 settings.json（项目会覆盖全局的同一规则）

### 钩子系统（Hooks）

可以在特定事件触发时自动执行脚本：

| 事件 | 触发时机 |
|------|----------|
| `PreToolUse` | 工具调用**之前**执行 |
| `PostToolUse` | 工具调用**之后**执行 |
| `Notification` | Claude 等待用户回复时 |
| `Stop` | Claude 完成本轮回复时 |
| `PreCompact` | 上下文压缩之前 |
| `SessionStart` | 会话启动时 |

**示例 — 编辑后自动格式化**：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "npx prettier --write $CLAUDE_FILE_PATH"
      }]
    }]
  }
}
```

特殊环境变量：`$CLAUDE_FILE_PATH`（被操作的文件路径）、`$CLAUDE_TOOL_NAME`（工具名）、`$CLAUDE_TOOL_INPUT`（工具输入 JSON）。

### 记忆系统（Memory）

Claude Code 有一个持久化的记忆系统，存放在 `~/.claude/projects/<项目路径>/memory/`。

```bash
/memory           # 打开记忆管理界面
```

记忆类型：
- **user** — 你的角色、偏好、知识背景
- **project** — 项目目标、决策、约束
- **feedback** — 你对 Claude 行为的纠正和确认
- **reference** — 外部系统链接（Jira、Grafana 等）

不需要手动管理，Claude 会在合适时机自动保存。你也可以直接说"记住 X"来主动添加。

### IDE 集成

**VS Code**：
```bash
# 安装 VS Code 扩展后，在终端面板中使用
code --install-extension anthropic.claude-code
```

**JetBrains**：
在插件市场搜索 "Claude Code" 安装。

两种方式下，Claude Code 都在 IDE 的内置终端中运行，可以访问项目文件结构。

### MCP 服务器（Model Context Protocol）

MCP 让 Claude 对接外部工具和数据源（数据库、API、文件系统等）。

**配置方式** — 在 `settings.json` 中添加：

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-filesystem", "/path/to/allowed/dir"]
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"]
    }
  }
}
```

配置后，Claude 会自动获得对应工具（如直接访问 GitHub Issues、数据库查询等）。

**常用官方 MCP Server**：
- `@anthropic-ai/mcp-server-filesystem` — 安全访问文件系统
- `@anthropic-ai/mcp-server-github` — 访问 GitHub 仓库、Issues、PRs
- `@anthropic-ai/mcp-server-postgres` — 直连 PostgreSQL
- `@anthropic-ai/mcp-server-puppeteer` — 浏览器自动化

### Worktree 隔离

当你想在干净的分支上做实验，不影响当前工作区：

```bash
# Claude 自动创建临时 worktree，在新分支上工作
# 在对话中直接说 "创建一个 worktree 来做这个功能"
```

或者直接要求 Claude："在隔离的 worktree 里做这个改动"。

### 定时任务

```bash
/loop 5m 检查部署状态      # 每 5 分钟执行一次
/loop 1h 运行 npm test     # 每小时运行测试
```

最小间隔 60 秒，定时任务 7 天后自动过期。

---

## 高级层 — 懂行人专用

### 自定义斜杠命令（Custom Slash Commands）

在 `~/.claude/commands/` 目录下创建 `.md` 文件，文件名即命令名：

```markdown
<!-- ~/.claude/commands/deploy-staging.md -->
帮我完成以下部署步骤：
1. 运行 npm run build
2. 执行 docker build -t app:staging .
3. 推送到 staging 环境：docker push registry.example.com/app:staging
4. SSH 到 staging 服务器重启容器
5. 运行冒烟测试确认部署成功
```

使用时直接输入 `/deploy-staging`。

### 自定义子代理（Agent）

在 `CLAUDE.md` 中定义专门化代理来处理特定任务：

```markdown
## Agents

### code-reviewer
Purpose: 在每次 commit 前做代码审查
Rules:
- 检查 security、performance、readability
- 不修改代码，只给建议
- 报告少于 200 字
```

### 批量处理与自动化脚本

```bash
# 非交互模式 — 一次性任务
claude -p "修复 src/ 下所有 TypeScript 文件的 lint 错误"

# 管道模式
echo "解释这个 diff" | claude -p --input-format text - < my-diff.patch

# 跳过所有权限确认（危险！）
claude --dangerously-skip-permissions -p "批量重命名所有 .js 文件为 .ts"
```

**⚠️ `--dangerously-skip-permissions` 会跳过所有安全确认，仅用于你完全信任的一次性脚本。**

### 高级 HOOK 模式

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash(git commit *)",
      "hooks": [{
        "type": "command",
        "command": "npm test && npm run lint"
      }]
    }]
  }
}
```

这个钩子会在 Claude 执行 `git commit` 之前自动运行测试和 lint，失败了就中断 commit。

### 自定义 MCP Server 开发

用 Python/TypeScript 开发自己的 MCP Server，让 Claude 访问任意内部系统：

```python
# 一个查询内部 API 的 MCP Server 骨架
from mcp.server import Server, stdio_server
from mcp.types import Tool

server = Server("my-internal-api")

@server.tool()
async def query_users(department: str) -> str:
    """查询某部门的用户列表"""
    # 调用内部 API
    return json.dumps(users)

if __name__ == "__main__":
    stdio_server.run(server)
```

### 上下文窗口管理

Claude Code 最大支持 1M token 上下文窗口。随着对话增长：

- `/compact` — 手动压缩对话历史，保留关键上下文
- 压缩由系统自动触发，但你可以在感觉上下文"膨胀"时主动执行
- 将不相关的探索性对话主动 `/clear` 掉，避免浪费 token

**实际策略**：
1. 一个会话专注一件事，完成就 `/clear`
2. 复杂任务拆分多个阶段，每个阶段开新会话
3. 跨会话的信息通过 CLAUDE.md 或 memory 传递，不依赖对话历史

### 管道与组合

```bash
# 将 Claude 的输出路由到文件
claude -p "生成这个模块的 API 文档" > docs/api.md

# 用 git diff 作为输入
git diff HEAD~5 | claude -p "审查这些变更，输出安全问题清单"

# 在 CI 中使用
claude -p "检查这个 PR 的代码变更是否符合项目规范" --dangerously-skip-permissions
```

### 会话持久化与恢复

```bash
claude -c                    # 恢复最近一次会话
claude --resume <session-id>  # 恢复到指定 ID 的会话
```

会话数据存储在 `~/.claude/sessions/` 和 `.claudian/sessions/` 中，包含完整对话历史和元数据。

### 模型选择

```bash
claude --model opus          # 使用 Opus（最强，最慢）
claude --model sonnet        # 使用 Sonnet（默认，平衡）
claude --model haiku         # 使用 Haiku（最快，适合简单任务）
```

也可以在 `settings.json` 中设置：
```json
{ "model": "claude-sonnet-4-6" }
```

### TMUX/终端复用集成

Claude 可以在 tmux 会话中运行，配合 worktree 实现真正的隔离开发环境。Claude 会自动检测 tmux 并适配合适的显示格式。

### 密钥与安全

- API Key 存储在系统密钥链中（macOS Keychain / Linux libsecret），不会明文暴露
- OAuth token 有自动刷新机制
- `settings.local.json` 适合存敏感配置，加入 `.gitignore`

---

## 速查卡片

| 层级 | 关键词 |
|------|--------|
| 基础 | `claude`、`/help`、`/clear`、`/compact`、`/cost`、`/undo`、模式切换、@引用、拖拽文件 |
| 进阶 | CLAUDE.md、settings.json、permissions、hooks、MCP、memory、worktree、IDE |
| 高级 | 自定义命令、子代理、-p 批处理、CI/CD 管道、自定义 MCP Server、上下文策略、模型选择 |
