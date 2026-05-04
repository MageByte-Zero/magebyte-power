---
name: cross-verified-feature-development
description: "Guides agents through a 7-phase rigorous feature development workflow with 4 independent cross-verification passes: systematic self-review, cold-context code review (reviewer sees no design docs — finds design-level blind spots single reviewers miss), behavior-preservation diff, and cross-repo impact scan.

Use this skill — and proactively suggest it even when not explicitly requested — whenever: financial transactions, payments, or refunds are involved; order/inventory state machines have status transitions; distributed locks, concurrency control, or idempotent retry logic is needed; cross-service MQ/RPC contracts or shared proto/model definitions change; online schema migration or dual-write strategy is required; estimated implementation effort ≥ 3 person-days with high cost-of-failure.

Trigger phrases: /cross-verified-workflow, \"cross-verified\", \"交叉验证\", \"严谨工作流\", \"幂等重试\", \"分布式锁\", \"状态机\", \"schema 迁移\", \"跨服务接口\"."
---

# Cross-Verified Feature Development

> A structured workflow for high-stakes backend features. After implementation, it runs 4 independent verification passes to catch bugs that self-review and standard code review miss — particularly concurrency races, idempotency gaps, and cross-service contract breaks.

## Prerequisites · 前置条件

本 skill 编排 [Superpowers](https://superpowers.anthropic.com/) 生态中的子 skill 完成各阶段工作。

**Install Superpowers（一次性）：**
```bash
claude mcp add --transport http superpowers https://superpowers.anthropic.com/mcp
```

**本 skill 依赖的子 skill：**

| Phase | Skill | 用途 |
|-------|-------|------|
| 1 | `superpowers:brainstorming` | 需求分析 → 结构化 spec |
| 2 | `superpowers:writing-plans` | spec → 可执行 task 清单 |
| 3 | `superpowers:subagent-driven-development` | 每个 task 独立 subagent 实施 |
| 4.1 | `superpowers:systematic-debugging` | 自查阶段的结构化调试框架 |
| 5 | `superpowers:writing-plans` + `superpowers:subagent-driven-development` | 修复迭代 |

**没有 Superpowers 也能用（fallback）：**

| Phase | Fallback 做法 |
|-------|-------------|
| Phase 1 | 手动撰写 spec 文档，确保包含：问题陈述、技术方案、不变式清单、失败模式分析、风险表格 |
| Phase 2 | 手动拆 task 清单，每个 task 须有文件 + 行号 + 验证命令 |
| Phase 3 | 逐 task 实施，每个 task 完成后做 self-review 再进下一个 |
| Phase 4.1 | 按 `references/cross-verification-techniques.md` 中的 4.1 checklist 手动自查 |
| Phase 4.2–4.5 | 直接使用 `references/cross-verification-techniques.md` 里的 agent prompt 模板 dispatch subagent |

## Overview · 概览

这是一套**多轮交叉验证的严谨特性开发方法论**，用于开发关键业务特性时**最小化生产事故概率**。

### Core Insight · 核心洞察

**单一视角的 review 有系统性盲点**。即使最有经验的开发者也会：
- 被自己的设计思路"带偏"
- 默认某些字段语义是唯一的（实际可能多义）
- 假设"应该幂等"的操作真的幂等（实际不是）
- 看漏并发窗口
- 忘记更新文档导致设计文档与代码失联

**独立视角产生的信号彼此独立**。用 N 个互不知情的 reviewer 轮询一个特性，发现的 bug 集合**接近是并集而不是重复**。所以这个工作流的本质是：

> **设计 → 实施 → 多轮独立交叉验证 → 修复 → 谨慎简化 → 文档同步**

### Comparison with Standard Workflow · 与普通开发工作流的差异

| 环节 | 普通做法 | 本工作流 |
|------|---------|---------|
| 需求 | 口头沟通/简单 PRD | `brainstorming` 形成完整 spec |
| 实施计划 | 开发者脑内规划 | `writing-plans` 显式化成 task 清单 |
| 实施 | 一人连续写完 | `subagent-driven-development`，每个 task 独立 review |
| 验证 | 自测 + 一次 code review | **4 轮独立交叉验证**（本 skill 的核心） |
| 修复 | 直接改 | 发现问题再走一轮 plan + execute |
| 简化/优化 | 直接删减代码 | **带怀疑的验证**，发现陷阱勇敢回滚 |
| 文档 | 实施后遗忘 | **强制回填**，记录 evolution log |

### When to Use · 适用场景

本工作流**值得的成本**：预估实施工作量 ≥ 3 人日、且失败代价高的特性。

**典型适用领域**（按"bug 代价高低"排序，任意一项命中即建议使用）：

| 领域 | 典型场景 |
|------|---------|
| **资金流 / 支付** | 收款、退款、结算、优惠券核销、余额变动 |
| **订单 / 交易状态机** | 订单状态推进、取消、改单、履约、售后 |
| **库存 / 库位** | 扣减、回滚、预占、跨仓调拨 |
| **权限 / 身份 / 合规** | 授权、鉴权、脱敏、审计链路 |
| **并发控制** | 分布式锁、乐观锁、CAS、幂等重试 |
| **跨服务协作** | 新增跨微服务接口、MQ 协议、异步消息链路 |
| **核心数据模型** | 共享 proto / model 变更、主键语义变更 |
| **数据迁移 / schema** | 在线 DDL、双写切换、回填、历史数据修复 |
| **配置/开关** | feature flag、灰度规则、影响资金或订单的配置变更 |

**不适用**：
- 纯 UI / 前端展示调整
- 纯 CRUD 无状态机语义
- 一次性数据处理脚本
- 修个小 bug
- 实施工作量 < 1 人日

**判定启发式**：如果你能用一句话回答"这个 feature 最坏的 bug 会造成什么？"并且答案包含**资金损失 / 数据错乱 / 订单卡死 / 用户权限越权 / 生产事故**之一，那就值得走本工作流。

**Decision tree · 决策树：**

```
Does the feature involve any of the following?
│
├── 💰 Financial transactions, payments, refunds, settlements?      ──→ YES → Use this workflow
├── 🔄 Order / inventory state machines with status transitions?    ──→ YES → Use this workflow
├── 🔒 Distributed locks, concurrency control, idempotent retry?    ──→ YES → Use this workflow
├── 🔗 Cross-service MQ/RPC contracts or shared proto/model change? ──→ YES → Use this workflow
├── 🗄️  Online schema migration or dual-write strategy?             ──→ YES → Use this workflow
└── ⏱️  Estimated effort ≥ 3 person-days?
    └── AND worst-case bug causes: money loss / data corruption /
        stuck orders / privilege escalation / production incident?  ──→ YES → Use this workflow

None of the above?  ──→  Standard workflow is fine ✓
```

---

## 7-Phase Workflow · 完整 7 阶段工作流

```
① 需求/设计        → superpowers:brainstorming
①.5 架构决策评审   → ADR（高风险特性必做）
② 实施计划        → superpowers:writing-plans（含部署策略）
③ 实施           → superpowers:subagent-driven-development
④ 🔥 多轮交叉验证  ← 本 skill 的核心创新
⑤ 迭代修复        → writing-plans round 2 + executing（含回归保护）
⑥ 谨慎简化        → 带怀疑的优化
⑦ 文档同步        → 回填 evolution log + 下游通知
```

### Phase 1: Requirements & Design · 需求/设计

**目标**：把模糊的业务诉求变成**结构化的设计文档**。

**怎么做**：调用 `superpowers:brainstorming`。产出一份 spec，通常保存在 `docs/superpowers/specs/YYYY-MM-DD-<feature>-design.md`。

**关键输出检查点**：spec 必须包含
- 问题陈述与业务边界
- 技术方案与**决策点**（每个决策都要有"为什么不选 B"的论证）
- 不变式清单（业务层硬约束）
- 失败模式分析（至少 4 种崩溃/失败场景）
- 风险表格
- 工作量评估

**反模式**：把 brainstorming 省掉，直接进实施。→ 后面的验证阶段会反复翻车。

**✅ Exit Criteria — Phase 1 完成标准：**
- [ ] Spec 文档已创建（`docs/superpowers/specs/YYYY-MM-DD-<feature>-design.md`）
- [ ] Spec 包含：问题陈述、技术方案、不变式清单、≥4 种失败模式、风险表格、工作量评估
- [ ] 每个决策点有"为什么不选备选方案"的论证
- [ ] 利益相关方已确认需求范围

### Phase 1.5: Architecture Decision Review · 架构决策评审（高风险特性必做）

**触发条件**：设计中涉及以下任一项时必做——分布式锁方案、幂等机制设计、状态机转换图、跨服务数据一致性方案、在线 schema 迁移策略、共享 proto/model 变更。

**目标**：在动代码之前，把**架构层面的关键决策**显式验证一遍。设计阶段发现错误的代价是讨论成本；Phase 4 发现设计本身有误的代价是推翻 + 重写。

**怎么做**：对 spec 中每个关键技术决策，写一份 **Architecture Decision Record (ADR)**，回答：

1. **选了什么**：具体方案（例：用 Redis + DB CAS 双保险实现幂等）
2. **备选方案是什么，为什么不选**（每个备选都要有论证，不能只说"不合适"）
3. **选定方案的失败模式**：列举 3 种最可能的失败场景（不是假设不会失败）
4. **不变式在设计层面能满足吗**：逐条检查 Phase 1 的不变式清单，确认设计有对应的保证机制

**典型输出**：1-3 个 ADR，追加到 spec 文档末尾（不是单独文件，避免碎片化）。

**何时可以跳过**：特性是纯新增接口、无并发控制要求、无状态机、无跨服务数据写入时可跳过。

**反模式**：把 ADR 写成"我们选了 X"然后只写 X 的优点。ADR 的价值在于**记录为什么不选备选方案**，这是日后维护者最需要的信息。

**✅ Exit Criteria — Phase 1.5 完成标准：**
- [ ] 每个关键技术决策（分布式锁 / 幂等机制 / 状态机 / 一致性方案 / schema 迁移）有对应 ADR
- [ ] 每个 ADR 回答：选了什么、为什么不选备选方案、选定方案的失败模式
- [ ] Phase 1 的所有不变式均已确认在架构层有对应保证机制

---

### Phase 2: Implementation Plan · 实施计划

**目标**：把 spec 拆成**可独立执行、可独立验证的 task 清单**。

**怎么做**：调用 `superpowers:writing-plans`。保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`。

**关键输出检查点**：
- 每个 task 有明确文件 + 行号 + 代码示例
- 依赖关系清晰
- 验证方式具体（build / lint / test 命令）
- commit message 模板
- TDD 结构（先写失败测试再实现）
- **上线策略**：全量 / feature flag / 灰度比例 / 双写切换窗口（任选其一，但必须明确）
- **回滚标准**：什么指标异常时触发回滚？谁来决定？（不能留到上线时再想）
- **新增监控/告警**：本 feature 上线后需要增加哪些 metric 和 alert？
- **运维 SOP 变化**：DBA/SRE 需要做什么特殊操作？（如有）

**反模式**：task 里写 "TODO"、"稍后处理"、"类似 Task N"。→ 执行时会卡住或生成不一致代码。上线策略留白 → Phase 3 实施时没有 feature flag hook → 临时改代码上线。

**✅ Exit Criteria — Phase 2 完成标准：**
- [ ] 每个 task 有明确文件路径 + 行号 + 代码示例
- [ ] 每个 task 有具体验证命令（`make build` / `make test` / lint 等）
- [ ] TDD 结构：失败测试先于实现写入 plan
- [ ] 上线策略已明确（全量 / feature flag / 灰度 / 双写切换）
- [ ] 回滚标准已定义：什么指标触发回滚、谁来决定
- [ ] 新增监控/告警已列入 plan

### Phase 3: Implementation · 实施

**目标**：按 plan 逐 task 落地，每个 task 有独立 review。

**怎么做**：调用 `superpowers:subagent-driven-development`。每个 task：
1. Dispatch implementer subagent（fresh context）
2. Implementer 自 review + commit
3. Dispatch spec compliance reviewer（验证是否建了要求的东西）
4. Dispatch code quality reviewer（验证代码质量）
5. 任何 reviewer 找到问题 → implementer 修 → re-review
6. 全通过 → 下一 task

**关键心态**：**每个 task 独立的 fresh subagent** 比一个大 context 连续写完**更不容易犯错**，因为没有累积偏见。

**反模式**：图省事把多个 task 合并给一个 subagent 写完 → 冗长 context → 关键约束被忘记。

**✅ Exit Criteria — Phase 3 完成标准：**
- [ ] Plan 中所有 task 标记完成，有对应 commit SHA
- [ ] 构建通过：`make build`（或项目等效命令）
- [ ] 测试通过：`make test`（或项目等效命令）
- [ ] 每个 task 经过 spec compliance review + code quality review
- [ ] 关键路径无遗留 TODO / stub

---

### Phase 4: 🔥 Cross-Verification Rounds · 多轮交叉验证（本 skill 核心）

Phase 3 结束后，代码**表面**已经能工作。但是**能编译 + 能过单测 ≠ 生产就绪**。

这一阶段用 **4 种独立视角** 轮流审查代码，每种视角提供的信号彼此独立。**详细操作见** `references/cross-verification-techniques.md`。

#### 4.1 Systematic Debugging（内部自查）

**视角**：以 `superpowers:systematic-debugging` 为框架，从"假如此时出 bug，会是什么 bug"角度扫描自己的代码。

**何时停**：没有发现新的 critical issue，且已检查过所有声称的不变式。

**典型产出**：发现 1-3 个 pre-existing bug（不是本次引入但顺路发现的）。

#### 4.2 Cold-Context Code Review（外部独立评审）⭐

**这是本工作流最高价值的验证**。启动一个 fresh reviewer agent，**不给它设计文档**，只给它：
- Feature 一句话目标
- 分支名 + 仓库路径
- "找 bug" 的 prompt

**关键约束**：`DO NOT read the design document`。这句话要写进 agent prompt。

**为什么有效**：设计文档代表"作者相信系统应该如何工作"。给 reviewer 看文档会让它被**作者的盲点感染**。Cold-context reviewer 只看代码实际做什么，反而能发现设计本身的漏洞。

**典型产出**：5-15 个 critical / high 级别的并发 / 一致性 / 幂等漏洞。

#### 4.3 Behavior-Preservation Diff（master vs feature 行为差异）

**视角**：枚举 master 分支的**所有副作用**（DB 写、MQ 发布、RPC 调用、缓存写、日志告警），与 feature 分支逐条对比，回答"是否改变了原有业务逻辑"。

**何时需要**：当 feature 包含对"已有流程的改造"而不仅仅是"加新功能"时必须做。

**典型产出**：发现 2-5 处副作用的顺序/语义变化，每一处都需要确认是预期的。

#### 4.4 Cross-Repo Impact Scan（跨仓库影响）

**视角**：识别本次 feature 是否影响**其他仓库**的代码行为，需不需要对应改动。

**典型检查维度**：
- MQ 消费者是否要改去重逻辑（feature 可能让重复消息变得更频繁）
- RPC 调用方是否要处理新错误码
- 共享 DB 表的读/写方是否要适应新状态
- 共享 proto / model 是否向后兼容

**典型产出**：识别 0-3 个其他仓库的影响点，大多数情况是"不需要改"但**需要确认**。

#### 4.5 Business Invariant 矩阵（资金/状态机/库存/权限场景必做）

**视角**：列出 feature 必须保持的**业务层硬约束**（例如"一个业务单据最多产生一次资金变动"、"retry_count 单调递增"、"状态只能前进不倒退"），逐条验证代码在所有路径下都保持。

**何时做**：命中以下任一项时**必做**，与 4.2 并列（不是可选）：
- 资金流 / 支付 / 退款 / 余额变动
- 订单 / 库存状态机（有明确的状态转换约束）
- 库存扣减 / 预占 / 归还
- 权限授予 / 鉴权链路

其余场景（纯新增接口、无状态机语义）可跳过。

**✅ Exit Criteria — Phase 4 完成标准：**
- [ ] 4.1 完成：并发点 / 失败模式 / 幂等性 / 状态机 / 边界均已检查；所有发现已记录
- [ ] 4.2 完成：cold-context reviewer 未获得设计文档；所有 Critical/High 问题已记录
- [ ] 4.3 完成（如适用）：所有副作用已标注 ✅/🟢/🟡/🟠/🔴/🆕/❌
- [ ] 4.4 完成（如适用）：所有跨仓库影响已确认或 flagged
- [ ] 4.5 完成（如适用）：所有不变式逐条验证通过
- [ ] 所有发现已汇总为 issue 表（来源 / 严重度 / 位置 / 修复方向），准备交给 Phase 5

---

### Phase 5: Fix Iteration · 迭代修复

**目标**：把 Phase 4 发现的问题修到干净。

**怎么做**：把所有发现的 issue 汇总成一个新的 plan（`docs/superpowers/plans/YYYY-MM-DD-<feature>-review-fixes.md`），再走一遍 `writing-plans` + `subagent-driven-development`。

**关键原则**：
- **按严重程度分批**：Critical → High → Medium → Low
- **每个修复独立 commit**，便于回滚
- **每个 Critical fix 必须有对应的自动化测试覆盖**——没有测试的 fix 不算完成，因为下次迭代极容易 regression
- **所有 fix 合入后必须重跑一次 4.2 cold-context review**（不能跳过）——修复本身可能引入新 bug；reviewer 可以把注意力集中在 diff 上，不需要重新审查全量代码
- **如果重跑 4.2 发现超过 2 个新 High 级别问题**，停下来评估：是继续打补丁，还是重新考虑设计方案

**常见量级**：冷评审找到 10+ 个问题 → 修复 plan 10-15 个 task → 3-5 人日。

**✅ Exit Criteria — Phase 5 完成标准：**
- [ ] 所有 Critical 问题已修复，每个修复有对应自动化测试
- [ ] 所有 High 问题已修复，或有明确的接受风险说明（含批准人）
- [ ] 重跑 4.2 cold-context review：新发现 High 级别问题 < 2 个
- [ ] 每个修复有独立 commit（便于单独回滚）
- [ ] 回归测试套件全部通过

---

### Phase 6: Careful Simplification · 谨慎简化

**触发点**：Phase 5 结束后，有时会发现代码"似乎有冗余"、"看起来可以简化"。

**⚠️ 核心纪律**：**简化前必须验证所有隐藏假设**。详见 `references/anti-patterns.md` 的"Simplification Trap"章节。

**最容易中招的陷阱**：

1. **多义字段陷阱**：以为某字段只存一种语义，实际另一分支存的是别的。**简化掉某个"看似冗余"的存储/缓存，结果数据错乱**。

2. **重新推导陷阱**：以为某值能从 DB 状态重建，实际有的值是局部变量快照，没有确定性 DB 来源。

3. **过度自信陷阱**：被 review 的代码跑过压测、通过了所有验证，开发者有"这代码我非常理解了"的感觉。这种时候最容易判断失误。

**正确做法**：
- 简化前**显式列出**你要去掉的代码**为什么**存在
- 对每个原因，找**代码证据**证明它"真的可以不要了"
- 无法证明的，**保留**
- 简化后**再跑一遍 cold-context review**（新视角）确认没有退化

**遇到翻车**：果断 `git revert`，把教训写进 evolution log，不要试图"再简化一次"。

**✅ Exit Criteria — Phase 6（如做了简化）：**
- [ ] `anti-patterns.md` Simplification Checklist 所有项已勾选（有代码证据，不是"感觉可以"）
- [ ] 简化后运行了新一轮 cold-context review，无新 High 级别问题
- [ ] 简化失败并回滚时：教训已写入 evolution log

---

### Phase 7: Doc Sync · 文档同步

**目标**：把**最终代码状态回填到原设计文档**。

**为什么重要**：
- Phase 5 的修复会让**代码偏离原设计**，如果不同步，文档会误导下一个维护者
- 每次偏离背后都有**决策理由**，这些理由是宝贵的知识资产，只在代码里找不到

**怎么做**：详见 `references/doc-sync-playbook.md`。核心原则：
1. **不要完全重写 spec**，保留原始设计意图
2. 在偏离处**标注"原设计 vs 实际实现"**
3. 文档末尾追加 **Implementation Evolution Log** 章节，按时间线记录 Phase 1/2/3...
4. Plan 文档在每个 task 标注实际 commit SHA + 偏离点
5. 失败的简化尝试（Phase 6 回滚）**一定要记录**，教训比成功更有价值

**不要**：

- 只 commit 代码，不 commit 文档更新
- 默认"代码就是文档"——半年后没人能从代码反推决策
- 删掉原设计内容"因为跟实际不一致"——那是历史，是教训

**✅ Exit Criteria — Phase 7 完成标准：**
- [ ] Spec 文档中每处偏离原设计的地方已标注 `> ⚠️ 原设计 vs 实际实现`
- [ ] Spec 末尾追加了 Implementation Evolution Log（按时间线记录 Phase 1→7 关键决策）
- [ ] 失败的简化尝试（如有）已记录在 evolution log
- [ ] Plan 文档中每个 task 有实际 commit SHA + 偏离点标注
- [ ] 下游团队已收到接口 / 协议 / MQ 变更通知（如有）

---

## Superpowers Integration · 与 Superpowers 生态的协作关系

本 skill **不重复造轮子**，只做编排：

| Phase | 使用的 skill |
|-------|------------|
| 1 | `superpowers:brainstorming` |
| 2 | `superpowers:writing-plans` |
| 3 | `superpowers:subagent-driven-development` |
| 4.1 | `superpowers:systematic-debugging` |
| 4.2-4.5 | 自主 dispatch agent（本 skill 提供 prompt 模板，见 `references/cross-verification-techniques.md`）|
| 5 | `superpowers:writing-plans` + `superpowers:subagent-driven-development` |
| 6 | 自主执行（附带 `anti-patterns.md` 警示）|
| 7 | 自主执行（附带 `doc-sync-playbook.md`）|

---

## Getting Started · 启动本工作流

### Trigger Modes · 两种触发方式

**A. 用户显式触发**（必须走）：用户说 `/cross-verified-workflow <feature 需求>`、"按交叉验证方式开发"、"走严谨工作流"、"用 cross-verified 工作流" 等。

**B. 主动识别并建议**：当用户的任务描述命中"适用场景"表中的任一领域（资金流、订单状态机、并发控制、数据迁移等），或用户提及"重构核心 X / 改造支付 / 幂等 / 分布式锁 / 跨服务新接口"等关键词，**即使用户没有显式要求**，也应主动提出：

> 我注意到这个任务涉及 <命中的领域>，属于 bug 代价较高的场景。我建议走一个更严谨的 cross-verified 工作流（brainstorm → plan → implement → 多轮交叉验证 → 修复 → 文档回填），会比普通做法多花约 40-50% 时间，但能把 critical bug 发现率从 ~40% 提到 ~95%。你要不要走这个流程？
>
> 或者如果你觉得成本太高，我们也可以走常规流程。

**不要硬性拉人走工作流**——告知价值和代价，让用户选择。但**不要默默跳过**让用户在高风险改动上裸奔。

### Startup Steps · 启动步骤

1. **第一步**：确认用户意图，如果需求还不够明确，先让用户澄清
2. **第二步**：评估是否真的适合（参考"适用场景"章节）——不适合的话建议走更简单的流程
3. **第三步**：如果合适，调用 Phase 1 (`superpowers:brainstorming`) 开始

### Progress Tracking · 进度追踪

每个 Phase 完成后，明确标注进度：

```
✅ Phase 1   (Brainstorming) → docs/superpowers/specs/<file>.md
✅ Phase 1.5 (Arch Pre-flight) → ADR 追加到 spec（或 N/A）
✅ Phase 2   (Planning) → docs/superpowers/plans/<file>.md
⏳ Phase 3   (Implementation) → 进行中 (5/12 tasks done)
⬜ Phase 4a  (Systematic Debugging)
⬜ Phase 4b  (Cold-Context Review)
⬜ Phase 4c  (Diff Audit + Cross-Repo + Invariant，并行)
⬜ Phase 5   (Fix iteration)
⬜ Phase 6   (Careful simplification)
⬜ Phase 7   (Doc sync + 下游通知)
```

---

## Common Rationalizations · 常见自我合理化

> These are the thoughts that mean **stop — you're about to skip something important**.
> 以下想法出现时，立刻停下——你正在为跳过关键步骤找理由。

| Rationalization · 合理化借口 | Reality · 现实 |
|---|---|
| "代码通过了所有测试，cold-context review 可以省" | 测试验证"是否实现了预期行为"，不验证"设计本身是否正确"。cold-context review 专门找**设计层面**的漏洞——这类漏洞正是作者最看不出来的，因为作者默认了自己的设计假设 |
| "这个改动很小，走全流程成本太高" | 历史上最贵的 bug 几乎都来自"看起来很小"的改动。改动小 ≠ 风险小；小改动往往触及核心路径的边界条件，而这些条件在 review 时最容易被忽视 |
| "我对这套代码非常熟悉，不需要自查了" | 熟悉会产生假设盲点。Phase 4.1 的价值**正因为你很熟悉**——你能更准确地列出所有隐性假设，然后逐条挑战它们。最危险的时刻就是最自信的时刻（见 anti-patterns Trap 3） |
| "Team 代码评审会发现问题的" | 标准 code review 的 reviewer 和作者共享同一套上下文和假设。cold-context review 的核心约束是**不给 reviewer 看设计文档**——发现的 bug 集合和 code review 几乎不重叠 |
| "这个 RPC 接口看起来是幂等的" | "看起来"和"是"之间有实际的代码差距。Phase 4.4 要求对每个 retry 路径的 RPC **读底层实现**，不是看接口签名。Case 4 就是信接口签名的代价 |
| "Phase 7 文档同步等有空再补" | 有空永远不会来。Phase 5 的修复已经让代码偏离了原设计；不同步文档就是在给下一个维护者埋雷——他会按错误的文档写代码，踩同一个坑的变体（见 Case 6） |
| "4 轮交叉验证太多了，挑 1-2 轮做就够" | 4 轮验证产生的 bug 信号**接近是并集而不是重复**。只做 4.1 找不到设计漏洞；只做 4.2 找不到跨仓库影响；省掉任何一轮都是系统性盲点，不是节省时间 |
| "修复完了，不需要重跑 cold-context review" | 修复本身可能引入新 bug。修复后的代码对 reviewer 是全新的，能提供完全独立的信号。Phase 5 明确要求：所有 fix 合入后**至少重跑一次 4.2** |

## Reference Files · 关键参考文件

本 skill 主体保持精炼，细节在 references：

| 文件 | 何时读 |
|------|-------|
| `references/cross-verification-techniques.md` | Phase 4 开始时必读，内含每种验证的 agent prompt 模板 |
| `references/anti-patterns.md` | Phase 6（简化）之前必读，避免踩坑 |
| `references/doc-sync-playbook.md` | Phase 7 开始时必读，规范化文档回填 |
| `references/case-studies.md` | 选读。真实项目中的踩坑案例合集，帮助理解为什么每条纪律存在 |

---

## FAQ · 常见问题

**Q：每个 Phase 都必须做吗？**

A：Phase 1-3 + 7 必做。Phase 4 至少做 4.1 和 4.2。Phase 5-6 按需。4.3-4.5 视特性复杂度。

**Q：Phase 4 的 4 轮验证要按顺序吗？**

A：4.1 建议最先（自查成本低），4.2（cold-context）紧跟。4.3-4.5 可以**并行 dispatch**（独立的 subagent），能显著节省时间。

**Q：发现新的一批 issue 时，Phase 4 要不要重新再来一遍？**

A：**修复后只需重点重验被改动的部分**。不需要每次改动都完整跑 4 轮。但是在最终 merge 前，建议至少再跑一次 4.2（cold-context）确认没有新引入问题。

**Q：这个工作流会不会太慢？**

A：一个中等复杂度特性（~5 人日实施）用本工作流总共约 **7-10 人日**。多出的 40-50% 时间换来的是：**Critical bug 发现率从典型的 40% 提升到 95%**。对于关键特性，这个 ROI 是压倒性的正收益。

**Q：前端/UI 特性能用这个工作流吗？**

A：不建议。纯展示、纯 CRUD、无状态机语义的 UI 改动不值得这个成本。只有当前端改动涉及**支付流程、订单确认、权限授权、关键数据提交**这类场景时才考虑。
