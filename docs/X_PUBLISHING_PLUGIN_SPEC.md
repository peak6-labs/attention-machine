# x-publishing-plugin Specification

> Architecture spec for Phase B — Content Pipeline
> Status: Draft
> Date: 2026-03-31

## Overview

The x-publishing-plugin handles the write side of X operations: drafting, reviewing, scheduling, and publishing content. It is the companion to x-intelligence-plugin (read side) — the two plugins communicate through Paperclip's event system.

**Key principle**: Publishing requires human approval. Every post goes through `in_review` -> `done` before it can be published. This is a compliance requirement, not a technical limitation.

---

## Plugin Identity

```
id: "peak6-labs.x-publishing"
version: "0.1.0"
displayName: "X Publishing"
categories: ["connector", "automation"]
```

## Credentials

Unlike x-intelligence-plugin (which uses App Bearer Token for read-only access), x-publishing requires **OAuth 2.0 User Context** per account:

| Secret Ref | Purpose |
|------------|---------|
| `x_oauth_hub_access_token` | @PEAK6 hub account OAuth 2.0 access token |
| `x_oauth_hub_refresh_token` | @PEAK6 hub account refresh token |
| `x_oauth_spoke_{name}_access_token` | Spoke account access tokens (one per team member) |
| `x_oauth_spoke_{name}_refresh_token` | Spoke account refresh tokens |
| `x_oauth_client_id` | OAuth 2.0 client ID (shared across accounts) |
| `x_oauth_client_secret` | OAuth 2.0 client secret |

**Token refresh job**: OAuth 2.0 tokens expire every 2 hours. A `token-refresh` job runs every 90 minutes to refresh all account tokens.

---

## Config Schema

```typescript
interface PublishingConfig {
  company_id: string;
  accounts: {
    [key: string]: {
      display_name: string;       // "PEAK6 Hub" or team member name
      x_handle: string;           // "PEAK6" or team handle
      account_type: "hub" | "spoke";
      access_token_ref: string;   // Secret ref
      refresh_token_ref: string;  // Secret ref
      daily_post_limit: number;   // Default: 4
      enabled: boolean;
    };
  };
  oauth_client_id_ref: string;
  oauth_client_secret_ref: string;
  require_approval: boolean;       // Default: true. NEVER set to false in production.
  schedule_check_interval_minutes: number;  // Default: 15
}
```

---

## Entity Types

### `draft` — Draft tweet entity

```typescript
interface DraftData {
  text: string;                    // Tweet text (max 280 chars, or 25000 for long-form)
  account_key: string;             // Config key of the target account
  format: "short" | "standard" | "thread";
  thread_tweets?: string[];        // For thread format: each tweet as a separate string
  media_ids?: string[];            // Pre-uploaded media IDs
  in_reply_to_tweet_id?: string;   // For reply drafts
  quote_tweet_id?: string;         // For quote-tweet drafts
  scheduled_for?: string;          // ISO 8601 timestamp for scheduled posting
  content_bucket?: string;         // From content doctrine
  issue_id?: string;               // Paperclip issue this draft is attached to
  created_by: string;              // Agent ID or "manual"
  created_at: string;
  updated_at: string;
}
```

Status flow: `draft` -> `in_review` -> `approved` -> `published` (or `rejected`)

### `published-post` — Published tweet entity

```typescript
interface PublishedPostData {
  tweet_id: string;                // X API tweet ID
  text: string;
  account_key: string;
  published_at: string;
  draft_id?: string;               // Link back to the draft entity
  issue_id?: string;               // Link back to the issue
  metrics_snapshot?: {             // Captured periodically
    likes: number;
    reposts: number;
    replies: number;
    quotes: number;
    impressions: number;
    captured_at: string;
  };
}
```

---

## Tools

### Core Write Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `draft-post` | Create a draft tweet entity | `text`, `account_key`, `format`, `thread_tweets?`, `in_reply_to_tweet_id?`, `quote_tweet_id?`, `content_bucket?` |
| `update-draft` | Modify an existing draft | `draft_id`, `text?`, `thread_tweets?`, `scheduled_for?` |
| `submit-for-review` | Move draft to `in_review` and create/update a Paperclip issue | `draft_id`, `reviewer_notes?` |
| `publish-post` | Post to X via API. **Gated**: draft must be `approved` status. | `draft_id` |
| `schedule-post` | Set a publish time for an approved draft | `draft_id`, `publish_at` |

### Read Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `get-drafts` | List drafts by status | `status?`, `account_key?`, `limit?` |
| `get-schedule` | View upcoming scheduled posts | `account_key?`, `days_ahead?` |
| `get-post-metrics` | Get engagement metrics for a published post | `tweet_id` |

### Engagement Tools (Phase D)

| Tool | Description | Parameters |
|------|-------------|------------|
| `reply-to-tweet` | Draft and submit a reply | `tweet_id`, `text`, `account_key` |
| `quote-tweet` | Draft and submit a quote-tweet | `tweet_id`, `text`, `account_key` |
| `repost` | Repost a tweet (no approval needed for reposts) | `tweet_id`, `account_key` |

---

## Jobs

| Job Key | Schedule | Description |
|---------|----------|-------------|
| `publish-scheduled` | Every 15 minutes | Publish approved posts that have reached their scheduled time |
| `token-refresh` | Every 90 minutes | Refresh OAuth 2.0 access tokens for all configured accounts |
| `metrics-capture` | Every 6 hours | Snapshot engagement metrics for published posts (last 7 days) |
| `rate-limit-reset` | Daily at midnight | Reset daily post counters per account |

---

## Events

### Emitted

| Event | When | Payload |
|-------|------|---------|
| `post.published` | After successful X API post | `{ tweet_id, account_key, text, draft_id }` |
| `post.scheduled` | After a post is scheduled | `{ draft_id, account_key, publish_at }` |
| `draft.created` | After a new draft is created | `{ draft_id, account_key, format, created_by }` |
| `post.engagement-milestone` | When a post crosses engagement thresholds | `{ tweet_id, milestone, metrics }` |

### Subscribed

| Event | Source | Action |
|-------|--------|--------|
| `corpus.updated` | x-intelligence-plugin | Optional: auto-create "content opportunity" issues for agents |
| `tweet.high-engagement` | x-intelligence-plugin | Alert: trending content that may need a response |

---

## UI Slots

### Dashboard Widget
- Upcoming scheduled posts (next 24 hours)
- Recently published posts with engagement snapshot
- Draft queue count by status

### Page: Content Calendar (`/x-publishing`)
- Calendar view of scheduled and published posts
- Draft status pipeline (draft -> in_review -> approved -> published)
- Per-account view with daily post counts

### Settings Page
- Account configuration and OAuth status
- Rate limit usage per account
- Token health (last refresh, expiry time)

---

## Approval Flow

```
Agent calls draft-post
  -> draft entity created (status: "draft")

Agent calls submit-for-review
  -> draft status: "in_review"
  -> Paperclip issue created/updated with draft content
  -> Human notified via board UI

Human reviews on board
  -> Approves: issue status -> "done", draft status -> "approved"
  -> Rejects: issue comment with feedback, draft status -> "draft" (back to agent)

Agent (or scheduler) calls publish-post
  -> Verifies draft status == "approved"
  -> Calls X API POST /2/tweets
  -> Creates published-post entity
  -> Draft status: "published"
  -> Emits post.published event
```

**The `require_approval` config defaults to `true`. Even if set to `false` (for testing), publish-post STILL requires the draft to be in `approved` status. The only difference is whether the issue review step is mandatory.**

---

## X API Endpoints Used

| Endpoint | Method | Purpose | Rate Limit |
|----------|--------|---------|------------|
| `POST /2/tweets` | POST | Publish a tweet | 200/15min per user |
| `DELETE /2/tweets/:id` | DELETE | Delete a published tweet | 50/15min per user |
| `POST /2/oauth2/token` | POST | Refresh OAuth 2.0 token | N/A |
| `GET /2/tweets/:id` | GET | Fetch post metrics | 3500/15min per app |
| `GET /2/users/:id/tweets` | GET | User timeline for scheduling context | 1500/15min per user |

---

## Cross-Plugin Data Flow

```
x-intelligence-plugin                    x-publishing-plugin
---------------------                    --------------------
discovery-run job
  -> corpus.updated event --------->     (optional) agent creates content issues
                                         agent picks up issue
                                         agent calls search-corpus (x-intel tool)
                                         agent drafts post (x-pub tool)
                                         human reviews
                                         agent publishes
  <- post.published event <---------     post.published emitted
  (x-intel can track the post's
   performance in the corpus)
```

---

## Implementation Notes

1. **Start with hub account only.** Add spoke accounts after the approval flow is validated.
2. **OAuth token management is the hardest part.** Consider storing tokens in Paperclip secrets with auto-refresh, or building a lightweight token proxy.
3. **Rate limit tracking must be per-account**, not per-app. X API v2 user-context rate limits are separate from app-context limits.
4. **The `publish-post` tool should be defensive**: check rate limits, verify token validity, and validate tweet text length BEFORE calling the X API. A failed publish is worse than a delayed publish.
5. **Thread publishing needs atomic handling**: if tweet 3/5 fails, the partial thread is already live. Consider a "thread draft" entity that tracks per-tweet publish status.
