# Collection Transformation Patterns

Transducer-based patterns for efficient, single-pass collection operations.

## Core Principle

**Use transducers with `into` for all multi-step collection transformations.** This ensures single-pass execution and eliminates intermediate allocations.

---

## Prohibited Patterns

**NEVER use these patterns:**

```clojure
;; ❌ PROHIBITED - doall defeats laziness, creates intermediate collection
(doall (map f coll))
(doall (filter pred (map f coll)))

;; ❌ PROHIBITED - vec realizes entire lazy sequence
(vec (map f coll))
(vec (filter pred (map f coll)))

;; ❌ PROHIBITED - Threading macros create intermediate lazy sequences
(->> coll (map f) (filter pred))
(->> coll (map f) (filter pred) (map g))
(-> coll (map f) (filter pred))

;; ❌ PROHIBITED - for macro creates lazy sequence, use xf/for or transducers
(for [x coll
      :when (pred x)]
  (f x))
;; Especially problematic with nested bindings
(for [x outer-coll
      y (get-inner x)
      :when (valid? x y)]
  [x y])
```

---

## Required Patterns

**ALWAYS use transducers with `into`:**

```clojure
;; ✅ CORRECT - Single pass, no intermediate collections
(into [] (map f) coll)

;; ✅ CORRECT - Composed transducers, single pass
(into [] (comp (map f) (filter pred)) coll)

;; ✅ CORRECT - Multiple transformations composed
(into []
      (comp (map f)
            (filter pred)
            (map g))
      coll)
```

---

## xforms Library for Extended Transducers

**The [xforms library](https://github.com/cgrand/xforms) provides additional transducers beyond clojure.core.**

**Prefer xforms transducers when applicable:**

```clojure
(require '[net.cgrand.xforms :as xf])

;; ✅ Use xf/sort instead of external sort
;; ❌ WRONG - Sort outside transducer pipeline
(sort (into [] (comp (filter pred) (map f)) coll))

;; ✅ RIGHT - xf/sort inside transducer pipeline
(into [] (comp (filter pred) (map f) (xf/sort)) coll)

;; ✅ Use xf/sort-by with key function
(into [] (comp (filter :active?) (xf/sort-by :created-at)) users)

;; ✅ Use xf/for - transducer version of list comprehension
(into []
      (xf/for [x (range 10)
               y (range 10)
               :when (< x y)]
        [x y])
      [nil])  ; xf/for doesn't consume input

;; ✅ Use xf/reduce - reduce as transducer
(into []
      (comp (partition-all 10)
            (xf/reduce + 0))  ; Sum each partition
      (range 100))

;; ✅ Use xf/multiplex - fan-out to multiple transducers
(into []
      (xf/multiplex [(map inc) (map dec)])
      (range 5))
;; => [[1 -1] [2 0] [3 1] [4 2] [5 3]]
```

**Common xforms transducers:**

```clojure
xf/sort           ; Sort elements
xf/sort-by        ; Sort by key function
xf/for            ; Transducer comprehension
xf/reduce         ; Reduce as transducer
xf/reductions     ; Intermediate reductions
xf/multiplex      ; Fan-out to multiple transducers
xf/by-key         ; Group and process by key
xf/into           ; Build collections as transducer
xf/window         ; Sliding/tumbling windows
```

---

## Pattern Conversion Guide

**Before → After examples:**

```clojure
;; Pattern 1: Simple map
;; ❌ (doall (map f coll))
;; ❌ (vec (map f coll))
✅ (into [] (map f) coll)

;; Pattern 2: Map + filter
;; ❌ (doall (filter pred (map f coll)))
;; ❌ (->> coll (map f) (filter pred))
✅ (into [] (comp (map f) (filter pred)) coll)

;; Pattern 3: Multiple transformations
;; ❌ (->> coll (map f) (filter pred) (map g))
✅ (into [] (comp (map f) (filter pred) (map g)) coll)

;; Pattern 4: Take/drop operations
;; ❌ (->> coll (map f) (take 10))
✅ (into [] (comp (map f) (take 10)) coll)

;; Pattern 5: Sorting
;; ❌ (sort (map f (filter pred coll)))
✅ (into [] (comp (filter pred) (map f) (xf/sort)) coll)

;; Pattern 6: Different output collection
;; ❌ (set (filter pred (map f coll)))
✅ (into #{} (comp (map f) (filter pred)) coll)

;; Pattern 7: Into existing collection
;; ❌ (concat existing (map f new-items))
✅ (into existing (map f) new-items)

;; Pattern 8: For comprehension - simple case
;; ❌ (for [x coll :when (pred x)] (f x))
✅ (into [] (comp (filter pred) (map f)) coll)

;; Pattern 9: For comprehension - nested bindings
;; ❌ (for [x outer-coll y (get-inner x) :when (valid? x y)] [x y])
✅ (into []
        (comp (mapcat (fn [x] (map #(vector x %) (get-inner x))))
              (filter (fn [[x y]] (valid? x y))))
        outer-coll)
;; Or use xf/for from xforms library:
✅ (into [] (xf/for [x outer-coll y (get-inner x) :when (valid? x y)] [x y]) [nil])
```

---

## Transducer Composition

**Use `comp` to compose transformations right-to-left:**

```clojure
;; Define reusable transducers
(def xf-active-users
  (comp (filter :active?)
        (map :user-id)
        (distinct)))

;; Apply to different sources
(into [] xf-active-users users-from-db)
(into #{} xf-active-users users-from-api)

;; Compose with additional transforms
(into []
      (comp xf-active-users
            (take 100))
      all-users)

;; Combine with xforms transducers
(def xf-top-users
  (comp (filter :active?)
        (xf/sort-by :score)
        (take 10)
        (map :user-id)))

(into [] xf-top-users users)
```

---

## Lazy vs Eager Decision Matrix

**When to use eager evaluation (transducers with `into`) - DEFAULT:**

- ✅ Data fits comfortably in memory (< 100K elements typically)
- ✅ Immediate error detection is important
- ✅ Debugging complex transformations
- ✅ Working with concurrent operations
- ✅ Multiple transformations need to be composed
- ✅ Performance matters (avoid lazy seq overhead)

**When to use lazy sequences (traditional `map`, `filter`) - REQUIRES USER APPROVAL:**

- ✅ Processing very large or unbounded datasets (millions+ elements)
- ✅ Memory efficiency is critical (streaming from disk/network)
- ✅ Creating infinite sequences or generators
- ✅ Early termination likely (e.g., `(first (filter pred large-coll))`)
- ✅ Single transformation only (no composition needed)
- ⚠️ **Must document performance justification in code**

**Example - When lazy is appropriate (with approval):**

```clojure
;; ✅ LAZY - Infinite sequence, early termination (approved pattern)
(defn first-prime-over [n]
  (first (filter prime? (iterate inc n))))

;; PERFORMANCE DEVIATION (approved 2025-01-31):
;; Streaming 10GB file, transducer would require loading entire file into memory.
;; Lazy approach processes incrementally with 200MB peak memory.
(defn process-large-file [path]
  (with-open [rdr (io/reader path)]
    (doall  ; Only realize filtered results
      (into []
            (comp (map parse-line)
                  (filter valid?))
            (line-seq rdr)))))

;; ❌ WRONG - Should use transducer for in-memory collection
(defn active-user-ids [users]
  (map :user-id (filter :active? users)))  ; Creates intermediate seq

;; ✅ RIGHT - Transducer for in-memory collection
(defn active-user-ids [users]
  (into [] (comp (filter :active?) (map :user-id)) users))
```

### REPL Tip: Measuring Eager Evaluation Cost

```clojure
;; Use time to measure transducer pipeline cost at the REPL
(time (into [] xf coll))
;; "Elapsed time: 12.345 msecs"
;; => result vector (also useful for verifying output)
```

---

## Performance Characteristics

**Why transducers are faster:**

```clojure
;; ❌ SLOW - Three passes, two intermediate collections
(->> (range 1000000)
     (map inc)           ; Pass 1: lazy seq of 1M elements
     (filter even?)      ; Pass 2: lazy seq, realizes previous
     (map #(* % 2)))     ; Pass 3: lazy seq, realizes previous
;; Result: 3 passes, 2 intermediate lazy seqs

;; ✅ FAST - One pass, no intermediate collections
(into []
      (comp (map inc)
            (filter even?)
            (map #(* % 2)))
      (range 1000000))
;; Result: 1 pass, direct into vector
```

---

## Transducer Building Blocks

**Common core.clojure transducers:**

```clojure
;; Transformation
(map f)           ; Transform each element
(mapcat f)        ; Transform and flatten
(replace m)       ; Replace elements using map

;; Filtering
(filter pred)     ; Keep elements matching predicate
(remove pred)     ; Remove elements matching predicate
(distinct)        ; Remove duplicates
(dedupe)          ; Remove consecutive duplicates

;; Selection
(take n)          ; Take first n elements
(drop n)          ; Drop first n elements
(take-while pred) ; Take while predicate true
(drop-while pred) ; Drop while predicate true
(take-nth n)      ; Take every nth element

;; Partitioning
(partition-all n) ; Partition into chunks of n
(partition-by f)  ; Partition when (f x) changes
```

---

## Advanced Patterns

**Using `transduce` for custom reductions:**

```clojure
;; ✅ transduce - Transform and reduce in one pass
(transduce
  (comp (filter even?) (map #(* % 2)))
  +
  0
  (range 100))
;; Returns: sum of doubled even numbers

;; Equivalent to, but more efficient than:
(reduce + 0 (map #(* % 2) (filter even? (range 100))))
```

**Using `sequence` for lazy transducer results (REQUIRES APPROVAL):**

```clojure
;; ✅ sequence - Lazy evaluation with transducers
;; Use only when lazy evaluation is necessary (infinite sequences, etc.)
(sequence
  (comp (filter prime?) (take 1000))
  (iterate inc 2))
;; Returns: lazy seq of first 1000 primes
```

**Using `eduction` for reusable transformations:**

```clojure
;; ✅ eduction - Reusable, non-cached transformation
(def xf (comp (filter even?) (map #(* % 2))))
(def evens-doubled (eduction xf (range 100)))

;; Can iterate multiple times, recalculates each time
(reduce + evens-doubled)  ; First pass
(reduce + evens-doubled)  ; Second pass (recalculates)
```

---

## Common Mistakes

**❌ Mixing threading macros with collections:**
```clojure
;; WRONG - Creates intermediate lazy sequences
(->> users
     (filter :active?)
     (map :user-id)
     (sort))
```

**✅ Use transducers with xf/sort:**
```clojure
;; RIGHT - Single pass with xf/sort
(into [] (comp (filter :active?) (map :user-id) (xf/sort)) users)
```

**❌ Using doall to "force" evaluation:**
```clojure
;; WRONG - doall is a code smell, requires deviation approval
(doall (map process-item items))
```

**✅ Use into for eager evaluation:**
```clojure
;; RIGHT - Explicit eager evaluation
(into [] (map process-item) items)
```

**❌ Sorting outside transducer pipeline:**
```clojure
;; WRONG - Breaks single-pass efficiency
(sort-by :created-at (into [] (filter :active?) users))
```

**✅ Use xf/sort-by inside pipeline:**
```clojure
;; RIGHT - Single pass with sorting
(into [] (comp (filter :active?) (xf/sort-by :created-at)) users)
```

---

## Summary

1. **NEVER use doall** - Use `into` for eager evaluation (unless approved deviation)
2. **NEVER use threading macros for collections** - Use transducers (unless approved deviation)
3. **NEVER use `for` macro** - Use `xf/for` or transducers with `mapcat`/`filter` (unless approved deviation)
4. **ALWAYS compose with `comp`** - Right-to-left composition
5. **Default to `into`** - Eager evaluation for in-memory data
6. **Use xforms transducers** - Prefer `xf/sort`, `xf/for`, etc. when applicable
7. **Use `transduce`** - For reduce operations
8. **Deviations require approval** - Ask user, document with rationale and benchmark ID
9. **Single pass wins** - Avoid intermediate collections
10. **Lazy only when necessary** - Large/infinite data or early termination (with approval)
11. **Document performance decisions** - Include benchmark reference and approval date

**See also:**
- [FUNCTIONAL-PRINCIPLES.md](./FUNCTIONAL-PRINCIPLES.md) - Pure functions, immutability, composition
- [CODE-ORGANIZATION.md](./CODE-ORGANIZATION.md) - Function/namespace size guidelines
- Main [SKILL.md](./SKILL.md) - Unified approval policies
