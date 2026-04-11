---
name: testing-patterns-clojure
description: "Clojure testing anti-patterns with clojure.test code examples: #' var access, with-redefs, config divergence"
---

Clojure companion to the testing-patterns skill (in pyze-workflow). This skill provides Clojure-specific code examples for each anti-pattern using `clojure.test`.

## Anti-Pattern 1: Private Var Access (`#'`)

Tests reaching into implementation details with `#'` (var quote) bypass public API guards.

```clojure
;; WRONG - Accessing private var directly
(deftest test-internal-parse
  (is (= {:parsed true} (#'my.ns/parse-internal "input"))))

;; CORRECT - Test through the public API
(deftest test-parse
  (is (= {:parsed true} (my.ns/parse "input"))))
```

**Detection:** Search for `#'` (var quote) in test files. Each occurrence is a test reaching into implementation details.

```bash
grep -rn "#'" test/
```

**Severity:** HIGH

---

## Anti-Pattern 2: Configuration Divergence

Tests using different default configurations than production silently test different behavior.

```clojure
;; Production code with defaults
(defn create-pipeline
  [{:keys [plugins strategy]
    :or {plugins [:validation :enrichment]
         strategy :merge-all}}]
  ...)

;; WRONG - Test omits production defaults
(deftest test-pipeline
  (let [p (create-pipeline {:plugins [:validation]})]  ;; Missing :enrichment!
    (is (valid? p))))

;; CORRECT - Test uses production defaults (or tests override explicitly)
(deftest test-pipeline-defaults
  (let [p (create-pipeline {})]  ;; Gets all production defaults
    (is (valid? p))))

(deftest test-pipeline-custom-plugins
  ;; Explicitly testing override behavior - this is intentional
  (let [p (create-pipeline {:plugins [:validation]})]
    (is (valid? p))))
```

**Detection:**
- Find `:or` defaults in public function destructuring
- Compare test invocations against these defaults
- Flag tests that accidentally omit production defaults

```bash
grep -rn ":or {" src/
```

**Severity:** MEDIUM

---

## Anti-Pattern 3: Manual Internal Data Construction

Tests that hand-build maps matching internal data shapes become stale when internals change.

```clojure
;; WRONG - Hand-building internal error shape
(deftest test-error-handling
  (let [error {::pipeline/error {:message "fail" :code :timeout}}]
    (is (retryable? error))))

;; CORRECT - Let production code produce the shape
(deftest test-error-handling
  (let [error (pipeline/make-error :timeout "fail")]
    (is (retryable? error))))
```

**Detection:** Search for namespace-qualified keywords from production namespaces used in test map literals (as test input, not in assertions).

```bash
grep -rn "::[a-z]" test/
```

**Severity:** MEDIUM

---

## Anti-Pattern 4: Test Helper Passthrough Gaps

Test helpers that wrap production APIs but don't forward all relevant options.

```clojure
;; WRONG - Helper extracts some opts but doesn't forward :plugins
(defn test-signal [resolvers query opts assertion-fn]
  (let [env (pci/register resolvers)
        n (:n opts 1)
        flow (reactive-signal-eql env query)]  ;; opts NOT passed!
    ...))

;; CORRECT - Forwards relevant opts
(defn test-signal [resolvers query opts assertion-fn]
  (let [env (pci/register resolvers)
        n (:n opts 1)
        flow (reactive-signal-eql env query (select-keys opts [:plugins]))]
    ...))
```

**Severity:** HIGH

---

## Anti-Pattern 5: Excessive Mocking (`with-redefs`)

`with-redefs` on internal functions creates tests coupled to implementation rather than behavior. If you need `with-redefs`, the production code has a hidden dependency that should be an explicit parameter.

```clojure
;; WRONG - Mocking internal function
(deftest test-process
  (with-redefs [my.ns/internal-helper (fn [_] :mock)]
    (is (= :expected (my.ns/process input)))))

;; CORRECT - Make dependency explicit
(deftest test-process
  (is (= :expected (my.ns/process input {:helper mock-helper}))))
```

**Detection:** Search for `with-redefs` and `with-bindings` in test files.

```bash
grep -rn "with-redefs\|with-bindings" test/
```

**Severity:** MEDIUM for `with-redefs` on internal functions. LOW for `with-redefs` on external dependencies.

---

## Anti-Pattern 6: Duplicated Production Logic

Test helpers that reimplement logic already in production code diverge over time.

```clojure
;; WRONG - Test helper reimplements production normalization
(defn test-normalize [entity]
  (assoc entity :id (str (:type entity) "-" (:name entity))))

;; CORRECT - Use the production function
(defn test-normalize [entity]
  (production.ns/normalize entity))
```

**Severity:** MEDIUM

---

## Summary Table

| Anti-Pattern | Severity | Detection Signal |
|-------------|----------|-----------------|
| Private var access (`#'`) | HIGH | `grep "#'" test/` |
| Test helper passthrough gaps | HIGH | Compare helper args vs production API |
| Configuration divergence (`:or`) | MEDIUM | `grep ":or {" src/` |
| Manual internal construction (`::ns/key`) | MEDIUM | `grep "::[a-z]" test/` |
| Excessive mocking (`with-redefs`) | MEDIUM | `grep "with-redefs" test/` |
| Duplicated production logic | MEDIUM | Test helper body matches production fn |

**The test**: For every test, ask: "Would this test still pass if I changed the internals but kept the public behavior identical?" If the answer is no, the test is coupled to implementation.
