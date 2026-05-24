# opencode Agent Authoring Rules

## Scope

- 本文约束 opencode 自定义配置和扩展文件：`.opencode/opencode.jsonc`、`opencode.json`、`opencode.jsonc`、`.opencode/agent/**/*.md`、`.opencode/agents/**/*.md`、`.opencode/skill/**/SKILL.md`、`.opencode/skills/**/SKILL.md`、`.opencode/plugin/*.{ts,js}`、`.opencode/plugins/*.{ts,js}`、MCP、permission、instructions。
- 只在处理 opencode 配置、agent、subagent、skill、plugin、MCP、permission 规则时使用本文；普通应用代码仍遵循仓库根 `AGENTS.md` 和更近目录的规则。
- 修改任何 opencode 配置、agent、skill、plugin、MCP 或 instruction 文件后，最终回复必须提醒用户重启 opencode；运行中的会话不会热加载这些变更。

## Source Of Truth

- 配置 schema 以 `https://opencode.ai/config.json` 为准。
- 官网 agent 文档以 `https://opencode.ai/docs/agents` 为准。
- 官网 rules 文档以 `https://opencode.ai/docs/rules` 为准。
- 本仓库源码优先参考 `packages/opencode/src/config/agent.ts`、`packages/opencode/src/config/config.ts`、`packages/opencode/src/config/permission.ts`、`packages/opencode/src/agent/agent.ts`。
- 不确定字段形状时，先查 schema 或源码；不要凭记忆自造字段。
- 涉及 opencode 自定义配置时，优先加载 `customize-opencode` skill（如果当前环境提供该 skill）。

## Agent File Format

- 非平凡 agent 优先写成 Markdown 文件，不要把长 prompt 内联到 `opencode.json`。
- 项目 agent 放在 `.opencode/agent/<name>.md` 或 `.opencode/agents/<name>.md`。
- 全局 agent 放在 `~/.config/opencode/agent/<name>.md` 或 `~/.config/opencode/agents/<name>.md`，除非用户明确要求，否则不要写全局配置。
- 文件名会成为 agent 名；默认使用小写 kebab-case，例如 `code-reviewer.md`。
- 嵌套路径会成为带斜杠的名称，例如 `.opencode/agents/review/security.md` 对应 `review/security`。
- Markdown frontmatter 只写配置，正文写 system prompt。
- 使用 Markdown 文件时，不要在 frontmatter 写 `prompt:`；正文会被加载为 prompt。
- 示例必须保持 YAML 可解析，frontmatter 用 `---` 包裹。

## Frontmatter Rules

- 每个可选或可调用 agent 都必须写 `description`，一句话说明能力和何时使用。
- 每个 agent 都必须显式写 `mode`，只能是 `primary`、`subagent` 或 `all`。
- 当前 schema 支持的 agent 字段包括 `name`、`model`、`variant`、`prompt`、`description`、`temperature`、`top_p`、`mode`、`hidden`、`color`、`steps`、`options`、`permission`、`disable`、`tools`、`maxSteps`。
- Markdown agent 新文件不要使用 `prompt`、`tools`、`maxSteps`；`prompt` 改写正文，`tools` 改写 `permission`，`maxSteps` 改写 `steps`。
- `subagent` 的 `description` 必须包含触发条件，因为 primary agent 会根据它决定是否调用。
- `model` 必须使用 `provider/model-id` 格式，例如 `anthropic/claude-sonnet-4-20250514`。
- 不指定 `model` 时，primary agent 使用全局模型，subagent 通常继承调用它的 primary agent 模型。
- 需要限制成本或循环时使用 `steps`，不要使用 deprecated `maxSteps`。
- 需要低随机性分析时设置较低 `temperature`，例如 `0.1`；不要为了“更智能”随意提高 temperature。
- 需要控制采样多样性时使用 `top_p`，字段名必须是 snake_case。
- `color` 只能使用 `#RRGGBB` 或 `primary`、`secondary`、`accent`、`success`、`warning`、`error`、`info`。
- `hidden: true` 只适合内部 subagent；hidden agent 仍可能被 Task tool 调用，必须配合 `permission.task` 控制。
- `disable: true` 只用于显式禁用已有 agent 或内置 agent，不要用空 prompt 代替禁用。
- agent frontmatter 中未知字段会进入 provider `options`；只有在确认 provider 支持时才写额外选项。

## Deprecated Fields

- 新配置不要写 `tools`，使用 `permission`。
- 新配置不要写 `maxSteps`，使用 `steps`。
- 顶层 `mode` 配置已废弃，使用顶层 `agent`。
- 如果遇到已有 legacy 字段，不要顺手迁移无关配置；只在用户要求或当前任务需要时迁移。

## Prompt Standards

- prompt 第一段写 agent 的身份、职责和不负责的范围。
- prompt 必须写清“何时使用”和“何时不要使用”。
- prompt 必须写清只读、可写、危险操作、外部网络、外部目录访问边界。
- prompt 必须写清工作流，例如先探索、再判断、再编辑、最后验证。
- prompt 必须写清输出格式，尤其是 review、plan、report、debug 类 agent。
- prompt 必须写清何时向用户提问，何时自主执行。
- prompt 不要写空泛要求，例如“写高质量代码”“遵循最佳实践”，必须转成可验证规则。
- prompt 不要重复模型身份、营销语、泛化安全声明或与 opencode 系统规则冲突的内容。
- prompt 不要要求 agent 使用不存在的工具、字段、命令或权限。
- prompt 中提到文件路径时使用仓库真实路径或明确的占位符。

## Permission Rules

- 默认最小权限；只授予 agent 完成职责所需的权限。
- 避免顶层或 agent 级 `permission: "allow"`，这等价于允许几乎所有权限。
- permission action 只能是 `allow`、`ask`、`deny`。
- 可按 pattern 配置的常用权限包括 `read`、`edit`、`glob`、`grep`、`list`、`bash`、`task`、`external_directory`、`repo_clone`、`repo_overview`、`lsp`、`skill`。
- 只能使用 flat action 的权限包括 `todowrite`、`question`、`webfetch`、`websearch`、`doom_loop`。
- pattern 规则按插入顺序保留，最后匹配的规则生效；宽泛规则写在前，具体规则写在后。
- 只读 agent 必须显式 `edit: deny`，并按需限制 `bash`、`external_directory`、`webfetch`、`websearch`。
- 写作或文档 agent 通常可以允许 `edit`，但应禁止或询问 `bash`，除非明确需要命令验证。
- 安全审计、review、exploration 类 subagent 默认不应写文件。
- 涉及外部目录时必须配置 `external_directory`，不要依赖默认行为。
- 能调用 subagent 的 primary/orchestrator agent 必须用 `permission.task` 限定可调用的 agent 名称。
- 禁止某个 subagent 被 Task tool 调用时，使用 `permission.task` 的 deny 规则；用户仍可通过 `@` 手动调用可见 subagent。

## Primary vs Subagent

- `primary` 用于用户可直接切换并长期对话的主 agent。
- `subagent` 用于被 primary agent 委派的专项任务，不应假设自己拥有完整会话目标。
- `all` 只在 agent 同时适合作为主 agent 和 subagent 时使用；不确定时优先选更窄的模式。
- `default_agent` 必须指向存在的非 hidden、非 subagent agent。
- 不要把内部 hidden/system agent 配成默认 agent。
- 新增 subagent 时必须让 `description` 足够具体，否则 primary agent 无法稳定选择它。

## Built-in Agents

- `build` 是默认 primary agent，适合执行开发修改；覆盖它前必须说明原因。
- `plan` 是规划和分析 agent，应保持编辑受限；不要把它改成默认写文件 agent。
- `general` 是通用 subagent，适合复杂多步骤任务；不要用它替代有明确边界的专项 subagent。
- `explore` 是快速只读探索 subagent；不要给它写权限。
- `scout` 是外部文档和依赖源码研究 subagent，可能受实验开关控制；不要假设所有环境都存在。
- `compaction`、`title`、`summary` 是 hidden system agents；不要作为普通业务 agent 模板。
- 覆盖内置 agent 时只覆盖必要字段，保留其职责边界和安全意图。

## Skills, Plugins, MCP

- skill 文件必须是 `.opencode/skill/<name>/SKILL.md` 或 `.opencode/skills/<name>/SKILL.md`。
- skill frontmatter 必须包含 `name` 和 `description`，`description` 要写触发条件和关键词。
- `skills` 配置是对象，形如 `{ "paths": [], "urls": [] }`，不是数组。
- server plugin 配置字段是 `plugin`，值是数组，不是对象。
- 本地 server plugin 可自动发现于 `.opencode/plugin/*.{ts,js}` 或 `.opencode/plugins/*.{ts,js}`。
- 不要混淆 server plugin 和 TUI plugin；TUI plugin 使用 `tui.json`，不会通过 `.opencode/plugins` 自动发现。
- MCP 配置字段是 `mcp`，值是以 server name 为 key 的对象。
- MCP local server 必须写 `type: "local"` 和字符串数组 `command`。
- MCP local server 的环境变量字段以当前 schema 为准，此版本使用 `environment`。
- MCP remote server 必须写 `type: "remote"` 和 `url`。
- 禁用继承来的 MCP server 时使用 `{ "enabled": false }`。

## Change Workflow

- 修改前先读取现有配置，保留用户已有字段和注释语义，不重排无关内容。
- 修改 `opencode.json` 或 `opencode.jsonc` 时保留 `"$schema": "https://opencode.ai/config.json"`。
- 顶层未知字段会导致配置校验失败；不要新增 schema 中不存在的顶层字段。
- agent 内未知字段会进入 `options`，不要把拼写错误伪装成 provider option。
- 新增 agent 前检查同名 agent 是否已存在，避免覆盖用户配置。
- 优先新增独立 agent/skill/plugin 文件；只有短小配置才内联到 `.opencode/opencode.jsonc`。
- 不要手动维护 `.opencode/node_modules`、lockfile 或安装产物，除非用户明确要求。
- 修改后最终回复必须列出改动文件，并提醒重启 opencode。

## Validation Checklist

- 路径是否正确：agent 在 `.opencode/agent(s)/*.md`，skill 在 `.opencode/skill(s)/*/SKILL.md`。
- frontmatter 是否可解析，字段缩进是否正确。
- `description` 是否写清能力和触发场景。
- `mode` 是否为 `primary`、`subagent` 或 `all`。
- 是否使用了 deprecated `tools` 或 `maxSteps`。
- `permission` 是否最小化，是否避免了 `permission: "allow"`。
- pattern 权限是否把宽泛规则放前、具体规则放后。
- prompt 是否包含职责、边界、工作流、输出格式和提问条件。
- `model` 是否是 `provider/model-id`。
- `default_agent` 是否指向非 hidden、非 subagent agent。
- 修改配置后是否提醒用户重启 opencode。

## Standard Subagent Example

```markdown
---
description: Reviews code changes for correctness, security, regressions, and missing tests without editing files.
mode: subagent
temperature: 0.1
permission:
  edit: deny
  bash:
    "*": ask
    "git diff*": allow
    "git status*": allow
    "git log*": allow
  webfetch: deny
  websearch: deny
---

You are a code review subagent.

Focus on bugs, behavior changes, security risks, and missing tests. Do not edit files.

Workflow:

- Inspect the relevant diff and nearby code before judging.
- Report findings first, ordered by severity.
- Include file and line references when available.
- If no findings exist, say so and mention residual risks or unrun tests.

Output format:

- Findings
- Open questions
- Testing notes
```

## Docs Writer Example

```markdown
---
description: Writes and updates project documentation with file edits but without running shell commands by default.
mode: subagent
temperature: 0.2
permission:
  edit: ask
  bash: deny
  webfetch: ask
---

You are a documentation writer.

Create concise, accurate documentation that matches the existing project style. Ask before changing public behavior claims you cannot verify from the repository or referenced docs.

Workflow:

- Read nearby docs and source before writing.
- Prefer small edits over broad rewrites.
- Preserve terminology already used by the project.
- Summarize changed files and any assumptions at the end.
```

## Anti-patterns

- Do not create agents with vague descriptions like `Helps with code`.
- Do not grant `permission: "allow"` to convenience agents.
- Do not use `tools` in new configs.
- Do not use `maxSteps` in new configs.
- Do not define long prompts inline in `opencode.json`.
- Do not create global agents unless the user explicitly asks for global behavior.
- Do not override `build`, `plan`, `explore`, or hidden system agents without a concrete reason.
