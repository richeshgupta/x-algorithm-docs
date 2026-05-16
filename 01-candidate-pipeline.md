# Candidate Pipeline — Framework

> **TL;DR**: `candidate-pipeline/` is a pure Rust trait library that defines the abstract stages of a recommendation pipeline. It contains zero business logic — only the orchestration skeleton. Every concrete pipeline (ForYou, Phoenix, etc.) implements these traits and gets parallel execution, error handling, and stats tracking for free.

---

## Role in Pipeline

This is the **foundation layer**. Both `ForYouCandidatePipeline` and `PhoenixCandidatePipeline` in `home-mixer/` are built on top of this framework. It dictates the execution order, parallelism model, and result structure.

```
candidate-pipeline (traits + orchestration)
     ▲
     │ implements
home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs
home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs  (implied)
```

---

## Files

| File | Purpose |
|------|---------|
| `candidate_pipeline.rs` | Core `CandidatePipeline` trait and `execute()` orchestration |
| `filter.rs` | `Filter` trait and `FilterResult<C>` |
| `hydrator.rs` | `Hydrator`, `CachedHydrator`, `CacheStore` traits |
| `query_hydrator.rs` | `QueryHydrator` trait — enriches query before candidate fetch |
| `scorer.rs` | `Scorer` trait — async, sequential |
| `selector.rs` | `Selector` trait — sort + truncate |
| `side_effect.rs` | `SideEffect` trait — fire-and-forget post-pipeline actions |
| `source.rs` | `Source` trait — fetches raw candidates |
| `util.rs` | `short_type_name()` helper |
| `lib.rs` | Module re-exports only |

---

## Key Data Structures

| Type | Definition | Purpose |
|------|-----------|---------|
| `PipelineResult<Q, C>` | `candidate_pipeline.rs` | Final output: `retrieved_candidates: Vec<C>`, `filtered_candidates: Vec<C>`, `selected_candidates: Vec<C>`, `query: Arc<Q>` — only `query` is `Arc`-wrapped |
| `FilterResult<C>` | `filter.rs` | `kept: Vec<C>` + `removed: Vec<C>` |
| `SelectResult<C>` | `selector.rs` | `selected: Vec<C>` + `non_selected: Vec<C>` |
| `SideEffectInput<Q, C>` | `side_effect.rs` | `query: Arc<Q>`, `selected_candidates: Vec<C>`, `non_selected_candidates: Vec<C>` |
| `PipelineComponents` | `candidate_pipeline.rs` | Stage label + list of component names (for introspection/metrics) |
| `PipelineStage` (enum) | `candidate_pipeline.rs` | QueryHydrator, DependentQueryHydrator, Source, Hydrator, PostSelectionHydrator, Filter, PostSelectionFilter, Scorer, Selector, SideEffect |

---

## Traits

### `PipelineQuery`
```rust
// candidate_pipeline.rs
trait PipelineQuery {
    fn params(&self) -> &Params;           // feature switches
    fn decider(&self) -> Option<&Decider>; // A/B experiment decider
}
```

### `PipelineCandidate`
Blanket impl — any type that is `Clone + Send + Sync + 'static`.

### `CandidatePipeline<Q, C>`
The main trait. Implementors define their component lists; `execute()` is a provided default impl.

```rust
trait CandidatePipeline<Q: PipelineQuery, C: PipelineCandidate> {
    fn query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>];
    fn dependent_query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>];
    fn sources(&self) -> &[Box<dyn Source<Q, C>>];
    fn hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
    fn filters(&self) -> &[Box<dyn Filter<Q, C>>];
    fn scorers(&self) -> &[Box<dyn Scorer<Q, C>>];
    fn selector(&self) -> &dyn Selector<Q, C>;
    fn post_selection_hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
    fn post_selection_filters(&self) -> &[Box<dyn Filter<Q, C>>];
    fn side_effects(&self) -> Arc<Vec<Box<dyn SideEffect<Q, C>>>>;
    fn result_size(&self) -> usize;              // top-K limit (always required)
    fn finalize(&self, query: &Q, candidates: &mut Vec<C>) {}  // optional override, mutates in place
    async fn execute(&self, query: Q) -> PipelineResult<Q, C>;
}
```

### `Filter<Q, C>`
```rust
trait Filter<Q, C> {
    fn filter(&self, query: &Q, candidates: Vec<C>) -> FilterResult<C>;  // synchronous
}
```
Stats (`KEPT_SCOPE`, `REMOVED_SCOPE`) tracked automatically per filter name.

### `Hydrator<Q, C>`
```rust
trait Hydrator<Q, C> {
    async fn hydrate(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>>;
}
```
**Contract**: returned vector length MUST equal input length. Violated contract → entire stage silently skipped.

### `CachedHydrator<Q, C>`
Extends `Hydrator` with a `CacheStore`. Blanket impl: any `CachedHydrator` automatically implements `Hydrator`.

### `QueryHydrator<Q>`
```rust
trait QueryHydrator<Q> {
    async fn hydrate(&self, query: &Q) -> Result<Q, String>;
}
```
Results merged back via `update()`. Dependent hydrators can read values set by initial hydrators.

### `Scorer<Q, C>`
```rust
trait Scorer<Q, C> {
    async fn score(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>>;
}
```
**Contract**: same length requirement as `Hydrator`. Scorers run **sequentially** (not parallel).

### `Selector<Q, C>`
```rust
trait Selector<Q, C> {
    fn select(&self, query: &Q, candidates: Vec<C>) -> SelectResult<C>;
    fn score(&self, candidate: &C) -> f64;  // for default sort
    fn size(&self) -> Option<usize>;        // optional truncation (no query param)
}
```
Default impl sorts by `score()` descending, then truncates at `size()`.

### `SideEffect<Q, C>`
```rust
trait SideEffect<Q, C> {
    async fn side_effect(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String>;
}
```

### `Source<Q, C>`
```rust
trait Source<Q, C> {
    async fn source(&self, query: &Q) -> Result<Vec<C>, String>;
}
```

---

## Execution Order (Fixed)

```
execute(query)
  │
  ├─ 1. QueryHydrators         (ALL parallel via join_all)
  ├─ 2. DependentQueryHydrators (ALL parallel via join_all, after step 1)
  ├─ 3. Sources                (ALL parallel)
  ├─ 4. Hydrators              (ALL parallel)
  ├─ 5. Filters                (sequential)
  ├─ 6. Scorers                (sequential)
  ├─ 7. Selector               (single)
  ├─ 8. PostSelectionHydrators (ALL parallel)
  ├─ 9. PostSelectionFilters   (sequential)
  ├─ 10. finalize(query)
  └─ 11. SideEffects           (tokio::spawn — fire-and-forget)
```

**Critical ordering notes:**
- QueryHydrators, DependentQueryHydrators, Sources, Hydrators, and PostSelectionHydrators all run in parallel via `join_all`
- Filters, Scorers, and PostSelectionFilters run in list order (sequence matters)
- Side effects are spawned after the response is built — they do NOT block the response

---

## Latency Buckets (per stage)

| Stage | Bucket |
|-------|--------|
| Pipeline execute, Source | `Bucket500To2500` ms |
| Filter, Selector | `Bucket0To50` ms |
| Hydrator | `Bucket50To500` ms |

---

## Configuration & Parameters

| Constant | File | Effect |
|----------|------|--------|
| `FINAL_RESULT_SIZE_SCOPE` | `candidate_pipeline.rs` | Stat bucket label for selected count |
| `FINAL_RESULT_EMPTY_SCOPE` | `candidate_pipeline.rs` | Stat bucket when selection is empty |
| `KEPT_SCOPE` | `filter.rs` | Stat bucket label for kept candidates |
| `REMOVED_SCOPE` | `filter.rs` | Stat bucket label for removed candidates |
| `CACHE_HIT_SCOPE` | `hydrator.rs` | Stat bucket for cache hits |
| `CACHE_MISS_SCOPE` | `hydrator.rs` | Stat bucket for cache misses |

---

## Gotchas & Non-Obvious Behaviors

- **Silent skip on length violation**: If a `Hydrator` or `Scorer` returns a vector of different length than its input, the entire stage output is discarded and an error is logged. No panic, no exception — silent data loss. Always verify length preservation when implementing these traits.

- **Side effects receive non-selected candidates too**: `SideEffectInput` includes both `selected_candidates` AND `non_selected_candidates` (candidates beyond `result_size()`). This is intentional — side effects like serving history and Kafka logging need to know about rejected candidates too.

- **`DependentQueryHydrators` run after initial hydrators**: They can read values set in step 1. This enables two-phase query enrichment (e.g., hydrate user ID → then use it to hydrate feature store values).

- **`CachedHydrator` is a blanket impl**: Implementing `CachedHydrator<Q,C>` automatically satisfies `Hydrator<Q,C>` — no need to implement both separately.

- **`finalize()` is a no-op by default**: Concrete pipelines may override it to apply post-processing to the query after selection (e.g., update timestamps).

- **`result_size()` returns `usize`** (not `Option`) — a concrete limit is always required. Candidates beyond the limit become `non_selected` (passed to side effects) rather than being discarded.

- **Filters vs PostSelectionFilters**: Filters run before scoring (cheaper to reject early). PostSelectionFilters run after the selector on the top-K result — typically for expensive or freshness-sensitive checks (e.g., VF/visibility filter for deleted posts).

---

## Cross-References

- [`00-overview.md`](00-overview.md) — System context
- [`02-home-mixer.md`](02-home-mixer.md) — Concrete pipeline implementations
