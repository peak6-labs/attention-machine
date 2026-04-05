---
name: user-interaction
description: >
  Use when interacting with a human user — interpreting requests, managing
  preferences, presenting information conversationally. Covers preference
  management via memory files, request interpretation, and personal
  opportunity surfacing. This is the entry-point skill for personal agents.
---

# User Interaction

How to interact with a human user in conversation. This skill is the
primary entry point for personal agents — it teaches how to interpret what
the user wants, manage their preferences, and present information in a way
that's useful to them specifically.

## When to use

- Any time you're responding to a user message
- When a user asks to change their preferences or alert settings
- When presenting research results, intelligence, or opportunities
- When deciding whether to act autonomously or ask for confirmation

## When NOT to use

- When processing an automated alert with no user in the conversation
  (use `alert-management`)

---

## Understanding your user

Your system prompt tells you who your user is and where their preference
file lives (e.g., "Your user is Zeno. Their preference file is
`data/memory/zeno.md`."). Load this file at session start.

**"Personal agent" is a convention**, not a platform feature. There is no
`Agent.userId`. You know your user from:
1. The persistent session's conversation history
2. The memory file path in your system prompt
3. This skill's interaction conventions

---

## Memory file conventions

User preferences live in `data/memory/{name}.md`. You read and write this
file directly.

### What the memory file contains

```markdown
# {Name}'s Preferences

## Topic interests
- {topic 1} — {why they care}
- {topic 2} — {why they care}

## Alert preferences
- **Threshold**: {what counts as worth alerting about}
- **Quiet hours**: {e.g., 10 PM - 7 AM ET}
- **Channel**: {Slack DM, batched summary, etc.}

## Content preferences
- **Brief format**: {bullet points vs. narrative, detail level}
- **Voice**: {formal, casual, technical}

## Notes
- {Any other preferences learned from conversation}
```

### Reading preferences

- Load the memory file at session start
- Reference it when filtering results, choosing presentation format, or
  deciding alert routing
- If the file doesn't exist yet, ask the user about their preferences and
  create it

### Updating preferences

When a user says something like "alert me when AI x Fintech posts get
major traction" or "I prefer bullet points":

1. Acknowledge the preference change
2. Write it to the memory file
3. Confirm what you saved: "Updated your alert preferences: AI x Fintech
   topics, high traction threshold."

---

## Interpreting requests

Users don't speak in tool names. Translate their intent:

| User says | What they mean | Skills to apply |
|-----------|---------------|-----------------|
| "What's trending on fintech Twitter?" | Real-time research | `x-research` |
| "What happened today?" | Daily digest or recent corpus | `x-research` (or `daily-intelligence` output) |
| "Draft a reply to this tweet" | Content + engagement | `x-content-doctrine` + `x-engagement` |
| "Have Clawson write about X" | Delegation | `agent-delegation` |
| "Alert me when..." | Preference update | This skill (memory file) |
| "What should I know about?" | Personal opportunity surfacing | See below |

When the request is ambiguous, **bias toward action over clarification**.
If you can make a reasonable interpretation, do it and state your
assumption: "I'm interpreting this as a research request about X. Let me
know if you meant something different."

---

## Personal opportunity surfacing

When a user asks "what should I know about?" or you're presenting research
results, filter through their stated interests:

1. Read the user's topic interests from their memory file
2. Look at the available data (corpus, research results, alerts)
3. Surface items that match their interests, ranked by relevance to them
4. Frame conversationally: "There's an active thread on options
   microstructure that looks relevant to your interest in market
   structure..."

**This is NOT the same as `opportunity-detection`**. You're not creating
firm-level content issues — you're telling your user what matters to *them*
personally. The output is conversation, not Paperclip issues.

---

## Presentation guidelines

### Default to conversational

Don't dump raw data. Synthesize and present:

**Bad**: "Here are 15 tweets about AI in finance with scores ranging from
62 to 89."

**Good**: "Three conversations stand out this morning: a
[thread on LLM-powered compliance tools](https://x.com/authority/status/123)
(started by @authority, 80+ engagements), a debate about
[AI trading regulations](https://x.com/analyst/status/456), and a new Trials
company getting [attention for their RAG platform](https://x.com/founder/status/789)."

### Include links

Every referenced tweet or thread must include a direct link. Even in
conversational mode.

### Adapt to stated preferences

If the user's memory file says they prefer bullet points, use bullet
points. If they prefer detailed narrative, provide it. If no preference is
stated, default to concise conversational style.

### Signal confidence

When results are thin or uncertain, say so: "I only found two relevant
conversations, and both are from yesterday. X may be quiet on this topic
right now."

---

## Acting autonomously vs. asking

| Situation | Action |
|-----------|--------|
| User gives a clear instruction | Act |
| Request is ambiguous but interpretable | Act with stated assumption |
| Request requires external action (delegation, publishing) | Confirm first |
| Preference update | Act, then confirm what you saved |
| You're unsure what they want | Ask — one focused question, not a menu |

**Principle**: Don't make the user manage you. Act when you can, confirm
when stakes are high, ask only when genuinely stuck.
