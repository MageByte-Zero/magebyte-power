# MageByte Power Skills

> 从生产事故中蒸馏出来的 [Claude Code Superpowers](https://superpowers.anthropic.com/) skills —— 把踩过的坑封装成可复用的工程工作流。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![English](https://img.shields.io/badge/README-English-blue)](README.md)

---

## Skills 列表

| Skill | 用途 | 适用场景 |
|-------|------|---------|
| [`cross-verified-feature-development`](#cross-verified-feature-development) | 7 阶段特性开发工作流，含 4 轮独立交叉验证 | 高风险后端特性：支付、状态机、并发控制、跨服务改造 |

---

## 什么是 Superpowers？

[Superpowers](https://superpowers.anthropic.com/) 是 Claude Code 的 MCP 插件，提供通用 skill 库：`brainstorming`（需求分析）、`writing-plans`（计划拆解）、`subagent-driven-development`（子 agent 实施）、`systematic-debugging`（系统调试）、`code-reviewer`（代码评审）等。

本仓库的 skill 是**领域专属的编排层** —— 在正确的阶段、以正确的上下文调用 Superpowers 的通用 skill，让你不需要手动把各个 skill 串起来。

```
你（用户 prompt）
    │
    ▼
cross-verified-feature-development  ← 本仓库
    │
    ├── 阶段 1   →  superpowers:brainstorming
    ├── 阶段 2   →  superpowers:writing-plans
    ├── 阶段 3   →  superpowers:subagent-driven-development
    ├── 阶段 4.1 →  superpowers:systematic-debugging
    ├── 阶段 4.2 →  superpowers:code-reviewer（冷上下文，不给设计文档）
    ├── 阶段 4.3–4.5 → dispatch 独立 agent（含领域专属 prompt 模板）
    └── 阶段 5   →  superpowers:writing-plans + subagent-driven-development
```

**没有 Superpowers 也能用** —— 每个阶段都有 fallback 模式，使用 Claude Code 内置工具完成。

---

## 前置条件

**1. Claude Code**
```bash
npm install -g @anthropic-ai/claude-code
```

**2. Superpowers 插件**（推荐安装）
```bash
claude mcp add --transport http superpowers https://superpowers.anthropic.com/mcp
```
首次使用需要 Anthropic 账号授权。

---

## 安装

```bash
git clone https://github.com/MageByte-Zero/magebyte-power.git
export SKILLS_REPO="$PWD/magebyte-power"

mkdir -p ~/.claude/skills

ln -sf "$SKILLS_REPO/skills/cross-verified-feature-development" \
       ~/.claude/skills/cross-verified-feature-development
```

验证安装：
```bash
ls ~/.claude/skills/cross-verified-feature-development/SKILL.md
```

---

## cross-verified-feature-development

多轮交叉验证的严谨特性开发方法论，在实施与合并之间插入 **4 轮独立验证**，发现并发竞态、幂等缺陷、跨服务契约破坏等 —— 这些 bug 恰恰是单一视角的 review 会系统性漏掉的。

### 为什么需要这个工作流

单一视角的 review 有结构性盲点。即使经验丰富的开发者也会：
- 被自己的设计思路"带偏"，默认自己的假设是对的
- 假设"应该幂等"的操作真的幂等（实际不是）
- 用顺序思维推导并发场景，看漏 race window
- 忘记其他服务在消费自己的共享表或 MQ topic

**独立视角产生独立信号。** 用 N 个互不知情的 reviewer 轮询同一特性，发现的 bug 集合接近**并集而不是重复集合**。这个工作流把这个洞察系统化：

> 设计 → 实施 → **多轮交叉验证** → 修复 → 谨慎简化 → 文档同步

### Superpowers 在各阶段的增强

| 阶段 | Superpowers skill | 增强内容 |
|------|------------------|---------|
| 1. 需求/设计 | `superpowers:brainstorming` | 把模糊诉求结构化为含不变式、失败模式、风险表格的完整 spec |
| 2. 实施计划 | `superpowers:writing-plans` | 把 spec 拆成有文件+行号、TDD 结构、上线策略的可执行 task 清单 |
| 3. 实施 | `superpowers:subagent-driven-development` | 每个 task 在 fresh context 的独立 subagent 中执行，消除累积偏差 |
| 4.1 自查 | `superpowers:systematic-debugging` | 以"假设存在 bug"的角度对整个 feature 结构化扫描 |
| 4.2 冷评审 ⭐ | `superpowers:code-reviewer` | Reviewer **不获得设计文档**，发现作者自己看不到的设计层漏洞 |
| 4.3–4.5（并行） | 自定义 agent prompt | 行为保持 diff、跨仓库影响扫描、业务不变式矩阵 |
| 5. 迭代修复 | `superpowers:writing-plans` + `subagent-driven-development` | 修复阶段与实施阶段同等严谨度 |

**4.2 冷上下文评审是价值最高的步骤。** 典型产出：5–15 个 Critical/High 级别的 bug，全部是标准 code review 会放过的问题。

### 适用场景

```
特性是否命中以下任一项？
│
├── 💰 资金流 / 支付 / 退款 / 结算？                    → 是 → 走本工作流
├── 🔄 订单 / 库存状态机，有状态转换？                   → 是 → 走本工作流
├── 🔒 分布式锁 / 并发控制 / 幂等重试？                  → 是 → 走本工作流
├── 🔗 跨服务 MQ/RPC 协议或共享 proto/model 变更？       → 是 → 走本工作流
├── 🗄️  在线 schema 迁移或双写切换？                     → 是 → 走本工作流
└── ⏱️  预估工作量 ≥ 3 人日，且失败代价高？              → 是 → 走本工作流

以上都不命中？ →  走普通工作流即可 ✓
```

**一句话判定**：如果"这个 feature 最坏的 bug 会怎样"的答案包含**资金损失 / 数据错乱 / 订单卡死 / 权限越权**，就值得走本工作流。

**不适用场景**：纯 UI / 纯展示调整、无状态机语义的简单 CRUD、一次性脚本、小 bug 修复、工作量 < 1 人日。

### 7 阶段总览

```
① 需求/设计        → superpowers:brainstorming
①.5 架构决策评审   → ADR（高风险特性必做）
② 实施计划         → superpowers:writing-plans
③ 实施            → superpowers:subagent-driven-development
④ 🔥 多轮交叉验证  ← 本工作流的核心创新（4 轮独立验证）
⑤ 迭代修复         → writing-plans + subagent-driven-development
⑥ 谨慎简化         → 带怀疑的优化 + anti-patterns 检查
⑦ 文档同步         → 回填 evolution log + 下游通知
```

每个阶段都有明确的 **Exit Criteria**（完成标准检查列表）。

### 4 轮验证详解

| 轮次 | 视角 | 典型产出 |
|------|------|---------|
| 4.1 系统调试（自查） | 自己扮演 bug 猎人 | 1–3 个已有 bug |
| 4.2 冷上下文评审 ⭐ | **不读设计文档**的独立 reviewer | 5–15 个并发 / 幂等 bug |
| 4.3 行为保持 diff | 对比 master vs feature 副作用 | 2–5 处语义回归 |
| 4.4 跨仓库影响扫描 | 识别其他仓库的联动影响 | 0–3 个外部影响点 |
| 4.5 业务不变式矩阵 | 资金/状态机/库存的硬约束验证 | 0–2 个不变式被破坏 |

4.3–4.5 可以并行 dispatch 多个 subagent，大幅节省时间。

### 触发方式

```
/cross-verified-workflow <feature 需求描述>
```

或直接描述高风险特性 —— 当 skill 检测到相关模式，会主动提出建议：

> 我注意到这个任务涉及 `<命中的领域>`，属于 bug 代价较高的场景。建议走 cross-verified 工作流（brainstorm → plan → implement → 多轮交叉验证 → 修复 → 文档回填），会比普通做法多花约 40–50% 时间，但能把 critical bug 发现率从 ~40% 提到 ~95%。要不要走这个流程？

### 内含参考文件

| 文件 | 何时读 |
|------|-------|
| `references/cross-verification-techniques.md` | Phase 4 开始前必读，内含每种验证的 agent prompt 模板 |
| `references/anti-patterns.md` | Phase 6（简化）前必读，12 个高频踩坑模式 |
| `references/doc-sync-playbook.md` | Phase 7 前必读，规范化文档回填流程 |
| `references/case-studies.md` | 选读，电商订单域的真实 bug 博物馆 |

### 成本与收益

| 指标 | 普通工作流 | 本工作流 |
|------|-----------|---------|
| 额外时间成本 | — | +40–50% |
| Critical bug 发现率 | ~40% | ~95% |
| 4.2 冷评审典型产出 | 0 | 5–15 个 High/Critical bug |

---

## 贡献新 Skill

本仓库的 skill 来自真实工程实践，最有价值的贡献是**在生产中验证过的工作流**。

1. 创建 `skills/<skill-name>/SKILL.md`，包含必要的 frontmatter：
   ```markdown
   ---
   name: <skill-name>
   description: <触发时机描述 —— 尽量具体，包含关键词和场景>
   ---
   ```
2. 把参考文件放在 `skills/<skill-name>/references/`
3. 在真实任务上测试后再提交 PR
4. PR 描述中说明领域背景和解决的问题

---

Maintainer: [MageByte-Zero](https://github.com/MageByte-Zero) · [MIT License](LICENSE)
