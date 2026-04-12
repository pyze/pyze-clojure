---
name: error-handling-clojure
description: "This skill should be used when implementing error handling in Clojure with Truss assertions, Telemere structured logging, ex-info exceptions, or debugging Integrant configuration issues."
---

Clojure companion to the error-handling-patterns skill (in pyze-workflow). This skill provides Clojure-specific implementation using Truss, Telemere, and Integrant.

## Libraries

| Library | Role | Require |
|---------|------|---------|
| **Truss** | Fail-fast assertions at boundaries | `[taoensso.truss :refer [have have!]]` |
| **Telemere** | Structured logging/signals | `[taoensso.telemere :as t]` |

---

## Decision Tree: Which Tool?

```
+-- Is this a boundary check (function entry, system init)?
|   +-- YES -> Truss `(have pred? value)` -- assertion, returns value
|
+-- Is this parsing untrusted/user input (EDN, JSON)?
|   +-- YES -> try/catch with Telemere warn signal
|
+-- Is this a catch block that logs and rethrows?
|   +-- YES -> Telemere `(t/log! {:level :error :error e} [msg data])`
|
+-- Is this a catch block that returns a fallback value?
|   +-- YES -> Telemere warn signal + return fallback
|
+-- Is this operational logging (info, debug)?
    +-- YES -> `(t/log! :info [msg])`
```

---

## Truss Assertions at Boundaries

`(have pred? value)` -- returns `value` if `(pred? value)` is truthy, throws otherwise.

```clojure
(require '[taoensso.truss :refer [have have!]])

;; Returns the value on success -- use inline
(have string? doc-id)       ;; Returns doc-id
(have map? entity)          ;; Returns entity
(have pos-int? port)        ;; Returns port

;; Custom predicates
(have #(or (keyword? %) (string? %)) var-ref)
```

### Where to Assert

| Boundary | Pattern |
|----------|---------|
| **System init** | `(have string? project-id)`, `(have pos-int? port)` |
| **Store operations** | `(have map? entity)`, `(have vector? ident)` |
| **Factory functions** | `(have string? id)`, `(have map? config)` |
| **Pipeline entry** | `(have map? input)`, `(have coll? items)` |

### `have` vs `have!`

- **`have`** -- elidable in production builds. Use for internal boundaries.
- **`have!`** -- never elided. Use for security-critical checks (auth tokens, API keys).

### Contextual Assertions

```clojure
(require '[taoensso.truss :as truss])

(truss/with-ctx+ {:handler 'user/process-request :step :validation}
  (have string? input-text)
  (have pos-int? item-count))
;; Failure includes context: {:handler ..., :step ...}
```

---

## Telemere Structured Logging

### Basic Logging

```clojure
(require '[taoensso.telemere :as t])

(t/log! :info ["Server started" {:port port}])
(t/log! :warn ["Unexpected state" {:key k :value v}])

;; Debug-level tracing (silent by default)
(t/log! :debug ["resolver path:" path "| columns:" (keys first-row)])
;; Enable: (t/set-min-level! nil "my.app.*" :debug)
```

### Error Logging with Exception Attachment

```clojure
;; BAD -- exception converted to string, stack trace lost
(t/log! :error (str "Operation failed: " (ex-message e)))

;; GOOD -- exception attached as structured data
(t/log! {:level :error :error e}
        ["Resolver error" {:op op-name :path path}])
```

### Testing Telemere Signals

```clojure
(let [{:keys [value signals]} (t/with-signals (some-fn-that-logs))]
  ;; value = return value of the body
  ;; signals = vector of captured Telemere signal maps
  (is (= 1 (count signals)))
  (is (= :warn (:level (first signals)))))
```

---

## Fail-Fast Clojure Patterns

```clojure
;; BAD - Silent fallback
(defn get-api-key []
  (or (System/getenv "API_KEY")
      (if dev-mode? "mock-key" nil)))

;; GOOD - Explicit failure
(defn get-api-key []
  (or (System/getenv "API_KEY")
      (throw (ex-info "Missing API_KEY environment variable"
                      {:type :config/missing-api-key}))))
```

### Fix the Source, Don't Route Around It

| Instead of (fallback) | Use (fix the source) |
|----------------------|---------------------|
| `(or name "Anonymous")` | `(have string? name)` |
| `(get config :key default-val)` | `(have some? (get config :key))` |
| `(when-not x (create-default))` | `(have some? x)` |
| `(try (op) (catch _ default))` | `(op)` -- let it fail; fix why it fails |

---

## Exception Handling Patterns

### Catch-Log-Rethrow

```clojure
(catch #?(:clj Exception :cljs :default) e
  (t/log! {:level :error :error e}
          ["Operation failed" {:op op-name :context ctx}])
  (throw e))
```

### Catch-and-Return

```clojure
(try
  {:success true :result (operation-fn)}
  (catch #?(:clj Exception :cljs :default) e
    (t/log! {:level :error :error e} ["Operation failed" context])
    {:success false :error (ex-message e)}))
```

### Exception Chaining (Preserve Cause)

```clojure
;; BAD - Cause lost!
(catch Exception e
  (throw (ex-info "Failed" {:type :error})))

;; GOOD - Cause preserved
(catch Exception e
  (throw (ex-info "Failed" {:type :error} e)))  ; e is the cause!
```

---

## Safe EDN Parsing Pattern

```clojure
(defn try-parse-edn
  "Parse EDN from string, returning nil on failure. Logs warning for non-blank invalid input."
  [s]
  (when-not (str/blank? s)
    (try
      (clojure.edn/read-string s)
      (catch #?(:clj Exception :cljs :default) e
        (t/log! :warn ["EDN parse failed" {:input s :error (ex-message e)}])
        nil))))
```

---

## Additional Resources

For detailed patterns and specialized topics:

- [EXCEPTION_HANDLING.md](./EXCEPTION_HANDLING.md) - Exception chaining, logging, retryable errors, async handling
- [TRUSS_PATTERNS.md](./TRUSS_PATTERNS.md) - Advanced Truss assertion patterns and complex predicates
- [ANTI_PATTERNS.md](./ANTI_PATTERNS.md) - Common mistakes to avoid in error handling
- [INTEGRANT_CONFIG.md](./INTEGRANT_CONFIG.md) - Integrant configuration gotchas and debugging
