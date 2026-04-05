---
name: daily-intelligence
description: >
  Use when creating the daily X intelligence digest after the discovery
  pipeline has run. Requires the corpus to exist (pipeline must have
  completed). Covers fetching corpus, grouping by pillar, ranking, checking
  new voices, and writing the digest. Do NOT use for ad-hoc research
  (use x-research).
---

# Daily Intelligence

An orchestration skill for the daily intelligence cycle. This skill assumes
the discovery pipeline has already run and a corpus exists. It teaches how to
process that corpus into a structured digest.

**This is the one skill that depends on the pipeline.** All other skills work
without the corpus. If the pipeline hasn't run today, use `x-research` for
ad-hoc investigation instead.

## When to use

- After `corpus.updated` event (triggered by the discovery pipeline ~6 AM UTC)
- When creating the daily intelligence digest
- During a catch-up scan (heartbeat cycle) using `search-corpus` with
  `since` filter

## When NOT to use

- For ad-hoc research (use `x-research`)
- For real-time queries ("what's trending right now?") (use `x-research`)
- For content drafting (use `x-content-doctrine`)

---

## Daily intelligence cycle

### 1. Fetch the corpus

Call `get-today` to get today's scored corpus.

- If the corpus has 20+ items: proceed normally
- If the corpus has 10-20 items: note thin volume, still proceed
- If the corpus has < 10 items: likely a weekend/holiday. Use
  `search-corpus` without a date filter for a 3-day window to supplement.

### 2. Group by content pillar

Organize results into the five content pillars:

- Market structure & regulation
- Fintech & trading technology
- Venture capital & investment
- Chicago business ecosystem
- Company culture & talent

Note any pillar with zero results — this is useful signal (either nothing
is happening or the pipeline's queries need adjustment).

### 3. Rank within pillars

Within each pillar, rank by:

1. **Score** (primary) — the plugin's composite relevance score
2. **Authority participation** — tweets from authority handles rank higher
3. **Engagement** — higher engagement = more social proof
4. **Freshness** — prefer newer content within the same score band

Surface the top 3-5 items per pillar.

### 4. Identify engagement opportunities

Flag conversations where PEAK6 could meaningfully participate:

- Active threads with authority handles (< 12h old)
- Questions from high-follower accounts in PEAK6 domains
- Debates where PEAK6 has a differentiated perspective

These become candidates for content issues.

### 5. Check for new voices

Call `suggest-handles` to see handles approaching promotion criteria.

For each suggestion:
- If 5+ appearances and 0.7+ avg relevance → call `promote-handle`
- If promising but not yet at threshold → note for future cycles

Also scan for untracked handles producing quality content in the corpus.
Call `track-handle` for any new voice worth monitoring. Note: manually
tracked handles start at `appearances: 0` and won't auto-promote until
the pipeline encounters their tweets.

### 6. Rate tweets

Call `rate-tweet` on 10+ tweets from the corpus:
- `relevant` — on-topic, good signal
- `irrelevant` — noise, off-topic

This feedback improves future discovery scoring.

### 7. Write the digest

Post as a Paperclip issue or issue comment:

```markdown
## Daily X Intelligence — YYYY-MM-DD

**Corpus size:** N items | **Pipeline run:** HH:MM UTC

### Top Stories
1. **@author** (score: 85): "Tweet snippet..." — Why it matters.
   [View post](https://x.com/author/status/ID)
2. ...

### Market Structure & Regulation
- **@author** (score: 78): "Key quote..." [View post](...)

### Fintech & Trading Technology
- ...

### Venture Capital & Investment
- ...

### Chicago Business Ecosystem
- ...

### Company Culture & Talent
- ...

### Engagement Opportunities
- Thread by @author on [topic] — PEAK6 could add value by [reason].
  [View thread](...)

### New Voices
- @handle (relevance: 0.82, domain: fintech) — Tracked/promoted.
  [Profile](https://x.com/handle)

### Pillar gaps
- {Pillar}: No corpus items today
```

Keep concise — the digest should be a 2-minute skim.

---

## Catch-up scan (heartbeat cycle)

On heartbeat triggers (3x/day), run a lighter version:

1. Call `search-corpus` with `since: "8h"` to get new items since last scan
2. Quick-scan for obvious opportunities (80+ score, authority threads)
3. Create issues only for high-priority findings
4. Skip the full digest — that's for the primary cycle

---

## Weekend/holiday handling

The pipeline runs daily regardless, but weekend corpora are thinner:

- **Saturday/Sunday**: Expect 30-50% normal volume. Lower your opportunity
  thresholds (60+ instead of 80+ for issue creation).
- **Market holidays**: Similar to weekends for market-related pillars.
  Fintech/tech pillars may still be active.
- **If corpus is empty**: Don't create a "nothing found" digest. Use
  `search-corpus` with a 3-day window and produce a brief summary.
