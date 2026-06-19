# Skills

A small collection of reusable skills I find useful.

## Install in Codex

In Codex, ask the built-in skill installer to install this repo:

```text
Use $skill-installer to install the skill at https://github.com/JeroenGar/skills/tree/main/handoff-review
```

Restart Codex after installation so the new skill is picked up.

## Add to Claude.ai

Claude.ai can use the same skill folder:

1. Download or clone this repo.
2. Zip the `handoff-review/` folder.
3. Upload the zip as a custom Skill in Claude.ai's Skills settings.

Claude uses the `SKILL.md` file. The `agents/openai.yaml` file is Codex-specific metadata and is not needed by Claude.

## Use with Claude Code

For Claude Code, clone this repo and point your project instructions at the skill file:

```markdown
@/absolute/path/to/this-repo/handoff-review/SKILL.md
```

Inspect any third-party skill before installing or referencing it.
