# Spring Boot Notes — Bean Scopes (Singleton, Prototype, Request, Session)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Bean Scope?

*Definition:* A **bean scope** decides **how many instances** of a bean Spring creates and **how long each one lives** (when it's shared vs when a fresh one is made). It answers: "When I ask for this bean, do I get the same shared object, or a brand-new one?"

Real life:
- **Singleton** = the office printer — one shared by everyone.
- **Prototype** = a fresh paper cup each time you want water.
- **Request** = a visitor badge, valid for one visit (one web request).
- **Session** = a gym membership card, valid for your whole membership period (one user session).

---

## 2. The Main Scopes at a glance

| Scope | How many instances | Lives for | Where it works |
|-------|--------------------|-----------|----------------|
| **singleton** (default) | One per container | Whole app | Everywhere |
| **prototype** | New one every time you ask | Until garbage collected | Everywhere |
| **request** | One per HTTP request | One request | Web apps only |
| **session** | One per user session | One user's session | Web apps only |
| **application** | One per ServletContext | Whole web app | Web apps only |

> `singleton` and `prototype` are **core** (any Spring app). `request`, `session`, `application` are **web-only**.

How to set a scope:
```java
@Service
@Scope("prototype")          // or @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class ReportGenerator { }
```

---

## 3. Singleton Scope (the default)

*Definition:* **Singleton** means Spring creates **exactly one** instance of the bean for the whole container and **shares that same object** everywhere it's injected.

```java
@Service                 // singleton by default — no annotation needed
class UserService { }
```

```java
@Autowired UserService a;
@Autowired UserService b;
// a == b  → true (same single shared object)
```

- Created once, eagerly, when the container starts.
- Shared everywhere → memory-efficient.

> ⚠️ **Biggest gotcha:** Because one object is shared by all threads, a singleton must be **stateless** (no mutable instance fields that change per user). Storing per-user data in a singleton field = race conditions and data bleeding between users.

```java
@Service
class CartService {
    private List<Item> items;   // ❌ DANGER: shared across all users!
}
```

**Interview one-liner:** Singleton (the default) = one shared instance per container. Keep it stateless because all threads share it.

---

## 4. Prototype Scope

*Definition:* **Prototype** means Spring creates a **brand-new instance every single time** the bean is requested or injected. Nothing is shared.

```java
@Service
@Scope("prototype")
class ReportGenerator { }
```

```java
@Autowired ReportGenerator a;
@Autowired ReportGenerator b;
// a != b  → false (two different objects)
```

- Use when each user/operation needs its **own separate state**.
- Spring creates it and hands it over — but then **stops managing it**. `@PreDestroy` is **NOT** called for prototypes; you handle cleanup.

**Interview one-liner:** Prototype = a new instance every time it's requested; use it for stateful objects. Spring doesn't manage prototype destruction.

---

## 5. The Singleton-injecting-Prototype trap (very common interview question!)

Problem: You inject a **prototype** bean into a **singleton** bean. Since the singleton is created only once, its prototype dependency is also injected only **once** — so you keep getting the **same** prototype forever. The "new every time" promise is broken!

```java
@Service                          // singleton
class OrderService {
    @Autowired
    private ReportGenerator gen;   // prototype, but injected only ONCE → always same
}
```

### Fixes
1. **`ObjectProvider`** (modern, clean):
```java
@Service
class OrderService {
    private final ObjectProvider<ReportGenerator> genProvider;
    OrderService(ObjectProvider<ReportGenerator> p) { this.genProvider = p; }

    void run() {
        ReportGenerator gen = genProvider.getObject();  // fresh prototype each call
    }
}
```
2. **`@Lookup`** method injection.
3. **Scoped proxy** (`@Scope(value="prototype", proxyMode=TARGET_CLASS)`).

**Interview one-liner:** A prototype injected into a singleton is only injected once, so you keep the same instance. Use `ObjectProvider.getObject()`, `@Lookup`, or a scoped proxy to get a fresh one each time.

---

## 6. Web Scopes — request, session, application

These only exist in a **web application** (HTTP context).

### Request scope
*Definition:* **Request** scope creates **one bean instance per HTTP request**. When the request ends, the bean is gone. A different request gets its own instance.

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
class RequestContext {
    private String requestId;   // safe — isolated per request
}
```
Use for: per-request data like a request ID, the current user of *this* request, request timing.

### Session scope
*Definition:* **Session** scope creates **one bean instance per user session** (the period a user is active). It persists across that user's many requests until their session ends.

```java
@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
class UserCart {
    private List<Item> items = new ArrayList<>();   // one cart per user
}
```
Use for: shopping cart, logged-in user preferences, multi-step wizard data.

### Application scope
*Definition:* **Application** scope = one instance shared across the **entire web application** (per `ServletContext`). Similar to singleton, but tied to the web app context rather than the Spring container.

> **Why `proxyMode` for web scopes?** A singleton bean lives longer than a request/session. To inject a short-lived request/session bean into a long-lived singleton, Spring injects a **proxy** (a stand-in) that fetches the correct current-request/session instance on each call.

**Interview one-liner:** Request = one bean per HTTP request; Session = one per user session; Application = one per web app. They need a scoped proxy when injected into longer-lived beans.

---

## 7. Quick decision guide

| Need | Use |
|------|-----|
| Shared, stateless service (most beans) | **singleton** |
| Fresh stateful object each time | **prototype** |
| Data for the current HTTP request only | **request** |
| Data that persists across a user's session | **session** |

**Easy rule of thumb:** 95% of your beans should stay **singleton** (and stateless). Reach for other scopes only when you genuinely need per-instance, per-request, or per-user state.

---

## Quick Revision Cheat Sheet

- **Scope** = how many instances + how long they live.
- **singleton** (default): one shared instance per container → must be **stateless**.
- **prototype**: new instance every request; Spring won't call `@PreDestroy` on it.
- **Singleton-with-prototype trap**: prototype injected once → use `ObjectProvider`/`@Lookup`/scoped proxy.
- **request**: one per HTTP request (web only).
- **session**: one per user session (web only) — e.g. shopping cart.
- **application**: one per whole web app.
- Web scopes injected into singletons need a **scoped proxy** (`proxyMode`).
- Default to **singleton + stateless**; use others only for needed state.
