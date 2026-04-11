---
name: wemble-gemini
description: Use when using Wemble's Gemini integration - client lifecycle, chat-turn, ask/ask-json/ask-text, tool data maps, context caching, ->schema DSL, and schema compatibility validation
---

# Wemble Gemini Integration

## Overview

Wemble wraps the Google Gemini Java SDK with idiomatic Clojure functions for LLM-powered workflows. Includes client management, multi-turn chat with function calling, structured output, tool definitions as data, and schema utilities.

## Namespaces

| Namespace | Purpose |
|-----------|---------|
| `wemble.gemini` | Client, chat-turn, ask, ask-json, ask-text, config builders, context caching |
| `wemble.tools` | Tool data maps → SDK objects, dispatch function |
| `wemble.schema` | ->schema DSL, validate-compatible |

## Client

```clojure
(require '[wemble.gemini :as gemini])

(def client (gemini/create-client))  ;; reads GOOGLE_API_KEY from env
```

## Model Selection

Default model is `"gemini-3-flash-preview"`. Override via dynamic binding:

```clojure
(binding [gemini/*model* "gemini-2.5-flash"]
  (gemini/ask-text client "Hello"))
```

All API functions (`ask`, `ask-text`, `ask-json`, `chat-turn`) respect `gemini/*model*`. The `wemble.workflow/run-loop` function binds this var automatically from its `:gemini {:model "..."}` spec, so workflow specs do not need manual binding.

## chat-turn

Multi-turn conversation with automatic function call handling:

```clojure
(gemini/chat-turn client history config
  {:fc-handler  (fn [name args] (dispatch name args))
   :max-rounds  10})
;; => {:text "..." :history [...] :function-calls [{:name :args :result}]}
```

If `:fc-handler` is nil, the loop exits on the first function call (returns text only).

When max rounds are reached and the model still wants tool calls, chat-turn processes the final calls and nudges the model for a text response.

## ask

Single-shot question with tools:

```clojure
(gemini/ask client system-prompt tools fc-handler "What are the top activities?"
  :context "Additional context here")
;; => "The top activities are..."
```

## ask-text

Plain text generation — no tools, no system prompt, no schema:

```clojure
(gemini/ask-text client "Summarize these findings: ...")
;; => "The analysis shows..."
```

## ask-json

Structured JSON response — no tools:

```clojure
(gemini/ask-json client "Rate this from 1-5" my-schema)
;; => {:score 4 :rationale "..."}
```

## Config Builders

```clojure
;; Full config with system prompt and tools
(gemini/make-config "You are an analyst..."
  :tools my-tool-obj
  :schema my-response-schema
  :temperature 0.2)

;; Config using cached context (omits system prompt and tools)
(gemini/make-cached-config cache-name
  :schema my-response-schema
  :temperature 0.2)
```

## Tool Data Maps

Define tools as plain Clojure data:

```clojure
(require '[wemble.tools :as tools])

(def sql-tool
  {:name "run_sql"
   :description "Execute a SQL query"
   :parameters {:type :object
                :properties {"query" {:type :string :description "SQL query"}}
                :required ["query"]}
   :handler (fn [args] {"result" (str (run-query (get args "query")))})})

;; Convert to SDK objects
(tools/->function-declaration sql-tool)  ;; => FunctionDeclaration
(tools/->tool [sql-tool schema-tool])    ;; => Tool (bundle)
(tools/make-handler [sql-tool schema-tool]) ;; => (fn [name args] result)
```

Handler return values should be maps — values are stringified for Gemini's function response format.

## Context Caching

Reuse system prompt + tools across multiple API calls:

```clojure
;; Manual lifecycle
(let [cache-name (gemini/create-context-cache! client system-prompt
                    :tools my-tool :ttl-minutes 60)]
  (try
    (let [config (gemini/make-cached-config cache-name :schema my-schema)]
      (gemini/chat-turn client [] config {:fc-handler handler}))
    (finally
      (gemini/delete-context-cache! client cache-name))))

;; Or let run-loop manage it:
(wf/run-loop {:gemini {:context-cache {:enabled true
                                       :system-prompt "..."
                                       :tools my-tool
                                       :ttl-minutes 60}} ...})
```

## ->schema DSL

Converts Clojure maps to Gemini `Schema` objects:

```clojure
(require '[wemble.schema :as gs])

(gs/->schema {:type :object
              :properties {:title {:type :string :description "Short title"}
                           :score {:type :integer :description "Score 1-5"}
                           :tags  {:type :array :items {:type :string}}}
              :required [:title :score :tags]})
```

Supports: `:string`, `:integer`, `:boolean`, `:number`, `:array`, `:object`.
Use `:description` to guide LLM output. Use `:enum` for constrained string values.

## Schema Compatibility Validation

Check that Malli event schemas and Gemini response schemas agree:

```clojure
(gs/validate-compatible
  ;; Malli schema
  [:map [:title :string] [:score :int] [:tags [:vector :string]]]
  ;; Gemini schema map (for LLM output)
  {:type :object
   :properties {:title {:type :string} :score {:type :integer} :tags {:type :array :items {:type :string}}}
   :required [:title :score :tags]})
;; => :ok

;; Mismatches return:
;; => {:errors [{:field :score :malli-type :int :gemini-type :string}]}
```

## extract-function-calls

Low-level: parse function calls from a Gemini response:

```clojure
(gemini/extract-function-calls response)
;; => ({:name "run_sql" :args {"query" "SELECT ..."}})
```
