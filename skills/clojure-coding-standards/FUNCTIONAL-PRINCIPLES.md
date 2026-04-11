# Functional Programming Principles

Core functional programming patterns for Clojure code.

## Immutability by Default

**⚠️ CRITICAL: Any use of mutable state without explicit user approval is a VIOLATION of coding standards.**

**Immutable data structures are REQUIRED by default.** ALL mutable state constructs require explicit user approval before introduction. Proceeding without approval = violation.

**Covered constructs (ALL require approval):**
- `atom` / `swap!` / `reset!`
- `volatile!` / `vswap!` / `vreset!`
- `ref` / `dosync` / `alter` / `commute`
- `agent` / `send` / `send-off`
- `defonce` with mutable values (atoms, volatiles, etc.)
- JS interop mutable state (`set!`, `.push`, `.splice`, mutable JS objects)

**Before reaching for mutable state, STOP and ask the user.** See main [SKILL.md](./SKILL.md) for the full approval workflow and alternatives table.

---

## Pure Functions (Strict Enforcement)

**Functions MUST be deterministic and side-effect free unless explicitly marked with `!` suffix.**

```clojure
;; ✅ PURE - Same inputs always produce same output
(defn compute-join [left-tuples right-tuples]
  (for [l left-tuples
        r right-tuples
        :when (= (:entity l) (:entity r))]
    (merge l r)))

;; ❌ IMPURE - Depends on external state (REQUIRES APPROVAL)
(defn compute-join [left-tuples right-tuples]
  (let [config @global-config]  ; External dependency - requires approval!
    ...))

;; ✅ IMPURE BUT MARKED - ! suffix indicates side effects
(defn save-result! [db result]
  (jdbc/insert! db :results result))
```

**Side effects policy:**

- ✅ Push to boundaries (I/O, logging, state updates at edges)
- ✅ Mark with `!` suffix by convention
- ✅ Separate from business logic
- ❌ NEVER hide side effects in pure-looking functions

**Hidden global state detection:** grep for `@global-`, `@app-`, `deref` of module-level vars inside function bodies (not at top level). These are hidden dependencies that should be explicit arguments.

---

## Defaults at the Edges

**Resolve default values at system boundaries so inner functions receive fully-specified data.**

Inner functions should not compute, check for, or apply defaults. When defaults are scattered throughout the call stack, every function must handle "what if this key is missing?" — adding conditional branches, coupling to configuration knowledge, and making functions harder to test.

```clojure
;; ❌ WRONG - Defaults scattered through inner functions
(defn build-chart [config]
  (let [width  (or (:width config) 800)        ; Default here
        height (or (:height config) 600)]       ; Default here
    (render-axes {:width width :height height})))

(defn render-axes [{:keys [width height margin]}]
  (let [margin (or margin 20)]                   ; And here too
    ...))

;; ✅ CORRECT - Defaults resolved once at the edge
(def chart-defaults {:width 800 :height 600 :margin 20})

(defn create-chart [user-config]          ; Edge function
  (let [config (merge chart-defaults user-config)]
    (build-chart config)))                ; Inner functions receive complete data

(defn build-chart [{:keys [width height]}]  ; No defaults needed
  (render-axes {:width width :height height}))

(defn render-axes [{:keys [margin]}]        ; No defaults needed
  ...)
```

**Where to apply defaults:**
- Public API entry points
- Configuration loading / system startup
- HTTP handler parameter parsing
- CLI argument processing

**Why:**
- Inner functions stay pure and simple — no conditional branches for missing values
- Defaults are documented in one place, not discovered by reading the call stack
- Testing inner functions requires no setup for "what if the key is absent?"
- Changing a default is a one-line edit at the edge, not a grep-and-replace

---

## Bang Suffix Convention

**Symbols ending with `!` indicate state-affecting operations.**

| Function | Has Side Effect? | Correct Suffix |
|----------|------------------|----------------|
| `save-to-db!` | Yes (I/O) | Needs `!` |
| `reset-state!` | Yes (mutation) | Needs `!` |
| `validate-token` | No (pure) | NO `!` |
| `calculate-total` | No (pure) | NO `!` |

**Two Contexts for `!` Suffix:**
- **Function names** (this section): `save-to-db!`, `send-email!` - marks functions with side effects
- **Effect keywords** (in event-driven architectures): `:http/post!`, `:store/merge!` - marks effect types

**Enforcement**:
- All functions with side effects MUST use bang
- All pure functions MUST NOT use bang
- Code review should catch violations

**Common miss:** Functions that call `swap!`/`reset!` on atoms they create internally or receive as parameters. The function's *primary* purpose may be "return a flow" or "build a signal map," but if it mutates any atom, it has a side effect and needs `!`. Public API functions without `!` are **high priority** — callers assume no-bang functions are pure.

```clojure
;; ❌ WRONG — mutates caller's atom but name suggests pure
(defn build-signal-graph [env order {:keys [instrument]}]
  ;; ... builds sig-map ...
  (reset! instrument counters)  ;; side effect!
  sig-map)

;; ✅ CORRECT — bang signals the mutation
(defn build-signal-graph! [env order {:keys [instrument]}]
  ;; ... builds sig-map ...
  (reset! instrument counters)
  sig-map)
```

---

## State-Based Over Timing-Based

**CRITICAL: Always prefer state-based approaches over timing-based solutions.**

When solving async/event problems, use existing application state instead of timestamps, delays, or debouncing.

```clojure
;; WRONG - Timing-based (timestamp comparison)
(defn check-should-load [db {:keys [near-top?]}]
  (let [now (js/Date.now)
        last-nav (:navigation-timestamp db)
        just-navigated? (< (- now last-nav) 200)]  ; Magic number!
    (when (and near-top? (not just-navigated?))
      [[:load-more-messages]])))

;; RIGHT - State-based (semantic state flag)
(defn check-should-load [db {:keys [near-top?]}]
  (let [scroll-mode (:scroll-mode db)]
    ;; Only load when user has manually scrolled
    (when (and near-top? (= scroll-mode :manual))
      [[:load-more-messages]])))
```

**Decision framework**:
1. Identify the trigger (scroll, click, navigation)
2. Find existing state that represents this condition
3. Use or add semantic state over timing mechanisms
4. Avoid: timestamps, delays, debouncing

**Exceptions** (require explicit approval):
- Performance throttling for high-frequency events
- External API rate limiting
- Animations/transitions (inherently time-based)

**Anti-patterns to avoid:**

| Pattern | Problem | Better Approach |
|---------|---------|-----------------|
| `setTimeout` / `js/setTimeout` | Masks race condition with arbitrary delay | Use state flag, callback, or watch |
| `requestAnimationFrame` for state sync | Papers over missing render cycle | Ensure state update triggers re-render |
| Timer-based debounce for dedup | Unreliable under varying load | Content-hash comparison or idempotent operations |
| `Thread/sleep` in watchers | Blocks thread, misses rapid changes | Content-hash dedup with immediate processing |

The use of `setTimeout` or `requestAnimationFrame` for state synchronization usually indicates papering over a deeper bug — a missing state transition, an event not being dispatched, or a render cycle not being triggered. Fix the root cause instead.

---

## Data Representation

**Keyword/symbol to string conversion** is a code smell. If you're calling `(name :foo)` or `(str :foo)` to build a string, you're likely working around a boundary that should accept keywords directly. Keywords are interned, fast to compare, and self-documenting. Strings are opaque. Keep data as keywords throughout; convert only at system boundaries (JSON serialization, HTTP headers, external APIs).

---

## Composition Over Inheritance

**Build functionality through function composition, not class hierarchies.**

```clojure
;; ✅ COMPOSITION - Combine simple functions
(def process-query
  (comp materialize-results
        apply-projection
        execute-plan
        optimize-plan
        parse-query))

;; ✅ PROTOCOLS - Define behavior contracts
(defprotocol Operator
  (push [this delta])
  (pull [this]))

;; Compose operators through delegation
(deftype JoinOp [left right]
  Operator
  (push [this delta]
    (let [left-result (push left delta)
          right-result (push right delta)]
      (merge-results left-result right-result))))
```

---

## Declarative Over Imperative

**Express WHAT to compute, not HOW to compute it.**

```clojure
;; ✅ DECLARATIVE - Express intent with transducers
(defn entities-with-age-over [db min-age]
  (into []
        (comp (filter #(> (:age %) min-age))
              (map :name))
        db))

;; ❌ IMPERATIVE - Focus on mechanics
(defn entities-with-age-over [db min-age]
  (let [results (atom [])]
    (doseq [entity db]
      (when (> (:age entity) min-age)
        (swap! results conj (:name entity))))
    @results))
```

**ALWAYS prefer transducers over `->>` threading for collection transformations.** Transducers are more efficient (single pass, no intermediate collections) and compose better. See [COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md) for comprehensive transducer patterns.

---

## Lazy Evaluation

**Use lazy sequences for large or infinite data.** Avoid realizing entire sequences unnecessarily.

```clojure
;; ✅ LAZY - Process incrementally
(defn process-large-result [query-result]
  (->> query-result
       (map transform-tuple)     ; Lazy
       (filter valid?)           ; Lazy
       (take 100)))              ; Only realizes 100

;; ❌ EAGER - Realizes entire sequence
(defn process-large-result [query-result]
  (let [all-transformed (doall (map transform-tuple query-result))]  ; Realizes all!
    (take 100 (filter valid? all-transformed))))
```

**When to force realization:**
- Small collections (< 1000 elements)
- Need side effects (logging, I/O)
- Performance testing (avoid lazy seq overhead in benchmarks)

**Note:** For in-memory collections, prefer transducers (see COLLECTION-PATTERNS.md) over lazy sequences.

---

## Higher-Order Functions

**Functions that take or return functions enable powerful abstractions.**

```clojure
;; ✅ HIGHER-ORDER - Generic transformation
(defn map-zset [f zset]
  (reduce-kv
    (fn [acc tuple mult]
      (assoc acc (f tuple) mult))
    {}
    zset))

;; Use it with different functions
(map-zset project-to-names zset)
(map-zset (fn [[e a v]] [v a e]) zset)  ; Reverse tuple
```

---

## Avoid `with-redefs` — Even in Tests

`with-redefs` is a code smell even in test code. If you need `with-redefs` to test something, it means the production code has a hidden dependency that should be an explicit parameter.

```clojure
;; BAD — production code with hidden dependency
(defn fetch-user [id]
  (db/query :users {:id id}))  ;; hardcoded db call

;; BAD — test uses with-redefs to work around it
(deftest test-fetch-user
  (with-redefs [db/query (fn [& _] {:id 1 :name "test"})]
    (is (= "test" (:name (fetch-user 1))))))

;; GOOD — production code takes dependency explicitly
(defn fetch-user [db id]
  (db/query db :users {:id id}))

;; GOOD — test passes a mock directly
(deftest test-fetch-user
  (let [mock-db (reify IDB (query [_ table pred] {:id 1 :name "test"}))]
    (is (= "test" (:name (fetch-user mock-db 1))))))
```

**When you encounter `with-redefs`**: don't just use it — fix the composition model. Make the dependency explicit (as a function argument, protocol, or Integrant component). This makes the code both more testable and more decomplected.

**No acceptable uses in production code.** In test code, treat it as a temporary workaround that should be replaced with proper dependency injection.

### `alter-var-root` is the same smell

`alter-var-root` mutates a var globally — it's `with-redefs` without the scoping. If you're using it to configure behavior at startup, use Integrant components or explicit configuration maps instead. If you're using it to swap implementations, use protocols.

```clojure
;; BAD — global mutation to configure behavior
(alter-var-root #'db-pool (constantly (create-pool config)))

;; GOOD — Integrant component with explicit lifecycle
(defmethod ig/init-key ::db-pool [_ config]
  (create-pool config))
```

---

## Pattern Matching & Destructuring

**Use destructuring to extract data declaratively.**

**Prefer point-free style** except when it duplicates function calls or harms readability:

```clojure
;; ✅ POINT-FREE - Clear composition
(def process-users
  (comp (partial map :name)
        (partial filter :active?)))

(process-users users)

;; ✅ EXPLICIT - Point-free would duplicate get-data call
(defn analyze-data []
  (let [data (get-data)]  ; Called once
    (merge (summarize data)
           (validate data))))

;; ❌ POINT-FREE - Would call get-data twice
(def analyze-data
  (juxt (comp summarize get-data)
        (comp validate get-data)))  ; get-data called twice!
```

**Destructuring examples:**

```clojure
;; ✅ DESTRUCTURING - Clear and concise
(defn join-tuples [[e1 a1 v1] [e2 a2 v2]]
  (when (= e1 e2)
    [e1 a1 v1 a2 v2]))

;; ✅ MAP DESTRUCTURING
(defn execute-query [{:keys [find where order-by limit]}]
  ...)

;; ❌ MANUAL ACCESS - Verbose and error-prone
(defn join-tuples [t1 t2]
  (let [e1 (nth t1 0)
        a1 (nth t1 1)
        v1 (nth t1 2)
        e2 (nth t2 0)]
    ...))
```

---

## Error Handling - Preserve Exception Context

**CRITICAL: Never transmute exceptions except at cross-process API boundaries.**

**Core principles:**

```clojure
;; ✅ PRESERVE CONTEXT - Wrap and add information
(try
  (dangerous-operation)
  (catch Exception e
    (throw (ex-info "Failed to process zip file"
                    {:file-id   file-id
                     :operation :decrypt}
                    e))))  ; Original exception preserved as cause

;; ✅ TRANSFORM AT BOUNDARY ONLY - REST endpoint
(defn rest-endpoint [request]
  (try
    (process-request request)
    (catch Exception e
      {:status 500
       :body   {:error (.getMessage e)}})))  ; OK - API boundary

;; ❌ NEVER TRANSMUTE IN CORE LOGIC
(defn get-entry-names [zip-file]
  (try
    (process-entries zip-file)
    (catch Exception e
      {:error {:type :failed}})))  ; WRONG - loses stack trace!

;; ✅ LET EXCEPTIONS PROPAGATE OR WRAP
(defn get-entry-names [zip-file]
  (process-entries zip-file))  ; Propagate, or wrap with ex-info if adding context
```

**Exception handling rules:**

1. **Default**: Let exceptions propagate
2. **Add context**: Use `ex-info` to wrap with additional data
3. **Preserve chain**: Always pass original exception as third arg to `ex-info`
4. **Transform ONLY at boundaries**: HTTP, CLI, RPC interfaces

**Prohibited patterns:**

```clojure
;; ❌ PROHIBITED - Catch and return error map
(try
  (operation)
  (catch Exception e
    {:error "failed"}))

;; ❌ PROHIBITED - Transmute exception type
(try
  (operation)
  (catch IOException e
    (throw (IllegalArgumentException. "Bad input"))))  ; Lost original cause!

;; ❌ PROHIBITED - Silent failure
(try
  (operation)
  (catch Exception e
    nil))  ; Information lost!
```

---

## Polymorphism: Protocols Over instance? Checks

**Use protocols for type-based dispatch, not `instance?` checks.**

The `instance?` anti-pattern creates brittle code that's hard to refactor:

```clojure
;; ❌ ANTI-PATTERN - Manual type dispatch with instance?
(cond
  (instance? ArrowZSet x) (handle-arrow x)
  (instance? WideSchemaArrowZSet x) (handle-wide x)
  (instance? FilteredView x) (handle-filtered x))
```

**Problems with `instance?`:**
1. **Namespace coupling** - Hardcodes fully-qualified class names
2. **Violates Open-Closed** - Adding new types requires modifying all dispatch sites
3. **Blocks refactoring** - Moving `defrecord` to new namespace breaks all checks
4. **Not idiomatic** - Clojure has protocols for this

**Solution: Protocol-based dispatch:**

```clojure
;; ✅ CORRECT - Define protocol for polymorphic operations
(defprotocol IZSetOps
  (row-count [this])
  (to-seq [this])
  (close! [this]))

;; Each type implements the protocol
(extend-type ArrowZSet
  IZSetOps
  (row-count [this] (.getRowCount (:root this)))
  (to-seq [this] (datom-seq this))
  (close! [this] (.close (:root this))))

(extend-type WideSchemaArrowZSet
  IZSetOps
  (row-count [this] ...)
  ...)

;; ✅ CORRECT - Usage with automatic dispatch
(row-count x)  ; Works for any type implementing IZSetOps

;; ✅ CORRECT - Type predicate using protocol
(defn zset? [x] (satisfies? IZSetOps x))
```

**When you need type-specific behavior:**
1. Add a method to an existing protocol, OR
2. Create a new protocol for the specific capability
3. Use `satisfies?` for type predicates, not `instance?`

**Exception:** `instance?` is acceptable for:
- Interop with Java classes you don't control
- Performance-critical inner loops (measure first!)
- Debugging/logging (not control flow)

---

## Data-Oriented Design

**Represent domain concepts as plain data structures, not objects.**

```clojure
;; ✅ DATA-ORIENTED - Plain maps and vectors
(def query
  {:find '[?name ?age]
   :where '[[?e :person/name ?name]
            [?e :person/age ?age]]
   :order-by '[:age :desc]})

;; Process with generic functions
(defn validate-query [query]
  (and (vector? (:find query))
       (vector? (:where query))))

;; ❌ OBJECT-ORIENTED - Hidden data, methods
(defrecord Query [find where order-by]
  IValidatable
  (validate [this] ...))
```

**Benefits:**
- Generic transformation functions (map, filter, reduce)
- Easy serialization and debugging
- Clear separation of data and behavior
- REPL-friendly exploration

---

## Summary

1. **Immutability**: Required by default, mutation needs approval
2. **Purity**: Pure by default, mark side effects with `!`
3. **Defaults at edges**: Resolve defaults at system boundaries, not in inner functions
4. **Composition**: Build with functions and protocols, not inheritance
5. **Declarative**: Express intent with ->, ->>, map, filter, reduce (or transducers)
6. **Point-free**: Use when clear, avoid when duplicating calls
7. **Higher-Order**: Abstract patterns with functions that take/return functions
8. **Destructuring**: Extract data declaratively
9. **Preserve exceptions**: Never transmute except at API boundaries
10. **Wrap, don't replace**: Use ex-info to add context while preserving cause
11. **Data-Oriented**: Plain maps/vectors over objects with methods

**See also:**
- [COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md) - Transducer patterns for collections
- [CODE-ORGANIZATION.md](./CODE-ORGANIZATION.md) - Function/namespace size guidelines
- Main [SKILL.md](./SKILL.md) - Unified approval policies
