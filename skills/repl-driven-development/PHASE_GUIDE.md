# REPL-Driven Development: Phase Guide

Detailed checklists for all 13 phases of the REPL-driven development cycle.

---

## Phase 1: Specify

**Goal**: Define requirements clearly before coding.

### Checklist
- [ ] Define inputs and outputs
- [ ] Document expected behavior with examples
- [ ] List edge cases and uncertainties
- [ ] Write specifications in issue acceptance criteria
- [ ] Create sample input/output pairs

### Example
```clojure
;; Specification
;; Function: calculate-discount
;; Input: {:total 100 :user-type :premium}
;; Output: 90 (10% discount for premium users)
;; Edge cases: nil total, unknown user-type, negative total
```

---

## Phase 2: Research

**Goal**: Gather technical context and validate assumptions.

### Checklist
- [ ] Test library candidates at REPL
- [ ] Discover actual behavior (not just docs)
- [ ] Validate assumptions from Phase 1
- [ ] Document library constraints
- [ ] Identify potential integration issues

### Example
```clojure
;; Research clojure.string library
(require '[clojure.string :as str])
(dir str)  ;; List all functions
(doc str/split)  ;; Read documentation
(str/split "a,b,c" #",")  ;; Test actual behavior
```

---

## Phase 3: Explore

**Goal**: Understand problem space through experimentation.

### Checklist
- [ ] Load required namespaces
- [ ] Examine actual data structures
- [ ] Try multiple approaches
- [ ] Identify simplest solution
- [ ] Document unexpected findings

### Example
```clojure
;; Load namespace
(require '[my.app.data :as data] :reload)

;; Examine data
(keys (first (data/get-users)))

;; Try different queries
(filter #(= (:type %) :premium) (data/get-users))
```

---

## Phase 4: Validate

**Goal**: Test assumptions systematically.

### Checklist
- [ ] Test happy path
- [ ] Test edge cases (nil, [], {}, "")
- [ ] Test error cases
- [ ] Verify performance characteristics
- [ ] Document validation results

### Example
```clojure
;; Happy path
(my-fn {:valid "data"})
;; => expected result

;; Edge cases
(my-fn nil)  ;; => should handle gracefully
(my-fn {})   ;; => should return default
(my-fn "")   ;; => should validate input
```

---

## Phase 5: Design

**Goal**: Create solution plan based on experiments.

### Checklist
- [ ] Choose data structures based on experiments
- [ ] Design function signatures
- [ ] Document design decisions
- [ ] Plan error handling strategy
- [ ] Identify reusable components

### Example
```clojure
;; Design decision: Use map for user lookup (fast access)
;; Function signature:
;; (defn get-user-discount [user-db user-id]
;;   "Returns discount percentage for user or 0 if not found")
```

---

## Phase 6: Develop

**Goal**: Build incrementally with immediate testing.

### Checklist
- [ ] Start with core function
- [ ] Test immediately at REPL
- [ ] Refine incrementally
- [ ] Add one feature at a time
- [ ] Verify each addition works

### Example
```clojure
;; Step 1: Core function
(defn calculate-discount [total user-type]
  (case user-type
    :premium (* total 0.9)
    total))

;; Test immediately
(calculate-discount 100 :premium)  ;; => 90.0
(calculate-discount 100 :standard) ;; => 100

;; Step 2: Add validation
(defn calculate-discount [total user-type]
  (when (and (number? total) (pos? total))
    (case user-type
      :premium (* total 0.9)
      total)))

;; Test validation
(calculate-discount -10 :premium)  ;; => nil
```

---

## Phase 7: JVM Unit Tests

**Goal**: Validate in production environment.

### Checklist
- [ ] Run tests for specific namespace
- [ ] Verify all tests pass
- [ ] Check for missing edge cases
- [ ] Add tests for discovered bugs
- [ ] Document test coverage

### Commands
```bash
# Run tests for specific namespace
clojure -M:test :only [namespace.under.test]

# Run all tests
clojure -M:test
```

---

## Phase 8: Browser Validation

**Goal**: Test in real browser environment.

### Checklist
- [ ] Navigate to feature in browser
- [ ] Interact with page
- [ ] Check for console errors
- [ ] Verify DOM updates correctly
- [ ] Test async behavior

### Example
```clojure
;; Connect to browser REPL
(shadow/repl :app)

;; Inspect DOM
(js/console.log "Current state:" @app-state)

;; Test interaction
(dispatch! [:user/update-preference :theme "dark"])
```

---

## Phase 9: Critique

**Goal**: Review implementation quality.

### Checklist
- [ ] Specification alignment - does it meet requirements?
- [ ] Idiomatic code - follows Clojure conventions?
- [ ] Performance acceptable - no obvious bottlenecks?
- [ ] Error handling complete - handles edge cases?
- [ ] Code clarity - easy to understand?

---

## Phase 10: Build

**Goal**: Compose components into complete workflows.

### Checklist
- [ ] Compose from validated pieces
- [ ] Test complete workflows
- [ ] Verify component interaction
- [ ] Check integration points
- [ ] Validate end-to-end behavior

### Example
```clojure
;; Compose workflow
(defn process-order [order-data user-id]
  (let [user (get-user user-id)
        discount (calculate-discount (:total order-data) (:type user))
        final-total (apply-discount order-data discount)]
    (save-order! final-total)))

;; Test complete workflow
(process-order {:total 100 :items [...]} "user-123")
```

---

## Phase 11: Edit

**Goal**: Refine and polish implementation.

### Checklist
- [ ] Remove unnecessary complexity
- [ ] Improve naming clarity
- [ ] Add documentation
- [ ] Extract reusable functions
- [ ] Simplify control flow

### Before/After Example
```clojure
;; Before
(defn f [x]
  (if (not (nil? x))
    (if (> x 0)
      (* x 2)
      x)
    0))

;; After
(defn double-positive [n]
  "Doubles positive numbers, returns 0 for nil."
  (cond
    (nil? n) 0
    (pos? n) (* n 2)
    :else n))
```

---

## Phase 12: Verify

**Goal**: Ensure compliance with requirements.

### Checklist
- [ ] Run specification examples
- [ ] Write unit tests for new code
- [ ] Run full test suite
- [ ] Verify acceptance criteria met
- [ ] Document test results

### Example
```bash
# Run full test suite
clojure -M:test

# Verify specific behavior
clojure -M:test :only [my.app.orders-test]
```

---

## Phase 13: Code Quality

**Goal**: Catch syntax issues and style violations.

### Checklist
- [ ] Run clj-kondo lint
- [ ] Fix warnings
- [ ] Review for common mistakes
- [ ] Final code review
- [ ] Commit clean code

### Commands
```bash
# Lint source directories
clj-kondo --lint src/ test/

# Find specific issues
clj-kondo --lint src/ | grep "unused"

# Fix all warnings before commit
```

---

## Summary

Use this guide to work through each phase systematically. Not every phase is needed for every task - see the main SKILL.md for guidance on when to use compressed vs full cycle.
