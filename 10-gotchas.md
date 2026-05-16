# Gotchas & Non-Obvious Behaviors — Consolidated

> **TL;DR**: A curated list of the most surprising, counter-intuitive, and dangerous behaviors across the entire X algorithm codebase. Each entry includes the component, the behavior, and why it matters.

---

## Architecture

### Two pipelines, not one
**Component**: home-mixer  
**Behavior**: `ForYouCandidatePipeline` (outer) has zero filters, zero hydrators, zero scorers. All ranking happens inside `PhoenixCandidatePipeline` (inner), invoked via `ScoredPostsSource`. The outer pipeline is purely a feed assembly and blending layer.  
**Why it matters**: If you're debugging a ranking issue, look at the inner pipeline. If you're debugging feed layout/ads/WTF module positions, look at the outer pipeline.

### `ForYouCandidatePipeline` is also used for other surfaces
**Component**: home-mixer  
**Behavior**: The same `ScoredPostsServer` / `PhoenixCandidatePipeline` serves "for_you", "topics", "ranked_following", and "for_you_with_snoozed_topics". The surface is inferred from query flags.  
**Why it matters**: A parameter change to `PhoenixCandidatePipeline` affects all surfaces simultaneously.

---

## Candidate Pipeline Framework

### Silent skip on hydrator/scorer length violation
**Component**: candidate-pipeline  
**Behavior**: If a `Hydrator` or `Scorer` returns a vector of different length than its input, the entire stage output is silently discarded and an error is logged. No panic, no exception.  
**Why it matters**: A bug in a hydrator that changes the vector length will silently discard all hydration results for that request. Symptom: candidates appear unhydrated with no visible error.

### Side effects receive non-selected candidates
**Component**: candidate-pipeline  
**Behavior**: `SideEffectInput` includes both `selected_candidates` AND `non_selected_candidates` (those beyond `result_size()`). Side effects see the full pre-truncation picture.  
**Why it matters**: Serving history, Kafka logging, and other side effects may log posts that were never shown to the user.

### Side effects run fire-and-forget after response
**Component**: candidate-pipeline  
**Behavior**: Side effects are spawned via `tokio::spawn` after the response is built. They do NOT block the response.  
**Why it matters**: Side effect failures (Kafka write errors, Redis timeouts) are invisible to the caller. Served history may be incomplete for a request if the side effect fails.

---

## Home Mixer

### `has_cached_posts` disables three major components
**Component**: home-mixer  
**Behavior**: When `has_cached_posts == true`, `ThunderSource`, `PhoenixSource`, AND `PhoenixScorer` all check this flag and disable themselves. Only `CachedPostsSource` runs.  
**Why it matters**: Cached mode is a fundamentally different code path. Debugging a serving issue requires knowing whether cached mode is active.

### Retweets scored on source tweet ID
**Component**: home-mixer / `PhoenixScorer`  
**Behavior**: `get_original_tweet_id()` returns `retweeted_tweet_id` if set, else `tweet_id`. Phoenix predictions use this as the key.  
**Why it matters**: All reposts of the same content share the same ML prediction. A viral tweet's ML score is the same regardless of which repost you're looking at.

### `weighted_score` vs `score` are different fields
**Component**: home-mixer scorers  
**Behavior**: `weighted_score` = pre-diversity weighted sum (written by `WeightedScorer`/`RankingScorer`). `score` = final post-diversity, post-OON adjusted score (written by `AuthorDiversityScorer`/`RankingScorer`).  
**Why it matters**: Using the wrong field for debugging or logging gives the wrong picture. `TopKScoreSelector` sorts by `score`, not `weighted_score`.

### `in_network == None` treated as out-of-network
**Component**: home-mixer / `scored_posts_server.rs`  
**Behavior**: `in_network.unwrap_or(false)` — a missing value defaults to `false` (out-of-network). Candidates where `in_network` was never set are treated as OON, not in-network.  
**Why it matters**: A bug that fails to set `in_network` on in-network candidates (from Thunder) would apply the OON penalty incorrectly, causing them to rank lower than intended.

### Brand safety defaults to `MediumRisk`
**Component**: home-mixer / `scored_posts_server.rs`  
**Behavior**: Posts with no VF safety verdict default to `MediumRisk`, not safe.  
**Why it matters**: If VF hydration fails silently, posts are treated as medium risk — not shown to users who have sensitive content filters. Prefer investigating hydration failures rather than assuming posts are safe.

### `NEW_USER_OON_WEIGHT_FACTOR` is a compile-time constant
**Component**: home-mixer / `RankingScorer`  
**Behavior**: The OON weight factor for new users is hardcoded, not a feature-switch param like `OonWeightFactor`.  
**Why it matters**: You cannot tune new-user OON weight via feature flags. A code change + redeploy is required.

### `cont_dwell_time` excluded from normalization denominator
**Component**: home-mixer / `RankingScorer`  
**Behavior**: `cont_dwell_time` and `cont_click_dwell_time` contribute to the raw weighted score but are excluded from `positive_sum` (used for offset normalization).  
**Why it matters**: The normalization math doesn't account for these signals. Posts heavy on continuous dwell time may score differently than expected from the normalization formula.

### Four scorer implementations with different behaviors
**Component**: home-mixer  
**Behavior**: `RankingScorer`, `WeightedScorer`, `AuthorDiversityScorer`, `OONScorer` are four separate implementations. `WeightedScorer` uses compile-time constants; `RankingScorer` uses runtime params. `WeightedScorer` covers fewer action types than `RankingScorer`.  
**Why it matters**: Which scorer is active depends on pipeline configuration. Running `WeightedScorer` means some action types (`not_dwelled`, `quoted_vqv`, `cont_click_dwell_time`) are not included in the score — they are present in `RankingScorer` but absent from `WeightedScorer`'s compile-time constants.

---

## Thunder

### Purely recency-sorted — no ML ranking
**Component**: thunder  
**Behavior**: `score_recent()` sorts by `created_at` descending. There is zero engagement-based ranking in Thunder.  
**Why it matters**: Thunder candidates enter the pipeline as recency-ranked. Phoenix scoring downstream is what makes them relevant, not any Thunder-side scoring.

### `DELETE_EVENT_KEY` sentinel in user timeline map
**Component**: thunder / `PostStore`  
**Behavior**: `original_posts_by_user` stores delete event timestamps under a special sentinel user ID (`DELETE_EVENT_KEY`), not in the `deleted_posts` map.  
**Why it matters**: `original_posts_by_user` contains a non-user entry. Code iterating user timelines must handle this sentinel entry.

### Delete events may arrive before creates (Kafka ordering)
**Component**: thunder  
**Behavior**: `finalize_init()` applies delete events retroactively after sorting all timelines, because Kafka does not guarantee create-before-delete ordering.  
**Why it matters**: During startup/replay, a post may briefly appear undeleted before `finalize_init()` runs. Do not assume the store is consistent before initialization completes.

### At-capacity requests fail immediately (no queue)
**Component**: thunder / `ThunderServiceImpl`  
**Behavior**: `try_acquire()` is non-blocking. At-capacity → immediate `RESOURCE_EXHAUSTED` gRPC error.  
**Why it matters**: Under traffic spikes, home-mixer may receive `RESOURCE_EXHAUSTED` errors from Thunder. There is no retry queue — home-mixer must handle this gracefully.

### Viewer's own retweets excluded
**Component**: thunder  
**Behavior**: Posts where `source_user_id == requesting_user_id` are filtered from results.  
**Why it matters**: If you retweet something, you won't see that retweet in your own Following feed.

### Reply inclusion applies network-proximity rules
**Component**: thunder  
**Behavior**: Replies only appear if the replied-to post is a root post, OR if the replied-to user is in the viewer's following list.  
**Why it matters**: Deep reply chains are filtered. You see replies that are directly relevant to your network, not all replies in a thread.

---

## Phoenix Retrieval

### `in_reply_to_tweet_id` always `Some(...)` from PhoenixSource
**Component**: home-mixer / `phoenix_source.rs`  
**Behavior**: PhoenixSource sets `in_reply_to_tweet_id = Some(value)` even when the value is 0 (non-reply posts). ThunderSource only sets it when truly a reply.  
**Why it matters**: Downstream code must check `unwrap_or(0) != 0` to determine if a post is truly a reply. Checking `.is_some()` on PhoenixSource candidates will incorrectly classify all posts as replies.

### Retrieval disabled for single-topic requests
**Component**: home-mixer / `phoenix_source.rs`  
**Behavior**: `PhoenixSource` is disabled when `is_topic_request() && !is_bulk_topic_request()` (i.e., 1-6 topics). Only bulk topic requests (>6 topics) use Phoenix retrieval.  
**Why it matters**: For most topic feed requests, Phoenix retrieval is not running. Candidates come from different sources.

---

## Phoenix Ranking

### Action encoding is signed (+1/-1), not binary (1/0)
**Component**: phoenix / `recsys_model.py`  
**Behavior**: Actions encoded as `2 * action - 1`. No-action = -1, action = +1.  
**Why it matters**: The model receives symmetric signals. A 50/50 engagement history produces a zero-sum input, not a half-positive input. This affects gradient flow during training.

### Missing timestamp → age bucket 0 (not overflow)
**Component**: phoenix / `recsys_model.py`  
**Behavior**: `compute_post_age_bucket()` returns 0 when timestamp is absent. Bucket 0 is the padding/unknown bucket, NOT the "very old" overflow bucket.  
**Why it matters**: Posts with missing timestamps appear to the model the same as padding tokens — the model sees them as "unknown age", not "old". May affect how the model weighs these posts.

### Candidate age only on candidate side
**Component**: phoenix / `recsys_model.py`  
**Behavior**: `candidate_post_age_bucket` is added to candidate tokens only. History tokens do NOT get age embeddings.  
**Why it matters**: The model knows how fresh candidates are, but treats all historical engagements as age-agnostic. Recent engagements are not weighted more heavily by an explicit age signal (though RoPE positions provide implicit recency).

### Candidate isolation means batch composition doesn't matter
**Component**: phoenix / `recsys_model.py`  
**Behavior**: The attention mask prevents candidates from attending to each other. Score for post A is identical whether post B is in the batch or not.  
**Why it matters**: Phoenix predictions are cacheable per post per user. You can re-use a prediction for post A from a previous request even if the candidate set has changed.

### `scoring_sequence == None` → no-op scoring
**Component**: home-mixer / `phoenix_scorer.rs`  
**Behavior**: If the user has no engagement history (`scoring_sequence` is `None`), `PhoenixScorer` returns default `PostCandidate` for every input — all predictions are zero.  
**Why it matters**: Users with no history get zero Phoenix predictions. Scoring falls back entirely to the weights applied to zero-valued probabilities (effectively random or OON-penalized ranking).

### RoPE is right-anchored, not left-anchored
**Component**: phoenix / `grok.py`  
**Behavior**: `right_anchored_rope_positions` assigns position 0 to the most recent token, not the first.  
**Why it matters**: The model's positional encoding is designed for recency. The most recent engagement always has position 0 regardless of history length. Implementations using standard left-anchored RoPE would get wrong positions for variable-length histories.

---

## Grox

### All plans run on every task (no pre-filtering)
**Component**: grox / `plan_master.py`  
**Behavior**: `asyncio.gather` runs all 9 plans concurrently for every task. Each plan must check eligibility and early-exit if not applicable.  
**Why it matters**: If a plan fails to check eligibility, it will run on every post (potentially very expensive). There is no dispatcher-level pre-filtering.

### `success = False` if any one plan fails
**Component**: grox / `plan_master.py`  
**Behavior**: `TaskResult.success` is True only if ALL plans succeed. One plan failure marks the whole task as failed.  
**Why it matters**: A consistently-failing plan (e.g., ASR failing on text-only posts that shouldn't trigger ASR) will mark all text post tasks as failed, even if safety and embedding succeeded.

### Only the first multimodal embedding is kept
**Component**: grox / `plan_master.py`  
**Behavior**: `merge_results()` keeps only the first multimodal embedding from across all plans.  
**Why it matters**: If two plans both produce embeddings, the second one is silently discarded. Plan ordering matters for which embedding ends up in the index.

### Engine exits via `os._exit(0)` — no cleanup
**Component**: grox / `engine.py`  
**Behavior**: Engine subprocess exits with `os._exit(0)` after `asyncio.run()` returns.  
**Why it matters**: Python `__del__`, `atexit` handlers, and `finally` blocks do NOT run. Resources are not cleaned up. This is intentional to avoid multiprocessing hang issues.

### Task-level retry is independent of dispatcher retry
**Component**: grox  
**Behavior**: `Task.exec()` retries up to 2 times with 1s wait before returning failure. The dispatcher then retries the whole task up to `max_attempts` times.  
**Why it matters**: A single task can be executed up to `2 × max_attempts` times total. Factor this into idempotency design for task implementations.

### Dispatcher ACKs Kafka message on final failure
**Component**: grox / `dispatcher.py`  
**Behavior**: When a task exhausts all retries, the Kafka message is still ACKed (committed offset advances).  
**Why it matters**: Failed posts are not replayed indefinitely — they are silently dropped after `max_attempts`. Posts that consistently fail processing may never get safety or embedding annotations.

---

## Filtering

### `TopicIdsFilter` drops unknown-topic posts during exclusion
**Component**: home-mixer / filters  
**Behavior**: When topic exclusions are active, posts with `None` or empty `filtered_topic_ids` are dropped (treated as having an unknown topic that might be excluded).  
**Why it matters**: Newly published posts that haven't been classified by Grox yet will be excluded from feeds with topic exclusions active. This can cause a freshness gap in filtered feeds.

### `VFFilter` runs post-selection
**Component**: home-mixer / filters  
**Behavior**: VF (visibility filter) runs AFTER `TopKScoreSelector`, not before. Deleted/spam posts can make it into the top-K selection and are only removed at the very end.  
**Why it matters**: If many top-K posts are VF-removed, the final served set may be significantly smaller than `TOP_K_CANDIDATES_TO_SELECT`. Stale VF hydration increases the probability of VF removals at serving time.

### Bloom filter may produce false positives
**Component**: home-mixer / `PreviouslySeenPostsFilter`  
**Behavior**: `bloom_filter_entries` use probabilistic matching. A post may be incorrectly filtered if it hashes to a previously-seen ID.  
**Why it matters**: Users may occasionally miss posts due to false positives. The false positive rate is configured per filter entry (`false_positive_rate` field).

### `AgeFilter` drops posts whose age cannot be determined
**Component**: home-mixer / filters  
**Behavior**: If tweet age cannot be derived from the Snowflake ID, `unwrap_or(false)` causes the post to be dropped.  
**Why it matters**: Posts with invalid or non-Snowflake tweet IDs are silently removed by the age filter — not just old posts.

---

## Cross-References

- [`00-overview.md`](00-overview.md) — System overview
- [`01-candidate-pipeline.md`](01-candidate-pipeline.md) — Framework gotchas
- [`02-home-mixer.md`](02-home-mixer.md) — Home mixer gotchas
- [`03-thunder.md`](03-thunder.md) — Thunder gotchas
- [`04-phoenix-retrieval.md`](04-phoenix-retrieval.md) — Retrieval gotchas
- [`05-phoenix-ranking.md`](05-phoenix-ranking.md) — Ranking gotchas
- [`06-grox.md`](06-grox.md) — Grox gotchas
- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — Scoring gotchas
- [`08-filtering.md`](08-filtering.md) — Filtering gotchas
