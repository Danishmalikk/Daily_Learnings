# Spring Boot Notes — @Component vs @Service vs @Repository vs @Controller

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The big idea: they're all "register me as a bean"

*Definition:* These four are called **stereotype annotations**. They all tell Spring "create and manage this class as a bean" (so component scanning picks it up). The difference is mostly about **role/intention** — which *layer* of your app the class belongs to.

> Key fact for interviews: **`@Service`, `@Repository`, and `@Controller` are all specializations of `@Component`.** Open their source and you'll literally see `@Component` on top of each. So they're the *same* at the core, just with extra meaning (and sometimes extra behavior).

```
            @Component  (generic bean)
            /     |      \
    @Service  @Repository  @Controller
   (business) (data layer) (web layer)
                              |
                        @RestController
                     (@Controller + @ResponseBody)
```

---

## 2. The typical 3-layer architecture

Spring apps are usually built in layers. Each annotation marks a layer:

```
   HTTP request
        ↓
  @Controller  →  handles web requests, returns response
        ↓
  @Service     →  business logic, rules, calculations
        ↓
  @Repository  →  talks to the database
        ↓
     Database
```

`@Component` = the generic fallback for anything that doesn't fit a specific layer (a utility, a helper, a config holder).

---

## 3. @Component — the generic bean

*Definition:* **`@Component`** is the base, all-purpose stereotype. Use it for any class you want Spring to manage that doesn't clearly belong to the service, repository, or controller layer.

```java
@Component
class PriceFormatter {
    String format(double amount) { return "₹" + amount; }
}
```

Use for: utility/helper classes, custom converters, generic beans.

**Interview one-liner:** `@Component` is the generic stereotype that registers any class as a Spring bean; the others are specialized versions of it.

---

## 4. @Service — the business logic layer

*Definition:* **`@Service`** marks a class that holds **business logic** — the rules, calculations, and decisions of your app. Technically it behaves like `@Component`, but it clearly signals "this is a service-layer class."

```java
@Service
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo) { this.repo = repo; }

    void placeOrder(Order o) {
        if (o.getAmount() <= 0) throw new IllegalArgumentException("Invalid amount");
        // business rules here
        repo.save(o);
    }
}
```

Use for: the "brain" of your app — validation, workflows, coordinating repositories.

**Interview one-liner:** `@Service` marks the business-logic layer; functionally same as `@Component` but communicates intent.

---

## 5. @Repository — the data access layer

*Definition:* **`@Repository`** marks a class that **talks to the database** (data access layer / DAO). Unlike the others, it adds a **real extra feature**: it automatically translates low-level, database-specific exceptions into Spring's clean, unified `DataAccessException`.

```java
@Repository
interface OrderRepository extends JpaRepository<Order, Long> {
    // Spring Data gives you save(), findById(), findAll()... for free
}
```

> **The one with extra behavior:** `@Repository` enables **exception translation** — e.g. a vendor-specific `SQLException` gets converted into Spring's consistent `DataAccessException`, so your code isn't tied to a specific database's error types. This is the key "why is it different" interview point.

**Interview one-liner:** `@Repository` marks the data layer and adds automatic exception translation (DB-specific exceptions → Spring's `DataAccessException`).

---

## 6. @Controller — the web layer (returns views)

*Definition:* **`@Controller`** marks a class that **handles incoming web requests**. By default it's used in MVC apps that return **views** (HTML pages, e.g. Thymeleaf templates) — the returned String is treated as a *view name*.

```java
@Controller
class PageController {
    @GetMapping("/home")
    String home(Model model) {
        model.addAttribute("user", "Danish");
        return "home";        // → renders home.html (a VIEW name)
    }
}
```

**Interview one-liner:** `@Controller` handles web requests and (by default) returns view names for rendering HTML pages.

---

## 7. @RestController — the web layer (returns data/JSON)

*Definition:* **`@RestController`** is a shortcut = **`@Controller` + `@ResponseBody`**. It's used for REST APIs: instead of returning a view name, whatever you return is sent **directly as the response body** (usually JSON).

```java
@RestController
class OrderApi {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id) {
        return new Order(id, 500);   // auto-converted to JSON
    }
}
```

> With `@Controller`, `return "home"` means "render home.html". With `@RestController`, `return order` means "serialize this object to JSON and send it." (Full comparison in the RestController vs Controller file.)

**Interview one-liner:** `@RestController` = `@Controller` + `@ResponseBody`; returns data (JSON) directly instead of a view — used for REST APIs.

---

## 8. Comparison table (the money table)

| Annotation | Layer | Specializes | Extra behavior | Returns |
|------------|-------|-------------|----------------|---------|
| **@Component** | Generic | — (base) | None | — |
| **@Service** | Business logic | @Component | None (intent only) | — |
| **@Repository** | Data access | @Component | ✅ Exception translation | — |
| **@Controller** | Web (MVC) | @Component | Handles requests | View name |
| **@RestController** | Web (REST API) | @Controller | + @ResponseBody | JSON/data |

---

## 9. "If they're the same, why not use @Component everywhere?"

A classic interview follow-up. Reasons:
1. **Readability** — `@Service` instantly tells any developer the class's role.
2. **Real behavior** — `@Repository` adds exception translation; `@Controller` enables request mapping. Those aren't just labels.
3. **Targeting** — tools and AOP can target a layer (e.g. "apply this aspect to all `@Service` classes").
4. **Convention** — clean layered architecture, easier to maintain.

**Easy rule of thumb:** Use the **most specific** annotation for the layer. Generic helper → `@Component`. Business logic → `@Service`. DB access → `@Repository`. Web endpoints → `@Controller` (views) or `@RestController` (JSON).

---

## Quick Revision Cheat Sheet

- All four register a class as a **bean**; `@Service`/`@Repository`/`@Controller` are **specializations of `@Component`**.
- **@Component**: generic bean (utilities, helpers).
- **@Service**: business-logic layer (intent only, no extra behavior).
- **@Repository**: data layer + **exception translation** (DB exceptions → `DataAccessException`) — the one with real extra behavior.
- **@Controller**: web layer, returns **view names** (HTML).
- **@RestController** = `@Controller` + `@ResponseBody`, returns **JSON/data**.
- Use the **most specific** annotation for readability, behavior, and targeting.
