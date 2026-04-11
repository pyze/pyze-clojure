---
name: rewrite-clj-transforms
description: Structural Clojure code modification via bb + rewrite-clj. Use instead of sed/Edit for s-expression-aware transforms — ns require manipulation, defn transforms, EDN config updates, bulk keyword renames. Prefer clojure-lsp for single-symbol rename/find.
---

# rewrite-clj Transforms

Structural code modification for Clojure and EDN files via Babashka + rewrite-clj.

**rewrite-clj** parses Clojure into a zipper that preserves whitespace, comments, and formatting. It operates on s-expressions, not lines — eliminating the class of bugs where sed/awk corrupt bracket nesting.

**Babashka bundles rewrite-clj** — no extra dependencies. Scripts start in ~5ms.

---

## When to Use What

```
Need to modify Clojure/EDN code?
    │
    ├─ Single symbol rename → clojure-lsp (rename_symbol)
    │
    ├─ Find all callers → clojure-lsp (find_references)
    │
    ├─ Structural transform across files → bb + rewrite-clj
    │   ├─ ns :require manipulation
    │   ├─ defn signature changes
    │   ├─ Bulk keyword renames
    │   ├─ EDN config updates
    │   └─ Custom code mods
    │
    ├─ One-off small edit, known location → Edit tool (pragmatic)
    │
    └─ Non-Clojure file → Edit tool or sed
```

**Rule:** For any one-off script touching Clojure or EDN files, prefer `bb + rewrite-clj` over python, sed, or awk.

---

## Core API

```clojure
(require '[rewrite-clj.zip :as z])

;; Parse
(z/of-string "(defn foo [x] (+ x 1))")   ; from string
(z/of-file "src/my/ns.clj")               ; from file

;; Navigate
(z/next zloc)       ; depth-first forward
(z/right zloc)      ; next sibling
(z/down zloc)       ; first child
(z/up zloc)         ; parent
(z/find-value zloc z/next 'defn)   ; find by value
(z/get zloc :key)   ; navigate into map by key

;; Read
(z/sexpr zloc)      ; read as Clojure data
(z/string zloc)     ; read as string (preserving whitespace)
(z/node zloc)       ; raw node

;; Transform
(z/replace zloc new-value)          ; replace node
(z/edit zloc f & args)              ; apply f to sexpr
(z/insert-right zloc new-node)      ; insert sibling after
(z/insert-left zloc new-node)       ; insert sibling before
(z/remove zloc)                     ; remove node

;; Output
(z/root-string zloc)   ; serialize entire tree back to string
```

---

## Recipes

### 1. Add a require to ns form

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z])

(defn add-require! [file-path ns-to-add]
  (let [zloc (z/of-file file-path)
        ;; Find :require inside ns form
        req (-> zloc
                (z/find-value z/next 'ns)
                (z/find-value z/next :require))]
    (when req
      (-> req
          z/up               ; up to the (:require ...) list
          z/rightmost        ; last require entry
          (z/insert-right ns-to-add)
          z/root-string
          (->> (spit file-path)))
      (println "Added require to" file-path))))

;; Usage:
;; (add-require! "src/my/ns.clj" '[clojure.string :as str])
```

### 2. Rename keyword across files

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z]
         '[babashka.fs :as fs])

(defn rename-keyword! [dir old-kw new-kw]
  (doseq [f (fs/glob dir "**/*.{clj,cljc,cljs,edn}")]
    (let [content (slurp (str f))
          zloc (z/of-string content)
          transformed (loop [loc zloc]
                        (if (z/end? loc)
                          (z/root-string loc)
                          (recur (z/next
                                  (if (= (z/sexpr loc) old-kw)
                                    (z/replace loc new-kw)
                                    loc)))))]
      (when (not= content transformed)
        (spit (str f) transformed)
        (println "Updated:" (str f))))))

;; Usage:
;; (rename-keyword! "src" :old/name :new/name)
```

### 3. Update EDN config value

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z])

(defn update-edn! [file-path ks new-val]
  (let [zloc (z/of-file file-path)
        target (reduce (fn [loc k] (z/get loc k)) zloc ks)]
    (when target
      (->> (z/replace target new-val)
           z/root-string
           (spit file-path))
      (println "Updated" ks "in" file-path))))

;; Usage:
;; (update-edn! "resources/config.edn" [:server :port] 8080)
```

### 4. Find and list all defns in a namespace

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z])

(defn list-defns [file-path]
  (loop [loc (z/of-file file-path)
         results []]
    (if (z/end? loc)
      results
      (recur (z/next loc)
             (if (and (z/list? loc)
                      (#{'defn 'defn- 'defmethod}
                       (z/sexpr (z/down loc))))
               (conj results {:name (z/sexpr (z/right (z/down loc)))
                              :line (-> loc z/node meta :row)})
               results)))))

;; Usage:
;; (list-defns "src/my/ns.clj")
;; => [{:name foo :line 5} {:name bar :line 20}]
```

### 5. Remove a function definition

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z])

(defn remove-defn! [file-path fn-name]
  (let [zloc (z/of-file file-path)]
    (loop [loc zloc]
      (if (z/end? loc)
        (do (spit file-path (z/root-string loc))
            (println "Removed" fn-name "from" file-path))
        (recur (z/next
                (if (and (z/list? loc)
                         (#{'defn 'defn-} (z/sexpr (z/down loc)))
                         (= fn-name (z/sexpr (z/right (z/down loc)))))
                  (z/remove loc)
                  loc)))))))
```

### 6. Batch transform: add missing `!` suffix

```clojure
#!/usr/bin/env bb
(require '[rewrite-clj.zip :as z]
         '[babashka.fs :as fs]
         '[clojure.string :as str])

(def side-effect-fns #{"swap!" "reset!" "send" "send-off" "vswap!" "vreset!"
                        "spit" "println" "prn"})

(defn defn-has-side-effects? [zloc]
  (let [body (z/string zloc)]
    (some #(str/includes? body %) side-effect-fns)))

(defn defn-name-needs-bang? [zloc]
  (let [name-loc (z/right (z/down zloc))
        name-str (str (z/sexpr name-loc))]
    (and (defn-has-side-effects? zloc)
         (not (str/ends-with? name-str "!"))
         ;; Skip -main and test functions
         (not (str/starts-with? name-str "-"))
         (not (str/ends-with? name-str "-test")))))

;; Walk files, find defns needing !, report (don't auto-rename — use LSP for that)
```

---

## Integration with clojure-lsp

rewrite-clj and clojure-lsp are complementary:

| Operation | Tool | Why |
|-----------|------|-----|
| Rename a function | clojure-lsp `rename_symbol` | Updates all callers, AST-aware |
| Check if function is used | clojure-lsp `find_references` | Reliable cross-file search |
| Add/remove ns requires | bb + rewrite-clj | LSP doesn't expose this |
| Bulk keyword rename | bb + rewrite-clj | LSP is one-at-a-time |
| EDN config changes | bb + rewrite-clj | LSP doesn't handle EDN |
| Custom code mods | bb + rewrite-clj | Programmatic, any transform |

**Workflow for refactoring:**
1. Use `find_references` to understand impact
2. Use `rename_symbol` for symbol renames
3. Use bb + rewrite-clj for structural transforms LSP can't do
4. Run `get_diagnostics` to verify no issues introduced

---

## Integration with /code-cleanup

Agents can produce bb fix scripts alongside their findings:

```
Agent output:
  findings: [{:file "src/viz/chart.clj" :line 42 :violation "scattered-default"}]
  fix-script: |
    #!/usr/bin/env bb
    (require '[rewrite-clj.zip :as z])
    ;; Extract defaults from inner functions to edge...
```

The orchestrator presents both findings AND the fix script for user approval before running.

---

## Related Skills

- [babashka](../babashka/) — bb scripting basics, project structure
- [clojure-coding-standards](../clojure-coding-standards/) — the standards these transforms enforce
- [repl-semantic-search](../repl-semantic-search/) — REPL for searching, rewrite-clj for transforming
