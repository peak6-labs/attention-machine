---
name: opportunity-detection
description: >
  Use when evaluating X content for actionable content opportunities for
  PEAK6. Covers opportunity identification, cluster detection, scoring, and
  issue creation for handoff to content agents. Do NOT use for research
  methodology (use x-research) or content drafting (use x-content-doctrine).
---

# Opportunity Detection

A framework for identifying content opportunities from X/Twitter data and
creating well-structured issues for content agents. This is a firm-level
operational skill — it produces Paperclip issues in the AIS project, not
conversational output.

## When to use

- After researching a topic (via `x-research`) and evaluating whether PEAK6
  should create content about it
- During daily intelligence analysis
- When a high-engagement alert arrives and you need to assess actionability
- When scanning corpus or real-time data for content angles

## When NOT to use

- For personal user requests (personal agents use `user-interaction` instead)
- For the research itself (use `x-research` first, then apply this skill)

---

## What makes a strong opportunity

**Create an issue** when you find:

- High-authority thread on a PEAK6 domain topic — active (< 12h old,
  multiple replies)
- Cluster: 3+ tweets from different authors on the same theme within 24h
- Regulatory or market breaking event where PEAK6 has genuine expertise
- Direct PEAK6 mention or relevant conversation about PEAK6 products
- Authority asking a question PEAK6 can credibly answer

**Skip** when you find:

- Generic fintech commentary without a specific angle
- Tweets from excluded handles
- Stale conversations (> 24h, no new replies)
- Topics outside PEAK6's content pillars
- Low-engagement tweets from non-authority accounts

---

## Cluster detection

Look for convergence — multiple independent voices on the same subject:

1. Group results by theme (not by search query)
2. Count unique authors per theme
3. **3+ tweets from different authors = cluster** — this is a real
   conversation, not one person's take
4. Check for authority participation within the cluster
5. Assess which PEAK6 content pillar the cluster maps to

Clusters are the strongest opportunity signal. A single viral tweet may be
noise; three independent experts discussing the same shift is signal.

---

## Scoring framework

Score each opportunity on three dimensions:

| Dimension | Weight | What to assess |
|-----------|--------|----------------|
| **Relevance** | 40% | Does this map to a PEAK6 content pillar? Does PEAK6 have genuine expertise? |
| **Timeliness** | 35% | Is the conversation active? Is there a closing window? |
| **Engagement potential** | 25% | Are high-follower accounts involved? Is there reply activity? Visibility? |

**Thresholds**:
- **80+** — Create issue immediately, priority: `high`
- **60-79** — Create issue, priority: `medium`
- **40-59** — Note in digest only, do not create issue
- **Below 40** — Skip

---

## Handle management during analysis

As you review content, manage the handle pipeline:

- **Check suggestions**: Call `suggest-handles` — handles with 5+ appearances
  and 0.7+ avg relevance are promotion candidates
- **Promote strong handles**: Call `promote-handle` with domain and
  explanation for handles meeting promotion criteria
- **Track new voices**: Call `track-handle` for untracked handles producing
  high-quality, relevant content
- **Rate tweets**: Call `rate-tweet` on 10+ tweets per analysis cycle
  (relevant/irrelevant) to improve future discovery

Note: Manually tracked handles start at `appearances: 0` and won't
auto-promote until the pipeline encounters their tweets. Use
`promote-handle` to bypass this for clearly authoritative handles.

---

## Issue creation

Create issues in the **AIS project** with this structure:

**Title**: `[Opportunity Type] Brief description`

Opportunity types: `Breaking News`, `Trending Thread`, `Expert Take`,
`Content Seed`, `Authority Discovery`

**Status**: `backlog` (normal priority) or `todo` (high/urgent priority)

**Labels**: Include voice recommendation — `voice:hub` or `voice:builder`

**Body template**:
```markdown
**Pillar:** {content pillar}
**Opportunity score:** {score}/100
**Urgency:** {high|medium|low} — {why}
**Suggested voice:** {hub|builder} — {why}
**Suggested format:** {single tweet|thread|reply|quote-tweet}

## What's happening

{2-3 sentences: @handles involved, paraphrased positions, direct links}

## Why PEAK6 should participate

{1-2 sentences: specific expertise PEAK6 brings, not generic "thought
leadership"}

## Evidence

| Tweet | Author | Score | Age | Link |
|-------|--------|-------|-----|------|
| "Key quote..." | @author | 82 | 3h | [View](https://x.com/author/status/ID) |

## Suggested angle

{1-2 sentences: seed idea for content agent, not a full draft}
```

**Assignment**: Do NOT assign to a specific agent unless instructed. Use
voice labels (`voice:hub`, `voice:builder`) to route — content agents pick
up issues matching their voice assignment.

---

## Analysis cycle output

After completing an analysis cycle, produce a brief summary:

```markdown
## Analysis Summary — YYYY-MM-DD

**Source:** {corpus / real-time / both}
**Items reviewed:** N | **Opportunities found:** N
**Issues created:** [list IDs with links]

### Key trends
- {Trend}: {description} — {N items, highest score}

### Skipped topics
- {Topic}: {why not actionable}

### New voices
- @{handle} (relevance: X, domain: Y) — tracked/promoted
```
