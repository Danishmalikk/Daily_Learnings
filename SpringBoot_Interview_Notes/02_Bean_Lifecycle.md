# Spring Boot Notes — Bean Lifecycle (@PostConstruct, @PreDestroy)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is a Bean's Lifecycle?

*Definition:* A **bean** is an object managed by the Spring container. Its **lifecycle** is the full journey the container takes it through — from creating it, to injecting its dependencies, to letting you run setup code, to finally cleaning it up before the app shuts down.

Real life: Hiring an employee — recruit (create), assign a desk & tools (inject dependencies), onboarding/training (init), do the job (use), exit interview & return laptop (destroy).

> You don't call `new` and you don't clean up manually. The **container** does it. You just plug into specific moments with annotations.

---

## 2. The Lifecycle Steps (the big picture)

```
1. Container starts
2. Bean is instantiated        (object created)
3. Dependencies injected       (DI happens)
4. @PostConstruct runs         ← your setup code (init)
5. Bean is ready & used        (handles requests, does work)
   ... app runs ...
6. @PreDestroy runs            ← your cleanup code
7. Bean destroyed, container shuts down
```

The two hooks you care about most as a beginner are **steps 4 and 6**.

**Interview one-liner:** A bean is instantiated → dependencies injected → `@PostConstruct` (init) → used → `@PreDestroy` (cleanup) → destroyed, all managed by the container.

---

## 3. @PostConstruct — run setup code after the bean is ready

*Definition:* **`@PostConstruct`** marks a method that Spring runs **once**, right after the bean is created and all its dependencies are injected. Use it for any setup that needs those dependencies to already be in place.

Why not just use the constructor? Because in the constructor, injected fields might not be fully set yet (especially with setter/field injection). `@PostConstruct` runs *after* injection is complete — so it's the safe place for initialization.

```java
@Service
class CacheService {
    private final DatabaseService db;

    CacheService(DatabaseService db) { this.db = db; }

    @PostConstruct
    void loadCache() {
        // runs once, AFTER db is injected
        System.out.println("Warming up cache from DB...");
        // db.fetchAll() ...
    }
}
```

Common uses: pre-load a cache, validate config, open a connection, log "service ready".

**Interview one-liner:** `@PostConstruct` runs once after the bean is built and dependencies are injected — the right place for initialization logic that needs those dependencies.

---

## 4. @PreDestroy — run cleanup code before the bean dies

*Definition:* **`@PreDestroy`** marks a method Spring runs **once**, just before the bean is destroyed (usually at app shutdown). Use it to release resources so nothing leaks.

```java
@Service
class ConnectionService {
    @PreDestroy
    void cleanup() {
        System.out.println("Closing DB connections, flushing data...");
        // connection.close() ...
    }
}
```

Common uses: close DB/file connections, stop background threads, flush buffers, save final state.

> `@PreDestroy` only runs for **singleton** beans on a graceful shutdown. For **prototype** beans, Spring creates them but does **not** manage their destruction — *you* must clean those up. (See Bean Scopes file.)

**Interview one-liner:** `@PreDestroy` runs once before the bean is destroyed (at shutdown) — use it to release resources like connections and threads.

---

## 5. Full working example

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Service
class PaymentGateway {

    public PaymentGateway() {
        System.out.println("1. Constructor — object created");
    }

    @PostConstruct
    public void init() {
        System.out.println("2. @PostConstruct — connecting to payment provider");
    }

    public void pay() {
        System.out.println("3. Working — processing payment");
    }

    @PreDestroy
    public void shutdown() {
        System.out.println("4. @PreDestroy — closing connection");
    }
}
```

Output during app start → use → stop:
```
1. Constructor — object created
2. @PostConstruct — connecting to payment provider
3. Working — processing payment
4. @PreDestroy — closing connection
```

> Note: in Spring Boot 3+ these annotations come from **`jakarta.annotation`** (older Spring used `javax.annotation`). Common interview/gotcha point.

---

## 6. Other ways to hook the lifecycle (good to know)

There are 3 ways to run init/destroy logic. `@PostConstruct`/`@PreDestroy` is the cleanest and most common.

| Approach | Init | Destroy | Notes |
|----------|------|---------|-------|
| **Annotations** | `@PostConstruct` | `@PreDestroy` | ✅ Recommended, simple, standard |
| **Interfaces** | `InitializingBean.afterPropertiesSet()` | `DisposableBean.destroy()` | Ties your code to Spring interfaces |
| **Bean methods** | `@Bean(initMethod="..")` | `@Bean(destroyMethod="..")` | Good for third-party classes you can't annotate |

```java
// Interface approach
class MyBean implements InitializingBean, DisposableBean {
    public void afterPropertiesSet() { /* init */ }
    public void destroy() { /* cleanup */ }
}

// @Bean method approach (for classes you don't own)
@Bean(initMethod = "start", destroyMethod = "stop")
ExternalClient client() { return new ExternalClient(); }
```

**Easy rule of thumb:** Use **`@PostConstruct` / `@PreDestroy`** in your own classes. Use `@Bean(initMethod/destroyMethod)` for third-party classes you can't edit.

**Interview one-liner:** Three ways to hook init/destroy — annotations (`@PostConstruct`/`@PreDestroy`, preferred), Spring interfaces (`InitializingBean`/`DisposableBean`), or `@Bean` init/destroy methods (for third-party classes).

---

## 7. If init order matters (BeanPostProcessor — advanced, just be aware)

A **`BeanPostProcessor`** lets you run logic *around every bean's* initialization (before and after `@PostConstruct`). It's how Spring itself applies things like AOP proxies. As a beginner, just know it exists — interviewers sometimes ask "what runs around bean init?" → answer: `BeanPostProcessor`.

```
... dependencies injected
→ BeanPostProcessor.postProcessBeforeInitialization()
→ @PostConstruct
→ BeanPostProcessor.postProcessAfterInitialization()
→ bean ready
```

---

## Quick Revision Cheat Sheet

- **Lifecycle**: instantiate → inject deps → `@PostConstruct` → use → `@PreDestroy` → destroy.
- **@PostConstruct**: runs once after DI; for setup that needs injected deps (not the constructor).
- **@PreDestroy**: runs once before destroy (shutdown); release resources.
- **Prototype beans**: container does NOT call `@PreDestroy` — you clean up yourself.
- **Spring Boot 3+**: annotations live in `jakarta.annotation` (was `javax.annotation`).
- **3 hook styles**: annotations (best), `InitializingBean`/`DisposableBean`, `@Bean(initMethod/destroyMethod)` for third-party.
- **BeanPostProcessor** runs logic around every bean's init (how Spring adds AOP/proxies).
