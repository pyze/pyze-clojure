# Code Organization Guidelines

Function and namespace size guidelines for maintainability, testability, and readability.

## Function Size Guidelines

**Recommended LOC ranges:**

- **Ideal: 10-40 LOC** - Pure functions with single responsibility
- **Acceptable: 40-80 LOC** - Complex logic with clear structure, multi-arity functions
- **Review Needed: 80-150 LOC** - Candidates for extraction and refactoring
- **Red Flag: 150+ LOC** - Almost always should be decomposed into smaller functions

**Why these limits matter:**

```
50 LOC   → Fits one screen, easy to hold context
100 LOC  → Two screens, requires scrolling, harder to reason about
150+ LOC → Difficult to track all branches and dependencies
```

### Clojure-Specific Size Factors

**Multi-arity functions** add 10-20 LOC but are acceptable if total < 80:

```clojure
;; ✅ ACCEPTABLE - Multiple arities, total ~60 LOC
(defn make-join
  ([keys]
   (make-join keys :inner))

  ([keys join-type]
   (make-join keys join-type nil))

  ([keys join-type callback]
   ;; ~50 LOC actual implementation
   (let [combine-fn ...]
     ...)))
```

**Let-binding depth** affects readability more than LOC count:

```clojure
;; ❌ HARD TO FOLLOW - Nested let bindings bury logic
(defn compile-circuit [operators]
  (let [optimized (optimize operators)]
    (let [operators (compile-operators optimized)]
      (let [circuit (chain operators)]
        ...))))

;; ✅ CLEARER - Extract helpers to reduce nesting
(defn compile-circuit [operators]
  (let [optimized (optimize operators)]
    (-> optimized
        compile-operators
        chain
        finalize)))
```

**High-arity functions** (5+ parameters) reduce clarity:

```clojure
;; ❌ Too many parameters - hard to remember order
(defn query-entities [storage attr min-val max-val limit offset]
  ...)

;; ✅ CLEARER - Use maps for options
(defn query-entities [storage attr {:keys [min max limit offset]}]
  ...)

;; ✅ CLEARER - Use records for related params
(defn query-entities [storage {:keys [attr min max limit offset]}]
  ...)
```

---

## Namespace Size Guidelines

**Recommended LOC ranges:**

- **Ideal: 200-400 LOC** - Cohesive module with 5-15 functions
- **Acceptable: 400-1000 LOC** - Related functionality, single domain concern
- **Review Needed: 1000-1500 LOC** - Candidates for namespace splitting
- **Red Flag: 1500+ LOC** - Likely combines multiple concerns, should be split

**Codebase metrics (reactive-datascript):**

```
✅ HEALTHY:
arrow_types.clj         ~260 LOC   (8 functions)
operator_abstractions   ~717 LOC   (4 protocol implementations)
join.clj                ~813 LOC   (13 functions)

⚠️ OVER THRESHOLD:
arrow_zset.clj          3381 LOC   (52 functions) → Split recommended
compiler.clj            1606 LOC   (17 functions) → Split recommended
window_operators.clj    1449 LOC   (multiple join types) → Split recommended
```

### When to Split a Namespace

**Split when ANY of these are true:**

1. ✗ File has 1500+ LOC
2. ✗ Functions solve 3+ distinct problems (e.g., compilation + circuit building)
3. ✗ Some functions are only used by 1-2 other functions in the module
4. ✗ High-level API mixed with low-level implementation details
5. ✗ Can't describe module purpose in one sentence

**Example: `compiler.clj` candidates for split:**

```clojure
;; SPLIT 1: compiler.clj (~600 LOC)
;; Purpose: High-level query compilation
(defn compile-circuit [...])
(defn optimize-operators [...])

;; SPLIT 2: compiler_operators.clj (~500 LOC)
;; Purpose: Individual operator compilation
(defn compile-join [...])
(defn compile-filter [...])
(defn compile-scan [...])

;; SPLIT 3: compiler_optimizations.clj (~500 LOC)
;; Purpose: Optimization passes
(defn optimize-filter-pushdown [...])
(defn optimize-join-ordering [...])
```

### Minimize Use of `(declare ...)`

`(declare ...)` enables forward references, allowing functions to be called before they're defined. Minimize its use:

- **`declare` is a code smell** — it usually indicates circular dependencies within the namespace or poor top-to-bottom ordering. Both are signs that the namespace is trying to do too much.
- **Prefer reordering** — Clojure namespaces read top to bottom. Move helper functions above their callers. If you can't find a clean ordering, the functions may belong in separate namespaces.
- **Prefer extracting** — if two functions are mutually recursive, consider whether they're actually one concept that should be a single function, or whether they belong in a shared namespace with a protocol.

```clojure
;; BAD — declare to work around ordering
(declare process-children)
(defn process-node [node]
  (map process-children (:children node)))
(defn process-children [children]
  (map process-node children))

;; BETTER — single function with recursion
(defn process-tree [node]
  (when node
    (map process-tree (:children node))))
```

**Acceptable uses**: mutual recursion that genuinely can't be collapsed (rare), and REPL development where you're sketching before reordering.

---

## Refactoring Triggers

**Extract a helper function when:**

✗ Function has 2+ independent loops or recursions
```clojure
;; ❌ TWO SEPARATE JOBS
(defn process-data [data]
  (loop [result [] items data]          ; Job 1: Filtering
    (if (empty? items)
      (loop [acc {} r result]           ; Job 2: Grouping
        ...))))

;; ✅ EXTRACTED HELPERS
(defn process-data [data]
  (let [filtered (filter-by-criteria data)]
    (group-by-key filtered)))
```

✗ Function has 3+ nested let-bindings
```clojure
;; ❌ NESTED LETS - Hard to follow
(defn join-tables [table-a table-b]
  (let [index-a (index-by-key table-a)]
    (let [index-b (index-by-key table-b)]
      (let [matching (find-matches index-a index-b)]
        ...))))

;; ✅ HELPER FUNCTIONS
(defn join-tables [table-a table-b]
  (let [index-a (index-by-key table-a)
        index-b (index-by-key table-b)
        matching (find-matches index-a index-b)]
    ...))

;; OR USE THREADING
(defn join-tables [table-a table-b]
  (-> [table-a table-b]
      (index-tables)
      (find-matches)
      (build-result)))
```

✗ Function has 3+ major conditional branches with different domains
```clojure
;; ❌ MULTIPLE CONCERNS - Query parsing + execution + formatting
(defn execute-query [query]
  (cond
    ;; Concern 1: Query validation
    (invalid-query? query) (throw (ex-info "Invalid" {}))

    ;; Concern 2: Choose execution strategy
    (join-query? query) (execute-join query)
    (scan-query? query) (execute-scan query)

    ;; Concern 3: Format result
    :else (format-result result)))

;; ✅ SEPARATE FUNCTIONS
(defn execute-query [query]
  (validate-query query)
  (let [result (choose-executor query)]
    (format-result result)))
```

✗ Function uses variable names 2+ times (indicates scope creep)
```clojure
;; ❌ Variables reused - suggests multiple jobs
(defn transform [data]
  (let [data (filter-data data)]      ; data reassigned
    (let [data (map-data data)]        ; data reassigned again
      (let [data (sort-data data)]     ; data reassigned third time
        data))))

;; ✅ CLEAR DATA FLOW
(defn transform [data]
  (-> data
      filter-data
      map-data
      sort-data))
```

✗ Function performs both business logic and error handling
```clojure
;; ❌ MIXED CONCERNS - Logic + error handling interleaved
(defn process-items [items]
  (try
    (let [validated (validate items)]
      (let [processed (transform validated)]
        (save-result processed)))))

;; ✅ SEPARATED - Helper handles validation
(defn process-items [items]
  (-> items
      validate-items
      transform-items
      save-result))
```

---

## Function Complexity Metrics

**Red flags for overly complex functions:**

| Metric | Yellow Flag | Red Flag | Action |
|--------|------------|----------|--------|
| **Lines** | 80-120 | 150+ | Extract helpers |
| **Nesting depth** | 3+ levels | 5+ levels | Reduce with threading macros |
| **Parameters** | 4-5 | 6+ | Use maps/records |
| **Cyclomatic complexity** | 5-8 paths | 10+ paths | Split into sub-functions |
| **Let bindings** | 5-7 | 10+ | Extract to helpers or use threading |
| **Cond branches** | 4-6 | 8+ | Use multimethod or separate functions |

**Count nesting depth:**

```clojure
;; DEPTH 1: Top level
(defn f []
  ;; DEPTH 2: In let
  (let [x ...]
    ;; DEPTH 3: In nested let
    (let [y ...]
      ;; DEPTH 4: In map function
      (map (fn [item]
             ;; DEPTH 5: In map function body - PROBLEMATIC
             ...)
           items))))

;; DEPTH ANALYSIS:
;; Depth 2: Acceptable - let binding
;; Depth 3: Acceptable - nested let
;; Depth 4: Acceptable - map function
;; Depth 5: Red flag - too nested, extract helper
```

**Cyclomatic complexity:**

```clojure
;; COMPLEXITY 1
(defn simple [x]
  (+ x 1))

;; COMPLEXITY 3 - if/else adds 2 paths
(defn medium [x]
  (if (> x 0)
    (+ x 1)
    (- x 1)))

;; COMPLEXITY 5 - two if statements = 4 paths
(defn complex [x y]
  (if (> x 0)
    (if (> y 0)
      x
      y)
    0))

;; RED FLAG: Complexity 8+, consider splitting
(defn overly-complex [x y z]
  (cond
    (and (> x 0) (> y 0)) ...      ; Path 1
    (and (> x 0) (< y 0)) ...      ; Path 2
    (and (< x 0) (> y 0)) ...      ; Path 3
    (and (< x 0) (< y 0)) ...      ; Path 4
    (zero? z) ...                   ; Path 5
    :else ...))                     ; Path 6
```

---

## Decision Tree - When to Refactor

```
┌────────────────────────────────────────┐
│ Is function > 80 LOC?                 │
└────────────────────────────────────────┘
         │
         ├─ YES → Can I extract 2-3 helpers?
         │         │
         │         ├─ YES → Extract and test
         │         │
         │         └─ NO → Document why complex
         │                 Ask user if refactoring needed
         │
         └─ NO → OK, commit

┌────────────────────────────────────────┐
│ Is namespace > 1500 LOC?              │
└────────────────────────────────────────┘
         │
         ├─ YES → Does it solve 2+ problems?
         │         │
         │         ├─ YES → Split namespace
         │         │
         │         └─ NO → OK, if tightly related
         │
         └─ NO → OK, commit

┌────────────────────────────────────────┐
│ Does function have 3+ nested let?     │
└────────────────────────────────────────┘
         │
         └─ YES → Can I use threading macro?
                   │
                   ├─ YES → Use -> or ->> to flatten
                   │
                   └─ NO → Extract helper functions
```

---

## Tools & Metrics

**Automated checks (clj-kondo):**

```bash
# Check for unused functions
clj-kondo --lint src/ | grep "unused"

# Check for reflection warnings (impacts performance)
clj-kondo --lint src/ | grep "reflection"

# Count functions in file
grep -c "^(defn " src/mymodule.clj
```

**Manual metrics:**

```bash
# File size analysis
wc -l src/**/*.clj | sort -n | tail -20

# Functions per file
for f in src/**/*.clj; do
  count=$(grep -c "^(defn " "$f")
  echo "$(basename $f): $count functions"
done

# Average LOC per function
# Manual: (total LOC) / (function count)
# Example: arrow_zset.clj = 3381 / 52 = 65 LOC/function average
```

---

## Summary

1. **Function size**: 10-40 LOC ideal, 40-80 acceptable, 80-150 review, 150+ red flag
2. **Namespace size**: 200-400 ideal, 400-1000 acceptable, 1000-1500 review, 1500+ split
3. **Multi-arity**: OK if total < 80 LOC
4. **Let-binding depth**: Extract helpers if 3+ nested levels
5. **Parameters**: Use maps/records if 5+
6. **Complexity**: Refactor if 3+ independent jobs or 3+ nested controls
7. **Variables**: Avoid reassigning same variable (reuse suggests multiple jobs)
8. **Nesting**: Extract helpers for depth > 4
9. **Reduce nesting to reduce edit risk** — deeply nested s-expressions are fragile under non-structural edits
10. **Split namespaces** when they solve multiple problems or exceed 1500 LOC
11. **Use threading macros** to reduce nesting and improve readability
12. **Minimize reader conditionals** — isolate platform code in dedicated files, keep `.cljc` pure

---

## Reduce Nesting to Reduce Edit Risk

**As s-expressions nest deeper, non-structural edits become increasingly error-prone.**

Text-based tools (sed, awk, the Edit tool, manual regex) operate on lines and character positions, not s-expression boundaries. At nesting depth 2-3, a misplaced paren is easy to spot. At depth 5+, matching parens span dozens of lines and a single deletion or insertion can silently unbalance the entire form.

```clojure
;; ❌ HIGH RISK — depth 6, fragile under text edits
(defn process [env query]
  (let [plan (compute-plan env query)]
    (when-let [nodes (seq (:nodes plan))]
      (reduce (fn [acc node]
                (if (valid? node)
                  (let [result (resolve-node env node)]
                    (if (error? result)
                      (assoc acc :errors
                        (conj (:errors acc) result))  ;; depth 6 — which ) closes what?
                      (update acc :results conj result)))
                  acc))
              {:results [] :errors []}
              nodes))))

;; ✅ LOW RISK — flat, each step is independently editable
(defn resolve-node-result [env node]
  (when (valid? node)
    (resolve-node env node)))

(defn accumulate-result [acc result]
  (if (error? result)
    (update acc :errors conj result)
    (update acc :results conj result)))

(defn process [env query]
  (let [nodes (:nodes (compute-plan env query))]
    (transduce (keep #(resolve-node-result env %))
               (completing accumulate-result)
               {:results [] :errors []}
               nodes)))
```

**Strategies to reduce nesting:**
1. **Extract named helpers** — each depth level becomes a named function
2. **Use threading macros** — `->`, `->>`, `cond->` flatten conditional chains
3. **Use transducers** — `transduce` replaces nested `reduce` + `if` + `let`
4. **Early returns via `when-let`/`some->`** — avoid nesting `if` inside `let`

**When nesting is unavoidable and edits are needed:** Use [rewrite-clj-transforms](../rewrite-clj-transforms/) via Babashka instead of text-based tools. rewrite-clj operates on the s-expression tree and cannot produce unbalanced parens.

---

## Minimize Reader Conditionals

**Structure files so reader conditionals (`#?`) are rare or absent.**

Reader conditionals add cognitive load — every `#?(:clj ... :cljs ...)` is a hidden branch that doubles the code paths to reason about. Instead of sprinkling reader conditionals through a file, isolate platform-specific code into dedicated files.

```clojure
;; ❌ WRONG - Reader conditionals scattered through business logic
(defn resolve-data [env query]
  (let [result (execute-query env query)
        formatted #?(:clj  (format-jvm result)
                     :cljs (format-browser result))]
    #?(:clj  (log/info "Resolved" (count result) "rows")
       :cljs (js/console.log "Resolved" (count result) "rows"))
    formatted))

;; ✅ CORRECT - Pure logic in .cljc, platform code in .clj/.cljs
;; resolve.cljc (no reader conditionals)
(defn resolve-data [env query format-fn log-fn]
  (let [result (execute-query env query)
        formatted (format-fn result)]
    (log-fn "Resolved" (count result) "rows")
    formatted))

;; resolve_jvm.clj
(defn resolve-data-jvm [env query]
  (resolve/resolve-data env query format-jvm #(log/info %&)))

;; resolve_browser.cljs
(defn resolve-data-browser [env query]
  (resolve/resolve-data env query format-browser js/console.log))
```

**Acceptable uses of reader conditionals:**
- Namespace `:import` / `:require` blocks (unavoidable)
- Thin adapter functions whose entire purpose is platform bridging
- Constants that differ by platform (e.g., line separator)

**Strategies to eliminate reader conditionals:**
1. **Push platform code to the edges** — pure `.cljc` core, thin `.clj`/`.cljs` wrappers
2. **Pass platform functions as arguments** — inject `format-fn`, `log-fn`, etc.
3. **Use protocols** — define a protocol in `.cljc`, implement in `.clj`/`.cljs`
4. **Use the `-pure` suffix convention** — `kpi_pure.cljc` (logic) + `kpi.cljs` (lifecycle)

See also: [clojurescript-cross-platform-code](../clojurescript-cross-platform-code/) for file extension decisions and JVM TDD patterns.

---

## File Extension Naming: No .cljs/.cljc Overlap

**`.cljs` and `.cljc` files must never share the same namespace.**

In ClojureScript (shadow-cljs), when both `foo.cljs` and `foo.cljc` exist for the same namespace, the `.cljs` file takes precedence and the `.cljc` is silently ignored. This means:

- The `.cljs` cannot `require` the `.cljc` (they're the same namespace)
- Code in the `.cljc` is unreachable from ClojureScript
- JVM tests may load the `.cljc` while production loads the `.cljs`, creating divergent behavior

**Pattern: Use a `-pure` suffix for cross-platform logic:**

```clojure
;; ✅ CORRECT - Separate namespaces
;; src/myapp/component/kpi_pure.cljc  → myapp.component.kpi-pure (pure logic, JVM-testable)
;; src/myapp/component/kpi.cljs       → myapp.component.kpi (requires kpi-pure, adds JS lifecycle)

;; ❌ WRONG - Same namespace, different extensions
;; src/myapp/component/kpi.cljc       → myapp.component.kpi
;; src/myapp/component/kpi.cljs       → myapp.component.kpi  (shadows .cljc!)
```

---

**See also:**
- [FUNCTIONAL-PRINCIPLES.md](./FUNCTIONAL-PRINCIPLES.md) - Pure functions, composition patterns
- [COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md) - Transducer patterns for clean data flow
- Main [SKILL.md](./SKILL.md) - Unified quality standards
- **token-efficiency** skill - Efficient editing patterns
- [rewrite-clj-transforms](../rewrite-clj-transforms/) - Structural code modification via bb + rewrite-clj
