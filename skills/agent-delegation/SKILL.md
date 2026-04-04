---
name: agent-delegation
description: >
  Use when you need another agent to handle a task — content drafting,
  research, publishing, or any work outside your current scope. Covers
  issue creation, labels, assignment, and status tracking via Paperclip
  issues.
---

# Agent Delegation

How to recruit other agents by creating well-structured Paperclip issues.
Use this skill when a task should be handed off — either because another
agent is better suited or because your current scope doesn't include the
required action.

## When to use

- A content opportunity needs drafting (hand off to a content agent)
- A research request is too large for your current cycle
- You need something published but don't have publishing capabilities
- A user asks you to have another agent do something

## When NOT to use

- You can handle the task yourself with your assigned skills
- The task is a simple question answerable in conversation

---

## Issue creation

All delegation uses Paperclip issues in the **AIS project**.

### Required fields

**Title**: `[Task Type] Brief description`

Task types:
- `Content` — draft a post, thread, or reply
- `Research` — investigate a topic in depth
- `Engagement` — reply to or quote-tweet a conversation
- `Review` — review a draft or analysis

**Status**:
- `backlog` — normal priority, pick up when available
- `todo` — high priority, pick up next

**Body**: Include enough context for the receiving agent to act without
asking follow-up questions:

```markdown
## Task

{1-2 sentences: what needs to be done}

## Context

{Why this task exists — what triggered it, what the user asked, what
opportunity was identified}

## Evidence / inputs

{Links to tweets, corpus references, prior analysis — everything the
receiving agent needs}

- [View post](https://x.com/author/status/ID) — @author: "relevant quote"

## Constraints

{Voice (hub/builder), deadline, specific instructions, compliance notes}

## Success criteria

{How the receiving agent knows the task is done correctly}
```

### Labels

Use labels to route issues to the right agent:

| Label | Routes to |
|-------|-----------|
| `voice:hub` | Content agent working hub account |
| `voice:builder` | Content agent working builder accounts |
| `content-opportunity` | Content agents (general) |
| `research-request` | Research-capable agents |
| `engagement` | Agents with `x-engagement` skill |

### Assignment

**Do NOT assign to a specific agent** unless:
- Your system prompt tells you which agent handles which tasks
- The user explicitly names an agent

Use labels for routing. Agents pick up issues matching their assignments.

---

## Tracking delegated work

After creating an issue:

1. **Note the issue ID** — report it to the user or include in your summary
2. **Check status later** — if asked, look up the issue to see if it's been
   picked up and what progress has been made
3. **Don't duplicate** — if an issue already exists for the same task, update
   it rather than creating a new one

---

## When to delegate vs. handle directly

| Situation | Action |
|-----------|--------|
| User asks you to draft content and you have `x-content-doctrine` | Handle it yourself |
| User asks you to draft content and you don't have `x-content-doctrine` | Delegate |
| Opportunity found during analysis | Delegate to content agent |
| User asks "have Clawson write about X" | Delegate (user explicitly named an agent) |
| Simple research question | Handle it yourself with `x-research` |
| Complex multi-day research project | Delegate |

**Principle**: Delegate when the task requires skills you don't have, when
another agent is explicitly requested, or when the task is a distinct work
item that should be tracked independently.
