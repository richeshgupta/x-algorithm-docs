# Home Mixer — Orchestration Layer

> **TL;DR**: Home Mixer is the Rust service that assembles the For You feed. It runs two nested `CandidatePipeline` instances: an inner `PhoenixCandidatePipeline` that does all the retrieval, hydration, filtering, and ML scoring, and an outer `ForYouCandidatePipeline` that blends the ranked posts with ads, Who-To-Follow modules, prompts, and push-to-home posts.

---

## Role in Pipeline

```
ForYouFeedServer (gRPC → Thrift URT)
  └── ForYouCandidatePipeline (outer — blending only)
        └── ScoredPostsSource
              └── ScoredPostsServer (gRPC)
                    └── PhoenixCandidatePipeline (inner — ranking)
```

---

## Files

| Path | Purpose |
|------|---------|
| `for_you_server.rs` | gRPC handler → URT serialization |
| `scored_posts_server.rs` | Routes requests to `PhoenixCandidatePipeline`, converts to `ScoredPost` protos |
| `server.rs` | Wire-up: both servers share the same process |
| `main.rs` | Entry point |
| `candidate_pipeline/for_you_candidate_pipeline.rs` | Outer pipeline definition |
| `candidate_pipeline/candidate.rs` | `PostCandidate` — the universal candidate type |
| `candidate_pipeline/query.rs` | `ScoredPostsQuery` — the universal query/context type |
| `scorers/phoenix_scorer.rs` | Calls Phoenix inference service |
| `scorers/ranking_scorer.rs` | Weighted engagement sum + diversity + OON |
| `scorers/weighted_scorer.rs` | Simpler weighted sum (subset of actions) |
| `scorers/author_diversity_scorer.rs` | Standalone diversity attenuation |
| `scorers/oon_scorer.rs` | Standalone OON weight multiplier |
| `selectors/blender_selector.rs` | Feed assembly: posts + ads + WTF + prompts |
| `selectors/top_k_score_selector.rs` | Top-K by score for inner pipeline |
| `sources/thunder_source.rs` | In-network post source |
| `sources/phoenix_source.rs` | Out-of-network retrieval source |
| `ads/` | Ad injection and blending logic |

---

## Two-Pipeline Design

### Inner: `PhoenixCandidatePipeline`

Exposed via `ScoredPostsServer`. Contains all the heavy lifting:

| Stage | Components |
|-------|-----------|
| Query Hydrators | User features, scoring sequence, retrieval sequence, followed topics, starter packs, impression bloom filters, IP/location, mutual follow graph, served history |
| Sources | ThunderSource, PhoenixSource, CachedPostsSource, TweetMixerSource, PhoenixMoeSource, PhoenixTopicsSource |
| Hydrators | Core post data, author info, engagement counts, brand safety, language, media detection, quote expansion, VF labels, mutual follow scores |
| Pre-Scoring Filters | DropDuplicates, CoreDataHydration, Age, Selftweet, RetweetDedup, MutedKeyword, AuthorSocialgraph, PreviouslySeen, PreviouslyServed, IneligibleSubscription, TopicIds, Video, NewUserTopicIds |
| Scorers | PhoenixScorer → RankingScorer |
| Selector | TopKScoreSelector |
| Post-Selection Filters | VFFilter, DedupConversationFilter |
| Side Effects | Redis cache, scored stats, reranking Kafka, response timestamps |

### Outer: `ForYouCandidatePipeline`

Exposed via `ForYouFeedServer`. Zero filters, zero hydrators, zero scorers:

| Stage | Components |
|-------|-----------|
| Query Hydrators | ServedHistoryQueryHydrator, PastRequestTimestampsQueryHydrator |
| Sources | ScoredPostsSource, AdsSource, WhoToFollowSource, PromptsSource, PushToHomeSource |
| Selector | BlenderSelector |
| Side Effects | Ads logging, seen IDs → Kafka, served candidates → Kafka, client events, response stats, update past request timestamps, update/truncate served history |

---

## Key Data Structures

### `PostCandidate` (`candidate_pipeline/candidate.rs`)

The single candidate type used throughout both pipelines. Key fields:

| Field | Type | Notes |
|-------|------|-------|
| `tweet_id` | `u64` | Primary key |
| `author_id` | `u64` | 0 = hydration failed (used by `CoreDataHydrationFilter`) |
| `phoenix_scores` | `PhoenixScores` | Raw model outputs for all action types |
| `weighted_score` | `Option<f64>` | Pre-diversity score set by WeightedScorer/RankingScorer |
| `score` | `Option<f64>` | Final score after diversity + OON adjustment |
| `in_network` | `Option<bool>` | `None` treated as in-network |
| `served_type` | `Option<ServedType>` | `ForYouInNetwork`, `ForYouPhoenixRetrieval`, etc. |
| `ancestors` | `Vec<u64>` | Reply chain IDs — used by `DedupConversationFilter` |
| `visibility_reason` | `Option<FilteredReason>` | VF system result |
| `filtered_topic_ids` | `Option<Vec<i64>>` | Post-safety topic IDs |
| `unfiltered_topic_ids` | `Option<Vec<i64>>` | Pre-safety topic IDs |
| `mutual_follow_jaccard` | `Option<f64>` | Similarity score with viewer's network |

Important helpers on `PostCandidate`:
- `get_original_tweet_id()` — returns `retweeted_tweet_id` if set, else `tweet_id`. This is the **Phoenix prediction key** — retweets are scored on their source tweet.
- `get_original_author_id()` — same pattern for author.

### `ScoredPostsQuery` (`candidate_pipeline/query.rs`)

Carries all user context through the pipeline. Key fields:

| Field | Notes |
|-------|-------|
| `scoring_sequence` | User engagement history for ranking |
| `retrieval_sequence` | User engagement history for retrieval |
| `user_features` | Muted keywords, blocked/muted/followed/subscribed user IDs |
| `seen_ids` | Client-reported already-seen post IDs |
| `served_ids` | Server-tracked post IDs served this session |
| `bloom_filter_entries` | Probabilistic impression filter (skipped in serde) |
| `topic_ids / excluded_topic_ids / new_user_topic_ids` | Topic filtering context |
| `in_network_replies` | `OnceLock` populated by ThunderSource |
| `followed_grok_topics` | `Option<[bool; 32]>` — 32-element fixed array |
| `followed_starter_packs` | `Option<[bool; 20]>` — 20-element fixed array |
| `in_network_only` | True for Ranked Following tab |
| `has_cached_posts` | Disables Phoenix scoring and most sources |

---

## Feed Assembly (BlenderSelector)

Order of insertion into the final feed:

1. Posts and ads go through the ads blender first:
   - `AdsBlenderType == "safe_gap"` → `SafeGapAdsBlender` (gap-based insertion)
   - Otherwise → `PartitionOrganicAdsBlender`
2. Prompts inserted at positions 0, 1, 2, ... (prepended before organic content)
3. First WTF module inserted at `WHO_TO_FOLLOW_POSITION - 1` (clamped to list length)
4. `PushToHomePost` pinned to index 0 (overrides everything)

**Only the first WTF module is used** — additional modules are silently dropped.

---

## Surface Routing

`scored_posts_server.rs` tracks which surface a request is for:

| Surface | Condition |
|---------|-----------|
| `for_you` | Default |
| `topics` | `is_topic_request()` |
| `ranked_following` | `in_network_only == true` |
| `for_you_with_snoozed_topics` | `has_excluded_topics()` |

`product_surface` in the Phoenix scoring request also uses `in_network_only`: `HomeTimelineRankedFollowing` vs `HomeTimelineRanking`.

---

## Brand Safety

- Default verdict when absent: **`MediumRisk`** (`scored_posts_server.rs`)
- 18 safety label types mapped from `SafetyLabelType` enum to proto fields
- Safety labels contributed by Grox (offline pre-computation via VF hydrators)
- `AncillaryVFFilter` removes posts where `drop_ancillary_posts == Some(true)`

---

## Test / Debug Short-Circuits

Both `ForYouFeedServer::get_for_you_feed()` and `ScoredPostsServer` check for `TEST_USER_IDS` and short-circuit with a hardcoded empty/stub response. This is a test path, not a production path.

`following_user_ids` in ThunderSource can be fetched from Strato (bypassing the query) only when the list is empty AND `debug=true`.

---

## Gotchas & Non-Obvious Behaviors

- **`ForYouCandidatePipeline` has zero filters/scorers** — all ranking happens inside the inner `PhoenixCandidatePipeline`, which is invoked via `ScoredPostsSource`. The outer pipeline only blends and assembles.

- **`in_network_replies` is a `OnceLock`** — ThunderSource can only write it once. If somehow two sources tried to write it, the second write would silently fail.

- **`in_reply_to_tweet_id` from PhoenixSource is always `Some(...)`** even when 0. ThunderSource only sets it when truly a reply. Downstream code must check `unwrap_or(0) != 0` to treat it as a real reply.

- **Retweets are scored on the source tweet**, not the retweet's own ID (`get_original_tweet_id()`). This means a retweet's Phoenix score is the score of the original post — all reposts of the same content get the same ML prediction.

- **`weighted_score` stores the pre-diversity score; `score` stores the final score**. AuthorDiversityScorer reads `weighted_score` and writes to `score`. RankingScorer writes both. Do not confuse these two fields.

- **`has_cached_posts == true` disables**: ThunderSource, PhoenixSource, and PhoenixScorer — all three check this flag. Only the cache source runs.

- **Brand safety default is `MediumRisk`** — posts that failed VF hydration (no safety verdict) are treated as medium risk, not safe. This is a conservative default.

- **`cont_dwell_time` and `cont_click_dwell_time` are NOT in `positive_sum`** in `RankingScorer` — they contribute to the weighted score but are excluded from the total for normalization. This means offset normalization doesn't account for them.

---

## Cross-References

- [`00-overview.md`](00-overview.md) — System context and two-pipeline diagram
- [`01-candidate-pipeline.md`](01-candidate-pipeline.md) — Pipeline framework
- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — Scoring math in detail
- [`08-filtering.md`](08-filtering.md) — All filters
- [`09-decision-parameters.md`](09-decision-parameters.md) — All params and constants
