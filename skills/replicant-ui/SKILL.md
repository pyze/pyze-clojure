---
name: replicant-ui
description: Build UI components with Replicant vDOM and hiccup syntax. Use when creating views, rendering components, handling UI events, adding pages, or debugging event handler wrapping.
---

# Replicant UI

Replicant is a vDOM library for ClojureScript using hiccup syntax. **NOT React-based**.

## When to Use This Skill

**Use THIS skill if:**
- Building UI components with hiccup syntax
- Rendering vDOM with Replicant
- Handling UI events and user interactions
- Adding new pages or components
- Debugging event handler wrapping (`[[...]]`)

**Related Files:**
- [LIFECYCLE.md](LIFECYCLE.md) - Lifecycle hooks, memory pattern, JS interop
- [ADVANCED.md](ADVANCED.md) - Keyboard handling, custom dispatch, SVG patterns, debugging

## Version 2025.12.1 Changes

**Bug Fixes:**
- SVG child nodes from aliases now use correct namespaced DOM methods
- `foreignObject` elements no longer force SVG namespace on HTML children
- `value` attribute set after `min`/`max` to prevent range input clamping

**New Features:**
- **Seq rendering**: Top-level seqs now render, not just lists
- **`replicant.dom/recall`**: Retrieve DOM element state outside event handlers
- **Dispatch in hooks**: `:replicant/dispatch` available in lifecycle hooks and event handlers

## Core Concepts

### Hiccup Syntax

```clojure
;; [tag attributes children...]
[:div {:class "container"}
 [:h1 "Title"]
 [:p "Paragraph"]]

;; CSS class shorthand
[:div.container.main
 [:h1.title "Title"]]

;; ID shorthand
[:div#app.container "Content"]
```

### Rendering

```clojure
(require '[replicant.dom :as d])

;; Render to DOM element
(d/render
  (js/document.getElementById "app")
  [:div "Hello World"])

;; Re-render updates efficiently (vDOM diff)
```

### Components (Just Functions)

**CRITICAL**: Replicant components are just functions. Call them directly - do NOT wrap in vectors like React/Reagent.

```clojure
;; Component is a function returning hiccup
(defn greeting [{:keys [name]}]
  [:div.greeting
   [:h1 "Hello, " name "!"]])

;; CORRECT - Call the function
[:div
 (greeting {:name "Alice"})
 (greeting {:name "Bob"})]

;; WRONG - React/Reagent style (doesn't work in Replicant!)
[:div
 [greeting {:name "Alice"}]   ; This renders as raw vector, not hiccup
 [greeting {:name "Bob"}]]
```

**Why this matters**: `[fn {...}]` is React/Reagent's component mounting syntax. Replicant has no component lifecycle - it's pure hiccup data. Calling the function returns hiccup; wrapping in a vector returns a raw vector.

### Event Handlers (Pure Data)

**CRITICAL: Pure data only - NEVER use function callbacks**

Replicant event handlers must be **pure data** (effect vectors), never functions. This enables:
- JVM testing of views without browser
- Serializable event data
- Deterministic behavior

```clojure
;; CORRECT - Pure data: double-wrapped effect vector
[:button {:on {:click [[:my/action {:param "value"}]]}}
 "Click Me"]

;; WRONG - Function callback (not pure data!)
[:button {:on {:click #(dispatch! [:my/action])}}  ; Never do this!
 "Click Me"]
```

**CRITICAL: Double-wrapped vectors `[[...]]`** - Event handlers require actions wrapped in an outer vector.

```clojure
;; CORRECT - Double-wrapped: outer vector = handler, inner = action
[:button {:on {:click [[:my/action {:param "value"}]]}}
 "Click Me"]

;; WRONG - Single-wrapped: would try to execute at render time
[:button {:on {:click [:my/action {:param "value"}]}}  ; Don't do this!
 "Click Me"]

;; Multiple actions (still double-wrapped)
[:button {:on {:click [[:action-a] [:action-b {:x 1}]]}}
 "Do Both"]

;; Form inputs - note :on {:input ...} syntax
[:input {:type "text"
         :value current-value
         :on {:input [[:form/update-field {:field :name}]]}}]
```

### What Happens With Wrong Wrapping

```
CORRECT: [[:action]]     -> On click, dispatches [:action]
WRONG:   [:action]       -> Treats :action as function, calls at RENDER time
WRONG:   [[[:action]]]   -> Extra nesting, dispatch fails silently
```

**Debugging Checklist**:
- [ ] Action dispatches when I click? -> Check for double-wrap `[[...]]`
- [ ] Action fires at page load? -> You have single-wrap `[...]`
- [ ] Action never fires? -> Check for triple-wrap or typo in action name

### Stop Propagation Effect (Navigation Pattern)

**CRITICAL**: When navigation changes DOM during click, the click event can continue to elements that appear in the same position.

```clojure
;; CORRECT - Stop propagation BEFORE navigation action
[:button {:on {:click [[:app/stop-propagation!] [:nav/go-home!]]}}
 "Back"]
```

**When to use**: Any click handler that triggers immediate DOM re-render and might have a different clickable element appear at the same screen position.

## Adding a New Component

```clojure
(defn task-list [{:keys [title items]} db]
  [:div.task-list
   [:h2 title]
   [:ul
    (for [item items]
      [:li {:key (:id item)} (:name item)])]])
```

## Adding a New Page

1. **Create page component**
2. **Add to router** (if using routing)
3. **Add view dispatch**

## Styling with uix.css

```clojure
(require '[uix.css.replicant :refer [defclass]])

(defclass container
  {:display "flex"
   :flex-direction "column"
   :padding "1rem"})

[:div {:class container}
 "Styled content"]
```

## Common Patterns

### Conditional Rendering

```clojure
(defn user-card [{:keys [show?]} db]
  [:div
   (when show?
     [:p "Visible content"])
   (if loading?
     [:div.spinner "Loading..."]
     [:div.content "Loaded!"])])
```

### Lists with Keys

```clojure
;; ALWAYS use :key for list items
[:ul
 (for [item items]
   [:li {:key (:id item)}
    (:name item)])]

;; Seqs work too (not just lists) - added in 2025.12.1
[:ul
 (map (fn [item]
        [:li {:key (:id item)} (:name item)])
      items)]
```

### Disabled States

```clojure
[:button {:disabled loading?
          :on-click [[:submit/form]]}
 (if loading? "Submitting..." "Submit")]
```

## Anti-Patterns

```clojure
;; WRONG: Missing key in list
[:ul (for [item items] [:li (:name item)])]

;; CORRECT: Always include key
[:ul (for [item items] [:li {:key (:id item)} (:name item)])]

;; WRONG: React-style state (Replicant doesn't have hooks)
(def [state set-state] (use-state nil))  ; Doesn't exist!

;; CORRECT: Use a normalized store for state
(defn component [props db]
  (let [value (queries/get-value db)]
    [:div value]))

;; WRONG: Mutating props
(defn bad-component [props db]
  (assoc props :extra "value")  ; Props are immutable
  [:div (:name props)])
```
