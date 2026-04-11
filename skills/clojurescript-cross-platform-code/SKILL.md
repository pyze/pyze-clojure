---
name: clojurescript-cross-platform-code
description: Write cross-platform ClojureScript code for JVM TDD. Use when deciding file extensions, using reader conditionals, or enabling JVM testing.
---

# ClojureScript Cross-Platform Code

## When to Use This Skill

- [ ] Writing code that runs in both Clojure (JVM) and ClojureScript (browser)
- [ ] Deciding between `.clj`, `.cljs`, and `.cljc` file extensions
- [ ] Using reader conditionals (`#?(:cljs ...)`)
- [ ] Enabling JVM-based unit testing for frontend code
- [ ] Refactoring browser-specific code for testability

## Core Principle: JVM TDD

**Goal**: All business logic, state management, views, and actions must be testable on the JVM without requiring a browser.

**Why**:
- JVM tests are faster (no browser startup)
- JVM tests are more reliable (no flaky DOM timing)
- JVM tests integrate with standard Clojure tooling
- Enables REPL-driven development for all code

## File Extension Decision Tree

```
Is this code browser-only (DOM, window, etc.)?
├─ YES → Use .cljs
│        Examples: init.cljs, browser-effects.cljs
│
└─ NO → Is this code JVM-only (file I/O, sockets)?
        ├─ YES → Use .clj
        │        Examples: api.clj, loader.clj
        │
        └─ NO → Use .cljc (DEFAULT)
                Examples: views.cljc, actions.cljc, schema.cljc
```

## Layer-by-Layer Guidelines

### Actions (`.cljc` - Pure)

Actions MUST be platform-agnostic. They receive db and params, return effect vectors.

```clojure
;; CORRECT - Pure function, no platform code
(defn select-item-action [_db {:keys [item-id]}]
  [[:store/merge! {:ui/id :singleton :selected item-id}]
   [:ir/fetch-item! {:id item-id}]])

;; WRONG - Platform-specific call in action
(defn select-item-action [_db {:keys [item-id]}]
  #?(:cljs (js/console.log "Selected:" item-id))  ; NO!
  [...])
```

### Views (`.cljc` - Pure Hiccup Data)

Views MUST return pure hiccup data. Events return effect vectors, not function callbacks.

```clojure
;; CORRECT - Pure data, events return effect vectors
(defn item-list [{:keys [items]}]
  [:ul
   (for [item items]
     ^{:key (:id item)}
     [:li {:on {:click [[:item/select! {:id (:id item)}]]}}
      (:name item)])])

;; WRONG - Function callback (not pure data)
(defn item-list [{:keys [items on-select]}]
  [:ul
   (for [item items]
     [:li {:on {:click #(on-select (:id item))}}  ; NO - function
      (:name item)])])

;; WRONG - Direct DOM manipulation
(defn item-list [{:keys [items]}]
  [:ul {:ref #(when % (.focus %))}  ; NO - DOM in view
   ...])
```

### State/Schema (`.cljc` - Pure Data)

State schemas and entity definitions are pure data. Always cross-platform.

```clojure
;; CORRECT - Initialize state with schema function
(defn initial-db []
  (schema/initial-db))
```

### Effects (`.cljc` with splits OR `.cljs`)

Effects isolate I/O. Two patterns:

**Pattern 1: Split Implementation (`.cljc`)**
```clojure
;; Cross-platform effect with platform-specific implementations
(defn- fetch-impl [url on-success on-error]
  #?(:cljs (-> (js/fetch url) ...)
     :clj  (try (on-success (slurp url)) (catch Exception e (on-error e)))))

(defn fetch-effect [ctx _system {:keys [url]}]
  (fetch-impl url
              #((:dispatch ctx) [[:data-loaded! %]])
              #((:dispatch ctx) [[:error! %]])))
```

**Pattern 2: Browser-Only Effect (`.cljs`)**
```clojure
;; When effect is inherently browser-only
(defn scroll-to-effect [ctx _system {:keys [element-id]}]
  (when-let [el (js/document.getElementById element-id)]
    (.scrollIntoView el #js {:behavior "smooth"})))
```

### Init (`.cljs` - Browser Bootstrap)

App initialization is inherently browser-specific. Use `.cljs`.

```clojure
;; init.cljs - Browser-only
(ns myapp.init
  (:require [replicant.dom :as d]
            [myapp.views :as views]))

(defn ^:export init []
  (d/render (js/document.getElementById "app")
            [views/app {:db @app-state}]))
```

## Reader Conditionals

**Minimize reader conditional usage.** Structure files so `.cljc` code is pure and platform-free. See [CODE-ORGANIZATION.md — Minimize Reader Conditionals](../clojure-coding-standards/CODE-ORGANIZATION.md) for strategies.

When you must use reader conditionals, follow these patterns:

### Imports
```clojure
(ns myapp.effects
  (:require [clojure.edn :as edn])
  #?(:clj (:import [java.net URLEncoder])))
```

### Function Bodies
```clojure
(defn encode-uri [s]
  #?(:cljs (js/encodeURIComponent s)
     :clj  (URLEncoder/encode s "UTF-8")))
```

### Logging
```clojure
(defn log-error [msg]
  #?(:cljs (js/console.error msg)
     :clj  (println "ERROR:" msg)))
```

## Testing on JVM

With platform-agnostic code, JVM tests are straightforward:

```clojure
(ns myapp.actions-test
  (:require [clojure.test :refer [deftest is testing]]
            [myapp.actions :as actions]
            [myapp.state.schema :as schema]))

(deftest select-item-test
  (testing "select-item-action returns fetch effect"
    (let [db (schema/initial-db)
          effects (actions/select-item-action db {:item-id "123"})]
      (is (= :store/merge! (ffirst effects)))
      (is (= :ir/fetch-item! (first (second effects)))))))
```

## Common Pitfalls

| Mistake | Fix |
|---------|-----|
| `js/console.log` in action | Move to effect or use logging abstraction |
| `js/document` in view | Pass callback prop, dispatch effect |
| `.cljs` file for shared logic | Use `.cljc` with reader conditionals if needed |
| All effects in `.cljs` | Only browser-specific effects need `.cljs` |
| `[component props]` in hiccup | Check your UI framework's component invocation pattern |
| Plain `{}` for initial state | Use your schema's `initial-db` for proper initialization |

## Verification

Before committing, verify cross-platform compatibility:

```bash
# Load all .cljc files on JVM
clj -e "(require '[myapp.views] '[myapp.actions] '[myapp.state.schema])"

# Run JVM tests
clj -M:test

# Check for unintended platform dependencies
clj-kondo --lint src
```
