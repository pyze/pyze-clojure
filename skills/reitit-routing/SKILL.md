---
name: reitit-routing
description: Configure server and client routing with Reitit. Use when adding routes, implementing navigation, or handling browser history.
---

# Reitit Routing Skill

## When to Use This Skill

**Use THIS skill if:**
- Adding new routes (server or client)
- Implementing navigation between views
- Working with browser history (push-state, replace-state)
- Setting up route parameter extraction
- Configuring dev server push-state fallback

**Use DIFFERENT skill if:**
- HTTP request/response patterns (not routing) → check project-specific skills

## Overview

Use **reitit** for all routing in this project - both server-side (API routes) and client-side (browser navigation with history). Reitit provides data-driven, consistent routing across the full stack.

## Why Reitit

- **Consistent** - Same library for backend API routes and frontend navigation
- **Data-driven** - Routes are plain data, easily composed and validated
- **Fast** - Linear-time router implementation
- **Bi-directional** - Generate URLs from route names (reverse routing)
- **Coercion** - Built-in parameter validation with spec/malli

## Server-Side Routing (reitit-ring)

### Route Definition

```clojure
(require '[reitit.ring :as ring])

(def app
  (ring/ring-handler
    (ring/router
      [["/api"
        ["/health" {:get {:handler health-handler}}]
        ["/users/:id" {:get {:handler get-user}
                       :put {:handler update-user}}]
        ["/items" {:get {:handler list-items}
                   :post {:handler create-item}}]]

       ;; Nested routes with shared middleware
       ["/admin" {:middleware [auth-middleware]}
        ["/dashboard" {:get {:handler admin-dashboard}}]]])))
```

### Key Patterns

1. **Nested Routes** - Share path prefixes and middleware
2. **Method Handlers** - `:get`, `:post`, `:put`, `:delete` per route
3. **Path Parameters** - `:id` syntax extracts to `:path-params`
4. **Route Data** - Attach any data (middleware, handlers, metadata)

## Client-Side Routing (reitit-frontend)

### Route Definition

```clojure
(ns app.router
  (:require [reitit.frontend :as rf]
            [reitit.frontend.easy :as rfe]
            [reitit.coercion.spec :as rss]))

(def routes
  [["/" {:name ::dashboard}]
   ["/users/:id" {:name ::user
                  :parameters {:path {:id string?}}}]
   ["/items/:category/:item" {:name ::item
                              :parameters {:path {:category string?
                                                  :item string?}}}]])
```

### Starting the Router

```clojure
(defn on-navigate [match]
  ;; Called when URL changes (navigation, back/forward, direct URL)
  (when match
    (let [route-name (-> match :data :name)
          path-params (:path-params match)]
      (dispatch! [[:navigate-from-url {:view route-name
                                       :params path-params}]]))))

(defn start! [dispatch!]
  (rfe/start!
    (rf/router routes {:data {:coercion rss/coercion}})
    (partial on-navigate dispatch!)
    {:use-fragment false}))  ;; HTML5 history, not hash-based
```

### Navigation Functions

```clojure
;; Programmatic navigation (updates URL + triggers on-navigate)
(rfe/push-state ::user {:id "123"})           ;; /users/123
(rfe/push-state ::item {:category "books"
                        :item "clojure"})     ;; /items/books/clojure

;; Replace current history entry (no new history)
(rfe/replace-state ::dashboard)

;; Generate URL without navigating
(rfe/href ::user {:id "123"})                 ;; "/users/123"
```

### Event System Integration Pattern

**Effects call reitit navigation:**

```clojure
;; effects/router.cljs
(defn navigate-user-effect [{:keys [user-id]}]
  (rfe/push-state ::user {:id user-id}))
```

**Event handlers dispatch router effects:**

```clojure
;; handlers.cljc
(defn select-user-handler [db {:keys [user-id]}]
  ;; Update state and trigger navigation
  (-> db
      (assoc-in [:users user-id :selected?] true))
  ;; Side effect: navigate
  (navigate-user-effect {:user-id user-id}))
```

**Router calls back with handler:**

```clojure
;; router.cljs on-navigate
(defn on-navigate [dispatch! match]
  (when match
    (dispatch! {:type :app/navigate-from-url
                :view (-> match :data :name)
                :params (:path-params match)})))
```

## URL Design Guidelines

### Structure

```
/                                    → Dashboard/home
/resource                            → Resource list
/resource/:id                        → Resource detail
/resource/:id/sub-resource           → Nested resource
/resource/:id/sub-resource/:sub-id   → Nested detail
```

### Path Parameters

- Use kebab-case: `/case-type/:case-type` not `/caseType/:caseType`
- URL-encode special characters: `encodeURIComponent`/`decodeURIComponent`
- Keep paths readable and bookmarkable

### Query Parameters vs Path Parameters

- **Path**: Required, identifies resource (`/users/123`)
- **Query**: Optional, filters/modifies (`/users?role=admin&active=true`)

## History Behavior

| Action | History Effect |
|--------|----------------|
| `push-state` | Adds entry (back button returns here) |
| `replace-state` | Replaces current (no new history) |
| Browser back | Triggers `on-navigate` with previous URL |
| Direct URL | Triggers `on-navigate` on page load |

**When to use each:**

- **push-state**: Main navigation (dashboard → detail → sub-detail)
- **replace-state**: State updates that shouldn't create history (tab switches, filters)

## Dev Server Configuration

For direct URL access in development, configure Shadow-CLJS push-state fallback:

```clojure
;; shadow-cljs.edn
{:dev-http {8080 {:root "resources/public"
                  :push-state/index "index.html"}}}
```

**Note**: When using `:proxy-url`, ensure the backend returns 200 for unknown routes or configure routing priority.

## Production Server Configuration

The backend must serve `index.html` for all client routes:

```clojure
;; Reitit ring handler with SPA fallback
(def app
  (ring/ring-handler
    (ring/router
      [["/api" api-routes]])
    ;; Default handler serves index.html for client routes
    (ring/routes
      (ring/create-resource-handler {:path "/"})
      (ring/create-default-handler
        {:not-found (fn [_]
                      {:status 200
                       :headers {"Content-Type" "text/html"}
                       :body (slurp (io/resource "public/index.html"))})}))))
```

## Common Patterns

### Breadcrumb Navigation

```clojure
;; Route with parent info
["/items/:category/:item"
 {:name ::item
  :breadcrumb [{:name ::home :label "Home"}
               {:name ::category :label #(str "Category: " %)}
               {:name ::item :label #(str "Item: " %)}]}]
```

### Protected Routes

```clojure
;; Check auth in on-navigate
(defn on-navigate [dispatch! match]
  (when match
    (if (and (requires-auth? match) (not (authenticated?)))
      (rfe/replace-state ::login {:redirect (rfe/href (-> match :data :name)
                                                       (:path-params match))})
      (dispatch! [[:navigate-from-url ...]]))))
```

### Route Controllers (Lifecycle)

```clojure
(require '[reitit.frontend.controllers :as rfc])

(def routes
  [["/user/:id"
    {:name ::user
     :controllers [{:parameters {:path [:id]}
                    :start (fn [{:keys [path]}]
                             (js/console.log "Loading user" (:id path)))
                    :stop (fn [_]
                            (js/console.log "Leaving user page"))}]}]])
```

## Anti-Patterns

### Don't Mix Router Libraries

```clojure
;; WRONG - using multiple routers
(:require [reitit.frontend.easy :as rfe]
          [secretary.core :as secretary])  ;; Don't mix!
```

### Don't Manipulate History Directly

```clojure
;; WRONG - bypasses reitit
(.pushState js/history nil "" "/new-url")

;; RIGHT - use reitit
(rfe/push-state ::route {:param value})
```

### Don't Duplicate URL Logic

```clojure
;; WRONG - URL building scattered
(str "/users/" user-id "/items/" item-id)

;; RIGHT - use rfe/href
(rfe/href ::user-item {:user-id user-id :item-id item-id})
```

## Testing Routes

```clojure
(require '[reitit.core :as r])

(def router (r/router routes))

;; Test path matching
(r/match-by-path router "/users/123")
;; => {:template "/users/:id", :path-params {:id "123"}, ...}

;; Test URL generation
(r/match-by-name router ::user {:id "123"})
;; => {:path "/users/123", ...}
```

## References

- [Reitit Documentation](https://cljdoc.org/d/metosin/reitit)
- [Frontend Routing Guide](https://cljdoc.org/d/metosin/reitit/0.7.2/doc/frontend/basics)
- [Ring Integration](https://cljdoc.org/d/metosin/reitit/0.7.2/doc/ring/basics)
