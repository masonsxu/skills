# Masons Xu's Skills

Personal collection of Claude Agent Skills.

## Skills

| Skill | Description |
|-------|-------------|
| [pretext-integration](./skills/pretext-integration) | Integrate @chenglou/pretext for fast, DOM-free multiline text measurement and layout |

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # Skill definition with YAML frontmatter + instructions
    evals/
      evals.json      # Eval test cases for the skill
```

## Usage

Install a skill by copying its folder to your agent's skills directory (e.g., `~/.agents/skills/`).

For more information about the Agent Skills standard, see [agentskills.io](https://agentskills.io).
