# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin (`wb`) for the WorkBuddy/JMS platform. It provides skills, commands, and agents — not a traditional application with build/test/lint steps. The plugin is registered via `.claude-plugin/plugin.json`. This repo also serves as a **private plugin marketplace** via `.claude-plugin/marketplace.json`, allowing other projects to install the plugin with `/plugin install wb@wb-tools`.

## Architecture

- **Skills** (`skills/`): Knowledge modules with `SKILL.md` entry points and optional `references/` subdirectories. The `conventions` skill auto-loads (not user-invocable); others (`angular`, `cqrs-implementation`, `entity-framework`) are invoked on demand.
- **Commands** (`commands/`): User-invocable slash commands prefixed `/wb:` (e.g., `/wb:cqrs-scaffold`, `/wb:review-code`). Each is a single markdown file.
- **Agents** (`agents/`): Specialized agent definitions (currently `cqrs-specialist` using claude-sonnet-4-6).

## Key Conventions

- All content is markdown — no build steps, no tests, no dependencies.
- Commands, skills, and agents are registered by directory convention in `plugin.json` (pointing to `skills/`, `commands/`, `agents/`).
- Skills reference files go in `references/` subdirectories within each skill folder.
- The plugin targets three sibling repositories: jms-web (Angular 20+), jms-model, and jms-api (Entity Framework / CQRS).
