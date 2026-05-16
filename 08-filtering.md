# Filtering

> **TL;DR**: Filtering runs at two stages: pre-scoring (cheap, large-scale reduction) and post-selection (expensive or freshness-sensitive checks on the top-K). There are 15 pre-scoring filters and 2 post-selection filters. Filters run sequentially in list order — sequence matters for correctness and efficiency.

---

## Filter Stages

```
Candidates (post-hydration)
     │
     ▼
PRE-SCORING FILTERS (15 filters, sequential)
     │
     ▼
Scorers (PhoenixScorer → RankingScorer)
     │
     ▼
TopKScoreSelector
     │
     ▼
POST-SELECTION FILTERS (2 filters, sequential)
     │
     ▼
Final candidates served
```

---

## Pre-Scoring Filters

### 1. `DropDuplicatesFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/drop_duplicates_filter.rs` |
| Dedup key | `tweet_id` |
| Logic | Keep first occurrence; subsequent occurrences dropped |
| Notes | Pure ID-based dedup; runs first to eliminate duplicates cheaply before any other filter |

---

### 2. `CoreDataHydrationFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/core_data_hydration_filter.rs` |
| Logic | Drops candidates where `author_id == 0` |
| Notes | `author_id = 0` is the sentinel for failed core data hydration. Must run after core data hydrator. |

---

### 3. `AgeFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/age_filter.rs` |
| Field | `max_age: Duration` |
| Logic | Keeps candidates whose age (derived from Snowflake tweet_id) ≤ `max_age`. Drops candidates where age cannot be determined (`unwrap_or(false)` → drop). |
| Notes | Age is computed from Snowflake ID timestamp bits — no separate timestamp field needed. Unknown-age posts are dropped conservatively. |

---

### 4. `SelfTweetFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/self_tweet_filter.rs` |
| Logic | Drops candidates where `author_id == viewer_user_id` |
| Notes | You don't see your own posts in the For You feed |

---

### 5. `RetweetDeduplicationFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/retweet_deduplication_filter.rs` |
| Dedup key | `retweeted_tweet_id` if present; else `tweet_id` |
| Logic | Keeps first occurrence per dedup key |
| Notes | Ensures original tweet + any retweet of it are not both shown. If both the original and a retweet are candidates, only one survives. |

---

### 6. `IneligibleSubscriptionFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/ineligible_subscription_filter.rs` |
| Logic | If `subscription_author_id` is set, keeps only if viewer is subscribed to that author. Posts with `subscription_author_id == None` always pass. |
| Notes | Paywalled posts are only shown to subscribers. Non-subscription posts are never affected. |

---

### 7. `PreviouslySeenPostsFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/previously_seen_posts_filter.rs` |
| Logic | Removes candidates where any related post ID matches `query.seen_ids` (exact) OR `bloom_filter_entries` (probabilistic) |
| Related post IDs | `tweet_id`, `retweeted_tweet_id`, `quoted_tweet_id`, and potentially others via `get_related_post_ids()` |
| Notes | Bloom filter may produce false positives — a post could be incorrectly filtered if it hashes to a seen ID. Backup filter (`PreviouslySeenPostsBackupFilter`) covers when bloom filter is unavailable. |

---

### 8. `PreviouslySeenPostsBackupFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/previously_seen_posts_backup_filter.rs` |
| Enable | Only when `impressed_post_ids` is non-empty |
| Logic | Removes candidates where any related post ID is in `impressed_post_ids`. Passes all through if list is empty. |
| Notes | Backup path when bloom filter is unavailable. Uses exact matching, not probabilistic. |

---

### 9. `PreviouslyServedPostsFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/previously_served_posts_filter.rs` |
| Enable | When `EnableServedFilterAllRequests` param is true OR (`is_bottom_request` AND context != `ForegroundTruncate`) |
| Logic | Removes candidates where any related post ID is in `query.served_ids` |
| Notes | Served IDs are tracked server-side across requests in the same session. Different from seen IDs (which are client-reported). Disabled for some top-of-feed requests. |

---

### 10. `MutedKeywordFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/muted_keyword_filter.rs` |
| Field | `tokenizer: Arc<TweetTokenizer>` |
| Logic | Tokenizes muted keyword list into `UserMutes`, builds `MatchTweetGroup`, removes candidates whose `tweet_text` matches. |
| Enable | Short-circuits (passes all) when muted keyword list is empty |
| Notes | Runs `tokio::task::block_in_place` (blocking I/O offload). Matching is token-based, not substring — "football" would not match "footballer" unless tokenizer produces the same token. |

---

### 11. `AuthorSocialgraphFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/author_socialgraph_filter.rs` |
| Logic | Removes a candidate if ANY of: viewer muted author, viewer blocked author, author blocked viewer (`author_blocks_viewer`), quoted author blocked viewer (`quoted_author_blocks_viewer`), viewer blocked quoted author, viewer blocked retweeted user |
| Notes | Muting only applies to the primary author, not quoted/retweeted authors. Blocking is checked bidirectionally for both primary author and quoted author (both directions: viewer→author and author→viewer). |

---

### 12. `TopicIdsFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/topic_ids_filter.rs` |
| Enable | Only relevant when `is_topic_request()` or `has_excluded_topics()` |

**Phase 1 — Topic Inclusion** (runs if `is_topic_request()`):

- **Non-bulk** (≤6 topics): for each query topic, keeps candidates that match via:
  - Path 1: `filtered_topic_ids` intersects `category_ids(topic)` (direct topic match)
  - Path 2: `unfiltered_topic_ids` intersects `category_ids(topic)` AND `filtered_topic_ids` intersects `expand_supertopic(topic)` (supertopic match — post matches a broader category)
- **Bulk** (>6 topics): inverts the logic — builds exclusion set = all production topic IDs minus expanded requested set; drops posts whose `filtered_topic_ids` are entirely in the exclusion set.

**Phase 2 — Topic Exclusion** (runs if `has_excluded_topics()`):
- Drops candidates whose `filtered_topic_ids` intersect excluded topics
- Candidates with empty or `None` `filtered_topic_ids` are **also dropped** when exclusions are active (conservative — uncertain topic posts removed)

**`TopicIdExpansion`** — full taxonomy:
- `category_ids(topic_id)` — maps topic to subtopics (~85+ IDs covering sports, politics, entertainment, etc.)
- `supertopic(topic_id)` — finds parent topic (e.g., `XAI_NFL → XAI_AMERICAN_FOOTBALL`)
- `expand_supertopic(topic_id)` — expands all siblings of the supertopic

---

### 13. `VideoFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/video_filter.rs` |
| Enable | Only when `query.exclude_videos == true` |
| Logic | Keeps candidates where `min_video_duration_ms.is_none()` (posts without video) |
| Notes | Removes video posts from the feed when the user/client requests a video-free feed |

---

### 14. `NewUserTopicIdsFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/new_user_topic_ids_filter.rs` |
| Enable | When `EnableNewUserTopicFiltering` param AND `has_new_user_topic_ids()` AND not a topic request |
| Logic | In-network candidates always pass. OON candidates kept only if their `filtered_topic_ids` intersects `expanded new_user_topic_ids`. |
| Notes | Focuses new users' OON content on their declared topic interests during onboarding |

---

### 15. `AncillaryVFFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/ancillary_vf_filter.rs` |
| Logic | Drops candidates where `drop_ancillary_posts == Some(true)` |
| Notes | Applied to ancillary posts (quotes, replies) that failed a separate VF (visibility filter) check |

---

## Post-Selection Filters

These run after `TopKScoreSelector` on the final top-K result.

### 1. `VFFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/vf_filter.rs` |
| Logic | Removes candidates where `visibility_reason` indicates a drop: `Some(SafetyResult(r))` where `r.action == Action::Drop(_)` OR any other `Some(...)` reason |
| Notes | Runs post-selection because VF labels may not be available for all candidates at pre-scoring time. Catches deleted/spam/violence/gore content that slipped through. `None` visibility reason = keep. |

### 2. `DedupConversationFilter`

| Attribute | Value |
|-----------|-------|
| File | `filters/dedup_conversation_filter.rs` |
| Dedup key | `min(ancestors)` — minimum tweet ID in the reply chain. Falls back to `tweet_id` if no ancestors. |
| Logic | Keeps only the highest-scored candidate per conversation. |
| Notes | Prevents multiple tweets from the same thread appearing in the feed. Uses minimum ancestor ID (root tweet) as the thread identifier. Score defaults to `0.0` if `None`. |

---

## Filter Interactions and Edge Cases

| Scenario | Filter behavior |
|----------|----------------|
| Post age cannot be determined from Snowflake ID | `AgeFilter` drops it (conservative) |
| Post has no `filtered_topic_ids` but topic exclusion is active | `TopicIdsFilter` drops it (conservative — unknown topic = excluded) |
| Retweet + original both in candidates | `RetweetDeduplicationFilter` keeps only the first one seen in the list |
| Author blocks viewer AND viewer hasn't blocked author | `AuthorSocialgraphFilter` still drops it (bidirectional block check) |
| `bloom_filter_entries` empty | `PreviouslySeenPostsFilter` falls back to exact `seen_ids` only |
| `impressed_post_ids` empty | `PreviouslySeenPostsBackupFilter` passes all through (disabled) |
| `visibility_reason == None` | `VFFilter` keeps the post (unknown = safe assumption) |

---

## Gotchas & Non-Obvious Behaviors

- **Filter order matters** — `DropDuplicatesFilter` runs first intentionally (cheapest). `CoreDataHydrationFilter` must run after core data hydration. Running them out of order could produce incorrect results or miss removals.

- **`TopicIdsFilter` drops unknown-topic posts during exclusion** — if a user has excluded topics, posts with no `filtered_topic_ids` are also excluded. This is a conservative choice that could cause content gaps for posts that haven't been classified yet.

- **`VFFilter` runs post-selection** — this means deleted or spam posts can make it into the top-K selection and only get removed at the very end. If many top-K posts are VF-dropped, the final served set may be significantly smaller than `TOP_K_CANDIDATES_TO_SELECT`.

- **`DedupConversationFilter` uses minimum ancestor ID** — not the root tweet's ID directly. Multiple tweets in the same thread all share the same minimum ancestor, so only the highest-scored one makes it into the feed.

- **`AuthorSocialgraphFilter` mutes only primary author** — muting someone only removes their direct posts. If a muted user is quoted or retweeted by someone you follow, that post passes through (unless blocked).

- **Bloom filter false positives** — `PreviouslySeenPostsFilter` may incorrectly filter posts that were never seen. The false positive rate is configured per `bloom_filter_entries[].false_positive_rate`.

---

## Cross-References

- [`02-home-mixer.md`](02-home-mixer.md) — Pipeline context
- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — Scoring after filtering
- [`09-decision-parameters.md`](09-decision-parameters.md) — Filter params and constants
