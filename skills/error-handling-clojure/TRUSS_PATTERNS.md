# Truss Assertion Patterns

Advanced patterns for using Taoensso Truss for declarative, composable error handling.

## Gathering and Maintaining Context

**ARCHITECTURAL PRINCIPLE**: Use `with-ctx+` throughout the call chain to accumulate debugging context.

### Request-Scoped Context Pattern

```clojure
;; At handler entry point - establish request context
(defn handle-request [request]
  (truss/with-ctx+ {:request-id (get-in request [:headers "x-request-id"])
                    :user-id    (get-in request [:identity :user-id])
                    :handler    :api/handle-request
                    :timestamp  (System/currentTimeMillis)}
    ;; All downstream assertions include this context
    (process-request request)))

;; In processing layer - add processing context
(defn process-request [request]
  (truss/with-ctx+ {:step :processing}
    (let [data (get-in request [:body-params :data])]
      (have! [:and map? (complement empty?)] data
             :data {:type :validation/invalid-data})
      (save-to-db data))))

;; In service layer - add service context
(defn save-to-db [data]
  (truss/with-ctx+ {:step :database
                    :record-count (count data)}
    (have! (< (count data) 1000)
           :data {:type :validation/too-many-records})
    ;; Error now includes: request-id, user-id, handler, step, record-count
    (db/insert data)))
```

### Context in Event Handlers

```clojure
;; Event/action handler with context
(defn submit-form-handler
  [db {:keys [now input-data]}]
  (truss/with-ctx+ {:action :form/submit
                    :has-input? (some? input-data)
                    :timestamp now}
    ;; All assertions in this handler include context
    (have! (not (empty? input-data))
           :data {:type :validation/empty-data})
    ;; Process the submission
    (save-form-data! db input-data)))
```

---

## Complex Predicates

Truss supports rich predicate syntax for sophisticated validation:

### AND Predicates

```clojure
;; AND - must match all conditions
(have [:and string? (complement str/blank?)] text)
;; Requires: string AND not empty

;; Multiple predicates
(have [:and map?
       (fn [m] (contains? m :id))
       (fn [m] (contains? m :name))] user)
;; Must be a map AND have :id key AND have :name key
```

### OR Predicates

```clojure
;; OR - must match at least one
(have [:or nil? pos-int?] optional-count)
;; Allows: nil OR positive integer

;; Multiple alternatives
(have [:or string? keyword? symbol?] identifier)
;; Accepts: string OR keyword OR symbol
```

### Map Key Validation

```clojure
;; Map contains only allowed keys (no extra keys)
(have [:ks<= #{:name :email :age}] user-data)
;; Map contains only these keys (no extra keys allowed)

;; Map contains required keys
(have [:ks>= #{:id :name}] user-data)
;; Map must have at least :id and :name keys

;; Exact key match
(have [:ks= #{:id :name :email}] user-data)
;; Map must have exactly these keys, no more, no less
```

### Collection Validation

```clojure
;; Collection length
(have [:and vector? (fn [v] (<= 1 (count v)))] items)
;; Must be a vector with at least 1 item

;; Collection predicates
(have [:and coll? seq (fn [c] (<= (count c) 100))] data)
;; Must be a collection, non-empty, with at most 100 items

;; Every element matches predicate
(have [:and vector? (fn [v] (every? string? v))] strings)
;; Must be a vector and every element must be a string
```

### Custom Predicates

```clojure
;; Define custom predicate functions
(defn valid-email? [s]
  (and (string? s)
       (re-matches #".+@.+\..+" s)))

(have valid-email? email)
;; Uses custom validation logic

;; Inline predicate
(have (fn [x] (and (number? x) (< 0 x 100))) percentage)
;; Must be a number between 0 and 100
```

---

## Error Handling Patterns

### Pattern 1: Initialization Validation

```clojure
;; Validate all required config at startup
(defn initialize-system [config]
  (have! (get config :api-key)
         :data {:type :config/missing-api-key})
  (have! (get config :db-url)
         :data {:type :config/missing-db-url})
  (have! (number? (get config :timeout))
         :data {:type :config/invalid-timeout})
  ;; Only continue if all checks pass
  (setup-services config))
```

### Pattern 2: Input Validation

```clojure
;; Validate user input at handler entry point
(defn create-item-handler [request]
  (let [body (:body-params request)
        name (get body :name)
        quantity (get body :quantity)]

    ;; Fail fast on invalid input
    (have! name :data {:type :validation/missing-name})
    (have! [:and string? (complement str/blank?)] name
           :data {:type :validation/invalid-name})
    (have! [:and number? pos?] quantity
           :data {:type :validation/invalid-quantity})

    ;; Process only if validation passes
    (db/create-item name quantity)))
```

### Pattern 3: Type Safety

```clojure
(defn process-payment [user-id amount]
  ;; Ensure types are correct
  (have! [:and string? (complement str/blank?)] user-id
         :data {:type :validation/invalid-user-id})
  (have! [:and number? pos?] amount
         :data {:type :validation/invalid-amount
                :value amount})

  ;; Type-safe processing
  (payment-service/charge user-id amount))
```

### Pattern 4: State Validation

```clojure
;; Validate state before operations that depend on it
(defn send-notification [system user-id message]
  ;; Ensure dependencies are initialized
  (have! (get system :db)
         :data {:type :system/missing-db})
  (have! (get system :notification-service)
         :data {:type :system/missing-notification-service})

  ;; Ensure user state is valid
  (have! user-id
         :data {:type :validation/missing-user-id})

  ;; Safe to proceed
  (process-notification system user-id message))
```

---

## Advanced Context Patterns

### Conditional Context

```clojure
;; Add context conditionally based on runtime state
(defn process-with-conditional-context [request]
  (let [base-context {:handler :api/process}
        context (if-let [user-id (get-in request [:identity :user-id])]
                  (assoc base-context :user-id user-id)
                  base-context)]
    (truss/with-ctx+ context
      (process-data request))))
```

### Dynamic Context from Data

```clojure
;; Build context from data being processed
(defn process-batch [items]
  (doseq [item items]
    (truss/with-ctx+ {:item-id (:id item)
                      :item-type (:type item)
                      :batch-size (count items)}
      (have! (:id item) :data {:type :validation/missing-item-id})
      (process-item item))))
```

---

## REPL Debugging

When catching Truss assertion failures at the REPL, use these to navigate error context:

```clojure
;; Extract structured data from ex-info exceptions
(try
  (have! string? 42)
  (catch clojure.lang.ExceptionInfo e
    (ex-data e)))          ;; => {:type :validation/... :value 42 ...}

;; Walk the cause chain for wrapped exceptions
(.getCause e)              ;; => original exception if wrapped with ex-info

;; Extract Truss context from caught exceptions
;; Truss attaches context via with-ctx+ as :truss/context in ex-data
(:truss/context (ex-data e))
```

---

## Best Practices

1. **Use `have!` for security**: Never elide security-critical checks
2. **Accumulate context**: Use `with-ctx+` throughout the call chain
3. **Fail fast**: Don't defer validation to deeper layers
4. **Rich predicates**: Combine predicates with `:and`, `:or`, custom functions
5. **Clear error types**: Use namespaced keywords (`:validation/invalid-email`)
6. **Include data**: Attach relevant debugging data to errors
7. **Document predicates**: Comment complex custom predicates
