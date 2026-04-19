# golang-skills

A collection of Claude Code skills for Go development.

## How to install a skill

Copy the skill directory into your Claude skills folder:

```bash
# macOS / Linux
cp -r <skill-name> ~/.claude/skills/

# Example
cp -r golang-clean-architecture ~/.claude/skills/
```

After copying, the skill is immediately available in any Claude Code session — no restart needed.

## Available skills

### `golang-clean-architecture`

Go 整洁架构（Clean Architecture）实践指南。

Covers:
- Four-ring model: domain / application / adapter / infra
- Dependency direction rules and quick-reference table
- Interface placement strategy (consumer-defined interfaces)
- Bootstrap / composition root wiring
- Over-abstraction detection and rollback

**Trigger keywords:** clean architecture, 整洁架构, hexagonal, ports and adapters, 依赖反转, domain, application, infra, bootstrap, 组合根

**Usage:** mention any trigger keyword or type `/golang-clean-architecture` in the prompt.

---

## Contributing

1. Create a directory named after your skill
2. Add a `SKILL.md` with the required frontmatter:

```markdown
---
name: your-skill-name
description: one-line description used for trigger matching
---

# Skill content here
```

3. Submit a PR.
