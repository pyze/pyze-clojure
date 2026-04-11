---
name: bdd-scenarios-clojure
description: "BDD scenarios in clojure.test: Given/When/Then with deftest, testing blocks, and Clojure predicates"
---

Clojure companion to the bdd-scenarios skill (in pyze-workflow). This skill provides Clojure-specific implementation of Given/When/Then scenarios using `clojure.test`.

## Given/When/Then in clojure.test

BDD scenarios use `testing` blocks with descriptive Given/When/Then labels:

```clojure
(deftest user-authentication-test
  (testing "Given valid credentials, When authenticating, Then returns auth token"
    (let [credentials {:email "user@example.com" :password "secret"}
          result (authenticate-user credentials)]
      (is (uuid? (:user-id result)))
      (is (string? (:auth-token result)))))

  (testing "Given invalid password, When authenticating, Then returns error"
    (let [credentials {:email "user@example.com" :password "wrong"}
          result (authenticate-user credentials)]
      (is (= :invalid-credentials (:type result))))))
```

### Structure

```
Given [initial state/context]
When  [action/event occurs]
Then  [expected outcome]
```

Each `testing` block is one scenario. Group related scenarios in one `deftest`.

---

## Clojure Predicate Assertions

Use Clojure predicates in `is` assertions for flexible verification:

```clojure
(testing "Given valid input, When creating entity, Then entity has expected shape"
  (let [entity (create-entity {:name "test"})]
    (is (uuid? (:id entity)))        ;; generated ID is a UUID
    (is (string? (:name entity)))    ;; name is a string
    (is (inst? (:created-at entity))) ;; timestamp is an instant
    (is (pos-int? (:version entity))) ;; version is a positive integer
    (is (keyword? (:status entity))) ;; status is a keyword
    (is (nil? (:deleted-at entity))))) ;; not deleted
```

Common predicates for BDD assertions:
- `uuid?`, `string?`, `keyword?`, `int?`, `pos-int?`, `nat-int?`
- `map?`, `vector?`, `set?`, `coll?`, `seq?`
- `inst?`, `nil?`, `some?`, `true?`, `false?`
- `empty?`, `not-empty`

---

## Converting Specifications to Scenarios

### From Spec Examples

```clojure
;; Specification
{:specification/examples
 [{:input {:email "user@example.com" :password "secret"}
   :output {:user-id #uuid "..." :auth-token "..."}}
  {:input {:email "user@example.com" :password "wrong"}
   :output {:type :invalid-credentials :message "..."}}]}

;; Each example becomes a scenario
(deftest authenticate-user-scenarios
  (testing "Given valid credentials, When authenticating, Then succeeds"
    ...)
  (testing "Given wrong password, When authenticating, Then returns invalid-credentials"
    ...))
```

### From Spec Uncertainties

```clojure
;; Specification uncertainty
;; [?] Token expiration: 24 hours or 7 days?

;; Becomes scenario after resolution
(testing "Given expired token (>24h), When accessing resource, Then returns unauthorized"
  ...)
```

---

## Complete Worked Example

Starting from a specification for a shopping cart:

```clojure
;; Spec examples:
;; 1. Empty cart -> add item -> cart has item
;; 2. Cart with item -> remove item -> cart is empty
;; 3. Cart with item -> add same item -> quantity increases
;; 4. Empty cart -> checkout -> error

;; Converted to clojure.test scenarios:

(deftest shopping-cart-scenarios
  (testing "Given empty cart, When adding item, Then cart contains item"
    (let [cart (create-cart)
          item {:sku "ABC-123" :name "Widget" :price 9.99}
          updated (add-item cart item)]
      (is (= 1 (count (:items updated))))
      (is (= "ABC-123" (:sku (first (:items updated)))))
      (is (= 1 (:quantity (first (:items updated)))))))

  (testing "Given cart with item, When removing item, Then cart is empty"
    (let [cart (-> (create-cart)
                   (add-item {:sku "ABC-123" :name "Widget" :price 9.99}))
          updated (remove-item cart "ABC-123")]
      (is (empty? (:items updated)))))

  (testing "Given cart with item, When adding same item, Then quantity increases"
    (let [cart (-> (create-cart)
                   (add-item {:sku "ABC-123" :name "Widget" :price 9.99}))
          updated (add-item cart {:sku "ABC-123" :name "Widget" :price 9.99})]
      (is (= 1 (count (:items updated))))
      (is (= 2 (:quantity (first (:items updated)))))))

  (testing "Given empty cart, When checking out, Then returns error"
    (let [cart (create-cart)
          result (checkout cart)]
      (is (= :cart/empty (:error result))))))
```

---

## Scenario vs Unit Test Decision (Clojure)

| Use | When |
|-----|------|
| `deftest` + `testing` with Given/When/Then | Public API behavior, user-facing features, spec verification |
| `deftest` + `is` (unit style) | Internal functions, data transformations, pure function properties |

**BDD scenarios** test behavior through the public API.
**Unit tests** test implementation details of internal functions.

Both use `clojure.test` -- the difference is in what they exercise and how they're structured.
