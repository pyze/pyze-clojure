---
name: gemini-sdk-clojure
description: Use when writing Clojure code that calls the Google Gemini Java SDK (com.google.genai), adding function declarations, building schemas, handling function calls, or debugging varargs interop errors
---

# Google Gemini Java SDK from Clojure

## Overview

The `com.google.genai` Java SDK requires specific interop patterns from Clojure. Varargs methods, inner enums, and builder patterns all have gotchas.

## Imports

```clojure
(:import [com.google.genai Client]
         [com.google.genai.types Content Part FunctionResponsePart
          GenerateContentConfig GenerateContentResponse
          Tool FunctionDeclaration Schema Type Type$Known
          CreateCachedContentConfig])
```

## Varargs Interop (Critical)

ALL Java varargs methods require explicit `into-array`. Without it: `IllegalArgumentException: No matching method found`.

| Method | Clojure Pattern |
|--------|----------------|
| `Content/fromParts(Part...)` | `(Content/fromParts (into-array Part [...]))` |
| `.functionDeclarations(FunctionDeclaration...)` | `(.functionDeclarations builder (into-array FunctionDeclaration [...]))` |
| `.tools(Tool...)` | `(.tools builder (into-array Tool [...]))` |
| `Part/fromFunctionResponse(name, map, FunctionResponsePart...)` | Pass `(into-array FunctionResponsePart [])` for empty vararg |

**Exception**: `.parts` on Content builder takes `List<Part>` (not varargs) — use `(.parts builder ^java.util.List (vec parts))`.

## Schema Builder

```clojure
(-> (Schema/builder)
    (.type (Type. Type$Known/OBJECT))
    (.properties {"query" (-> (Schema/builder)
                              (.type (Type. Type$Known/STRING))
                              (.description "The SQL query")
                              (.build))})
    (.required ["query"])
    (.build))
```

**Enum values**: Use `.enum_` (trailing underscore), NOT `.enum`.

```clojure
(.enum_ builder ["high" "medium" "low"])  ;; correct
(.enum builder ["high" "medium" "low"])   ;; ERROR: NoSuchMethodException
```

**Type constants**: Inner enum `Type$Known`, wrap in `Type.` constructor.

| Keyword | Constant |
|---------|----------|
| string | `(Type. Type$Known/STRING)` |
| integer | `(Type. Type$Known/INTEGER)` |
| boolean | `(Type. Type$Known/BOOLEAN)` |
| number | `(Type. Type$Known/NUMBER)` |
| array | `(Type. Type$Known/ARRAY)` |
| object | `(Type. Type$Known/OBJECT)` |

## Wemble ->schema DSL

`wemble.schema/->schema` converts declarative Clojure maps to `Schema` objects:

```clojure
(require '[wemble.schema :as gs])

(gs/->schema {:type :object
              :properties {:name {:type :string :description "Name"}
                           :items {:type :array :items {:type :string}}}
              :required [:name]
              :enum ["a" "b"]})
```

Supports nested recursion for `:properties` and `:items`.

## Wemble Tool Data Maps

`wemble.tools` defines tools as data, converting to SDK objects:

```clojure
(require '[wemble.tools :as gt])

(def my-tool
  {:name "run_sql"
   :description "Execute a SQL query"
   :parameters {:type :object
                :properties {"query" {:type :string :description "SQL query"}}
                :required ["query"]}
   :handler (fn [args] (run-query (get args "query")))})

(gt/->function-declaration my-tool)  ;; => FunctionDeclaration
(gt/->tool [my-tool])                ;; => Tool (bundle)
(gt/make-handler [my-tool])          ;; => dispatch fn
```

## Structured Output (JSON Mode)

```clojure
(.responseMimeType builder "application/json")
(.responseSchema builder some-schema)
```

**Requires `gemini-3-flash-preview` or later to combine with function calling.** On `gemini-2.5-flash`, returns `400` when combining tools with `responseMimeType`.

Parse response: `(json/read-str (.text response) :key-fn keyword)`

## Function Calling

**Declare tools:**
```clojure
(-> (Tool/builder)
    (.functionDeclarations (into-array FunctionDeclaration [decl1 decl2]))
    (.build))
```

**Extract function calls from response:**
```clojure
(let [fc (.functionCall part)]  ;; returns Optional
  (when (.isPresent fc)
    (let [fc-obj (.get fc)]
      {:name (-> (.name fc-obj) (.orElse nil))
       :args (-> (.args fc-obj) (.orElse {}))})))
```

**Send function results back:**
```clojure
(Part/fromFunctionResponse
 "function_name"
 {"key" "value"}  ;; must be Map<String,String>
 (into-array FunctionResponsePart []))
```

## Context Caching

```clojure
(require '[wemble.gemini :as gemini])

;; Create cache
(def cache-name
  (gemini/create-context-cache! client system-prompt
    :tools tool-obj :ttl-minutes 60))

;; Use in config
(gemini/make-cached-config cache-name :schema my-schema :temperature 0.2)

;; Cleanup
(gemini/delete-context-cache! client cache-name)
```

Or let `wemble.workflow/run-loop` manage it declaratively via the `:gemini` spec.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `No matching method found taking 1 args` | Varargs without `into-array` | Wrap in `(into-array Type [...])` |
| `No matching method enum found` | Using `.enum` | Use `.enum_` (trailing underscore) |
| `ClassCastException: PersistentVector` | Vec where array expected | Use `into-array` |
| `No matching ctor found` for Type | Using `Type$Known` directly | Wrap: `(Type. Type$Known/STRING)` |
