# Phoenix — Retrieval Model (Two-Tower)

> **TL;DR**: Phoenix retrieval is a two-tower model that finds out-of-network posts relevant to a user. The user tower encodes the user's engagement history into an embedding; the candidate tower encodes every post in the global corpus. At serving time, a dot-product similarity search retrieves the top-K most similar posts. All embeddings use hash-based lookups — no vocabulary, no OOV.

---

## Role in Pipeline

```
(Offline) Grox → post embeddings → embedding index
                                         │
User engagement history                  │ (at serving time)
        │                                │
        ▼                                ▼
  PhoenixSource ──────► PhoenixRetrievalModel.retrieve()
        │                     ├── User Tower: history → user embedding
        │                     ├── Candidate Tower: corpus → post embeddings
        │                     └── top_k via dot product similarity
        ▼
  PostCandidate (served_type = ForYouPhoenixRetrieval)
```

---

## Files

| File | Purpose |
|------|---------|
| `phoenix/recsys_retrieval_model.py` | `PhoenixRetrievalModel` and `CandidateTower` |
| `phoenix/runners.py` | `load_model_params()`, `load_embedding_table()` |
| `phoenix/run_retrieval.py` | CLI for standalone retrieval inference |
| `phoenix/run_pipeline.py` | End-to-end retrieval → ranking CLI |
| `home-mixer/sources/phoenix_source.rs` | Calls retrieval service from pipeline |

---

## Architecture

### User Tower

```
User hashes ──► embedding lookup (multi-hash)
                      │
History post hashes ──► embedding lookup (multi-hash)
History author hashes ──► embedding lookup
History action IDs ──► action embedding
History product surface ──► product surface embedding
(optional) History continuous actions ──► normalized + embedded
(optional) User IP hashes ──► embedding lookup
                      │
All concatenated per timestep ──► proj_mat_3 projection
                      │
[user_token | history_tokens] ──► Transformer
                      │
Mean pool over valid positions ──► L2 normalize
                      │
user_representation (d_model dims)
```

### Candidate Tower

Two modes (selected at model config time via `enable_linear_proj`):

**MLP mode** (more expressive):
```
candidate_post_hashes ──► embedding lookup (multi-hash)
candidate_author_hashes ──► embedding lookup
(optional) age bucket ──► age embedding
Concatenated ──► 2-layer MLP (SiLU activation) ──► L2 normalize
```

**Mean-pool mode** (parameter-efficient):
```
candidate_post_hashes ──► embedding lookup (multi-hash)
candidate_author_hashes ──► embedding lookup
Concatenated ──► mean pool ──► L2 normalize
```

### Similarity Search

```python
scores = user_representation @ candidate_embeddings.T  # [B, N_candidates]
top_k_scores, top_k_indices = jax.lax.top_k(scores, k)  # JAX returns (values, indices)
```

---

## Key Data Structures

### `PhoenixRetrievalModelConfig`

| Field | Default | Notes |
|-------|---------|-------|
| `d_model` | — | Embedding dimension |
| `num_heads` | — | Transformer attention heads |
| `num_layers` | — | Transformer depth |
| `history_seq_len` | 128 | Max history timesteps |
| `product_surface_vocab_size` | 16 | |
| `num_hash_functions` | — | Multi-hash count |
| `enable_linear_proj` | — | MLP vs mean-pool candidate tower |

### `HashConfig`

| Field | Default | Notes |
|-------|---------|-------|
| `num_user_hashes` | 2 | Hash functions for user ID |
| `num_item_hashes` | 2 | Hash functions for post ID |
| `num_author_hashes` | 2 | Hash functions for author ID |
| `num_ip_hashes` | 0 | Disabled by default |

### `RetrievalOutput` (NamedTuple)

```python
RetrievalOutput(
    user_representation,  # [B, d_model] — L2-normalized user embedding
    top_k_indices,        # [B, K] — indices into candidate corpus
    top_k_scores,         # [B, K] — dot product similarities
)
```

---

## Hash Embedding Scheme

All IDs (users, posts, authors) are embedded via multiple hash functions, not a vocabulary lookup:

```python
def _hash_ids(ids, scale, bias, modulus, num_buckets):
    # Two-step: first linear congruential, then remap to valid bucket range
    raw = (ids * scale + bias) % modulus
    return (raw % (num_buckets - 1)) + 1  # maps to [1, num_buckets-1]; 0 = padding
```

For `num_hashes > 1`, multiple `(scale, bias)` pairs are applied and the resulting embeddings are summed. This eliminates OOV (out-of-vocabulary) problems and allows generalization to unseen IDs.

### Unified Embedding Table

All entity types (user, post, author) share a single embedding table constructed as:

```python
def build_unified_emb_table(user_emb, item_emb, author_emb):
    pad = jnp.zeros((65, d))   # 65-row padding prefix
    return jnp.concatenate([pad, user_emb, item_emb, author_emb])
```

The 65-row padding prefix means ID 0 maps to a zero embedding (padding token).

---

## Configuration & Parameters (Numeric)

| Constant | Value | Effect |
|----------|-------|--------|
| `EPS` | `1e-12` | Denominator floor for L2 normalization |
| `INF` | `1e12` | Used for masking invalid positions in mean pool |
| Embedding table padding | 65 rows | Zero-embedding prefix; ID 0 = padding |
| `run_pipeline.py top_k_retrieval` | 200 (default CLI) | Number of candidates retrieved |
| `run_pipeline.py top_k_display` | 30 (default CLI) | Top candidates shown in CLI output |
| `N_neg` in `run_pipeline.py` | 64 | Negative samples used in retrieval forward pass |
| `history_seq_len` | 128 | Retrieval model history window |

---

## `PhoenixSource` in home-mixer

### Enable Conditions

`PhoenixSource` is **disabled** when any of:
- `is_topic_request() && !is_bulk_topic_request()` — topic requests (≤6 topics) use a different retrieval path
- `EnableNewUserTopicRetrieval` param AND `has_new_user_topic_ids()` — new user topic path
- `in_network_only == true` — Ranked Following tab
- `has_cached_posts == true` — cached mode

### Cluster Routing

1. Base cluster: `PhoenixRetrievalInferenceClusterId` param
2. New-user override: if action count in `retrieval_sequence` < `PhoenixRetrievalNewUserHistoryThreshold` → switch to `PhoenixRetrievalNewUserInferenceClusterId`
3. Decider flags: `enable_phoenix_retrieval_lap7_to_fou`, `enable_phoenix_retrieval_fou_to_lap7`

### `in_reply_to_tweet_id` Quirk

PhoenixSource always sets `in_reply_to_tweet_id = Some(value)` even when the value is 0 (non-reply posts). ThunderSource only sets it when truly a reply. Downstream code must check `unwrap_or(0) != 0`.

---

## Gotchas & Non-Obvious Behaviors

- **Mean pooling masks padding** — the mean pool in `build_user_representation()` uses `valid_mask` to exclude padding timesteps. Positions with all-zero hashes are treated as padding and do not contribute to the user embedding.

- **Retrieval and ranking use different aggregation strategies** — retrieval: mean-pool transformer outputs → single user vector. Ranking: use only the candidate-position outputs from the transformer output sequence.

- **`enable_linear_proj` selects candidate tower mode at model init** — MLP vs mean-pool is a config-time decision, not a runtime flag. The two modes have different parameter counts and capabilities.

- **IP embeddings are additive** — if enabled (`num_ip_hashes > 0`), IP address embeddings are added to the user token embedding, not concatenated. Setting `num_ip_hashes=0` completely disables IP-based personalization.

- **L2 normalization uses `EPS=1e-12` floor** — the denominator is `max(norm, EPS)` to avoid division by zero for zero-vector embeddings (e.g., unseen users).

- **`N_neg=64` in run_pipeline.py** — the CLI uses 64 random negatives to build the candidate matrix for retrieval. In production, the full corpus is indexed separately.

---

## Cross-References

- [`00-overview.md`](00-overview.md) — System context
- [`05-phoenix-ranking.md`](05-phoenix-ranking.md) — Ranking model (uses same embedding scheme)
- [`09-decision-parameters.md`](09-decision-parameters.md) — All params
