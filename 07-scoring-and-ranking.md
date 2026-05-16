# Scoring and Ranking

> **TL;DR**: Ranking is a 3-step pipeline: (1) `PhoenixScorer` fetches ML-predicted probabilities for 14+ engagement types per candidate; (2) `RankingScorer` combines them into a weighted sum, applies author diversity decay, and multiplies out-of-network posts by an OON weight factor; (3) `TopKScoreSelector` sorts by final score and picks the top K.

---

## Scoring Pipeline (in order)

```
Candidates (post-filter)
     │
     ▼
1. PhoenixScorer
   → calls Phoenix inference service
   → sets candidate.phoenix_scores (raw probabilities per action)
     │
     ▼
2. RankingScorer
   → computes weighted_score = Σ(weight_i × P(action_i))
   → applies author diversity attenuation
   → applies OON weight factor
   → sets candidate.weighted_score (pre-diversity)
   → sets candidate.score (final)
     │
     ▼
3. TopKScoreSelector
   → sorts by candidate.score descending
   → truncates at TOP_K_CANDIDATES_TO_SELECT
```

---

## Step 1: Phoenix Scores

`PhoenixScorer` (`scorers/phoenix_scorer.rs`) calls the Phoenix inference service and populates `candidate.phoenix_scores`:

| Score field | Action | Signal direction |
|-------------|--------|----------------|
| `favorite_score` | Like | Positive |
| `reply_score` | Reply | Positive |
| `retweet_score` | Retweet | Positive |
| `quote_score` | Quote tweet | Positive |
| `click_score` | Post click | Positive |
| `profile_click_score` | Author profile click | Positive |
| `vqv_score` | Video quality view | Positive |
| `photo_expand_score` | Photo expand | Positive |
| `share_score` | Share | Positive |
| `share_via_dm_score` | Share via DM | Positive |
| `share_via_copy_link_score` | Copy link share | Positive |
| `dwell_score` | Dwell time | Positive |
| `follow_author_score` | Follow author | Positive |
| `not_interested_score` | Not interested | **Negative** |
| `block_author_score` | Block author | **Negative** |
| `mute_author_score` | Mute author | **Negative** |
| `report_score` | Report | **Negative** |
| `not_dwelled_score` | Not dwelled | **Negative** |
| `quoted_click_score` | Click on quoted tweet | Positive |
| `quoted_vqv_score` | Video view in quoted tweet | Positive |
| `cont_dwell_time_score` | Continuous dwell time | Positive |
| `cont_click_dwell_time_score` | Click + dwell time | Positive |

All values are sigmoid-activated probabilities in `[0, 1]`.

---

## Step 2: Weighted Score (`RankingScorer`)

### 2a. Raw Weighted Sum

```
weighted_score = Σ (weight_i × P(action_i))
```

Action weights are loaded from `Params` (feature switches) at runtime:

| Param key | Maps to | Notes |
|-----------|---------|-------|
| `FavoriteWeight` | P(favorite) | |
| `ReplyWeight` | P(reply) | |
| `RetweetWeight` | P(retweet) | |
| `QuoteWeight` | P(quote) | |
| `ClickWeight` | P(click) | |
| `ProfileClickWeight` | P(profile_click) | |
| `VqvWeight` | P(vqv) | Conditional on video duration (see below) |
| `PhotoExpandWeight` | P(photo_expand) | |
| `ShareWeight` | P(share) | |
| `ShareViaDmWeight` | P(share_via_dm) | |
| `ShareViaCopyLinkWeight` | P(share_via_copy_link) | |
| `DwellWeight` | P(dwell) | |
| `FollowAuthorWeight` | P(follow_author) | |
| `NotInterestedWeight` | P(not_interested) | Negative weight |
| `BlockAuthorWeight` | P(block_author) | Negative weight |
| `MuteAuthorWeight` | P(mute_author) | Negative weight |
| `ReportWeight` | P(report) | Negative weight |
| `NotDwelledWeight` | P(not_dwelled) | Negative weight |
| `QuotedClickWeight` | P(quoted_click) | |
| `QuotedVqvWeight` | P(quoted_vqv) | Conditional on video duration |

**Video duration gating**: `vqv_score` and `quoted_vqv_score` are only applied if `candidate.min_video_duration_ms > MinVideoDurationMs` param. Posts with short or no video get weight 0 for these actions. Also gated by `EnableQuotedVqvDurationCheck` param for the quoted variant.

**`positive_sum`** = sum of all positive weights (excludes `cont_dwell_time` and `cont_click_dwell_time`).

**`negative_sum`** = -(not_interested + block_author + mute_author + report + not_dwelled) — always non-negative in magnitude.

### 2b. Offset Normalization (`offset_score`)

Separates positive and negative scoring posts into different ranges:

```
NEGATIVE_SCORES_OFFSET = some_constant  (from Params)

if total_sum == 0.0:
    return max(0.0, combined_score)

if combined_score < 0.0:
    # Compress into [0, NEGATIVE_SCORES_OFFSET)
    return (combined_score + negative_sum) / total_sum × NEGATIVE_SCORES_OFFSET

else:
    # Shift positive scores above NEGATIVE_SCORES_OFFSET
    return combined_score + NEGATIVE_SCORES_OFFSET
```

**Effect**: posts that net-negative (more predicted blocks/mutes/reports than likes) occupy `[0, NEGATIVE_SCORES_OFFSET)`. Posts that net-positive occupy `[NEGATIVE_SCORES_OFFSET, ∞)`. This ensures positive-signal posts always outrank negative-signal posts in the selector.

### 2c. Author Diversity Attenuation

Prevents a single author from dominating the feed:

```
multiplier(position) = (1 - floor) × decay^position + floor

where:
  position = 0-indexed rank of this author's Nth post in the batch
  decay    = AuthorDiversityDecay param  (< 1.0, e.g. 0.5)
  floor    = AuthorDiversityFloor param  (> 0.0, e.g. 0.1)
```

| Author post position | Multiplier (example: decay=0.5, floor=0.1) |
|---------------------|---------------------------------------------|
| 0 (first post) | 1.0 — no penalty |
| 1 (second post) | 0.55 |
| 2 (third post) | 0.325 |
| 3 (fourth post) | 0.2125 |
| ∞ | → floor (0.1) |

Applied to `weighted_score`; result stored in `score`.

### 2d. Out-of-Network (OON) Weight Factor

Posts from Phoenix retrieval (`in_network == Some(false)`) get their score multiplied by an OON factor:

```
effective_oon_weight =
    if topic_ids.len() > 0:      TopicOonWeightFactor param
    else if is_new_user:          NEW_USER_OON_WEIGHT_FACTOR constant
    else:                         OonWeightFactor param

final_score = diversity_adjusted_score × effective_oon_weight  (if OON)
final_score = diversity_adjusted_score                          (if in-network)
```

**New user definition**: `user_age_secs < NewUserAgeThresholdSecs` AND `following_count >= NEW_USER_MIN_FOLLOWING`.

Note: `NEW_USER_OON_WEIGHT_FACTOR` is a **compile-time constant**, while `OonWeightFactor` and `TopicOonWeightFactor` are **runtime params** (feature-switch controlled).

---

## Step 3: Selector

`TopKScoreSelector` (`selectors/top_k_score_selector.rs`):
- Sorts by `candidate.score` descending
- `None` score treated as `NEG_INFINITY` (sorts to bottom)
- Truncates at `TOP_K_CANDIDATES_TO_SELECT` (compile-time constant)
- Candidates beyond the limit become `non_selected` (passed to side effects, not discarded)

---

## Scorer Variants

There are actually **four scorer implementations** in home-mixer. Which runs depends on pipeline configuration:

| Scorer | Field written | Notes |
|--------|--------------|-------|
| `PhoenixScorer` | `phoenix_scores` | Always first in inner pipeline |
| `RankingScorer` | `weighted_score` + `score` | Full scorer with all features |
| `WeightedScorer` | `weighted_score` only | Simpler subset of actions; compile-time weights |
| `AuthorDiversityScorer` | `score` only | Standalone diversity; reads `weighted_score` |
| `OONScorer` | `score` only | Standalone OON; compile-time constant `OON_WEIGHT_FACTOR` |

`RankingScorer` is the primary scorer in production — it subsumes the functions of `WeightedScorer`, `AuthorDiversityScorer`, and `OONScorer` in a single pass. The standalone scorers may be used for alternative pipeline configurations or experiments.

---

## Gotchas & Non-Obvious Behaviors

- **`cont_dwell_time` and `cont_click_dwell_time` not in `positive_sum`** — they contribute to the raw weighted sum but are excluded from the normalization denominator. This is an intentional asymmetry: continuous-value signals affect scores but don't shift the normalization baseline.

- **`weighted_score` vs `score`**: `weighted_score` is the pre-diversity weighted sum (used by standalone `AuthorDiversityScorer`). `score` is the final post-diversity, post-OON adjusted score used by the selector. Don't confuse them.

- **`in_network == None` treated as in-network** — if the `in_network` field is absent on a candidate, `OONScorer` does not apply the OON penalty. Candidates of unknown provenance are treated favorably.

- **Author diversity uses rank order at scoring time, not absolute position in feed** — it sorts by `weighted_score` to determine "which of author X's posts is highest ranked", then applies the decay multiplier. The second-highest scoring post from author X gets `multiplier(1)` regardless of its absolute feed position.

- **`WeightedScorer` uses compile-time constants** while `RankingScorer` loads from `Params` at runtime. If running `WeightedScorer`, weights cannot be changed without redeploying. `RankingScorer` can be tuned via feature flags.

- **New user OON factor is a constant, not a param** — `NEW_USER_OON_WEIGHT_FACTOR` is hardcoded in the binary. To change it, you need a code change + redeploy.

---

## Cross-References

- [`05-phoenix-ranking.md`](05-phoenix-ranking.md) — How Phoenix model produces the raw scores
- [`08-filtering.md`](08-filtering.md) — Filters that run before scoring
- [`09-decision-parameters.md`](09-decision-parameters.md) — All weight param keys and constants
