# Claude Skills

Reusable Claude Code skills for software development workflows.

## Available Skills

| Skill | Alias | Description | Version |
|-------|-------|-------------|---------|
| [figma-to-code](./f2c/) | `/f2c` | Build production Next.js websites from Figma designs | 1.7.0 |
| [sphereos-cms-api](./sos-api/) | `/sos-api` | SphereOS CMS GraphQL API reference | 1.0.0 |
| [production-audit](./production-audit/) | `/p-audit` | Production readiness audit for Next.js/NestJS apps | 1.3.0 |

## Version Control

Skills include automatic version checking. When you invoke a skill, it will:

1. Check if a newer version is available on GitHub
2. Prompt you with options:
   - **Update and continue** - Downloads latest version
   - **Continue with current** - Uses local version
   - **View changelog** - Shows what's changed

## Installation

Skills are installed to `~/.claude/skills/`. To install or update manually:

```bash
# Clone the repo
git clone https://github.com/gautam-lulla/claude-skills.git ~/claude-skills

# Copy to Claude skills directory
cp -r ~/claude-skills/f2c ~/.claude/skills/
cp -r ~/claude-skills/sos-api ~/.claude/skills/
cp -r ~/claude-skills/production-audit ~/.claude/skills/

# Create short aliases (symlinks)
cd ~/.claude/skills
ln -sf figma-to-code f2c 2>/dev/null || ln -sf f2c f2c
ln -sf sphereos-cms-api sos-api 2>/dev/null || ln -sf sos-api sos-api
ln -sf production-audit p-audit
```

## Structure

Each skill directory contains:
- `SKILL.md` - Main skill definition and instructions
- `CHANGELOG.md` - Version history and changes
- `LEARNINGS.md` - Accumulated learnings and improvements (optional)
- Additional reference files as needed

## Contributing

1. Make changes to the skill files
2. Update the version in `SKILL.md` frontmatter
3. Add entry to `CHANGELOG.md`
4. Push to GitHub

Users will be prompted to update on next invocation.
