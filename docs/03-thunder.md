# Thunder — In-Network Post Store

> **TL;DR**: Thunder is a Rust service that keeps a hot, in-memory store of recent posts from all users. It consumes post create/delete events from Kafka, maintains per-user timelines in `DashMap<VecDeque>` structures, and serves in-network post candidates to home-mixer via gRPC in sub-milliseconds. There is no ML scoring in Thunder — it returns posts purely sorted by recency.

---

## Role in Pipeline

```
Kafka (tweet create/delete events)
     │
     ▼
Thunder (in-memory PostStore)
     │ gRPC: GetInNetworkPosts
     ▼
home-mixer / ThunderSource
     │
     ▼
PhoenixCandidatePipeline (post candidates → hydration → ML scoring)
```

---

## Files

| File | Purpose |
|------|---------|
| `thunder_service.rs` | gRPC service impl (`InNetworkPostsService`) |
| `posts/post_store.rs` | In-memory post store |
| `kafka/tweet_events_listener.rs` | Kafka consumer for tweet events |
| `kafka/tweet_events_listener_v2.rs` | V2 Kafka consumer variant |
| `deserializer.rs` | Protobuf tweet event deserialization |
| `main.rs` | Service entry point |

---

## Key Data Structures

### `PostStore` (`posts/post_store.rs`)

```
PostStore
  ├── posts: DashMap<tweet_id, LightPost>         // all posts, full data
  ├── original_posts_by_user: DashMap<user_id, VecDeque<TinyPost>>  // timeline
  ├── secondary_posts_by_user: DashMap<user_id, VecDeque<TinyPost>> // replies/RTs
  ├── video_posts_by_user: DashMap<user_id, VecDeque<TinyPost>>     // video-eligible
  └── deleted_posts: DashMap<tweet_id, bool>      // tombstone map
```

### `LightPost` (protobuf-defined)

Full post data kept in the `posts` map:

| Field | Type | Notes |
|-------|------|-------|
| `post_id` | `i64` | |
| `author_id` | `i64` | |
| `created_at` | `i64` | Unix seconds |
| `in_reply_to_post_id` | `Option<i64>` | |
| `in_reply_to_user_id` | `Option<i64>` | |
| `is_retweet` | `bool` | |
| `is_reply` | `bool` | |
| `source_post_id` | `Option<i64>` | Original tweet if RT |
| `source_user_id` | `Option<i64>` | Original author if RT |
| `has_video` | `bool` | |
| `conversation_id` | `Option<i64>` | Root of reply chain |

### `TinyPost` (in-memory only)

Minimal struct stored in per-user timelines — minimizes memory footprint:
```rust
struct TinyPost {
    post_id: i64,
    created_at: i64,  // Unix seconds
}
```

---

## How It Works

### Kafka Ingestion

1. `TweetEventsListener` consumes Kafka topic for tweet create/delete events.
2. On **create**: Deserializes to `LightPost`, inserts into `posts` map, appends `TinyPost` to per-user timeline queues (`original_posts_by_user`, `secondary_posts_by_user` if applicable, `video_posts_by_user` if video-eligible).
3. On **delete**: Inserts `tweet_id` into `deleted_posts` map. Also appends a sentinel entry to `original_posts_by_user[author_id]` under `DELETE_EVENT_KEY` — used to track delete event timestamps for retroactive trim ordering.
4. `finalize_init()` runs after bulk replay: sorts all timelines, trims old posts, applies delete events retroactively (because Kafka ordering is not guaranteed — a delete may arrive before its corresponding create).

### Serving

`ThunderServiceImpl::get_in_network_posts()`:
1. Acquires semaphore slot (non-blocking `try_acquire`). Returns `RESOURCE_EXHAUSTED` immediately if at capacity.
2. `score_recent(posts)` — sorts `LightPost`s by `created_at` descending, takes top `max_results`.
3. Returns `InNetworkPostsResponse`.

### Reply Inclusion Rules (`PostStore::get_posts_from_map`)

Replies go into `secondary_posts_by_user`. They are included in results only if:
1. The replied-to post (`in_reply_to_post_id`) is a root post (non-reply, non-RT), **OR**
2. The replied-to post is itself a reply to the conversation root AND `in_reply_to_user_id` is in `following_user_ids`.

This filters reply chains to mutual-network interactions — you see replies only when both sides are in your network.

### Video Eligibility

A post is video-eligible and added to `video_posts_by_user` if:
- `has_video == true`, OR
- It's a retweet whose `source_post_id` has video (even if `has_video == false` on the retweet itself)

AND it is not a reply.

---

## Configuration & Parameters

| Parameter / Constant | Source | Default | Effect |
|---------------------|--------|---------|--------|
| `retention` | `PostStore::default()` | **2 days** (172800 sec) | Posts older than this are trimmed and not served |
| `MAX_INPUT_LIST_SIZE` | `config` | — | Caps `following_user_ids` and `exclude_tweet_ids` in request |
| `MAX_POSTS_TO_RETURN` | `config` | — | Default result limit for posts |
| `MAX_VIDEOS_TO_RETURN` | `config` | — | Default result limit for video posts |
| `MAX_ORIGINAL_POSTS_PER_AUTHOR` | `config` | — | Per-author cap for original posts returned |
| `MAX_REPLY_POSTS_PER_AUTHOR` | `config` | — | Per-author cap for reply/retweet posts returned |
| `MAX_TINY_POSTS_PER_USER_SCAN` | `config` | — | Max `TinyPost` entries scanned per user before stopping |
| `MIN_VIDEO_DURATION_MS` | `config` | — | Minimum video duration to qualify as video-eligible |
| `ThunderMaxResults` | params (home-mixer) | — | Overrides `MAX_POSTS_TO_RETURN` at request time |
| `ThunderClusterId` | params (home-mixer) | — | Which Thunder cluster to route to |
| `ThunderAlgorithm` | params (home-mixer) | — | Algorithm variant to request |

---

## Gotchas & Non-Obvious Behaviors

- **Purely recency-sorted** — `score_recent()` sorts only by `created_at`. There is no engagement-based ranking in Thunder at all. All ML scoring happens downstream in home-mixer.

- **`DELETE_EVENT_KEY` sentinel** — `original_posts_by_user` stores delete event timestamps under a special sentinel user ID (`DELETE_EVENT_KEY`). This is not a real user; it's a mechanism to record when delete events arrived, enabling retroactive cleanup during `finalize_init()`.

- **Retroactive delete application** — Kafka event ordering is not guaranteed. During bulk replay, a delete may arrive before the create. `finalize_init()` post-processes all timelines to apply deletes retroactively after sorting.

- **Viewer's own retweets excluded** — posts where `source_user_id == requesting_user_id` are filtered from results. You don't see your own retweets in your Following feed.

- **At-capacity requests fail fast** — `try_acquire()` is non-blocking. If the semaphore is exhausted (Thunder under heavy load), the gRPC call immediately returns `RESOURCE_EXHAUSTED` rather than queueing.

- **VecDeque memory reclaim on trim** — when VecDeque capacity > 2× current size during trim, capacity is shrunk to 1.5× current length. This prevents unbounded memory growth after pruning old posts.

- **`following_user_ids` from Strato (debug-only)** — if the request has an empty `following_user_ids` AND `debug=true`, Thunder fetches the list from Strato. This is a debugging/testing path and does not run in production.

- **Kafka log counter** — batch logs are emitted only every 1000th batch (not every 100th as a code comment suggests — the actual constant is 1000).

---

## Cross-References

- [`00-overview.md`](00-overview.md) — System context
- [`02-home-mixer.md`](02-home-mixer.md) — ThunderSource integration
- [`08-filtering.md`](08-filtering.md) — AuthorSocialgraphFilter (mute/block applied downstream)
