# MageByte Power Skills

> Battle-tested [Claude Code Superpowers](https://superpowers.anthropic.com/) skills — engineering workflows and verification methodologies distilled from real production incidents.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![中文文档](https://img.shields.io/badge/文档-中文版-blue)](README.zh-CN.md)

---

## Skills

| Skill | What it does | When to use |
|-------|-------------|-------------|
| [`cross-verified-feature-development`](#cross-verified-feature-development) | 7-phase feature workflow with 4 independent cross-verification passes | High-stakes backend features: payments, state machines, concurrency, cross-service changes |

---

## What is Superpowers?

[Superpowers](https://superpowers.anthropic.com/) is an MCP plugin for Claude Code that provides general-purpose skills: `brainstorming`, `writing-plans`, `subagent-driven-development`, `systematic-debugging`, `code-reviewer`, and more.

The skills in this repo are **domain-specific orchestrators** — they call Superpowers' general skills at the right phase with the right context, so you get the full power of the ecosystem without wiring it up yourself.

```
You (user prompt)
    │
    ▼
cross-verified-feature-development  ← this repo
    │
    ├── Phase 1   →  superpowers:brainstorming
    ├── Phase 2   →  superpowers:writing-plans
    ├── Phase 3   →  superpowers:subagent-driven-development
    ├── Phase 4.1 →  superpowers:systematic-debugging
    ├── Phase 4.2 →  superpowers:code-reviewer  (cold-context, no design docs)
    ├── Phase 4.3–4.5 → dispatches agents with domain-specific prompt templates
    └── Phase 5   →  superpowers:writing-plans + subagent-driven-development
```

**Without Superpowers**, the skill still works — each phase has a documented fallback using Claude Code's built-in tools.

---

## Prerequisites

**1. Claude Code**
```bash
npm install -g @anthropic-ai/claude-code
```

**2. Superpowers plugin** (recommended)
```bash
claude mcp add --transport http superpowers https://superpowers.anthropic.com/mcp
```
First-time use requires an Anthropic account. Fallback mode is available without Superpowers.

---

## Installation

```bash
git clone https://github.com/MageByte-Zero/magebyte-power.git
export SKILLS_REPO="$PWD/magebyte-power"

mkdir -p ~/.claude/skills

ln -sf "$SKILLS_REPO/skills/cross-verified-feature-development" \
       ~/.claude/skills/cross-verified-feature-development
```

Verify:
```bash
ls ~/.claude/skills/cross-verified-feature-development/SKILL.md
```

---

## cross-verified-feature-development

A rigorous 7-phase feature development methodology that inserts **4 independent verification passes** between implementation and merge — catching the concurrency, idempotency, and cross-service contract bugs that self-review and standard code review systematically miss.

### The core problem this solves

Single-perspective review has structural blind spots. Even experienced developers will:
- Default to their own design assumptions when reading their own code
- Assume "should be idempotent" operations actually are
- Miss concurrent race windows because their mental model is sequential
- Forget that other services consume their shared tables and MQ topics

**Independent perspectives produce independent signals.** Running N reviewers who don't share context produces a bug set that's close to a union, not a duplicate. This workflow operationalizes that insight:

> Design → Implement → **Multi-round cross-verification** → Fix → Carefully simplify → Sync docs

### How Superpowers powers each phase

| Phase | Superpowers skill | What it adds |
|-------|------------------|-------------|
| 1. Requirements & Design | `superpowers:brainstorming` | Structures vague requirements into a complete spec with invariants, failure modes, and risk table |
| 2. Implementation Plan | `superpowers:writing-plans` | Breaks spec into independently verifiable tasks with file+line targets, TDD structure, rollout strategy |
| 3. Implementation | `superpowers:subagent-driven-development` | Each task runs in a fresh subagent — no accumulated bias, independent spec compliance + quality review per task |
| 4.1 Self-review | `superpowers:systematic-debugging` | Structured "assume there's a bug, find it" scan against the full feature |
| 4.2 Cold-context review ⭐ | `superpowers:code-reviewer` | Reviewer gets **no design docs** — finds design-level blind spots the author can't see |
| 4.3–4.5 (parallel) | Custom agent prompts | Behavior-preservation diff, cross-repo impact scan, business invariant matrix |
| 5. Fix iteration | `superpowers:writing-plans` + `subagent-driven-development` | Same rigor for fixes as for original implementation |

**Phase 4.2 (Cold-Context Review) is the highest-value step.** Typical yield: 5–15 Critical/High bugs per review that would have reached production with standard code review.

### When to use

```
Does the feature involve any of the following?
│
├── 💰 Financial transactions, payments, refunds, settlements?      → YES → Use this workflow
├── 🔄 Order / inventory state machines with status transitions?    → YES → Use this workflow
├── 🔒 Distributed locks, concurrency control, idempotent retry?    → YES → Use this workflow
├── 🔗 Cross-service MQ/RPC contracts or shared proto/model change? → YES → Use this workflow
├── 🗄️  Online schema migration or dual-write strategy?             → YES → Use this workflow
└── ⏱️  Estimated effort ≥ 3 person-days with high cost-of-failure? → YES → Use this workflow

None of the above?  →  Standard workflow is fine ✓
```

**One-line heuristic:** If the worst possible bug involves *money loss / data corruption / stuck orders / privilege escalation*, use this workflow.

### The 7 phases

```
① Requirements & Design    → superpowers:brainstorming
①.5 Architecture Review    → ADR (required for high-risk features)
② Implementation Plan      → superpowers:writing-plans
③ Implementation           → superpowers:subagent-driven-development
④ 🔥 Cross-Verification    ← 4 independent passes (the core innovation)
⑤ Fix Iteration            → writing-plans + subagent-driven-development
⑥ Careful Simplification   → skeptical optimization with anti-pattern checklist
⑦ Doc Sync                 → backfill evolution log + notify downstream
```

### The 4 verification passes

| Pass | Perspective | Typical findings |
|------|-------------|-----------------|
| 4.1 Systematic Debugging | Self as bug hunter | 1–3 pre-existing bugs |
| 4.2 Cold-Context Review ⭐ | Reviewer with **no design docs** | 5–15 concurrency / idempotency bugs |
| 4.3 Behavior-Preservation Diff | Master vs feature side-effects | 2–5 semantic regressions |
| 4.4 Cross-Repo Impact Scan | Other repos that may need changes | 0–3 external impact points |
| 4.5 Business Invariant Matrix | Hard constraints (money/state/inventory) | 0–2 invariant violations |

Passes 4.3–4.5 can be dispatched in parallel via subagents.

### How to trigger

```
/cross-verified-workflow <feature description>
```

Or describe a high-stakes feature — the skill proactively suggests itself when it detects relevant patterns.

### Bundled reference files

| File | When to read |
|------|-------------|
| `references/cross-verification-techniques.md` | Before Phase 4 — agent prompt templates for all 4 passes |
| `references/anti-patterns.md` | Before Phase 6 — 12 high-frequency trap patterns |
| `references/doc-sync-playbook.md` | Before Phase 7 — structured doc backfill |
| `references/case-studies.md` | Optional — real-world bug museum from e-commerce order domain |

### Cost vs. benefit

| Metric | Standard workflow | This workflow |
|--------|------------------|---------------|
| Extra time | — | +40–50% |
| Critical bug detection rate | ~40% | ~95% |
| Phase 4.2 typical yield | 0 | 5–15 High/Critical bugs |

---

## Contributing

Skills in this repo are distilled from real engineering experience. The best contributions are workflows proven in production.

1. Create `skills/<skill-name>/SKILL.md` with required frontmatter:
   ```markdown
   ---
   name: <skill-name>
   description: <when to trigger — be specific about contexts and keywords>
   ---
   ```
2. Add supporting files in `skills/<skill-name>/references/`
3. Test it on real tasks before submitting
4. Open a PR describing the domain and what problem it solves

---

Maintainer: [MageByte-Zero](https://github.com/MageByte-Zero) · [MIT License](LICENSE)
