# Clojure Idioms

This document covers threading macros, control flow patterns, destructuring, and common anti-patterns.
For naming conventions, see the summary in CLAUDE.md.

## Threading Macros

### `->` (thread-first) — Object/map transformations
```clojure
(-> user
    (assoc :last-login (Instant/now))
    (update :login-count inc)
    (dissoc :temporary-token))
```

### `->>` (thread-last) — String/IO operations only

**Note:** For collection transformations, prefer transducers (see COLLECTION-PATTERNS.md).
Use `->>` only for string operations or I/O pipelines where transducers don't apply.

```clojure
;; OK - string operations
(->> lines
     (str/split-lines)
     (str/join "\n"))

;; PREFER transducers for collections (see COLLECTION-PATTERNS.md)
(into [] (comp (filter active?) (map :email)) users)
```

### `some->` — Short-circuit on nil
```clojure
(some-> user :address :postal-code (subs 0 5))
```

### `cond->` — Conditional transformations
```clojure
(cond-> request
  authenticated? (assoc :user current-user)
  admin?         (assoc :permissions :all))
```

**Guideline:** Keep pipelines to 3-7 steps. Break up longer chains.

## Control Flow

### Use `when` for single-branch with body
```clojure
;; Good
(when (valid-input? data)
  (log-event "Processing")
  (process data))

;; Avoid if without else
(if (valid-input? data)
  (do (log-event "Processing") (process data)))  ; unnecessary
```

### Use `cond` for multiple conditions
```clojure
;; Good
(cond
  (< n 0) :negative
  (= n 0) :zero
  :else   :positive)

;; Avoid nested ifs
(if (< n 0) :negative (if (= n 0) :zero :positive))  ; hard to read
```

### Use `case` for constant dispatch
```clojure
(case operation
  :add      (+ a b)
  :subtract (- a b)
  (throw (ex-info "Unknown op" {:op operation})))
```

## Destructuring

### In function arguments
```clojure
(defn format-user [{:keys [first-name last-name email]}]
  (str last-name ", " first-name " <" email ">"))
```

### With defaults
```clojure
(defn connect [{:keys [host port] :or {port 8080}}]
  (create-connection host port))
```

### Sequential destructuring
```clojure
(let [[head & tail] items]
  (process head tail))
```

## Anti-Patterns

### Mutable atoms for accumulation
```clojure
;; BAD - using atom for accumulation
(let [result (atom [])]
  (doseq [x items]
    (swap! result conj (transform x)))
  @result)

;; GOOD - use reduce or into
(into [] (map transform) items)
```

### Nested null checks
```clojure
;; BAD
(when user
  (when (:address user)
    (when (:postal-code (:address user))
      (process (:postal-code (:address user))))))

;; GOOD - use some->
(some-> user :address :postal-code process)
```

## Code Layout

- **Line length:** Keep under 80 characters
- **Indentation:** 2 spaces, never tabs
- **Closing parens:** Gather on single line (not on separate lines)
