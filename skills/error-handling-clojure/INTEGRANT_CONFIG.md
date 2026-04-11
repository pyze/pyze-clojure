# Integrant Configuration Gotchas

Understanding Integrant configuration as code and avoiding common setup issues.

## Core Principle: Configuration is Code

**Service implementations must be explicitly declared in Integrant EDN configuration files.** Creating the implementation namespace isn't sufficient - the component must be wired into the system configuration with proper dependencies.

```clojure
;; Implementation exists in src/my/service.clj
(defmethod ig/init-key :my/service [_ config]
  (create-service config))

;; But if not wired in config.edn or profiles.edn...
;; Handler receives NIL service -> 500 errors!
```

**Common symptom**: Handler works in tests but fails at runtime with nil service errors.

---

## Debugging Pattern

When you get nil service errors, follow this diagnostic process:

### Step 1: Check System Keys

```clojure
;; Check if service is loaded in system
(keys @user/system)
;; => (:server/http :my/service ...)  ; Is :my/service here?
```

**If missing**: Service is not in Integrant config.

### Step 2: Verify Service Retrieval

```clojure
;; Verify service retrieval
(get @user/system :my/service)
;; => nil means it's not in config!
```

**If nil**: Check configuration files for missing service declaration.

### Step 3: Check Configuration Files

**Configuration location**: `resources/config/system.edn`

```edn
;; Ensure service is declared
{:my/service {:api-key #env "MY_API_KEY"
              :timeout 5000}

 ;; And injected into handlers that need it
 :handler/my-handler {:service #ig/ref :my/service
                      :db #ig/ref :db/connection}}
```

---

## Common Configuration Mistakes

### Mistake 1: Implementation Without Configuration

```clojure
;; ❌ Implementation exists
(ns my.service
  (:require [integrant.core :as ig]))

(defmethod ig/init-key :my/service [_ config]
  (println "Initializing service")
  (create-service config))
```

```edn
;; ❌ But NOT in system.edn
{:server/http {...}
 :handler/routes {...}}
 ;; :my/service is missing!
```

**Result**: Service never initialized, handlers get nil.

### Mistake 2: Missing Dependency References

```clojure
;; ❌ Handler expects :service key
(defn my-handler [request]
  (let [service (get-in request [:system :service])]  ; Expects :service
    (process-with-service service request)))
```

```edn
;; ❌ But config doesn't inject it
{:handler/my-handler {:db #ig/ref :db/connection}
                     ;; Missing :service reference!
 :my/service {:timeout 5000}}
```

**Result**: Handler gets nil for service.

### Mistake 3: Wrong Service Key

```clojure
;; Service defined as :services/my-service
(defmethod ig/init-key :services/my-service [_ config]
  (create-service config))
```

```edn
;; Handler references :my/service (wrong key!)
{:handler/my-handler {:service #ig/ref :my/service}  ; Wrong!
 :services/my-service {:timeout 5000}}
```

**Result**: Reference doesn't match, handler gets nil.

---

## Prevention Checklist

After implementing a new service, verify:

- [ ] **Service implementation** exists with `ig/init-key` method
- [ ] **Configuration entry** added to `resources/config/system.edn`
- [ ] **Dependencies** correctly referenced with `#ig/ref`
- [ ] **Key names** match between implementation and config
- [ ] **REPL verification**: `(keys @user/system)` includes service key
- [ ] **Service retrieval**: `(get @user/system :service/name)` returns service, not nil
- [ ] **Handler injection**: Handlers receive service via request or direct injection

---

## Testing Service Availability

### At REPL

```clojure
;; 1. Check system has service
(contains? @user/system :my/service)
;; => true (service configured) or false (missing)

;; 2. Get service instance
(get @user/system :my/service)
;; => #object[...] (service exists) or nil (not configured)

;; 3. List all services
(keys @user/system)
;; => (:server/http :db/connection :my/service ...)
```

### In Tests

```clojure
(deftest service-configured-test
  (testing "Service is configured in system"
    (let [system (ig/init test-config)]
      (is (contains? system :my/service)
          "Service should be in system")
      (is (some? (get system :my/service))
          "Service should not be nil"))))
```

---

## Configuration File Locations

**Standard locations**:
- `resources/config/system.edn` - Base system configuration
- `resources/config/profiles.edn` - Environment-specific overrides
- `dev/user.clj` - Development REPL configuration

**Loading order**:
1. Load base `system.edn`
2. Merge profile-specific config (dev, test, prod)
3. Initialize system with merged config

---

## Debugging Failed Initialization

### Common Errors

#### Error: "No method in multimethod 'init-key'"

```
No method in multimethod 'init-key' for dispatch value: :my/service
```

**Cause**: Implementation namespace not loaded or `ig/init-key` method not defined.

**Fix**: Ensure namespace with `defmethod ig/init-key :my/service` is required.

#### Error: "Invalid reference"

```
Invalid reference: #ig/ref :my/service
```

**Cause**: Referenced key doesn't exist in config.

**Fix**: Add `:my/service` entry to config file.

#### Error: Service returns nil

**Cause**: `ig/init-key` implementation returns nil.

**Fix**: Check implementation returns actual service instance.

---

## Best Practices

1. **Configuration is code** - Treat EDN files like source code
2. **Verify at REPL** - Always check `(keys @user/system)` after adding service
3. **Test service retrieval** - Ensure `(get @user/system :service/name)` works
4. **Consistent naming** - Use same key in implementation and config
5. **Document dependencies** - Comment config files with what services need
6. **Version control** - Track all config changes in git
7. **Environment variables** - Use `#env` for secrets, not hardcoded values

---

## Summary

**Key insights**:
- Implementation namespace ≠ configured service
- Configuration must explicitly wire services into system
- Use REPL to verify services are loaded
- Check `(keys @user/system)` and `(get @user/system :service/name)`
- Missing config = nil service = 500 errors

**Rule of thumb**: If service exists in code but not in `(keys @user/system)`, it's a configuration issue.
