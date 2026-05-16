# Decision Parameters — Master Reference

> **TL;DR**: This doc catalogs every parameter, constant, and configuration value found across the X algorithm codebase. Parameters come in two types: **runtime params** (feature-switch controlled, can be changed without deploy) and **compile-time constants** (hardcoded, require code change + redeploy).

---

## Parameter Types

| Type | Source | Changeable at runtime? |
|------|--------|----------------------|
| `Params` (feature switch) | Loaded from feature-switch system per request | Yes |
| `Decider` flag | A/B experiment system | Yes |
| Compile-time constant (`p::` prefix or `const`) | Hardcoded in binary | No — requires redeploy |
| Config file value | Service config at startup | No — requires restart |

---

## Scoring Parameters

### Action Weights (`Params` — runtime, loaded by `RankingScorer::from_params`)

All weights are `f64`. Positive weights increase score; negative weights decrease it.

| Param Key | Action | Notes |
|-----------|--------|-------|
| `FavoriteWeight` | P(like) | |
| `ReplyWeight` | P(reply) | |
| `RetweetWeight` | P(retweet) | |
| `QuoteWeight` | P(quote tweet) | |
| `ClickWeight` | P(post click) | |
| `ProfileClickWeight` | P(author profile click) | |
| `VqvWeight` | P(video quality view) | Conditional: only applied if `min_video_duration_ms > MinVideoDurationMs` |
| `PhotoExpandWeight` | P(photo expand) | |
| `ShareWeight` | P(share) | |
| `ShareViaDmWeight` | P(share via DM) | |
| `ShareViaCopyLinkWeight` | P(copy link share) | |
| `DwellWeight` | P(dwell time) | |
| `FollowAuthorWeight` | P(follow author) | |
| `QuotedClickWeight` | P(click on quoted tweet) | |
| `QuotedVqvWeight` | P(video view in quoted tweet) | Conditional on video duration + `EnableQuotedVqvDurationCheck` |
| `NotInterestedWeight` | P(not interested) | **Negative signal** |
| `BlockAuthorWeight` | P(block) | **Negative signal** |
| `MuteAuthorWeight` | P(mute) | **Negative signal** |
| `ReportWeight` | P(report) | **Negative signal** |
| `NotDwelledWeight` | P(not dwelled) | **Negative signal** |

### Score Normalization (`Params`)

> **Note**: `NEGATIVE_SCORES_OFFSET` is a compile-time constant (`p::NEGATIVE_SCORES_OFFSET`), not a runtime Params key. It is listed below under compile-time constants.

### Score Normalization (Compile-time constants, `WeightedScorer`)

| Constant | Effect |
|----------|--------|
| `p::WEIGHTS_SUM` | Positive weight sum for offset normalization |
| `p::NEGATIVE_WEIGHTS_SUM` | Negative weight sum for offset normalization |
| `p::NEGATIVE_SCORES_OFFSET` | Offset constant |
| `p::VQV_WEIGHT` | VQV weight value |
| `p::MIN_VIDEO_DURATION_MS` | Video duration threshold for VQV eligibility |
| `p::FAVORITE_WEIGHT` | Compile-time favorite weight (used in `WeightedScorer`) |
| `p::REPLY_WEIGHT` | Compile-time reply weight |

(Full list mirrors the `Params` keys above but as compile-time constants)

---

## Author Diversity Parameters

| Param Key / Constant | Type | Effect |
|---------------------|------|--------|
| `AuthorDiversityDecay` | `Params` (runtime) | Geometric decay rate per repeated author. Lower = steeper penalty. E.g., `0.5` halves the score for each additional post from same author. |
| `AuthorDiversityFloor` | `Params` (runtime) | Minimum score multiplier regardless of author repetition count. E.g., `0.1` = 10% minimum. |
| `p::AUTHOR_DIVERSITY_DECAY` | Compile-time constant | Default value for `AuthorDiversityScorer` (standalone scorer) |
| `p::AUTHOR_DIVERSITY_FLOOR` | Compile-time constant | Default floor for standalone scorer |

**Formula**: `multiplier = (1 - floor) × decay^position + floor`

---

## Out-of-Network (OON) Parameters

| Param Key / Constant | Type | Effect |
|---------------------|------|--------|
| `OonWeightFactor` | `Params` (runtime) | Score multiplier for OON posts (general case). E.g., `0.8` = OON posts score at 80% of equivalent in-network posts. |
| `TopicOonWeightFactor` | `Params` (runtime) | OON multiplier when request is a topic feed (`topic_ids.len() > 0`) |
| `NEW_USER_OON_WEIGHT_FACTOR` | **Compile-time constant** | OON multiplier for new users. Cannot be changed without redeploy. |
| `NewUserAgeThresholdSecs` | `Params` (runtime) | User account age threshold (in seconds) to qualify as "new user" |
| `NEW_USER_MIN_FOLLOWING` | Compile-time constant | Minimum following count required to apply new-user OON factor |
| `p::OON_WEIGHT_FACTOR` | Compile-time constant | OON factor used by standalone `OONScorer` |

**New user OON applies when**: `user_age_secs < NewUserAgeThresholdSecs` AND `following_count >= NEW_USER_MIN_FOLLOWING`

---

## Video Parameters

| Param Key | Type | Effect |
|-----------|------|--------|
| `MinVideoDurationMs` | `Params` (runtime) | Minimum video duration (ms) for VQV weight eligibility |
| `EnableQuotedVqvDurationCheck` | `Params` (runtime) | When true, also applies duration check to `quoted_vqv_score` |
| `p::MIN_VIDEO_DURATION_MS` | Compile-time constant | Same threshold used by `WeightedScorer` |

---

## Phoenix Scoring (Ranker) Parameters

| Param Key | Type | Effect |
|-----------|------|--------|
| `PhoenixInferenceClusterId` | `Params` (runtime) | Base Phoenix ranker cluster |
| `PhoenixRankerNewUserHistoryThreshold` | `Params` (runtime) | Action count threshold; below this, route to new-user cluster |
| `PhoenixRankerNewUserInferenceClusterId` | `Params` (runtime) | New-user Phoenix ranker cluster |
| `UseEgressSidecar` | `Params` (runtime) | When true, send ranking request to egress sidecar; fall back to main on failure |

**Decider flags (A/B experiments)**:

| Decider Key | Effect |
|-------------|--------|
| `override_qf_use_lap7` | If cluster is `Experiment1Fou`, switches to `Experiment1Lap7` |
| `override_qf_use_fou` | If cluster is `Experiment1Lap7`, switches back to `Experiment1Fou` |

---

## Phoenix Retrieval Parameters

| Param Key | Type | Effect |
|-----------|------|--------|
| `PhoenixRetrievalInferenceClusterId` | `Params` (runtime) | Base retrieval cluster |
| `PhoenixRetrievalNewUserHistoryThreshold` | `Params` (runtime) | Action count threshold for new-user retrieval cluster |
| `PhoenixRetrievalNewUserInferenceClusterId` | `Params` (runtime) | New-user retrieval cluster |
| `PhoenixMaxResults` | `Params` (runtime) | Max candidates to retrieve |

**Decider flags**:

| Decider Key | Effect |
|-------------|--------|
| `enable_phoenix_retrieval_lap7_to_fou` | Switches Lap7 cluster to Fou |
| `enable_phoenix_retrieval_fou_to_lap7` | Switches Fou cluster to Lap7 |

---

## Thunder Parameters

| Param Key | Type | Effect |
|-----------|------|--------|
| `ThunderClusterId` | `Params` (runtime) | Which Thunder cluster to route to |
| `ThunderMaxResults` | `Params` (runtime) | Max posts to request from Thunder |
| `ThunderAlgorithm` | `Params` (runtime) | Algorithm variant to request |

**Config constants** (startup config, require restart):

| Constant | Effect |
|----------|--------|
| `MAX_INPUT_LIST_SIZE` | Caps `following_user_ids` and `exclude_tweet_ids` per request |
| `MAX_POSTS_TO_RETURN` | Default post result limit |
| `MAX_VIDEOS_TO_RETURN` | Default video result limit |
| `MAX_ORIGINAL_POSTS_PER_AUTHOR` | Per-author cap for original posts |
| `MAX_REPLY_POSTS_PER_AUTHOR` | Per-author cap for reply/retweet posts |
| `MAX_TINY_POSTS_PER_USER_SCAN` | Max VecDeque entries scanned per user |
| `MIN_VIDEO_DURATION_MS` | Minimum video duration for video-eligible classification |
| `retention` | 2 days (172800 sec) — post retention window |

---

## Selection Parameters

| Param Key / Constant | Type | Effect |
|---------------------|------|--------|
| `FOR_YOU_MAX_RESULT_SIZE` | Compile-time constant | Max result size for outer `ForYouCandidatePipeline` |
| `TOP_K_CANDIDATES_TO_SELECT` | Compile-time constant | Max candidates from inner `PhoenixCandidatePipeline` |

---

## Feed Assembly Parameters (BlenderSelector)

| Param Key | Type | Effect |
|-----------|------|--------|
| `AdsBlenderType` | `Params` (runtime) | `"safe_gap"` → `SafeGapAdsBlender`; else → `PartitionOrganicAdsBlender` |
| `PROMPTS_POSITION` | Compile-time constant | Feed position for prompt insertion |
| `WHO_TO_FOLLOW_POSITION` | Compile-time constant | Feed position for Who-To-Follow module |

---

## Filtering Parameters

| Param Key / Constant | Type | Effect |
|---------------------|------|--------|
| `EnableServedFilterAllRequests` | `Params` (runtime) | When true, `PreviouslyServedPostsFilter` runs on all requests (not just bottom-of-feed) |
| `EnableNewUserTopicFiltering` | `Params` (runtime) | Enables `NewUserTopicIdsFilter` |
| `EnableNewUserTopicRetrieval` | `Params` (runtime) | Disables `PhoenixSource` when user has new-user topic IDs |

---

## Phoenix Model Constants (Python, `recsys_model.py`)

| Constant | Value | Effect |
|----------|-------|--------|
| `POST_AGE_MAX_MINUTES` | 4800 (80 hours) | Age values beyond this map to overflow bucket |
| `post_age_granularity_mins` | 60 | Age bucket size in minutes |
| Total age buckets | 81 | 80 hour-buckets + 1 overflow + 0 for padding |
| `widening_factor` | 2.0 | Transformer MLP hidden dim = d_model × 2.0 |
| `attn_output_multiplier` | 0.125 | Scales attention output before residual |
| `NormConfig.norm_scale` | 30.0 | Clip ceiling for continuous value normalization |
| `ContinuousActionConfig.tweedie_power` | 1.5 | Tweedie loss power for count-type targets |
| `EPS` | `1e-12` | Normalization denominator floor |
| `INF` | `1e12` | Masking value for invalid positions |
| Embedding table padding | 65 rows | Zero-vector prefix; ID 0 = padding |

---

## Phoenix Model Config Defaults (`PhoenixModelConfig`)

| Field | Default |
|-------|---------|
| `history_seq_len` | 128 |
| `candidate_seq_len` | 32 |
| `product_surface_vocab_size` | 16 |
| `post_age_granularity_mins` | 60 |
| `num_continuous_actions` | 8 |
| `continuous_action_hidden_dim` | 64 |

## Hash Config Defaults (`HashConfig`)

| Field | Default |
|-------|---------|
| `num_user_hashes` | 2 |
| `num_item_hashes` | 2 |
| `num_author_hashes` | 2 |
| `num_ip_hashes` | 0 (disabled) |

---

## CLI / Inference Defaults (`run_pipeline.py`)

| Argument | Default | Notes |
|----------|---------|-------|
| `top_k_retrieval` | 200 | Candidates retrieved by two-tower model |
| `top_k_display` | 30 | Top candidates shown in CLI output |
| `N_neg` | 64 | Negative samples in retrieval forward pass |

**Demo display weights** (not production weights):

| Action | Demo weight |
|--------|-------------|
| Favorite | 1.0 |
| Reply | 0.5 |
| Retweet | 0.3 |
| Dwell | 0.2 |

---

## Grox Configuration

| Parameter | Source | Effect |
|-----------|--------|--------|
| `max_in_flight` | `grox_config.dispatcher` | Back-pressure threshold for task queue filling |
| `max_attempts` | `grox_config.dispatcher` | Max retry count per task |
| `graceful_shutdown_timeout` | config | Process join timeout on shutdown |
| Task retry policy | hardcoded | `stop_after_attempt(2), wait_fixed(1)` |
| Dispatcher init retry | hardcoded | `wait_incrementing(start=1, increment=3, max=9)`, up to 3 attempts |

---

## Cross-References

- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — How weights are used
- [`08-filtering.md`](08-filtering.md) — How filter params are used
- [`05-phoenix-ranking.md`](05-phoenix-ranking.md) — Phoenix model constants
- [`04-phoenix-retrieval.md`](04-phoenix-retrieval.md) — Retrieval model constants
- [`03-thunder.md`](03-thunder.md) — Thunder config constants
