# MemPalace For Codex

这份 README 只讲一件事：

如何把这个仓库按当前这套可工作的全局 Codex 配置装起来，让 Codex 能把 MemPalace 当作长期记忆来用。

这套配置的目标是：

- MemPalace 作为全局 MCP server 暴露给 Codex
- Codex 新会话启动时自动补录已有 transcript
- 对话中每 5 条用户消息自动触发一次记忆保存
- 用户不点名 `mempalace` 时，Codex 也会在“回忆之前的对话/决策/关系/时间线”这类请求上优先查 MemPalace

## 1. 安装到 Codex 使用的 Python 环境

建议直接用 editable install，这样你改这个仓库的代码，Codex 侧会立刻生效。

```bash
cd /path/to/mempalace
pip install -e .
```

验证：

```bash
python -m mempalace status
```

如果这里能正常输出 palace 状态，说明 Python 环境里的 `mempalace` 已经可用。

## 2. 在 Codex 里注册全局 MCP

编辑 `~/.codex/config.toml`。

至少要有这几项：

```toml
hooks = "/home/yourname/.codex/hooks.json"

[mcp_servers.mempalace]
command = "/path/to/python"
args = ["-m", "mempalace.mcp_server"]

[features]
codex_hooks = true
```

说明：

- `command` 必须指向安装了 `mempalace` 的那个 Python
- `hooks` 指向下面第 3 步要创建的 hook 文件
- `codex_hooks = true` 必须打开，否则 `SessionStart` / `Stop` 不会触发

## 3. 配置 Codex 全局 hooks

编辑 `~/.codex/hooks.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/python -m mempalace hook run --hook session-start --harness codex",
            "timeout": 30
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/python -m mempalace hook run --hook stop --harness codex",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

这两个 hook 的作用：

- `SessionStart`
  - 新会话开始时，后台扫描 `~/.codex/sessions`
  - 自动补录之前已经落盘、但还没进 MemPalace 的 Codex transcript
  - 用锁文件避免并发重复补录

- `Stop`
  - 统计 transcript 里的用户消息数
  - 默认每 5 条触发一次保存 checkpoint
  - 触发时会返回一个 block decision，让模型去把当前会话里的关键信息写入 MemPalace

默认保存间隔已经是 `5`。如果你想改，设置环境变量：

```bash
export MEMPAL_SAVE_INTERVAL=5
```

如果想把它写死进 hook 命令，也可以这样：

```json
{
  "type": "command",
  "command": "MEMPAL_SAVE_INTERVAL=5 /path/to/python -m mempalace hook run --hook stop --harness codex",
  "timeout": 30
}
```

## 4. 安装全局 skill

创建目录：

```bash
mkdir -p ~/.codex/skills/mempalace
```

写入 `~/.codex/skills/mempalace/SKILL.md`：

```md
---
name: "mempalace"
description: "Use the globally installed MemPalace MCP server whenever a request depends on earlier conversations, prior decisions, remembered facts, timelines, relationships, or durable cross-session context."
---

# MemPalace

MemPalace is installed globally in this Codex environment through MCP as `mempalace`.

Use it whenever the user asks to:
- recall earlier conversations or decisions
- remember facts about people, projects, timelines, or relationships
- search long-term memory across prior Codex sessions
- save important context from the current session

Also use it when the request implicitly depends on memory, even if the user
does not explicitly say "use MemPalace". Typical cues include:
- "之前/上次/记得/回忆/我们讨论过"
- "what did we decide", "what changed", "who is related to whom"
- requests for project history, timelines, unresolved decisions, or prior context

## Read First

1. Start with `mempalace_status` if you need a quick overview of the palace.
2. For factual recall, prefer `mempalace_search` or `mempalace_kg_query` before answering.
3. For time-bounded facts or relationship changes, prefer `mempalace_kg_query` or `mempalace_kg_timeline`.
4. Do not guess past decisions or relationships when MemPalace can verify them.

## Writing Memory

When the user asks to save memory, or when a session produces durable decisions worth preserving:
- Use `mempalace_diary_write` for a concise session diary.
- Use `mempalace_add_drawer` for verbatim memory entries when appropriate.
- Use `mempalace_kg_add` for durable facts and relationships.
- If a fact changed, invalidate the old one with `mempalace_kg_invalidate` before adding the new one.
```

这个 skill 解决的是：

- 让 Codex 知道什么时候该主动用 MemPalace
- 让 Codex 知道读什么工具、写什么工具
- 避免它只在用户明确说“去 mempalace 里查”时才调用

## 5. 加一层全局 AGENTS 规则

创建 `~/AGENTS.md`：

```md
# Global Codex Rules

## MemPalace First For Memory

- If a request depends on earlier conversations, prior decisions, remembered facts, project history, timelines, people, or relationships, consult MemPalace before answering.
- Treat phrases like `之前`, `上次`, `记得`, `回忆`, `我们讨论过`, `what did we decide`, `what changed`, and `who is related to whom` as a strong signal to use MemPalace even if the user did not name it.
- Prefer `mempalace_search` for prior conversation and decision recall.
- Prefer `mempalace_kg_query` or `mempalace_kg_timeline` when the answer depends on who/what/when, relationship validity, or status changes over time.
- If MemPalace returns nothing relevant, say memory did not contain it. Do not invent history.

## Writing Back Durable Context

- When a session produces durable decisions, stable project state, confirmed facts, or relationship updates that will matter later, write them back to MemPalace before finishing or when a save hook requests it.
- Use verbatim memory storage when exact wording matters. Use the knowledge graph for time-bound facts and relationship changes.
```

这一步不是 MCP 的替代品，而是行为层兜底。

没有这层时，模型往往知道有 `mempalace`，但不会主动去用。

## 6. 重开 Codex 会话

全局 skill、`AGENTS.md`、hooks、`config.toml` 都是给新会话读的。

所以改完以后要：

1. 关闭当前 Codex 会话
2. 新开一个会话

旧会话一般不会热加载这些配置。

## 7. 验证安装是否成功

### 7.1 验证 MCP

```bash
codex exec --json --skip-git-repo-check "只回答 OK。"
```

如果 Codex 能正常启动，并且不会报 `mempalace` server 注册错误，说明 MCP 基本通了。

### 7.2 验证新会话是否主动查记忆

在一个新会话里直接问：

```text
回忆我们上次关于 wfplan 的讨论；如果没有记录就直说。
```

正常行为应该是：

- Codex 先走 MemPalace
- 自动调用 `mempalace_search` 或 `mempalace_kg_query`
- 找不到就明确说没找到，而不是编造历史

### 7.3 验证自动补录

`SessionStart` 生效后，Codex 新会话启动时会后台扫描 `~/.codex/sessions`。

这意味着：

- 上一次对话如果异常关闭
- 但 transcript 已经落盘
- 那么下一次新会话启动时，MemPalace 会自动补录那部分历史

这套补录逻辑带锁，不会因为两个新会话同时发现旧 transcript 而重复写入。

## 8. 当前这套配置的实际行为

按当前仓库代码：

- `Stop` hook 默认 5 条触发一次，不再是 15 条
- 保存提示会明确要求模型“Use MemPalace tools”
- `SessionStart` 会触发 Codex transcript backfill
- 已经入库的 transcript 不会无脑重复补录

如果你是 editable install，这些仓库改动会立刻作用到 Codex 的全局安装。

## 9. 一份可直接对照的本机配置

下面是当前这台机器实际在用的配置形态。

`~/.codex/config.toml`

```toml
hooks = "/home/caohanghang/.codex/hooks.json"

[mcp_servers.mempalace]
command = "/home/caohanghang/miniconda3/bin/python"
args = ["-m", "mempalace.mcp_server"]

[features]
codex_hooks = true
```

`~/.codex/hooks.json`

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/home/caohanghang/miniconda3/bin/python -m mempalace hook run --hook session-start --harness codex",
            "timeout": 30
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/home/caohanghang/miniconda3/bin/python -m mempalace hook run --hook stop --harness codex",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

## 10. 不在这份 README 里讨论的东西

这里不展开讲：

- benchmark
- AAAK 设计
- Claude/Gemini/Cursor 集成
- 通用营销介绍

这份 README 只服务一个目的：

让 Codex 按这套配置把 MemPalace 装成一个真的会用、会补录、会自动保存的长期记忆系统。
