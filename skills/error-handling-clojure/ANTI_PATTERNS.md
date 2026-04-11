# Error Handling Anti-Patterns

Common mistakes to avoid when implementing error handling.

## Silent Failures

### DON'T: Swallow Exceptions

```clojure
;; BAD - Silently ignores errors
(try
  (risky-operation)
  (catch Exception _
    nil))  ;; Returns nil, no visibility into error
```

**Problem**: Error disappears, impossible to debug, user gets unexpected nil.

### DO: Log and Rethrow

```clojure
;; GOOD - Makes error visible
(try
  (risky-operation)
  (catch Exception e
    (log/error "Operation failed" e)
    (throw (ex-info "risky-operation failed"
                    {:cause e :type :operation/failed}))))
```

**Why**: Error is logged, context is preserved, caller can handle appropriately.

---

## Vague Error Messages

### DON'T: Generic Messages

```clojure
;; BAD - No context
(throw (Exception. "Error"))

;; BAD - Unhelpful
(throw (Exception. "Something went wrong"))
```

**Problem**: No information to debug or fix the issue.

### DO: Clear, Actionable Messages

```clojure
;; GOOD - Clear, actionable, includes context
(throw (ex-info "Failed to create user: email already exists"
                {:type :validation/duplicate-email
                 :email email
                 :existing-user-id existing-id}))
```

**Why**: Message explains what failed, why it failed, and provides debugging data.

---

## Fallback Logic That Hides Problems

### DON'T: Silent Mode Switching

```clojure
;; BAD - Mode switching hides configuration issues
(defn get-auth-service [config]
  (if-let [api-key (get config :api-key)]
    (init-real-service api-key)
    (init-mock-service)))  ;; Silently switches to mock mode
```

**Problem**: Developer thinks they're testing production auth, but using mock. Configuration errors are hidden.

### DO: Explicit Configuration

```clojure
;; GOOD - Explicit about what's configured
(defn get-auth-service [config]
  (if (get config :use-mock-service?)
    (init-mock-service)
    (let [api-key (get config :api-key)]
      (have! api-key :data {:type :config/missing-api-key})
      (init-real-service api-key))))
```

**Why**: Explicit flag controls mode, missing API key fails fast, no ambiguity.

---

## Fallback Examples

### DON'T: Fallback for Configuration

```clojure
;; BAD - Config fallback hides setup problems
(defn get-database-url []
  (or (System/getenv "DATABASE_URL")
      "postgresql://localhost/dev"))  ;; Silent fallback
```

**Problem**: Missing env var is a setup issue, should fail fast, not silently use localhost.

### DO: Fail Fast on Missing Config

```clojure
;; GOOD - Missing config is an error
(defn get-database-url []
  (or (System/getenv "DATABASE_URL")
      (throw (ex-info "DATABASE_URL environment variable required"
                      {:type :config/missing-database-url}))))
```

**Why**: Configuration issues surface immediately during development.

---

### When Fallback IS Acceptable

```clojure
;; ✅ Fallback OK: Optional analytics (logged, non-critical)
(defn track-event! [event]
  (try
    (analytics/send! event)
    (catch Exception e
      (log/warn "Analytics unavailable, continuing" {:event event})
      nil)))  ; Logged fallback, non-critical feature
```

**Why**: Analytics failure shouldn't break the app, fallback is logged, user flow continues.

```clojure
;; ❌ No Fallback: Auth service unavailable (security-critical)
(defn validate-token [token]
  (let [result (auth/validate token)]
    (have! result :data {:type :auth/validation-failed})
    result))  ; Don't silently proceed without auth
```

**Why**: Security checks must never be silently bypassed.

---

## Improper Use of Try-Catch

### DON'T: Catch Everything Indiscriminately

```clojure
;; BAD - Catches all exceptions, including programming errors
(try
  (process-user-data user-data)
  (catch Exception e
    (log/error "Failed to process user" e)
    {:success false}))
```

**Problem**: Catches NullPointerException, ClassCastException, etc. - hides programming bugs.

### DO: Catch Specific Exceptions

```clojure
;; GOOD - Only catch expected exceptions
(try
  (process-user-data user-data)
  (catch java.io.IOException e
    ;; Expected: network/file I/O issues
    (log/error "I/O error processing user" e)
    {:success false :error :io-error})
  (catch clojure.lang.ExceptionInfo e
    ;; Expected: business logic validation failures
    (log/warn "Validation failed" (ex-data e))
    {:success false :error (:type (ex-data e))}))
;; Let programming errors (NPE, etc.) bubble up
```

**Why**: Expected errors are handled, programming errors fail fast and get fixed.

---

## Missing Context in Errors

### DON'T: Throw Bare Exceptions

```clojure
;; BAD - No debugging context
(when (nil? user-id)
  (throw (Exception. "User ID required")))
```

**Problem**: No information about where error occurred, what operation was attempted, what values were involved.

### DO: Use ex-info with Context

```clojure
;; GOOD - Rich context for debugging
(when (nil? user-id)
  (throw (ex-info "User ID required for payment processing"
                  {:type :validation/missing-user-id
                   :operation :payment/process
                   :amount amount
                   :timestamp (System/currentTimeMillis)})))
```

**Why**: Error includes what failed, why, and all relevant debugging data.

---

## Returning Errors Instead of Throwing

### DON'T: Mix Success and Error Return Values

```clojure
;; BAD - Caller must check for error map
(defn create-user [email password]
  (if (valid-email? email)
    (db/insert-user email password)
    {:error :invalid-email}))  ;; Returns error map

;; Caller must remember to check
(let [result (create-user email password)]
  (if (:error result)
    (handle-error result)
    (process-user result)))  ;; Easy to forget error check
```

**Problem**: Caller can forget to check for error, error maps propagate silently.

### DO: Throw Exceptions for Errors

```clojure
;; GOOD - Exceptions can't be ignored
(defn create-user [email password]
  (have! (valid-email? email)
         :data {:type :validation/invalid-email :email email})
  (db/insert-user email password))

;; Caller gets user or exception (can't forget to handle)
(let [user (create-user email password)]
  (process-user user))
```

**Why**: Exceptions force handling, can't be accidentally ignored.

---

## Macro Nesting Anti-Pattern

Never wrap resilience macros; merge configs instead.

```clojure
;; BAD - Wrapping macro in macro causes double application
(defmacro with-resilience [& body]
  `(bw/bulwark my-spec ~@body))

(defn call-api [prompt]
  (bw/bulwark api-spec            ;; Already has rate limiting!
    (with-resilience              ;; Double rate limiting!
      (api/generate prompt))))

;; GOOD - Merge configurations into one spec
(def combined-spec
  (merge base-resilience-spec rate-limit-spec retry-spec))

(defn call-api [prompt]
  (bw/bulwark combined-spec
    (api/generate prompt)))
```

---

## Best Practices Summary

1. **Never swallow exceptions** - Log and rethrow or transform
2. **Clear error messages** - Explain what, why, include context
3. **Fail fast on config** - No silent fallbacks for setup issues
4. **Catch specific exceptions** - Let programming errors bubble up
5. **Use ex-info** - Attach structured data to exceptions
6. **Throw, don't return errors** - Exceptions force proper handling
7. **Fallback only when appropriate** - Optional features, with logging
8. **Never wrap macros in macros** - Merge configs into single spec instead
