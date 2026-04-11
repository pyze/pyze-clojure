---
name: clojure-mcp-repl
description: Execute Clojure/ClojureScript code at the REPL using clojure-mcp (MCP server). Use when evaluating expressions, starting/stopping the backend, querying state, or running tests interactively.
---

# clojure-mcp REPL Skill

**clojure-mcp** is the primary tool for REPL-driven development. Use it for ALL Clojure evaluation via the native `clojure_eval` MCP tool.

**Always permitted**: clojure_eval is read-only exploration. It is allowed in every PDCA phase, including plan mode. It never changes production code.

**Source**: https://github.com/bhauman/clojure-mcp

## Skill Boundary

This skill covers: **clojure-mcp tool mechanics** - syntax, connection, commands.

**Use DIFFERENT skill if:**
- Workflow philosophy → [repl-driven-development](../repl-driven-development/)
- TDD methodology → `superpowers:test-driven-development`
- System lifecycle details → [integrant-lifecycle](../integrant-lifecycle/)

## Installation

```bash
# Requires: Clojure 1.12+
clojure -Ttools install-latest :lib io.github.bhauman/clojure-mcp :as mcp
```

---

## Quick Reference

### Direct Expression Evaluation (Primary Pattern)

The `clojure_eval` MCP tool evaluates arbitrary Clojure expressions directly. No `load-string` wrapper needed.

```clojure
clojure_eval: (+ 1 2 3)
;; → 6

clojure_eval: (str "hello" " " "world")
;; → "hello world"

clojure_eval: (clojure.string/upper-case "hello")
;; → "HELLO"

clojure_eval: (let [x 10] (* x x))
;; → 100
```

### Multi-Form Expressions

```clojure
clojure_eval: (do
                (require '[clojure.string :as str])
                (str/join ", " [1 2 3]))
;; → "1, 2, 3"
```

---

## nREPL Connection

### Automatic Connection (Recommended)

clojure-mcp connects to existing nREPL if `.nrepl-port` file exists:

```bash
# Copy shadow-cljs's dynamic nREPL port (NOT hardcoded!)
cp .shadow-cljs/nrepl.port .nrepl-port
```

Then use `clojure_eval` as normal. It automatically discovers the port from `.nrepl-port`.

### Manual Port Selection

The `clojure_eval` tool accepts an optional `port` parameter:

```clojure
clojure_eval (port: 7888): (+ 1 2)
```

### Performance

| Mode | Startup Time | Use Case |
|------|--------------|----------|
| Connected to nREPL | ~0.3s | Development (preferred) |
| Fresh process | ~6s | One-off scripts, CI |

---

## System Management (integrant-repl)

**CRITICAL**: The JVM running your nREPL is your development environment. NEVER kill it. Use `reset` to apply code/config changes without JVM restart.

```clojure
# Start system (if not running)
clojure_eval: (in-ns 'user) (go)

# Stop system (keeps JVM running)
clojure_eval: (in-ns 'user) (halt)

# PREFERRED: Reload changed code + restart
clojure_eval: (in-ns 'user) (reset)

# Full reload (use if reset fails, e.g., protocol changes)
clojure_eval: (in-ns 'user) (reset-all)

# Access running system state
clojure_eval: integrant.repl.state/system
clojure_eval: integrant.repl.state/config
```

**Key insight**: `reset` reloads changed `.clj`/`.cljc` files THEN restarts. Always prefer `reset` over manual stop+start.

### CRITICAL: When to Run `(reset)`

**ALWAYS run `(reset)` after modifying `.clj` or `.cljc` files.** Server-side Clojure does NOT hot-reload like ClojureScript.

| File Type | Hot Reload? | Action After Edit |
|-----------|-------------|-------------------|
| `.clj` | **NO** | Run `(reset)` |
| `.cljc` | Browser: YES, Server: **NO** | Run `(reset)` |
| `.cljs` | YES (Shadow-CLJS) | None needed |

**Common mistake**: Editing server-side code, then testing without running `(reset)`. The old code is still active. This wastes time debugging unchanged code.

**Rule**: After ANY `.clj` or `.cljc` edit, immediately run:
```clojure
clojure_eval: (in-ns 'user) (reset)
```

---

## Common Patterns

### Inspecting State

```clojure
clojure_eval: @my.ns/my-atom

clojure_eval: (keys @my.ns/my-atom)

clojure_eval: (clojure.pprint/pprint @my.ns/my-atom)
```

### Requiring Namespaces

```clojure
clojure_eval: (require 'my.namespace)

clojure_eval: (require '[my.namespace :as ns])

clojure_eval: (require 'my.namespace :reload)
```

### Running Tests

```clojure
clojure_eval: (test-my-fn)

clojure_eval: (clojure.test/run-tests 'my.namespace-test)
```

---

## REPL Exploration

Before using unfamiliar functions, explore them:

### List functions in a namespace
```clojure
clojure_eval: (do
                (require '[clojure.repl :refer [doc dir apropos source]])
                (dir clojure.string))
;; Shows: blank?, capitalize, join, split...
```

### Get documentation
```clojure
clojure_eval: (doc map)
;; Shows arglists, description
```

### Search by name pattern
```clojure
clojure_eval: (apropos "split")
;; Finds: split-at, split-with, clojure.string/split
```

### Search documentation text
```clojure
clojure_eval: (find-doc "thread")
;; Finds functions mentioning "thread"
```

### Read source code
```clojure
clojure_eval: (source filter)
;; Shows implementation
```

### Check arglists programmatically
```clojure
clojure_eval: (:arglists (meta #'reduce))
;; => ([f coll] [f val coll])
```

---

## ClojureScript REPL (shadow-cljs)

For ClojureScript browser code, use shadow-cljs's `cljs-eval` API:

```bash
# Syntax: (cljs-eval build-id code-string opts-map)
npx shadow-cljs clj-eval '(shadow.cljs.devtools.api/cljs-eval :app "(+ 1 2)" {})'
# → {:results ["3"], :out "", :err "", :ns cljs.user}
```

### Escaping Tips

- Use single quotes for outer clj-eval string
- Use double quotes for inner CLJS code string
- Use `(quote [...])` instead of `'[...]`

---

## Quick Reference Table

| Task | Command |
|------|---------|
| Expression eval | `clojure_eval: (+ 1 2 3)` |
| Multi-form | `clojure_eval: (do (require ...) (fn ...))` |
| Inspect state | `clojure_eval: @my.ns/atom` |
| Start system | `clojure_eval: (in-ns 'user) (go)` |
| Stop system | `clojure_eval: (in-ns 'user) (halt)` |
| **Reload + restart** | `clojure_eval: (in-ns 'user) (reset)` |
| Full reload | `clojure_eval: (in-ns 'user) (reset-all)` |
| **CLJS eval** | `npx shadow-cljs clj-eval '(shadow.cljs.devtools.api/cljs-eval :app "code" {})'` |

---

## Troubleshooting

See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for:
- Connection issues
- Startup error diagnosis
- State persistence gotchas
