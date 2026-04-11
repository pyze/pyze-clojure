# REPL-Driven Development: Patterns & Best Practices

Common patterns, anti-patterns, and best practices for REPL-driven development.

**Primary tool**: clojure_eval (MCP tool). See [clojure-mcp-repl](../clojure-mcp-repl/) for complete reference.

---

## MANDATORY: Use clojure_eval MCP Tool

**All REPL evaluation MUST use the clojure_eval MCP tool connected to the existing nREPL.**

```clojure
;; Use clojure_eval MCP tool for all evaluation

;; Function invocation
(my.namespace/my-fn arg1 arg2)

;; Any Clojure expression
(your-code-here)

;; For ClojureScript, switch context first
(shadow/repl :app)
(your-cljs-code)
```

**NEVER start a new REPL process. Always use the clojure_eval MCP tool.**

---

## REPL Best Practices

### Reload Strategy Decision Tree

```
Code not reflecting changes?
       │
       ├─ Simple function changes → (require '[ns] :reload)
       │
       ├─ Multimethod/protocol changes → (remove-ns 'ns) then require
       │
       └─ Everything seems broken → Restart REPL (fresh JVM)
```

### Always Reload with :reload

```clojure
;; CORRECT - Gets latest version
(require '[my.namespace :as ns] :reload)

;; WRONG - Might use stale cached version
(require '[my.namespace :as ns])

;; Nuclear option for stuck state
(remove-ns 'my.namespace)  ;; Removes entire namespace
(require '[my.namespace :as ns])  ;; Fresh load
```

**When to use `remove-ns`**: Multimethod definitions, protocol extensions, or macro changes may not update with simple `:reload`. Use `remove-ns` to fully clear stale definitions.

### Use REPL Helper Functions

```clojure
(require '[clojure.repl :refer [doc source dir]])

;; (dir ns) - List all public functions in a namespace
(dir clojure.string)
;; => blank? capitalize escape includes? index-of ...

;; (doc fn) - Show documentation for a function
(doc map)
;; => Shows arglist, docstring, and examples

;; (source fn) - View the source code of a function
(source filter)
;; => Prints the actual implementation
```

### Immediate Feedback Loop

```
Write function
    ↓
Test at REPL immediately
    ↓
Refine based on results
    ↓
Repeat
```

### Keep REPL Session Alive

**Instead of restarting**:
- Use `:reload` flag when requiring namespaces
- Use `(remove-ns 'ns)` + `(require '[ns] :reload)` for stuck state
- Only restart for JVM-level issues or corrupted multimethod state

---

## Common Patterns

### Exploring a New Library

```clojure
;; 1. Load the library
(require '[some.library :as lib])

;; 2. Explore the namespace
(dir lib)

;; 3. Read docs for interesting functions
(doc lib/some-function)

;; 4. Try it with sample data
(lib/some-function {:test "data"})

;; 5. Test edge cases
(lib/some-function nil)
(lib/some-function {})
```

### Debugging at REPL

```clojure
;; Inspect intermediate values
(let [step1 (first-fn data)
      _ (println "step1:" step1)
      step2 (second-fn step1)
      _ (println "step2:" step2)]
  (final-fn step2))

;; Use tap> for non-blocking inspection
(tap> {:step "step1" :value step1})
```

### Testing State Changes

```clojure
;; Before
(get-current-state @db [:entity/id :my-entity])

;; Make change
(dispatch! [:some-action {:data "value"}])

;; After - verify change
(get-current-state @db [:entity/id :my-entity])
```

### Iterative Refinement

```clojure
;; Start simple
(defn process-data [data]
  (map :value data))

;; Test
(process-data [{:value 1} {:value 2}])
;; => (1 2)

;; Add validation
(defn process-data [data]
  (when (seq data)
    (map :value data)))

;; Test edge case
(process-data [])
;; => nil

;; Add transformation
(defn process-data [data]
  (when (seq data)
    (map (comp inc :value) data)))

;; Test transformation
(process-data [{:value 1} {:value 2}])
;; => (2 3)
```

---

## Running Tests

### JVM Unit Tests

```bash
# Run tests for specific namespace
clojure -M:test :only [namespace.under.test]

# Run all tests
clojure -M:test

# Run tests matching pattern
clojure -M:test :only [namespace.under.test/specific-test]
```

### At REPL

```clojure
;; Load test namespace
(require '[my.app.core-test :as t] :reload)

;; Run all tests in namespace
(clojure.test/run-tests 'my.app.core-test)

;; Run specific test
(clojure.test/test-vars [#'t/specific-test])
```

### Code Quality

```bash
# Lint source directories
clj-kondo --lint src/ test/

# Find specific issues
clj-kondo --lint src/ | grep "unused"
```

---

## Anti-Patterns

### Don't: Batch Implementing Without Testing

```clojure
;; WRONG - Implementing multiple functions without testing
(defn fn1 [x] (+ x 1))
(defn fn2 [x] (* x 2))
(defn fn3 [x] (- x 3))
;; Test all at once - bugs compound!

;; CORRECT - Test each immediately
(defn fn1 [x] (+ x 1))
(fn1 5)  ;; => 6

(defn fn2 [x] (* x 2))
(fn2 5)  ;; => 10

(defn fn3 [x] (- x 3))
(fn3 5)  ;; => 2
```

### Don't: Skip REPL Validation

```clojure
;; WRONG - Assuming simple changes work
(defn calculate-total [items]
  (reduce + (map :price items)))
;; Ship to production without testing

;; CORRECT - Validate immediately
(defn calculate-total [items]
  (reduce + (map :price items)))

(calculate-total [{:price 10} {:price 20}])  ;; => 30
(calculate-total [])  ;; => 0
(calculate-total nil)  ;; => NullPointerException! Fix needed.
```

### Don't: Restart REPL Instead of Reload

```clojure
;; WRONG - Restart entire REPL for small changes
;; (kills all state, wastes time)

;; CORRECT - Use targeted reload
(require '[my.namespace :as ns] :reload)

;; For stuck state - remove and reload
(remove-ns 'my.namespace)
(require '[my.namespace :as ns])
```

### Don't: Write Code Then Test

```clojure
;; WRONG - Write entire implementation first
(defn complex-workflow [data]
  (let [cleaned (clean-data data)
        validated (validate cleaned)
        transformed (transform validated)
        enriched (enrich transformed)
        formatted (format-output enriched)]
    formatted))
;; Now test and find bugs everywhere

;; CORRECT - Build and test incrementally
(defn clean-data [data] ...)
(clean-data test-data)  ;; Test!

(defn validate [data] ...)
(validate (clean-data test-data))  ;; Test!

(defn transform [data] ...)
(transform (validate (clean-data test-data)))  ;; Test!

;; Continue building on validated pieces
```

---

## Summary

**Key Principles**:
1. **Test immediately** - Every function, every change
2. **Use `:reload`** - Don't restart the REPL unnecessarily
3. **Explore first** - Try libraries at REPL before committing to them
4. **Incremental refinement** - Start simple, add complexity gradually
5. **Keep REPL alive** - Preserve state, reload namespaces strategically

**Remember**: The REPL is your primary development tool. Use it continuously for exploration, validation, and debugging.
