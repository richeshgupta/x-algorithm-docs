# X Thread Draft

Goal: Drive traffic to https://richeshgupta.github.io/x-algorithm-docs/
Source: Open-source X recommendation algorithm (xai-org/x-algorithm)

---

## Post 1 — The Burst Penalty (Hook)

Post 5 times in a burst and your algorithmic reach isn't 5x — it's ~2.2x.

The algorithm applies an exponential decay multiplier to each successive post from the same author in a batch: 1.0, 0.55, 0.325, 0.21, 0.16...

The math is in X's open-source ranking code. Here's what else is in there:

**Character count: 274**

---

## Post 2 — The Silent Killer

Every time someone scrolls past your post without pausing, the algorithm records a signal called `not_dwelled`.

It's a named negative engagement type with a negative weight — the same category as mutes and reports. It fires on every impression. Most creators have no idea it exists.

**Character count: 277**

---

## Post 3 — The Offset Cliff

A post with 9,999 raw score points loses to a post with 0.001 — if the first one has net-negative signals (mutes + reports > likes).

The algorithm puts net-negative posts in a score bucket that ranks below ALL positive-scoring posts. The separation is absolute, not relative.

**Character count: 265**

---

## Post 4 — The Geometry of Niche

Posting off-topic doesn't just confuse your audience — it geometrically degrades your discovery reach.

The retrieval model maps your posts as embeddings in vector space. Scattered topics = scattered embeddings = lower recall when X runs similarity search to find you for new audiences.

**Character count: 273**

---

## Post 5 — CTA

Full signal hierarchy, scoring math, and an AI agent guide you can copy into Claude or ChatGPT to get algorithm-backed advice on your account:

https://richeshgupta.github.io/x-algorithm-docs/

Sourced from the open-source X algorithm repo (xai-org/x-algorithm).

**Character count: 263**
