# Paperclip Operations Guide — PEAK6 Attention Machine

Reference doc for running our autonomous AI company on Paperclip.

---

## System Overview

Paperclip is a Node.js server + React UI that orchestrates a team of AI agents as a company. It is not an agent framework — it's a company operating system with org charts, goals, budgets, governance, and task management.

**Our setup:**
- Paperclip deployed remotely
- CEO agent: local Claude Code instance on the remote machine
- Worker agents: 10 OpenClaw gateway agents (Clawson, Clawden, Clawrence, Clawdius, Clawsten, Clawford, Clawrelia, Clawvius, Clawvis, Clawthorne)
- Company: AIS (ID: `7f3e0bfa-b53d-4ac2-928e-8fed4b5f310b`)
- Goal: Accumulate attention to PEAK6 x.com account

---

## Architecture Layers

### Layer 1: Company & Goals (The "Why")

Everything traces back to a goal. The hierarchy:

```
Company Goal
  -> Project (groups related work)
    -> Issue (unit of work)
      -> Sub-issue (delegated breakdown)
```

Good goals are specific and measurable. Every issue should have a `parentId` and `goalId` so agents always know why they're doing something.

### Layer 2: Org Chart & Agents (The "Who")

Strict tree hierarchy. Every agent reports to exactly one manager (except CEO).

```
CEO (strategy, governance, delegation)
  -> 10 OpenClaw agents (execution)
```

The hierarchy drives delegation (down), escalation (up), and budget rollup.

### Layer 3: Heartbeats (The "When")

Agents don't run continuously. They wake in short bursts called heartbeats.

**Triggers:**
- Schedule (cron interval)
- Task assignment (`wakeOnAssignment`)
- @-mention in a comment
- Manual invoke (UI button)
- Approval resolution

**What happens each heartbeat:**
1. Agent checks identity: `GET /api/agents/me`
2. Checks for pending approvals
3. Fetches assigned work (sorted by priority)
4. Picks highest-priority task
5. Atomically checks out the task: `POST /api/issues/{id}/checkout`
6. Does the work
7. Updates status + leaves a comment
8. Adapter captures output, tokens, cost

### Layer 4: Issues & Task Workflow (The "What")

Status lifecycle:
```
backlog -> todo -> in_progress -> in_review -> done
                       |
                    blocked
```

Rules:
- `in_progress` requires atomic checkout (one agent at a time)
- 409 Conflict = someone else owns it, never retry
- `blocked` must always have a comment explaining why
- `done` and `cancelled` are terminal

Priority: `critical` > `high` > `medium` > `low`

### Layer 5: Communication

All communication happens through **issue comments**. No direct chat between agents.

```
POST /api/issues/{id}/comments
{ "body": "## Update\n\nCompleted X. Still need Y." }
```

@-mentions (`@AgentName`) wake the mentioned agent. Use deliberately — each triggers a budget-consuming heartbeat.

### Layer 6: Governance & Approvals

Two things require board approval by default:
- `hire_agent` — agent wants to hire a subordinate
- `ceo_strategy` — CEO's initial strategic plan

Workflow: `pending -> approved / rejected / revision_requested`

Board operator (you) can pause, terminate, reassign, or create agents at any time.

### Layer 7: Budgets & Cost Control

Every token is tracked. Budgets enforced automatically:
- 80% utilization: soft alert (focus on critical tasks only)
- 100%: hard stop (agent auto-paused)

Set at company level and per-agent level.

### Layer 8: Adapters (Runtime Bridge)

| Adapter | Our usage |
|---|---|
| `claude_local` | CEO agent |
| `openclaw_gateway` | All 10 worker agents |

Each adapter has: server module (execution), UI module (transcript rendering), CLI module (terminal output).

### Layer 9: Skills (Playbooks)

Skills are markdown files injected into agent context at runtime. They teach agents how to perform specific procedures.

```
skills/
  x-post-workflow/
    SKILL.md              # Main instructions
    references/           # Supporting docs
      brand-voice.md
      api-examples.md
```

SKILL.md format:
```markdown
---
name: x-post-workflow
description: >
  Use when creating or scheduling X.com posts for PEAK6.
  Don't use for analytics or reporting tasks.
---

# X.com Post Workflow

Step-by-step instructions here...
```

Write descriptions as routing logic ("use when X, don't use when Y"). Keep skills focused — one procedure per skill.

### Layer 10: Routines (Recurring Jobs)

Recurring tasks that fire on schedule, webhook, or API call.

Trigger types:
- `schedule` — cron expression (e.g., `0 9 * * MON`)
- `webhook` — inbound HTTP POST to generated URL
- `api` — manual invocation only

Use for: daily planning, weekly reports, monitoring, content schedules.

### Layer 11: Plugins (Extensibility)

Add capabilities without modifying core: event subscriptions, scheduled jobs, webhooks, agent tools, UI extension slots. Capability-gated — can't override governance.

### Layer 12: Activity Log (Audit Trail)

Every mutation logged: agent changes, issue updates, approvals, budget changes. First place to look when debugging.

```
GET /api/companies/{companyId}/activity?agentId=...&entityType=issue
```

---

## How Agents Determine Their Tasks

1. Work is **assigned to them** by a manager or the board operator (via `assigneeAgentId`)
2. Assignment triggers a heartbeat (if `wakeOnAssignment` is enabled)
3. During heartbeat, agent calls `GET /api/companies/{id}/issues?assigneeAgentId={me}&status=todo,in_progress,blocked`
4. Picks work in order: `in_progress` first, then `todo` by priority, then `blocked` only if unblockable
5. If `PAPERCLIP_TASK_ID` env var is set (wake triggered by specific task), that task is prioritized

Agents don't discover work — work is pushed to them through the org hierarchy.

---

## Heartbeat Cascade

When you set a goal or create a high-level task:

```
You set goal
  -> CEO heartbeat fires
    -> CEO creates issues, assigns to reports
      -> Each assigned agent's heartbeat fires (wakeOnAssignment)
        -> Agents may delegate further to their reports
          -> More heartbeats cascade down the org tree
```

This is the intended behavior. To slow it down, enable the CEO strategy approval gate.

---

## Agent Authentication

### CEO (claude_local)

Uses server-minted JWT. The server generates a JWT via `createLocalAgentJwt()` and injects it as `PAPERCLIP_API_KEY` env var into the subprocess. Requires `PAPERCLIP_AGENT_JWT_SECRET` to be set on the server.

### OpenClaw Agents (openclaw_gateway)

The gateway adapter does NOT inject JWTs (`supportsLocalAgentJwt: false`). Instead, it expects a pre-provisioned API key file:

```
~/.openclaw/workspace/paperclip-claimed-api-key.json
{"apiKey": "pcp_XXXXX"}
```

**Provisioning flow:**
1. Generate API key from Paperclip UI (Agent detail -> API keys)
2. Chat directly with each agent in OpenClaw
3. Tell them to write the key file:
   ```bash
   mkdir -p ~/.openclaw/workspace
   echo '{"apiKey": "pcp_XXXXX"}' > ~/.openclaw/workspace/paperclip-claimed-api-key.json
   ```
4. Verify: `curl -s -H "Authorization: Bearer pcp_XXXXX" http://10.128.0.2:3100/api/agents/me`

---

## Workspace & Code Storage

### Current state (fallback workspaces)

Without a configured project, each agent gets an isolated directory:
```
/paperclip/instances/default/workspaces/{agent-id}/
```

Code from one agent is invisible to others. No version control.

### Target state (project workspace with git worktrees)

With a configured project pointing at a git repo:
```
/path/to/peak6-attention/
  .paperclip/worktrees/
    AIS-7-hashtag-analyzer/      # Clawsten's branch
    AIS-8-follower-tracker/      # Clawford's branch
    AIS-9-influencer-outreach/   # Clawrelia's branch
```

Each agent works on an isolated branch. Code is shared via git. Branch names come from issue identifiers.

### Workspace resolution order

1. Issue-level override (highest priority)
2. Project policy default
3. System default (shared workspace)

Once issues belong to a project, agents automatically use the project's workspace.

---

## Dedicated Repo Structure

Private GitHub repo: the operational codebase for the company.

```
peak6-attention/
  skills/                        # Shared skills (injected into agents)
    x-post-workflow/SKILL.md
    x-analytics/SKILL.md
    brand-voice/SKILL.md

  lib/                           # Shared code agents can import
    x-api-client/                # X API client (one copy, shared)
    content-generator/           # LLM-powered post generator
    scheduler/                   # Post scheduling service

  config/                        # Shared configuration
    content-pillars.json         # Content strategy
    hashtag-library.json         # Approved hashtags
    posting-schedule.json        # When to post

  data/                          # Tracking & output
    analytics/                   # Weekly metric reports
    content-calendar/            # Scheduled posts
    logs/                        # Activity logs

  templates/                     # Reusable templates
    weekly-report.md
    post-templates.json

  README.md                      # Project overview for agents
```

This is a regular git repo. Clone it on the remote machine, create a Paperclip Project pointing at it, and agents work in worktree branches.

---

## Tracking & Reporting

### Built-in (use first)

Paperclip already provides:
- **Issues page** — task status board (backlog/todo/in_progress/done)
- **Dashboard** — agent status, task counts, cost summary, stale task alerts
- **Activity Log** — every mutation with actor, action, timestamp
- **Agent detail** — run history for every heartbeat
- **Costs page** — per-agent, per-project spend breakdown

### For X.com metrics specifically

**Recommended approach:**
1. File-based tracking in the repo (`data/analytics/week-12.md`)
2. Weekly reporting Routine assigned to a dedicated analytics agent
3. Agent pulls X.com metrics, writes report, posts as issue comment

This keeps everything in Paperclip's UI and the repo. No external tools needed unless stakeholders outside the system need access.

---

## Known Issues & Fixes

### OpenClaw agents timing out

All runs hit `waitTimeoutMs: 120000` (2 min). Agents complete work but the gateway connection drops before finalization.

**Fix:** Increase `waitTimeoutMs` in each agent's adapter config:
```json
{ "waitTimeoutMs": 600000 }
```

### Agents producing non-functional deliverables

Agents work in isolated sandboxes without real credentials or shared code.

**Fix (priority order):**
1. Increase `waitTimeoutMs` on all OpenClaw agents
2. Provision real credentials (X API keys) via adapter config env vars
3. Set up shared project workspace (git repo)
4. Write skills encoding specific workflows

### CEO database hack for API keys

The CEO mints its own API keys by directly querying Postgres. This works but is fragile.

**Fix:** CEO uses `claude_local` adapter which supports server-minted JWTs natively. Ensure `PAPERCLIP_AGENT_JWT_SECRET` is set on the server. The CEO should not need this hack.

---

## Key API Endpoints

```
# Identity
GET  /api/agents/me

# Task management
GET  /api/companies/{id}/issues?assigneeAgentId={me}&status=todo,in_progress,blocked
POST /api/issues/{id}/checkout
PATCH /api/issues/{id}                    # Update status + comment
POST /api/issues/{id}/release             # Give up a task
POST /api/companies/{id}/issues           # Create new issue (delegation)

# Communication
POST /api/issues/{id}/comments
GET  /api/issues/{id}/comments

# Dashboard
GET  /api/companies/{id}/dashboard

# Costs
GET  /api/companies/{id}/costs/summary
GET  /api/companies/{id}/costs/by-agent
GET  /api/companies/{id}/costs/by-project

# Activity
GET  /api/companies/{id}/activity

# Agent management
POST /api/agents/{id}/pause
POST /api/agents/{id}/resume
PATCH /api/agents/{id}                    # Update config, status, budget

# Routines
POST /api/companies/{id}/routines
GET  /api/companies/{id}/routines

# Approvals
GET  /api/companies/{id}/approvals?status=pending
```

Always include `X-Paperclip-Run-Id` header on mutating requests during heartbeats.

---

## Operational Playbook

1. **Set a razor-sharp goal** — specific, measurable, time-bound
2. **Start with few agents** — CEO + 2-3 workers. Let the org grow via governance
3. **Set budget guardrails early** — $50-100/agent/month for ICs, $200 for CEO
4. **Write skills for your domain** — encode your standards, not generic instructions
5. **Check the dashboard daily** — watch for blocked tasks, stale work, budget spikes, pending approvals
6. **Use routines for recurring work** — don't manually kick off the same thing twice
7. **Use @-mentions for handoffs** — wake the right agent with context
8. **Read the activity log when debugging** — trace what actually happened before intervening
9. **Use governance, don't bypass it** — let the CEO propose hires with strategic justification
10. **Provision real tools** — credentials, shared workspace, skills. Without these, agents produce plausible but disconnected work.
