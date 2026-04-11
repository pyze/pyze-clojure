---
name: caching-and-purity-clojure
description: "Clojure caching examples: impure vs pure function patterns, memoize correctness"
---

Clojure companion to the caching-and-purity skill (in pyze-workflow). This skill provides Clojure-specific code examples for reasoning about caching and referential transparency.

## Impure vs Pure: Clojure Examples

### Identifying Hidden Dependencies

```clojure
;; IMPURE: output depends on external state not in inputs
(defn resolve-value [_ {:keys [var-name]}]
  (let [entity (lookup-in-mutable-db var-name)]  ;; hidden dependency
    {:value (:control/value entity)}))

;; PURE: all dependencies are explicit inputs
(defn resolve-value [_ {:keys [var-name entity]}]
  {:value (:control/value entity)})
```

The impure version reads from a mutable database -- the result depends on DB state that is not captured in the function's inputs. Caching this function would produce stale results when the DB changes.

### Keyword Destructuring and Hidden State

```clojure
;; IMPURE: closed-over atom is a hidden dependency
(defn make-counter []
  (let [n (atom 0)]
    (fn [] (swap! n inc))))

;; PURE: same inputs always produce same outputs
(defn increment [n] (inc n))
```

```clojure
;; IMPURE: config read from global
(defn format-output [{:keys [data]}]
  (let [fmt @global-format-config]  ;; hidden dependency!
    (render data fmt)))

;; PURE: config is an explicit parameter
(defn format-output [{:keys [data fmt]}]
  (render data fmt))
```

---

## `memoize` Correctness

Clojure's `memoize` caches based on all arguments. It is only correct for pure functions.

### Safe: Pure Function

```clojure
;; Safe to memoize -- pure function, all dependencies in args
(defn expensive-transform [data config]
  (reduce (fn [acc item] (merge acc (transform item config)))
          {}
          data))

(def cached-transform (memoize expensive-transform))
```

### Unsafe: Impure Function

```clojure
;; UNSAFE to memoize -- reads external state
(defn fetch-and-transform [id]
  (let [data (db/fetch id)]  ;; external read!
    (transform data)))

;; This caches the FIRST result forever, even if DB changes
(def cached-fetch (memoize fetch-and-transform))  ;; BUG!
```

### Fix: Make Pure, Then Memoize

```clojure
;; Step 1: Separate the impure fetch from the pure transform
(defn fetch [id] (db/fetch id))           ;; impure -- do NOT memoize
(defn transform-data [data] (transform data))  ;; pure -- safe to memoize

;; Step 2: Memoize only the pure part
(def cached-transform (memoize transform-data))

;; Step 3: Compose at the call site
(defn fetch-and-transform [id]
  (cached-transform (fetch id)))
```

---

## Reasoning Checklist (Clojure-Specific)

Before adding `memoize` or any cache to a Clojure function:

1. **List all dependencies** -- Does the function deref any atoms, read from DB, call external services?
2. **Check closure** -- Does the function close over any mutable state (atoms, volatiles)?
3. **Check args** -- Do the args fully determine the output?
4. **If yes to all** -- `memoize` is safe
5. **If no** -- Either make the hidden dependency an explicit arg (preferred) or do not memoize

---

## Anti-Patterns in Clojure

| Thought | Problem |
|---------|---------|
| "Wrap it in `memoize` to make it faster" | Only correct if the function is pure w.r.t. its args |
| "The cache is returning stale data" | The function was never pure -- find the hidden dependency |
| "Add `(reset! cache {})` to clear stale entries" | Treating the symptom; make the function pure instead |
| "Use `volatile!` for the cache since it's single-threaded" | `volatile!` doesn't change whether caching is correct -- purity does |
