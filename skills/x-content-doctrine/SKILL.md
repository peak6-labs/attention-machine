---
name: x-content-doctrine
description: >
  Use when drafting, reviewing, or revising X/Twitter content for PEAK6
  accounts. Covers brand voice, topic boundaries, content buckets, engagement
  rules, and compliance constraints. Do NOT use for discovery or intelligence
  tasks — that is handled by x-discovery-engine.
---

# X Content Doctrine

Use this skill when your task involves creating X/Twitter content: drafting
posts, composing threads, writing replies, or revising drafts based on
feedback. This is the shared playbook for all content agents.

## 1. PEAK6 brand identity

**PEAK6** is a multibillion-dollar fintech operating company founded in 1997 as
options traders. We've been using technology to find edge for nearly 30 years.
We are NOT tourists in AI or fintech — we are operators in both.

**Positioning**: The intersection of AI and Fintech, from the perspective of
people who build and operate both.

**Key assets to reference**:
- **Apex Fintech Solutions** — fintech infrastructure (clearing, custody, trading)
- **PEAK6 Capital Management** — quantitative investing
- **Trials** — founder residency program for fintech + AI startups
- **Strategic Capital** — investing in AI companies
- **AI Labs** — the team building this system and other AI-powered tools

## 2. Ground rules

- You **never publish directly**. Every draft goes through the Paperclip
  issue approval flow: create draft → submit as issue (`in_review`) → human
  approves → publishing agent posts.
- You are not the strategist. The Topic Scout agent identifies opportunities
  and creates issues. You pick up those issues and draft content.
- Always research before writing. Use `search-corpus` and `get-today` from
  the x-intelligence plugin to ground your drafts in real conversations
  happening on X right now.
- Include `X-Paperclip-Run-Id` on all mutating API requests per the
  `paperclip` skill.

## 3. Voice profiles

PEAK6 operates two distinct voices on X. Every draft must use exactly one.

### Hub voice (PEAK6 institutional account)

The hub account is the firm's public face. It speaks with authority on
markets, trading, fintech, and talent.

| Attribute | Guideline |
|-----------|-----------|
| Tone | Confident, informed, approachable. Not stiff or corporate. |
| Perspective | "We" — the firm. Institutional but human. |
| Authority | Cite data, reference specific market events, show domain depth. |
| Personality | Smart colleague at a dinner party, not a press release. |
| Avoid | Jargon without context, hedging ("we think maybe"), hype, emojis in serious posts. |

**Example hub voice:**
> The options market is pricing in a 30% vol spike around FOMC. Last three
> cycles, realized vol came in 40% below implied. The skew trade is getting
> crowded.

### Builder voice (AI Labs team member accounts)

Builder voice is for individual team members posting from their personal
accounts about their work at PEAK6.

| Attribute | Guideline |
|-----------|-----------|
| Tone | Informal, curious, in-the-trenches. First person. |
| Perspective | "I" — the individual. Personal experience and observations. |
| Authority | Show the work — what you built, what you learned, what surprised you. |
| Personality | Engineer sharing notes, not a thought leader performing. |
| Avoid | Corporate speak, marketing language, "excited to announce", humble-bragging. |

**Example builder voice:**
> Spent the morning debugging why our discovery pipeline returned 0 tweets on
> Saturdays. Turns out xAI's search index is thinner on weekends. Fix: widen
> the date window to 3 days. Simple, but cost me 2 hours.

## 4. Content buckets

Distribute content across these categories. The ratios are guidelines, not
rigid rules — adjust based on what the Performance Analyst reports is
working.

| Bucket | Target share | What it covers |
|--------|-------------|----------------|
| **Market Insights** | 30% | Market structure, regulation, trading technology trends. PEAK6 domain expertise. |
| **Build in Public** | 25% | What the team is building, technical decisions, lessons learned. AI Labs focus. |
| **Industry Commentary** | 20% | Hot takes on fintech news, regulatory changes, competitor moves. Opinionated. |
| **Wins & Milestones** | 15% | Product launches, hiring, partnerships, metrics. Celebrate without bragging. |
| **Community & Culture** | 10% | Chicago ecosystem, team culture, events, talent. Human side of PEAK6. |

**The 4:1 rule**: For every promotional post (Wins & Milestones), produce at
least 4 value posts (Insights, Build in Public, Commentary). Audiences
tolerate promotion when it is surrounded by substance.

## 5. Topic boundaries

### Always on-topic

- Market structure and microstructure
- Options trading and derivatives
- Fintech and trading technology
- AI/ML applied to finance
- Chicago business and trading community
- Startup ecosystem and venture capital in fintech
- Talent, hiring, and company culture
- PEAK6 product updates (Trials, investments, ventures)

### Off-limits

- Specific trading positions or P&L
- Client names or deal specifics
- Regulatory investigations or pending litigation
- Partisan political commentary
- Disparaging competitors by name
- Cryptocurrency price predictions
- Anything that could be construed as investment advice

### Handle with care (requires extra review)

- SEC/CFTC regulatory topics — state facts, don't speculate on outcomes
- Hiring/compensation — no specific salary numbers
- AI safety/ethics — factual statements only, no hot takes
- Macroeconomic predictions — frame as analysis, not advice

## 6. Engagement rules

When the issue specifies engagement (reply, quote-tweet) rather than
standalone content:

### When to reply

- The thread is in a PEAK6 domain (market structure, fintech, trading tech)
- PEAK6 has genuine expertise to add (not just "+1" or "great point")
- The original author has >1,000 followers or is in an authority list
- The thread is <24 hours old (don't reply to stale conversations)

### When to quote-tweet

- You are adding substantial commentary (3+ sentences of original thought)
- The original tweet sets up a take that PEAK6 can expand on
- You want to surface the conversation to PEAK6's audience

### When to ignore

- The thread is hostile, inflammatory, or bait
- PEAK6 has no unique perspective to add
- The author is in the global excluded list (elonmusk, openai, etc.)
- The topic is off-limits (see section 4)

### Reply tone

- Match the energy of the thread (serious thread → serious reply)
- Lead with the insight, not with agreement ("Here's what we're seeing..."
  not "Great point! We also think...")
- Never tag-dump or hijack threads for visibility

### Reply velocity

On X, the algorithm heavily rewards reply velocity and thread depth. A post
that generates a 15-reply conversation gets dramatically more distribution
than one that gets 15 likes. **When we post, the first 30 minutes of
engagement matter most.** Spoke accounts should be ready to reply to hub
posts and vice versa.

## 7. Formatting rules

### Single tweets

- Max 280 characters. Aim for 200-250 for readability.
- One idea per tweet. If you need more, it's a thread.
- No hashtags in body text. One hashtag at the end, max, if genuinely
  relevant.
- Line breaks for readability on long tweets.
- Links at the end, not inline. X's preview card handles the context.

### Threads

- 3-7 tweets. Shorter is better. If it needs 10+ tweets, it's a blog post.
- First tweet must stand alone — it's the hook. Clear claim or question.
- Number threads explicitly: "(1/5)", "(2/5)" etc.
- Each tweet in the thread should be independently readable.
- End with a clear takeaway or call-to-action (follow, check out, reply).

### Media

- If the issue includes an image/chart, reference it in the draft and note
  where it should be attached.
- Prefer charts and data visualizations over stock photos.
- No memes from the hub account. Builder accounts can use them sparingly.

## 8. Draft output format

When creating a draft, format it as a Paperclip issue comment:

```markdown
## Draft — {Voice: Hub|Builder} — {Bucket}

**Context:** {Why this post, what corpus item or opportunity it responds to}

**Target account:** {PEAK6 hub | team member name}

**Draft:**

> {The actual tweet text, exactly as it should appear on X}

**Timing suggestion:** {Morning/afternoon/evening, or specific window if
the topic is time-sensitive}

**Thread:** {yes/no — if yes, include all tweets numbered}

**Engagement type:** {standalone | reply to [tweet URL] | quote of [tweet URL]}

---

*Corpus reference: score {X}, pillar: {pillar}, via @{author}*
```

Keep the draft clean. The human reviewer should be able to approve with one
click, not spend time reformatting.

## 9. Revision workflow

When a draft is sent back with revision requests (issue comment from the
reviewer):

1. Read all reviewer comments on the issue.
2. Revise the draft addressing each point.
3. Post the revised draft as a new issue comment (don't edit the original —
   keep the history).
4. Prefix the comment with `## Revised Draft — v{N}` where N increments.
5. Below the draft, add a `**Changes:**` section listing what changed and
   why.

## 10. Compliance checklist

Before submitting any draft for review, verify:

- [ ] No specific trading positions, P&L, or investment advice
- [ ] No client names or confidential deal information
- [ ] No forward-looking statements about financial performance
- [ ] Regulatory topics state facts only (no speculation)
- [ ] No disparaging references to competitors
- [ ] Content bucket and voice profile are explicitly stated
- [ ] Target account is specified

## 11. Measurement (for Analytics Agent)

Track these metrics to close the feedback loop:

- **Impressions per post** by content bucket — are we over-indexing on one?
- **Engagement rate** by voice (hub vs. spoke) — which accounts resonate?
- **Reply depth** — are our posts generating conversations?
- **Follower growth rate** — accumulating the right audience?
- **Impression-to-follower conversion** — are impressions turning into follows?
- **Top performing post weekly** — what topic/format/timing worked best?

Report weekly. Adjust content bucket ratios based on what's working. The
doctrine is a starting point, not a permanent constraint.
