---
name: x-discovery-engine
description: >
  Use the x-intelligence plugin tools to curate daily X/Twitter intelligence
  for PEAK6. Use when assigned discovery, curation, or intelligence digest
  tasks. Do NOT use for posting content or engagement — that is handled by
  the content pipeline.
---

# X Discovery Engine

Use this skill when your task involves X/Twitter intelligence: fetching the
daily corpus, curating digests, tracking new authority handles, or answering
questions about what's trending in PEAK6's domains.

## 1. Ground rules

- You are **discovery-only**. Never post, reply, like, or engage on X.
- The x-intelligence plugin runs a daily automated pipeline (6 AM UTC) that
  collects, enriches, scores, and stores tweets. You consume its output — you
  do not run the pipeline yourself.
- Always include `X-Paperclip-Run-Id` on mutating API requests per the
  `paperclip` skill.

## 2. Plugin tool API

Plugin tools are called via the Paperclip plugin tools API, not direct
function calls. The plugin ID is `peak6-labs.x-intelligence`.

### Discover available tools

```bash
curl -sS "$PAPERCLIP_API_URL/api/plugins/tools?pluginId=peak6-labs.x-intelligence" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

### Execute a tool

```bash
curl -sS -X POST "$PAPERCLIP_API_URL/api/plugins/tools/execute" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -d '{
    "tool": "peak6-labs.x-intelligence:<tool-name>",
    "parameters": { ... },
    "runContext": {
      "agentId": "'"$PAPERCLIP_AGENT_ID"'",
      "runId": "'"$PAPERCLIP_RUN_ID"'",
      "companyId": "'"$PAPERCLIP_COMPANY_ID"'",
      "projectId": "<project-id-from-task>"
    }
  }'
```

### Bridge endpoints (alternative)

If the tool dispatcher returns an error, you can call the plugin directly
via bridge endpoints. The plugin database ID is
`fce02f01-608f-4ea0-ae6b-be03f33de44a`.

```bash
# Fetch data (equivalent to get-today, search-corpus, get-authorities, suggest-handles)
curl -sS -X POST "$PAPERCLIP_API_URL/api/plugins/fce02f01-608f-4ea0-ae6b-be03f33de44a/bridge/data" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "<tool-name>", "params": { ... }}'

# Perform action (equivalent to track-handle)
curl -sS -X POST "$PAPERCLIP_API_URL/api/plugins/fce02f01-608f-4ea0-ae6b-be03f33de44a/bridge/action" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "<tool-name>", "params": { ... }}'
```

## 3. Available tools

### peak6-labs.x-intelligence:get-today

Fetch today's scored corpus. Start here for daily curation.

```json
{
  "tool": "peak6-labs.x-intelligence:get-today",
  "parameters": { "limit": 20 }
}
```

Optional parameters: `pillar` (filter by content pillar), `limit` (default 20).

### peak6-labs.x-intelligence:search-corpus

Semantic search across the scored corpus. Use for targeted queries or when
get-today returns empty (weekends, holidays).

```json
{
  "tool": "peak6-labs.x-intelligence:search-corpus",
  "parameters": { "query": "market structure reform", "limit": 10 }
}
```

Optional parameters: `date` (YYYY-MM-DD), `pillar`, `limit` (default 20).

**Weekend/holiday fallback:** If get-today returns empty, use search-corpus
without a date filter or with a 3-day window. xAI returns fewer results on
weekends.

### peak6-labs.x-intelligence:get-authorities

List authority handle lists by domain. Use to understand who the plugin
considers high-signal voices.

```json
{
  "tool": "peak6-labs.x-intelligence:get-authorities",
  "parameters": {}
}
```

Optional: `list_name` (e.g., `"market_structure"`, `"fintech"`,
`"chicago_trading"`, `"venture_capital"`) to get a specific domain's handles.

### peak6-labs.x-intelligence:suggest-handles

Get handles that are candidates for promotion to authority lists based on
tracking data and promotion policy thresholds.

```json
{
  "tool": "peak6-labs.x-intelligence:suggest-handles",
  "parameters": {}
}
```

### peak6-labs.x-intelligence:track-handle

Record a discovered handle for future promotion consideration.

```json
{
  "tool": "peak6-labs.x-intelligence:track-handle",
  "parameters": {
    "handle": "username",
    "relevance": 0.85,
    "domain": "market_structure"
  }
}
```

Parameters: `handle` (without @, required), `relevance` (0–1, required),
`domain` (required — must match an authority list domain).

## 4. Scoring interpretation

Scores range 0–100. Higher is better.

| Range | Signal | Action |
|-------|--------|--------|
| 80+ | High — authority handles or highly relevant content | Always include in digests |
| 60–79 | Moderate — worth surfacing | Include if space permits |
| Below 60 | Low — skip unless uniquely relevant | Omit from digests |

Authority-boosted tweets score higher because they come from vetted domain
experts in the plugin's authority lists.

## 5. Daily curation workflow

When assigned a daily intelligence or digest task:

1. **Fetch corpus.** Call `get-today` with no filters.
2. **Group by pillar.** Organize results into content pillars.
3. **Rank within pillars.** Pick the top 3–5 highest-scored items per pillar.
   Identify engagement opportunities (high-value threads PEAK6 could join).
4. **Check for new voices.** Call `suggest-handles` to surface promotion
   candidates.
5. **Track new discoveries.** For any new high-quality handle found in the
   corpus that isn't already tracked, call `track-handle`.
6. **Write the digest.** Post as a Paperclip issue comment or update the
   issue description using the output format below.

## 6. Output format

**Every tweet or post reference MUST include a direct link** using the `url`
field from tool results (format: `https://x.com/{author}/status/{id}`). This
applies to digests, ad-hoc answers, issue comments, and any other output that
references X content. Never cite a tweet without its link.

```markdown
## Daily X Intelligence — {YYYY-MM-DD}

### Top Stories
1. **@author** (score: 85): "Tweet snippet..." — Why it matters to PEAK6. [View post](https://x.com/author/status/123456)
2. ...

### Market Structure & Regulation
- **@author** (score: 78): "Tweet snippet..." [View post](https://x.com/author/status/123456)
- ...

### Fintech & Trading Technology
- ...

### Venture Capital & Investment
- ...

### Chicago Business Ecosystem
- ...

### Company Culture & Talent
- ...

### Engagement Opportunities
- Thread by @author on [topic] — PEAK6 could add value by [reason]. [View thread](https://x.com/author/status/123456)
- ...

### New Voices
- @handle (relevance: 0.82, domain: fintech) — Reason for tracking. [Profile](https://x.com/handle)
- ...
```

Keep it concise. Leaders will skim this in 2 minutes.

## 7. Answering ad-hoc intelligence questions

When asked a question like "what are people saying about X?" or "find tweets
about Y":

1. Call `search-corpus` with the relevant query.
2. Summarize results with author attribution, scores, and **direct links to
   each post** (use the `url` field from tool results).
3. If results are thin, note the corpus date range and suggest the operator
   trigger a fresh discovery run if needed.

## 8. Content pillars

The plugin organizes content around these domains:

- `market structure & regulation` — SEC reform, market microstructure, options regulation, CFTC
- `fintech & trading technology` — financial technology, payments, AI in finance, algorithmic trading
- `venture capital & investment` — VC deals, startup ecosystem, funding rounds, financial infrastructure
- `Chicago business ecosystem` — Chicago trading community, prop firms, PEAK6 ecosystem
- `company culture & talent` — talent acquisition, company culture, team building

Use these as `pillar` filter values in tool calls. Authority list domains
use short keys (`market_structure`, `fintech`, `chicago_trading`,
`venture_capital`) which map to these pillars.
