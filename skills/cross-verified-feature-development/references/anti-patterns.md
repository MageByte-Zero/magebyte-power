# 反模式与陷阱

这里记录在本工作流中**最容易翻车的场景**。每条都来自真实项目的翻车经历——读它是为了在 Phase 6（简化）或 Phase 4（验证）时**提前识别**。

> 本文保持领域中立，用通用伪代码举例。具体项目的真实案例见 `case-studies.md`。

---

## 🚨 陷阱 1：多义字段陷阱（Polymorphic Field Trap）

**症状**：在 Phase 6 简化代码时，你打算"把这个缓存的值换成从 DB 重算"，觉得"反正值都能从 DB 字段拿到，缓存是冗余的"。

**通用模式**：某张表有一对字段 `(ref_type, ref_id)`，`ref_id` 的语义**依赖 `ref_type` 的值**：

```
ref_type = "refund"   → ref_id 存退款单号
ref_type = "payment"  → ref_id 存支付单号
ref_type = "voucher"  → ref_id 存凭证号
...
```

同一个列在不同分支/场景下被**不同业务流**写入不同语义的值。

**翻车场景**：开发者只看过主路径（比如 `ref_type=refund`），以为 `ref_id` 就是退款单号；于是在简化时把某个缓存 JSON 去掉，改成"从 `ref_id` 字段直接取"——结果**少补/冲正等支路**下，返回了错误语义的 ID 给调用方，下游用错 ID 去查询引发连锁故障。

**识别方法**：简化前对每个"看似冗余"的字段/缓存/变量问：

1. 这个值**源头**是什么？（某个 RPC 返回值？某个计算结果？某个外部输入？）
2. 如果用 DB 字段代替，那个 DB 字段**还可能被谁写**？（grep 整个代码库的 UPDATE/INSERT 点）
3. 那个 DB 字段的**类型语义**是唯一的，还是依赖另一个字段（例如 `type` 列）？
4. 如果你**无法确定**某个写入路径的语义，**就不要简化**。

**修复纪律**：发现翻车后**立即 `git revert`**，不要试图"打补丁救回来"。把教训写进 evolution log。

---

## 🚨 陷阱 2：Context 污染（Context-Biased Review）

**症状**：你 dispatch 一个 reviewer agent，给它看完整的设计文档、你的实施思路、你的自测结果，然后 reviewer 只找到几个小 bug 就说"看起来都合理"。

**踩坑原因**：设计文档代表你的**信念系统**。给 reviewer 看文档 = 让它**被你的盲点感染**。它会把文档里的假设当成前提，然后在前提之上找问题——找出来的也是**表面问题**，不是**设计层面的漏洞**。

**正确做法（Phase 4.2）**：
- ❌ 不给：设计文档、实施计划、你的 review 笔记、你的理解总结
- ✅ 只给：分支名、仓库路径、Feature 一句话目标、"找 bug" 的 prompt
- ✅ prompt 里明确写：`DO NOT read the design document. Your value comes from NOT knowing what the author intended.`

**极限用法**：如果你对某个设计决策**特别自信**，专门问 cold-context reviewer："这段代码做了 X。假设这是错的，最可能错在哪？" 强制它从反方向找。

---

## 🚨 陷阱 3：Simplification Hubris（过度自信的简化）

**症状**：Phase 4 和 Phase 5 完成后，你对代码非常熟悉，有"这代码我完全懂了"的感觉。这时看到某段"看起来多余"的代码，就想删掉。

**踩坑原因**：**最危险的时刻就是你感觉最自信的时候**。Phase 4 验证的是"当前代码正确"，不是"某一段可以去掉"。这是两个完全不同的命题。

**识别信号**：
- "这个成功结果缓存好像是多余的"
- "这个前置检查应该可以去掉"
- "这个字段看起来冗余"
- "这个 defer / finally 不需要了吧"

**纪律**：简化前强制走**4 步检查清单**（见下一节）。

---

### Simplification Checklist（4 步）

**Step 1：列出这段代码为什么存在**
- 读 git blame，找到它被加入时的 commit message
- 找到对应的 issue / 文档（如果有）
- 问自己：**没有这段代码，什么场景会出错？**

**Step 2：验证每个"原因"是否仍然有效**
- 对每个你识别的原因，**找代码证据**证明它现在"真的不需要了"
- 如果某个原因你**无法证伪**，**保留**

**Step 3：考虑并发/异步/边界情况**
- 简化后的代码在**失败路径**、**并发路径**、**retry 路径**、**边界值**下仍然正确吗？
- 不是"正常情况下能 work"，是"所有情况下都 work"

**Step 4：简化后再跑一轮 cold-context review**
- 新状态对 reviewer 是"fresh"的，能提供独立信号
- 如果 reviewer 找到新问题，**果断回滚**不要打补丁

---

## 🚨 陷阱 4：Doc Drift（文档漂移）

**症状**：几个月后，有人 onboard 读你的设计文档，然后按文档写代码——发现文档说的 API 和实际代码对不上。要么他直接删文档（信息丢失），要么他按错文档写了错的代码（引入 bug）。

**根本原因**：Phase 5 的修复会**让代码偏离原设计**。如果不同步文档：
- 代码的"为什么这么做"知识消失
- 新人无法理解决策背后的 trade-off
- 下一个维护者会**重复犯你踩过的坑**

**纪律**：Phase 7 **强制执行**，不能跳过：
- 在每个偏离处标注 `> ⚠️ 原设计 vs 实际实现`
- 末尾追加 **Implementation Evolution Log**
- **失败的简化尝试也要记录**（教训比成功更有价值）

详见 `doc-sync-playbook.md`。

---

## 🚨 陷阱 5：锁粒度不足（Lock Scope Gap）

**症状**：设计时加了分布式锁，某种并发 bug 依然发生。

**通用模式**：开发者只在"retry 入口"加锁，认为"首次执行不会并发"。但实际上：

```
  ┌─────────────────────────────┐
  │ 并发 req A ──┐              │
  │             ├─→ 都读到初始状态 PENDING
  │ 并发 req B ──┘              │
  │                             │
  │ 都通过无条件 UPDATE          │
  │ SET status='processing'     │
  │ WHERE id=?                  │
  │                             │
  │ 都转到 processing 状态       │
  │ 各自执行全量副作用 ←─── 重复! │
  └─────────────────────────────┘
```

**翻车根因**：`UPDATE ... WHERE id=?` 没有带**预期的前置状态**，两个并发请求都"抢成功"。

**识别方法**：对每个锁问：

1. **锁覆盖哪些状态转换？** 如果某个状态也会并发（哪怕你"觉得不会"），也要覆盖。
2. **锁的 TTL 能覆盖最坏情况吗？** 业务逻辑内部慢依赖加起来可能 > 锁 TTL，锁过期后第二个请求进入。
3. **锁失效（Redis 故障）时有 fallback 吗？** DB 层的 CAS 作为 defense-in-depth 是标配。
4. **锁释放时机正确吗？** `defer unlock()` 必须在**成功获取锁之后**才注册，否则释放别人的锁。

**修复 pattern**：
- 把锁提到**入口层**，覆盖**所有会并发的状态**，不要只守 retry 口
- TTL 选 `max(单次处理耗时) × 3` 的保守值
- DB 层用**带前置状态的 CAS**作为第二道防线：
  ```sql
  UPDATE t SET status='processing' WHERE id=? AND status='pending'
  -- 检查 RowsAffected=1 才是真的"第一次成功"
  ```
- `defer unlock()` 只在 `IsDuplicate == false` 的成功分支之后注册

---

## 🚨 陷阱 6：Final-State Marker 过早（Terminal-State-Before-Cleanup Ordering）

**症状**：某些"清理"步骤（释放锁、关联记录更新、下游通知）失败时，系统进入永久卡住状态，不能靠重试自愈。

**通用模式**：

```
  ❌ 错误顺序：
      FINISH(final state) → cleanup_step_1 → cleanup_step_2

  场景：cleanup_step_2 失败
  结果：status 已经 FINISH
       → 下次 retry 命中 "status==FINISH 直接返回缓存" 短路
       → cleanup_step_2 永远不会被重试
       → 相关资源永久卡住
```

**正确做法（Final-state-last ordering）**：

```
  ✅ 正确顺序：
      cleanup_step_1 → cleanup_step_2 → persist_cache → FINISH
```

这样 FINISH 是**整体完成的 checkpoint**。FINISH 之前任何失败 → status 保持中间状态 → retry 走 Path B → 再次尝试清理步骤 → 最终成功才 FINISH。

**通用原则**：

> **"终态标记" 必须是最后一步**，不能早于任何可能失败的清理操作。retry 要能从任何中间失败点继续推进。

---

## 🚨 陷阱 7：假定 RPC 幂等（Assumed RPC Idempotency）

**症状**：你写 retry 代码，调用某个已有 RPC，假设它"应该是幂等的"，但从没验证过。

**通用模式**：
```sql
-- 某个"解锁"/"归还"/"释放资源"的 RPC 底层：
UPDATE resource SET owner=NULL WHERE owner=? AND status='locked'

-- 第二次调用时 status 已经不是 'locked'
-- → 返回 RowsAffected=0
-- → RPC 把 RowsAffected=0 当成错误返回
-- → 调用方误以为"解锁失败"
```

不是真正的幂等——第一次成功后，第二次的行为和第一次的**返回语义**不一样。

**识别方法**：对每个 retry 路径调用的 RPC / SQL：
- 读它的**底层实现**，不是看接口签名
- 特别检查：
  - **RowsAffected 检查**——第二次 affected=0 会 error 吗
  - **WHERE 子句**——是否依赖"第一次"的前置状态
  - **唯一约束**——INSERT 在第二次会 dup-key 吗
  - **外部副作用**——第二次调用会重发消息/通知吗

**修复 pattern**：
- 无法改 RPC 内部？在调用方做**预检查**：先读状态，只在需要时调用
- 能改？让 RPC 天然幂等：
  - `INSERT ... ON DUPLICATE KEY UPDATE`
  - `UPDATE ... WHERE status IN (expected_pre, expected_post)`（已到终态也算成功）
  - 返回值区分 "acted"/"already-done" 两种成功

---

## 🚨 陷阱 8：MQ 重发 + 下游未去重

**症状**：Retry 路径重新走一遍流程，重新发出相同 MQ 消息。你假设下游会按业务键去重。结果下游重复处理，造成事故（重复扣款、重复发货、重复发券、重复计积分）。

**识别方法**：Phase 4.4（cross-repo scan）时对每个 MQ topic 专门问：
- 本 feature 的 retry 路径会 republish 这个 topic 吗？
- 下游消费者是什么？（grep all repos）
- 消费者**显式去重**吗？去重 key 是什么？和本次 retry 的 payload 稳定一致吗？
- 如果消费者**不去重**，要么修消费者，要么在上游标记 `repeated=true`

**反模式**：把"消费者应当幂等"作为**未经验证的假设**——这是一种责任甩锅。上游发出重复消息的责任，不能用"下游应该去重"免除。

---

## 🚨 陷阱 9：Success path 的 Async 写入竞态

**症状**：主流程提交 status=FINISH，立即发起 async goroutine/worker 写入某种"成功缓存"。快速 follow-up 的幂等调用命中 FINISH 状态 → 去读 cache → **cache 还没写入** → 降级返回或报错。

**通用模式**：

```
  T0: db.commit(status=FINISH)
  T1: go persist_cache_async()   ← 毫秒级延迟才完成
  T2: 并发调用 retry() 命中 status=FINISH，立即去 read_cache() ← 读不到
```

**修复方向**：任何"成功后写入 cache"的逻辑，都要想清楚：
- **Sync 写入**（阻塞主流程）vs **async**（有竞态）
- 如果是 async，follow-up 调用是否有兜底（rebuild / fallback）？
- 如果 sync，写入失败是否应该阻塞主流程？

**通常的权衡**：优先 sync，让 FINISH 成为"cache 就绪"的保证。Async 只在写入本身昂贵且 fallback 健壮时使用。

---

## 🚨 陷阱 10：Multi-pod Cron 双重执行

**症状**：定时任务 scheduler 部署在多个 pod，每个 pod 都跑同样的 cron，捞同一批待处理 record，导致重复处理。

**识别方法**：对每个 scheduler 任务：
- 它部署几份？（K8s replicas）
- 它怎么选择处理哪些 record？（`WHERE status='pending' LIMIT N`？）
- **多个 pod 同时选中同一批 record 时会怎样？**

**修复 pattern**：**Atomic claim**
```sql
UPDATE pending_task
SET status='processing', claimed_at=?
WHERE id=? AND status='pending'
```
检查 `RowsAffected > 0` 才是 claim 成功。第二个 pod 拿到 `RowsAffected=0`，**跳过**。

**不要**用"先 SELECT 再 UPDATE"——这是经典的 lost-update race。

**进阶**：如果业务允许，直接用分布式锁把整个 cron 串行化（只允许一个 pod 同时跑）。

---

## 🚨 陷阱 11：向前兼容而不向后兼容

**症状**：共享 proto / model 定义做了"无害的小改"（加字段、改注释、调整 enum 顺序），跨服务部署顺序错了就炸。

**识别方法**：任何共享 proto / model / RPC 签名的改动，问：
- 老版本客户端调用新版本服务端，会怎样？（通常 OK）
- 新版本客户端调用老版本服务端，会怎样？（经常炸！）
- enum 删除/重排 → 老服务端收到新 enum 值会如何？
- 字段改类型（int32 → int64）→ 会不会被截断？
- 字段改"可选 → 必填"→ 老客户端不填会被拒绝吗？

**修复 pattern**：
- Proto：只加字段、不改字段号、不删字段、不改类型、不改 required/optional
- 部署顺序：**服务端先升级兼容新旧的版本，再升级客户端**
- 老字段要淘汰时，先发"deprecated" → 跨若干版本 → 才删

---

## 🚨 陷阱 12：迁移/双写的"读旧 vs 读新"漂移

**症状**：数据迁移期间采用"双写 + 渐进切读"策略。某天发现新旧表数据有细微不一致，但已经来不及回滚——谁是 source of truth 都说不清。

**识别方法**：在迁移 spec 里必须回答：
1. 双写期间写失败的处理策略？（新表失败是否阻塞主流程？旧表失败呢？）
2. 写入顺序？（先新后旧 vs 先旧后新，失败补偿怎么做？）
3. 数据对账：**有没有独立的 checker 定期比对新旧数据并告警？**
4. 切读时机：依据什么指标（对账差异率 < X%）判定"可以切"？
5. 切完之后的 rollback 窗口：多长时间内如果发现问题可以回切？

**不要**：只凭"应该是一致的"这种信念切读。一定要有数据证据。

---

## 快速自检表（Phase 6 简化前必过）

打算简化？先勾选以下项：

- [ ] 我已经理解这段代码**被加入时解决的具体问题**
- [ ] 我验证了那个问题**现在确实不会再发生**（有代码证据）
- [ ] 简化后的代码在 **retry 路径**下仍然正确
- [ ] 简化后的代码在 **并发路径**下仍然正确
- [ ] 简化后的代码在 **失败路径**下仍然正确
- [ ] 没有任何"多义字段"假设
- [ ] 没有任何"应当幂等"的未经验证假设
- [ ] 简化的代码量 > 新引入的潜在风险（ROI 判断）
- [ ] 我愿意承担"如果错了，就 git revert"的纪律

**以上任何一项不能勾选，不要简化**。
