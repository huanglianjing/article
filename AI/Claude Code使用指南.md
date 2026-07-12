# 1. 准备工作

## 1.1 安装 claude

claude 是在终端启用 claude cli 的命令。

macOS 安装 Claude Code，官方提供的命令是通过脚本安装。

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

实际发现安装报错了，还推荐可以通过 homebrew 来安装，但实测会给我的 m 芯片 macbook 安装了 x86 版本的 claude，因此不推荐这个方法。

```bash
brew install --cask claude-code
```

从官网下载 Node.js 安装：https://nodejs.org，然后通过 npm 安装 claude。

```bash
# 安装
sudo npm install -g @anthropic-ai/claude-code

# 更新
sudo npm update -g @anthropic-ai/claude-code
```

安装完成后，在启动前需要先开启代理和 TUN 模式，最好再通过命令检测一下，确保当前 IP 地区不是中国大陆或香港，否则启动的时候会报错地区不支持 Claude。

```bash
curl ipinfo.io
```

启动 Claude Code：

```bash
claude
```

初次进入会进入配置页面，需要依次配置主题颜色、绑定的账号。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_open_configuration.jpg)

## 1.2 配置第三方 API Key

由于 Anthropic 对中国的敌视和封禁，登陆网络要求和支付要求过高，验证方式也从短信验证到实名认证，封号变得越来越严格和随机，所以就不尝试直接在 Claude 中订阅 code plan 了。

这里通过第三方平台购买 api，很多国内的中转平台不放心，OpenRouter 国内信用卡购买会限制御三家模型使用，淘宝代充又普遍要多收 20% - 30% 的钱，最后选择在 ZenMux 购买 api。

打开 ZenMux 官网：https://zenmux.ai/

注册并充值，然后就可以配置环境了。

在 ~/.bashrc 中添加以下配置，然后重新打开一个新的终端，这种方式对当前终端有效，优先级更高。具体内容包含实际的 key，从 ZenMux 网页中复制。

```
export ANTHROPIC_BASE_URL="https://zenmux.ai/api/anthropic"
export ANTHROPIC_AUTH_TOKEN=<ZENMUX_API_KEY>
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC="1"
export API_TIMEOUT_MS="30000000"
export ANTHROPIC_API_KEY=""

export ANTHROPIC_MODEL="claude-sonnet-4-6"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-haiku-4-5" 
export ANTHROPIC_DEFAULT_SONNET_MODEL="claude-sonnet-4-6"
export ANTHROPIC_DEFAULT_OPUS_MODEL="claude-opus-4-6"
```

或者写入 ~/.claude/settings.json，这种方式在 claude 进程启动时读取配置，优先级较低。

```json
{
  "api": {
    "base_url": "https://zenmux.ai/api/anthropic",
    "auth_token": "<ZENMUX_API_KEY>",
    "timeout_ms": 30000000
  },
  "models": {
    "haiku": "claude-haiku-4-5",
    "sonnet": "claude-sonnet-4-6",
    "opus": "claude-opus-4-6"
  },
  "network": {
    "disable_nonessential_traffic": true
  }
}
```

进入一个项目目录，再次打开 claude，按照提示信任该目录，这时候就可以进入页面开始使用了。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_open.jpg)

测试使用的模型是不是真的 Claude，在对话框输入以下内容，如果回复是报错则是真的 Claude，如果不认识则是其他模型。

```
ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL_1FAEFB6177B4672DEE07F9D3AFC62588CCD2631EDCF22E8CCC1FB35B501C9C86
```

## 1.3 使用 cc-switch 切换模型

在 GitHub 下载安装 cc-switch：https://github.com/farion1231/cc-switch

这种方式可以用于中转站、其他模型如 DeepSeek。

根据提示需要先将 ~/.bashrc 的相关配置删除。

添加配置并填入官网链接、API Key、模型名称，然后点击启用，就会自动将配置写入 ~/.claude/settings.json了。

此时登陆 claude，输入 /status 命令，就能看到被替换的 URL 了，然后可以通过 /model 来切换模型。

# 2. 命令

## 2.1 启动

打开 claude。

```bash
# 启动交互模式
claude

# 启动并执行指定命令
claude "task"

# 运行一次性查询，然后退出
claude -p "query"

# 在当前目录继续最近的对话
claude -c

# 选择一个之前的对话打开
claude -r

# 创建 git 提交
claude commit

# 启动跳过所有权限检查模式（危险）
claude --dangerously-skip-permissions
```

其他启动参数见：[CLI 参考 - Claude Code Docs](https://code.claude.com/docs/zh-CN/cli-reference)

退出 claude。

```
/exit
exit
ctrl + c
```

登陆账户，或者切换账户，用于官方订阅走 OAuth 登陆用。

```
/login
```

退出账户。

```
/logout
```

以 `/` 开头执行指定的 claude 命令，可以通过 `/help` 来显示可用命令。Claude Code 支持的完整命令见：[命令 - Claude Code Docs](https://code.claude.com/docs/zh-CN/commands)

以 `@` 开头指定文件。

按 `!` 进入终端命令执行模式。

以 `#` 加上内容，可以快速将一条信息写入记忆文件 CLAUDE.md。

以 `&` 带上命令，在后台并行执行，本地可以继续做其他事。

执行命令期间，按 esc 可以直接中断执行。

按 `shift` + `tab` 以切换交互模式：

* ? for shortcuts：默认模式，修改文件前询问用户；
* accept edits on：允许模型自动修改文件；
* plan mode on：输出规划而不执行；

## 2.2 配置

查看当前状态、配置项、当前对话使用量、最近使用量统计，可以通过方向键切换 tab 查看。

```
/status
/config
/usage
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_config.jpg)

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_stats.jpg)

查看正在使用的模型以及切换模型。

```
/model
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_model.jpg)

查看以及设置推理深度。

```
/effort
```

启动快速模式，消耗更多 token 换来更高的回复速度。

```
/fast
```

查看和管理模型在本地环境的权限范围。

```
/permissions
```

自动生成 CLAUDE.md 文件。

```
/init
```

## 2.3 对话

清空当前对话历史。

```
/clear
```

列出之前的对话，可以选择一个打开继续对话。

```
/resume
```

命名当前对话的名称。

```
/rename
```

将当前对话或者代码回退到之前的一个点。

```
/rewind
```

将当前对话内容复制到粘贴板或导出到文件。

```
/export
```

提一个额外的问题，但不插入当前对话。

```
/btw "question"
```

从当前对话分叉处一个新会话，原来的对话不受影响。

```
/branch
```

查看上下文使用情况，包含系统提示词、skills、信息、可用空间等的占比，如果有插件、子 agent、skills、CLAUDE.md 等也会列出来 token 占用。

```
/context
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/claude_code_context.jpg)

压缩上下文。

```
/compact
```

将一个额外的目录加入上下文。

```
/add-dir <dir>
```

查看和编辑当前已加载的 CLAUDE.md 文件。

```
/memory
```

## 2.4 代码

启用 plan mode。

```
/plan
```

检查本地代码修改。

```
/diff
```

对提交前的代码做 code review。

```
/review
```

启动三个 agent 分别从代码复用、代码质量、运行效率来审查改动的代码。

```
/simplify
```

## 2.5 分析

生成过去一段时间使用情况的分析报告。

```
/insights
```

## 2.6 任务

定时执行一个 prompt 或脚本。

```
/loop <间隔> <提示内容>

# 每 30 分钟检查部署状态
/loop 30m check the deploy
```

## 2.7 扩展能力

管理 agent 配置。

```
/agents
```

查看 hook 配置。

```
/hooks
```

列出可用的 skills。

```
/skills
```

管理 MCP 服务器。

```
/mcp
```

管理插件。

```
/plugin
```

# 3. 配置文件

## 3.1 .claude

Claude Code 在用户 home 目录下创建了 .claude 文件夹，用来管理全局的配置、记忆、历史和扩展能力。

~/.claude/ 目录结构如下：

```
~/.claude/
├── CLAUDE.md           # 全局记忆（最重要）
├── settings.json       # 全局配置
├── settings.local.json # 本机私有配置（可选）
├── projects/           # 各目录路径的项目的会话历史
├── commands/           # 自定义命令
├── agents/             # 自定义子 Agent
├── hooks/              # 钩子脚本
├── skills/             # SKILL
├── plugins/            # 已安装的插件
├── todos/              # 任务列表持久化
├── shell-snapshots/    # shell 环境快照
├── statsig/            # 遥测/实验开关缓存
├── ide/                # IDE 集成相关
├── file-history/       # 每个 session 中被编辑文件的版本快照，支持撤销操作
├── session-env/        # 存储每个 session 的环境变量、工作目录，用于 session 恢复
├── .credentials.json   # 登录凭证（敏感！）
├── history.jsonl       # 历史的对话内容以及对应的 session
└── logs/ 或 *.log      # 日志文件
```

项目目录下也有 .claude 文件夹，该文件夹仅作用于当前项目。

./.claude/ 目录结构如下：

```
./.claude/
├── settings.json       # 全局配置
├── settings.local.json # 本机私有配置（可选）
├── commands/           # 自定义斜杠命令 /xxx
└── agents/             # 自定义子 Agent
```

## 3.2 CLAUDE.md

CLAUDE.md 是 Claude Code 的记忆文件，用于给模型提供上下文，以提供项目背景、常用命令、编码规范、约束和设计模式、团队约定等信息，Claude Code 在执行时会自动读取这些文件，且会话压缩后会重新注入，不受压缩影响。

CLAUDE.md 文件名应该是全大写的，支持多层级加载，此外可以在 CLAUDE.md 中通过 @path 的方式导入其他文件，实现模块化拆分。

| 配置级别   | 文件路径               | 用途                                                         |
| ---------- | ---------------------- | ------------------------------------------------------------ |
| 全局用户级 | ~/.claude/CLAUDE.md    | 个人偏好，通用行为规则                                       |
| 项目级     | ./CLAUDE.md            | 项目架构，构建命令，团队规范                                 |
| 项目本地级 | ./CLAUDE.local.md      | 个人在该项目的私有备注、本地路径、测试账号，不应该提交至 git，而是添加到 .gitignore |
| 子目录     | ./sub/CLAUDE.md        | 子模块的特殊说明                                             |
| 向上递归   | 父目录链上的 CLAUDE.md | 从当前目录向上查找知道根目录或 home 目录，全部合并           |

Claude Code 在启动时，会从当前工作目录开始，向上递归直到 home 目录，收集每一级的 CLAUDE.md 和 CLAUDE.local.md，然后加载全局配置，切换到子目录时按需加载该目录下的 CLAUDE.md，将所有内容拼接进系统提示词，越靠近当前工作目录的文件优先级越高。

在记忆文件中使用 `@` 加上文件路径，可以引用另一个记忆文件。

也可以使用更多 Agent 工具通用的 AGENTS.md，将 CLAUDE.md 改为 @AGENTS.md 加上 Claude 特有补充。

## 3.3 settings.json

settings.json 是 Claude Code 的配置文件，以 json 格式存储配置。

重要字段：

* env：注入会话的环境变量，包含代理、API Key、替换的模型；
* model：默认使用的模型；
* effortLevel：推理程度；
* alwaysThinkingEnabled：是否默认开启 extended thinking；
* permissions：其中的 allow/ask/deny 成员分别记录哪些工具/命令无需确认执行、必须询问、不允许执行；
* includeCoAuthoredBy：git 提交是否带 Claude 署名，默认带，不想要就关掉；
* hooks：事件钩子，在特定事件自动跑脚本，如工具调用前后、发消息时、会话开始和结束时、发通知时；
* statusLine：自定义状态栏；
* extraKnownMarketplaces：已安装的 marketplace；
* enabledPlugins：已安装的 plugin；

## 3.4 .credentials.json

.credentials.json 是登陆凭证，存放 OAuth token、订阅登陆信息，不能提交到 git 或分享。

## 3.5 commands

.claude/commands/ 目录中可以自定义命令，然后可以在对话中手动使用。

在 ~/.claude/commands/ 下定义的是所有项目都能使用的全局命令，在 ./.claude/commands/ 下定义的是当前项目的命令。

创建一个自定义命令 ~/.claude/commands/commit.md，文件名是以命令命名的 markdown 文件。

```markdown
---
description: 查看当前暂存改动，按 Conventional Commits 规范生成并执行 git commit
argument-hint: [可选的额外说明]
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git commit:*)
---

你的任务：为当前暂存（staged）的改动生成一条高质量的 git commit。

执行步骤：

1. 运行 `git status` 确认有已暂存的改动；如果没有，提示我先 `git add` 后退出。
2. 运行 `git diff --staged` 查看具体变更。
3. 运行 `git log -5 --oneline` 参考本仓库历史的提交风格。
4. 综合 diff 内容，按 **Conventional Commits** 规范起草提交信息：
   - 格式：`<type>(<scope>): <subject>`
   - 常用 type：feat / fix / refactor / docs / test / chore / perf
   - subject 用祈使句、首字母小写、不加句号、≤ 72 字
   - 如有 break change 或多文件改动，加 body 说明"为什么"
5. 把草稿展示给我，等我确认后再用 `git commit -m "..."` 提交。
6. 如果用户在调用时附带了说明（见下方），把它作为额外上下文纳入。

用户附加说明：$ARGUMENTS
```

在 Claude Code 对话中像使用内置命令一样使用自定义命令。

```
/commit
/commit 这次改动包含数据库迁移，注意 review
```

## 3.6 agents

.claude/agents/ 目录中定义的是子代理（Sub Agent），可以被主 Agent 自动委派，也可以通过 `@agent名 显示调用。子 Agent 拥有独立的子对话和上下文窗口，结束后只把结论返回主 Agent。

在 ~/.claude/agents/ 下定义的是所有项目都能使用的全局子 Agent，在 ./.claude/agents/ 下定义的是当前项目的子 Agent。

创建一个子 Agent ～/.claude/agents/code-reviewer.md，文件名是以子 Agent 命名的 markdown 文件。

```markdown
---
name: code-reviewer
description: 资深代码审查员。当用户请求 code review、PR review、审查改动、找潜在 bug / 安全问题时使用。
tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Bash(git status:*)
model: claude-sonnet-4-5
---

你是一名拥有 10 年经验的资深工程师，专精代码审查。**你只读不写**——绝不修改任何文件。

## 工作流程

1. 通过 `git status` / `git diff` 了解待审查的改动范围（若用户指定了 PR 或文件，则聚焦该范围）。
2. 必要时用 Read / Grep 阅读相关上下文，理解改动在系统中的位置。
3. 按以下维度逐项审查：
   - **正确性**：边界情况、错误处理、并发、空值
   - **安全**：注入、鉴权、密钥泄漏、依赖漏洞
   - **性能**：复杂度、N+1 查询、不必要的同步阻塞
   - **可读性**：命名、函数粒度、注释是否解释"为什么"
   - **测试**：是否有对应测试、覆盖关键路径
   - **架构一致性**：是否符合项目既有模式

## 输出格式（严格遵守）

最终用以下结构汇报，**必须分级**：

### 🔴 必须修复（Blocker）
- 文件:行号 — 问题描述 — 建议

### 🟡 建议改进（Should）
- ...

### 🟢 可选优化（Nit）
- ...

### 总体评价
1–2 句话总结，并明确给出 **APPROVE / REQUEST CHANGES / COMMENT** 之一。

## 行为约束

- 不要做大段复述代码，引用关键行即可。
- 没有发现问题的维度可以直接说"无问题"，不要硬凑。
- 用中文回答。
```

该子 Agent 可以在 Claude Code 对话中显示调用或者被自动委派。

```
@code-reviewer 帮我审查一下当前分支相对 main 的改动
我刚写完 src/auth/login.ts，帮我 review 一下
```

## 3.7 hooks

hooks 是一种允许在 agent 执行特定事件时自动运行 shell 命令或脚本的机制，通过监听 agent 生命周期中的事件触发。

常见事件包括：

* PreToolUse: 工具调用前触发（可以阻止调用）
* PostToolUse: 工具调用后触发
* UserPromptSubmit: 用户提交消息时触发
* Stop: agent 完成响应时触发
* SubagentStop: 子 agent 完成时触发
* Notification: 需要用户注意时触发
* SessionStart / SessionEnd: 会话开始/结束时触发

执行脚本通过 stdin 接收 JSON 格式事件数据，通过 stdout 返回结果，通过 exit code 控制流程（exit 0 = 允许，exit 2 = 阻止并把 stderr 反馈给 agent）。

将需要执行的脚本放在 .claude/hooks/ 中：

```shell
#!/usr/bin/env bash

echo "pretooluse"
exit 0
```

在 setting.json 中配置 hooks：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/pretooluse.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Session ended at $(date)\" >> ~/.claude/session.log"
          }
        ]
      }
    ],
  }
}
```

## 3.8 skills

skills 是一种按需加载的专项能力包，在 Claude 遇到特定任务时，读取预先写好的指南，按里面的步骤、规范、最佳实践来完成任务。

skills 存放在 .claude/skills 中，以 skill 名称命名一个文件夹，再在其中创建一个 SKILL.md，文件夹内可以额外放其他脚本、模版、参考文档供 skill 调用。在 ~/.claude/skills/\<skill-name\>/SKILL.md 下定义的是所有项目都能使用的全局 skill，在 ./.claude/skills/\<skill-name\>/SKILL.md下定义的是当前项目的 skill。

创建一个 skill 在 ~/.claude/skills/pr-splitter/SKILL.md，开头部分是 skill 的名称和介绍说明，后面部分则是具体的内容。

```markdown
---
name: pr-splitter
description: Split current branch's changes into multiple small PRs. Use when the user asks to split a branch / PR / changes into smaller pieces.
---

# PR Splitter

## Steps

1. `git diff main...HEAD --stat` 查看所有改动
2. 按功能内聚把文件分组，每组 → 一个 PR
3. 对每组：
   - `git checkout main && git checkout -b <topic>`
   - cherry-pick 或 checkout 该组文件
   - 推送并 `gh pr create`

## Rules
- 每个 PR ≤ 400 行
- 每个 PR 必须能独立通过 CI
- 有依赖的 PR 在描述里标注 "depends on #N"
```

该 skill 可以在 Claude Code 对话中显示调用或者被自动触发，或者创建一个自定义命令并在其中显示调用该 skill。

```
用 pr-splitter skill 帮我把当前分支拆成多个 PR
帮我把这个分支 split 成几个小 PR
```

skill 目录还可以包含其他辅助文件，如示例、脚本。

```
my-skill/
├── SKILL.md           # 主要说明（必需）
├── template.md        # Claude 要填写的模板
├── examples/
│   └── sample.md      # 显示预期格式的示例输出
└── scripts/
    └── validate.sh    # Claude 可以执行的脚本
```

## 3.9 plugins

插件（Plugins）可以扩展 Claude Code 的能力，安装一个插件可能包含对应的命令、skill、子 agent、hooks、MCP 服务器。

使用 /plugin 命令查看和管理插件，已安装的插件将会记录到 setting.json 中。

```
# 打开插件面板
/plugin

# 安装插件
/plugin install <name>
/plugin install <plugin-name>@<marketplace-name>

# 列出插件
/plugin list

# 启用插件
/plugin enable <name>

# 停用插件
/plugin disable <name>

# 卸载插件
/plugin remove <name>
```

marketplace 是一份插件的列表清单。

```
# 列出 marketplace
/plugin marketplace list

# 添加 marketplace
/plugin marketplace add <repo>

# 更新 marketplace
/plugin marketplace update <repo>

# 移除 marketplace
/plugin marketplace remove <repo>
```

一些热门的 plugin 如下：

* feature-dev：把做新功能拆成多阶段结构化工作流；
* code-review：多 agent 代码审查；
* context7：实时拉取库的最新文档，防止 Claude 训练数据的 API 过时；

安装了 plugin 后，会在 .claude/plugins/marketplaces/\<marketplace-name\>/plugins/\<plugin-name\>/ 内存放 plugin 支持的能力，而不是将其散落到 .claude 的默认文件夹内。示例如下：

```
my-plugin/
├── plugin.json                  # manifest：名称、版本、描述
├── commands/
│   ├── deploy.md                # /deploy 命令
│   └── review.md                # /review 命令
├── skills/
│   └── api-design/
│       ├── SKILL.md             # 技能描述与触发条件
│       └── reference.md         # 技能引用的资料
├── agents/
│   └── test-writer.md           # 子 agent 定义
├── hooks/
│   ├── hooks.json               # 钩子配置
│   └── pre-commit.sh            # 钩子脚本
└── .mcp.json                    # plugin 自带的 MCP 服务器
```

安装完对应的 plugin 之后，就可以使用其中包含的命令、skill、子 agent、hooks、MCP 服务器能力了。

# 4. 参考

* [Claude Code Docs](https://code.claude.com/docs/zh-CN/)

