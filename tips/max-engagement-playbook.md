# Maximum Engagement Playbook
## Based on X Open-Source Recommendation Algorithm Source Code

---

## TL;DR

- **Replies, likes, and follows are the highest-value signals.** A post that sparks conversation or earns new followers outperforms one that gets spread silently. Exact weight ordering is runtime-configurable, but these signals drive the most durable engagement.
- **Post frequency works against you.** Each additional post from you in the same ranking batch receives exponentially decayed score. Your 2nd post may score only ~55% of its actual value; your 3rd, ~33%. Post less, make each post count more.
- **Negative signals are asymmetric.** A single "Not Interested" tap or mute does more damage than most positive signals can recover. Never provoke these.
- **Recency is non-negotiable.** The algorithm derives post age from the Snowflake tweet ID and actively penalizes old content. There is no "going viral days later" — freshness is a hard filter.
- **Out-of-network reach has a built-in tax.** Every post shown to non-followers gets multiplied by `OonWeightFactor` (typically < 1.0). You overcome this tax only by maximizing the positive signal weights that remain.

---

## The Signal Hierarchy

### High-Value Positive Signals

| Signal | What It Represents | Creator Implication |
|---|---|---|
| `reply` | User replied to your post | Thread-starting content wins. Questions, debates, takes that demand response. |
| `favorite` (like) | User liked your post | Broad approval signal. Lowest friction positive action — optimize for this at minimum. |
| `follow_author` | User followed you after seeing the post | Extremely high-weight signal. Posts that make strangers want more of you are algorithm gold. |
| `quote_tweet` | User quote-tweeted | Generates reply-class engagement signal. Controversial or quotable posts drive this. |
| `retweet` | User retweeted | Lower weight than reply/like but still positive. Broad, shareable content. |
| `share_via_dm` | Shared to someone via DM | Private sharing signal. Content people want to send specifically to someone — personal relevance. |
| `share` | Generic share action | Catch-all share. Platform-agnostic shareability. |
| `share_via_copy_link` | Link copied | Indicates intent to share outside X. Blog-worthy, reference-worthy content. |

### Medium-Value Positive Signals

| Signal | What It Represents | Creator Implication |
|---|---|---|
| `vqv` (video quality view) | Video watched past `MinVideoDurationMs` threshold | Only counts if viewer watches long enough. Short videos that don't cross threshold score zero here. |
| `quoted_vqv` | Video view in a quoted tweet | Same threshold applies. Quoted video content still accumulates signal. |
| `click` | User opened/expanded the post | Curiosity-driven. Intriguing headlines, truncated text, and thread hooks drive this. |
| `profile_click` | User clicked your profile | Strong interest signal. Posts that make users want to know who you are. |
| `quoted_click` | Click on a quoted tweet | If you embed another post, clicks on it count for you. |
| `photo_expand` | User expanded a photo | Image quality and intrigue matter. Thumbnails that compel a tap. |
| `dwell_time` | Time spent reading | Longer text with genuine value scores here. Dense, readable content. |
| `cont_dwell_time` | Continuous dwell (uninterrupted read) | Scroll-stopping content. No dwell if user bounces away mid-read. |
| `cont_click_dwell_time` | Click-through + dwell combined | Rewards posts where user both opens and reads thoroughly. |

### Negative Signals to Avoid

| Signal | What It Represents | Creator Implication |
|---|---|---|
| `not_interested` | "Not Interested" tapped | Hard negative. Suppresses your content in that user's feed. Avoid bait-and-switch content. |
| `mute_author` | User muted you | Persistent negative. That user never sees you again. High-volume spammy posting triggers this. |
| `block_author` | User blocked you | Strongest social rejection signal. Harassing or offensive content. |
| `report` | User reported your post | Severe. Triggers downstream moderation review on top of score penalty. |
| `not_dwelled` | Scrolled past without reading | Low-grade negative. Happens at scale with thumbnail-only posts, clickbait that doesn't land, or low-quality text. |

---

## The 10 Rules for Maximum Score

**1. Make replies the primary success metric, not likes.**

Design every post around a question, a take, or a tension that makes people want to respond. Replies are a high-friction positive signal — they require the user to compose a response, making them a strong positive indicator.

> Why: `reply` weight in `Σ(weight_i × P(action_i))` is a positive signal with a configurable `ReplyWeight` param. The Phoenix transformer predicts reply probability — posts that structurally invite response raise this probability.

---

**2. Treat each post as a standalone product. Post 1-2 times per day maximum.**

Your 2nd post in a ranking batch is scored at approximately `(1 - floor) × decay^1 + floor` of its raw score. By your 3rd post, you're at roughly a third of earned value. Posting 5 times a day means posts 3-5 are nearly invisible regardless of quality.

> Why: Author diversity decay applies a positional multiplier: `multiplier = (1 - floor) × decay^position + floor`. Only your top-scoring post escapes this penalty.

---

**3. Publish at the moment your audience is active, not when you have something to say.**

The algorithm hard-filters posts by age derived from the Snowflake tweet ID. A brilliant post published at 3 AM competes for ranking slots against fresh content from peak hours. You cannot recover lost freshness.

> Why: `age` is derived from Snowflake ID bits and used as a filter gate — posts beyond the age threshold are dropped before scoring, not merely downranked.

---

**4. Write text that requires reading, not just glancing.**

The `dwell_time`, `cont_dwell_time`, and `cont_click_dwell_time` signals are additive positive contributions to your score. A post read for 8 seconds scores more than one skimmed in 0.5 seconds, all else equal.

> Why: These three dwell signals each have independent weight coefficients in the scoring sum. They stack. Content that earns continuous, uninterrupted reading time accumulates all three.

---

**5. Make your first 2-3 lines earn the click.**

The `click` signal fires when a user opens/expands your post. Posts with hooks that create curiosity — truncated threads, surprising first lines, open loops — increase the predicted probability of `click`. This is distinct from engagement; it's pure interest.

> Why: `P(click)` is predicted by Phoenix transformer. Posts that structurally truncate at a compelling point raise this probability before a single user interacts.

---

**6. Use video only when you can hold attention past the minimum duration threshold.**

A video that gets skipped after 2 seconds contributes zero `vqv` signal and may contribute `not_dwelled`. A 45-second video watched to completion contributes `vqv` signal. Short, low-retention video is a net negative.

> Why: `vqv` (video quality view) only counts if video duration exceeds `MinVideoDurationMs`. Views below threshold do not trigger the signal — they may trigger `not_dwelled` instead.

---

**7. Build content that makes strangers want to follow you.**

`follow_author` is a distinct conversion signal — unlike a like, a follow represents lasting relationship formation. A post that earns follows from non-followers is doing something qualitatively different from a post that only earns passive approval.

> Why: `follow_author` score is a separate positive weight in the scoring sum (`FollowAuthorWeight` param). It represents a durable conversion event distinct from one-time engagement actions. The exact weight is runtime-configurable.

---

**8. Never post content that triggers "Not Interested" — not even once per audience segment.**

A single `not_interested` tap from a user effectively removes you from their future feeds. At scale, if 5% of impressions trigger this, your OON reach degrades rapidly because the algorithm reduces the probability of showing your posts to similar non-followers.

> Why: `not_interested` is a hard negative signal in the scoring formula with a negative weight coefficient. It is not offset by likes from other users on the same post — scoring is per-user.

---

**9. Accept that out-of-network reach costs more effort than in-network reach.**

Every impression on a non-follower's feed has your score multiplied by `OonWeightFactor < 1.0`. You start at a structural disadvantage. To appear in OON feeds consistently, your positive signals must overcome this multiplier on every post.

> Why: OON posts receive `score × OonWeightFactor` before competing for feed slots. In-network posts do not receive this penalty. Going viral requires clearing a higher absolute score threshold.

---

**10. Never let users see the same post twice — but know that they won't.**

The platform tracks previously seen posts via bloom filter and exact ID matching. There is no benefit to reposting or boosting old posts hoping new users see them — the system actively filters out content already shown to a given user.

> Why: Previously-seen filter runs before scoring. A post already shown to a user is dropped from their candidate pool regardless of its score, bloom filter false-positive rate is low, and exact ID matching catches the rest.

---

## Common Mistakes That Kill Your Score

**Posting 5-10 times per day**
Triggers author diversity decay on posts 2+. Posts 3 and beyond score at roughly 33% or less of their raw value. You are paying a compounding tax on your own output volume.
Algorithm signal: `multiplier = (1 - floor) × decay^position + floor`

---

**Writing bait-and-switch headlines**
Users who feel misled tap "Not Interested" or mute. These negative signals are weighted heavily and are permanent per-user. Your click rate goes up; your net score goes down.
Algorithm signal: `not_interested`, `mute_author`

---

**Uploading short videos under the minimum duration threshold**
If your video doesn't hold attention past `MinVideoDurationMs`, it generates zero `vqv` signal. If users swipe past without engaging, it generates `not_dwelled`. Net effect: negative.
Algorithm signal: `vqv` (absent), `not_dwelled`

---

**Posting without understanding your audience's active hours**
Snowflake-derived post age is computed at ranking time. A post that is 8 hours old when your audience is active may not survive the age filter. You cannot retroactively make a post fresher.
Algorithm signal: Age filter (pre-scoring drop gate)

---

**Posting content that triggers blocks or reports**
`block_author` and `report` are the two strongest negative signals. A report triggers downstream moderation review, not just a score penalty. This is not recoverable through subsequent positive engagement on the same post.
Algorithm signal: `block_author`, `report`

---

**Relying entirely on retweets for reach**
Retweets are a positive signal but a passive one — no composition required. A post that generates active replies and likes alongside retweets scores more broadly than one that spreads silently. Optimize for conversation, not just redistribution.
Algorithm signal: `retweet_score` × `RetweetWeight` — positive but lower-friction than `reply_score`

---

**Ignoring image quality**
`photo_expand` is a positive signal. A blurry, unreadable, or uncompelling thumbnail generates `not_dwelled` instead. Every image is an opportunity to earn a medium-weight positive signal or lose one.
Algorithm signal: `photo_expand` vs `not_dwelled`

---

**Posting threads as separate posts at once**
Each post in a thread is a separate scoring candidate. If you publish all 10 thread posts in rapid succession, all 10 compete in the same batch under author diversity decay. Post your thread opener, let it score, then continue.
Algorithm signal: Author diversity decay applies per-batch across all your posts

---

**Assuming virality compounds over days**
The age filter is a binary gate. Once a post ages out, it stops entering candidate pools regardless of engagement. Viral moments happen in the first hours or not at all.
Algorithm signal: Age filter (posts dropped before scoring once beyond threshold)

---

**Never engaging back in your own replies**
Your reply to your own thread is itself a new post by you. It generates a new scoring candidate, but it also means you are the author of two posts in the same batch — triggering diversity decay on yourself.
Algorithm signal: Author diversity decay + your reply competes as a separate candidate

---

## Cross-References

For tactics specific to content format decisions (threads vs. single posts, image ratios, video length recommendations):
- [Content Format Guide](./content-format-guide.md)

For understanding how the candidate pipeline selects posts before scoring even begins:
- [Candidate Pipeline Overview](../01-candidate-pipeline.md)

For the Phoenix ranking model internals and how it predicts each action probability:
- [Phoenix Retrieval](../04-phoenix-retrieval.md)
- [Phoenix Ranking](../05-phoenix-ranking.md)

For the Home Mixer and how posts are assembled into your final feed:
- [Home Mixer](../02-home-mixer.md)

For filtering mechanics — age filter, bloom filter, previously-seen filtering:
- [Filtering](../08-filtering.md)

For the decision parameters that govern OonWeightFactor and other tunable constants:
- [Decision Parameters](../09-decision-parameters.md)

For known edge cases and gotchas the algorithm documentation calls out explicitly:
- [Gotchas](../10-gotchas.md)

---

*Source: X open-source recommendation algorithm repository. Weighted scoring formula: `weighted_score = Σ(weight_i × P(action_i))`. The final ranking score additionally applies author diversity decay and OON weight multiplier. All signal names are from Phoenix transformer action definitions in the source code.*
