# Grox — Content Understanding Pipeline

> **TL;DR**: Grox is an async Python pipeline that runs offline (not in the serving path) to classify, embed, and annotate every post. It consumes post events from Kafka, runs all applicable content-understanding plans concurrently per post (safety, spam, embeddings, reply ranking, ASR), and writes results to Kafka/Strato sinks. These pre-computed annotations feed into home-mixer's VF hydrators and Phoenix's embedding index at serving time.

---

## Role in Pipeline

```
Kafka (post create events)
     │
     ▼
Grox Dispatcher
  └── PriorityTaskGenerator (weighted-random over multiple Kafka streams)
        │
        ▼
Grox Engine (subprocess, asyncio event loop)
        │ (per task)
        ▼
PlanMaster.exec() ─── asyncio.gather() → all plans run in parallel
  ├── PlanInitialBanger        → banger quality screen
  ├── PlanPostSafety           → post safety classification
  ├── PlanPostEmbeddingV5      → multimodal embedding
  ├── PlanPostEmbeddingV5ForReply → embedding for reply posts
  ├── PlanPostEmbeddingWithSummary → embedding + summarization
  ├── PlanReplyRanking         → reply ranking score
  ├── PlanSafetyPtos           → PTOS policy classification
  └── PlanSpamComment          → spam detection
        │
        ▼
Results → Kafka / Strato sinks
  ├── Safety annotations → VF system → home-mixer VF hydrators
  ├── Embeddings → Phoenix retrieval embedding index
  └── Spam / category labels → content filtering
```

---

## Files

### Top-Level

| File | Purpose |
|------|---------|
| `engine.py` | Runs in a subprocess; asyncio event loop polling task queue |
| `dispatcher.py` | Manages Kafka generators, fills task queue, handles retries |
| `main.py` | Entry point; wires dispatcher, engine, processors |

### Plans

| File | Purpose |
|------|---------|
| `plans/plan_master.py` | Runs all plans in parallel via `asyncio.gather` |
| `plans/plan_initial_banger.py` | Initial quality screen for new posts |
| `plans/plan_post_safety.py` | Post safety classification |
| `plans/plan_post_embedding_v5.py` | Multimodal embedding (v5) |
| `plans/plan_post_embedding_v5_for_reply.py` | Same for replies |
| `plans/plan_post_embedding_with_summary.py` | Embedding + summarization |
| `plans/plan_post_embedding_with_summary_for_reply.py` | Same for replies |
| `plans/plan_reply_ranking.py` | Reply ranking scores |
| `plans/plan_safety_ptos.py` | PTOS policy classification |
| `plans/plan_spam_comment.py` | Spam comment detection |

### Tasks

| File | Purpose |
|------|---------|
| `tasks/task.py` | Abstract `Task`, `TaskWithPost`, `TaskWithUser`, etc. |
| `tasks/task_asr.py` | Automatic speech recognition |
| `tasks/task_banger_screen.py` | Banger quality screen |
| `tasks/task_embedding_pub.py` | Publish embedding results |
| `tasks/task_multimodal_post_embedding.py` | Run multimodal model |
| `tasks/task_post_safety_screen_deluxe.py` | Enhanced safety screen |
| `tasks/task_spam_detection.py` | Spam detection |
| `tasks/task_safety_ptos_category.py` | PTOS category classification |
| `tasks/task_safety_ptos_policy.py` | PTOS policy enforcement |
| `tasks/task_summarizer_for_post_embedding.py` | Summarize post for embedding |
| `tasks/task_rank_replies.py` | Rank replies |
| `tasks/task_rate_limit.py` | Rate limiting |
| `tasks/disable_rules.py` | `DisableTaskRule` — pluggable kill-switch mechanism |

### Generators

| File | Purpose |
|------|---------|
| `generators/task_generator.py` | `TaskGenerator`, `PriorityTaskGenerator` |
| `generators/stream_generator.py` | All Kafka stream generators |

---

## Key Data Structures

### `TaskPayload` (Pydantic)

| Field | Type | Notes |
|-------|------|-------|
| `payload_id` | `str` | Unique ID |
| `post` | `Optional[Post]` | The post being processed |
| `user` | `Optional[User]` | Post author |
| `user_context` | `Optional[UserContext]` | Author's context |
| `attempt` | `int` | Retry counter (starts at 0) |
| `eligibilities` | `Set[TaskEligibility]` | Which plans/tasks apply |
| `deadline_ts_secs` | `float` | Task expiry time |
| `task_type` | `TaskGeneratorType` | Which stream this came from |
| `grox_content_analysis` | `Optional[ContentAnalysis]` | Pre-computed analysis (for recovery streams) |

### `TaskResult` (Pydantic)

| Field | Type | Notes |
|-------|------|-------|
| `task` | `TaskPayload` | Input task |
| `task_started_at` | `datetime` | |
| `task_finished_at` | `datetime` | |
| `content_categories` | `List[ContentCategory]` | Aggregated from all plans |
| `multimodal_post_embedding` | `Optional[List[float]]` | Only first plan's embedding kept |
| `reason` | `str` | Human-readable result reason |
| `success` | `bool` | True only if ALL plans succeeded |
| `error` | `Optional[str]` | |

### `TaskEligibility` (enum)

Controls which plans run on a given task:
- `SPAM_COMMENT`, `BANGER_INITIAL_SCREEN`, `POST_EMBEDDING_WITH_SUMMARY`, `MM_EMB_V4`, `MM_EMB_V5`, `MM_EMB_V5_FOR_REPLY`, `REPLY_RANKING`, `SAFETY_PTOS`, `POST_SAFETY`

### `TaskContext`

Mutable context passed through plan tasks. Accumulates:
- `eligibilities`, `content_categories`, `summary`, `embeddings`, `reply_scores`, `safety_annotations`, `errors`

---

## Execution Model

### Process Architecture

```
Main process
  ├── Dispatcher (subprocess)
  │     └── PriorityTaskGenerator → fills task_queue (multiprocessing.Queue)
  └── Engine (subprocess)
        └── asyncio event loop
              └── Per task: asyncio.create_task(PlanMaster.exec(task))
```

- Dispatcher and Engine run in **separate subprocesses**
- Communication via `multiprocessing.Queue` (`task_queue`, `resp_queue`)
- Engine exits via `os._exit(0)` — hard process exit to avoid multiprocessing cleanup issues

### Plan Execution (per task)

```python
# PlanMaster.exec()
results = await asyncio.gather(
    plan_initial_banger.execute(task),
    plan_post_safety.execute(task),
    plan_post_embedding_v5.execute(task),
    # ... all 9 plans
)
merged = merge_results(results)
```

All plans run concurrently for every task. Each plan's `execute()` is expected to be a no-op if the task's `eligibilities` don't include it.

### Retry Logic (two levels)

**Task-level** (`Task.exec()`):
- `@retry(stop=stop_after_attempt(2), wait=wait_fixed(1))`
- 2 attempts, 1-second wait between attempts

**Dispatcher-level**:
- Failed tasks have `attempt` incremented and re-enqueued in `task_queue`
- Up to `max_attempts` total retries
- On final failure: ACKs the Kafka message anyway (prevents infinite replay)

### Back-pressure

`Dispatcher._fill_loop()` only fills `task_queue` when `len(in_flights) < max_in_flight`. Simple in-memory rate limiter.

---

## Task Generator Types

All handled in `Dispatcher`:

| Type | Description |
|------|-------------|
| `POST_STREAM` | Main post stream |
| `POST_STREAM_RECOVERY` | Recovery/replay for missed posts |
| `POST_STREAM_TEST` | Testing stream |
| `POST_STREAM_DELAYED` | Delayed processing stream |
| `POST_SAFETY_STREAM` | Dedicated safety stream |
| `POST_MIN_TRACTION_STREAM_FOR_GROX` | Posts that reached minimum engagement |
| `POST_MIN_TRACTION_STREAM_FOR_GROX_PTOS` | Same, for PTOS classification |
| `POST_MIN_TRACTION_STREAM_FOR_GROX_MULTI_MODAL` | Same, for multimodal embedding |
| `POST_EMBEDDING_V5_STREAM` | V5 embedding stream |
| `POST_EMBEDDING_V5_FOR_REPLY_STREAM` | V5 embedding for replies |
| `REPLY_RANKING_RECOVERY` | Reply ranking recovery |
| `POST_EMBEDDING_REQUEST_STREAM_WITH_SUMMARY` | Embedding + summary |
| `POST_EMBEDDING_REQUEST_STREAM_WITH_SUMMARY_RECOVERY` | Recovery for above |
| `POST_EMBEDDING_REQUEST_STREAM_WITH_SUMMARY_FOR_REPLY_RECOVERY` | Reply variant |
| `SAFETY_PTOS_RECOVERY` | PTOS recovery |
| `SAFETY_PTOS_DELUXE` | Enhanced PTOS stream |

---

## Disable / Skip Mechanism

Two independent skip mechanisms on `Task`:

1. **`DISABLE_RULES` (class-level)** — a list of `DisableTaskRule` instances. If any rule's `should_disable()` returns true, the task is skipped entirely. Controlled by feature flags — acts as a kill-switch.

2. **`_should_skip()` (instance-level)** — task-specific logic. E.g., a post safety task might skip if the post has no text.

Both are checked in `Task.exec()` before executing the task's logic.

---

## Configuration & Parameters

| Parameter | Source | Notes |
|-----------|--------|-------|
| `max_in_flight` | `grox_config.dispatcher` | Back-pressure threshold |
| `max_attempts` | `grox_config.dispatcher` | Max retries per task |
| `graceful_shutdown_timeout` | config | Process join timeout |
| Dispatcher retry wait | `wait_incrementing(start=1, increment=3, max=9)` | On `_init_run()` failure; up to 3 attempts |
| Task retry | `stop_after_attempt(2), wait_fixed(1)` | Task-level retry policy |

---

## Gotchas & Non-Obvious Behaviors

- **All plans run on every task** — `PlanMaster` runs all 9 plans concurrently via `asyncio.gather` regardless of eligibility. Each plan must check eligibility itself and early-exit. This means there is no pre-filtering at the dispatcher level.

- **Only the first multimodal embedding is kept** — `merge_results()` takes the embedding from the first plan that produces one. If two plans both generate embeddings (e.g., `MM_EMB_V5` and `MM_EMB_V5_FOR_REPLY`), only the first is kept.

- **`success` requires ALL plans to succeed** — if any plan fails, `TaskResult.success = False`, even if the safety and embedding plans succeeded. The failure of one plan (e.g., ASR failing on a non-video post) marks the whole task as failed.

- **Dispatcher ACKs on final failure** — when a task exhausts `max_attempts`, the Kafka message is still ACKed (committed). This prevents infinite replay but means some posts may be processed without all annotations.

- **Engine uses `os._exit(0)` not `sys.exit()`** — this is a hard process kill that bypasses Python's cleanup handlers. Intentional to avoid multiprocessing resource leaks.

- **`TaskWithPost` raises `TaskStopExecution` (not an exception)** — when `task.post` is `None`, `TaskWithPost` raises a special control-flow signal that the plan treats as a skip (not an error). This is different from raising a real exception.

- **`PriorityTaskGenerator` uses weighted-random selection** — not round-robin. A higher-weight stream generator will produce tasks more frequently. Weights are set in the dispatcher config.

- **Recovery streams resubmit with pre-computed `grox_content_analysis`** — recovery tasks may carry partial analysis from a previous attempt, allowing plans to skip re-computation of already-completed stages.

---

## Cross-References

- [`00-overview.md`](00-overview.md) — Where Grox fits in the overall system
- [`08-filtering.md`](08-filtering.md) — VFFilter and AncillaryVFFilter consume Grox safety outputs
- [`04-phoenix-retrieval.md`](04-phoenix-retrieval.md) — Phoenix retrieval uses Grox-generated embeddings
