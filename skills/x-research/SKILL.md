---
name: x-research
description: >
  Use when you need to understand what's happening on X about a topic, find
  relevant conversations, or gather evidence for content decisions. Teaches
  a research methodology — not tool API docs. Works with or without the
  daily corpus.
---

# X Research

A methodology for researching any topic on X/Twitter. Follow this process
whether you're investigating a user question, preparing content, or scanning
for trends. The same methodology works with real-time search or the pre-built
corpus — use whichever data source is available.

## When to use

- Answering "what's happening on X about [topic]?"
- Gathering evidence before drafting content
- Investigating a trending conversation or thread
- Finding who's talking about a subject and what they're saying

## When NOT to use

- Creating content (use `x-content-doctrine`)
- Evaluating opportunities for firm-level issues (use `opportunity-detection`)
- Running the daily intelligence digest (use `daily-intelligence`)

---

## Research Methodology

### Step 1: Decompose the question

Break the research topic into 3-5 distinct search queries. Each query should
approach the topic from a different angle.

Example — "What's the conversation around tokenized securities?"
- `tokenized securities regulation`
- `RWA tokenization adoption`
- `security token infrastructure`
- `tokenized assets institutional`

**Why multiple queries**: A single query returns one slice. Decomposition finds
conversations you wouldn't see otherwise — different communities use different
language for the same topic.

### Step 2: Execute searches

Choose your data source based on what's available and what you need:

| Source | When to use | Tool |
|--------|-------------|------|
| **Real-time search** | Default. Any time, any topic. | `search-x` |
| **Trending topics** | "What's hot right now?" in configured domains | `get-trending` |
| **User timeline** | Investigating a specific person's recent posts | `get-user-timeline` |
| **Corpus search** | When the daily pipeline has run and you want pre-scored, deduplicated data | `search-corpus` |
| **Today's corpus** | Quick overview of today's curated intelligence | `get-today` |

**Cost awareness**: Use `quick: true` on `search-x` for exploratory queries
(single query, no enrichment). Save full enrichment for focused follow-ups.

**Authority context**: Call `get-authorities` to know which handles are
recognized domain experts. Authority tweets carry more signal weight.

### Step 3: Assess signal quality

Not all results are equal. Filter by these signals:

**Engagement thresholds**:
- **< 10 engagements** — Noise floor. Skip unless from an authority handle.
- **10-49** — Signal. Worth reading.
- **50-99** — Expert tier. Investigate further.
- **100+** — Escalation tier. Always investigate.

**Freshness**: Conversations older than 24 hours are cooling. Older than 48
hours are cold unless still actively receiving replies.

**Authority participation**: A thread with 2+ authority handles engaging is
significantly higher signal than raw engagement numbers suggest.

**Cluster signal**: 3+ tweets from different authors on the same theme within
24 hours indicates a real conversation, not just one person's take.

### Step 4: Follow promising threads

For high-signal results:

1. **Get full context** — Use `get-thread` to see the complete conversation,
   not just the tweet that matched your search.
2. **Go deep on key topics** — Use `analyze-topic` for a multi-perspective
   breakdown when a topic warrants deeper analysis (50+ engagement, authority
   involvement, or cluster detected).
3. **Check the author** — Is this person in the authority lists? Are they a
   new voice worth noting?

### Step 5: Synthesize by theme

Group your findings by **theme**, not by query. The user doesn't care which
search string found the result — they care about the story.

Good synthesis:
- "Three distinct conversations are happening: regulatory clarity (SEC
  guidance), institutional adoption (BlackRock, Fidelity), and infrastructure
  challenges (settlement finality)."

Bad synthesis:
- "Query 1 returned 12 results. Query 2 returned 8 results. Query 3..."

### Step 6: Surface with attribution

Every referenced post MUST include a direct link. No exceptions.

**Format for each referenced tweet**:
```
@author (score: X, engagements: Y): "Key quote or paraphrase"
[View post](https://x.com/author/status/ID)
```

Group by theme. Lead with the most important finding. Include 2-3 supporting
posts per theme. Note any authority handles involved.

---

## Research Quality Checklist

Before presenting results, verify:

- [ ] Multiple queries executed (not just one search)
- [ ] Results filtered by signal quality (not just dumping everything)
- [ ] Threads followed for high-signal items (not just surface-level tweets)
- [ ] Findings grouped by theme (not by query)
- [ ] Every tweet includes a direct link
- [ ] Authority handles identified where relevant
- [ ] Freshness noted (when were these conversations active?)
- [ ] Gaps acknowledged ("No significant conversation found on [subtopic]")

---

## Handling thin results

If searches return few or no results:

1. **Broaden queries** — Remove specific terms, try related concepts
2. **Check authorities** — Search for what domain experts are talking about,
   even if it's adjacent to the original topic
3. **Try corpus if available** — Use `search-corpus` without date filter for
   a wider time window
4. **Report honestly** — "X has limited conversation on this topic right now"
   is a valid finding. Don't stretch thin results into false confidence.

---

## Weekend/holiday awareness

xAI search returns fewer results on weekends and holidays. If results seem
unusually thin, this is likely the cause — not a lack of conversation. The
underlying tools use a 3-day minimum search window to mitigate this, but
volume will still be lower.

---

## Content pillars (PEAK6 domains)

When researching for PEAK6 purposes, these are the primary domains:

- **Market structure & regulation** — SEC, microstructure, options, CFTC
- **Fintech & trading technology** — fintech, payments, AI in finance
- **Venture capital & investment** — VC deals, startups, infrastructure
- **Chicago business ecosystem** — Chicago community, prop firms
- **Company culture & talent** — talent, culture, team building

Use pillar context to assess relevance. A high-engagement tweet about
cryptocurrency prices is noise; a moderate-engagement thread about options
market microstructure is signal.
