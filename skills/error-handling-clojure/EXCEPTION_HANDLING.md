# Exception Handling Best Practices

Patterns for robust exception handling that preserves context, ensures visibility, and enables debugging.

## Core Principles

| Principle | Description |
|-----------|-------------|
| **Never lose context** | Always chain exceptions; root cause must be accessible |
| **Never swallow exceptions** | Every exception must be logged or rethrown |
| **Always log server-side** | Log before sending errors to clients |
| **Classify errors** | Distinguish retryable from non-retryable errors |
| **Extract for clients** | Send safe messages to clients, full context to logs |

---

## Exception Chaining (Never Lose Context)

### DON'T: Lose the Original Cause

```clojure
;; BAD - Original exception lost
(try
  (external-api-call)
  (catch Exception e
    (throw (ex-info "API call failed" {}))))  ;; No :cause!
```

**Problem**: Root cause (network error? auth failure? timeout?) is lost. Debugging impossible.

### DO: Always Chain Exceptions

```clojure
;; GOOD - Cause chain preserved
(try
  (external-api-call)
  (catch Exception e
    (throw (ex-info "API call failed"
                    {:type :api/call-failed
                     :endpoint "/users"
                     :method :POST}
                    e))))  ;; e is the cause!
```

**Why**: Full stack trace available, root cause accessible via `(.getCause ex)`.

### Accessing the Cause Chain

```clojure
;; Extract full cause chain for logging
(defn cause-chain [e]
  (take-while some? (iterate #(.getCause %) e)))

;; Example: Get all messages in chain
(defn full-error-message [e]
  (->> (cause-chain e)
       (map #(.getMessage %))
       (str/join ": ")))

;; "API call failed: Connection refused: connect"
```

---

## Always Log Server-Side (Even If Sending to Client)

### The Extract-and-Log Pattern

When returning errors to clients, ALWAYS log the full error server-side first:

```clojure
(require '[clojure.tools.logging :as log])

(defn- extract-client-message
  "Extract safe message for client, preserving cause chain."
  [e]
  (let [msg (ex-message e)
        cause (.getCause e)
        cause-msg (when cause (.getMessage cause))]
    (if (and cause-msg (not= msg cause-msg))
      (str msg ": " cause-msg)
      msg)))

(defn try-operation
  "Execute operation, logging errors server-side before returning to client."
  [operation-fn context]
  (try
    {:success true :result (operation-fn)}
    (catch Exception e
      ;; ALWAYS log full exception server-side
      (log/error e "Operation failed" context)
      ;; Return safe message to client
      {:success false
       :error (extract-client-message e)})))
```

### Why This Matters

```
┌─────────────────────────────────────────────────────────────────┐
│  SERVER LOG (full context)          CLIENT RESPONSE (safe)      │
│  ─────────────────────────          ─────────────────────────   │
│  ERROR Operation failed             {:success false             │
│  {:user-id "123" :op :gen}           :error "Generation failed: │
│  com.google.genai.errors...          invalid JSON response"}    │
│    at gemini_impl.clj:147                                       │
│    at orchestrator.clj:89                                       │
│  Caused by: JsonEOFException                                    │
│    Unexpected end-of-input                                      │
│    at JsonParser.java:1234                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Server has**: Full stack trace, context map, root cause
**Client gets**: Safe, readable message without sensitive details

---

## Classifying Retryable vs Non-Retryable Errors

Not all errors should be retried. Classify errors to handle appropriately:

### Retryable Errors

| Error Type | Why Retryable | Example |
|------------|---------------|---------|
| Rate limit (429) | Temporary, will resolve | API quota exceeded |
| Server errors (5xx) | Transient service issues | 503 Service Unavailable |
| Network timeouts | Connection issues | SocketTimeoutException |
| JSON parse errors* | LLM output varies | JsonEOFException from Gemini |

*LLMs can produce invalid JSON even with schemas; retry may succeed.

### Non-Retryable Errors

| Error Type | Why Not Retryable | Example |
|------------|-------------------|---------|
| Auth errors (401/403) | Credentials won't change | Invalid API key |
| Bad request (400) | Request itself is wrong | Missing required field |
| Not found (404) | Resource doesn't exist | Invalid endpoint |
| Validation errors | Business logic failure | Invalid email format |

### Implementation Pattern

```clojure
(defn- retryable-error?
  "Determine if exception should trigger retry."
  [ex]
  (let [cause (unwrap-exception ex)]
    (cond
      ;; Rate limit - retryable
      (and (instance? ClientException cause)
           (= 429 (.code cause)))
      true

      ;; Server errors - retryable
      (instance? ServerException cause)
      true

      ;; Network timeouts - retryable
      (instance? java.net.SocketTimeoutException cause)
      true

      ;; JSON parse errors - retryable (LLM may produce valid JSON on retry)
      (json-parse-error? cause)
      true

      ;; Everything else - not retryable
      :else false)))

(defn- json-parse-error?
  "Check if exception is a JSON parsing error."
  [cause]
  (or (instance? com.fasterxml.jackson.core.JsonParseException cause)
      (instance? com.fasterxml.jackson.core.io.JsonEOFException cause)
      (when-let [class-name (some-> cause class .getName)]
        (str/includes? class-name "jackson"))))
```

---

## Exception Handling in Async/Parallel Code

Special care needed when exceptions occur in worker threads or async operations.

### DON'T: Lose Exceptions in Worker Threads

```clojure
;; BAD - Exception lost in future
(future
  (try
    (risky-operation)
    (catch Exception e
      nil)))  ;; Exception disappears!
```

### DO: Capture and Return Errors

```clojure
;; GOOD - Exceptions captured in result
(defn submit-task! [ctx task-fn task-id]
  (.put task-queue
        {:task-fn (fn []
                    (try
                      (task-fn)
                      (catch Exception e
                        {:task-id task-id
                         :error (.getMessage e)
                         :error-data (ex-data e)})))
         :task-id task-id}))

;; Caller receives error in result, can handle appropriately
```

### Aggregate Errors from Parallel Operations

```clojure
(defn process-batch [items process-fn]
  (let [results (pmap (fn [item]
                        (try
                          {:success true :item item :result (process-fn item)}
                          (catch Exception e
                            {:success false :item item :error (ex-message e)})))
                      items)
        failures (filter #(not (:success %)) results)]
    ;; Log all failures server-side
    (doseq [failure failures]
      (log/error "Batch item failed" failure))
    ;; Return aggregated results
    {:successes (filter :success results)
     :failures failures
     :stats {:total (count items)
             :succeeded (count (filter :success results))
             :failed (count failures)}}))
```

---

## Logging Best Practices

### Log Levels for Exceptions

| Level | When to Use | Example |
|-------|-------------|---------|
| ERROR | Unexpected failures, requires attention | Auth service unavailable |
| WARN | Expected but unusual, retry triggered | Rate limit hit, retrying |
| INFO | Normal operation with notable events | Request completed with warnings |
| DEBUG | Detailed troubleshooting info | Full request/response bodies |

### Structured Logging with Context

```clojure
;; GOOD - Structured context for searchability
(log/error e "Generation failed"
           {:element-type :field
            :field-name (:pyPropertyName field)
            :class-name (:pyClassName field)
            :attempt attempt-number})

;; Enables queries like:
;; grep "element-type.*:field" server.log
;; grep "class-name.*MyClass" server.log
```

### What to Include in Error Logs

```clojure
;; Comprehensive error log entry
(log/error exception "Operation failed"
           {:operation :generate-requirements
            :entity-type :procedure
            :entity-id (:name proc)
            :timestamp (System/currentTimeMillis)
            :thread (.getName (Thread/currentThread))
            :attempt (inc retry-count)
            :duration-ms elapsed
            ;; DON'T include: passwords, tokens, PII
            })
```

---

## Checklist: Exception Handling

When implementing exception handling, verify:

- [ ] **Cause chained**: `(ex-info msg data cause)` - cause is third arg
- [ ] **Server-side logged**: Full exception logged before client response
- [ ] **Context included**: Relevant identifiers in log context
- [ ] **Retryable classified**: Error types properly categorized
- [ ] **Client message safe**: No stack traces or sensitive data to client
- [ ] **Async captured**: Worker thread exceptions captured in results
- [ ] **Aggregate reported**: Batch operation failures summarized

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  EXCEPTION HANDLING FLOW                                         │
│                                                                  │
│  1. Catch exception                                              │
│  2. LOG FULL EXCEPTION SERVER-SIDE (with context)               │
│  3. Classify: retryable?                                        │
│     ├─ YES → Retry with backoff                                 │
│     └─ NO  → Extract safe message for client                    │
│  4. Chain exception if rethrowing (preserve cause)              │
│  5. Return/throw with appropriate error type                    │
└─────────────────────────────────────────────────────────────────┘
```

**Key rules**:
1. Never lose exception context - always chain with cause
2. Always log server-side - even if sending to client
3. Classify errors - retry transient failures, fail fast on permanent ones
4. Extract safe messages - full context in logs, safe message to client
