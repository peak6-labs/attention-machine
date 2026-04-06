# Attention Machine

Operational hub for the PEAK6 Attention Machine — the system of AI agents, skills, and plugins that manages marketing, audience growth, and X/Twitter intelligence across PEAK6 products.

This repo is the **Paperclip Project workspace**. Agents work in git worktree branches from this repo and have filesystem access to everything here.

## Repository Structure

```
attention-machine/
  docs/             # Operational reference docs (Paperclip guide, workflows, deployment)
  skills/           # Paperclip Skills — markdown playbooks injected into agent context
  config/           # Shared configuration (content pillars, brand voice, schedules)
  data/             # Agent-generated data (corpus, analytics, memory — largely gitignored)
  hooks/            # Agent behavior enforcement scripts
  customers/        # Product marketing plans (Peak6 Trials, etc.)
```

## Skills

Skills are markdown files that teach agents how to perform specific procedures. They are injected into agent context at runtime based on the `description` field in their frontmatter.

| Skill | Purpose | Used by |
|-------|---------|---------|
| [x-research](skills/x-research/SKILL.md) | Research methodology for any X topic — query decomposition, signal assessment, synthesis | All agents |
| [x-content-doctrine](skills/x-content-doctrine/SKILL.md) | Brand voice, content buckets, topic boundaries, compliance | Content + personal agents |
| [x-engagement](skills/x-engagement/SKILL.md) | Engagement tactics — replies, quotes, threading, timing | Content + personal agents |
| [opportunity-detection](skills/opportunity-detection/SKILL.md) | Identify content opportunities, cluster detection, scoring | Operational agents |
| [daily-intelligence](skills/daily-intelligence/SKILL.md) | Orchestrate daily corpus digest after pipeline run | Clawden (Topic Scout) |
| [agent-delegation](skills/agent-delegation/SKILL.md) | Create issues to delegate work to other agents | Operational + personal agents |
| [user-interaction](skills/user-interaction/SKILL.md) | Human interaction, preference management, conversational presentation | Personal agents |
| [alert-management](skills/alert-management/SKILL.md) | Handle plugin alerts, severity assessment, routing | Personal agents |

### Adding a new skill

Create a new directory under `skills/` with a `SKILL.md` file:

```
skills/
└── your-skill-name/
    └── SKILL.md          # YAML frontmatter (name, description) + instructions
```

The `description` field should include routing logic: "Use when X. Don't use when Y." Push to main and agents pick it up on next heartbeat.

## Docs

| Document | Purpose |
|----------|---------|
| [PAPERCLIP_GUIDE.md](docs/PAPERCLIP_GUIDE.md) | Platform architecture, agent lifecycle, API reference |
| [WORKFLOW_BREAKDOWN.md](docs/WORKFLOW_BREAKDOWN.md) | Full workflow specs (W1-W11), agent assignments, hooks |
| [CLOUD_RUN_MIGRATION.md](docs/CLOUD_RUN_MIGRATION.md) | GCP deployment guide |
| [Workflows.md](docs/Workflows.md) | Brief workflow inventory |

## Customers

Each product gets its own folder under `customers/` with marketing plans, ICPs, and tracking.

| Customer | Goal | Folder |
|----------|------|--------|
| Peak6 Trials | 1,500 applications by May 29 | [customers/peak6trials/](customers/peak6trials/) |

## Related Repos

| Repo | Purpose |
|------|---------|
| [x-intelligence-plugin](https://github.com/peak6-labs/x-intelligence-plugin) | X/Twitter sensing plugin (npm: `peak6-x-intelligence-plugin`) |
| [paperclip-deploy](https://github.com/peak6-labs/paperclip-deploy) | Docker build + Cloud Run CI/CD |
| [paperclip](https://github.com/peak6-labs/paperclip) | Thin fork of upstream Paperclip server |
