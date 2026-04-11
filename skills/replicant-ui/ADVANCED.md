# Advanced Replicant Patterns

Keyboard handling, custom dispatch, SVG patterns, and debugging.

**Parent skill**: [SKILL.md](SKILL.md)

## Keyboard Event Handling

### Pattern: Effect-Based (Preferred for Simple Cases)

**Keep views pure by handling keyboard logic in effects.** The DOM event is available via `ctx :dispatch-data :replicant/dom-event`.

```clojure
;; 1. View - pure data, dispatches effect
[:input {:type "text"
         :placeholder "Type and press Enter..."
         :on {:keydown [[:my/handle-keydown!]]}}]

;; 2. Effect - reads DOM event from ctx
#?(:cljs
   (defn handle-keydown-effect
     "Handle keydown - submit on Enter key."
     [ctx _system _params]
     (when-let [event (some-> ctx :dispatch-data :replicant/dom-event)]
       (when (= "Enter" (.-key event))
         (.preventDefault event)
         ((:dispatch ctx) [[:my/submit-message]])))))

;; 3. Register effect
(register-effect! :my/handle-keydown! handle-keydown-effect)
```

### Pattern: Inline Function (When Pure Data Isn't Sufficient)

For complex keyboard handling with modifiers, use `:replicant.event/wrap-handler?`:

```clojure
[:input
 {:on
  {:keydown
   {:replicant.event/wrap-handler? true
    :replicant.event/handler
    (fn [{:replicant/keys [^js dom-event dispatch]}]
      (when (and (= "Enter" (.-key dom-event))
                 (not (.-shiftKey dom-event)))
        (.preventDefault dom-event)
        (dispatch [[:form/submit]])))}}}]
```

**Use `^js` type hint** on `dom-event` to prevent property name mangling.

## Custom Dispatch (Advanced Event Handling)

**Use custom dispatch when you need imperative access to DOM events before dispatching actions.**

### When to Use

- Filtering events based on keyboard modifiers (Ctrl, Shift, etc.)
- Performing imperative work before dispatching
- Integrating third-party libraries that need raw event access

### Setup

```clojure
[:button
 {:on
  {:keydown
   {:replicant.event/wrap-handler? true
    :replicant.event/handler
    (fn [{:replicant/keys [^js dom-event dispatch]}]
      (when (.-ctrlKey dom-event)
        (dispatch [[:actions/ctrl-clicked]])))}}}
 "Ctrl+Click Me"]
```

## SVG with foreignObject

**Fixed in 2025.12.1**: HTML inside `foreignObject` now renders correctly.

```clojure
[:svg {:width 400 :height 300}
 [:foreignObject {:x 10 :y 10 :width 180 :height 100}
  [:div {:xmlns "http://www.w3.org/1999/xhtml"}
   [:p "This is HTML inside SVG"]
   [:button {:on {:click [[:my/action]]}} "Click me"]]]]
```

## Range Inputs

**Fixed in 2025.12.1**: `value` now set after `min`/`max` to prevent clamping.

```clojure
[:input {:type "range"
         :min 0
         :max 100
         :value current-value
         :on {:input [[:slider/changed]]}}]
```

## Debugging

### Common Issues Checklist

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Action fires on page load | Single-wrapped `[:action]` | Use double-wrap `[[:action]]` |
| Action never fires | Triple-wrapped or typo | Check nesting and action name |
| Click "leaks" to new element | Missing stop-propagation | Add `[:app/stop-propagation!]` before nav |
| Element doesn't scroll into view | Missing lifecycle hook | Use `:replicant/on-mount` with scroll effect |
| JS library not initializing | Missing DOM node | Use `:replicant/on-mount`, check `:replicant/node` |
| Memory not persisting | Forgot to call `remember` | Call `(remember {...})` in on-mount |
