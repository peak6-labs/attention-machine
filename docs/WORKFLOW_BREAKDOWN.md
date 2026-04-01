# Workflow Breakdown — PEAK6 Attention Machine

## System Architecture

### Two-Tier Agent Model

The system has two tiers of agents with fundamentally different execution models:

**Operational Agents** — run the business. Scheduled Routines, issue-triggered workflows, heartbeat-based execution. These agents produce content, monitor channels, and manage pipelines.

**Personal Agents** — one per person. Always-on, real-time conversational. Each person's agent is their interface to the entire system. Personal agents can access everything operational agents produce, maintain personal memory, and execute on behalf of their human.

```
OPERATIONAL AGENTS (Paperclip heartbeat model)
  Clawson ......... Content Lead (copy, X posts, distribution)
  Clawrence ....... LinkedIn & Compliance (content pipeline, digest)
  Clawdius ........ X Discovery Engine (daily firm-wide intelligence)
  Clawthorne ...... PR Email Monitor (external comms follow-ups)

PERSONAL AGENTS (persistent OpenClaw sessions — real-time)
  Zeno's Agent .... Personal assistant for Zeno
  Andrew's Agent .. Personal assistant for Andrew
  Riyanka's Agent . Personal assistant for Riyanka
  Chloe's Agent ... Personal assistant for Chloe
  Sydney's Agent .. Personal assistant for Sydney

UNASSIGNED (available for new roles or personal agents)
  Clawden
```

### Why Two Tiers?

Operational work (content pipelines, scheduled digests, email monitoring) fits Paperclip's heartbeat model — wake up, do a task, sleep. This is efficient and cost-controlled.

Conversational interaction (texting Clawdia, asking for a tweet draft, giving opinions) needs real-time response. Forcing this through heartbeat wake cycles adds 30-60s latency that makes the experience feel broken.

Personal agents run as **persistent OpenClaw sessions** that stay alive. They bypass Paperclip's heartbeat cycle for the conversational loop but still:
- Register with Paperclip (for tracking, budgets, activity log)
- Read from Paperclip API (issues, comments, what other agents produced)
- Create issues via Paperclip API (delegate work to operational agents)
- Log conversations back to Paperclip (audit trail)

---

## Workflows

### W1: Stale Task Monitoring & Nudges — DEFERRED

> "The System adds to a tracker... emails Zeno, Andrew, Riyanka and Chloe asking for status updates"

**Status:** Deferred. Not a day-one workflow.

**Rationale:** Team now has Paperclip accounts and can see the Issues page and Dashboard directly. The dashboard already shows stale tasks. Building a push notification system on top of a notification system adds complexity without proven need.

**Revisit when:** Team consistently misses stale tasks despite having Paperclip access. At that point, add Slack nudges via a Routine.

---

### W2: PR/External Follow-ups

> "The System is cc'ed on emails with external PR parties and reminds Riyanka of daily follow ups"

**What it does:** Monitors email threads with external PR contacts (newsletters, events, podcasts). Sends Riyanka daily reminders of what needs follow-up.

**Agent:** Clawthorne (dedicated operational agent for PR email monitoring)

**Tools needed:**
- AgentMail (`clawthorne@agentmail.to`) — cc'd on all PR threads, inbound webhook triggers agent, threads API to follow conversations
- AgentMail send — daily digest email to Riyanka
- AgentMail labels — track thread state (`needs-followup`, `responded`, `escalated`)

**Skill:** `pr-followup` — how to parse PR emails, what counts as needing follow-up, reminder format

**Routine:** Daily morning cron (digest). AgentMail webhook (real-time inbound monitoring).

**Setup required:** CC `clawthorne@agentmail.to` on all external PR email threads. Configure AgentMail webhook (message.received) -> Paperclip Routine webhook URL -> triggers Clawthorne.

**Human-in-the-loop:** Riyanka receives digest, follows up manually. No auto-replies to external parties.

**Hooks:**
- `PreToolUse: empty-digest-blocker` — blocks sending if digest body has no content items (P2)
- `PostToolUse: post-send-checklist` — enforces thread labeling, Paperclip logging, issue update after every send (P1)
- `PostToolUse: thread-label-enforcer` — reminds to classify threads (`needs-followup`/`responded`/`escalated`) before generating digest (P3)

**Execution pattern:** Sequential pipeline — fetch threads → classify → generate digest → send + label (see [Execution Patterns](#execution-patterns))

---

### W3: X Discovery Engine — REDESIGNED

> "Texts leaders at PEAK6 trending tweets in the morning and asks them to send in an anonymous voice note on their takes"

**Revised approach:** The original "morning briefing" is actually the first layer of a four-layer X intelligence system. Split the send (operational, scheduled) from the receive (personal agent, real-time).

#### Layer 1: Raw Data Collection (daily Routine)

**Agent:** Clawdius (X Discovery Engine)

**What it does:** Every morning, collects the day's relevant X content using two complementary tools:
- **X API v2** — keyword/hashtag search for finance, trading, PEAK6-relevant topics. Structured data with engagement metrics (likes, RTs, replies). Monitor competitor accounts, track hashtags.
- **xAI x_search (Grok)** — semantic discovery. Finds relevant conversations even without exact keyword matches. "Institutional investors questioning market structure" is a meaning search, not a keyword search.

**Tools needed:**
- X API v2 (search recent, user timelines, trending)
- xAI API with x_search capability
- Paperclip API (store results as issue comments or repo files)

**Skill:** `x-discovery-engine` — search queries to run, accounts to monitor, how to combine X API structured results with xAI semantic results, output format, storage location

**Routine:** Daily 6am cron

**Output:** Raw tweet corpus stored in repo (`data/x-intelligence/YYYY-MM-DD.json`) with tweet text, author, metrics, relevance score, and source (X API vs xAI).

#### Layer 2: Firm-Wide Curation (daily, after Layer 1)

**Agent:** Clawdius (same agent, second pass)

**What it does:** LLM filters and ranks the raw corpus from Layer 1:
- Groups by theme (market structure, fintech, regulation, trading tech, etc.)
- Identifies engagement opportunities (high-value threads worth joining)
- Highlights what PEAK6 specifically should care about
- Flags tweets that align with PEAK6's content pillars

**Output:** Firm-wide daily digest stored as a Paperclip issue with status `in_review`. Available to all personal agents.

**Routine:** Daily 7am cron (runs after Layer 1 completes)

**Hooks (Layers 1-2):**
- `PreToolUse: clawdius-read-only` — blocks any X API write/post calls; Clawdius is discovery-only (P0)
- `PreToolUse: duplicate-run-blocker` — blocks if `data/x-intelligence/YYYY-MM-DD.json` already exists today (P3)
- `PostToolUse: corpus-storage-verifier` — after search API calls, verifies corpus was written as valid JSON with expected fields (P3)

**Execution pattern:** Sequential pipeline — X API search → xAI semantic search → merge corpus → store JSON → curate → create Paperclip issue (see [Execution Patterns](#execution-patterns))

#### Layer 3: Personal Curation (via personal agents)

**Agent:** Each person's personal agent

**What it does:** Filters the firm-wide digest through individual preferences (from personal memory). "Zeno, here are 3 threads you'd want to weigh in on today."

**How:** Personal agent reads the firm digest issue, cross-references with `data/memory/zeno.md`, surfaces the most relevant items. Delivered via SMS, chat, or however the person prefers.

This replaces the old "text leaders trending tweets" with something smarter — the tweets are pre-filtered to what each person actually cares about.

#### Layer 4: Engagement Execution (on-demand, via personal agents)

**Agent:** Each person's personal agent

**What it does:** User says "I want to respond to this" (voice note, text, or picks from curated list). Agent drafts reply/quote/retweet using brand voice + personal voice. User approves. Clawson posts via X API.

**Handoff:** Personal agent creates a Paperclip issue with the draft, assigns to Clawson for posting (or posts directly if the person has X API access via their personal agent).

---

### W4: Personal Agents — REDESIGNED (merges old W4 + W10)

> "They can text back and ask Clawdia for the types of tweets they are interested in"
> "User can go back and forth with the System to give their opinions over text and its stored in personal memory"

**Revised approach:** Every team member gets their own persistent OpenClaw agent. This is their personal "Clawdia" — not a shared bot, but a dedicated assistant that knows them.

#### What each personal agent can do

| Capability | How |
|---|---|
| Real-time conversation | Persistent OpenClaw session. Multiple interfaces: SMS (Twilio), Slack DM, web chat. No heartbeat latency. |
| Access all system output | Paperclip API — query issues, comments, activity across the company. Read what operational agents produced. |
| Personal memory | Per-user memory file (`data/memory/{name}.md`). Updated on every conversation. Stores preferences, opinions, interests, communication style. |
| Personalized X feed | Reads firm-wide X digest (Layer 2), filters through personal preferences (Layer 3) |
| Engagement drafting | Drafts tweets, replies, quotes in the person's voice (Layer 4) |
| Voice note processing | Transcribes voice notes (Whisper), feeds into engagement finder or stores as raw content |
| Create work for others | Creates Paperclip issues, assigns to operational agents ("draft me a LinkedIn post about X" -> creates issue for Clawrence) |
| Manage issues via conversation | Check status ("what's blocked?"), update issues ("mark AIS-12 done"), create issues ("add a task for Clawson to draft copy for the newsletter"), reassign — all via natural language |
| General assistant | Answers questions about system status, looks up what other agents did, surfaces relevant information |

#### Execution model

```
User interacts via ANY channel:
  SMS (Twilio) ──┐
  Slack DM ──────┤──> Personal agent (persistent OpenClaw session, always running)
  Web chat ──────┘       -> Responds in seconds
                         -> Reads Paperclip API for context
                         -> Updates personal memory
                         -> Creates/updates Paperclip issues
                         -> Logs conversation to Paperclip activity
```

The agent runs as a persistent OpenClaw session — not triggered by Paperclip heartbeats. It registers with Paperclip for tracking and budgets but the conversational loop is direct.

**Hooks (all personal agents):**
- `PreToolUse: personal-x-post-redirect` — blocks direct X API posting; forces Paperclip issue creation for Clawson instead (P0)
- `PreCompact: state-save` — saves conversation state, pending delegations, and unsaved memory before context compaction (P1)
- `Stop: memory-consolidation` — extracts preference updates from conversation, appends to `data/memory/{name}.md` (P2)
- `Stop: activity-logger` — logs conversation actions to Paperclip activity (async, P1)

#### Multi-channel interface

Each personal agent is reachable through multiple channels. The agent is the same — only the transport differs.

| Channel | How | Best for |
|---|---|---|
| **Slack DM** | Slack bot routes DMs to the person's OpenClaw agent | Quick status checks, issue management, team context ("what's Clawson working on?") |
| **SMS (Twilio)** | Twilio webhook routes inbound SMS to the agent | On-the-go, voice notes, quick opinions |
| **Web chat** | Direct OpenClaw chat interface | Longer conversations, detailed work |
| **Paperclip UI** | Not conversational — direct issue management | Full dashboard, bulk operations |

Slack becomes another **window into Paperclip** — team members can `@clawdia what's the status of the compliance pipeline?` in Slack and get an answer drawn from Paperclip's live data, without opening the Paperclip UI.

#### Skills for personal agents

Each personal agent gets:
- `personal-assistant` — base skill: how to use Paperclip API, how to read other agents' work, how to create issues
- `personal-memory` — how to store and recall preferences, how to update memory files
- `x-engagement` — how to use X discovery data, draft tweets, handle voice notes
- Role-specific skills based on the person (Riyanka gets `compliance-daily-digest` awareness, Zeno gets strategy context, etc.)

#### Agent assignment from existing pool

Reassign 5 of the current 10 OpenClaw agents as personal agents:

| Current Agent | Becomes | Person |
|---|---|---|
| Clawsten | Zeno's Agent | Zeno |
| Clawford | Andrew's Agent | Andrew |
| Clawrelia | Riyanka's Agent | Riyanka |
| Clawvius | Chloe's Agent | Chloe |
| Clawvis | Sydney's Agent | Sydney |

These agents were previously underutilized (thin roles in research/support). As personal agents they become the primary human interface layer.

---

### W5: Copy Generation & Distribution

> "The System is asked for copy and emails back copy for: Evil Geniuses Discord, comingsoon.peak6.com, website subscribers, recruiter lists"

**What it does:** On demand, generates tailored copy for different audiences and distributes via email/Discord.

**Agent:** Clawson (Content Lead)

**Tools needed:**
- LLM (the agent itself handles generation)
- AgentMail (`clawsonmail@agentmail.to`) — send copy to distribution lists
- Discord webhook URL (for Evil Geniuses server — stored in `config/discord.json` in repo)
- Distribution lists (stored in `config/distribution-lists.json` in repo)

**Skill:** `copy-generation` — tone per audience, format per channel, character limits, compliance rules

**Trigger:** Manual (create an issue asking for copy), @-mention, or personal agent creates issue on behalf of user

**Human-in-the-loop:** Agent creates AgentMail **draft** (not sent). Posts draft preview as Paperclip issue comment. Human reviews and approves in Paperclip. Agent sends on approval. No unsupervised distribution to external lists.

**Hooks:**
- `PreToolUse: content-distribution-gate` — blocks AgentMail send and Discord webhook unless linked Paperclip issue status = `done` (P0)
- `PostToolUse: issue-creation-reminder` — after content generation, reminds to create Paperclip issue with `in_review` status (P3)
- `PostToolUse: post-send-checklist` — enforces Paperclip logging and issue status update after send (P1)

---

### W6: LinkedIn Content Pipeline

> "Taking a google drive link of content and creating captions and posts that save in a google doc to send compliance"

**What it does:** Takes raw content from Google Drive, generates LinkedIn captions, saves to a Google Doc for compliance review before posting.

**Agent:** Clawrence (LinkedIn & Compliance)

**Tools needed:**
- Google Drive API (read content from shared links)
- Google Docs API (write captions to a compliance review doc)
- LLM (caption generation)

**Skill:** `linkedin-pipeline` — how to read Drive content, caption format, compliance doc structure, where to save

**Trigger:** Issue created with a Google Drive link in the description

**Human-in-the-loop:** Riyanka reviews generated captions in Google Doc before posting to LinkedIn. The agent never posts directly. This is the best-designed workflow — ship first.

**Content staging:** Agent creates Paperclip issue with status `in_review` when content is ready for compliance. This feeds into W7.

**Hooks:**
- `PostToolUse: issue-creation-reminder` — after Google Docs write, reminds to create Paperclip issue with `in_review` status (P3)
- `PostToolUse: brand-voice-validator` — validates generated captions against `config/brand-voice.json` (async, P2)

---

### W7: Social Compliance Daily Digest

> "The System sends Riyanka a daily link + caption with summary of posts to post into social-compliance"

**What it does:** Daily aggregation of pending social posts with captions, sent to Riyanka for compliance review.

**Agent:** Clawrence (same as W6)

**Tools needed:**
- AgentMail (`strategies@agentmail.to`) — send daily digest to Riyanka
- Paperclip API — query issues with status `in_review` and label `content`

**Data source:** All content workflows (W5, W6, W8) create Paperclip issues with status `in_review` when content is ready for compliance. W7 simply queries: `GET /api/companies/{id}/issues?status=in_review` and aggregates.

**Skill:** `compliance-daily-digest` — what to include, format, how to aggregate across channels (X, LinkedIn, Discord, email)

**Routine:** Daily afternoon cron (e.g., 3pm)

**Human-in-the-loop:** Riyanka receives digest, reviews each item, approves or rejects in Paperclip (moves issue to `done` or comments with revision request).

**Hooks:**
- `PreToolUse: empty-digest-blocker` — blocks sending if no `in_review` issues exist (P2)
- `PostToolUse: post-send-checklist` — enforces Paperclip logging after digest send (P1)

---

### W8: X Content Pipeline

> "Taking a google drive link of content and creating captions and posts that save in peoples X drafts"

**What it does:** Takes content from Google Drive, generates X posts, stages for human review before posting.

**Agent:** Clawson (Content Lead)

**Tools needed:**
- Google Drive API (read content)
- X API v2 (post tweets on approval — not drafts, since X has no public drafts API)
- LLM (post generation)

**Skill:** `x-content-pipeline` — brand voice, character limits, hashtag library, staging format

**Trigger:** Issue with Drive link, or Routine pulling from a shared content queue

**Content staging:** Agent creates Paperclip issue with status `in_review`, includes the draft tweet text in the issue description. No X drafts API exists — staging happens in Paperclip. Human approves in Paperclip, then either:
- Clawson posts via X API (if the content was pre-approved by compliance)
- Human copies and posts manually (if compliance requires manual posting)

**Sydney's content:** Sydney emails or messages her content to her personal agent. The personal agent creates a Paperclip issue with the content, assigns to Clawson for formatting.

**Hooks:**
- `PreToolUse: content-distribution-gate` — blocks X API post unless linked Paperclip issue status = `done` (P0)
- `PostToolUse: brand-voice-validator` — checks generated posts against `config/brand-voice.json` (async, P2)
- `PostToolUse: post-send-checklist` — enforces Paperclip logging and issue status update after post (P1)

---

### W9: Voice Note to Engagement — REDESIGNED

> "User sends the System a voice note and she finds the relevant tweets to reply or comment or requote in the last 24 hours and if she does not find it, draft the tweet as is"

**Revised approach:** This is now a capability of personal agents (Layer 4 of X Intelligence), not a standalone workflow with a dedicated agent.

**How it works:**

1. User sends voice note to their personal agent (via Twilio MMS or web chat)
2. Personal agent transcribes (Whisper/xAI)
3. Personal agent runs **Engagement Finder**:
   - Uses xAI x_search for semantic matching against the voice note content
   - Uses X API v2 to get tweet objects with engagement metrics
   - Filters to last 24h, ranks by relevance + engagement + author quality
4. If matches found: presents ranked list with draft reply/quote for each
5. If no matches: drafts standalone tweet from the voice note content
6. User approves → personal agent creates issue for Clawson to post, OR posts directly via X API

**Tools needed (on each personal agent):**
- Speech-to-text (Whisper or xAI)
- xAI x_search (semantic tweet matching)
- X API v2 (tweet objects, metrics, posting)

**Skill:** `x-engagement` — how to interpret voice notes, search strategy, reply vs. quote vs. standalone decision tree, draft format, approval flow

**Hooks:** Inherits personal agent hooks — `personal-x-post-redirect` (P0), `memory-consolidation` (P2), `activity-logger` (P1)

---

### W11: TikTok Trend Research

> "The System looks at peoples TikToks to find out what trends are happening on TikTok and give content suggestions"

**What it does:** Monitors TikTok for trends, gives content suggestions to the team.

**Agent:** Clawdius (X Discovery Engine — expanded to cover TikTok as part of general trend intelligence)

**Tools needed:**
- Web research (manual browsing via OpenClaw — no TikTok API for MVP)
- LLM (trend analysis, content suggestions)

**Skill:** `trend-research` — which TikTok accounts to monitor, how to identify trends, suggestion format, how to connect TikTok trends to X content opportunities

**Routine:** Daily or twice-daily cron, runs alongside X discovery

**MVP approach:** Agent uses web browsing to check TikTok manually. No API integration. Upgrade to TikTok API later if the workflow proves valuable.

**Output:** Trend suggestions stored as Paperclip issue comments on the daily discovery digest issue. Personal agents surface relevant trends to their person.

**Hooks:** Inherits Clawdius hooks — `clawdius-read-only` (P0), `corpus-storage-verifier` (P3)

---

## Agent Assignment Map

### Operational Agents (heartbeat-based)

| Agent | Role | Workflows | Key Tools |
|---|---|---|---|
| **Clawson** | Content Lead | W5 (copy + distribution), W8 (X content pipeline) | AgentMail (`clawsonmail`), X API, Google Drive, Discord webhook |
| **Clawrence** | LinkedIn & Compliance | W6 (LinkedIn pipeline), W7 (compliance digest) | AgentMail (`strategies`), Google Drive/Docs API |
| **Clawdius** | X & Trend Discovery | W3 Layers 1-2 (daily X intelligence), W11 (TikTok trends) | X API v2, xAI x_search, web research |
| **Clawthorne** | PR Email Monitor | W2 (PR follow-ups) | AgentMail (`clawthorne`), AgentMail labels/threads |

### Personal Agents (persistent sessions — real-time)

| Agent | Person | Key Capabilities |
|---|---|---|
| **Clawsten** | Zeno | Personal assistant, personalized X feed, engagement drafting, voice notes, personal memory |
| **Clawford** | Andrew | Same |
| **Clawrelia** | Riyanka | Same + awareness of compliance pipeline, daily digest context |
| **Clawvius** | Chloe | Same |
| **Clawvis** | Sydney | Same + content intake (Sydney sends her content to her agent) |

### Available

| Agent | Status |
|---|---|
| **Clawden** | Unassigned. Available for W1 (stale task nudges) if needed later, or can become a 6th personal agent. |

### AgentMail Inbox Allocation (3/3 free tier)

| Inbox | Agent(s) | Purpose |
|---|---|---|
| `clawsonmail@agentmail.to` | Clawson | Content distribution, copy emails |
| `clawthorne@agentmail.to` | Clawthorne | PR email monitoring, thread tracking |
| `strategies@agentmail.to` | Clawrence | Compliance digest outbound |

Personal agents do NOT need email inboxes — they communicate via SMS (Twilio) or web chat, and delegate email tasks to operational agents.

---

## X Intelligence System

### Overview

A four-layer system combining X API v2 (structured data) and xAI x_search (semantic understanding):

```
Layer 1: RAW DATA COLLECTION (Clawdius, daily 6am cron)
  X API: keyword searches, tracked accounts, hashtag monitoring
  xAI x_search: semantic discovery on finance/trading topics
  Output: raw tweet corpus → data/x-intelligence/YYYY-MM-DD.json

Layer 2: FIRM-WIDE CURATION (Clawdius, daily 7am cron)
  LLM filters and ranks raw corpus
  Groups by theme, identifies engagement opportunities
  Output: firm digest → Paperclip issue with status in_review

Layer 3: PERSONAL CURATION (personal agents, on-demand or daily push)
  Filters firm digest through personal memory/preferences
  "Zeno, here are 3 threads you'd want to weigh in on today"
  Delivered via SMS/chat

Layer 4: ENGAGEMENT EXECUTION (personal agents, on-demand)
  User picks a tweet or sends voice note
  Agent drafts reply/quote using brand voice + personal voice
  User approves → Clawson posts via X API
```

### How X API and xAI x_search complement each other

| Tool | Strength | Limitation |
|---|---|---|
| X API v2 search | Keyword/hashtag search, user timelines, engagement metrics (likes, RTs), structured tweet objects | Keyword-only — misses semantically related content |
| xAI x_search | Semantic understanding ("tweets about institutional investors questioning market structure"), nuanced topic matching | Less structured — LLM-interpreted results, not raw tweet objects |
| Combined | xAI finds relevant conversations -> X API gets actual tweet objects with metrics -> LLM ranks and drafts responses | Requires both API keys |

### Connection to /last30days pattern

The Discovery Engine follows the same pattern as the `/last30days` skill: topic in, curated notable content out. But focused on X, run daily instead of on-demand, and with firm-specific + personal filtering.

---

## External Integrations

### Decision: Email = AgentMail, Google = Drive/Docs/Calendar only, Slack deferred

We use **AgentMail** (agentmail.to) for all email — not Gmail, Resend, or SendGrid. Reasons:
- Per-agent inboxes via API (each agent gets its own email address)
- Inbound webhooks (instant notification when email arrives, triggers Paperclip Routines)
- Thread tracking as a first-class API resource (agents can follow multi-message conversations)
- Drafts API for human-in-the-loop compliance review
- MCP integration available

We use **Google** (one service account) for Drive, Docs, and Calendar — not email.

We use **Slack** as a conversational interface to personal agents. One Slack app with Socket Mode, bot routes DMs to the correct personal agent's OpenClaw instance. Not used for push notifications or channels — purely as a chat interface into the system.

### AgentMail Setup

**Current state:** Three inboxes (free tier limit):
- `clawsonmail@agentmail.to` — Clawson (content)
- `clawthorne@agentmail.to` — Clawthorne (PR)
- `strategies@agentmail.to` — Clawrence (compliance)

**API key format:** `am_*` prefix. Get from [console.agentmail.to](https://console.agentmail.to) -> API Keys. Shown only once — copy immediately.

**Key scoping:** Org-wide, pod-scoped, or inbox-scoped. Use inbox-scoped keys for production (one key per agent/inbox).

#### Install AgentMail skill on remote OpenClaw

Chat with any OpenClaw agent and send:

```
Install the agentmail skill from ClawHub by running:

npx clawhub@latest install agentmail

Then verify it installed by checking that ~/.openclaw/skills/agentmail/ exists and contains a SKILL.md file. List its contents so I can confirm.
```

#### Configure the API key

Chat with the agent and send:

```
Set up AgentMail credentials. Run these commands:

1. Create the config directory if it doesn't exist:
mkdir -p ~/.openclaw/workspace

2. Write the AgentMail config:
cat > ~/.openclaw/workspace/agentmail-config.json << 'EOF'
{
  "apiKey": "am_XXXXX",
  "defaultInboxId": "INBOX_ID_HERE",
  "defaultFrom": "clawsonmail@agentmail.to"
}
EOF

3. Also export as an environment variable for SDK usage:
echo 'export AGENTMAIL_API_KEY="am_XXXXX"' >> ~/.bashrc

4. Verify the API key works:
curl -s -H "Authorization: Bearer am_XXXXX" https://api.agentmail.to/v1/inboxes | head -20

Show me the output of step 4 so I can confirm it's working.
```

#### Provision for Paperclip heartbeats

Add to each operational agent's Paperclip adapter config env:

```json
{
  "env": {
    "AGENTMAIL_API_KEY": "am_XXXXX",
    "AGENTMAIL_INBOX_ID": "inbox_id",
    "AGENTMAIL_FROM": "clawsonmail@agentmail.to"
  }
}
```

#### Real-time options

| Method | Use case | How |
|---|---|---|
| **Webhooks** | Inbound email triggers Paperclip Routine | AgentMail console: message.received -> POST to Paperclip Routine webhook URL |
| **WebSockets** | Low-latency flows | Python/TS SDK: `client.websocket.connect()` with event handlers |
| **Polling** | Simple inbox checks during heartbeats | `GET /v1/inboxes/{id}/messages?unread=true` |

### Google Setup

One Google Cloud service account with:
- Google Drive API (read content for W6, W8)
- Google Docs API (write compliance docs for W6)
- Google Calendar API (if needed later)

Only provision to agents that need it. Add to adapter config env:
```json
{
  "env": {
    "GOOGLE_SERVICE_ACCOUNT_JSON": "{...the JSON key...}"
  }
}
```

### Full Integration Map

| Integration | Credential | Provisioned via | Used by |
|---|---|---|---|
| AgentMail (`clawsonmail`) | API key (own inbox) | Adapter config `env` | Clawson (W5, W8) |
| AgentMail (`clawthorne`) | API key (own inbox) | Adapter config `env` | Clawthorne (W2) |
| AgentMail (`strategies`) | API key | Adapter config `env` | Clawrence (W7) |
| Google (Drive/Docs) | Service account JSON | Adapter config `env` | Clawson (W8), Clawrence (W6) |
| X API v2 | OAuth tokens | Adapter config `env` | Clawdius (W3), Clawson (W8), personal agents |
| xAI API (x_search) | API key | Adapter config `env` | Clawdius (W3), personal agents |
| Twilio (SMS/MMS) | Account SID + Auth Token | Adapter config `env` | Personal agents |
| Slack (Bot) | Bot token + signing secret | Adapter config `env` | Personal agents (DM channel per person) |
| Speech-to-text (Whisper) | OpenAI API key or xAI | Adapter config `env` | Personal agents |
| Discord | Webhook URL | Repo config file | Clawson (W5) |

---

## Routines (Scheduled Work)

| Routine | Schedule | Agent | Workflow |
|---|---|---|---|
| X raw data collection | Daily 6am | Clawdius | W3 Layer 1 |
| X firm-wide curation | Daily 7am | Clawdius | W3 Layer 2 |
| TikTok trend scan | Daily 8am | Clawdius | W11 |
| PR follow-up digest | Daily 9am | Clawthorne | W2 |
| Compliance daily digest | Daily 3pm | Clawrence | W7 |

---

## Hooks

### Why hooks, not prompts

LLMs forget prompt instructions ~20% of the time. Hooks enforce behavior at the tool level — the agent physically cannot skip them. For anything where a failure means content reaches the outside world, a hook is mandatory.

### Hook types

- **PreToolUse** — runs before a tool executes. Can **block** (exit code 2) or **warn** (stderr). Use for compliance gates and safety rails.
- **PostToolUse** — runs after a tool completes. Cannot block, but injects reminders into agent context. Use for follow-through checklists.
- **Stop** — runs after each agent response. Use for memory consolidation, activity logging, cost tracking.
- **PreCompact** — runs before context window compaction. Use for saving state in long-running personal agent sessions.
- **SessionStart** — runs at session start. Use for loading agent-specific context (brand voice, contacts, memory).

### Hook implementation

Hooks are Node.js scripts that receive tool input as JSON on stdin and output JSON on stdout. They run on the OpenClaw agent's VM.

```javascript
// Example: content-distribution-gate.js
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', async () => {
  const input = JSON.parse(data);
  const cmd = input.tool_input?.command || '';

  // Detect outbound send API calls
  if (/agentmail.*send|api\.twitter.*tweet|discord.*webhook/.test(cmd)) {
    // Extract Paperclip issue ID from context
    const issueMatch = cmd.match(/AIS-\d+/);
    if (issueMatch) {
      const res = await fetch(
        `${PAPERCLIP_URL}/api/issues/${issueMatch[0]}`,
        { headers: { 'Authorization': `Bearer ${PAPERCLIP_API_KEY}` }}
      );
      const issue = await res.json();
      if (issue.status !== 'done') {
        console.error(`[Hook] BLOCKED: Issue ${issueMatch[0]} status is "${issue.status}", not "done". Content must be approved before distribution.`);
        process.exit(2);
      }
    } else {
      console.error('[Hook] BLOCKED: No Paperclip issue ID found. All outbound content must be linked to an approved issue.');
      process.exit(2);
    }
  }
  console.log(data);
});
```

### Priority tiers

| Tier | Meaning | Ships with |
|---|---|---|
| **P0** | Failure = content reaches external parties unsupervised | Phase 1 (Clawthorne) |
| **P1** | Failure = audit trail gaps, lost state, skipped cleanup | Phase 2 (Clawrence) |
| **P2** | Failure = quality drift, empty digests, budget overruns | Phase 2-3 |
| **P3** | Failure = wasted API quota, duplicate runs, state tracking gaps | Iterative |

### Full hook inventory

#### P0 — Compliance & Safety Gates (PreToolUse, blocking)

| Hook | Matcher | Agents | Workflows | Behavior |
|---|---|---|---|---|
| `content-distribution-gate` | Bash | Clawson, Clawrence | W5, W6, W8 | Blocks AgentMail send, X API post, Discord webhook unless linked Paperclip issue status = `done` |
| `clawdius-read-only` | Bash | Clawdius | W3, W11 | Blocks any X API write/post calls. Clawdius is discovery-only — never posts |
| `personal-x-post-redirect` | Bash | All personal agents | W4, W9 | Blocks direct X API posting from personal agents. Forces Paperclip issue creation for Clawson |
| `config-protection` | Write\|Edit | All agents | Cross-cutting | Blocks agent writes to `config/*.json`. Config changes come from human commits only |
| `secret-leak-prevention` | Write\|Edit | All agents | Cross-cutting | Blocks writes containing API key patterns (`am_*`, `sk-*`, `xai-*`, `pcp_*`) |

#### P1 — Audit & State Integrity (PostToolUse + Lifecycle)

| Hook | Type | Agents | Workflows | Behavior |
|---|---|---|---|---|
| `post-send-checklist` | PostToolUse | Clawson, Clawthorne, Clawrence | W2, W5, W7, W8 | After any AgentMail send or API post: reminds to label/archive threads, log to Paperclip, update issue status |
| `activity-logger` | Stop (async) | All agents | Cross-cutting | After each response, POST to Paperclip activity log: what was done, tools called, issues touched |
| `state-save` | PreCompact | Personal agents | W4 | Before context compaction: saves conversation state, pending delegations, unsaved memory to `.session-state` |

#### P2 — Quality & Efficiency

| Hook | Type | Agents | Workflows | Behavior |
|---|---|---|---|---|
| `empty-digest-blocker` | PreToolUse (block) | Clawthorne, Clawrence | W2, W7 | Blocks sending if digest body has no content items or no `in_review` issues exist |
| `brand-voice-validator` | PostToolUse (async) | Clawson, Clawrence | W5, W6, W8 | Validates generated content against `config/brand-voice.json`: character limits, hashtag count, banned phrases |
| `memory-consolidation` | Stop | Personal agents | W4 | Extracts preference updates from conversation, appends to `data/memory/{name}.md` |
| `cost-tracker` | Stop (async) | All agents | Cross-cutting | Emits token count + external API call count to telemetry file for budget enforcement |

#### P3 — Operational Hygiene

| Hook | Type | Agents | Workflows | Behavior |
|---|---|---|---|---|
| `thread-label-enforcer` | PostToolUse | Clawthorne | W2 | Reminds to classify threads (`needs-followup`/`responded`/`escalated`) before generating digest |
| `issue-creation-reminder` | PostToolUse | Clawson, Clawrence | W5, W6, W8 | After content generation, reminds to create Paperclip issue with `in_review` status |
| `duplicate-run-blocker` | PreToolUse (block) | Clawdius | W3 | Blocks if `data/x-intelligence/YYYY-MM-DD.json` already exists today |
| `corpus-storage-verifier` | PostToolUse | Clawdius | W3 | After search API calls, verifies corpus was written as valid JSON with expected fields |
| `data-directory-protection` | PreToolUse (block) | All agents | Cross-cutting | Blocks `rm`, `mv`, or destructive overwrites on `data/` directory |
| `context-loader` | SessionStart | All agents | Cross-cutting | Loads agent-specific config (brand voice, contacts, memory) into context at session start |

### Hook-to-agent provisioning

Each agent's OpenClaw config gets only the hooks relevant to its role:

```
Clawthorne:  empty-digest-blocker, post-send-checklist, thread-label-enforcer,
             config-protection, secret-leak-prevention, activity-logger,
             cost-tracker, context-loader

Clawrence:   content-distribution-gate, empty-digest-blocker, post-send-checklist,
             issue-creation-reminder, brand-voice-validator, config-protection,
             secret-leak-prevention, activity-logger, cost-tracker, context-loader

Clawson:     content-distribution-gate, post-send-checklist, issue-creation-reminder,
             brand-voice-validator, config-protection, secret-leak-prevention,
             activity-logger, cost-tracker, context-loader

Clawdius:    clawdius-read-only, duplicate-run-blocker, corpus-storage-verifier,
             config-protection, secret-leak-prevention, activity-logger,
             cost-tracker, context-loader

Personal:    personal-x-post-redirect, state-save, memory-consolidation,
             activity-logger, config-protection, secret-leak-prevention,
             cost-tracker, context-loader, data-directory-protection
```

---

## Skills Needed

### Operational skills

| Skill | Used by | Purpose |
|---|---|---|
| `agentmail` (ClawHub) | Clawson, Clawthorne, Clawrence | Installed via `npx clawhub@latest install agentmail` |
| `google-apis` | Clawson, Clawrence | Read Drive files, write Docs via service account |
| `x-discovery-engine` | Clawdius | X API + xAI search queries, firm-wide curation, output format |
| `pr-followup` | Clawthorne | Parse PR emails, identify follow-ups, reminder format |
| `copy-generation` | Clawson | Tone per audience, format per channel, compliance rules, draft-before-send |
| `linkedin-pipeline` | Clawrence | Read Drive content, caption format, compliance doc structure |
| `compliance-daily-digest` | Clawrence | Aggregate `in_review` content, format digest |
| `x-content-pipeline` | Clawson | Brand voice, hashtags, staging in Paperclip issues |
| `trend-research` | Clawdius | TikTok monitoring, cross-platform trend identification |

### Personal agent skills

| Skill | Used by | Purpose |
|---|---|---|
| `personal-assistant` | All personal agents | Paperclip API usage, read other agents' work, create issues, delegate |
| `personal-memory` | All personal agents | Store/recall preferences, update memory files, personalization |
| `x-engagement` | All personal agents | Personalized X feed curation, engagement finder, voice note -> tweet drafting, approval flow |

---

## Community Skills (ClawHub)

Install community skills from [ClawHub](https://clawhub.com) to accelerate builds. These replace or supplement custom skills. Source: [awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) (5,211 curated skills).

### Install method

```bash
# On each agent's remote VM:
npx clawhub@latest install <skill-slug>

# Skills install to ~/.openclaw/skills/<skill-name>/
# Workspace-scoped skills (project/skills/) override global skills
```

### Per-agent community skills

#### Clawthorne (PR Email Monitor)

| Community Skill | Install | Replaces/Supplements |
|---|---|---|
| `agentmail` | `npx clawhub@latest install agentmail` | Email send/receive/threads (already planned) |
| `expanso-email-triage` | `clawhub install azvast-expanso-email-triage` | AI-powered email triage with calendar sync — supplements custom `pr-followup` |
| `postwall` | `clawhub install casperaiassist-postwall` | Human-in-the-loop approval gate before outbound sends |
| `daily-brief-digest` | `clawhub install rajtejani61-daily-brief-digest` | Morning summary template — supplements digest format |

#### Clawdius (X & Trend Discovery)

| Community Skill | Install | Replaces/Supplements |
|---|---|---|
| `readx` | `clawhub install wxtsky-readx` | Most feature-rich X intelligence toolkit: user analysis, trend tracking, community mapping |
| `bird` | `clawhub install steipete-bird` | X CLI for reading, searching, posting (by @steipete) |
| `kiro-x-publisher` | `clawhub install vmining-kiro-x-publisher` | Discover topics on X, enrich tweets, generate drafts — supplements `x-discovery-engine` |
| `social-intelligence` | `clawhub install atyachin-social-intelligence` | Cross-platform research: Twitter + Instagram + Reddit |

#### Clawson (Content Lead)

| Community Skill | Install | Replaces/Supplements |
|---|---|---|
| `brand-voice-profile` | `clawhub install dimitripantzos-brand-voice-profile` | Persistent brand voice storage and enforcement |
| `blogburst` | `clawhub install shensi8312-blogburst` | Convert articles into 10+ social media posts — supplements `copy-generation` |
| `content-creator` | `clawhub install alirezarezvani-content-creator` | SEO-optimized content with consistent branding |
| `brw-de-ai-ify` | `clawhub install brianrwagner-brw-de-ai-ify` | Remove AI jargon, restore human voice — de-sloppify pass for content |

#### Clawrence (LinkedIn & Compliance)

| Community Skill | Install | Replaces/Supplements |
|---|---|---|
| `clawemail` | `clawhub install cto1-clawemail` | Google Workspace: Gmail, Drive, Docs, Sheets — supplements `google-apis` |
| `brw-newsletter-creation-curation` | `clawhub install brianrwagner-brw-newsletter-creation-curation` | B2B newsletter creation, industry-adaptive — supplements `compliance-daily-digest` |

#### Personal Agents

| Community Skill | Install | Replaces/Supplements |
|---|---|---|
| `voice-notes-pro` | `clawhub install toniaczlog-voice-notes-pro` | Intelligent transcription + categorization of voice notes |
| `auto-whisper-safe` | `clawhub install neal-collab-auto-whisper-safe` | RAM-safe transcription with auto-chunking (works on 16GB VMs) |
| `session-watchdog` | `clawhub install session-watchdog` | Monitor context levels, save checkpoints — supplements `state-save` hook |
| `task-resume` | `clawhub install task-resume` | Automatic interrupted-task recovery |
| `close-loop` | `clawhub install close-loop` | End-of-session memory consolidation workflow |
| `adaptlypost` | `clawhub install adaptlypost` | Schedule posts across X, LinkedIn, Instagram, TikTok |
| `pine-voice` | `clawhub install bojieli-pine-voice` | Give agent a real phone number (alternative to Twilio setup) |

### Skill layering (precedence order)

```
1. Workspace-scoped skills (project/skills/)     — HIGHEST: custom domain skills
2. Global skills (~/.openclaw/skills/)            — community skills from ClawHub
3. Built-in skills                                — OpenClaw defaults
```

Custom skills (e.g., `pr-followup`) can reference community skills (e.g., `agentmail`) in their instructions. The custom skill adds PEAK6-specific logic on top of the community skill's general capabilities.

---

## Execution Patterns

### Why sequential pipelines

Operational agents run on Paperclip's heartbeat model: wake up, do a task, sleep. The best execution pattern is the **sequential pipeline** — each heartbeat runs a chain of focused steps. Each step gets a fresh context window, preventing bleed between stages.

### Pattern: Sequential pipeline (`claude -p` chain)

```bash
#!/bin/bash
# Each step: isolated context, builds on filesystem state from previous step
set -e

# Step 1: Fetch data
claude -p "Step 1 instructions. Save output to /tmp/step1-output.json"

# Step 2: Process data
claude -p "Read /tmp/step1-output.json. Process it. Save to /tmp/step2-output.json"

# Step 3: Generate deliverable
claude -p "Read /tmp/step2-output.json. Generate the deliverable. Save draft to /tmp/draft.md"

# Step 4: Distribute + log
claude -p "Read /tmp/draft.md. Send via API. Log activity to Paperclip."
```

**Key principles:**
- **Fresh context per step** — no context bleed between stages
- **Filesystem bridges context** — `.tmp` files pass state between steps
- **Exit codes propagate** — `set -e` stops on failure
- **Model routing** — use Haiku for classification, Sonnet for generation, Opus for complex reasoning

### Agent execution recipes

#### Clawthorne — W2 daily digest (9am cron)

```bash
#!/bin/bash
set -e

# Step 1: Fetch unread threads from AgentMail
claude -p "Fetch all unread email threads from AgentMail inbox (clawthorne@agentmail.to).
  Use GET /v1/inboxes/{id}/threads?unread=true.
  Save thread list to /tmp/unread-threads.json"

# Step 2: Classify threads (4-tier system)
claude -p "Read /tmp/unread-threads.json. Classify each thread:
  - skip: automated notifications, no-reply senders
  - info_only: CC'd threads, announcements, no action needed
  - meeting_info: event invitations, scheduling (cross-ref with calendar)
  - action_required: direct asks from PR contacts, pending RSVPs, quote requests
  Save classified results to /tmp/classified.json"

# Step 3: Generate digest
claude -p "Read /tmp/classified.json. Generate daily PR digest for Riyanka.
  Include only action_required and meeting_info items.
  Format: sender | subject | last action | days since response | recommended next step.
  Use template from templates/compliance-digest.md.
  Save to /tmp/digest-draft.md"

# Step 4: Send digest + label threads
claude -p "Read /tmp/digest-draft.md. If it has content items:
  1. Send via AgentMail to riyanka@peak6.com
  2. Label processed threads: needs-followup, responded, or escalated
  3. Log activity to Paperclip (POST /api/companies/{id}/activity)
  4. Update related Paperclip issue status
  If digest is empty, skip sending and log 'no items today' to Paperclip."
```

#### Clawdius — W3 Layer 1+2 (6am + 7am crons)

```bash
#!/bin/bash
set -e

# === LAYER 1: Raw Data Collection (6am) ===

# Step 1: X API structured search
claude -p "Read config/x-discovery.json for search queries, tracked accounts, monitored hashtags.
  Run X API v2 searches: recent tweets matching queries, user timelines for tracked accounts.
  Collect tweet objects with engagement metrics (likes, RTs, replies).
  Save to /tmp/x-api-results.json"

# Step 2: xAI semantic discovery
claude -p "Read config/x-discovery.json for semantic search topics.
  Run xAI x_search for each topic (e.g., 'institutional investors questioning market structure').
  Save to /tmp/xai-results.json"

# Step 3: Merge and store corpus
claude -p "Read /tmp/x-api-results.json and /tmp/xai-results.json.
  Merge, deduplicate by tweet ID, add source field (x_api vs xai).
  Validate: each entry has tweet_id, author, text, metrics, relevance_score.
  Write to data/x-intelligence/$(date +%Y-%m-%d).json.
  Write bridge notes to data/x-intelligence/task-notes.md for Layer 2."

# === LAYER 2: Firm-Wide Curation (7am) ===

# Step 4: Curate and rank
claude -p "Read data/x-intelligence/$(date +%Y-%m-%d).json and data/x-intelligence/task-notes.md.
  Group tweets by theme: market structure, fintech, regulation, trading tech, etc.
  Identify engagement opportunities (high-value threads worth joining).
  Highlight what aligns with PEAK6 content pillars (read config/brand-voice.json).
  Rank by: relevance to PEAK6 * engagement potential * author quality.
  Save curated digest to /tmp/firm-digest.md"

# Step 5: Create Paperclip issue
claude -p "Read /tmp/firm-digest.md.
  Create Paperclip issue:
    POST /api/companies/{id}/issues
    title: 'X Intelligence Digest — $(date +%Y-%m-%d)'
    status: in_review
    description: [the curated digest content]
  Log activity to Paperclip."
```

#### Clawson — W5 copy generation (on-demand, issue-triggered)

```bash
#!/bin/bash
set -e

# Step 1: Read the request
claude -p "Read the assigned Paperclip issue description.
  Extract: target audience, channel (Discord/email/X), topic, any source material links.
  If a Google Drive link is provided, fetch the content via Drive API.
  Save extracted brief to /tmp/copy-brief.json"

# Step 2: Generate copy
claude -p "Read /tmp/copy-brief.json and config/brand-voice.json.
  Generate copy tailored to the target channel:
  - Discord: casual, community-forward, can be longer
  - Email: professional, concise, clear CTA
  - X: ≤280 chars, ≤2 hashtags, brand voice
  Generate 3 variations. Save to /tmp/copy-drafts.md"

# Step 3: De-sloppify pass
claude -p "Read /tmp/copy-drafts.md. Remove:
  - AI-sounding phrases ('In today's landscape', 'It's important to note', 'Dive into')
  - Excessive hashtags (max 2 per X post)
  - Generic calls to action
  - Any tone mismatches with config/brand-voice.json
  Pick the strongest variation. Save final to /tmp/copy-final.md"

# Step 4: Stage for approval
claude -p "Read /tmp/copy-final.md.
  Create AgentMail draft (NOT sent) via Drafts API.
  Post draft preview as Paperclip issue comment.
  Update issue status to in_review.
  Log activity to Paperclip."
```

#### Clawrence — W7 compliance digest (3pm cron)

```bash
#!/bin/bash
set -e

# Step 1: Aggregate pending content
claude -p "Query Paperclip API: GET /api/companies/{id}/issues?status=in_review
  Filter to content-related issues (W5, W6, W8 output).
  Save list to /tmp/pending-content.json"

# Step 2: Generate compliance digest
claude -p "Read /tmp/pending-content.json. If no items, write 'empty' to /tmp/digest-status.txt and exit.
  Otherwise, generate compliance digest:
  For each item: channel (X/LinkedIn/Discord/email), draft text, source workflow, created date.
  Use template from templates/compliance-digest.md.
  Save to /tmp/compliance-digest.md"

# Step 3: Send digest
claude -p "Read /tmp/digest-status.txt. If 'empty', log 'no pending content' to Paperclip and exit.
  Read /tmp/compliance-digest.md.
  Send via AgentMail (strategies@agentmail.to) to riyanka@peak6.com.
  Log activity to Paperclip."
```

### Pattern: De-sloppify pass (add-on for all content workflows)

Never constrain the content generator with negative instructions ("don't sound like AI"). Instead:
1. Let the generator be thorough and creative
2. Add a separate cleanup step that removes AI artifacts

Two focused passes outperform one constrained pass. The `brw-de-ai-ify` ClawHub skill automates this second pass.

### Pattern: Context bridging via task notes

For workflows that span multiple heartbeats (Clawdius Layer 1 at 6am → Layer 2 at 7am):

```markdown
# data/x-intelligence/task-notes.md (bridges between cron runs)

## 2026-03-27 Layer 1 (6am run)
- Collected 847 tweets across 12 search queries
- Top trending: "SEC infrastructure reform" (engagement spike 3.2x)
- xAI x_search found 23 semantic matches not in keyword results
- Corpus saved: data/x-intelligence/2026-03-27.json

## Layer 2 TODO
- Group by: market structure, fintech, regulation, trading tech
- Flag SEC reform cluster for Zeno (personal agent delivery)
- 3 engagement opportunities identified in Layer 1
```

The later cron reads this file, picks up where the earlier run left off. This replaces shared context windows (which don't exist between heartbeats) with filesystem state.

### Pattern: Model routing per step

Route expensive models only to steps that need deep reasoning:

| Step Type | Model | Why |
|---|---|---|
| Fetch/classify data | Sonnet | Pattern matching, structured extraction |
| Content generation | Opus | Creative quality, brand voice adherence |
| De-sloppify pass | Haiku | Simple removal, pattern matching |
| Digest aggregation | Sonnet | Structured transformation |
| Semantic discovery | Opus | Nuanced topic understanding |
| Issue creation/logging | Haiku | Templated API calls |

```bash
# Example: model routing in pipeline
claude -p --model sonnet "Classify these email threads..."
claude -p --model opus "Generate copy for the newsletter..."
claude -p --model haiku "Remove AI artifacts from the draft..."
```

---

## Repo Structure

Private GitHub repo — an ops repo, not a code repo.

```
peak6-attention/
  skills/
    # Operational
    google-apis/SKILL.md
    x-discovery-engine/SKILL.md
    x-content-pipeline/SKILL.md
    linkedin-pipeline/SKILL.md
    copy-generation/SKILL.md
    compliance-daily-digest/SKILL.md
    pr-followup/SKILL.md
    trend-research/SKILL.md

    # Personal agent
    personal-assistant/SKILL.md
    personal-memory/SKILL.md
    x-engagement/SKILL.md

    # NOTE: agentmail skill is installed via ClawHub, not in this repo
    # NOTE: community skills installed via ClawHub go to ~/.openclaw/skills/

  hooks/                             # Hook scripts (Node.js, deployed to each VM)
    content-distribution-gate.js     # P0: blocks sends unless Paperclip issue approved
    clawdius-read-only.js            # P0: blocks X API writes from Clawdius
    personal-x-post-redirect.js      # P0: blocks personal agent X posting
    config-protection.js             # P0: blocks agent writes to config/
    secret-leak-prevention.js        # P0: blocks writes containing API key patterns
    empty-digest-blocker.js          # P2: blocks empty digest sends
    post-send-checklist.js           # P1: post-send follow-through reminder
    brand-voice-validator.js         # P2: validates content against brand-voice.json
    activity-logger.js               # P1: logs actions to Paperclip
    state-save.js                    # P1: saves state before context compaction
    memory-consolidation.js          # P2: extracts preferences from conversations
    cost-tracker.js                  # P2: emits token/API usage telemetry
    context-loader.js                # P3: loads agent-specific config at session start
    thread-label-enforcer.js         # P3: reminds to classify PR email threads
    issue-creation-reminder.js       # P3: reminds to create Paperclip issues
    duplicate-run-blocker.js         # P3: blocks duplicate X corpus collection
    corpus-storage-verifier.js       # P3: verifies X corpus JSON integrity
    data-directory-protection.js     # P3: blocks destructive ops on data/

  config/
    contacts.json                  # Team members + email addresses + phone numbers
    distribution-lists.json        # Email lists per audience
    brand-voice.json               # Tone, hashtags, pillars
    posting-schedule.json          # When each channel gets content
    discord.json                   # Evil Geniuses webhook URL
    google-docs.json               # Compliance doc IDs
    x-discovery.json               # Search queries, tracked accounts, monitored hashtags

  data/
    memory/                        # Per-user memory files (personal agents read/write)
      zeno.md
      andrew.md
      riyanka.md
      chloe.md
      sydney.md
    x-intelligence/                # Daily X discovery corpus
      2026-03-26.json
    content-queue/                 # Staged content waiting for approval
    voice-notes/                   # Transcribed voice notes

  templates/
    compliance-digest.md
    tweet-draft.md
    linkedin-caption.md
    copy-request.md
```

---

## Content Staging Convention

All content workflows (W5, W6, W8) follow the same pattern:

1. Agent generates content
2. Agent creates Paperclip issue with status `in_review`
3. Issue description contains the draft content (tweet text, LinkedIn caption, email copy)
4. W7 (compliance digest) aggregates all `in_review` issues daily
5. Human reviews in Paperclip — approves (moves to `done`) or comments with revision
6. On approval, the producing agent distributes (posts to X, sends email, etc.)

This keeps all pending content visible in one place: the Paperclip Issues page filtered by `in_review`.

---

## Fit Assessment

### Fits Paperclip well (heartbeat + Routine model)

- W2: PR follow-ups — scheduled Routine, inbound webhook, clear deliverable
- W3 Layers 1-2: X Discovery — scheduled Routine, no human interaction during execution
- W5: Copy generation — on-demand issue creation, draft-before-send approval
- W6: LinkedIn pipeline — issue-triggered, compliance review in Google Docs
- W7: Compliance digest — scheduled Routine, aggregates from Paperclip
- W8: X content pipeline — issue-triggered, stages in Paperclip
- W11: TikTok research — scheduled Routine

### Runs outside Paperclip heartbeats (persistent sessions)

- W4: Personal agents — persistent OpenClaw sessions for real-time conversation. Still registered with Paperclip for tracking, budgets, and API access. This is by design, not a workaround.
- W3 Layers 3-4: Personal curation + engagement — handled by personal agents in real-time
- W9: Voice note -> engagement — handled by personal agents in real-time

### Deferred

- W1: Stale task monitoring — team has Paperclip accounts. Revisit if push notifications prove necessary.

---

## Decisions Made

- [x] **Email service:** AgentMail (agentmail.to) — per-agent inboxes, inbound webhooks, thread tracking, drafts API
- [x] **Google:** One service account for Drive/Docs/Calendar only — not Gmail
- [x] **Tracker:** Paperclip IS the tracker — no separate Google Sheets
- [x] **Personal agents:** One per person, persistent OpenClaw sessions (not heartbeat-based), reassigned from underutilized operational agents
- [x] **X intelligence:** Four-layer system using X API v2 + xAI x_search. Firm-wide daily discovery + personalized curation via personal agents
- [x] **Voice notes + engagement:** Capability of personal agents, not a standalone workflow
- [x] **W4 + W10 merged:** Every Clawdia conversation updates personal memory
- [x] **Content staging:** All content goes through Paperclip issues with `in_review` status before distribution
- [x] **Copy approval gate:** Agent creates draft, human approves, then agent sends. No unsupervised distribution.
- [x] **TikTok MVP:** Manual web research first, no API integration
- [x] **Slack:** Personal agent interface via DMs. One Slack app (Socket Mode), routes to OpenClaw instances.
- [x] **W1 deferred:** Not needed with team in Paperclip.
- [x] **Hooks over prompts:** Hook-enforced compliance gates (P0) ship with Phase 1. LLMs forget instructions ~20% of the time; hooks make violations physically impossible.
- [x] **Execution pattern:** Sequential pipelines (`claude -p` chains) for all heartbeat-based operational agents. Fresh context per step, filesystem bridges state.
- [x] **De-sloppify pattern:** All content generation (W5, W6, W8) gets a separate cleanup pass. Two focused passes > one constrained pass.
- [x] **Community skills:** Use ClawHub skills (`readx`, `bird`, `brand-voice-profile`, `postwall`, etc.) to accelerate builds. Custom skills layer PEAK6-specific logic on top.
- [x] **Model routing:** Haiku for classification/templated work, Sonnet for generation/extraction, Opus for creative content and semantic reasoning.
- [x] **Context bridging:** `task-notes.md` files bridge state between consecutive heartbeats (Clawdius Layer 1 → Layer 2).

## Open Questions

- [ ] Twilio account setup for SMS/MMS — who owns this? Needed for personal agents.
- [x] Slack bot setup — Slack app created (App ID: A0AP3LVKTDL), Socket Mode enabled, bot scopes + events configured. Tokens generated. Next: configure Clawford's OpenClaw instance with the tokens.
- [ ] X API v2 access level needed (Basic $100/mo, Pro $5000/mo?) — need search + post endpoints
- [ ] xAI API x_search — confirm this is available on your xAI key, test it
- [ ] Personal agent execution model — confirm OpenClaw supports persistent sessions (not just heartbeat-triggered)
- [ ] TikTok data access strategy — manual web research for MVP, revisit API later
- [ ] Voice note submission method — Twilio MMS? Web form? Voice message in chat?
- [ ] How does Sydney's content get into the system? (email to her personal agent, or direct upload?)
- [ ] Google Drive folder structure — which folders get shared with the service account?
- [ ] Compliance posting — does Riyanka post to LinkedIn/X manually after approval, or does the agent post?
- [ ] AgentMail custom domain — verify `ai.peak6.com` or similar when upgrading from free tier
