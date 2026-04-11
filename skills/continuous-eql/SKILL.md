---
name: continuous-eql
description: "Continuous EQL — Missionary Signal Graph Architecture. Use when working with reactive pipelines, MissionaryTask/Flow, signal graphs, or debugging reactive data resolution."
---

# Continuous EQL — Missionary Signal Graph Architecture

Reactive signal graph that turns Pathom resolvers into continuously-updating flows. Available as [`pyze/continuous-eql`](https://github.com/pyze/continuous-eql) library (namespace `continuous-eql.core`).

## Three-Sided API

### 1. Resolver API

Standard Pathom resolvers (`::pco/input`, `::pco/output`). Return wrapper types for async/streaming:

- `MissionaryTask` / `mtask` — one-shot async computation
- `MissionaryFlow` / `->MissionaryFlow` — continuous streaming value

**CRITICAL — `map?` gotcha:** `defrecord` implements `ILookup`, so `(map? (->MissionaryTask nil))` => `true`. In `cond` branches, ALWAYS check `instance?` before `map?`.

### 2. Plugin API

Standard `::pcr/wrap-resolve` plugins. `build-signal-graph` applies plugins via `p.plugin/compile-env-extensions`. Internal: `wrap-with-error-isolation` for per-resolver error-to-data propagation.

### 3. Client/Query API

```clojure
;; Static query
(reactive-signal-eql env [:attr1 :attr2])

;; Reactive query (flow of EQL vectors)
(reactive-signal-eql env (m/watch !query-atom))

;; Resolve-once (skip continuous updates)
(reactive-signal-eql env [(list :attr {:once true})])
```

## Public API

| Function | Arities | Purpose |
|----------|---------|---------|
| `reactive-signal-eql` | 1: `[config-flow]`, 2: `[env-or-config-flow eql-query-or-flow-or-opts]`, 3: `[env eql-query-or-flow opts]` | Entry point — returns a Missionary flow of resolved EQL results |
| `build-signal-graph` | `[env execution-order & [{:keys [root-flows instrument prior-inputs branch-metadata]}]]` | Constructs the signal DAG from topo-sorted resolver execution order |
| `walk-execution-graph` | `[env graph]` | Topo-sorts Pathom resolver index into execution order |
| `error-isolation-plugin` | Map with `::p.plugin/id` and `::pcr/wrap-resolve` | Plugin that catches resolver exceptions and converts to `::ceql/error` data |
| `input-has-error?` | `[input]` | Returns truthy when any input value contains `::ceql/error` |
| `first-upstream-error` | `[input]` | Extracts the first `::ceql/error` value from input |
| `dedupe-by-xf` | `[key-fn]` | Transducer for cascade prevention (dedup by projected key) |

## Core Missionary Patterns

### SwitchMap (cancel stale, keep latest)

```clojure
(m/ap
  (let [input (m/?< (m/watch !atom))]
    (try
      (m/? (some-async-task input))
      (catch Cancelled _ (m/amb>)))))
```

- `?<` = continuous fork: cancels previous branch on new value
- `(m/amb>)` = emit nothing in discrete flow (swallow cancellation)

### Glitch-Free Propagation

- `m/cp` with SINGLE `m/watch` + pure derivations = glitch-free
- `m/latest` with separate `m/watch` calls on same atom = GLITCHES
- Architecture: single `m/watch` on state atom -> reads from consistent snapshot

### Async in Flows

- `m/cp` + `m/?` = ERROR (cannot park in continuous flow)
- `m/ap` + `m/?` = OK (can park in discrete flow)

### Platform Rules for `m/?`

- Top-level `m/?` = **JVM-only** (blocks calling thread)
- `m/?` inside `m/sp` = **cross-platform** (macro rewrites to async CPS in CLJS)
- `m/?` inside `m/ap` = **cross-platform** (discrete flow fork)
- `m/?<` inside `m/ap` = **cross-platform** (continuous flow fork / switchMap)
- **SCI caveat:** `m/sp`, `m/ap`, `m/?`, `m/?<` are macros — SCI cannot use them directly. Pre-compile wrappers at CLJS compile time.
- **CLJS task execution:** Use Promise-based `run-task!` instead of top-level `m/?`:
  ```clojure
  (defn run-task! [task] (js/Promise. (fn [resolve reject] (task resolve reject))))
  ```

## 2-Layer Dedup Signal Pattern

**Advanced:** Only needed when non-root resolvers use MissionaryFlow.

**Problem**: Non-root resolvers with `m/?<` switchMap cancel in-flight HTTP requests when upstream emits a duplicate value.

**Solution**: Separate dedup into its own `m/signal` layer BEFORE the resolve `m/ap`:

```clojure
(let [deduped-input (m/signal
                      (m/eduction
                        (if selector (dedupe-by-xf selector) (dedupe))
                        (m/ap (m/?< upstream-combined))))]
  (m/signal
    (m/ap
      (let [input (m/?< deduped-input)]
        (let [result (resolve-fn env input)]
          (if (instance? MissionaryFlow result)
            (try (m/?< (:flow result))
                 (catch Cancelled _ (m/amb)))
            result))))))
```

## Sentinel Seeding Pattern

**Problem**: `m/signal` throws "Uninitialized flow" when MissionaryFlow resolvers have not emitted by the time `m/latest merge` subscribes.

**Solution**: Seed ALL root signals with `::uninitialized` sentinel via `(m/reductions (fn [_ v] v) seed flow)`.

Three layers:
1. **Root signals** seed with sentinel output
2. **Non-root resolvers** propagate sentinel when `(contains-sentinel? input)` — skip resolve-fn entirely
3. **Top-level** filters sentinel emissions

## Error Delivery Architecture (5 layers)

1. **`wrap-with-error-isolation`** — wraps each resolver's resolve-fn; catches exceptions -> `{::ceql/error {:error ex :resolver-name name}}`
2. **`compile-env-extensions`** — applies `::pcr/wrap-resolve` plugin before error isolation
3. **Top-level try/catch** — catches flow creation errors
4. **Consumer-level** — detects `::ceql/error` in results (defense-in-depth)
5. **Error view** — CLJS-only error rendering

Error isolation at resolver level makes layers 3-5 unreachable in practice. They exist as defense-in-depth.

## Signal Graph Construction

Pipeline: `pci/register` indexes -> `walk-execution-graph` (topo-sort) -> `build-signal-graph` -> `reactive-signal-eql`

- Each resolver becomes a `m/signal` node connected via `m/latest`
- Single `m/watch` on state atom -> consistent snapshot (glitch-free)
- Emission lifecycle: 2 emissions then stable (1: pending queries, 2: resolved data)

### `::ceql/input-selector` Pattern

Collect input paths -> sort stably -> project. Eliminates fan-out re-fetches when unrelated state changes:

```clojure
{::pco/input [:broad/input]
 ::pco/output [:narrow/output]
 ::ceql/input-selector (fn [input] (select-keys input [:only :what :matters]))}
```

### Cross-Replan Cache

`build-signal-graph` accepts `:prior-inputs {op-name {:dedup-key V :output V}}`. Non-root resolvers record `:last-dedup-key` and `:last-output` in instrument atom.

## Keyword Portability

The library uses `:continuous-eql.core/*` namespaced keywords internally. Consumers should use exported constant vars instead of `::ceql/*` to avoid namespace mismatch:

| Constant | Value | Usage |
|----------|-------|-------|
| `ceql/error-key` | `:continuous-eql.core/error` | Error data in resolver output |
| `ceql/input-selector-key` | `:continuous-eql.core/input-selector` | Dedup selector on resolver metadata |
| `ceql/output-keys-key` | `:continuous-eql.core/output-keys` | Output key metadata |

**WARNING**: `::ceql/error` in your consumer namespace resolves to a DIFFERENT keyword than the library writes. Always use `ceql/error-key` instead.
