# Discovery and Reach: How X Surfaces Your Content to Non-Followers

## 1. TL;DR

There are two separate retrieval paths feeding the For You feed. **Thunder** serves posts from accounts you follow — pure chronological, no ML. **Phoenix** is how non-followers find your content — a two-tower neural network that matches post embeddings to user interest embeddings. To reach non-followers you must (a) get your post embedded by Grox, (b) cluster in embedding space near users who engage with your content type, and (c) produce a predicted engagement score strong enough to survive the OON score penalty applied to every out-of-network post before ranking.

---

## 2. The Two Paths: In-Network (Thunder) vs Out-of-Network (Phoenix)

### Thunder — In-Network Retrieval

Thunder serves posts from accounts a user already follows. Its logic is simple:

- Candidates are fetched sorted by `created_at` descending.
- There is no ML scoring inside Thunder itself — posts are ordered purely by recency.
- The retention window is **2 days**. Posts older than 48 hours drop out of Thunder retrieval entirely.

**Implication:** Your followers will see your posts in Thunder as long as they follow you and the post is less than 2 days old. Posting at high-traffic times increases the chance they see it before it ages out. Thunder cannot help you reach anyone who does not follow you.

### Phoenix — Out-of-Network Retrieval

Phoenix is the only path that puts your content in front of non-followers. It is a two-tower neural network retrieval system:

- **User tower**: takes the user's recent engagement history (last 128 interactions) and encodes it into a dense user embedding representing their interests.
- **Candidate tower**: encodes posts into post embeddings that live in a searchable index.
- **Similarity search**: Phoenix computes a dot-product similarity between the user embedding and all candidate post embeddings, then retrieves the top-K most similar posts.

Posts retrieved this way are out-of-network (OON) candidates. They then flow into the ranking stage, where they compete against in-network candidates — but with a score penalty applied first (see section 5).

---

## 3. How Phoenix Finds Your Posts — The Embedding Pipeline Explained

Your post does not automatically enter the Phoenix index. It must be embedded first.

**The pipeline:**

1. **Grox processes your post offline.** Grox runs `PlanPostEmbeddingV5`, a multimodal embedding plan that processes your post's text and media.
2. **An embedding vector is produced.** This vector encodes what your post is about — its topic, tone, media type, and the kinds of users likely to engage with it.
3. **The embedding goes into the Phoenix retrieval index.** Phoenix can now retrieve your post when it finds a user whose user-tower embedding is similar.
4. **`filtered_topic_ids` are assigned by X's content classification pipeline.** These IDs determine which topic feeds your post is eligible for — separate from the general Phoenix retrieval embedding.

If Grox fails to process your post, no embedding is written. Phoenix cannot retrieve what is not indexed. This is the primary way a post becomes invisible to non-followers even when nothing else is wrong (see section 8).

---

## 4. Why Niche Consistency Matters for Discovery

Embedding vectors for posts on similar topics cluster together in vector space. Users who consistently engage with a topic cluster together too (via their user-tower embeddings).

When you post consistently within a niche:
- Your posts' embeddings cluster in a predictable region of the vector space.
- Users whose user-tower embedding falls near that region will repeatedly find your posts via dot-product similarity search.
- Phoenix builds a feedback loop: each engagement from a new user updates their user-tower embedding slightly toward your content cluster, increasing the probability future searches retrieve your posts.

When you post across unrelated topics:
- Your posts scatter across embedding space.
- No single user-tower embedding is close to all of them simultaneously.
- Discovery becomes inconsistent — you may reach many different users but none reliably.

**Implication:** Niche consistency is not a content strategy suggestion; it is a geometric property of how similarity search works. Scattered content means scattered embeddings, which means lower retrieval recall from any given user's query.

---

## 5. The OON Penalty and How to Overcome It

Every out-of-network post retrieved by Phoenix enters the ranking stage with a score multiplier applied — after the weighted engagement sum and author diversity adjustment:

- **Standard OON**: score × `OonWeightFactor` (runtime parameter, typically <1.0)
- **Posts with topic IDs** (`topic_ids.len() > 0` on the candidate): score × `TopicOonWeightFactor` (separate parameter, also <1.0; takes priority over standard OON)
- **New users** (account age < `NewUserAgeThresholdSecs` AND following count ≥ `NEW_USER_MIN_FOLLOWING`): ranking score × `NEW_USER_OON_WEIGHT_FACTOR` — a **compile-time constant**, not a runtime parameter. It cannot be tuned without a redeploy.

The multiplied score must still beat in-network candidates to win a slot. In-network posts from Thunder carry no such penalty.

**What drives the pre-penalty score high enough to overcome the multiplier:**

- Strong predicted engagement (likes, replies, retweets, bookmarks) — these are what the ranker's ML models predict before applying the weight factor.
- Author quality signals (Health Score, Tweepcred) — a higher-authority author produces a higher base score.
- Media richness — posts with images or video can score higher on engagement prediction for certain audiences.
- Recency — a very old post produces a lower base score even if it was historically engaging.

**Implication:** You cannot change `OonWeightFactor` — it is set by platform operators at runtime. The only lever you control is driving your base predicted engagement score as high as possible so that even after the multiplier it clears the threshold. That means: write content your target audience demonstrably engages with, maintain author authority, and keep posts fresh.

---

## 6. New Account Strategy — What's Different

New accounts are routed differently at two points:

**At Phoenix retrieval:**
- If your account's action count (engagements given, not received) is below `PhoenixRetrievalNewUserHistoryThreshold`, Phoenix routes your request to a **separate new-user inference cluster** rather than the main cluster.
- This cluster likely has different index coverage and capacity characteristics.

**At OON weighting:**
- New accounts receive `NEW_USER_OON_WEIGHT_FACTOR` instead of the runtime-configurable `OonWeightFactor`.
- Because it is a compile-time constant, the platform cannot tune it per-deployment without a code change.

**Implication for new creators:** When your account is new, two disadvantages compound. As a user, you get a different (potentially lower-quality) retrieval path. As a creator trying to reach new users, those new users apply a hardcoded OON weight that you cannot influence. The fastest path out of this state is accumulating genuine engagement history — both given (follows, likes, replies from your account) and received (engagement on your posts). Once action counts clear the threshold, you graduate to the main retrieval cluster.

---

## 7. Topics and How They Affect Discovery

Topics interact with Phoenix retrieval in a non-obvious way:

**Bulk topic requests (>6 topics in the query) → Phoenix OON retrieval.**
When a user's For You feed query includes more than 6 topics, Phoenix handles retrieval and your post competes normally via embedding similarity.

**Single-topic requests (≤6 topics) → different retrieval path, not Phoenix.**
Small topic queries are routed to a specialized path. Phoenix is explicitly skipped for these. This means your post's Phoenix embedding does not help you here — topic feed eligibility in this path depends on `filtered_topic_ids` assigned by Grox content understanding.

**`filtered_topic_ids`:**
Topic IDs are assigned to your post by X's content classification pipeline. These IDs control which topic feeds your post can appear in. If your post receives no topic IDs or incorrect ones, you may be absent from relevant topic feeds regardless of your post's quality.

**PhoenixSource is disabled entirely when:**
- The request is a single-topic request (≤6 topics).
- `EnableNewUserTopicRetrieval` is active AND the user has `has_new_user_topic_ids` — the new-user topic path takes over.
- The request is for the Ranked Following tab (`in_network_only == true`).
- The system is in cached mode.

**Implication:** To appear in topic feeds via the Phoenix path, you need posts that attract bulk topic queries. For single-topic feeds, you depend entirely on Grox correctly assigning `filtered_topic_ids` to your post. You cannot override these assignments directly — your only lever is writing clearly topical content that Grox's content-understanding models can classify accurately.

---

## 8. What Breaks OON Discovery

### Grox embedding failure
If Grox fails to run `PlanPostEmbeddingV5` on your post — due to infrastructure failure, rate limiting, or content processing errors — no embedding is written to the Phoenix index. Phoenix cannot retrieve a post with no embedding. The post is functionally invisible to non-followers even if it passes all safety filters and has strong engagement signals.

There is no user-visible signal that this has happened.

### Safety flags and filtering
Posts flagged by safety systems (NSFW classifiers, spam detectors, sensitive content labels, low Health Score author) can be filtered out at multiple stages: before embedding, before Phoenix retrieval, after retrieval but before ranking, or after ranking but before serving. A post can clear Grox embedding and still be filtered by safety passes downstream.

### Topic mismatch
If Grox assigns `filtered_topic_ids` that do not match your intended audience, your post may surface to users unlikely to engage with it. Low engagement from mis-targeted users feeds back negatively — it signals the post is not relevant, which can suppress it further in ranking.

### In-network-only requests
When a user is viewing the Following tab (ranked or chronological), `in_network_only` is set to true and Phoenix retrieval is skipped entirely. No OON posts, including yours, appear in this context regardless of embedding quality.

### Cached mode
When Phoenix is in cached mode, OON retrieval is not performed fresh. Your new posts will not surface to non-followers until the cache expires or the system exits cached mode.

---

## 9. Cross-References

- **Thunder (in-network retrieval):** `docs/03-thunder.md`
- **Phoenix retrieval — two-tower architecture, embedding similarity, new-user routing:** `docs/04-phoenix-retrieval.md`
- **Grox — offline embedding computation, `PlanPostEmbeddingV5`, `filtered_topic_ids`:** `docs/06-grox.md`
- **Ranking — OON weight factors, score penalties, engagement prediction:** `docs/05-phoenix-ranking.md`
- **Filtering — safety flags, Health Score, NSFW classifiers:** `docs/08-filtering.md`
- **Decision parameters — `OonWeightFactor`, `TopicOonWeightFactor`, `NewUserAgeThresholdSecs`, `PhoenixRetrievalNewUserHistoryThreshold`:** `docs/09-decision-parameters.md`
- **Full candidate pipeline overview:** `docs/01-candidate-pipeline.md`
