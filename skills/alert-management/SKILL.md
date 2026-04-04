---
name: alert-management
description: >
  Use when woken by a plugin alert (via ctx.agents.invoke) or when a user
  asks to configure alert preferences. Covers severity assessment, preference
  matching, routing decisions, quiet hours, and deduplication.
---

# Alert Management

How to handle incoming alerts from the x-intelligence plugin. Alerts arrive
via `ctx.agents.invoke()` — ephemeral invocations where all context
(including user preferences for personal agents) is in the prompt string.

## When to use

- You've been invoked with alert context in your prompt (e.g., high
  engagement tweet, corpus update, handle promotion)
- A user asks to configure their alert preferences

## When NOT to use

- For general user conversation (use `user-interaction`)
- For researching a topic (use `x-research`)

---

## How alerts arrive

The plugin detects events (high-engagement tweets, corpus updates) and
invokes agents with structured context in the prompt:

**For operational agents** (no memory file): Raw alert context — event type,
tweet data, engagement metrics, relevant links.

**For personal agents** (memory file configured): Enriched prompt — the raw
alert context plus the user's preferences (topic interests, alert
thresholds) extracted from their memory file by the plugin at invoke time.

You don't need to read any files — everything you need is in the prompt.

---

## Processing an alert

### Step 1: Parse the alert

Extract from the prompt:
- **Event type**: `tweet.high-engagement`, `corpus.updated`,
  `handle.promoted`, etc.
- **Content**: The tweet/thread/event data with links
- **User preferences** (if present): Topic interests, thresholds, channel

### Step 2: Assess relevance

For personal agents with user preferences in the prompt:

1. Does this alert match any of the user's stated topic interests?
2. Does it meet their alert threshold (e.g., "major traction" = 100+
   engagements)?
3. Is this information they'd want to know about?

If the answer to all three is no → **dismiss the alert**. "Nothing
actionable" is a valid outcome. Not every high-engagement tweet needs to
reach every user.

For operational agents (no user preferences):

1. Does this event require operational action?
2. Is there an opportunity to detect (apply `opportunity-detection`)?
3. Should other agents be notified?

### Step 3: Assess severity

| Severity | Criteria | Action |
|----------|----------|--------|
| **Urgent** | Direct PEAK6 mention, breaking news in core domain, engagement spike 500+ | Route immediately via fastest channel |
| **High** | Authority thread in PEAK6 domain, engagement 100+, time-sensitive opportunity | Route promptly, create issue if actionable |
| **Medium** | Relevant to user interests, moderate engagement | Include in next batched summary |
| **Low** | Tangentially related, low engagement | Log and dismiss |

### Step 4: Route

Based on severity and user preferences:

| Channel | When |
|---------|------|
| **Slack DM** | Urgent and high severity (default for personal agents) |
| **Batched summary** | Medium severity — collect and send at next summary interval |
| **Issue creation** | When the alert reveals an actionable opportunity (use `agent-delegation`) |
| **Dismiss** | Low severity or doesn't match user interests |

---

## Quiet hours

Respect the user's quiet hours from their preferences (delivered in the
prompt for personal agents).

During quiet hours:
- **Urgent**: Still route immediately (Slack DM)
- **High**: Hold for first post-quiet-hours summary
- **Medium/Low**: Hold for batched summary

If no quiet hours are specified, default to routing based on severity.

---

## Deduplication

Before routing, check if you've already processed a substantially similar
alert in this invocation:

- Same tweet ID → skip
- Same thread, different tweet → consolidate into one alert about the thread
- Same topic cluster, different tweets → consolidate with links to all

---

## Alert preference configuration

When a user asks to set up or change alerts (via conversation, not via
invoke):

1. Understand what they want: topics, thresholds, channels, quiet hours
2. Update their memory file (`data/memory/{name}.md`) — specifically the
   `## Alert preferences` section
3. Confirm what you saved

Example:
- User: "Alert me when AI x Fintech posts are getting major traction"
- Action: Update memory file with topic interest "AI x Fintech" and
  threshold "major traction (100+ engagements)"
- Response: "Got it. I'll alert you when AI x Fintech posts cross 100
  engagements. You can adjust the threshold anytime."

Note: The plugin reads this memory file at alert time to construct enriched
prompts. Changes take effect on the next alert cycle.

---

## Output format for routed alerts

When routing to the user (Slack DM or summary):

```markdown
**[Severity] Alert: {brief description}**

{1-2 sentences: what happened and why it matters to this user}

**Key post**: @author — "quote or paraphrase"
[View post](https://x.com/author/status/ID)

**Engagement**: {likes, reposts, replies}
**Why this matters to you**: {connection to user's stated interests}
**Suggested action**: {read the thread / consider drafting a response / no action needed}
```

Keep alerts concise. The user should be able to assess in 10 seconds
whether to dig deeper.
