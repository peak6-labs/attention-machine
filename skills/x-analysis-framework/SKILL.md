---
name: x-analysis-framework
description: >
  Use when evaluating X/Twitter corpus data for content opportunities. Covers
  how to identify trending conversations, score opportunities, and create
  well-structured issues for content agents. Use for Topic Scout and similar
  sensing roles. Do NOT use for writing content — that is handled by
  x-content-doctrine.
---

# X Analysis Framework

Use this skill when your task involves analyzing the X intelligence corpus
to find content opportunities: identifying trending conversations, evaluating
engagement potential, and creating actionable issues for content agents.

This skill complements x-discovery-engine (which teaches how to fetch and
curate the corpus) by teaching how to evaluate what you find and turn it into
content opportunities.

## 1. Ground rules

- You are a **sensing agent**, not a content agent. Your output is Paperclip
  issues, not drafts. You identify opportunities; content agents act on them.
- Never post, reply, like, or engage on X. You are read-only.
- Always use the x-intelligence plugin tools to ground your analysis in
  scored data. Don't hallucinate trends — the corpus is your source of truth.
- Include `X-Paperclip-Run-Id` on all mutating API requests per the
  `paperclip` skill.

## 2. Tool usage

You use the same x-intelligence plugin tools as the discovery engine. The
plugin ID is `peak6-labs.x-intelligence`. Refer to the x-discovery-engine
skill for full API documentation and curl examples.

### Your primary tools

| Tool | When to use |
|------|------------|
| `get-today` | Start of every analysis cycle. Fetch the full scored corpus. Use `min_score: 0.3` to filter noise. |
| `search-corpus` | Targeted investigation. Filter by pillar, engagement level, time window (`since: "8h"`), or score threshold. |
| `analyze-topic` | On-demand deep dive when you spot a trending topic. Returns 5-perspective synthesis (core, expert voices, pain points, positive signals, link-based) plus corpus cross-reference. |
| `get-thread` | Fetch conversation context when a tweet looks like part of a valuable thread. |
| `get-authorities` | Check if a handle is a known authority before weighing their content. |
| `suggest-handles` | Find emerging voices that might signal a growing conversation. |
| `track-handle` | Record a new high-quality handle found during analysis. |
| `promote-handle` | Promote a validated handle to an authority list. Include a note explaining why. |
| `rate-tweet` | Rate corpus tweets as relevant/irrelevant/skip to improve future discovery quality. |

### Weekend/holiday awareness

The xAI discovery pipeline returns fewer results on weekends and holidays.
If `get-today` returns a thin corpus (<10 items):

1. Use `search-corpus` without a date filter to pull from the last 3 days.
2. Note the date range in your analysis so content agents know the freshness.
3. Don't force opportunities from a thin corpus — "nothing actionable today"
   is a valid output.

## 3. Engagement threshold tiers

Use these thresholds (configurable in plugin settings under `engagement_thresholds`)
to filter signal from noise:

| Tier | Threshold | Action |
|------|-----------|--------|
| **Noise floor** | < 10 total engagements | Skip unless from an authority handle |
| **Signal** | 10-49 engagements | Worth reading, may be opportunity if topic-relevant |
| **Expert** | 50-99 engagements | Likely a quality conversation. Investigate. |
| **Escalation** | 100+ engagements | High-priority. Always investigate and create an issue if relevant. |

Total engagement = likes + reposts + replies + quotes.

When using `search-corpus`, apply these as `min_engagement` filters to focus
your attention.

## 4. Opportunity identification

### What makes a content opportunity?

A content opportunity is a conversation or trend on X where PEAK6 can add
value by participating. Not every interesting tweet is an opportunity.

**Strong opportunities** (create an issue):

- A high-authority account opens a thread on a PEAK6 domain topic and the
  conversation is still active (<12 hours old, multiple replies)
- Multiple corpus items cluster around the same topic, signaling a trend
  (3+ tweets, different authors, same theme)
- A regulatory or market event is breaking and PEAK6 has domain expertise
  (SEC announcement, exchange outage, notable deal)
- A direct mention of PEAK6, its products, or its people (always surface
  these, even if low-scored)
- An authority handle asks a question that PEAK6 can credibly answer

**Weak opportunities** (skip or note in digest, don't create an issue):

- Generic fintech commentary with no unique PEAK6 angle
- Tweets from excluded handles (elonmusk, openai, google, microsoft)
- Stale conversations (>24 hours, no recent replies)
- Topics outside PEAK6's domain boundaries (crypto prices, partisan politics)
- Low-scored tweets (<60) from non-authority handles with no cluster signal

### Cluster detection

When multiple corpus items point to the same topic, that is a signal:

1. **Scan for theme overlap.** After fetching the corpus, group items by
   apparent topic. Look for 3+ tweets from different authors on the same
   subject within 24 hours.
2. **Check authority participation.** If authority handles are in the cluster,
   the signal is stronger. Use `get-authorities` to verify.
3. **Assess PEAK6 fit.** Map the cluster to a content pillar. If it maps
   cleanly, it is likely actionable.
4. **Rate the urgency.** Breaking events (regulatory, market) are urgent
   (issue priority: `high`). Slow-building trends are standard
   (priority: `medium`).

## 5. Scoring opportunities

Rate each identified opportunity on three dimensions:

| Dimension | Weight | How to assess |
|-----------|--------|---------------|
| **Relevance** | 40% | How closely does this map to PEAK6's content pillars and domain expertise? |
| **Timeliness** | 35% | Is this conversation active right now? Is there a window closing? |
| **Engagement potential** | 25% | Are high-follower accounts involved? Is the thread getting replies? Could PEAK6's contribution get visibility? |

### Scoring thresholds

| Combined score | Action |
|----------------|--------|
| 80+ | **Create issue immediately.** High-confidence opportunity. Priority: `high`. |
| 60-79 | **Create issue.** Solid opportunity worth pursuing. Priority: `medium`. |
| 40-59 | **Note in digest only.** Mention it but don't create a dedicated issue. |
| Below 40 | **Skip.** Not actionable. |

These are judgment calls, not formulas. Use the scores as a framework, not
a calculator.

## 6. Issue creation

When you identify a content opportunity, create a Paperclip issue for a
content agent to pick up.

### Issue format

```
POST /api/companies/{companyId}/issues
```

```json
{
  "title": "Content Opportunity: {concise topic description}",
  "description": "see markdown template below",
  "priority": "medium",
  "status": "todo",
  "labels": ["content-opportunity", "{pillar}"],
  "goalId": "{company-goal-id}",
  "parentId": "{parent-project-id-if-applicable}"
}
```

### Issue description template

```markdown
## Content Opportunity

**Pillar:** {market structure | fintech | venture capital | chicago | culture}
**Opportunity score:** {0-100}
**Urgency:** {breaking — hours | trending — today | slow-build — this week}
**Suggested voice:** {hub | builder | either}
**Suggested format:** {standalone tweet | reply to [URL] | quote of [URL] | thread}

### What's happening

{2-3 sentences: what conversation or trend is occurring on X right now.
Include specific @handles and paraphrase their positions.}

### Why PEAK6 should participate

{1-2 sentences: what unique perspective, expertise, or insight PEAK6 brings
to this conversation. Be specific — "we know about markets" is not enough.}

### Corpus evidence

| Tweet | Author | Score | Age |
|-------|--------|-------|-----|
| "{snippet}" | @{handle} | {score} | {hours}h |
| "{snippet}" | @{handle} | {score} | {hours}h |
| ... | ... | ... | ... |

### Suggested angle

{1-2 sentences: a starting direction for the content agent. Not a draft —
just a seed. E.g., "Counter the claim that HFT harms retail by citing the
bid-ask spread compression data from 2024."}
```

### Assignment

Do **not** assign the issue to a specific agent unless your task instructions
say otherwise. Content agents use `wakeOnAssignment` — the human operator
or CEO agent handles assignment based on voice profile and workload.

If the opportunity clearly calls for hub voice, add the label `voice:hub`.
If builder voice, add `voice:builder`. This helps the operator route it.

## 7. Analysis cycle workflow

When triggered by a `corpus.updated` event or a scheduled heartbeat:

1. **Fetch the corpus.** Call `get-today` with no filters.
2. **Quick scan.** Read through all items. Flag anything that jumps out as a
   clear opportunity (direct PEAK6 mention, breaking event, authority
   question).
3. **Cluster analysis.** Group remaining items by theme. Look for 3+ item
   clusters.
4. **Score opportunities.** Rate each flagged item and cluster on relevance,
   timeliness, engagement potential.
5. **Check for new voices.** Call `suggest-handles`. If a promotion candidate
   is involved in an opportunity, note it — emerging voices sometimes signal
   emerging trends.
6. **Track new handles.** For any high-quality handle found during analysis
   that isn't tracked, call `track-handle`.
7. **Create issues.** For opportunities scoring 60+, create Paperclip issues
   using the template in section 5.
8. **Write summary.** Post a brief analysis summary as a comment on the
   parent discovery digest issue (if one exists), or as a standalone issue
   comment.

### Summary format

```markdown
## Analysis Summary — {YYYY-MM-DD}

**Corpus size:** {N} items | **Opportunities found:** {N}
**Issues created:** {list issue IDs}

### Trends
- {Trend 1}: {one-line description} — {N items, highest score: X}
- {Trend 2}: ...

### Skipped
- {Topic}: {why it was not actionable}

### New voices tracked
- @{handle} (relevance: {X}, domain: {domain})
```

## 8. Ad-hoc analysis

When asked to analyze a specific topic (not a scheduled cycle):

1. Call `search-corpus` with the relevant query.
2. If results are thin, try related queries (synonyms, adjacent topics).
3. Apply the same opportunity scoring framework.
4. Report findings directly — create an issue only if the analysis reveals
   an actionable opportunity.

## 9. Watchlist management

You are responsible for maintaining the authority handle ecosystem.

### Promoting handles

When you notice a handle consistently appearing in high-quality corpus results:

1. Call `suggest-handles` to check if they're already a candidate.
2. If they have 5+ appearances and 0.7+ avg relevance, promote them:
   call `promote-handle` with the handle, the best-matching domain
   (`market_structure`, `fintech`, `venture_capital`, `chicago_trading`), and
   a note explaining why.
3. If they don't yet meet the threshold, call `track-handle` to record the
   appearance. The pipeline will auto-promote when thresholds are met.

### Rating tweets

As you scan the corpus, rate tweets to improve future discovery:

- **relevant**: Directly useful for PEAK6's AI x Fintech positioning
- **irrelevant**: Noise that shouldn't have been in the corpus
- **skip**: Not clearly relevant or irrelevant

Aim to rate at least 10 tweets per cycle. This feedback loop tunes the
pipeline over time.

## 10. Cadence

| Trigger | Action |
|---------|--------|
| `corpus.updated` event | Full analysis cycle (steps 1-8 in section 7) |
| Heartbeat (3x/day) | Quick scan using `search-corpus` with `since: "8h"`. Focus on missed opportunities. |
| `tweet.high-engagement` event | Immediate investigation. Call `analyze-topic` + `get-thread`. Create issue if relevant. |

## 11. What you don't do

- **Don't write content.** Your job ends at the issue. Content agents draft.
- **Don't assign issues to agents.** The operator handles routing.
- **Don't evaluate draft quality.** That's the reviewer's job.
- **Don't run the discovery pipeline.** That's the plugin's scheduled job.
- **Don't engage on X.** No posting, replying, liking, or following.
