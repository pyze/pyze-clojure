---
name: clojure-coding-standards
description: Comprehensive Clojure code quality standards including functional programming principles, code organization guidelines, and collection transformation patterns - use for writing maintainable, testable, and performant Clojure code. Also covers immutability, pure functions, bang suffix conventions, state-based coordination, transducers, function/namespace size limits, and refactoring triggers.
version: 2.0.0
---

# Clojure Coding Standards Skill

Unified code quality standards for writing excellent Clojure code.

## Overview

This skill consolidates all Clojure code quality guidance:
- **Functional programming principles** - Immutability, purity, composition
- **Code organization** - Function/namespace size guidelines
- **Collection patterns** - Transducer-based transformations
- **Approval policies** - When deviations require user permission

## Documents

| Document | Covers |
|----------|--------|
| [IDIOMS.md](./IDIOMS.md) | Threading macros, control flow, destructuring, anti-patterns |
| [FUNCTIONAL-PRINCIPLES.md](./FUNCTIONAL-PRINCIPLES.md) | Immutability, pure functions, error handling |
| [CODE-ORGANIZATION.md](./CODE-ORGANIZATION.md) | Function/namespace size limits, refactoring triggers |
| [COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md) | Transducers, lazy evaluation |

## Quick Navigation

- **[IDIOMS.md](./IDIOMS.md)** - Threading macros, control flow, destructuring
- **[FUNCTIONAL-PRINCIPLES.md](./FUNCTIONAL-PRINCIPLES.md)** - FP patterns (immutability, purity, error handling)
- **[CODE-ORGANIZATION.md](./CODE-ORGANIZATION.md)** - Size guidelines, refactoring triggers
- **[COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md)** - Transducers, prohibited patterns

---

## Core Principle: Quality by Default

**Default to high-quality code.** Deviations from standards require explicit approval and documentation.

### No Backward Compatibility by Default

**Policy**: `fail fast > fallback`. Unless explicitly requested, there is no need for backward compatibility.

When replacing a v1 with v2:
- Delete v1 entirely — do not maintain both
- Remove all v1 routes, handlers, tests, and dead code
- No deprecation shims, re-exports, or compatibility layers

This aligns with the error handling hierarchy in `error-handling-patterns/`:
`fail fast > fallback > backward compatibility`

---

## Unified Approval Policy

**The following require explicit user approval and documentation:**

### 1. Mutable State — STRICT APPROVAL REQUIRED

**⚠️ CRITICAL: Any use of mutable state without explicit user approval is a VIOLATION of coding standards.**

**Covered constructs (ALL require user approval before introduction):**
- `atom` / `swap!` / `reset!`
- `volatile!` / `vswap!` / `vreset!`
- `ref` / `dosync` / `alter` / `commute`
- `agent` / `send` / `send-off`
- `defonce` with mutable values
- JS interop mutable state (`set!`, `.push`, mutable JS objects)

**Approval workflow** (per `decomplection-first-design`):
1. **STOP** — Do not write the mutable code yet
2. **ASK** — Present the atom/ref to the user with a justification
3. **JUSTIFY** — Explain why a pure design won't work for this case
4. **WAIT** — The user decides. Do not self-approve.
5. **DOCUMENT** — After user approves, include approval comment in code

**Without user approval = VIOLATION. Do not proceed.**

```clojure
;; ❌ VIOLATION - Using atom without user approval
(def cache (atom {}))  ; STOP: Must ask user first!

;; ❌ VIOLATION - defonce with atom without approval
(defonce db (atom {}))  ; STOP: Must ask user first!

;; ✅ WITH USER APPROVAL - Document in code
;; MUTATION APPROVED (2025-01-31) by [user]:
;; Benchmark cache-performance-001 showed 10x improvement with atom-based cache
;; vs rebuilding immutable map on each update (1000 updates/sec workload).
;; Pure design attempted first but showed O(N) rebuild on each update.
;; Isolated to cache module boundary.
(def cache (atom {}))
```

### 2. Lazy Evaluation Boundary

**Different rules for different contexts:**

**In-memory Clojure collections (use transducers):**
```clojure
;; ❌ PROHIBITED - lazy evaluation for in-memory collections
(->> users (map :id) (filter active?))

;; ✅ REQUIRED - transducers for in-memory
(into [] (comp (map :id) (filter active?)) users)
```

**Arrow streaming results (lazy OK):**
```clojure
;; ✅ OK - lazy for Arrow query streaming (see arrow-performance skill)
(defn query-results-lazy [storage query]
  (lazy-seq ...))  ; Defers materialization until consumer needs it
```

**Large file streaming (requires approval):**
```clojure
;; ❌ REQUIRES APPROVAL - lazy for streaming large datasets
;; PERFORMANCE DEVIATION (approved 2025-01-31):
;; Streaming 10GB file, transducer would require loading entire file.
;; Lazy approach processes incrementally with 200MB peak memory.
(defn process-large-file [path]
  (with-open [rdr (io/reader path)]
    (doall
      (into []
            (comp (map parse-line) (filter valid?))
            (line-seq rdr)))))
```

### 3. Dead Code Removal (REQUIRED - No Approval Needed)

**Always remove unused code.** This is a quality standard, not a deviation.

```clojure
;; ❌ DELETE - Unused function
(defn never-called-function [x] ...)

;; ❌ DELETE - Unused public var
(def unused-constant 42)

;; ❌ DELETE - Redundant test (when better test exists)
(deftest simple-addition-test
  (is (= 4 (+ 2 2))))
```

**Never prefix unused bindings with `_` to silence linter warnings.** Remove the dead code instead. Only use `_` prefix when the parameter is required by a function contract (callback arity, protocol method signature) and cannot be eliminated.

```clojure
;; ❌ BAD — silences linter but leaves dead code
(let [_unused-result (some-fn)]
  other-value)

;; ✅ GOOD — remove the binding entirely
(do (some-fn)  ; keep if side-effecting, delete if pure
    other-value)

;; ✅ OK — _ required by contract (e.g., callback arity)
(map (fn [_idx val] (process val)) (range) items)
```

**Exception: Check for carve false positives before deleting:**
- Code called dynamically (`apply-operator`, `resolve`, `var`)
- Callbacks registered elsewhere (event handlers, protocol extensions)
- Code matching `carve-tool-usage` false positive patterns

When in doubt, run `carve --interactive` and review each removal.

**Test prioritization (keep higher-value tests):**
```
Integration tests > Unit tests (for same coverage)
Property-based > Example-based (for same behavior)
Contract tests > Implementation-specific tests
```

---

## Approval Documentation Format

**When user approves a deviation:**

```clojure
;; MUTATION APPROVED (YYYY-MM-DD) by [user]:
;; [Justification — why pure design won't work]
;; [Scope — where it's isolated]
(deviation-code)
```

**Example:**

```clojure
;; MUTATION APPROVED (2025-01-31) by mark:
;; Cache must be mutable for hot-reload. 10x improvement vs immutable rebuild.
;; Isolated to cache module boundary; all reads go through get-cache.
(def query-cache (atom {}))

(defn update-cache! [k v]
  (swap! query-cache assoc k v))
```

---

## Decision Tree: When to Ask for Approval

```
┌─────────────────────────────────────────────────────────┐
│ Am I about to use: atom, ref, agent, volatile?          │
└─────────────────────────────────────────────────────────┘
           │
           ├─ YES → STOP: Ask user for approval
           │        Document with rationale + benchmark
           │
           └─ NO → Proceed

┌─────────────────────────────────────────────────────────┐
│ Am I using lazy evaluation (->, ->>, map, filter)?      │
└─────────────────────────────────────────────────────────┘
           │
           ├─ For in-memory collections? → Use transducers (required)
           │
           └─ For streaming/infinite data? → STOP: Ask user for approval
                                              Document with performance justification

┌─────────────────────────────────────────────────────────┐
│ Is this code unused (never called)?                     │
└─────────────────────────────────────────────────────────┘
           │
           └─ YES → DELETE immediately (no approval needed)
```

---

## Core Standards Summary

### Namespace and Require Conventions

1. **All requires in `ns` form** - Every `require` must be part of the `(ns ...)` declaration
2. **No dynamic require** - Never use `(require ...)` or `(require-resolve ...)` in source code
3. **REPL exception only** - Dynamic require/resolve is acceptable only at the REPL, never in committed source

```clojure
;; ❌ WRONG - dynamic require in source code
(defn init! []
  (require '[replicant.dom :as d])
  ((resolve 'd/set-dispatch!) dispatch-fn))

;; ❌ WRONG - require-resolve in source code
(defn load-handler []
  (when-let [f (requiring-resolve 'my.ns/handler)]
    (f)))

;; ✅ CORRECT - all requires in ns form
(ns my-app.init
  (:require [replicant.dom :as d]))

(defn init! []
  (d/set-dispatch! dispatch-fn))
```

### Functional Programming (see FUNCTIONAL-PRINCIPLES.md)

1. **Immutability by default** - Mutation requires approval
2. **Pure functions** - Deterministic, side-effect free (mark impure with `!`)
3. **Defaults at edges** - Resolve defaults at system boundaries, not in inner functions
4. **Bang suffix** - `!` = side effect, no `!` = pure (functions and effect keywords)
5. **State-based coordination** - Prefer semantic state over timing mechanisms
6. **Composition** - Build with functions/protocols, not inheritance
7. **Declarative style** - Express WHAT, not HOW; use transducers for collections
8. **Error handling** - See [error-handling-patterns](../error-handling-patterns/) for complete guidance

### Code Organization (see CODE-ORGANIZATION.md)

1. **Function size** - 10-40 LOC ideal, 150+ red flag
2. **Namespace size** - 200-400 LOC ideal, 1500+ split
3. **Refactoring triggers** - 2+ independent loops, 3+ nested lets
4. **Complexity metrics** - Track nesting depth, parameters, cyclomatic complexity

### Collection Patterns (see COLLECTION-PATTERNS.md)

1. **Use transducers with `into`** - Single-pass, no intermediate collections
2. **Compose with `comp`** - Right-to-left composition
3. **Never use `doall` or `vec` on lazy seqs** - Use `into` instead
4. **Use xforms library** - `xf/sort`, `xf/for`, `xf/reduce` for extended transducers
5. **Lazy only when necessary** - Large/infinite data or early termination (with approval)

---

## Quick Reference by Task

| Task | Standard | See |
|------|----------|-----|
| Transform collection | `(into [] (map f) coll)` | COLLECTION-PATTERNS.md |
| Multiple operations | `(into [] (comp (map f) (filter p)) coll)` | COLLECTION-PATTERNS.md |
| Sort with transform | `(into [] (comp (filter p) (xf/sort)) coll)` | COLLECTION-PATTERNS.md |
| Mutable state needed | Ask user approval, document | FUNCTIONAL-PRINCIPLES.md |
| Function too large | Extract helpers, use threading | CODE-ORGANIZATION.md |
| Error handling | Preserve context with `ex-info` | FUNCTIONAL-PRINCIPLES.md |
| Delete unused code | Remove immediately | FUNCTIONAL-PRINCIPLES.md |

---

## Integration with Other Skills

- **development-workflow** - Apply these standards during Develop/Critique phases
- **decomplection-first-design** - Use for architectural decisions before coding
- **`superpowers:systematic-debugging`** - Apply when debugging code quality issues
- **specification-first-development** - Define quality contracts in specifications

---

## Summary

1. **Immutability by default** - Mutation needs approval + documentation
2. **Delete dead code** - Unused functions, vars, redundant tests
3. **Pure functions** - Mark side effects with `!` suffix
4. **Defaults at edges** - Resolve defaults at system boundaries, not in inner functions
5. **Use transducers** - `into` with `comp` for collections
6. **Preserve exceptions** - Never transmute except at API boundaries
7. **Size guidelines** - Functions 10-40 LOC, namespaces 200-400 LOC
8. **Ask for approval** - Mutations, lazy evaluation deviations, performance trade-offs
9. **Document deviations** - Include approval date, rationale, benchmark ID

**For detailed guidance, see:**
- [FUNCTIONAL-PRINCIPLES.md](./FUNCTIONAL-PRINCIPLES.md)
- [CODE-ORGANIZATION.md](./CODE-ORGANIZATION.md)
- [COLLECTION-PATTERNS.md](./COLLECTION-PATTERNS.md)
