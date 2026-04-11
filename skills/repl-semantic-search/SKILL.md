---
name: repl-semantic-search
description: Use REPL introspection as semantic search over Clojure codebases. Use when searching for resolver relationships, attribute provenance, namespace structure, or function signatures — before falling back to Grep.
---

# REPL Semantic Search

The running REPL is the most accurate search tool for structural questions about Clojure code. Unlike Grep, it queries the actual loaded program — always current, never stale.

**This skill covers:** When and how to use the REPL as a search tool.

**Use DIFFERENT skill if:**
- REPL connection/eval mechanics → [clojure-mcp-repl](../clojure-mcp-repl/)
- Writing Pathom resolvers → [pathom3-eql](../pathom3-eql/)
- Single-var lookup (`doc`, `source`, `apropos`) → [clojure-mcp-repl REPL Exploration](../clojure-mcp-repl/)

---

## Decision Tree: REPL vs Grep vs LSP

| Question | Best tool | Why |
|----------|-----------|-----|
| Which resolvers produce `:data/rows`? | **REPL** | Pathom index is source of truth |
| What attributes are reachable from `:user/id`? | **REPL** | Requires graph traversal |
| What's the execution plan for `[:a :b :c]`? | **REPL** | Planner logic, not text |
| What functions exist in namespace X? | **REPL** | `ns-publics` is authoritative |
| What does function Y accept? | **REPL** | `meta` gives arglists/docstring |
| Where is the string `"foo-bar"` used? | **Grep** | Text matching across files |
| Find all callers of `my-fn` | **LSP** | AST-aware reference search |
| Count call sites for a refactor | **LSP** | `find_references` gives exact count without false positives from comments/strings |
| Why is this code structured this way? | **Skills/docs** | Intent isn't introspectable |

**Heuristic:** If the answer is a fact about loaded code (exists, signature, relationships), ask the REPL. If it's a text pattern or design rationale, use Grep/LSP/skills.

---

## Clojure Namespace Introspection

These recipes work in any Clojure project. For single-var lookups (`doc`, `source`, `apropos`, `find-doc`), see [clojure-mcp-repl REPL Exploration](../clojure-mcp-repl/).

### Find namespaces by concept

```clojure
;; All namespaces matching a pattern
(filter #(re-find #"render" (str %)) (all-ns))
;; => (viz.render.layout viz.render.logic viz.render.reactive-pipeline ...)
```

### Find vars across namespaces

```clojure
;; All public vars matching a pattern, across all loaded namespaces
(for [ns (all-ns)
      [sym _] (ns-publics ns)
      :when (re-find #"error" (str sym))]
  (symbol (str ns) (str sym)))
;; => (viz.pathom.env/handle-error continuous-eql.core/collect-errors ...)
```

### Multimethod dispatch table

```clojure
;; What dispatch values does a multimethod handle?
(keys (methods viz.markdown.render/render-node))
;; => (:heading :paragraph :code :blockquote ...)
```

### Protocol implementors

```clojure
(extenders SomeProtocol)
;; => (SomeRecord AnotherRecord)
```

---

## Pathom3 Graph Introspection

These recipes work with any Pathom3 env. Requires `env` — see [Obtaining the Pathom env](#obtaining-the-pathom-env) below.

### List all resolvers

```clojure
(keys (::pci/index-resolvers env))
;; => (r-user r-orders r-product ...)
```

### Resolver config (inputs, outputs)

```clojure
(pco/operation-config (pci/resolver env 'r-user))
;; => {::pco/input [:user/id] ::pco/output [:user/name :user/email] ...}
```

### What does a resolver output?

```clojure
(pci/resolver-provides env 'r-user)
;; => #{:user/name :user/email}
```

### Is an attribute resolvable?

```clojure
(pci/attribute-available? env :user/email)
;; => true
```

### What's reachable from known data?

```clojure
;; Given :user/id, what attributes can Pathom derive?
(pci/reachable-attributes env {:user/id 1})
;; => #{:user/name :user/email :user/orders ...}

;; With paths (dependency chains)
(pci/reachable-paths env {:user/id 1})
```

### Obtaining the Pathom env

The env is project-specific. Common patterns:

```clojure
;; Integrant system (most common in this ecosystem)
(-> integrant.repl.state/system :some/pathom-key)

;; Global var
@my.app/pathom-env

;; Build one ad-hoc for exploration
(require '[com.wsscode.pathom3.connect.indexes :as pci])
(def env (pci/register [resolver-a resolver-b]))
```

---

## continuous-eql Recipes

These require the `continuous-eql` library. Skip this section if the project doesn't use it.

### Which resolvers produce these attributes?

```clojure
(require '[continuous-eql.core :as ceql])

(ceql/resolvers-for-keys env [:data/rows :data/error])
;; => [r-query r-fallback]
```

### Execution plan (without running)

```clojure
(ceql/explain-query env [:target/attr])
;; => {:execution-order [{:resolver 'r-a :input-keys [] :output-keys [:a]}
;;                       {:resolver 'r-b :input-keys [:a] :output-keys [:target/attr]}]
;;     :attr->resolver {:a 'r-a :target/attr 'r-b}
;;     :resolvers #{'r-a 'r-b}
;;     :depth 2}
```

### Trace attribute provenance

```clojure
;; "Who produces :target/attr?"
(:attr->resolver (ceql/explain-query env [:target/attr]))
;; => {:a 'r-a :target/attr 'r-b}
```

### Execution plan with seeded entity

```clojure
;; "Given :user/id is already known, what runs?"
(ceql/explain-query env [:user/orders] {:entity {:user/id 1}})
```

---

## Dependency Topology

Recipes for understanding namespace coupling and dependency health. Works in any Clojure project with loaded namespaces.

### Circular namespace dependencies

```clojure
;; Find namespace pairs that require each other
(for [ns (all-ns)
      [_ req-ns] (ns-aliases ns)
      :when (contains? (set (map val (ns-aliases req-ns)))
                        (find-ns (ns-name ns)))]
  [(ns-name ns) (ns-name req-ns)])
```

### Import hotspots (most-required namespaces)

```clojure
;; Top 10 namespaces by fan-in — highest blast radius if changed
(->> (all-ns)
     (mapcat (fn [ns] (map (comp ns-name val) (ns-aliases ns))))
     frequencies
     (sort-by val >)
     (take 10))
;; => ([clojure.string 42] [clojure.core 38] [my.app.db 15] ...)
```

### Orphan namespaces

```clojure
;; Loaded namespaces not required by any other namespace
(let [required (set (mapcat (fn [ns] (map (comp ns-name val) (ns-aliases ns)))
                            (all-ns)))]
  (remove required (map ns-name (all-ns))))
;; Candidates for deletion or missing require
```

### Namespace dependency count (fan-out)

```clojure
;; Namespaces with the most outgoing dependencies — tightly coupled
(->> (all-ns)
     (map (fn [ns] [(ns-name ns) (count (ns-aliases ns))]))
     (sort-by second >)
     (take 10))
```
