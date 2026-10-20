# Agent Playbook

> A battle-tested playbook for **one human + many AI agents** (Claude Code / Codex / Cursor / Kimi / ...).
>
> 中文文档 · Chinese documentation. Covers context engineering, agent memory architecture, evidence discipline,
> handoff protocols, and copy-paste workflows for multi-agent collaboration.
>
> 全部内容已脱密：不含任何个人信息、机构名称、项目细节、学生信息或商业数据。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/AlexWangzhixin/agent-playbook?style=social)](https://github.com/AlexWangzhixin/agent-playbook/stargazers)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

**Keywords**: `ai-agents` `llm` `context-engineering` `agent-memory` `mcp` `claude-code` `codex`
`multi-agent` `ai-workflow` `best-practices` `knowledge-management` `awesome`

---

## 它解决什么问题

当一个人同时使用多个 AI Agent 处理不同类型的工作时，会反复撞上同一组问题：

1. **每个 agent 各记各的** —— 同一个事实在不同 agent 的记忆里出现不同版本，甚至互相矛盾。
2. **协作偏好无法传承** —— 每次新会话都要重新交代"我要 Markdown、要证据分级、别客套"。
3. **工作流无法沉淀** —— 同一个流程做了十遍，第十一遍还是要从头推。
4. **档案无处安放** —— 放在某个 agent 私有目录里，换工具就丢；放在云盘里，又怕隐私泄露。

这个仓库给出一套解法：**纯文本档案 + git 版本管理 + 明确的读写协议**。

---

## 三条核心原则

### 1. 证据分级，不说"已验证"

任何结论必须标注来源等级。最重要的一条红线：

> **方案、清单、诊断、合成数据、"有限工况吻合"——一律不得说成"已验证"。**

| 标号 | 含义 | 能否当事实用 |
| --- | --- | --- |
| E1 已核实 | 有证件/原件/系统记录/独立重算支撑 | 可以 |
| E2 本地证据 | 有文件、日志、截图，但未独立复核 | 可以，注明来源 |
| E3 自报 | 本人或他人陈述，无第三方证据 | 注明"自报 + 日期"，结论前刷新 |
| E4 推断 | 多条间接证据推出 | 必须标注【推断】，不得作事实陈述 |
| E5 待确认 | 存在冲突或只有单方说法 | **不得使用**，先问人 |
| E6 已作废 | 曾经的错误说法 | **禁止引用**，仅作回溯反例 |

配套硬规则：

- **编译通过不等于交付。** 文档类必须真实渲染并逐页检查版式、公式、字体、占位符。
- **不采信"文件自报的计数"** 和陈旧校验器，关键数字独立重算。
- **事实、假设、推测、团队成果、计划、待确认必须分开表述。**

### 2. 先只读盘点，再最小可追溯改动

- 进入写入前**等确认**，不要边看边改。
- 建立「反馈 -> 锚定原文 -> 改动 -> 状态」映射，每处改动都能回指到具体反馈。
- **保留**术语、数字、公式、编号、引用、作者自填字段，不要"顺手优化"。
- **不升级范围**：说好"一星、13 页"就保持一星 13 页，不要擅自扩成二星 24 页。
- **不把团队成果写成个人成果**，也不把计划写成已完成。

### 3. 每处改动都提交，方便回退

档案不是文档，是**版本库**。每条变更都要能回答"为什么改、证据是什么、谁确认的"。

```text
<type>(<scope>): <一句话说明>

<正文：为什么改，证据是什么，谁确认的>

Agent: <你的名字>
Confirmed-by: <确认人>
Source: <会话/文件/任务>
```

- `type`：`feat`（新增）/ `fix`（更正错误事实，必须写"旧说法 -> 新事实"）/ `docs`（措辞结构）/ `chore`（维护）
- 回退用 `git revert <hash>`，**不要 `reset --hard`**——保留可追溯性。

---

## 目录

```text
agent-playbook/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── method/
│   ├── evidence-discipline.md     证据分级与表述纪律
│   ├── memory-stack.md            多层记忆栈：归属、权威源、防竞争副本
│   └── agent-handoff.md           交接文档模板与纪律
├── workflows/
│   ├── deliverable-qa.md          交付物 QA 清单
│   ├── file-safety.md             文件操作安全协议
│   ├── shared-memory.md           跨 Agent 共享记忆
│   └── reproducible-env.md        可复现环境与后台任务
└── templates/
    ├── AGENTS.md                  Agent 接入说明模板
    ├── project-context.md         项目 00_CONTEXT 模板
    └── commit-message.txt         提交信息模板
```

---

## 快速开始

**作为 Agent 使用者**：把 [`templates/AGENTS.md`](templates/AGENTS.md) 复制到你的档案仓库根目录，
按其中五步协议填写，之后任何 agent 连上仓库都能自动进入状态。

**只想要一份清单**：直接看 [`workflows/deliverable-qa.md`](workflows/deliverable-qa.md)。

---

## 设计取舍

| 选择 | 理由 |
| --- | --- |
| 纯 Markdown，不用数据库 | 任何编辑器可读；git diff 有意义；长期可打开 |
| git 而非"另存为 v2" | 每次改动都有原因和作者，可逐条回退 |
| 显式证据标号而非自然语言 | 可被脚本检查，也逼人思考 |
| 反对 `git add -A` | 见 [`workflows/file-safety.md`](workflows/file-safety.md) 的真实事故 |

---

- 初始发布：2026-10-20
- 许可：MIT
