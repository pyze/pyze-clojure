---
name: repl-driven-development
description: "Use when writing, modifying, or debugging any Clojure code. Establishes the default development loop: evaluate each function at the REPL immediately after writing it, never batch-implement and validate at the end."
---

# REPL-Driven Development

Build incrementally with immediate feedback. REPL for exploration and verification, spec-driven testing for implementation.

**REPL exploration is safe during any PDCA phase** — it never changes production code. Use it freely to validate assumptions, discover data shapes, and test ideas before writing code.

**Primary tool**: clojure_eval (MCP tool). See [clojure-mcp-repl](../clojure-mcp-repl/) for mechanics.

## Core Principle

```
Explore at REPL → Implement → Verify at REPL → Write tests tracing to obligations
```

**REPL exploration**: Understand existing code, discover data shapes, experiment with approaches, debug live state.

**Spec-driven implementation**: For features, implement to satisfy spec obligations, then write tests tracing to each `allium:propagate` obligation. For bug fixes, write the failing test first (TDD), then implement the minimal fix.

```clojure
;; Verify at the REPL immediately after implementing
(defn my-fn [x] (* x 2))
(my-fn 21)   ; => 42 — verify before moving on
(my-fn 0)    ; => 0 — check edge cases
```

---

## Skill Boundary

**Use DIFFERENT skill if:**
- clojure_eval MCP tool mechanics → [clojure-mcp-repl](../clojure-mcp-repl/)
- Bug fix TDD workflow → `superpowers:test-driven-development`
- REPL as search tool → [repl-semantic-search](../repl-semantic-search/)

---

## Default Workflow (compressed cycle)

Most development follows this cycle:

1. **Explore** — Load namespaces, examine data shapes, test assumptions at REPL
2. **Develop** — Implement → verify at REPL → write tests tracing to spec obligations
3. **Test** — Run JVM tests for the affected namespace
4. **Verify** — Browser validation (if UI), run full test suite
5. **Lint** — clj-kondo, fix warnings

### When to expand

Use the full cycle (see [FULL_CYCLE.md](./FULL_CYCLE.md)) when ANY of these apply:
- Production-ready feature visible to users
- Multiple components or external services involved
- UI interactions or visual elements
- Unclear requirements or multiple approaches

### Phase 0: Brainstorming

Before starting, consider `superpowers:brainstorming` when requirements are unclear or multiple approaches exist.

---

## Development Environment

### Use clojure_eval for all REPL interaction

```clojure
;; Function invocation
(my.namespace/my-fn arg1 arg2)

;; Requiring namespaces
(require '[my.app.store :as store])

;; Helper commands
(dir my.namespace)           ; List vars
(source my.ns/my-fn)        ; Show source
(doc my.ns/my-fn)           ; Show docs
```

### Which context?

| Code type | Context |
|-----------|---------|
| Backend (.clj), Ring handlers | JVM (default) |
| ClojureScript (.cljs), DOM, browser events | Browser — `(shadow/repl :app)` first |
| Uncertain | Check for `js/*` → browser, JVM libs → JVM |

### System management

See [integrant-lifecycle](../integrant-lifecycle/) for full details. Key command: `(in-ns 'user) (reset)` — reloads changed code + restarts.

| Scenario | Command |
|----------|---------|
| Changed handler/route code | `(reset)` |
| Changed Integrant init-key | `(reset)` |
| Changed protocol/defrecord | `(reset-all)` |
| Added new dependency | Restart JVM |

### Verification as done criterion

REPL green = done. Unvalidated = not done. After touching server-side Clojure files (`.clj`) or EDN resources, run `(reset)` and verify the changed code compiles and behaves correctly before declaring the task complete.

This applies to both main sessions and dispatched agents. An agent that implements changes but skips `(reset)` has not completed its task. Do not defer REPL validation to a separate step or session — it costs seconds and prevents breakage from compounding across sessions.

---

## Design for REPL-Friendly Systems

Building systems that are inherently observable starts with design choices, not bolted-on tooling.

### Design for Printability

- **Maps over opaque objects** — `(keys x)` at the REPL beats reading source to understand shape
- **Error messages as data** — name what went wrong and what was expected, not prose explanations

### Design for REPL-Friendly Observability

- **`tap>` + Portal over logging** — observable from the REPL, filterable as data, no log level ceremony
- **Functions callable in isolation** — if you can't call it at the REPL without standing up half the system, it has too many coordination points
- **`deref`-able state over opaque service objects** — the REPL can see inside atoms and delays
- **Data literals over formatted strings** — compose with REPL tools (`keys`, `select-keys`, `frequencies`)

**Preference hierarchy**: REPL-inspectable > `tap>`/Portal > structured logging > println > external dashboard

---

## Additional Resources

- [FULL_CYCLE.md](./FULL_CYCLE.md) — Full 13-phase methodology for complex features
- [PHASE_GUIDE.md](./PHASE_GUIDE.md) — Detailed checklists for all phases
- [PATTERNS.md](./PATTERNS.md) — Common patterns and anti-patterns
