---
name: integrant-lifecycle
description: Manage system lifecycle with Integrant dependency injection. Use when adding backend services, configuring system.edn, or understanding service naming.
---

# Integrant Lifecycle

Integrant manages system lifecycle (start/stop) for backend services.

## When to Use This Skill

**Use THIS skill if:**
- ✅ Adding a new service to the system
- ✅ Configuring service dependencies
- ✅ Understanding service naming conventions (slash notation)
- ✅ Debugging service startup/shutdown issues
- ✅ Working with Aero environment configuration

**Use DIFFERENT skill if:**
- Error handling in services → [error-handling-patterns](../error-handling-patterns/)
- REPL system management → [repl-driven-development](../repl-driven-development/)

## Core Concepts

### System Configuration

Configuration lives in `resources/config/system.edn`:

```clojure
{:server/http {:port 3000
               :handler #ig/ref :app/handler}

 :app/handler {:routes #ig/ref :app/routes}

 :app/routes {}}
```

### Service Naming (CRITICAL) - Authoritative Source

**This is the authoritative documentation for Integrant service naming.**

**ALWAYS use slash notation** for Integrant keys:

```clojure
;; CORRECT
:server/http
:auth/service
:db/connection
:firebase/user-service
:gemini/service

;; WRONG - causes nil lookup errors
:server-http
:auth-service
:firebase-user-service
```

### Slash Notation Patterns

| Pattern | Example | When to Use |
|---------|---------|-------------|
| **Domain/service** | `:auth/service` | Single service per domain |
| **Provider/service** | `:firebase/user-service` | Multiple services from same provider |
| **Type/instance** | `:db/read-pool` | Multiple instances of same type |
| **Nested domain** | `:payment/stripe/service` | Avoid - use `:stripe/payment-service` instead |

**Hyphens within segments are OK**: `:firebase/user-service` (hyphen in service name)
**Hyphens instead of slash are NOT OK**: `:firebase-user-service` (missing slash)

**CRITICAL**: Hyphenated keys cause silent nil lookups → handlers receive nil services → 500 errors.

**Note**: This is the authoritative source for Integrant service naming conventions.

### init-key (Start Service)

```clojure
(require '[integrant.core :as ig])

(defmethod ig/init-key :server/http
  [_ {:keys [port handler]}]
  (println "Starting server on port" port)
  (http-kit/run-server handler {:port port}))
```

### halt-key! (Stop Service)

```clojure
(defmethod ig/halt-key! :server/http
  [_ server]
  (when server
    (server :timeout 100)))  ; Must be idempotent
```

## Discovery Patterns

```bash
# Find all init-key definitions
grep -r "ig/init-key" src/

# Find all halt-key! definitions
grep -r "ig/halt-key!" src/

# Find system configuration
cat resources/config/system.edn

# Find ig/ref dependencies
grep -r "#ig/ref" resources/
```

```clojure
;; REPL: View loaded config
(require '[aero.core :as aero])
(aero/read-config (io/resource "config/system.edn"))

;; REPL: List running services
(keys @system)
```

## Adding a New Service

1. **Add to system.edn**:
```clojure
;; resources/config/system.edn
{:my/service {:config-value "something"
              :dependency #ig/ref :other/service}

 ;; ... existing services
 }
```

2. **Create init-key**:
```clojure
;; src/myapp/services/my_service.clj
(ns myapp.services.my-service
  (:require [integrant.core :as ig]))

(defmethod ig/init-key :my/service
  [_ {:keys [config-value dependency]}]
  (println "Starting my/service")
  {:config config-value
   :dep dependency})
```

3. **Create halt-key!** (if cleanup needed):
```clojure
(defmethod ig/halt-key! :my/service
  [_ service]
  (when service
    (println "Stopping my/service")
    ;; Cleanup resources
    ))
```

4. **Require namespace** in main or dev namespace:
```clojure
;; Ensure namespace is loaded so methods are registered
(require '[myapp.services.my-service])
```

## Aero Configuration

Aero provides reader tags for environment-specific configuration:

```clojure
{:server/http
 {:port #profile {:dev 3000
                  :prod #or [#env "PORT" 8080]}
  :host #profile {:dev "localhost"
                  :prod "0.0.0.0"}}

 :db/connection
 {:uri #env "DATABASE_URL"}}  ; Reads environment variable
```

### Reader Tags

| Tag | Purpose | Example |
|-----|---------|---------|
| `#env` | Environment variable | `#env "PORT"` |
| `#or` | Fallback if nil | `#or [#env "PORT" 3000]` |
| `#profile` | Per-environment | `#profile {:dev X :prod Y}` |
| `#ig/ref` | Integrant dependency | `#ig/ref :other/service` |

### Loading Config

```clojure
(require '[aero.core :as aero]
         '[clojure.java.io :as io])

;; Load with profile
(defn load-config [profile]
  (aero/read-config (io/resource "config/system.edn")
                    {:profile profile}))

;; Development
(ig/init (load-config :dev))

;; Production
(ig/init (load-config :prod))
```

## Development Workflow with integrant-repl

**integrant-repl** provides enhanced REPL workflow with automatic namespace reloading.

**Maven dependency:** `integrant/repl {:mvn/version "0.5.0"}` (add to `:dev` alias)

### Standard Setup (dev/user.clj)

```clojure
(ns user
  (:require [integrant.repl :refer [go halt prep reset reset-all]]
            [integrant.repl.state :refer [system config]]
            [myapp.core :as core]))

;; Tell integrant-repl how to load config
(integrant.repl/set-prep! #(core/load-config :dev))

;; Now use these commands at REPL:
;; (go)        - Start the system
;; (halt)      - Stop the system
;; (reset)     - Stop, reload changed namespaces, start
;; (reset-all) - Stop, reload ALL namespaces, start
```

### Key Functions

| Function | Purpose | When to Use |
|----------|---------|-------------|
| `(go)` | Start system from scratch | Initial startup, after JVM restart |
| `(halt)` | Stop running system | Before restarting, cleanup |
| `(reset)` | Stop + reload changed + start | After code changes (most common) |
| `(reset-all)` | Stop + reload ALL + start | After changing ns structure, defmethods |
| `(prep)` | Prepare config without starting | Debug config loading |

### Using with clojure_eval MCP Tool

```clojure
;; Start system
(in-ns 'user)
(go)

;; Stop system
(in-ns 'user)
(halt)

;; Reload changed namespaces + restart
(in-ns 'user)
(reset)

;; Full reload + restart
(in-ns 'user)
(reset-all)
```

### Automatic Namespace Reloading

integrant-repl uses `clojure.tools.namespace.repl` to:
1. Track which files have changed
2. Reload only modified namespaces
3. Respect namespace dependencies

**This means**: Edit code → `(reset)` → Changes take effect without JVM restart.

### Accessing System State

```clojure
;; System is available via integrant.repl.state
(require '[integrant.repl.state :refer [system config]])

;; Access running services
(:server/http system)
(:db/connection system)

;; View current config
config
```

### Common Patterns

```clojure
;; Pattern 1: Quick reset after code change
;; Edit src/myapp/handler.clj, then:
user=> (reset)
:reloading (myapp.handler)
:resumed

;; Pattern 2: Force full reload after multimethod changes
user=> (reset-all)
:reloading (myapp.core myapp.handler myapp.routes ...)
:resumed

;; Pattern 3: Debug config issues
user=> (prep)  ; Load config without starting
user=> config  ; Inspect the config
```

## Accessing Services in Handlers

```clojure
;; System is passed to handler factory
(defn make-handler [system]
  (fn [request]
    (let [db-conn (:db/connection system)
          auth-svc (:auth/service system)]
      ;; Use services
      {:status 200 :body "OK"})))

;; In routes
(defmethod ig/init-key :app/routes
  [_ {:keys [system]}]
  (ring/router
    [["/api/data" {:get {:handler (make-handler system)}}]]))
```

## Anti-Patterns

### Missing halt-key! (Resource Leak)

For every `ig/init-key` defmethod, verify a matching `ig/halt-key!` exists. Missing halt-key! means resources leak on system restart — connections stay open, threads keep running, ports remain bound.

**Detection:** grep for `ig/init-key` defmethods, then check each has a corresponding `ig/halt-key!` for the same key. Flag any init-key without a matching halt-key!.

```clojure
;; WRONG: Hyphenated key
(defmethod ig/init-key :my-service [_ config] ...)
;; CORRECT: Slash notation
(defmethod ig/init-key :my/service [_ config] ...)

;; WRONG: Non-idempotent halt
(defmethod ig/halt-key! :my/service [_ svc]
  (close-connection svc))  ; Fails if already closed
;; CORRECT: Idempotent halt
(defmethod ig/halt-key! :my/service [_ svc]
  (when svc
    (try (close-connection svc)
         (catch Exception _))))

;; WRONG: Missing #ig/ref for dependency
{:my/service {:db :db/connection}}  ; Gets literal keyword
;; CORRECT: Use #ig/ref
{:my/service {:db #ig/ref :db/connection}}  ; Gets initialized service
```

## Configuration Change Policy

**CRITICAL**: Integrant resolves configuration values at system startup. When values supplied by Integrant change (e.g., system prompts, service configuration, feature flags in `system.edn`), **the backend server must be restarted** to pick up the new values.

```clojure
;; After changing system.edn or any Integrant-supplied config:
(in-ns 'user)
(reset)
```

**Why**: Integrant reads `system.edn` and injects resolved values into services via `init-key`. These values are captured once at init time. Changing `system.edn` without `(reset)` has no effect — services continue using stale values.

**Common symptoms of stale config**:
- Changed a system prompt but Gemini still uses the old one
- Updated a service URL but requests go to the old endpoint
- Modified parallel config but batch generation uses old concurrency settings

**Rule**: Any change to `resources/config/system.edn` or values injected via `#ig/ref` requires `(reset)`.

## Debugging

```clojure
;; Check what's in system
(keys @system)

;; Check specific service
(:my/service @system)

;; Trace init order
(defmethod ig/init-key :my/service [k config]
  (println "Initializing" k "with" config)
  ...)
```
