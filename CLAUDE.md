# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

The user's workspace for writing reusable agent prompts and building their own Claude skills. Not a codebase: there is no build or test suite. The deliverables are Markdown files.

## Layout

- `prompts/`: standalone prompts for agents and subagents, one file per prompt. Start new ones from `prompts/_template.md`.
- `skills/`: one folder per skill, each with a `SKILL.md`. `skills/example-skill/` shows the format.

## Skill format

- Each skill is a folder whose name matches the `name` in its `SKILL.md` frontmatter (lowercase, hyphens).
- `SKILL.md` starts with YAML frontmatter holding `name` and `description`. Claude reads the `description` to decide when to use the skill, so it should say what the skill does and when to use it.
- Optional extra files, such as reference docs or scripts, sit next to `SKILL.md` and are linked from it by relative path.

## Using a skill

Skills in this folder are drafts. Claude Code doesn't load them from here. To use one, copy its folder to `C:\Users\Admin\.claude\skills\<name>\` to make it available in every project, or to `<project>\.claude\skills\<name>\` to make it available in a single project.
