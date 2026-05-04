# 交叉验证技术详解

本文档展开讲 Phase 4 的 4+1 种验证技术，每种含**目的**、**agent prompt 模板**、**输出格式**、**反模式**。

**触发时机**：Phase 3（实施）结束，代码能编译通过、单测通过、self-review 完毕之后。

---

## Language Adaptation · 语言适配说明

The prompt templates below use language-neutral terminology where possible. When you encounter language-specific terms, substitute the equivalent for your stack using the table below.

下方 prompt 模板尽量使用语言中性的表述。遇到语言特定术语时，按下表替换为目标语言的对应概念：

| Generic concept · 通用概念 | Go | Java | Python | Node.js |
|---------------------------|-----|------|--------|---------|
| async task / coroutine | goroutine | CompletableFuture / Thread | asyncio.Task / threading.Thread | Promise / Worker |
| cleanup handler | defer | finally | try-finally / contextmanager | try-finally |
| cancellation signal | context cancel | InterruptedException / Future.cancel | asyncio.CancelledError | AbortController |
| message channel | chan | BlockingQueue / CompletableFuture | asyncio.Queue | EventEmitter |
| spawn async task | go func() | executor.submit() | asyncio.create_task() | setImmediate / queueMicrotask |
| rows affected by write | RowsAffected | executeUpdate() return value | cursor.rowcount | affectedRows |
| commit transaction | tx.Commit | connection.commit() | conn.commit() | await transaction.commit() |

---

## 4.1 Systematic Debugging（内部自查）

**目的**：以"假如此时出 bug，会是什么 bug"的角度扫描自己刚写的代码。与后续几轮外部 review 互补。

**何时做**：每次 Phase 3 后必做。成本极低（1 轮思考，不需要新 agent）。

**怎么做**：调用 `superpowers:systematic-debugging` skill，按其 4 个阶段（root cause → pattern analysis → hypothesis → implementation）走一遍**整个 feature**，不是单个 bug。

**关键问题清单**：
- 并发点：哪里有 check-then-act？哪里有 lost update？哪里的锁粒度不够？
- 失败模式：每个外部调用失败会怎样？DB 错误？MQ 错误？网络超时？
- 幂等性：每个**对外可观察的副作用**，retry 时会被重复执行吗？会破坏不变式吗？
- 状态机：所有 status 转换是否经过 FSM？是否有绕过 FSM 的写操作？
- 边界：nil 值、空 slice、0 值、超大值、非预期的类型？

**典型产出**：自己写的代码里**发现 1-3 个问题**（通常不是 critical 级别）。但这一步**不能替代** 4.2 的 cold-context review，只是补充。

---

## 4.2 Cold-Context Code Review（独立冷评审）⭐ 最高价值

**目的**：让一个**完全不知道设计思路**的 reviewer 从代码 diff 出发独立找 bug。这是本工作流发现**设计盲点**的主要手段。

**为什么有效**：设计文档代表"作者相信系统应该如何工作"。熟悉设计的 reviewer 会**默认作者的假设是对的**，从而看不到"这个假设本身就是错的"。Cold-context reviewer 只看代码实际做什么，反而能发现**设计层面的漏洞**。

**何时做**：Phase 3 后**必做**。这是本工作流最高 ROI 的一步。

**怎么做**：用 `Agent` 工具 dispatch 一个 `superpowers:code-reviewer`（或 `general-purpose`），**严格不提供**：
- ❌ 设计文档路径
- ❌ 实施计划
- ❌ 你对代码的理解或总结
- ❌ "我觉得 XX 已经覆盖了" 这类导向性说法

**只提供**：
- ✅ 分支名 + 仓库路径
- ✅ Feature 目标一句话
- ✅ "找 bug" 的明确 prompt

### Agent Prompt 模板（通用）

```
你是一个独立的 senior <语言> code reviewer。你**第一次**看到这个改动，没有参与任何前期设计讨论。

## 背景（只告诉你这些）

这是一个 <跨 N 个仓库 / 单仓库> 的 feature，分支名：`<branch-name>`

仓库：
- `<path-1>` <一句话描述职责>
- `<path-2>` <一句话描述职责>

Feature 目标（一句话）：<1-2 句话，只说做什么，不说怎么做>

## 你的任务

**不要信任作者的设计文档**（不要去读 `docs/` 下相关的任何设计/plan 文件，它们可能有设计盲点）。你的价值就是**独立**地从代码中找 bug。

请重点审查以下几类问题（按严重程度降序）：

### 1. 并发 / 竞态条件
- 分布式锁的 TTL、释放时机、异常/panic 场景、cleanup handler 注册时机
- check-then-act 模式是否有 race window
- async task / goroutine / thread 逃逸、cancellation signal 传递、资源清理
- 多 pod / replicas 场景下任务抢占是否 atomic

### 2. 数据一致性
- 数据库事务边界是否正确，事务里是否夹杂了外部调用
- 外部调用（RPC/MQ）和 DB 写的顺序（尤其是失败回滚时）
- 重试路径下 ID 生成、UUID、时间戳是否确定性（避免重试产生新 ID）
- 双写 / 迁移场景下新旧数据源的一致性保证

### 3. 幂等性实现缺陷
- 哪些操作不是幂等的但被假设成幂等（尤其是第三方 RPC）
- 部分失败（partial failure）的恢复逻辑
- 状态机转换的覆盖是否完整（有没有遗漏状态、能否从任一失败点 resume）
- 重复消息 / 重复请求 的去重键是否稳定

### 4. 错误处理
- 错误是否被吞掉（log 了但没返回/没上抛）
- cleanup handler（defer / finally / using）执行顺序，unhandled exception 下是否仍然执行
- 返回值和错误的一致性（error 非空但 result 同时非零值，或反之）
- 超时 / cancel 语义是否正确传递

### 5. 业务逻辑边界
- 边界条件（空 slice、nil map、0 值、负数、超大值）
- 业务线隔离（改动是否影响其他业务线 / 租户）
- 向后兼容性（字段删除、接口签名变化、enum 新值）
- 权限 / 鉴权链路是否完整

### 6. 数据访问性能
- 新增查询是否有合适的索引？（大致估算数据量 × 查询频率，有没有全表扫描风险）
- 是否有 N+1 查询（循环体内发 SQL / RPC）？
- 事务里是否夹杂了慢依赖（外部 RPC、文件 IO），导致行锁持有时间过长？
- 分页边界：大数据量时 offset 是否退化？是否应用游标分页？

### 7. 可观测性
- 关键业务路径是否有 metric 打点（成功/失败计数、延迟直方图）？
- 失败路径是否有足够的结构化日志（错误原因、trace ID、关联业务单号）？
- 能否静默失败而不触发任何告警？（这类路径是最难排查的）
- 是否有 distributed trace span 创建（如果项目使用 tracing）？

### 8. 安全与权限
- 新接口是否验证了调用方的身份 / 权限？
- 是否有 IDOR 风险（用户能通过猜测 ID 操作他人的数据）？
- 外部输入是否有适当的 validation（防注入、防超大值、防类型混淆）？
- 敏感数据（手机号、证件号、支付信息）是否在日志 / 响应中被脱敏？

## 输出要求

列出你找到的所有问题，每个问题：
- **严重程度**：Critical（会导致数据损坏/资金损失/用户越权）/ High（功能不可用）/ Medium（边缘情况 bug）/ Low（代码质量）
- **文件:行号**
- **问题描述**：具体是什么 bug，什么情况下触发
- **证据**：引用相关代码片段
- **修复建议**

如果某个方面你**没有发现问题**，也请明确说"已检查 X，未发现问题"，不要只列 bug 不说检查范围。

起点建议：
1. 先 `git log --oneline <base>..<branch-name>` 看提交历史
2. 然后 `git diff <base>...<branch-name> --stat` 看变更面
3. 再逐文件读代码

请**深度**审查，不要只看表面。
```

### 典型产出

| 质量指标 | 范围 |
|---------|------|
| 问题总数 | 5-20 |
| Critical | 0-6 |
| High | 2-8 |
| Medium/Low | 3-10 |

**如果 reviewer 只找到 0-1 个 critical 和 <3 个 high**，通常说明：
- feature 确实很干净（可能性小）
- reviewer prompt 不够严格（最常见）
- 给了过多设计文档信息污染（检查你的 prompt）

**如果 reviewer 找到 10+ critical**，通常说明：
- 原设计有较大问题
- 也可能是 reviewer 过于激进（但宁可错杀不可错放，认真逐条看）

---

## 4.3 Behavior-Preservation Diff（master vs feature 行为审计）

**目的**：枚举 master 分支的所有副作用，与 feature 分支逐条对比，回答"**重构/幂等化改造是否改变了原有业务逻辑**"。

**何时做**：当 feature 涉及"**对已有流程的改造**"时必做。纯新增接口时可跳过。

**怎么做**：dispatch 一个 agent，让它**双重阅读**（master + feature），生成 diff 表。

### Agent Prompt 模板（通用）

```
你在做 Behavior-Preservation Diff Audit —— 对比 master vs feature 分支，验证 <feature 名称> 是否改变了原有业务逻辑。

## 仓库

- `<path>` branch `<feature-branch>` vs `<base-branch, e.g. master>`

## 入口

<Feature 的 main entry function / handler，例如：HandleOrderRefund at refund_service.go>

## 任务

构造一个 **Side-Effect Diff Table**，对比：
- **Baseline**：`<base-branch>` 的行为（改造前做什么）
- **Feature Path A**：主路径（新流程做什么）
- **Feature Path B**：重试 / 补偿 / 变体路径（如果有）

对每条副作用标记：
- ✅ Equivalent — 完全不变
- 🟢 Additive — 新增但非破坏性
- 🟡 Reordered — 顺序变了，需核对下游是否敏感
- 🟠 Guarded — retry 路径有条件跳过
- 🔴 Changed — 语义变化，高风险
- 🆕 New — baseline 没有的副作用
- ❌ Removed — baseline 有但 feature 没有

### 什么算副作用

1. **DB writes** — 每张表的 INSERT/UPDATE/DELETE（精确到字段）
2. **MQ publishes** — 每个 publish 调用（topic + payload shape + 触发条件）
3. **RPC calls** — 每个外部服务调用（含调用方的错误处理语义）
4. **Cache operations** — SET/DEL/lock acquire
5. **Log / Alert events** — ops-observable 级别的告警或审计日志
6. **定时任务 / 后台 async task（goroutine / thread / Promise）的启动**

### 具体操作

1. 获取 baseline：
   ```
   git show <base>:<entry-file> > /tmp/baseline_<file>
   git show <base>:<dep-files...> > /tmp/baseline_*
   ```

2. 读当前 HEAD（feature 分支）的对应文件。

3. 对每个副作用，填表。不确定的打 ❓ 并列出具体疑问。

## 输出要求

生成一份完整的 markdown report：
1. 执行摘要
2. Side-Effect Diff Table（详细到字段级）
3. 每个 🔴 / 🟠 / 🆕 / ❌ 的 flagged concern，含：
   - 具体变化
   - 风险描述（谁会受影响）
   - 证据 (file:line)
   - 修复/确认建议
4. Areas Verified Equivalent —— 明确列出"没变"的部分作为 positive evidence
5. Open Questions —— 你判断不了、需要团队确认的点

## 约束

- 不分析 <用户指定跳过的深层依赖> 的内部实现，只标注调用点
- 不跨仓库（另有专门的 cross-repo scan）
- file:line 引用必须准确
```

### 典型产出

| 类别 | 典型数量 |
|------|---------|
| ✅ Equivalent | 10-20 |
| 🟢 Additive | 3-8 |
| 🟡 Reordered | 1-5 |
| 🟠 Guarded | 2-6 |
| 🔴 Changed | 0-3 |
| 🆕 New | 1-5 |
| ❌ Removed | 0-2 |

**🔴 和 ❌ 是最高优先级**，必须逐条确认是"预期的改动"还是"意外的 regression"。

---

## 4.4 Cross-Repo Impact Scan（跨仓库影响扫描）

**目的**：识别本次 feature 是否影响**其他仓库**的代码行为，需不需要对应改动。

**何时做**：当代码库是**微服务架构**、代码有**跨仓库依赖**时必做。单仓库项目可跳过。

**怎么做**：dispatch agent 扫描所有相关仓库，按 5 个维度检查。

### Agent Prompt 模板（通用）

```
你在做 Cross-Repository Impact Analysis —— 判断 <feature 名称> 是否需要修改其他仓库的代码。

## 本次 feature 变更范围

- `<path-1>` 分支 `<branch>`
- `<path-2>` 分支 `<branch>`

## 待评估仓库

- `<other-repo-1>` 含子服务 <X/Y/Z>
- `<other-repo-2>` 含子服务 <A/B>

## 评估维度

### 轴 1：MQ 消费者分析

列出本次 feature 可能发布 / 改动的所有 topic（grep `topic.go` / 配置文件 / 代码中的 publish 调用）。
对每个 topic，在待评估仓库中搜索消费者：
- 文件 + 函数名
- 收到消息做什么
- **是否按 <业务键> 幂等？** 去重键是否和本次 feature 发出的 payload 稳定一致？
- retry 路径下收到**重复消息**会破坏什么？（重复扣款 / 重复发货 / 重复发券等）

### 轴 2：RPC 入站/出站

本次 feature 调用哪些外部服务 RPC？这些 RPC 的服务端实现在哪？
- 服务端实现位置
- 服务端的幂等契约（读 proto 注释 + 实际 SQL/逻辑）
- retry 场景下会被调用 N 次，是否安全？
- 反过来：本 feature 提供的 RPC，调用方是否需要处理新错误码 / 新字段？

### 轴 3：共享 DB 表

待评估仓库是否读/写本次 feature 涉及的 DB 表？
- 如果读：新 status / 新状态是否会让读方出错或跳过？
- 如果写：并发写会不会冲突？新增字段默认值对旧写入方是否安全？

### 轴 4：共享 models / protos

本次 feature 是否修改了共享的 proto / model 定义？
- 向后兼容？（参考 anti-patterns 陷阱 11）
- 字段删除 / 语义变化？
- 新增 enum 值，老消费者会 panic / default 还是忽略？

### 轴 5：反向依赖 / 事件监听

待评估仓库的代码是否 LISTEN 本 feature 的完成事件（比如"订单完成时发奖励 / 退款成功时发通知"）？
retry / 幂等化 场景下事件可能重复触发，会 double-fire 吗？

## 输出要求

生成 markdown report：
1. 执行摘要：其他仓库**需不需要**改动？（YES / NO / PARTIAL）
2. MQ 消费者清单（表格：topic / 消费者 / 幂等性 / 风险）
3. RPC 入站分析
4. DB 表访问
5. 必需的改动（如有）
6. 建议 review 的点（不紧急但值得 review）
7. No-Change Confirmations（明确列出已验证无需改动的点，作为 positive evidence）

## 约束

- 用多种 grep pattern 交叉搜索，不要漏（同一个消费者可能有多种命名）
- file:line 引用必须准确
- 跳过 <用户指定跳过的专题>
- 如果无法判断幂等性，flag "需要 <某团队> 确认"
```

### 典型产出

- 大多数情况：**No changes needed**，但会列出 3-10 个值得跨团队确认的点（MQ 去重等）
- 偶尔发现：某共享服务需要改 1-3 个 MQ 消费者的去重逻辑

### Contract Testing 补充（RPC 接口变更时）

对所有 🟡/🟠/🔴 的 RPC/proto 变更，静态分析不够，还需要确认：

1. **序列化兼容性**：新旧版本能否互相正确序列化/反序列化？（proto 字段删除/改类型/改 required 都是高风险）
2. **部署顺序约束**：哪个服务必须先部署？（通常服务端先升级至兼容新旧请求，再升客户端）
3. **降级回滚路径**：如果新版服务端需要回滚到旧版，客户端还能正常工作吗？
4. **新 enum 值兼容性**：老版本服务端收到新 enum 值时会 panic / default / 忽略 —— 哪种行为？是否可接受？

**输出要求**：对每个存在 RPC 变更的服务，在 report 里补充一个"部署顺序"和"兼容性风险"章节。

---

## 4.5 Business Invariant 检查（资金/状态机/库存/权限场景必做）

**目的**：显式列出 feature 必须保持的**业务硬约束**，逐条验证代码在所有路径下保持。

**何时做**：命中资金流 / 状态机 / 库存扣减 / 权限授予之一时**必做**（与 4.1/4.2 并列，非可选）。其余场景可跳过。

**怎么做**：

1. 列出不变式清单。**通用模板**（按领域）：
    - **资金流**："每个业务单据最多产生一次资金变动"、"退款金额 ≤ 原支付金额"、"账户余额 ≥ 0"
    - **状态机**："status 只能前进不倒退"、"终态不可回退"、"所有状态转换必须经过 FSM 校验"
    - **库存**："可售库存 ≥ 0"、"预占 + 已售 + 可售 = 总量"
    - **幂等**："同一 idempotency_key 最多执行一次副作用"、"retry_count 单调递增"
    - **互斥**："同一业务单据期间只能有一个处理者（锁唯一）"
    - **权限**："用户只能访问自己的数据"、"只有 owner 能修改"

2. 对每条：
    - baseline 分支是怎么保证的？（定位代码）
    - feature 分支这段代码是否仍然有效？
    - **retry / 补偿路径**是否新增了破坏不变式的窗口？

### Agent Prompt 模板

```
审查 <feature> 在 <repo> 分支 `<branch>` 的代码，对以下业务不变式逐条验证是否保持：

## 不变式清单

1. <用户提供的不变式 1>
2. <用户提供的不变式 2>
...

## 对每条不变式

- 定位 baseline 分支里保证它的代码
- 检查 feature 分支该代码是否仍然有效
- 检查 feature 的 **retry / 错误恢复 / 补偿路径** 是否新增了破坏不变式的窗口
- 给出判断：**保持 / 破坏 / 需要核实**
- 如破坏：指出具体场景（reproduce steps）

## 输出

每条不变式一个 section，结论 + 证据 + （如破坏）修复建议。
```

### 典型产出

- 绝大多数不变式**保持**
- 偶尔发现 1-2 个因 retry 引入的窗口期（比如"资源释放失败后永久锁死"这类）

---

## 并行 vs 串行

**推荐顺序**：
1. 4.1 先做（最便宜，帮你自己清醒一下）
2. 4.2 紧跟（最高价值，找设计盲点）
3. 4.3 + 4.4 + 4.5 **并行 dispatch**（独立 subagent，互不干扰），大幅节省时间

**注意**：4.2 产出的 issue list 会影响 Phase 5 fix plan。所以**先跑完 4.2 再决定 4.3-4.5 的深度**：如果 4.2 已经揭示严重设计问题，先修完再做 4.3-4.5 更合理。

---

## 产出后：汇总到 Review Fix Plan

所有 4+1 轮验证结束后，把**全部发现的 issue**汇总成一张表：

| # | 来源 | 严重度 | 位置 | 问题 | 修复方向 |
|---|------|-------|------|------|---------|
| 1 | 4.2 cold-context | 🔴 Critical | service.go:XXX | 并发 race window | 锁提到入口 + CAS |
| 2 | 4.3 diff audit | 🟠 High | cleanup.go:YYY | 资源释放顺序变了导致永久锁死 | 重排终态标记到最后 |
| ... | | | | | |

然后把这张表交给 Phase 5 — 用 `superpowers:writing-plans` 规划修复 task，用 `superpowers:subagent-driven-development` 实施。
