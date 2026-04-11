---
name: decomplection-clojure
description: "Clojure code examples for decomplection principles: composition, protocols, state management approval workflow"
---

Clojure companion to the decomplection-first-design skill (in pyze-workflow). This skill provides Clojure-specific code examples for the general principles.

## Four Core Patterns in Clojure

### Pattern 1: Extract Concerns with `comp`

```clojure
;; ENTANGLED: One function doing multiple things
(defn load-query [s]
  (let [parsed (parse s)]
    (validate parsed)))

;; DECOMPLECTED: Separately testable, reusable
(comp validate parse)
```

### Pattern 2: Protocols and Records

```clojure
;; ENTANGLED: Hard-coded implementation
(defn execute-query [q]
  (let [result (db/query q)]
    (cache/store result)  ;; Mixed concerns
    result))

;; DECOMPLECTED: Protocol abstraction
(defprotocol QueryExecutor
  (execute [this query]))

(defrecord DirectExecutor [db]
  QueryExecutor
  (execute [_ query] (db/query db query)))

(defrecord CachingExecutor [executor cache]
  QueryExecutor
  (execute [_ query]
    (or (cache/get cache query)
        (let [r (execute executor query)]
          (cache/store cache query r)
          r))))
```

### Pattern 3: Explicit Dependencies via Destructuring

```clojure
;; ENTANGLED: Hidden dependencies
(defn compile [q]
  (let [p (create-planner)        ;; Hardcoded
        config @global-config]    ;; Hidden!
    ...))

;; DECOMPLECTED: All dependencies explicit
(defn compile [q {:keys [planner optimizer]}]
  ;; No hidden state, testable with any inputs
  ...)
```

### Pattern 4: DDRY (Decomplected Don't Repeat Yourself)

```clojure
;; BAD DRY: extracted shared code, but it does three things for two callers
(defn process-entity [entity mode]
  (let [validated (validate entity)
        enriched (if (= mode :admin) (add-audit validated) validated)
        result (if (= mode :admin) (save-with-log enriched) (save enriched))]
    result))

;; DDRY: composable pieces, each with one role
(defn validate [entity] ...)
(defn add-audit [entity] ...)
(defn save [entity] ...)
(defn save-with-log [entity] (save (add-audit entity)))

;; Callers compose what they need
(comp save validate)              ;; regular path
(comp save-with-log validate)     ;; admin path
```

---

## Mutable State Approval Workflow

**State is never simple.** It fundamentally complects value and time. Any mutable state (atom, volatile, ref, agent) is a decomplection violation unless explicitly approved by the user.

### Approval Process

1. **STOP** - Do not write the code yet
2. **ASK** - Present each atom to the user individually with a justification
3. **JUSTIFY** - Explain why a pure design won't work
4. **WAIT** - The user decides. Do not self-approve.
5. **DOCUMENT** - After user approves, include approval comment in code

### Approval Comment Format

```clojure
;; VIOLATION - atom without user approval
(defonce registry (atom {}))

;; VIOLATION - Claude self-approved (not valid)
;; MUTATION APPROVED: needed for state management
(defonce registry (atom {}))

;; AFTER USER APPROVAL
;; MUTATION APPROVED (2025-02-05) by mark:
;; Registry must be mutable for hot-reload component registration.
;; Isolated to registry module; all reads go through get-renderer.
;; Pure alternative considered: passing registry as arg -- rejected because
;; hot-reload requires stable reference across system restarts.
(defonce registry (atom {}))
```

---

## Side Effect Convention

Mark functions with side effects using the `!` suffix:

```clojure
;; Pure -- no suffix
(defn validate [entity] ...)
(defn transform [data] ...)

;; Side effects -- ! suffix
(defn save! [entity] ...)
(defn send-notification! [user msg] ...)
```

---

## Recognizing Entanglement in Clojure

**Red flags in Clojure code:**

```clojure
;; RED FLAG: Hidden dependency on global
(defn process-query [query]
  (let [schema @global-schema]  ;; Hidden!
    ...))

;; RED FLAG: Mixed concerns
(defn execute-query [query-string]
  (let [parsed (parse query-string)
        validated (validate parsed)
        result (execute validated)]
    result))

;; RED FLAG: Hard to test
(defn get-user [id]
  (or (cache-get id)
      (let [user (db-query id)]
        (cache-put id user)
        user)))
```

---

## Namespace Composition

Decomplected namespaces have one role each and compose at the edges:

```clojure
;; Each namespace has one concern
(ns my.app.validate)   ;; pure validation
(ns my.app.transform)  ;; pure transforms
(ns my.app.persist)    ;; IO boundary

;; Composition at the edge
(ns my.app.handler
  (:require [my.app.validate :as v]
            [my.app.transform :as t]
            [my.app.persist :as p]))

(defn handle [request]
  (-> request v/validate t/transform p/save!))
```

---

## Implementation Checklist

Before committing, verify:

- [ ] No hidden dependencies (all inputs explicit)
- [ ] No mixed concerns (one responsibility)
- [ ] Easy to test (simple inputs, no setup)
- [ ] Reusable (REPL, tests, multiple contexts)
- [ ] Composable (output feeds into other functions)
- [ ] Pure or boundary-marked (`!` suffix for side effects)
- [ ] Few coordination points (each component interacts with minimal others)
- [ ] Self-evident purpose (name and arity reveal intent without reading body)
