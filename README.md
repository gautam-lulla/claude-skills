# Claude Skills

Reusable Claude Code skills for software development workflows.

## Available Skills

| Skill | Description |
|-------|-------------|
| [production-audit](./production-audit/) | Comprehensive production readiness audit for Next.js/NestJS applications |

## Usage

These skills can be used with Claude Code by referencing them in your project's `.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "Skill(production-audit)"
    ]
  }
}
```

Or invoke directly: `/production-audit`

## Structure

Each skill directory contains:
- `SKILL.md` - The main skill definition and instructions
- `LEARNINGS.md` - Accumulated learnings and improvements

## Contributing

Feel free to add new skills or improve existing ones. Each skill should be self-contained and well-documented.
