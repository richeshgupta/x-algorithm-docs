# X For You Feed Algorithm — Deep Research Documentation

Comprehensive technical documentation of X's open-source "For You" feed recommendation system.
Source: [xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)

**Public Q&A Page:** https://richeshgupta.github.io/x-algorithm-docs/

---

## 5 Things That Will Surprise You

**How badly does posting in a burst hurt reach?**
Post 5 times in a row and your combined algorithmic weight is ~2.2 posts — the system applies an exponential decay multiplier to each successive post from the same author. (→ [Scoring & Ranking](docs/07-scoring-and-ranking.md#2c-author-diversity-attenuation))

**Is scrolling past a post really tracked?**
Yes. `not_dwelled` is a named negative engagement signal — every impression where a viewer scrolls past without pausing contributes a negative score to that post. (→ [Scoring & Ranking](docs/07-scoring-and-ranking.md))

**Can a highly-liked post rank below a post with zero likes?**
Yes. Net-negative posts (mutes + reports outweigh likes) are compressed into a score bucket below ALL positive-scoring posts, regardless of raw score magnitude. (→ [Scoring & Ranking](docs/07-scoring-and-ranking.md#2b-offset-normalization))

**Why does posting off-topic hurt discovery so much?**
The retrieval model uses vector embeddings — your posts cluster in a high-dimensional space. Scattered topics produce scattered embeddings, reducing how often you appear in similarity searches for any topic. (→ [Phoenix Retrieval](docs/04-phoenix-retrieval.md))

**Does X read your post content before showing it?**
Yes, but offline. The Grox pipeline pre-computes safety labels, spam scores, and semantic embeddings before your post enters the ranking queue. (→ [Grox](docs/06-grox.md))

---

## Use with AI

Copy [`tips/ai-agent-guide.md`](tips/ai-agent-guide.md) into any AI assistant (Claude, ChatGPT, etc.) to get algorithm-backed advice on your X account. The guide gives the AI the full signal hierarchy, scoring math, and filter logic — so it can answer questions like "why is my reach dropping?" or "what's the optimal posting cadence for my niche?" with citations from the source code.

---

## Documentation

| Doc | Description |
|-----|-------------|
| [00 — Overview](docs/00-overview.md) | End-to-end architecture, two-pipeline design, data flow, surface types |
| [01 — Candidate Pipeline](docs/01-candidate-pipeline.md) | Rust pipeline framework: traits, execution order, parallelism model |
| [02 — Home Mixer](docs/02-home-mixer.md) | Orchestration layer: inner/outer pipeline split, feed assembly, ad blending |
| [03 — Thunder](docs/03-thunder.md) | In-memory in-network post store: Kafka ingestion, reply rules, retention |
| [04 — Phoenix Retrieval](docs/04-phoenix-retrieval.md) | Two-tower retrieval model: hash embeddings, similarity search, cluster routing |
| [05 — Phoenix Ranking](docs/05-phoenix-ranking.md) | Grok transformer ranker: candidate isolation, RoPE, action prediction |
| [06 — Grox](docs/06-grox.md) | Content understanding pipeline: safety, spam, embeddings, ASR, reply ranking |
| [07 — Scoring & Ranking](docs/07-scoring-and-ranking.md) | Full scoring math: weighted sum, offset normalization, diversity decay, OON factor |
| [08 — Filtering](docs/08-filtering.md) | All 15 filters with logic, parameters, edge cases, and interaction table |
| [09 — Decision Parameters](docs/09-decision-parameters.md) | Master reference: every param key, constant, and config value with type and effect |
| [10 — Gotchas](docs/10-gotchas.md) | 30+ non-obvious behaviors, silent failures, and architectural surprises |

---

## Tips Guides

Practical guides derived from the algorithm docs — all claims map to real scoring variables and filter logic:

| Guide | Description |
|-------|-------------|
| [AI Agent Guide](tips/ai-agent-guide.md) | System prompt for AI assistants: turns any LLM into an algorithm-aware X advisor |
| [What Kills Your Reach](tips/what-kills-your-reach.md) | Diagnostic guide: every filter, negative signal, and silent suppression mechanism |
| [Maximum Engagement Playbook](tips/max-engagement-playbook.md) | Positive signal optimization: what the model actually rewards |
| [Posting Cadence](tips/posting-cadence.md) | Author diversity decay math translated into posting frequency strategy |
| [Content Format Guide](tips/content-format-guide.md) | How post structure (replies, quotes, media) affects scoring signals |
| [Discovery and Reach](tips/discovery-and-reach.md) | How out-of-network retrieval works and how to optimize for it |

---

## System Architecture

```
User Request
     │
     ▼
[Home Mixer — outer pipeline]      (feed assembly: posts + ads + WTF + prompts)
     │
     └──► [ScoredPostsServer — inner pipeline]   (retrieval + hydration + ML scoring)
               │
               ├──► [Thunder]          in-network posts (followed accounts, recency-sorted)
               ├──► [Phoenix]          out-of-network posts (two-tower ML retrieval)
               │
               │    Hydration → Filtering → Phoenix Transformer Scoring → Top-K Selection
               │
               └──► ScoredPost protos
```

In-network content (from accounts you follow) is served by Thunder. Out-of-network content (ML-discovered posts from a global corpus) is served by Phoenix retrieval. Both go through the same Grok-based transformer ranker before feed assembly.

---

## Scoring Formula

```
weighted_score = Σ(weight_i × P(action_i))
              → author diversity attenuation
              → OON weight factor (out-of-network posts only)
              → offset normalization
= final score
```

Positive actions (like, reply, repost, share, dwell) push scores up. Negative actions (block, mute, report, `not_dwelled`, not interested) push scores down. The offset normalization step ensures any net-negative post ranks below all net-positive posts in the final sort.

---

## Key Design Decisions

| Decision | What it means |
|----------|--------------|
| **No hand-engineered features** | The Grok transformer learns all relevance from raw engagement history. No manual feature pipelines. |
| **Candidate isolation** | During ranking, posts cannot attend to each other — only to user context. Scores are batch-independent and cacheable. |
| **Hash-based embeddings** | All IDs (users, posts, authors) use multiple hash functions. No vocabulary, no OOV problem. |
| **Multi-action prediction** | The model predicts 14+ engagement types simultaneously. Negative actions get negative weights. |
| **Two-pipeline architecture** | Inner pipeline handles ranking; outer pipeline handles feed assembly. Surfaces reuse the inner pipeline. |
| **Offline content understanding** | Safety labels and embeddings (Grox) are pre-computed before serving, decoupling latency from classification cost. |

---

## Components

| Component | Language | Role |
|-----------|----------|------|
| `home-mixer/` | Rust | Orchestration, gRPC → Thrift URT |
| `candidate-pipeline/` | Rust | Reusable pipeline framework |
| `thunder/` | Rust | In-memory in-network post store |
| `phoenix/` | Python (JAX) | Two-tower retrieval + Grok transformer ranker |
| `grox/` | Python (asyncio) | Offline safety, spam, embedding, ASR pipeline |

---

## License

Documentation in this repo is original research and analysis. The underlying source code is licensed under [Apache 2.0](https://github.com/xai-org/x-algorithm/blob/main/LICENSE) by xAI.
