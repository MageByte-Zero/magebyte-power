# Case Studies · 真实踩坑合集

本文件保留了本工作流诞生过程中的真实踩坑案例，作为**教学素材**。`anti-patterns.md` 是抽象后的通用教训，本文件是**原始现场**——具体的字段名、RPC 名、SQL，帮助理解"抽象陷阱"在真实业务里长什么样。

> These cases come from an e-commerce order/amendment/refund domain. If you work in a different domain, read them as a **structured bug museum** — focus on the patterns, not the specific names.
>
> ⚠️ 这些案例来自电商订单/改单/退款域。如果你不在这个领域工作，把它们当成**结构化的 bug 博物馆**来读，关注**模式**而不是具体名称。

---

## Case 1: Polymorphic Field Trap · 多义字段翻车

**（Anti-pattern 1 原型）**

**背景**：交易改单幂等重试特性，`transaction_application` 表用于记录改单申请的异步处理进度。该表有字段 `reference_id` 用于关联业务实体。

**"看起来"的事实**：
- 主路径（`reference_type=refund`）下，`reference_id` 存退款单号
- 主流程先插入 `transaction_application` 行，再异步处理退款

**开发者的（错误）假设**：`reference_id` 一定存退款单号，因为主路径就是这样。所以简化时觉得"`persistSuccessResult` 那个缓存 JSON 是冗余的——反正退款单号在 `reference_id` 里能直接拿到"。

**实际情况**：部分支付（补差价）场景下，`reference_type=payment`，`reference_id` 存**支付单号**。这个写入由**另一条完全不同的业务流**（`payment_handler.go`）在 `ProcessTransaction` 调用之前就完成了。

**翻车后果**：简化后的代码在部分支付场景下把 `PaymentID` 作为 `RefundID` 返回给调用方。下游用这个 "RefundID" 去查退款单详情 → 查不到 → 报"退款单不存在" → 人工介入排查。好在测试环境提前发现。

**教训**：
1. 任何"简化某个看似冗余的存储"前，必须 grep 所有写这个字段的点（而不仅仅主流程的写入点）
2. 带 `type` 字段的 polymorphic table 是头号高危区
3. commit 回滚是发现后唯一正确的反应——不要尝试"再加个分支判断打补丁"，那会让代码变得更难维护

**修复**：`git revert` 简化 commit，在 evolution log 里记录这次失败尝试。

---

## Case 2: Lock Scope Gap · 锁粒度不足导致并发全量副作用重复

**（Anti-pattern 5 原型）**

**背景**：`ProcessTransaction` 是改单主处理器。设计初期只在"retry 入口（status=PROCESSING）"加分布式锁，认为"首次处理（status=PENDING）不会并发"。

**并发窗口**：
```
  Client A 请求 ProcessTransaction(transaction_no=X)  \
                                                       > 同时到达（比如用户点了两次）
  Client B 请求 ProcessTransaction(transaction_no=X)  /

  两者都读到 status=PENDING
  两者都执行无条件 UPDATE status='processing' WHERE id=?
  两者都进入后续流程
  →→→ 都执行：发退款 + 发 MQ + 更新 fulfillment + 发通知
```

**翻车后果**：同一改单被执行两次完整副作用，退款被下发两次（虽然下游退款服务有幂等保护，但每次都多发了 MQ + 更新 fulfillment + 日志告警）。

**根本原因**：
1. **锁只守 retry 入口**，首次入口裸奔
2. PENDING→PROCESSING 的转换用的是**无条件 UPDATE**，没有 CAS 语义

**修复**：
1. 锁提到**入口层**，覆盖所有会并发的 status
2. PENDING→PROCESSING 改为 CAS：
   ```sql
   UPDATE transaction_application
   SET status='processing', processing_time=?
   WHERE id=? AND status='pending'
   ```
   检查 `RowsAffected=1` 才是"真·第一次成功"
3. `defer lock.Release(ctx)` 调整到"确认是第一次抢到锁"之后注册，避免释放别人的锁
4. 锁 TTL 从 60s 调整到 300s（内部处理可能慢到 2 分钟）

**教训**：**永远不要假设"某个状态不会并发"**。用户能点两次、前端能重试、网关能 double-submit。防御性设计的成本远低于事故成本。

---

## Case 3: Terminal State Before Cleanup · FINISH 早于清理导致永久卡死

**（Anti-pattern 6 原型）**

**背景**：改单完成后需要做三件事：`FINISH(transaction)` + `UpdateRelatedRecords(关联)` + `ReleaseResourceLock(资源锁)`。原始顺序是 `FINISH → UpdateRelatedRecords → ReleaseResourceLock`。

**故障场景**：`ReleaseResourceLock` RPC 超时失败。此时：
- `transaction.status` 已经是 FINISH
- 下次同一改单 retry 时，代码命中 `getCachedTransactionResponse`——看到 FINISH → 直接返回缓存 → 流程结束
- `ReleaseResourceLock` **永远**不会被重试
- 订单资源锁永久卡死，用户在前端看"处理中"转圈圈

**线上表现**：每周 1-2 个卡单 case，需要手动清理。

**修复**：重排为 `ReleaseResourceLock → UpdateRelatedRecords → persistCache → FINISH`

这样 FINISH 是"一切都完成"的 checkpoint。`ReleaseResourceLock` 失败 → status 保持 PROCESSING → 下次 retry 走 Path B → 再次尝试 `ReleaseResourceLock` → 成功后才 FINISH。

**教训提炼**（见 anti-patterns Trap 6）：**终态标记必须是最后一步**。这是一条通用原则，不只限于订单状态机。

---

## Case 4: Assumed RPC Idempotency · 假幂等 RPC

**（Anti-pattern 7 原型）**

**背景**：案例 3 的修复依赖"可以重复调用 `ReleaseResourceLock`"。但这个 RPC **不是**幂等的。

**底层实现**：
```sql
UPDATE order_record
SET lock_status=0, pending_transaction_no=''
WHERE order_id=? AND lock_status=1 AND pending_transaction_no=?
```

**为什么不幂等**：第二次调用时，`lock_status` 已经是 0，WHERE 子句不匹配 → `RowsAffected=0` → RPC 判定为"状态异常，拒绝解锁"并返回 error。

**发现过程**：Phase 4.4 跨服务扫描阶段，reviewer agent 被要求"对每个 RPC 读底层实现"时发现。

**修复选项**：
- **选项 A**（调用方做预检查）：先 `SELECT lock_status`，只在 `=1` 时调用
- **选项 B**（改 RPC 让它天然幂等）：
  ```sql
  UPDATE order_record
  SET lock_status=0, pending_transaction_no=''
  WHERE order_id=? AND pending_transaction_no=?
  -- 无论 lock_status 当前是 1 还是 0，都 UPDATE；
  -- 第二次 RowsAffected 可能是 0 但不再返回错误
  ```

最终选 B，因为选项 A 引入了 TOCTOU 窗口。

**教训**：**不要信接口签名，读底层 SQL / 实现**。尤其是"看起来是纯 setter"的 RPC，底层可能有隐藏的 `WHERE <前置状态>` 约束。

---

## Case 5: Async Cache Race · Success Cache 异步竞态

**（Anti-pattern 9 原型）**

**背景**：`persistSuccessResult` 负责把改单成功的完整 response 写入缓存，供后续幂等调用快速返回。最初实现：

```go
db.commit(status=FINISH)
go func() { persistSuccessResult(ctx, resp) }()
return resp, nil
```

**竞态窗口**：
```
  T0: db.commit(FINISH)
  T1: goroutine 启动但还没执行到 cache SET
  T2: 客户端立刻发来第二次请求（几毫秒延迟）
      → 代码走 getCachedTransactionResponse → cache miss
      → fallback 逻辑触发，可能重建出不一致的 response
  T3: 原 goroutine 终于写入 cache（此时已经晚了）
```

**发现过程**：Phase 4.2 cold-context reviewer 发现。Reviewer 原话（翻译）："这个 async 写入和 FINISH commit 之间有个毫秒级窗口，任何在这个窗口内的 retry 都读不到 cache，但看到的 status 已经是 FINISH。这会走 fallback，fallback 的正确性你验证过吗？"

**修复**：改为同步写入 + 失败不阻塞主流程（log + metric）：
```go
if err := persistSuccessResult(ctx, resp); err != nil {
    logger.Warnf(ctx, "persist success result failed, FINISH already committed: %v", err)
    // 不 return err——已经 FINISH 了，不能让主流程失败
}
```

配套增强 fallback 的 rebuild 逻辑（尽管正常流程下不会走到）作为 defense-in-depth。

**教训**：任何"主流程 commit 后 async 写入 cache"的模式，都要问：**在 async 完成之前 retry 进来会看到什么？** 通常的默认选择应该是 sync。

---

## Case 6: Documentation Drift · 文档漂移翻车

**（Anti-pattern 4 原型）**

**背景**：该改单特性最早的设计文档写于 Q2，7 个 task。实施完成后又经历 3 轮 review fix（加锁、改顺序、幂等化 RPC 等），代码已经大幅偏离原设计。文档未同步。

**三个月后**：另一个同学要加"改单完成后给用户发放奖励"的新特性。他读了原设计文档，按文档里的"FINISH → UpdateRelatedRecords → ReleaseResourceLock"顺序，把"发奖励"加到了 `UpdateRelatedRecords` 中。

**出事**：发奖励的 MQ publish 放在 `UpdateRelatedRecords` 里，但实际代码的 `UpdateRelatedRecords` 已在 FINISH 之后执行。他测试时走的是主路径，没有触发 retry。上线后一周，一个 retry 场景下奖励被重复触发（因为 cache 生效 + `UpdateRelatedRecords` 没有去重）。

**根因**：原设计文档没有记录"顺序已调整"这个决策。新同学踩了**同一个坑的变体**。

**教训**：**Phase 7 的文档回填不是 "nice to have"**。它是团队知识传递的基础设施。任何省掉 Phase 7 的项目，都是在给未来埋雷。

---

## Pattern Quick Reference · 模式速查

| 你观察到 | 立刻联想 | 对应 anti-pattern |
|---------|---------|-----------------|
| 表里有 `xxx_type` + `xxx_id` 成对出现 | Case 1 | Trap 1 |
| 锁只在 retry 路径加 | Case 2 | Trap 5 |
| 终态标记在清理步骤之前 | Case 3 | Trap 6 |
| 调用"状态转换"类 RPC 做 retry | Case 4 | Trap 7 |
| Commit 后 async 写 cache | Case 5 | Trap 9 |
| 设计文档最后一次改是 >1 个月前 | Case 6 | Trap 4 |

---

## Why These Cases Matter · 为什么保留这些案例

通用的 anti-patterns 告诉你"别做 X"，但人类记忆对**具体故事**的粘性远高于对抽象规则的粘性。下次你写类似代码时，脑子里能闪过"这和案例 1 的多义字段翻车好像"——这个反应就能救你一命。

这也是为什么每个好的 engineering team 都应该有自己的"事故博物馆"——真实案例是最好的老师。

> Generic anti-patterns tell you *what not to do*, but humans remember **stories** far better than abstract rules. When you next encounter a polymorphic `(type, id)` table or an async cache write after a commit, a mental flash of "this looks like Case 1" is worth more than any checklist.
