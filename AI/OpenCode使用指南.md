# 1. 准备工作

## 1.1 安装

OpenCode 是一个开源的 AI coding agent，可以接入任何模型的 API 使用。

官方网站：https://opencode.ai/

通过安装脚本安装。

```bash
curl -fsSL https://opencode.ai/install | bash
```

也可以通过 npm 安装 opencode。

```bash
# 安装
sudo npm install -g opencode-ai

# 更新
sudo npm update -g opencode-ai
```

启动 OpenCode：

```bash
opencode
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_open.jpg)

如果安装后没有配置模型，OpenCode 会默认使用官方提供的免费模型服务（OpenCode Zen/Big Pickle）。

## 1.2 配置模型

打开 OpenCode 后，输入命令 `/connect` 之后选择一个模型，这里我们选择 DeepSeek。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_connect.jpg)

然后粘贴 API Key，选择模型和推理程度，就可以看到配置了 DeepSeek 了。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_connect_deepseek.jpg)

# 2. 命令

## 2.1 启动

打开 opencode。

```bash
# 启动新对话
opencode

# 以指定工作目录启动
opencode /path/to/project

# 继续上次对话
opencode -c
opencode --continue

# 继续指定对话
opencode -s <sessionid>
opencode --session <sessionid>

# 执行一个对话然后退出
opencode run "question"

# 从浏览器打开
opencode web

# 列出所有对话
opencode session list

# 显示花费和用量
opencode stats

# 导出对话
opencode export <sessionid>

# 升级opencode
sudo opencode upgrade

# 查看命令用法
opencode --help
```

退出 opencode。

```bash
/exit
/quit
/q
exit
quit
```

以 `/` 开头执行命令。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_command.jpg)

按 `ctrl` + `p` 列出命令列表，可以鼠标滚动、点击、搜索命令。

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_commands.jpg)

以 `@` 开头指定文件。

按 `!` 进入终端命令执行模式。

按 `tab` 切换交互模式：

* Build：构建模式，可以回复消息或者改动文件；
* Plan：计划模式，不进行任何修改，而是讨论建议如何实现；

## 2.2 配置

添加提供商并添加密钥。

```
/connect
```

也可以通过配置文件定义自定义提供商的 Base URL，然后再通过命令添加密钥。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI ProviderDisplay Name",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "my-model-name": {
          "name": "My Model Display Name"
        }
      }
    }
  }
}
```

切换模型。

```
/models
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_model.jpg)

开启或关闭思考过程。

```
/thinking
```

切换主题，切换时还能实时看到切换的效果。

```
/themes
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_themes.jpg)

自动生成 AGENTS.md 文件。

```
/init
```

## 2.3 对话

查看已有对话，可以在这个界面切换、重命名和删除对话。

```
/sessions
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_sessions.jpg)

新建对话。

```
/new
```

压缩当前对话。

```
/compact
```

导出当前对话内容。

```
/export
```

![](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/ai/opencode_export.jpg)

## 2.4 代码

撤销修改。

```
/undo
```

重做修改。

```
/redo
```

## 2.5 分享

将对话生成一个链接，可以分享给其他人打开查看。

```
/share
```

取消分享当前对话。

```
/unshare
```

# 3. 配置文件

## 3.1 opencode.json

OpenCode 支持 json 和 jsonc 格式的配置文件，文件名称为 opencode.json。

配置文件分为多级：

* 全局配置：~/.config/opencode/opencode.json，作用于所有项目，适合放个人偏好，如默认模型、主题、权限、快捷键等；
* 自定义配置：环境变量 `OPENCODE_CONFIG` 设置的路径文件；
* 项目配置：项目目录中的 ./opencode.json，适合提交到 git，约束项目规则、工具权限、MCP 等，会覆盖全局配置的相同配置；

重要字段：

* $schema：声明配置的 schema；
* theme：主题；
* model：默认的大模型；
* small_model：默认的轻量任务模型；
* provider：api 提供商配置；
* permission：操作权限；
* tools：启用或禁用工具；
* instructions：项目规则文件；
* default_agent：默认 agent，内置代理如 build 或 plan；
* agent：自定义 agent；
* command：自定义命令：
* keybinds：自定义快捷键；
* mcp：MCP 服务器；
* tui：TUI 终端界面配置；
* watcher：文件监听忽略；

如下是一个例子：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  "permission": {
    "edit": "ask",
    "bash": "ask"
  },
  "instructions": ["AGENTS.md"],
  "watcher": {
    "ignore": ["node_modules/**", "dist/**", ".git/**"]
  }
}
```

使用环境变量作为值：

```json
{
  "model": "{env:OPENCODE_MODEL}",
}
```

使用文件的内容作为值：

```json
{
  "provider": {
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"
      }
    }
  }
}
```

详细配置见 https://opencode.ai/docs/config/

## 3.2 AGENTS.md

AGENTS.md 是给 agent 使用的规则文件，类似 Claude Code 的 CLAUDE.md，但是更加通用。OpenCode 也兼容 CLAUDE.md 作为 fallback。

全局级文件 ~/.config/opencode/AGENTS.md 用于个人偏好。

项目级文件 ./AGENTS.md 仅用于当前项目，用于说明项目情况和约束。

## 3.3 .opencode

.opencode 目录是配置目录，存放其他配置文件，同样分为全局配置 ~/.config/opencode/ 和 项目配置 ./.opencode/。

除了默认的配置路径，还可以设置环境变量 `OPENCODE_CONFIG_DIR` 来指定自定义配置目录。

其下级目录结构如下：

```
├── agents/   # 自定义 Agent
├── commands/
├── plugins/
├── skills/
├── tools/
└── themes/
```

## 3.4 agents

配置目录中的 agents 子目录中存放自定义 agent。

创建一个 md 文件为 .opencode/agents/review.md，文件名就是自定义 Agent 的名字。

```自定义
---
description: Review code for quality and security
mode: subagent
model: anthropic/claude-sonnet-4-5
tools:
  write: false
  edit: false
  bash: false
---

Focus on bugs, edge cases, security issues, and missing tests.
Do not modify files.
```

也可以将它配置在 opencode.json 配置里。

```
{
  "agent": {
    "review": {
      "description": "Review code without making changes",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-5",
      "tools": {
        "write": false,
        "edit": false,
        "bash": false
      }
    }
  }
}
```

在对话中可以显式调用自定义 agent：

```
@review 审查本地文件变更
```

## 3.5 commands

commands 子目录中存放自定义命令。

创建一个 md 文件 .opencode/commands/test.md，文件名就是自定义命令的名称。

```markdown
---
description: Run tests and analyze failures
agent: build
---

Run the full test suite.
If anything fails, explain the failure and suggest a fix.
```

也可以将它配置在 opencode.json 配置里。

```json
{
  "command": {
    "test": {
      "description": "Run tests and analyze failures",
      "agent": "build",
      "template": "Run the full test suite. If anything fails, explain the failure and suggest a fix."
    }
  }
}

```

定义完成后，可以在对话中使用该自定义命令：

```
/test
```

## 3.6 skills

skills 子目录存放的是自定义的 skill。

在 .opencode/skills/ 以 skill 名称创建文件夹，然后在其中创建 SKILL.md，如 .opencode/skills/git-release/SKILL.md。

```markdown
---
name: git-release
description: Create consistent releases and changelogs
license: MIT
compatibility: opencode
metadata:
  audience: maintainers
  workflow: github
---

## What I do

- Draft release notes from merged PRs
- Propose a version bump
- Provide a copy-pasteable `gh release create` command

## When to use me

Use this when you are preparing a tagged release.
Ask clarifying questions if the target versioning scheme is unclear.
```

skill 会在对话时显式使用或者被 agent 识别自动调用。

# 4. 参考

* [简介 | OpenCode](https://opencode.ai/docs)

