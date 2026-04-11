---
name: pyramid-state-management
description: Manage normalized state with Pyramid. Use when adding/modifying entities, querying state, understanding entity identity and normalization, or writing tests with store operations.
---

# Pyramid State Management

Normalized store for application state. Single source of truth.

**Use THIS skill if:**
- Adding or modifying entities in the store
- Querying state
- Understanding entity identity and normalization
- Writing tests that involve store operations

## Core Concepts

Pyramid is a **normalized store** backed by an atom. Entities are maps with a namespaced `:*/id` key. Pyramid automatically normalizes nested entities into a flat lookup table keyed by their ident.

**Maven**: `town.lilac/pyramid {:mvn/version "3.3.0"}`

### Custom Ident Function

Projects should use a custom ident function that only normalizes maps containing known entity keys. This prevents Pyramid from treating arbitrary maps with `:id` keys as entities.

```clojure
(require '[pyramid.core :as p])

(defn my-ident
  "Custom ident function — only normalizes maps with recognized entity keys."
  [m]
  (when (map? m)
    (some (fn [k]
            (when-let [v (get m k)]
              [k v]))
          [:doc/id :control/id :data/id :layout/id :ui/id])))  ;; add your entity keys

(defn initial-db []
  (p/db [my-ident] {}))
```

---

## Core API

| Function | Signature | Purpose |
|----------|-----------|---------|
| `initial-db` | `()` | Create empty db with custom ident fn |
| `add-entity` | `(db entity)` | Add/merge entity (wraps `pyramid.core/add`) |
| `remove-entity` | `(db ident)` | Delete entity + dangling refs by ident vector |
| `pull-entity` | `(db ident query)` | Pull attributes for one ident, returns map or nil |

### Examples

```clojure
;; Create a fresh database
(def db (initial-db))

;; Add an entity
(def db2 (add-entity db {:doc/id "sales" :doc/title "Sales Q1"}))

;; Pull attributes
(pull-entity db2 [:doc/id "sales"] [:doc/id :doc/title])
;; => {:doc/id "sales" :doc/title "Sales Q1"}

;; Remove an entity
(def db3 (remove-entity db2 [:doc/id "sales"]))
```

---

## Entity Model

All state follows the `:*/id` convention:

```clojure
;; Documents
{:doc/id "sales-q1" :doc/title "Sales Q1" :doc/content "..."}

;; Controls (UI-bound variables)
{:control/id :my-ns/region :control/type :select
 :control/value "East" :control/options ["East" "West"]}

;; Data cache
{:data/id "data-file.edn" :data/rows [{:month "Jan" :revenue 100}]}

;; UI singleton
{:ui/id :singleton :ui/current-doc "sales-q1"}
```

---

## State Update Patterns

Components should never mutate the store directly. State updates should flow through an action/effect system:

```
User action -> pure action fn -> effect vector -> store update
```

```clojure
;; Single entity update
(defn set-tab-action [_db {:keys [section-id panel]}]
  [[:store/merge! {:tab/id section-id :tab/active-panel panel}]])

;; Multiple entities in one effect
(defn update-action [db {:keys [doc-id]}]
  (let [entities [entity1 entity2]]
    [(into [:store/merge!] entities)]))
```

---

## Critical Anti-Patterns

### 1. Always Use `initial-db` in Tests

**ALWAYS test with your project's `initial-db`, NEVER with plain `{}`.**

```clojure
;; WRONG - uses default ident function, different normalization
(let [db (pyramid.core/db [] {})]
  (add-entity db {:doc/id "test" ...}))

;; CORRECT - uses custom ident function
(let [db (initial-db)]
  (add-entity db {:doc/id "test" ...}))
```

Plain `{}` uses Pyramid's default ident function which normalizes ANY map with an `:id`-like key. Custom ident functions only normalize maps with recognized entity keys. Testing with the wrong ident function produces incorrect normalization behavior.

### 2. Use `get-in` for Non-Entity Nested Data (NOT `p/pull`)

`p/pull` uses EQL internally, which treats 2-element keyword vectors like `[:hiccup :vars]` as **lookup refs** and converts them to maps `{:hiccup :vars}`. It also corrupts lists.

**Use `get-in` to retrieve deeply-nested non-entity data** like ASTs or complex nested structures:

```clojure
;; WRONG — p/pull corrupts keyword vectors and lists inside nested data
(defn get-ast [db doc-id]
  (:doc/ast (pull-entity db [:doc/id doc-id] [:doc/ast])))

;; CORRECT — get-in retrieves raw data without EQL interpretation
(defn get-ast [db doc-id]
  (get-in db [:doc/id doc-id :doc/ast]))
```

Use `p/pull` for flat entity attributes; use `get-in` when the value contains lists, parameterized queries, or 2-element keyword vectors that could be misinterpreted as lookup refs.

---

## REPL Inspection

Quick commands to explore Pyramid state at the REPL:

```clojure
;; See all entity idents (top-level keys are ident vectors)
(keys db)
;; => ([:doc/id "sales-q1"] [:control/id :my-ns/region] [:ui/id :singleton] ...)

;; Inspect a specific entity's structure
(p/pull db [:doc/id "sales-q1"] [:doc/id :doc/title :doc/content])
;; => {:doc/id "sales-q1" :doc/title "Sales Q1" :doc/content "..."}
```
