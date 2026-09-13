# agent-skill-inspector

一个独立的 companion GUI：选定项目目录后，静态查看 OMP、Claude Code、Codex CLI 对该项目会发现哪些技能，以及每个技能的来源、作用域、属性和判定原因。它只做观察层，不安装、不启停技能，不写回 `SKILL.md`，不调用模型、不联网。

- 产品边界与架构：`docs/superpowers/specs/2026-09-12-agent-skill-inspector-design.md`。
- 实现按 GitHub Issues 上的 T1–T9 票推进；目前主干还没有代码与测试脚本，加入后在此补上怎么跑测试。

## Agent skills

### Issue tracker

Issues 在本仓 GitHub Issues（`Eridanus117/agent-skill-inspector`）里，`gh` 在仓内自动识别。See `docs/agents/issue-tracker.md`.

### Triage labels

使用默认五个标签：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。See `docs/agents/triage-labels.md`.

### Domain docs

Single-context：根 `CONTEXT.md` + `docs/adr/`（按需生成）。See `docs/agents/domain.md`.
