# Optimal Posting Cadence — Algorithm-Backed Guide

> Based on the open-source X recommendation algorithm source code.

---

## 1. TL;DR

- Your post is eligible for in-network distribution for **exactly 2 days** (172,800 seconds).
- The algorithm applies an **exponential score penalty** to every post after your first in any scoring batch — posting 5 times a day actively cannibalizes your own reach.
- **1–2 high-quality posts per day**, spaced at least 8–12 hours apart, is the cadence that avoids the decay trap while maximizing the freshness bonus.
- Timing matters because the algorithm prefers fresh content — but freshness is relative to when your followers' feeds are refreshed, not a fixed clock time.
- **Deleting and reposting is visible at the infrastructure level** and does not reset your distribution.

---

## 2. Your Post's Lifespan — The 2-Day Window

Thunder, the in-network post store, is configured with a retention of **172,800 seconds (exactly 48 hours)**. When that window closes, Thunder trims the post and it is no longer served to your followers via the in-network ranking path.

What this means practically:

- A post published Monday at 9 AM stops being eligible for in-network distribution Wednesday at 9 AM.
- There is no gradual fade — it is a hard cutoff.
- The Phoenix pipeline's `AgeFilter` compounds this: candidates whose age (derived from the Snowflake tweet ID timestamp) exceeds the configured `max_age` are **dropped entirely** before scoring. Posts with unknown or undecodable ages are also dropped conservatively.

The algorithm does not reward posts that "aged well." A post that got no engagement in the first 2 days gets no second chance through the in-network path. Out-of-network paths (search, trends, Fediverse amplification) are separate, but in-network feed placement is gated by this window.

**Implication:** Effort invested in a post's first 48 hours — timing, replies, sharing to communities — has a hard return deadline.

---

## 3. The Author Diversity Decay Problem

When the ranking pipeline scores a batch of candidates for a given user's feed, it applies an **author diversity multiplier** to prevent any single author from flooding the feed.

The formula:

```
multiplier(position) = (1 - floor) × decay^position + floor
```

With typical parameters (`decay = 0.5`, `floor = 0.1`), positions are 0-indexed and the multipliers stack as follows:

| Post # (same author, same batch) | Position | Multiplier | Effective score |
|---|---|---|---|
| 1st | 0 | 1.0000× | Full score |
| 2nd | 1 | 0.5500× | 55% of score |
| 3rd | 2 | 0.3250× | 32.5% of score |
| 4th | 3 | 0.2125× | 21.25% of score |
| 5th+ | 4+ | → 0.1× | Approaching floor |

The ranking orders candidates by `weighted_score` descending. That means only your **single best-performing post** gets its full score. Every additional post in the same batch competes against itself at a fraction of its value.

The decay floor of 0.1 is the absolute minimum — even an extremely high-quality second post will be ranked as if it scored 10% of its raw value relative to your first post, once enough of your posts are in the same batch.

**Why posting 5 times a day hurts:**

If you publish 5 posts within a short window and all 5 land in the same scoring batch for a follower's feed refresh:

- Post 1 competes at full strength.
- Posts 2–5 contribute diminishing marginal reach — post 5 is effectively invisible (≈10% weight).
- You have split your "author slot" in the feed across 5 posts instead of concentrating it on 1.

The total weighted reach of 5 simultaneous posts is roughly equivalent to **1.3–1.5 posts worth of reach** — not 5. The remaining 3.5–3.7 posts' worth of potential reach is lost to the decay penalty.

---

## 4. Optimal Cadence Derived from Algorithm Mechanics

The decay penalty applies **per scoring batch**, not per day. A scoring batch is generated each time a user refreshes their feed. If your posts land in different batches — i.e., they are separated by enough time that the user refreshed between them — each post gets its own unpenalized slot.

The practical target: space posts so that most of your followers see each post in a **separate feed refresh**. Average feed refresh intervals vary, but the algorithm is designed around session-level browsing patterns. Posts separated by **8–12 hours** have a high probability of landing in distinct batches for the majority of your followers.

**Recommended cadence:**

| Daily volume | Spacing | Expected outcome |
|---|---|---|
| 1 post/day | Any | Zero decay penalty; full score for that post |
| 2 posts/day | 10–12 hours apart | Low decay risk; both posts likely in separate batches |
| 3 posts/day | 7–8 hours apart | Moderate decay risk for followers who binge-refresh |
| 4+ posts/day | Any | High probability of batch collision; significant self-cannibalization |

The algorithm does not distinguish between "good" and "bad" days — decay applies every day equally. A quiet day with 1 post outperforms a heavy day with 5 posts in terms of in-network reach per post.

**Compounding with the 2-day window:** With 1–2 posts per day, you will have 2–4 active posts in Thunder at any given time. Each has its own eligibility window and competes independently. With 5+ posts/day, you fill Thunder with overlapping candidates that will decay against each other on every refresh.

---

## 5. Timing Within a Day — What the Algorithm Cares About

The algorithm does not use a fixed "best time to post" clock. What it cares about is **relative freshness at the moment a follower refreshes their feed**.

The `AgeFilter` scores freshness based on the Snowflake ID timestamp. A post is "fresh" when it is new relative to the follower's last seen post, not relative to a fixed UTC hour.

What this means for timing:

- Post when **your specific audience is actively refreshing feeds**, not at a globally published "optimal hour."
- The freshness advantage is strongest in the first **1–4 hours** after publishing — this is when the post has the highest probability of appearing at or near the top of a follower's ranked feed before newer posts from other accounts push it down.
- After 24 hours, even a high-engagement post loses freshness priority to newer content from other accounts.

**Practical implication:** Identify your audience's peak active hours through your own analytics. Post 30–60 minutes before peak, not during — so the post is fresh when peak traffic hits.

---

## 6. Replies and Threads — How They're Treated Differently

The in-network ranking pipeline applies **additional filters to replies** that do not apply to root posts:

- A reply from someone you follow is **only included** if the replied-to post is itself a root post, **or** the replied-to user is in your following list.
- Reply chains involving non-mutual connections are filtered aggressively.

What this means for threads:

- A thread (a reply chain from the same author to their own root post) is partially filtered. Only the root post of your thread is guaranteed in-network distribution. Subsequent replies in the thread may be dropped for followers who do not follow back any mentioned users.
- **Posting as a thread reduces in-network reach for all posts after the first.** Each reply in a thread counts against you on two dimensions: author diversity decay AND the reply inclusion filter.

**Recommended approach for long-form content:**

- Use the root post as a standalone entry point with your strongest hook.
- Accept that thread replies reach a narrower audience by design.
- If each section of your thread has independent value, consider posting them as separate root posts spaced 8–12 hours apart instead.

---

## 7. What Happens When You Delete and Repost

Thunder processes **delete events via Kafka**. Because Kafka does not guarantee create-before-delete ordering, Thunder handles delete events retroactively — applying them even if the delete message arrives before or after the create.

During bulk replay or startup, posts may briefly appear and then disappear as delete events are applied to the store.

**Why this matters for the "delete and repost" tactic:**

- The delete is registered at the infrastructure level and processed against Thunder's store.
- The repost is a new Snowflake ID with a new timestamp — but the delete event for the original post is already in the Kafka stream.
- **There is no "reset" of distribution.** The original post's engagement signals (replies, likes, retweets that occurred before deletion) do not transfer to the new post ID.
- The new post starts from zero engagement with no inherited signal, competing in a feed environment where your followers may have already been served the original post in a recent session, and that post ID is tracked in their served history.

The pipeline has two distinct seen-post filters: `PreviouslySeenPostsFilter` (client-reported seen IDs + bloom filter) and `PreviouslyServedPostsFilter` (server-tracked IDs served in the current session). A deleted-and-reposted version gets a new ID and is not filtered by either check — but it also gets no boost from the prior engagement.

**Bottom line:** Deleting and reposting yields a new post with no engagement history, visible delete/create churn in the infrastructure, and no recovered reach from the deleted version.

---

## 8. Cross-References

| Topic | Source location |
|---|---|
| Thunder retention (172,800s) | `src/scala/com/twitter/recos/user_tweet_entity_graph/` — Thunder store config |
| AgeFilter implementation | Phoenix retrieval pipeline — `AgeFilter` candidate filter |
| Author diversity decay formula | Phoenix ranking — `AuthorDiversityAdjustment` scorer |
| Previously served posts filter | `PreviouslyServedPostsFilter`, `EnableServedFilterAllRequests` param |
| Kafka delete ordering | Thunder store Kafka consumer notes |
| Reply inclusion rules | In-network candidate source — reply inclusion predicate |

For deeper reading on the pipeline stages these facts derive from, see:

- `docs/03-thunder.md` — Thunder store mechanics and retention
- `docs/04-phoenix-retrieval.md` — AgeFilter and candidate filtering
- `docs/05-phoenix-ranking.md` — Scoring, diversity adjustments, and weighted_score
- `docs/08-filtering.md` — PreviouslyServedPostsFilter and bloom filters
- `docs/09-decision-parameters.md` — EnableServedFilterAllRequests and other tunable params
