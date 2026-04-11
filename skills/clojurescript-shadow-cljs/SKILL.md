---
name: clojurescript-shadow-cljs
description: Build ClojureScript with Shadow-CLJS compilation and hot reload. Use when debugging build issues, configuring compilation, or understanding the build system.
---

# ClojureScript & Shadow-CLJS

Shadow-CLJS handles ClojureScript compilation, hot reload, and development server.

## When to Use This Skill

**Use THIS skill if:**
- ✅ Configuring Shadow-CLJS build settings
- ✅ Debugging compilation or hot reload issues
- ✅ Setting up nREPL or browser REPL
- ✅ Understanding .cljs/.clj/.cljc file extensions
- ✅ Troubleshooting stale code or cache issues

**Use DIFFERENT skill if:**
- REPL development workflow → [repl-driven-development](../repl-driven-development/)

## Core Concepts

### Configuration

All configuration in `shadow-cljs.edn`:

```clojure
{:deps {:aliases [:dev]}           ; Use deps.edn dependencies
 :nrepl {:port 7889}               ; nREPL port (standard: 7889)
 :dev-http {8081 {...}}            ; Dev server config
 :builds {:app {...}}}             ; Build targets
```

**Standard Ports**:
| Port | Purpose | Config Location |
|------|---------|-----------------|
| **7889** | nREPL | `shadow-cljs.edn` → `:nrepl {:port N}` |
| **8081** | Dev server | `shadow-cljs.edn` → `:dev-http {N {...}}` |
| **3000** | Backend API | `resources/config/system.edn` → `:server/http {:port N}` |

**Note**: Always verify ports in config files. Use discovery commands in build/CLAUDE.md if unsure.

### Build Target

```clojure
{:builds
 {:app {:target :browser
        :output-dir "resources/public/js"
        :asset-path "/js"
        :modules {:main {:init-fn myapp.init/init-app}}
        :devtools {:after-load myapp.init/init-app}}}}
```

### Dev Server

```clojure
{:dev-http
 {8081                             ; Port (discover here!)
  {:root "resources/public"        ; Static files
   :proxy-url "http://localhost:3000"  ; Backend proxy
   :push-state/index "index.html"}}}   ; SPA routing
```

## Discovery Patterns

```bash
# Find dev server port
grep -A2 ":dev-http" shadow-cljs.edn

# Find nREPL port
grep -A1 ":nrepl" shadow-cljs.edn

# Find build entry point
grep "init-fn" shadow-cljs.edn

# Find hot reload function
grep "after-load" shadow-cljs.edn
```

## Development Commands

### Starting the Dev Environment

**Ensure your development environment is running before using these commands.**

```bash
# Install npm dependencies (first time only)
npm install

# CRITICAL: Export env vars BEFORE starting Shadow-CLJS
# JVM inherits environment at startup time
source .env.local 2>/dev/null && export GOOGLE_API_KEY

# Start watch mode (includes nREPL server)
./bin/shadow-watch &

# Wait for compilation, then create port file
sleep 30
echo "9889" > .nrepl-port
```

### Claude's Tool: clojure_eval MCP Tool

**Claude MUST use the clojure_eval MCP tool for all REPL operations. Never start new REPL processes.**

See [clojure-mcp-repl](../clojure-mcp-repl/) for complete reference.

```clojure
;; Verify nREPL is running (should already be started by user)
(+ 1 1)

;; JVM code evaluation
(in-ns 'user)
(start)

;; ClojureScript evaluation (switch context first)
(shadow/repl :app)
@some.namespace/state
```

### Build Commands (rarely needed)

```bash
# One-time compile
npx shadow-cljs compile app

# Production build
npx shadow-cljs release app

# Clear cache (if issues)
rm -rf .shadow-cljs/
```

## Hot Reload

Shadow-CLJS automatically reloads code when files change.

### after-load Hook

```clojure
;; In shadow-cljs.edn
:devtools {:after-load myapp.init/init-app}

;; The function is called after each reload
(defn init-app []
  (d/render (js/document.getElementById "app")
            (views/app {} @db)))
```

### Preserving State (CRITICAL)

Use `defonce` with a nil-check guard. `defonce` alone is NOT sufficient -- after-load hooks can still reset state.

```clojure
;; CORRECT - defonce + nil-check guard
(defonce app-state (atom nil))

(defn init-app []
  (when (nil? @app-state)           ;; Only init if not already initialized
    (reset! app-state (initial-db)))
  (render! app-state))

;; WRONG - regular def resets on every reload
(def this-resets-on-reload "value")
```

**See memories/** for related learnings on hot reload patterns.

## Build Hooks (uix.css)

```clojure
{:dev {:build-hooks [(uix.css/hook)]}
 :release {:build-hooks [(uix.css/hook)]}}
```

## Common Issues

### Stale Code After Changes

```bash
# Clear Shadow-CLJS cache
rm -rf .shadow-cljs/

# Restart watch
./bin/shadow-watch
```

### Compilation Warnings

Shadow-CLJS warnings often indicate real problems. Common ones:

| Warning | Cause | Fix |
|---------|-------|-----|
| Undeclared Var | Missing require | Add to `:require` |
| Deprecated | Old API | Update to new API |
| Arity mismatch | Wrong arg count | Check function signature |

### Hot Reload Not Working

1. Check `after-load` function exists and is correct
2. Check for compilation errors in terminal
3. Try hard refresh in browser
4. Clear `.shadow-cljs/` and restart

### REPL Connection Issues

```clojure
;; In shadow-cljs REPL, switch to ClojureScript
(shadow/repl :app)

;; If disconnected
(shadow/watch :app)
(shadow/repl :app)
```

## Browser REPL (via clojure_eval MCP Tool)

**MANDATORY**: Always use the clojure_eval MCP tool to connect to the existing Shadow-CLJS nREPL. Never start a new REPL process.

```clojure
;; WRONG: Don't start a new REPL process via Bash
;; npx shadow-cljs cljs-repl app  # NO!

;; CORRECT: Use clojure_eval MCP tool to connect to existing nREPL
;; First, switch to ClojureScript context
(shadow/repl :app)

;; Then execute ClojureScript
(js/console.log "Hello from REPL!")
@my.app/app-state
```

### Why clojure_eval MCP Tool?

- **Native MCP integration**: Direct tool support in Claude Code
- **Clean syntax**: Standard Clojure s-expressions
- **Helper functions available**: `(dir ns)`, `(source fn)`, `(doc fn)`
- **Existing connection**: Uses the already-running Shadow-CLJS nREPL

## Production Build

```bash
# Create optimized build
npx shadow-cljs release app

# Output in resources/public/js/
# - main.js (minified, dead-code eliminated)
```

### Release Configuration

```clojure
{:release
 {:build-hooks [(uix.css/hook)]
  :compiler-options {:optimizations :advanced}}}
```

## File Extension Reference

| Extension | When to Use |
|-----------|-------------|
| `.cljs` | Browser-only code (DOM, js/) |
| `.clj` | JVM-only code (Ring, Java interop) |
| `.cljc` | Shared code (pure Clojure, reader conditionals) |

### Reader Conditionals

```clojure
;; In .cljc file
(defn current-time []
  #?(:cljs (.now js/Date)
     :clj  (System/currentTimeMillis)))
```

## Anti-Patterns

### REPL Anti-Patterns (CRITICAL)

```bash
# WRONG: Starting a new REPL process
clj -M:dev                      # NO - don't start new REPL
lein repl                       # NO - don't start new REPL
npx shadow-cljs cljs-repl app   # NO - don't start new REPL

# WRONG: Interactive REPL sessions
# Don't open a REPL and type commands interactively

# WRONG: Using deprecated nvk tool
nvk my.namespace my-fn arg1 arg2  # NO - nvk is deprecated
```

```clojure
;; CORRECT: Always use clojure_eval MCP tool
(my.namespace/my-fn arg1 arg2)
(your-code-here)

;; CORRECT: For ClojureScript, switch context first
(shadow/repl :app)
(your-cljs-code)
```

### Code Anti-Patterns

```clojure
;; WRONG: Using def for state (resets on reload)
(def my-state (atom {}))

;; CORRECT: Use defonce
(defonce my-state (atom {}))

;; WRONG: Hardcoded port
"http://localhost:8081/..."

;; CORRECT: Relative path (proxy handles it)
"/api/..."

;; WRONG: js/ in .clj file
(js/console.log "x")  ; Won't compile

;; CORRECT: Use reader conditional or .cljs file
#?(:cljs (js/console.log "x"))
```
