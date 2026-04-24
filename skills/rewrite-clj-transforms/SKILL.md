---
name: rewrite-clj-transforms
description: "This skill should be used when performing fine-grained structural Clojure code modifications via bb + rewrite-clj: keyword rename across files, EDN config updates, ns require manipulation. For namespace-level ops (outline, extract, deps), use clj-surgeon."
---

# rewrite-clj Transforms

Fine-grained structural code modification for Clojure and EDN files via Babashka + rewrite-clj.

**rewrite-clj** parses Clojure into a zipper that preserves whitespace, comments, and formatting. It operates on s-expressions, not lines — eliminating the class of bugs where sed/awk corrupt bracket nesting.

**Babashka bundles rewrite-clj** — no extra dependencies. Scripts start in ~5ms.

---

## When to Use What

```
Need to modify Clojure/EDN code?
    │
    ├─ Namespace-level ops (outline, extract, deps, declares) → clj-surgeon
    │
    ├─ Single symbol rename → clojure-lsp (rename_symbol)
    │
    ├─ Find all callers → clojure-lsp (find_references)
    │
    ├─ Fine-grained structural transform → bb + rewrite-clj (this skill)
    │   ├─ ns :require manipulation
    │   ├─ Bulk keyword renames
    │   └─ EDN config updates
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

## Related Skills

- [clj-surgeon](../clj-surgeon/) — namespace-level structural ops (outline, extract, deps, declares)
- [clojure-coding-standards](../clojure-coding-standards/) — the standards these transforms enforce
- [tool-selection-clojure](../tool-selection-clojure/) — full decision tree for choosing the right tool
