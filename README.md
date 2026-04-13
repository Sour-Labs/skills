# skills

A collection of [Claude Code](https://claude.com/claude-code) skills.

Each subdirectory is a self-contained skill with its own `SKILL.md` defining the
instructions, references, and assets Claude loads when the skill is invoked.

## Using a skill

Copy or symlink a skill directory into your Claude Code skills location (for
example, `~/.claude/skills/`), then invoke it by name with the `Skill` tool or
via the `/<skill-name>` slash command.

## Adding a skill

Create a new directory at the repo root containing at minimum a `SKILL.md` file
with the skill's frontmatter and instructions. Supporting references, examples,
and assets can live alongside it.
