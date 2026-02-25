# WorkBuddy Claude Code Plugin

Development toolkit for the WorkBuddy/JMS platform - skills, commands, and agents for Angular, CQRS, Entity Framework, and UI/UX development.

## Installation

### From Marketplace (recommended)

Add the marketplace, then install the plugin:

```bash
/plugin marketplace add workbuddy/ai-toolkit
/plugin install wb@wb-tools
```

Auto-update is disabled by default for third-party marketplaces. To enable it, devs must opt in manually in their project settings.

### Consumer Setup

Add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": [
    "workbuddy/ai-toolkit"
  ],
  "enabledPlugins": [
    "wb@wb-tools"
  ]
}
```

See [`setup/consumer-settings.example.json`](setup/consumer-settings.example.json) for a ready-to-copy example.

### Direct (local path)

```bash
claude /plugin /path/to/ai-toolkit
```

## Skills

| Skill | Description |
|-------|-------------|
| `conventions` | Background coding conventions (auto-loaded, not user-invocable) |
| `angular` | Angular 20+ development patterns for jms-web |
| `cqrs-implementation` | WorkBuddy's custom CQRS architecture patterns |
| `entity-framework` | Entity Framework implementation across jms-model/api/web |

## Commands

| Command | Description |
|---------|-------------|
| `/wb:cqrs-scaffold` | Scaffold CQRS components following WorkBuddy patterns |
| `/wb:review-code` | Smart code reviewer with technology detection |
| `/wb:complete-fix` | Complete bug fix workflow (commit, comment, update Linear) |
| `/wb:commit-message` | Generate commit message from staged changes |
| `/wb:implement-feature` | Implement feature from Linear issue |
| `/wb:fix-bug` | Fix bug from Linear issue with guided workflow |
| `/wb:link-commits` | Link git commits to Linear issues automatically |

## Agents

| Agent | Description |
|-------|-------------|
| `cqrs-specialist` | Specialized CQRS architecture agent (claude-sonnet-4-6) |

## Structure

```
ai-toolkit/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── conventions/SKILL.md
│   ├── angular/
│   │   ├── SKILL.md
│   │   └── references/examples.md
│   ├── cqrs-implementation/
│   │   ├── SKILL.md
│   │   └── references/patterns.md
│   ├── entity-framework/
│   │   ├── SKILL.md
│   │   └── references/{guide,patterns,debug}.md
├── commands/*.md
├── agents/cqrs-specialist.md
├── setup/consumer-settings.example.json
└── README.md
```
