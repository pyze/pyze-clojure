---
name: pathom3-eql
description: Build EQL resolvers with Pathom 3 for data resolution. Use when adding resolvers, writing EQL queries, handling Pathom errors, configuring the Pathom graph, or working with ident-based data access.
---

# Pathom 3 / EQL

Pathom 3 is a library for unifying data sources via attribute modeling in a graph. It resolves EQL queries by planning and executing resolver chains automatically.

**Maven**: `com.wsscode/pathom3 {:mvn/version "2025.01.16-alpha"}`

## When to Use This Skill

**Use THIS skill if:**
- Adding or modifying Pathom resolvers
- Writing or debugging EQL queries
- Configuring the Pathom graph (env, plugins)
- Understanding Pathom error handling and fail-fast strategy
- Working with ident-based data access

## Namespace Aliases

| Alias | Full Namespace |
|-------|---------------|
| `pci` | `com.wsscode.pathom3.connect.indexes` |
| `pco` | `com.wsscode.pathom3.connect.operation` |
| `p.eql` | `com.wsscode.pathom3.interface.eql` |
| `psm` | `com.wsscode.pathom3.interface.smart-map` |
| `pbir` | `com.wsscode.pathom3.connect.built-in.resolvers` |
| `pbip` | `com.wsscode.pathom3.connect.built-in.plugins` |
| `p.plugin` | `com.wsscode.pathom3.plugin` |
| `pcr` | `com.wsscode.pathom3.connect.runner` |
| `pcp` | `com.wsscode.pathom3.connect.planner` |
| `p.error` | `com.wsscode.pathom3.error` |
| `p.path` | `com.wsscode.pathom3.path` |

## Core Concepts

### Resolvers

Resolvers are annotated functions expressing relationships between attributes. Pathom infers input from destructuring and output from the return map:

```clojure
;; Minimal form — input/output inferred
(pco/defresolver full-name [{::keys [first-name last-name]}]
  {::full-name (str first-name " " last-name)})

;; Explicit form — when inference is insufficient
(pco/defresolver ip->lat-long [env {:keys [ip]}]
  {::pco/output [:latitude :longitude]}
  (-> (fetch-geo ip)
      (select-keys [:latitude :longitude])))
```

**Key rule**: Resolvers receive **labeled maps** and return **labeled maps**. The labels (keywords) are the attributes that form the graph.

### Environment and Indexes

```clojure
;; Register resolvers → build indexes
(def env (pci/register [resolver-a resolver-b]))

;; Inject custom data (available to all resolvers via env)
(def env-with-db
  (pci/register {:my/db @app-state :my/context "some-context"}
                [resolver-a resolver-b]))

;; Resolvers access custom data from env (first arg)
(pco/defresolver my-resolver [env {:keys [some/input]}]
  {:some/output (query-db (:my/db env) input)})
```

### EQL Processing

`p.eql/process` is the primary query interface. **Synchronous** when all resolvers are synchronous:

```clojure
;; Basic query
(p.eql/process env [:temperature :humidity])
;; => {:temperature 23 :humidity 65}

;; Query with initial entity data
(p.eql/process env {:city "Recife"} [:temperature])
;; => {:temperature 23}

;; Ident-based lookup (read-side queries)
(p.eql/process env [{[:user/id 1] [:user/name :user/email]}])
;; => {[:user/id 1] {:user/name "Alice" :user/email "a@b.com"}}
```

### Programmatic Resolver Construction

For dynamic resolver creation:

```clojure
(pco/resolver `my-resolver
  {::pco/input  [:a]
   ::pco/output [:b]}
  (fn [env input] {:b (compute (:a input))}))

;; Named dynamic resolvers
(pco/resolver (symbol (str "prefix--" name))
  {::pco/input  []
   ::pco/output output-spec}
  (fn [_ _] {join-key {:entity/id var-name}}))
```

## CRITICAL: Attribute-Based Join Pattern

**All resolver inputs use attribute-based joins, NOT ident-based joins.**

Ident-based joins like `{[:data/source "x.edn"] [:data/rows]}` do NOT work in `::pco/input` — the Pathom planner cannot resolve them.

### Three-Step Chain Pattern

1. **Seed resolver**: `input []`, provides `{:var/region {:control/id :my-ns/region}}`
2. **Static resolver**: `input [:control/id]`, resolves to `{:control/value "East" ...}`
3. **Consumer resolver**: `input [{:var/region [:control/value]}]`, receives resolved values

This **seed -> static -> consumer** chain is the standard pattern for attribute-based data flow.

### Ident Queries (Read-Side Only)

Idents work in EQL **queries** (read-side), just not in `::pco/input` specs:

```clojure
;; WORKS — ident in query
(p.eql/process env [{[:data/source "x.edn"] [:data/rows]}])

;; FAILS — ident in resolver input spec
{::pco/input [{[:data/source "x.edn"] [:data/rows]}]}
;; Error: "Can't reach attribute [:data/source \"x.edn\"]"
```

## EQL Query Syntax

| Pattern | Syntax | Example |
|---------|--------|---------|
| Flat attributes | `[:a :b]` | `[:temperature :humidity]` |
| Join (nested) | `[{:a [:b]}]` | `[{:user/friends [:user/name]}]` |
| Ident lookup | `[{[:a "v"] [:b]}]` | `[{[:data/source "x.edn"] [:data/rows]}]` |
| Params | `[{(:a {k v}) [:b]}]` | `[{(:search {:q "x"}) [:result/name]}]` |
| Wildcard | `[:a *]` | Fetches `:a` plus all reachable attrs |

## CRITICAL: Error Handling Strategy

### Recommended: `wrap-resolve` Fail-Fast Plugin

Use strict mode (default) with a `wrap-resolve` plugin that logs errors with full context **before** Pathom wraps them:

```clojure
(def fail-fast-plugin
  {::p.plugin/id ::fail-fast
   ::pcr/wrap-resolve
   (fn [resolve-fn]
     (fn [env input]
       (try
         (resolve-fn env input)
         (catch #?(:clj Exception :cljs :default) e
           (let [op-name (::pco/op-name (::pcp/node env))
                 path    (::p.path/path env)]
             #?(:cljs (js/console.error "Pathom resolver error:" op-name
                                        "input:" (pr-str input) "path:" (pr-str path)
                                        "message:" (ex-message e))
                :clj  (println "Pathom resolver error:" op-name "input:" input "path:" path))
             (throw e))))))})
```

### NEVER Use Lenient Mode

Lenient mode (`::p.error/lenient-mode? true`) silently swallows errors into deeply nested `::pcr/attribute-errors` maps. Always use strict mode + fail-fast plugin.

## Built-in Resolvers

```clojure
(require '[com.wsscode.pathom3.connect.built-in.resolvers :as pbir])

(pbir/constantly-resolver :math/PI 3.14159)             ;; Constant value
(pbir/alias-resolver :impl/product-id :ui/product-id)   ;; One-directional alias
(pbir/single-attr-resolver :ms :sec #(/ % 1000))        ;; Single attr transform
(pbir/global-data-resolver {:server/port 8080})          ;; Global constants
```

## Plugins

```clojure
(def my-plugin
  {::p.plugin/id ::my-plugin
   ::pcr/wrap-resolve
   (fn [resolve-fn]
     (fn [env input]
       (let [result (resolve-fn env input)]
         result)))})

(def env (-> (pci/register resolvers)
             (p.plugin/register my-plugin)))
```

### Available Plugin Hooks

| Hook | Wraps |
|------|-------|
| `::pcr/wrap-resolve` | Individual resolver calls |
| `::pcr/wrap-mutate` | Mutation calls |
| `::pcr/wrap-resolver-error` | Resolver error handling |
| `::pcr/wrap-mutation-error` | Mutation error handling |
| `::pcr/wrap-merge-attribute` | How results merge into entity |
| `::pcr/wrap-run-graph!` | Entire graph execution |

## Resolver Purity — No `env` Access Without Justification

**All resolver data should flow through `::pco/input`. The `env` parameter should be `_` (unused) by default.**

Reading from `env` introduces hidden dependencies on mutable state invisible to the Pathom planner and cache. See [caching-and-purity](../caching-and-purity/) for the full principle and reasoning.

```clojure
;; DEFAULT: env is _ (unused), all data through ::pco/input
(pco/defresolver my-resolver [_ {:keys [declared/input]}]
  {::pco/input [:declared/input] ...}
  ...)

;; EXCEPTION: requires clear justification
;; ENV ACCESS: [justification]
(pco/defresolver my-impure-resolver [env {:keys [...]}]
  {::pco/cache? false ...}  ;; mandatory when accessing env
  ...)
```

## REPL Testing

```clojure
;; Build env and test a resolver chain
(require '[com.wsscode.pathom3.connect.operation :as pco])
(require '[com.wsscode.pathom3.connect.indexes :as pci])
(require '[com.wsscode.pathom3.interface.eql :as p.eql])

;; Test with a simple env
(def env (pci/register [my-resolver-a my-resolver-b]))
(p.eql/process env [:some/attribute])

;; Inspect registered resolvers
(keys (::pci/index-resolvers env))
```

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Ident-based joins in `::pco/input` | Use attribute-based joins with seed resolvers instead |
| Lenient mode hides errors | Never use `::p.error/lenient-mode?`. Use strict + fail-fast plugin |
| Resolver returns nil | Pathom treats nil as "attribute not found" — return empty map `{}` if resolver ran |
| `PersistentHashSet cannot be cast to Named` | Filter `keyword?` keys when reading `index-attributes` |
| Lazy seqs in results | Use `sequential?` in tests, not `vector?` |
| Same query re-plans every call | Share `plan-cache*` atom across calls |
| Dynamic resolver not found | Ensure env receives dynamic resolvers when building indexes |

## Related Skills

| Skill | Relationship |
|-------|-------------|
| caching-and-purity | Cache key must capture all dependencies; `env` access = impurity |
| clojure-coding-standards | Resolver functions follow FP principles |
