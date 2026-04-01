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
| [x-discovery-engine](skills/x-discovery-engine/SKILL.md) | Curate daily X intelligence using plugin tools | Clawdius (X Discovery) |
| [x-content-doctrine](skills/x-content-doctrine/SKILL.md) | Brand voice, content buckets, topic boundaries, compliance | Content agents (Brand Voice Writer, Builder Voice Writer, etc.) |
| [x-analysis-framework](skills/x-analysis-framework/SKILL.md) | Evaluate corpus for content opportunities, create issues | Topic Scout, sensing agents |

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
