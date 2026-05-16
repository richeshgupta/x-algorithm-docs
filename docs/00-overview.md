# X For You Feed Algorithm — System Overview

> **TL;DR**: The For You feed is a fully ML-driven recommendation pipeline that retrieves posts from two sources (accounts you follow + a global ML-discovered corpus), scores all candidates with a Grok-based transformer, and assembles the final feed with ads, prompts, and Who-To-Follow modules. There are no hand-engineered relevance features — the transformer does all the heavy lifting from raw engagement history.

---

## Pipeline at a Glance

```
User Request
     │
     ▼
[Home Mixer — outer pipeline]
     │  Query Hydration: served history, past timestamps
     │
     ├──► [ScoredPostsServer — inner pipeline]
     │         │  Query Hydration: user features, engagement seq, retrieval seq, topics,
     │         │                   bloom filters, IP, mutual follow graph
     │         │
     │         ├──► [Thunder]        → in-network posts (followed accounts, recency-sorted)
     │         ├──► [Phoenix]        → out-of-network posts (ML similarity search)
     │         ├──► [CachedPosts]    → previously cached scored posts
     │         ├──► [TweetMixer]     → supplemental posts
     │         │
     │         │  Hydration: core post data, author info, engagement counts,
     │         │             brand safety, language, media, quote expansion, VF labels
     │         │
     │         │  Pre-Scoring Filters: dedup, age, self-posts, blocked/muted,
     │         │                       muted keywords, seen/served posts
     │         │
     │         │  Scoring:
     │         │    1. PhoenixScorer       → transformer engagement predictions
     │         │    2. RankingScorer       → weighted sum + author diversity + OON factor
     │         │
     │         │  Selector: TopKScoreSelector
     │         │
     │         │  Post-Selection Filters: VF (deleted/spam/violence), dedup conversations
     │         │
     │         └──► ScoredPost protos
     │
     ├──► [Ads Source]         → injected ads
     ├──► [Who To Follow]      → user recommendation modules
     ├──► [Prompts Source]     → feed prompts
     ├──► [Push To Home]       → pinned push-notif posts
     │
     │  Selector: BlenderSelector (interleaves all above)
     │
     │  Side Effects: Kafka logging, served history update, client events
     │
     ▼
Ranked Feed Response (Thrift/URT)
```

---

## Component Map

| Component | Language | Role |
|-----------|----------|------|
| `home-mixer/` | Rust | Outer orchestration layer. Assembles final feed from scored posts + ads + modules. gRPC → Thrift URT. |
| `candidate-pipeline/` | Rust | Reusable pipeline framework. Defines traits: Source, Hydrator, Filter, Scorer, Selector, SideEffect. |
| `thunder/` | Rust | In-memory store of recent posts from all users. Kafka-fed. Serves in-network candidates in sub-milliseconds. |
| `phoenix/` | Python (JAX) | ML backbone. Two-tower retrieval model + Grok-based transformer ranker. |
| `grox/` | Python (asyncio) | Offline content-understanding pipeline: safety, spam, embeddings, ASR, reply ranking. Runs separately from serving. |

---

## Two-Pipeline Architecture

A critical design detail: there are **two nested `CandidatePipeline` instances**, not one.

### Inner: `PhoenixCandidatePipeline` (via `ScoredPostsServer`)
- Handles retrieval, hydration, filtering, and ML scoring
- Output: ranked `ScoredPost` protos with engagement probability scores
- Sources: Thunder, Phoenix, CachedPosts, TweetMixer, PhoenixMoE, PhoenixTopics
- All the filtering and ML scoring happens here

### Outer: `ForYouCandidatePipeline` (via `ForYouFeedServer`)
- Has **zero filters, zero hydrators, zero scorers**
- Its only job: take scored posts + ads + WTF + prompts → blend into a feed
- Manages served history and deduplication across sessions
- Output: Thrift URT timeline

**Why two pipelines?** The inner pipeline is also called directly by other surfaces (Ranked Following, Topics, etc.) that need scored posts but not the full assembled feed. The outer pipeline adds the feed-specific assembly on top.

---

## Scoring Formula

```
Final Score = Σ (weight_i × P(action_i))
```

The Phoenix transformer predicts probabilities for 14+ action types:

| Action | Weight direction |
|--------|----------------|
| favorite, reply, repost, quote, share | Positive |
| click, profile_click, video_view, photo_expand, dwell | Positive |
| follow_author | Positive |
| not_interested, block_author, mute_author, report | **Negative** |

Scores then go through:
1. **Author diversity attenuation** — exponential decay for repeated authors
2. **OON factor** — out-of-network posts get score multiplied by `OonWeightFactor` param
3. **Offset normalization** — negative-scoring posts are normalized to a different scale than positive ones

See [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) for full math.

---

## Data Flow for Content Understanding (Grox)

Grox runs **offline**, not in the serving path:

```
Kafka (post create events)
     │
     ▼
Grox Dispatcher → Engine → PlanMaster
     │  (all plans run in parallel per post)
     ├── Safety screening
     ├── Spam detection
     ├── Multimodal embedding (v4/v5)
     ├── Reply ranking scores
     └── PTOS policy classification
     │
     ▼
Results written to Kafka/Strato sinks
     │
     ▼
home-mixer reads safety labels (VF hydrators)
Phoenix retrieval reads embedding index
```

This means content safety labels and embeddings are **pre-computed** before serving — not computed at request time.

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| No hand-engineered features | Transformer learns all relevance from engagement sequences. Eliminates feature pipeline complexity. |
| Candidate isolation in ranking | Candidates can't attend to each other — only to user context. Scores are batch-independent and cacheable. |
| Hash-based embeddings | Multiple hash functions per entity; no OOV problem; memory-efficient at scale. |
| Multi-action prediction | Predicting 14 actions (not one score) lets the system balance positive engagement against negative signals (mute, block, report). |
| In-memory Thunder store | Avoids DB round-trips for in-network lookups; Kafka ensures near-realtime freshness. |
| Offline content understanding (Grox) | Safety and embeddings are expensive; pre-computing them decouples latency from serving path. |

---

## Surface Types

The same scoring infrastructure serves multiple feed surfaces. `scored_posts_server.rs` tracks these explicitly:

| Surface | Description |
|---------|-------------|
| `for_you` | Main For You feed |
| `topics` | Topic-filtered feed |
| `ranked_following` | Following tab (ranked, not chronological) |
| `for_you_with_snoozed_topics` | For You with topic snooze applied |

---

## Cross-References

- [`01-candidate-pipeline.md`](01-candidate-pipeline.md) — Pipeline framework internals
- [`02-home-mixer.md`](02-home-mixer.md) — Orchestration and ad blending
- [`03-thunder.md`](03-thunder.md) — In-network post store
- [`04-phoenix-retrieval.md`](04-phoenix-retrieval.md) — Two-tower retrieval model
- [`05-phoenix-ranking.md`](05-phoenix-ranking.md) — Transformer ranker
- [`06-grox.md`](06-grox.md) — Content understanding pipeline
- [`07-scoring-and-ranking.md`](07-scoring-and-ranking.md) — All scoring logic
- [`08-filtering.md`](08-filtering.md) — All filters
- [`09-decision-parameters.md`](09-decision-parameters.md) — Master parameter reference
- [`10-gotchas.md`](10-gotchas.md) — Non-obvious behaviors
