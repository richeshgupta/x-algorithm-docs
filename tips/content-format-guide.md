# Content Format Guide: How Your Post Format Affects Your Algorithm Score

> Based on the open-source X recommendation algorithm. All signal names reference actual scoring variables in the codebase.

---

## 1. TL;DR

The algorithm scores every post by combining weighted signals — engagement probability scores computed by ML models. Your **format determines which signals you activate**. A short video below the minimum duration threshold literally gets weight=0 on `vqv_score`. A text post with a great hook earns `click_score` and `dwell_score`. A photo that makes people curious earns `photo_expand_score`. Knowing which signals each format activates — and which it misses — is the clearest lever you have for optimizing distribution.

Key takeaways:
- **Dwell time is underrated.** It contributes to raw score without shifting the normalization baseline.
- **Video must clear a minimum duration threshold** or the quality-view signal is zeroed out.
- **Photos win on expand probability**, not just impressions.
- **Replies have the highest per-signal weight** among engagement types.
- **Grox embeds your content offline.** Niche consistency compounds over time into better retrieval.

---

## 2. Format Rankings

| Format | Key signals activated | Algorithm advantage |
|---|---|---|
| **Video (above duration threshold)** | `vqv_score`, `dwell_score`, `cont_dwell_time_score`, `share_score`, `reply_score` | `vqv_score` is a strong dedicated signal; long dwell is natural for video |
| **Photo** | `photo_expand_score`, `dwell_score`, `reply_score`, `share_score` | Expand action is a high-intent signal; photos stop the scroll |
| **Text with hook / thread** | `click_score`, `dwell_score`, `cont_dwell_time_score`, `cont_click_dwell_time_score`, `reply_score` | Click + dwell compound; threads create sustained reading time |
| **Quote tweet** | `quote_score`, `dwell_score`, `reply_score` | Quote signal is positive; adds conversation surface |
| **Short video (below duration threshold)** | `dwell_score`, `reply_score`, `share_score` (no `vqv_score`) | Loses the dedicated video quality-view signal entirely |
| **Bare link post** | `click_score` (weak), `not_dwelled_score` (risk) | Low native dwell; users may scroll past, triggering the negative signal |
| **Reply (standalone)** | `reply_score` on the parent thread | Only surfaced if replied-to post is a root post OR the replied-to user is in the viewer's following list |

---

## 3. Video: What Actually Counts

### The Duration Gate

The algorithm applies a hard gate on video quality-view scoring:

```
if video_duration_ms <= MinVideoDurationMs:
    weight(vqv_score) = 0
```

Videos below the minimum duration threshold receive **zero weight on `vqv_score`** — the dedicated video quality signal. They are not penalized, but they forfeit the signal entirely. Short clips compete only on generic engagement signals (dwell, reply, share) that any format can earn.

**Practical implication:** If you post video, make sure it clears the minimum duration threshold. A video that is one second too short is treated as if the video quality signal does not exist.

### VQV Mechanics

`vqv_score` (video quality view) measures the probability that a user watches the video in a meaningful, high-quality way — not just a passive scroll-past autoplay. It is a positive signal that stacks on top of dwell signals. A strong video therefore activates:

- `vqv_score` — video-specific quality view
- `dwell_score` — time spent reading/watching the post
- `cont_dwell_time_score` — continuous dwell (see Section 5 for why this matters)
- Standard engagement signals: `reply_score`, `share_score`, `share_via_dm_score`

### Quoted Videos

The same duration gate applies to quoted tweets containing video. `quoted_vqv_score` follows identical gating logic — if the quoted post's video is below the threshold, the quoted video quality signal is zeroed out.

### Video Eligibility in Thunder (In-Network Store)

In Thunder (the in-network candidate retrieval layer), a post is classified as video-eligible if:

- `has_video == true`, **OR**
- It is a retweet of a video post that is **not itself a reply**

Retweeting a reply-video does not confer video eligibility. This matters for candidate retrieval: video-eligible posts are handled differently during in-network surfacing.

---

## 4. Photos: The Expand Signal

Photos activate `photo_expand_score` — the probability that a viewer taps to expand the image. This is a distinct, high-intent action: it requires deliberate engagement beyond passive scrolling.

**Why this matters:**
- Expand is a stronger intent signal than a like that costs one tap.
- It naturally extends dwell time, contributing to `dwell_score` and `cont_dwell_time_score`.
- Photos that are visually cropped or partially obscured in the feed preview create curiosity that drives expand actions.

**Format guidance:**
- Compose photos that reward expanding — something that is partially visible in preview but fully meaningful when expanded.
- Avoid photos where the entire value is visible in the thumbnail. If there is no reason to expand, `photo_expand_score` stays low.
- Alt text and captions contribute to Grox's multimodal embedding of your post (see Section 6).

---

## 5. Text Posts and Threads

### Dwell Time Signals

Text posts live or die by dwell time. Three positive signals are in play:

- **`dwell_score`**: probability the user spends significant time reading. Long, substantive text activates this.
- **`cont_dwell_time_score`**: continuous dwell time. Both this signal and `cont_click_dwell_time_score` are **excluded from the normalization denominator** (`positive_sum`), which means they contribute to the raw score without shifting the baseline that other signals are measured against. In practice, continuous reading time has an outsized impact relative to its stated weight.
- **`cont_click_dwell_time_score`**: click + dwell combined. If a user clicks to expand a thread and then reads, both behaviors are captured together in a single compounded signal.

The negative mirror is **`not_dwelled_score`**: the probability a user scrolls past quickly. A post that fails to stop the scroll is actively penalized, not merely unrewarded.

### The Click Signal and Thread Hooks

`click_score` measures the probability a user clicks to expand a post or thread. This is activated when a hook creates enough curiosity that the preview alone is insufficient — the user needs to click to get the payoff.

A thread structured as:

> "The one algorithm fact that changed how I post — [thread]"

...generates `click_score` on the opener, then `dwell_score` and `cont_dwell_time_score` as the user reads through. That stack of compounding signals is why threads outperform single long tweets for algorithmic reach.

**Hook design principle:** Do not deliver the entire value in the preview. Create a gap between what is visible and what is promised. The click fills that gap, triggering `click_score`.

### Profile Click Signal

`profile_click_score` — probability a user clicks your profile — is a positive signal. A compelling hook that makes someone want to know who wrote it drives profile clicks. This makes your bio and profile presentation a format consideration: they are part of the engagement surface your text post unlocks.

---

## 6. What Grox Sees in Your Content

Grox is the offline content understanding layer. It runs on every post **before** it enters the ranking pipeline.

### Multimodal Embedding

Grox embeds your post using:
- **Text** — the post body, any quoted text
- **Images** — visual features extracted from attached photos
- **Video** — processed via ASR (automatic speech recognition), so spoken words in video are embedded as text signals

These embeddings are stored and used by **Phoenix retrieval** — the system that surfaces your content to non-followers. Phoenix matches your post's embedding against user interest embeddings to decide who sees your content in their For You feed.

### Niche Consistency and Discovery

When you post consistently within a topic area, your embeddings cluster together in the embedding space. Phoenix retrieval finds your posts when it searches for content similar to what a user has previously engaged with.

**Implication:** Inconsistent posting across unrelated topics causes your embeddings to scatter. You may have individual posts that perform well, but you build no retrieval advantage over time. A focused content niche compounds — each post reinforces the cluster, making the next post easier to retrieve for the same audience.

### Safety Screening

Grox also runs safety classification and spam detection. Posts flagged at this stage are filtered **before** they enter the scoring pipeline — they never accumulate the engagement signals described in this guide. Format choices that trigger safety classifiers (misleading media, certain link patterns) effectively remove the post from the recommendation system.

---

## 7. Format Anti-Patterns to Avoid

| Anti-pattern | Why it hurts |
|---|---|
| **Video under the minimum duration threshold** | `vqv_score` weight becomes 0; you lose the primary video quality signal |
| **Bare link posts with no native content** | Low `dwell_score`; high `not_dwelled_score` risk as users scroll past without stopping |
| **Photos where full value is visible in thumbnail** | No reason to expand; `photo_expand_score` stays low |
| **Replies to non-root posts by users not in the viewer's following list** | Thunder's reply inclusion rules filter these out before they reach ranking |
| **Quoted video below duration threshold** | `quoted_vqv_score` is zeroed out by the same gate that applies to direct video |
| **Thread hooks that over-deliver in the preview** | Eliminates the curiosity gap; `click_score` stays low; `cont_click_dwell_time_score` never fires |
| **Topic-scattered posting** | Grox embeddings fragment; Phoenix retrieval cannot build a consistent audience cluster |
| **Content flagged by Grox safety classifiers** | Filtered before scoring; all downstream signals become irrelevant |

---

## 8. Cross-References

For deeper context on how these signals interact in the full scoring pipeline:

- **Candidate retrieval (Thunder/Phoenix):** How posts enter the pipeline before format-based signals are applied. See `docs/03-thunder.md` and `docs/04-phoenix-retrieval.md`.
- **Scoring weights and signal combination:** How `vqv_score`, `dwell_score`, and engagement signals are weighted and combined. See `docs/05-phoenix-ranking.md`.
- **Grox content understanding:** Full detail on multimodal embedding, safety classification, and offline processing. See `docs/06-grox.md`.
- **Filtering rules (replies, safety):** How reply inclusion rules and safety flags remove content before scoring. See `docs/08-filtering.md`.
- **Decision parameters:** Tunable thresholds including `MinVideoDurationMs` and normalization exclusions. See `docs/09-decision-parameters.md`.
