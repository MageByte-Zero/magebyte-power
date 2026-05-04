# Doc Sync Playbook

Phase 7 的详细执行指南。把最终代码状态回填到原设计文档，保证**设计文档 = 代码真实状态**。

---

## 为什么必须做

Phase 4-5 的修复**一定会让代码偏离原设计**。如果不同步，三个月后：

1. **新人 onboard**：读原设计文档 → 按文档写相关代码 → 和真实代码对不上 → 要么删文档（信息丢失）要么按错文档写错代码
2. **下一次迭代**：有人想加新特性，参考原设计 → 基于错误假设设计 → 重复踩已经踩过的坑
3. **失败的经验遗失**：Phase 6 回滚的简化尝试特别宝贵（知道"不能这样做"比"应该这样做"更重要），如果不记录就永远丢失

---

## 执行前：判断同步的必要性

勾选以下任意一项即必须做 Phase 7：
- [ ] Phase 5 修复了 **Critical** 或 **High** 级别的 issue（改动了核心流程）
- [ ] 实施过程中有 **3 处及以上**偏离原设计的决策
- [ ] Phase 6 **回滚过一次失败尝试**（即使原设计没改，也要记录教训）
- [ ] 原设计文档里某些章节现在**与代码明显不一致**

**只有** "代码完全按原 plan 实施 + 零偏离 + 零失败尝试" 的罕见情况可以跳过。

---

## 核心原则

### 1. 保留原始设计意图

**不要**：完全重写 spec、删掉原有章节、直接覆盖内容

**要**：
- 保留原始设计文本
- 在偏离处加注释：`> ⚠️ 原设计 vs 实际实现：...`
- 末尾追加 Evolution Log（见模板）

### 2. 记录决策理由，不只记结果

**不要**：`MODIFY: Lock TTL 60s → 300s`

**要**：
```markdown
| 差异 | 原设计 | 实际实现 | 理由 |
|------|--------|---------|------|
| 锁 TTL | 60s | 300s | Review R1 发现：60s 会在 doRefundUnit + updateAllDate + createNewFulfilmentUnit 组合的慢依赖下过早释放 |
```

"为什么"比"什么"重要 10 倍。

### 3. 失败的尝试**必须**记录

这个工作流最有价值的部分往往是"我们试过 X，发现 Y 问题，回滚了"。这种教训如果不记录：
- 下一个人会重新试一遍
- 浪费时间
- 可能这次侥幸没翻车，生产上爆炸

**格式**：
```markdown
### Phase 3: 失败的简化尝试与回滚

**尝试**（commits `XXX` + `YYY`）：<描述尝试做了什么>

**回滚**（commits `ZZZ`）：

<根本原因>

**教训**：<为什么这条路走不通>
```

### 4. 文档末尾加 Evolution Log

这是整个 doc sync 的核心产物。格式见下方 Template。

---

## Template：文档尾部追加的 Evolution Log

### Spec 文档（design doc）

追加到末尾，格式：

```markdown
---

## N. 实现演进记录 (Implementation Evolution Log)

> 本章节记录从原始设计到最终实现过程中的决策演进。
>
> **完整修复清单**：`docs/superpowers/plans/YYYY-MM-DD-<feature>-review-fixes.md`
> **集成验证手册**：`docs/superpowers/tests/YYYY-MM-DD-<feature>-verification-runbook.md`（如有）

### N.1 Phase 1：初始实现（YYYY-MM-DD）

严格按原设计 §X~§Y 实施了 M 个 task。主要 commits: `<first-SHA>` ... `<last-SHA>`。
结果在后续的 cold-context review 中发现 P 个并发/一致性/幂等漏洞。

### N.2 Phase 2：Review Fixes（YYYY-MM-DD）

针对 cross-verification 发现的问题实施的 Q 项关键修复：

| # | 改动 | 原问题 | 修复 commit | 文件/行号 |
|---|------|-------|-------------|----------|
| R1 | <简短描述> | <原问题描述> | `<SHA>` | `<file:line>` |
| R2 | ... | ... | ... | ... |
| ... | | | | |

### N.3 Phase 3：失败的简化尝试与回滚（如有）

**尝试**（commits `<AAA>` + `<BBB>`）：<描述>

**回滚**（commits `<CCC>` + `<DDD>`）：

<根本原因，尽量详细，最好引用代码/数据证据>

**教训**：<一两段文字讲清楚为什么这条路走不通>

### N.4 当前决策与原设计的差异索引

| 差异点 | 原设计（§）| 当前实现 | 决策理由 |
|--------|-----------|---------|---------|
| <项 1> | <原设计说法，含章节引用> | <当前实际实现> | <R# - 理由> |
| <项 2> | ... | ... | ... |

### N.5 新增的防御机制（原设计未涉及）

原设计没有涉及、但实施/review 中新增的功能：

- `<FuncName>`（R#）：<简述用途>
- ...
```

### Plan 文档

**2 处修改**：

#### 修改 1：在顶部插入"实施状态总览"表

放在文档头部的 `**Goal:** ...` 之后、`## File Structure` 之前：

```markdown
## 实施状态总览（YYYY-MM-DD 更新）

> M 个原 task 全部完成，之后又经过 cold-context review 实施了 Q 个 review fixes（见 "Part G" 章节）。

| Task | 描述 | 主要 commit | 实施状态 | 后续 |
|------|------|------------|---------|------|
| Task 1 | <原 task 标题> | `<SHA>` | ✅ 完全一致 / ⚠️ 偏离（理由）/ 🔼 强化（理由）| - / 见 Part G |
| Task 2 | ... | ... | ... | ... |
| ... | | | | |

**Part G**（见本文末尾）：post-implementation review fixes 的完整清单，含 Q 个修复 task 和对应 commit SHA。
```

#### 修改 2：在文档末尾追加 Part G

```markdown
---

## Part G: Post-implementation Review Fixes (YYYY-MM-DD)

在完成原 M 个 task 之后，项目又经过 cold-context 独立 code review（由不知道设计方案的 reviewer 从代码 diff 出发，独立发现问题）。review 发现了 P 个并发/一致性/幂等漏洞，实施了 Q 个修复 task。

**详细修复计划**：`docs/superpowers/plans/YYYY-MM-DD-<feature>-review-fixes.md`
**集成验证手册**：`docs/superpowers/tests/YYYY-MM-DD-<feature>-verification-runbook.md`（如有）
**设计决策演进**：Spec §N Implementation Evolution Log

### G.1 修复任务清单

| # | Task | 严重度 | 主要 commit | 状态 |
|---|------|-------|-------------|------|
| R1 | <改动描述> | 🔴 Critical / 🟠 High / 🟢 Low | `<SHA>` | ✅ |
| ... | | | | |

### G.2 失败的简化尝试与回滚（如有）

<与 Spec §N.3 相同的内容，或引用 Spec 的对应章节>

### G.3 需要跨团队确认的下游消费者（如有）

<Phase 4.4 产出的 open question>

### G.4 运维 SOP 变更（如有）

旧的做法失效了 / 新的做法：

- ❌ <旧 SOP>
- ✅ <新 SOP>
```

---

## 实际操作步骤

### 1. 准备阶段

```bash
# 从 master 拉到 feature 的所有 commit
cd <repo>
git log --oneline master..HEAD

# 对每个 commit 分类，标注属于 Phase 1/2/3
# 建议用一个临时文本文件整理
```

### 2. 扫描需要修改的 spec section

对 spec 文档逐 section 阅读，标注：
- ✅ 与实际代码一致的 section（不动）
- ⚠️ 有偏离的 section（加注释）
- 🆕 新功能 / 原 spec 未涉及（追加到 Evolution Log）

### 3. 修改 spec section（inline 注释）

对每个 ⚠️ section，**不要删原内容**，改为：

```markdown
### 3.1 Original section title

[原内容保留...]

> ⚠️ **原设计与实际实现的关键差异**（见 §N Implementation Evolution Log）：
> 1. <差异点 1>
> 2. <差异点 2>

### 实际实现

[描述当前实际代码]
```

### 4. 追加 Evolution Log

按 Template 追加到 spec 末尾。

### 5. 更新 plan 文档

按 Template 做 2 处修改（顶部状态总览 + 末尾 Part G）。

### 6. 验证

```bash
# 对每个 commit SHA 快速验证存在
for sha in <list>; do git show $sha --stat | head -3; done

# 检查文件引用路径是否正确
grep -E "docs/superpowers/(specs|plans|tests)/" <doc> | xargs -I{} ls {}
```

### 7. 下游通知（跨团队变更时必做）

如果 Phase 4.4 的 cross-repo scan 列出了 open question 或其他团队需要跟进的事项：

**不能只写进文档等人来读**——主动通知。

执行步骤：
1. 列出 Phase 4.4 report 里所有标注"需要 <某团队> 确认"的条目
2. 对每个条目，用对应渠道（Slack / Email / Ticket）发送通知，包含：
   - Feature 名称 + 分支名
   - 具体问题（直接引用 4.4 report 的文字，不要口头转述）
   - 你的初步判断（需要改 / 不需要改 / 不确定）
   - 需要对方在什么时间点前确认
3. 在 Evolution Log 的 §N.3 或独立章节记录"已通知哪些团队、日期、链接"

**什么时候可以跳过**：Phase 4.4 result 是 "No changes needed" 且 open question 为空。

### 8. Runbook / SOP 更新（运维操作有变化时）

如果 feature 改变了运维操作（任意一项命中即需要更新）：
- 新增了一张表需要定期清理
- 某个错误码的处理方式变了（oncall 需要知道）
- 新增/变更了定时任务
- 数据迁移期间有特殊操作步骤

**操作**：更新对应的 oncall runbook / SOP 文档，并在 Evolution Log 的 §N.4 章节里引用（写明"运维 SOP 变更见 <链接>"）。

### 9. Commit

**一个 commit，包含所有 doc 改动**，不要拆分：

```bash
git add docs/superpowers/
git commit -m "docs(<feature>): sync design doc and plan with actual implementation

Align the original design spec and implementation plan with what the code
actually does after the Phase 2 review fixes and Phase 3 rollback.

## Spec changes
[简述]

## Plan changes
[简述]

Refs:
- docs/superpowers/plans/<review-fixes-plan>.md
- docs/superpowers/tests/<verification-runbook>.md
"
```

---

## 反模式

### ❌ 反模式 1：Big Bang Rewrite

"反正设计文档和代码都对不上了，干脆重写整个 spec"。

**为什么不要**：
- 原始设计的思路有价值（记录了**当时**的考虑）
- 完全重写会丢失"为什么后来改了"的信息
- 读者以为新 spec 是从零设计的，不知道背后有过迭代

**正确做法**：保留原设计 + 追加 Evolution Log。

### ❌ 反模式 2：只更新文档不 commit 代码

文档和代码应该**同时**推进。Phase 7 单独一个 commit 是为了**回顾历史**——看 git log 能清楚看到"代码完成"之后有一次"文档同步"。

### ❌ 反模式 3：删除"失败的尝试"记录

"这个简化失败了，就不记录了，免得显得菜"。

**为什么不要**：
- 下一个人会重新试一遍
- 失败的教训是**最宝贵**的知识
- 承认失败是团队文化成熟的标志

**正确做法**：把失败详细记录在 Evolution Log 的 Phase 3 章节。**教训**要讲清楚。

### ❌ 反模式 4：引用具体实现细节到过细

"Line 523 的 if 是改的……"

**为什么不要**：
- 代码会继续演进，行号会变
- 读者不关心精确行号，关心**设计决策**
- 过细的引用反而让文档难维护

**正确做法**：引用到**函数名 / section 级别**。精确行号留给 code 本身。

---

## 验收检查

Phase 7 完成后，用下面的问题检查：

- [ ] 如果我**明天忘记**所有上下文，明天看 spec 文档能搞清楚当前代码为什么这样写吗？
- [ ] 所有 Critical / High 的 review fix 都能在 Evolution Log 里找到吗？
- [ ] 失败的简化尝试（如有）记录了根本原因和教训吗？
- [ ] Plan 文档每个 task 能定位到对应的 commit SHA 吗？
- [ ] 对跨团队 / 外部依赖的 open question：**已经主动通知相关团队**（不只是写进文档）？
- [ ] 运维 SOP 的变化（如有）**已更新对应的 runbook**，并在 Evolution Log 里引用？

全部 ✅ → Phase 7 完成。

---

## 产出物清单

最终 `docs/superpowers/` 下应该有：

```
docs/superpowers/
├── specs/
│   └── YYYY-MM-DD-<feature>-design.md    # 含 §N Implementation Evolution Log
├── plans/
│   ├── YYYY-MM-DD-<feature>.md           # 含顶部实施状态总览 + Part G
│   └── YYYY-MM-DD-<feature>-review-fixes.md   # Phase 5 的修复 plan
└── tests/
    └── YYYY-MM-DD-<feature>-verification-runbook.md   # 如有，集成验证手册
```
