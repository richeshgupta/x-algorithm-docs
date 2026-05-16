# X For You Feed Algorithm — Deep Research Documentation

A comprehensive technical deep-dive into X's open-source "For You" feed recommendation system, released May 2026. This documentation covers the full pipeline from architecture to ML models, filtering logic, scoring math, and non-obvious behaviors.

> **Source repo**: [xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)

---

## What's Inside

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

The system combines **in-network content** (posts from accounts you follow, served by Thunder) with **out-of-network content** (ML-discovered posts from a global corpus, served by Phoenix retrieval), scores everything with a **Grok-based transformer**, and assembles the final feed with ads and modules.

---

## Key Design Decisions

| Decision | What it means |
|----------|--------------|
| **No hand-engineered features** | The Grok transformer learns all relevance from raw engagement history. No manual feature pipelines. |
| **Candidate isolation** | During ranking, posts cannot attend to each other — only to user context. Scores are batch-independent and cacheable. |
| **Hash-based embeddings** | All IDs (users, posts, authors) use multiple hash functions. No vocabulary, no OOV problem. |
| **Multi-action prediction** | The model predicts 14+ engagement types (like, reply, repost, block, mute, report, etc.). Negative actions get negative weights. |
| **Two-pipeline architecture** | Inner pipeline handles ranking; outer pipeline handles feed assembly. Surfaces (For You, Topics, Ranked Following) reuse the inner pipeline. |
| **Offline content understanding** | Safety labels and embeddings (Grox) are pre-computed before serving, decoupling latency from classification cost. |

---

## Scoring Formula

```
Final Score = Σ (weight_i × P(action_i))
            → author diversity attenuation
            → OON weight factor (for out-of-network posts)
            → offset normalization
```

Positive actions (like, reply, repost, share) push scores up. Negative actions (block, mute, report, not interested) push scores down.

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

## Who This Is For

- **Engineers** building or debugging recommendation systems — see [Candidate Pipeline](docs/01-candidate-pipeline.md), [Scoring](docs/07-scoring-and-ranking.md), and [Gotchas](docs/10-gotchas.md)
- **ML researchers** studying the model architecture — see [Phoenix Retrieval](docs/04-phoenix-retrieval.md) and [Phoenix Ranking](docs/05-phoenix-ranking.md)
- **Product and policy teams** understanding what affects feed composition — see [Filtering](docs/08-filtering.md) and [Decision Parameters](docs/09-decision-parameters.md)
- **AI agents** querying the system — each doc is structured for targeted lookup with explicit file references and cross-links

---

## License

Documentation in this repo is original research and analysis. The underlying source code is licensed under [Apache 2.0](https://github.com/xai-org/x-algorithm/blob/main/LICENSE) by xAI.
