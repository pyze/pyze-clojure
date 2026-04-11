# clojure-mcp Troubleshooting Guide

## Common Errors

### "Cannot resolve var"

- Check namespace spelling and capitalization
- Verify namespace is loaded: `clojure_eval: (require 'my.namespace)`
- Check for typos in var names

### Slow startup (~6s)

Create `.nrepl-port` file to connect to existing REPL instead of starting fresh process.

```bash
cp .shadow-cljs/nrepl.port .nrepl-port
```

### Connection refused

- Verify nREPL is running: `lsof -i :<port>`
- Check port matches `.nrepl-port` content
- Verify your nREPL server is started

### Diagnosing startup errors

**Problem**: `clojure_eval` returns exit code 1 or `{}` with no clear error message

**Diagnosis**: Wrap startup in try/catch to see the actual error:

```clojure
clojure_eval: (try
                (require '[integrant.core :as ig])
                (require '[clojure.java.io :as io])
                (let [config (ig/read-string (slurp (io/resource "config/system.edn")))]
                  (println "Config:" (keys config))
                  (let [system (ig/init config)]
                    (println "System started. Keys:" (keys system))
                    {:status :ok}))
                (catch Throwable t
                  (println "ERROR:" (.getMessage t))
                  (println "Cause:" (when-let [c (.getCause t)] (.getMessage c)))
                  {:status :error :message (.getMessage t)}))
```

**Common causes**:
| Error | Cause | Fix |
|-------|-------|-----|
| "API key must be provided" | Missing env var | Set environment variable before starting |
| "Error on key :service/name" | Invalid config | Check system.edn format |
| "Could not locate..." | Missing dependency | Check deps.edn |

### State Persistence Gotcha

**CRITICAL**: Each `clojure_eval` invocation is independent when nREPL is NOT connected. State defined with `def` or `defonce` does NOT persist between calls unless you're connected to a running nREPL.

```clojure
# This does NOT work without nREPL connection:
clojure_eval: (def x 42)     # defines x
clojure_eval: x              # ERROR: x not found!
```

**Solution 1**: Combine everything in one call:
```clojure
clojure_eval: (let [x 42] (* x x))
```

**Solution 2**: Ensure nREPL connection (`.nrepl-port` file exists):
```bash
cp .shadow-cljs/nrepl.port .nrepl-port
```

**Solution 3**: Design functions to be self-contained with memoization:
```clojure
;; In your namespace
(def ^:private load-data (memoize (fn [] (expensive-load))))

(defn my-fn [arg]
  (let [data (load-data)]  ; memoized within same process
    (process data arg)))
```

### nREPL Port Not Found

**Problem**: `clojure_eval` fails to find `.nrepl-port`

**Solution**:
```bash
# Verify nREPL server is running
ps aux | grep nrepl

# Copy the port file
cp .shadow-cljs/nrepl.port .nrepl-port

# Verify it exists
cat .nrepl-port
```

### MCP Tool Not Available

**Problem**: `clojure_eval` tool is not available in Claude

**Cause**: MCP server not configured

**Solution**: Verify your MCP configuration includes clojure-mcp:
```json
{
  "mcpServers": {
    "clojure-mcp": {
      "command": "clojure",
      "args": ["-M:cli-assist", "-m", "clojure-mcp.core"]
    }
  }
}
```

Restart Claude after config changes.
