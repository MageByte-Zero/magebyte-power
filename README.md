# MageByte Power Skills

A collection of [Claude Code Superpowers](https://superpowers.anthropic.com/) skills — battle-tested workflows, debugging runbooks, and prompt templates packaged for reuse.

工程实践沉淀的 Claude Code Superpowers skills 合集。把踩过的坑、总结出的流程、积累的 prompt 模板封装成可复用单元，让每个人都能直接站在这些经验上工作。

---

## Skills

| Skill | What it does · 用途 | Use when · 适用场景 |
|-------|---------------------|---------------------|
| [`cross-verified-feature-development`](#cross-verified-feature-development) | 7-phase rigorous feature workflow · 多轮交叉验证开发工作流 | High-risk features · 资金流、状态机、并发控制、跨服务改造 |

---

## Prerequisites · 前置条件

**1. Claude Code**
```bash
npm install -g @anthropic-ai/claude-code
```

**2. Superpowers plugin** (skills runtime · skills 运行环境)
```bash
claude mcp add --transport http superpowers https://superpowers.anthropic.com/mcp
```
First-time use requires an Anthropic account. · 首次使用需要 Anthropic 账号授权。

---

## Installation · 安装

Symlink from this repo — clone once, update with `git pull`. · 推荐符号链接方式：克隆一次，`git pull` 即可更新。

```bash
export SKILLS_REPO="$HOME/path/to/magebyte-power"
mkdir -p ~/.claude/skills

# cross-verified-feature-development
ln -sf "$SKILLS_REPO/skills/cross-verified-feature-development" ~/.claude/skills/cross-verified-feature-development
```

Verify · 验证：
```bash
ls ~/.claude/skills/cross-verified-feature-development/SKILL.md
```

---

## cross-verified-feature-development

A rigorous 7-phase feature development methodology that inserts 4 independent verification passes between implementation and merge — catching bugs that self-review and standard code review systematically miss.

多轮交叉验证的严谨特性开发方法论，在实施与合并之间插入 4 轮独立视角的验证，发现单一 review 会系统性漏掉的 bug。

### How it works · 核心逻辑

| Round | Perspective · 视角 | Typical findings · 典型发现 |
|-------|--------------------|-----------------------------|
| 4.1 Systematic Debugging | Self as bug hunter · 自己扮演 bug 猎人 | 1–3 pre-existing bugs |
| 4.2 Cold-Context Review | Reviewer with **no design docs** · 不读设计文档的独立 reviewer | 5–15 concurrency / idempotency bugs |
| 4.3 Behavior-Preservation Diff | Master vs feature side-effects · 逐条对比副作用变化 | 2–5 semantic regressions |
| 4.4 Cross-Repo Impact Scan | Other repos that may need changes · 扫描其他仓库联动影响 | 0–3 external impact points |

### When to use · 适用场景

Use when any of these apply · 满足任一条件：
- Financial transactions, payments, or refunds · 资金流 / 退款 / 结算
- Order / inventory state machines · 订单状态机 / 改单 / 履约
- Distributed locks, concurrency control, idempotent retry · 并发控制 / 分布式锁 / 幂等重试
- Cross-service MQ/RPC contracts or shared proto changes · 跨服务接口 / MQ 协议 / 共享 proto
- Online schema migration or dual-write · 数据迁移 / schema 变更
- Estimated effort ≥ 3 person-days with high cost-of-failure · 预估工作量 ≥ 3 人日且失败代价高

### Trigger · 触发方式

```
/cross-verified-workflow <feature description>
```

Or say: "cross-verified workflow", "交叉验证", "走严谨工作流", "按交叉验证方式开发"

### Bundled files · 内含参考资料

| File | Contents |
|------|----------|
| `references/cross-verification-techniques.md` | Agent prompt templates for all 4 verification passes · 4 种验证的 agent prompt 模板 |
| `references/anti-patterns.md` | 12 high-frequency trap patterns · 12 个高频踩坑模式 |
| `references/doc-sync-playbook.md` | Doc backfill playbook · 文档回填操作手册 |
| `references/case-studies.md` | Real-world case studies from e-commerce order domain · 电商订单领域真实踩坑案例 |

---

## Contributing a new skill · 贡献新 Skill

1. Create `skills/<skill-name>/SKILL.md` with required frontmatter:
   ```markdown
   ---
   name: <skill-name>
   description: <trigger description — this is what Claude uses to decide when to invoke>
   ---
   ```
2. Put supporting files in `skills/<skill-name>/references/` or alongside `SKILL.md`
3. Build the distribution bundle:
   ```bash
   cd skills && zip -r ../dist/<skill-name>.skill <skill-name>/
   ```
4. Update the Skills table in this README

---

Maintainer: [MageByte-Zero](https://github.com/MageByte-Zero)
