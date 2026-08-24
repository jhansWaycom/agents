# agents

Cursor Agent skills. Clone this repo and install a skill into `~/.cursor/skills` so it works in any Way-com product repo without committing skills into those repos.

## Pull

```bash
git clone git@github.com:jhansWaycom/agents.git
```

HTTPS: https://github.com/jhansWaycom/agents

## Skills

| Skill | Folder | Use when |
|-------|--------|----------|
| **Jira-bugfixingAgent** | `.cursor/skills/Jira-bugfixingAgent/` | Jira key → root-cause fix, tests (success / failure / edge / race), paired ES + DB queries for dual validation, PR, paste-ready description, self-assessment scorecard of its own run |

## Install Jira-bugfixingAgent

```bash
mkdir -p ~/.cursor/skills
cp -R agents/.cursor/skills/Jira-bugfixingAgent ~/.cursor/skills/Jira-bugfixingAgent
```

Start a **new Agent chat** in the product repo and say `Fix SR-3099`.
