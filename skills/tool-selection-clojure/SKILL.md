---
name: tool-selection-clojure
description: "This skill should be used when choosing between clojure-lsp, REPL, and grep for Clojure code navigation, or when navigating namespaces and finding references in Clojure projects."
---

Clojure companion to the tool-selection skill (in pyze-workflow). This skill provides Clojure-specific tool usage, operations, and decision guidance.

## Clojure Tool Hierarchy

```
clojure-lsp (cclsp) > REPL (clojure_eval / clojure-mcp) > Grep/Glob
```

---

## clojure-lsp / cclsp Operations

Use for anything about **code structure and relationships**:

| Task | LSP method |
|------|-----------|
| Find all usages of a function | `find_references` |
| Find all usages of a namespace | `find_references` on the ns symbol |
| Is this code dead? | `find_references` -- 0 results = dead |
| Where is this defined? | `goto_definition` |
| Rename a symbol safely | `rename` |
| What does this namespace export? | `document_symbols` |

### Why LSP Over Grep for Clojure

Grep misses aliased requires and catches false positives:

```clojure
;; Given this require:
(ns my.app.handler
  (:require [my.app.service :as svc]))

;; Grep for "my.app.service" will NOT find:
(svc/process data)    ;; aliased reference

;; Grep for "process" will find false positives:
;; - Comments mentioning "process"
;; - Other namespaces with a "process" function
;; - Strings containing "process"
```

LSP understands namespace aliases and finds all actual references.

---

## REPL Operations (clojure_eval / clojure-mcp)

Use for anything about **runtime behavior and data**:

| Task | REPL approach |
|------|--------------|
| What shape does this function return? | Call it and inspect |
| Does this API support X? | Try it |
| What's the current system state? | `(keys @user/system)` |
| Does this config key exist? | `(get config :key)` |
| Validate an assumption | Execute and verify |
| Explore unfamiliar code | Load namespace, call functions, inspect results |
| What resolvers provide this attribute? | `(pci/attribute-info env :attr)` |

### Namespace Exploration at REPL

```clojure
;; List public vars in a namespace
(keys (ns-publics 'my.app.service))

;; Get docstring
(doc my.app.service/process)

;; See source
(source my.app.service/process)

;; Check what a function returns
(my.app.service/process {:sample "input"})
```

---

## Clojure-Specific Decision Tree

```
What are you trying to find out?
|
+-- Who calls this function / is this dead code?
|   +-- Use clojure-lsp find_references
|
+-- What does this function return / what shape is the data?
|   +-- Use REPL: call it and inspect
|
+-- What namespace provides this alias?
|   +-- Use clojure-lsp goto_definition
|
+-- What resolvers / attributes exist?
|   +-- Use REPL: (pci/attribute-info env :attr)
|
+-- What's in the running system?
|   +-- Use REPL: (keys @user/system)
|
+-- Find a specific string / error message / TODO?
|   +-- Use Grep
|
+-- Find files matching a pattern?
|   +-- Use Glob
|
+-- Not sure?
    +-- Try clojure-lsp first, REPL second, grep last
```

---

## Common Mistakes in Clojure

### Using grep to find references

```
BAD:  grep -r "my-function" src/     -- finds text, not references
GOOD: clojure-lsp find_references     -- finds actual callers, follows aliases
```

### Using grep to check dead code

```
BAD:  grep -r "my-function" src/ | wc -l   -- counts text matches
GOOD: clojure-lsp find_references           -- counts actual callers
```

### Using grep when REPL would answer instantly

```
BAD:  grep through source to understand return value of (pci/register ...)
GOOD: (pci/register [...]) at the REPL -- see the actual env map
```

### Defaulting to grep because it's familiar

Grep is always available and always works -- that makes it the *easy* choice (nearby, familiar). But clojure-lsp and REPL are the *simple* choice -- they give you the right answer with less interleaving of irrelevant results.

---

## Summary

1. **clojure-lsp for structure** -- references, definitions, renames, dead code; understands aliases
2. **REPL for behavior** -- data shapes, system state, resolver exploration, live validation
3. **Grep for text** -- literal patterns, error messages, non-code files
4. **Never grep for Clojure references** -- use clojure-lsp; grep misses aliases and catches false positives
