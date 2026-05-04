# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of [Claude Code Superpowers](https://superpowers.anthropic.com/) skills — battle-tested workflows, development methodologies, and prompt templates packaged as reusable units that Claude Code loads on demand.

## Repo Structure

```
skills/<skill-name>/
  SKILL.md              # Entry point — required frontmatter (name, description) + skill content
  references/           # Supporting reference docs loaded by the skill
dist/
  <skill-name>.skill    # Zip bundle of skills/<skill-name>/ for distribution
.claude/
  commands/             # Custom slash commands that invoke skills
```

## Skills

| Skill | Purpose |
|-------|---------|
| `cross-verified-feature-development` | High-risk feature dev with 4-round cross-verification before merge |

## Creating a New Skill

1. Create `skills/<skill-name>/SKILL.md` with required frontmatter:
   ```markdown
   ---
   name: <skill-name>
   description: <trigger description — this is what Claude uses to decide when to invoke the skill>
   ---
   ```
2. Put supporting files in `skills/<skill-name>/references/` or alongside `SKILL.md`.
3. Build the distribution bundle:
   ```bash
   cd skills && zip -r ../dist/<skill-name>.skill <skill-name>/
   ```
4. Add an entry to the Skills table in `README.md`.

## Distribution Format

`.skill` files in `dist/` are zip archives of the skill directory. They are used for distribution to users who don't have the repo cloned. The symlink install (described in README) is the preferred local setup — it keeps skills current with `git pull`.

## Installation (for reference)

Skills are symlinked from this repo into `~/.claude/skills/`:
```bash
ln -sf "$PWD/skills/cross-verified-feature-development" ~/.claude/skills/cross-verified-feature-development
```
