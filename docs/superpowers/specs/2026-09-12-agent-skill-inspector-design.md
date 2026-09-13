# Agent Skill Inspector 设计

## 决定

构建一个独立的 companion GUI：用户选择一个项目目录后，查看 OMP、Claude Code、Codex CLI 对该项目的静态技能解析结果，并查看每个技能的来源、作用域、属性和判定原因。

Skills Manager 继续作为技能资产层，负责安装、更新、同步、部署、来源和版本。companion 作为观察层，不修改 Skills Manager 的数据库，不写回 `SKILL.md`，不负责安装或启停技能。

所有检查均为非对话、非模型的本地静态检查。companion 可以启动本机 CLI 子进程，但只能调用 JSON、JSON-RPC、validate 或其他明确的机器接口；禁止通过自然语言询问 agent，禁止抓取 TUI 文本，禁止启动模型会话。

本轮不决定具体 GUI 框架、数据库实现或发布渠道；这些属于实现计划阶段的技术取值，不改变本设计的产品边界。

## 基线

### 已确认事实

- 市场上已有 Skills Manager，提供技能资产管理、Project Workspace、Linked Workspace、来源查看和多 agent 部署。
- OMP 的运行时 resolver 能发现 OMP native、Claude、Codex、Agents、插件等技能并处理 provider 优先级、去重和过滤，但当前 CLI 没有完整的 `skills list --json` 报告命令。
- Claude Code 的公开 `/skills`、`/context`、`/plugin` 面向当前会话；`claude plugin validate --json` 是静态验证，不是任意目录的 effective resolver。
- Codex CLI 的公开源码包含技能目录发现和静态配置规则；app-server 的 `skills/list` 是结构化接口，但不提供完整的 provenance、覆盖原因或所有原始 frontmatter 字段。
- `disable-model-invocation` 等字段并不是所有 Agent Skills 实现的共同标准字段，需要按 runtime 归一化，同时保留未知字段。

### 目标用户问题

> 对一个选定项目目录，OMP、Claude Code、Codex CLI 会发现哪些技能？每个技能从哪里来？为什么它被视为有效、被覆盖、被禁用或无法判断？技能内部有哪些 agent-specific 属性？

### 约束

- 默认不联网、不调用模型、不执行技能正文、脚本、hooks 或 MCP server。
- 读取范围由 runtime adapter 明确规定；不因发现一个 `SKILL.md` 就递归执行或读取整个技能目录。
- 不把静态推导结果表述成模型实际遵循结果。
- 不把某个 CLI 的 UI 输出当作稳定协议。
- 不因某个 runtime 缺少机器接口而伪造“实际加载”结论。
- 用户现有 Skills Manager 安装、手工复制和 symlink 技能都应能被观察；未能取得 Skills Manager 元数据时，仍能显示文件扫描结果。

## 术语

- **项目目录**：用户在 GUI 中选择的扫描起点和 runtime cwd。
- **技能来源**：一个具体的 `SKILL.md` 及其 provider、作用域和安装位置。
- **静态解析**：只读取本地文件、配置和 CLI 提供的非对话结构化结果，不启动模型，不观察模型行为。
- **静态有效**：在已知 runtime 规则下预计会进入该 runtime 的技能集合；不代表 runtime 内部未公开状态或模型一定调用它。
- **证据模式**：结果来自非对话 CLI 结构化接口，或来自 companion 根据公开规则的 fallback 解析。
- **来源属性**：从 `SKILL.md` 或 CLI 结构化结果读出的属性，是源文件事实的快照。
- **本地追踪属性**：companion 自己保存的备注、标签、审计结论或补充配置，不写进技能源文件。

## 架构

```text
┌──────────────────────────┐
│ Skills Manager            │
│ 技能安装 / 同步 / 版本    │
└────────────┬─────────────┘
             │ path / source / version
┌────────────▼─────────────┐
│ Agent Skill Inspector      │
│                            │
│ Project selector           │
│ Static runtime adapters    │
│ Frontmatter parser         │
│ Resolution / provenance    │
│ Local tracking store       │
│ GUI report                 │
└──────┬─────────┬──────────┘
       │         │
     OMP       Claude       Codex
```

### 组件职责

#### Project selector

接受一个本地项目目录，规范化为绝对路径并记录文件系统边界。它不改变当前 shell cwd，不创建 worktree，不修改项目文件。

#### Runtime adapter

每个 adapter 对外提供同一接口：

```text
inspect(projectDir) -> RuntimeSnapshot
```

`RuntimeSnapshot` 至少包含：

```text
runtime: omp | claude | codex
resolverMode: cli | static-fallback | unsupported
runtimeVersion: string | unknown
projectDir: path
skills: SkillRecord[]
warnings: Warning[]
```

adapter 的实现顺序固定为：

1. 探测 CLI 是否存在及版本；
2. 只调用明确的非对话机器接口；
3. 如果接口缺失、版本不支持或会触发模型/网络，则转静态 fallback；
4. 如果公开规则不足以给出结论，则返回 `unsupported` 或 `unknown`，不猜测。

#### Frontmatter parser

解析每个 `SKILL.md` 顶部 frontmatter：

- 保留完整原始 key/value；
- 归一化已知 runtime 字段；
- 记录 YAML/frontmatter 错误；
- 不执行动态内容、命令替换、脚本或引用文件；
- 不把标准字段和 runtime 扩展字段混为一类。
- 未知字段在本地快照中保留；字段名或值疑似包含 `token`、`secret`、`password`、`credential`、`private-key` 等凭据内容时，GUI、JSON 和导出报告统一脱敏。

至少归一化：

```text
name
description
disable-model-invocation
user-invocable
allowed-tools
disallowed-tools
context
agent
globs
alwaysApply
hide
```

缺失字段保持 `unknown`，不自动填成 `false`，除非对应 runtime 的公开规则明确规定默认值。

#### Resolution / provenance

解析器输出候选集合和判定集合，不只输出最终胜者。每条 `SkillRecord` 至少包含：

```text
identity:
  canonicalPath
  provider
  scope
  contentHash

observed:
  directoryName
  declaredName
  description
  rawFrontmatter
  normalizedAttributes
  parseStatus

resolution:
  status: effective | shadowed | disabled | invalid | unknown | unsupported
  reason[]
  evidence: cli | documented-rule | static-fallback
  resolverVersion

manager:
  source
  sourceUrl
  installedVersion
  managedState
```

同名或同来源冲突必须保留所有候选，并在 `reason` 中说明：

- 哪个 provider 或作用域优先；
- 哪条设置规则禁用它；
- 哪个候选被哪个候选覆盖；
- 哪个字段或规则无法判断。

#### Local tracking store

本地追踪数据与技能文件分离。技能身份使用：

```text
runtime + provider + canonicalPath + contentHash
```

保存内容包括：

- 用户补充的标签、备注和审计状态；
- 对技能属性的本地观察快照；
- 上一次扫描的 resolver/version/evidence；
- 变更历史和失效记录。

本地追踪不能改变 runtime 的实际配置，也不能覆盖源文件属性。源文件更新后，旧快照保留为历史，新内容产生新身份。

## 三个 runtime 的适配边界

### OMP

优先利用 OMP 的非对话、机器可读能力。当前 CLI 没有完整 skills report 命令，因此首版允许使用 OMP 公开 discovery 规则做静态 fallback，并读取 OMP/Claude provider 对应的本地目录和配置。

OMP 对 Claude skill 的解析结果必须标记 `resolver: omp` 和 `evidence: static-fallback`，不能直接标成 Anthropic runtime 已确认。

### Claude Code

允许调用 `claude plugin validate --json` 做静态语法/schema 诊断，并读取公开的 project/user/plugin skill 路径。`/skills`、`/context`、自然语言 `claude -p` 均不作为数据源。

无法从本地公开文件还原的 `claude.ai sync`、managed skill 或内部 session 状态必须显示为 `unknown` 或 `unsupported`。

### Codex CLI

优先使用 Codex 提供的非对话结构化接口；如果调用 app-server 会引入模型、网络或不可控插件初始化，则使用 Codex 官方源码和文档定义的静态 roots、配置和 frontmatter 规则。

Codex 的 `agents/openai.yaml` interface、dependencies 和 invocation policy 作为 Codex 专属属性保存；不把它们强行转换成 Claude 字段。

## GUI 信息架构

### 项目选择页

- 选择一个项目目录；
- 显示上次扫描时间和 CLI 探测结果；
- 选择 runtime：OMP、Claude Code、Codex 或全部；
- 明确提示“静态检查，不启动对话”。

### 技能总览页

默认按技能名分组，提供 runtime、scope、status、evidence、source 和 managed 状态筛选。

同一技能存在多个来源时，显示为一个冲突组，展开后查看所有候选，不隐藏被覆盖项。

### 技能详情页

展示：

1. 文件路径和 provider/scope；
2. Skills Manager 来源/版本（如果取得）；
3. 原始 frontmatter；
4. 归一化属性；
5. 静态判定和逐条原因；
6. resolver、CLI 版本和证据模式；
7. companion 本地追踪属性；
8. 内容版本历史。

原始正文默认不自动展开，避免误把技能指令当成当前会话指令执行或阅读；用户主动点击后只做文本预览。

## 失败和未知处理

- CLI 不存在：转 `static-fallback`，同时显示缺失 CLI。
- CLI 版本不支持所需接口：转 fallback，记录版本和原因。
- CLI 接口返回错误：保留错误证据，不把空结果当成“没有技能”。
- frontmatter 损坏：记录 `invalid`，仍保留文件路径和可解析部分。
- 同名技能：全部保留，按规则输出 `effective`/`shadowed`。
- 禁用或忽略规则无法从公开配置确认：输出 `unknown`。
- 技能目录位于 workspace 外的 symlink：默认不跟随，输出安全警告。
- 读取超过大小边界：输出 `invalid`/`unsupported`，不继续读取正文或附属文件。
- Skills Manager 元数据不可用：不阻断扫描，`manager` 字段为空并说明原因。

任何错误都不能通过启动模型对话来“补答案”。

## 安全边界

- 默认只读；不执行 `SKILL.md`、脚本、hooks、MCP 配置或插件代码；
- 不启动 Claude、OMP 或 Codex 的对话会话；
- 不读取 credentials、session transcript 或无关环境变量；
- CLI 子进程使用固定参数和受控环境，不把用户输入拼接成 shell 字符串；
- 不接受 CLI 通过自然语言返回的结构化假设；
- 报告中区分公开项目路径和本地用户路径，默认不导出后者；
- 网络能力默认关闭；若某个 CLI 接口无法保证离线，则不调用，转 static fallback；
- 不提供安装、启停、编辑、删除、同步或自动修复按钮。

## 不做事项

- 不替代 Skills Manager 的技能安装和同步；
- 不实现一个新的 marketplace；
- 不把三个 runtime 合并为一个虚假的统一规则；
- 不判定模型是否遵守技能；
- 不恢复 Claude Code 的私有内部 prompt 或 session 状态；
- 不通过 `claude -p`、`codex "..."` 或 OMP 自然语言命令获取结果；
- 不执行技能携带的脚本、shell、hooks 或 MCP server；
- 不做团队 fleet 盘点；
- 不在第一版支持远程项目或云端 workspace。

## 可观察验收标准

1. 选择一个本地项目目录后，能够分别生成 OMP、Claude Code、Codex 三份静态技能报告。
2. 每条报告都明确显示 `resolverMode`、runtime 版本、证据模式和扫描警告。
3. 同名技能不会静默丢失；用户能看到生效候选、被覆盖候选及原因。
4. `disable-model-invocation`、`user-invocable`、`allowed-tools` 等字段能被读取、归一化和展示；未知字段不会被丢弃。
5. Skills Manager 来源信息可用时显示；不可用时扫描仍能完成。
6. 任何扫描过程都不启动模型对话、不执行技能代码、不联网。
7. companion 的本地追踪字段不会改变任何 runtime 或 `SKILL.md` 文件。
8. CLI 缺失、版本不支持、解析失败和无法判断都显示为明确状态，而不是空列表或成功。
9. 关闭并重新打开 companion 后，本地追踪属性和历史快照仍可恢复。
10. 报告能指出“静态预测”与“runtime 已观察”的区别；首版全部结果默认属于静态预测，除非明确取得非对话 CLI 结构化证据。

## 可逆性

首版只读，不改变技能目录、Skills Manager 数据库、runtime 配置或项目文件。删除 companion 本地追踪库即可移除观察层状态；技能资产和 runtime 配置不受影响。

如果某个 CLI 的机器接口发生变化，只需停用对应 adapter 并回退到静态规则，不影响其他 runtime。

## 未决技术取值

- GUI 框架和打包方式；
- 本地追踪库使用 SQLite 还是文件存储；
- OMP adapter 采用可选 SDK 依赖还是独立 helper 进程；
- Codex app-server 是否在离线、无插件初始化条件下可安全调用；
- Skills Manager 元数据读取采用 CLI JSON、数据库只读访问还是路径扫描。

这些问题不改变已确认的只读、非对话、CLI-first、静态 fallback 边界，应在实现计划前逐项验证。
