# Phoenix — Ranking Model (Grok Transformer)

> **TL;DR**: Phoenix ranking is a Grok-based transformer that takes a user's engagement history and a batch of candidate posts, and predicts the probability of each engagement type (like, reply, repost, click, etc.) for each candidate. The critical design choice is **candidate isolation**: candidates cannot attend to each other — only to the user context — so scores are batch-independent and cacheable.

---

## Role in Pipeline

```
ScoredPostsQuery (scoring_sequence, user context)
PostCandidate[] (tweet_id, author_id, etc.)
        │
        ▼
PhoenixScorer (home-mixer)
        │ gRPC request
        ▼
Phoenix inference service
        │
        ▼
PhoenixModel.__call__()
        │
        ▼
PhoenixScores per candidate (P(fav), P(reply), P(RT), ...)
        │
        ▼
RankingScorer → weighted sum → final score
```

---

## Files

| File | Purpose |
|------|---------|
| `phoenix/recsys_model.py` | `PhoenixModel`, `PhoenixModelConfig`, `RecsysBatch` |
| `phoenix/grok.py` | `Transformer`, `TransformerConfig`, RoPE implementation |
| `phoenix/runners.py` | Model param loading, embedding table construction |
| `phoenix/run_pipeline.py` | End-to-end CLI and action index constants |
| `phoenix/run_ranker.py` | Standalone ranking CLI |
| `home-mixer/scorers/phoenix_scorer.rs` | Calls inference service, maps results to candidates |

---

## Architecture

### Input Token Sequence

```
[user_token | history_token_1 | ... | history_token_H | cand_token_1 | ... | cand_token_C]

Total length = 1 + history_seq_len + candidate_seq_len
```

The transformer processes all tokens together; only the **candidate-position outputs** are used for scoring.

### User Token (`block_user_reduce`)

```python
user_hashes ──► embedding lookup (num_user_hashes × d_model)
(optional) IP hashes ──► embedding lookup
Concatenated ──► proj_mat_1 ──► d_model
```

### History Tokens (`block_history_reduce`, per timestep)

```python
history_post_hashes ──► embedding lookup
history_author_hashes ──► embedding lookup
history_action_ids ──► action embedding (signed: 2*action - 1)
history_product_surface ──► product surface embedding
(optional) continuous_actions ──► normalize + embed
(optional) post_age_bucket ──► age embedding  # NOT added to history tokens
All concatenated ──► proj_mat_3 ──► d_model
```

### Candidate Tokens (`block_candidate_reduce`, per candidate)

```python
candidate_post_hashes ──► embedding lookup
candidate_author_hashes ──► embedding lookup
candidate_product_surface ──► product surface embedding
candidate_post_age_bucket ──► age embedding   # ONLY candidate side has age
All concatenated ──► proj_mat_2 ──► d_model
```

### Transformer (from `grok.py`)

```
Transformer
  ├── num_layers × TransformerBlock
  │     ├── MultiHeadAttention (with RoPE, candidate isolation mask)
  │     └── DenseBlock (GELU activation)
  └── LayerNorm
```

> **Note on defaults**: `TransformerConfig` defaults are `widening_factor=4.0` and `attn_output_multiplier=1.0`. The open-source mini model uses overrides of `widening_factor=2.0` and `attn_output_multiplier=0.125` (set in `run_pipeline.py:151-152`), not architecture defaults.

**Candidate isolation mask**: candidates can attend to `[user | history]` tokens but NOT to each other. This is the key design decision.

### Output

```python
# Extract candidate positions from transformer output
cand_outputs = transformer_output[:, 1 + history_seq_len:, :]  # [B, C, d_model]

# Discrete action predictions
logits = cand_outputs @ unembedding_matrix  # [B, C, num_actions]
probs = sigmoid(logits)

# Continuous action predictions (optional)
cont_preds = sigmoid(cand_outputs @ continuous_mat)  # [B, C, num_continuous] — single linear layer
```

Output: `RecsysModelOutput(logits=[B,C,num_actions], continuous_preds=[B,C,num_continuous])`

---

## Action Indices

From `run_pipeline.py`:

The `IDX_*` constants in `run_pipeline.py` are **input action flag indices** (positions in the `history_actions` training vector), not output score indices:

| IDX Constant | Value | Input action meaning |
|---|---|---|
| `IDX_FAV` | 1 | `SERVER_TWEET_FAV` |
| `IDX_REPLY` | 4 | `SERVER_TWEET_REPLY` |
| `IDX_QUOTE` | 5 | `SERVER_TWEET_QUOTE` |
| `IDX_RT` | 6 | `SERVER_TWEET_RETWEET` |
| `IDX_DWELL` | 11 | `CLIENT_TWEET_RECAP_DWELLED` |
| `IDX_VQV` | 13 | `CLIENT_TWEET_VIDEO_QUALITY_VIEW` |

**Output score indices** (from `runners.py`) follow a different ordering: 0=favorite, 1=reply, 2=repost, etc. Do not conflate the two namespaces.

---

## Key Data Structures

### `RecsysBatch` (NamedTuple)

| Field | Shape | Notes |
|-------|-------|-------|
| `user_hashes` | `[B, num_user_hashes]` | User ID hashed |
| `history_post_hashes` | `[B, H, num_item_hashes]` | History post IDs |
| `history_author_hashes` | `[B, H, num_author_hashes]` | History author IDs |
| `history_actions` | `[B, H, num_actions]` | Binary action flags |
| `history_product_surface` | `[B, H]` | Surface IDs |
| `candidate_post_hashes` | `[B, C, num_item_hashes]` | Candidate post IDs |
| `candidate_author_hashes` | `[B, C, num_author_hashes]` | Candidate author IDs |
| `candidate_product_surface` | `[B, C]` | Surface IDs |
| `history_continuous_actions` | `[B, H, num_continuous]` | Optional continuous signals |
| `candidate_impr_ts` | `[B, C]` | Optional: candidate impression timestamp |
| `candidate_post_creation_ts` | `[B, C]` | Optional: post creation timestamp |
| `user_ip_hashes` | `[B, num_ip_hashes]` | Optional IP embedding |

### `PhoenixModelConfig`

| Field | Default | Notes |
|-------|---------|-------|
| `history_seq_len` | 128 | Max history timesteps fed to ranker |
| `candidate_seq_len` | 32 | Max candidates per batch |
| `product_surface_vocab_size` | 16 | |
| `post_age_granularity_mins` | 60 | Age bucket size in minutes |
| `num_continuous_actions` | 8 | Number of continuous prediction targets |
| `continuous_action_hidden_dim` | 64 | Hidden dim for projecting **input** continuous signals to embedding space (not the output head) |

### `ContinuousActionConfig`

| Field | Default | Notes |
|-------|---------|-------|
| `loss_weight` | — | Training loss weight |
| `loss_type` | — | |
| `tweedie_power` | 1.5 | Tweedie loss power parameter |
| `norm_config.norm_scale` | 30.0 | Clip ceiling for continuous value normalization |

---

## Configuration & Parameters (Numeric)

| Constant | Value | Effect |
|----------|-------|--------|
| `POST_AGE_MAX_MINUTES` | 4800 (80 hours) | Age values beyond this map to overflow bucket |
| `post_age_granularity_mins` | 60 | One bucket per hour |
| Total age buckets | `(4800 // 60) + 2` = **82** | `+2`: one padding/unknown bucket (0) + one overflow bucket |
| `widening_factor` (mini model) | 2.0 (override) | Default is 4.0; 2.0 is set in `run_pipeline.py` for the OSS mini model |
| `attn_output_multiplier` (mini model) | 0.125 (override) | Default is 1.0; 0.125 is set in `run_pipeline.py` for the OSS mini model |
| `norm_scale` | 30.0 | Max value before log normalization |
| `tweedie_power` | 1.5 | For count-type continuous targets |
| `EPS` | `1e-12` | Normalization floor |

---

## RoPE (Rotary Position Embedding)

From `grok.py`:

- Uses **right-anchored** RoPE: positions are computed relative to the most recent token, not from the left.
- Designed for variable-length histories where recent engagement should have consistent positional encoding regardless of history length.
- Actual signature: `right_anchored_rope_positions(padding_mask, history_seq_len, num_user_prefix_tokens)` — takes the padding mask array, not scalar seq/valid lengths.

---

## `PhoenixScorer` in home-mixer

### Enable Condition
```rust
fn enable(&self, query: &ScoredPostsQuery) -> bool {
    !query.has_cached_posts  // disabled when serving from cache
}
```

### Cluster Routing
1. Base: `PhoenixInferenceClusterId` param
2. New-user override: if action count in `scoring_sequence` < `PhoenixRankerNewUserHistoryThreshold` → use `PhoenixRankerNewUserInferenceClusterId`
3. Decider overrides: `override_qf_use_lap7` (Exp1Fou → Exp1Lap7), `override_qf_use_fou` (Exp1Lap7 → Exp1Fou)

### Prediction Key
Retweets are scored on their **source tweet ID** (`get_original_tweet_id()`), not the retweet's own ID. All reposts of the same content share the same ML prediction.

### Egress Fallback
If `UseEgressSidecar` param is true, uses the egress client. If egress fails, falls back to the main `phoenix_client`.

### `scoring_sequence == None`
If `scoring_sequence` is `None` (user has no engagement history), `PhoenixScorer` returns a default `PostCandidate` for every input — effectively no-op scoring. The ranking scorer will then use zero predictions.

---

## Gotchas & Non-Obvious Behaviors

- **Action encoding is signed**: `2 * action - 1`. So no-action = -1, action = +1. This is different from typical binary (0/1) encoding — it gives the model a symmetric signal for engagement vs. non-engagement.

- **`valid_mask` zeroes action embeddings** — timesteps where all action flags are zero (padding) have their action projections zeroed out before the transformer. This prevents padding tokens from contributing misleading action signals.

- **Candidate age embeddings are candidate-side only** — `candidate_post_age_bucket` is added to candidate tokens but there is no age embedding for history tokens. Post freshness is a prediction-time feature for candidates, not a historical encoding.

- **Missing timestamps → age bucket 0** — `compute_post_age_bucket()` returns 0 (not the overflow bucket) when timestamp is absent. Bucket 0 is the padding/unknown bucket, not "very old".

- **`mask_neg_feedback_on_negatives = True`** — a training config flag that masks negative action labels on negative samples (randomly drawn non-interacted posts). This prevents the model from learning that non-interacted posts always get negative actions.

- **Candidate isolation means scores don't depend on batch composition** — you can score a post alone or with 31 other posts and get the same prediction. This makes caching and partial re-scoring valid.

- **Unembedding matrix is an independent parameter** — `"unembeddings"` is a separate learned matrix, NOT weight-tied to the input embedding table. The doc previously claimed weight tying; this is incorrect.

- **Continuous predictions use a single linear head** — `continuous_mat` is a `[emb_size, num_continuous_actions]` matrix applied directly to `cand_outputs` followed by sigmoid. It is NOT a 2-layer MLP. The `continuous_action_hidden_dim` config field applies to a different function (`_project_continuous_value_to_embedding`) that embeds input continuous signals, not the output prediction head.

---

## Cross-References

- [`04-phoenix-retrieval.md`](04-phoenix-retrieval.md) — Two-tower retrieval (same hash embedding scheme)
- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — How Phoenix scores become the final ranking score
- [`09-decision-parameters.md`](09-decision-parameters.md) — All params and constants
