# Replicant Lifecycle Hooks

Post-render DOM operations, memory patterns, and JavaScript interop.

**Parent skill**: [SKILL.md](SKILL.md)

## Lifecycle Hooks (Post-Render DOM Operations)

**CRITICAL**: Use lifecycle hooks for DOM operations that must run after render (scroll, focus, measuring).

### Available Hooks

| Hook | Fires When |
|------|------------|
| `:replicant/on-mount` | Element first added to DOM |
| `:replicant/on-render` | After every DOM modification |
| `:replicant/on-unmount` | Element removed from DOM |

### Hook Data

Lifecycle hooks receive a map containing:
- `:replicant/node` - The actual DOM element
- `:replicant/life-cycle` - `:mount`, `:render`, or `:unmount`
- `:replicant/remember` - Function to store data (persists across renders)
- `:replicant/memory` - Previously remembered data
- `:replicant/dispatch` - The raw global dispatch function set via `d/set-dispatch!` (added in 2025.12.1)

### CRITICAL: Dispatch Calling Convention in Hooks

**`:replicant/dispatch` in function hooks is the RAW function registered via `d/set-dispatch!`.** It requires **two arguments** matching your dispatch integration signature:

```clojure
;; d/set-dispatch! wires: #(dispatch-fn %1 %2)
;; %1 = dispatch-data (map), %2 = actions vector

;; CORRECT — two arguments
(fn [{:replicant/keys [dispatch]}]
  (dispatch {} [[:my/action {:param "value"}]]))

;; WRONG — one argument: actions become dispatch-data, actions get nil
(fn [{:replicant/keys [dispatch]}]
  (dispatch [[:my/action {:param "value"}]]))  ; Silent no-op!
```

### CRITICAL: No Dispatch-Triggered Re-Render During Mount

**Dispatching from `on-mount` that updates the state atom will NOT trigger a visible re-render.** Replicant suppresses nested `d/render` calls during its own render/mount cycle.

**Correct approach — seed state before the first render**, not from `on-mount`.

### Pattern: Auto-Scroll on Mount (Data-Only, 2025.12.1+)

**PREFERRED**: Keep pure data hiccup by using effects that read `:replicant/node` from `:dispatch-data`.

```clojure
;; 1. In view - pure data, dispatches effect directly
[:div {:replicant/on-mount [[:app/scroll-into-view!]]}
 content]

;; 2. In effects - read :replicant/node from dispatch-data context
#?(:cljs
   (defn scroll-into-view-effect
     [ctx _system _params]
     (when-let [node (some-> ctx :dispatch-data :replicant/node)]
       (.scrollIntoView node #js {:behavior "smooth" :block "end"}))))

;; 3. Register effect
(register-effect! :app/scroll-into-view! scroll-into-view-effect)
```

### When to Use Each Hook

| Scenario | Hook |
|----------|------|
| Scroll newly added element into view | `:replicant/on-mount` |
| Focus input field on first render | `:replicant/on-mount` |
| Measure element dimensions | `:replicant/on-render` |
| Cleanup resources (timers, subscriptions) | `:replicant/on-unmount` |
| Update chart when data changes | `:replicant/on-render` |

## Web Component Sync Pattern

Web components (e.g., built on Lit) have async initialization that can desync from Replicant's vDOM. Use `on-render` hooks to re-assert state after each render.

### Why Desync Happens

Lit-based web components use IntersectionObserver, MutationObserver, and their own lifecycle. These async processes can override attributes set by Replicant. Once desynced, Replicant sees "no change" in subsequent vdom diffs and won't re-assert.

### Generic Boolean Sync Pattern

For web components with boolean attributes:

```clojure
;; Re-assert boolean properties via on-render hook
[:my-component {:open true
                :replicant/on-render
                (fn [{:replicant/keys [node]}]
                  (aset node "open" true))}
 content]
```

The hook uses property setters (not attributes) to trigger the web component's reactivity.

## RAF is Never Needed in Properly Architected Systems

**PRINCIPLE**: If you find yourself reaching for `requestAnimationFrame`, you have a data flow problem. Fix the architecture instead.

| "I need RAF for..." | Proper Solution |
|---------------------|-----------------|
| Wait for render to complete | Use `:replicant/on-mount` or `:replicant/on-render` |
| External data updates | All state through store -> actions coordinate effects |
| DOM reconciliation timing | Use `:replicant/on-render` (fires on every DOM modification) |
| Post-paint measurements | Use ResizeObserver or resize events |
| Animation timing | Use `transitionend` / `animationend` events |

## Memory Pattern

- **`:replicant/remember`** - Function to store values (stored in WeakMap by DOM node)
- **`:replicant/memory`** - Retrieve previously stored values
- **`(replicant.dom/recall node)`** - Access memory from outside hooks

```clojure
;; Store on mount
{:replicant/on-mount
 (fn [{:replicant/keys [node remember]}]
   (let [instance (create-instance node)]
     (remember {:instance instance})))}

;; Access in later hooks
{:replicant/on-render
 (fn [{:replicant/keys [memory]}]
   (when-let [instance (:instance memory)]
     (update-instance instance)))}

;; Access from outside (e.g., in effects)
(when-let [node (js/document.getElementById "my-element")]
  (when-let [memory (d/recall node)]
    (update-instance (:instance memory))))
```

## JavaScript Interop

**Integrate third-party JavaScript libraries using lifecycle hooks.**

### Integration Strategy (3 Steps)

1. **Create minimal HTML/JS example** - Verify the library works independently
2. **Translate to ClojureScript** - Catch API misunderstandings early
3. **Integrate into Replicant** - Use lifecycle hooks for DOM access

### Pattern: Mounting a JS Library

```clojure
(defn chart-component [{:keys [data chart-options]}]
  [:div
   {:replicant/on-mount
    (fn [{:replicant/keys [node remember dispatch]}]
      (let [chart (js/Chart. node (clj->js chart-options))]
        (dispatch {} [[:chart/initialized {:id (:id chart-options)}]])
        (remember {:chart chart})))

    :replicant/on-render
    (fn [{:replicant/keys [memory]}]
      (when-let [chart (:chart memory)]
        (.update chart)))

    :replicant/on-unmount
    (fn [{:replicant/keys [memory dispatch]}]
      (when-let [chart (:chart memory)]
        (.destroy chart)
        (dispatch {} [[:chart/destroyed]])))}

   [:canvas {:id "my-chart"}]])
```

### Best Practices

| Practice | Reason |
|----------|--------|
| Use `^js` type hints | Prevents property mangling in advanced compilation |
| Add `:key` for substantial data changes | Forces remount when data fundamentally changes |
| Prefer update hooks over remount | Smoother UX, preserves internal library state |
| Store library instances in memory | Access across lifecycle without globals |
