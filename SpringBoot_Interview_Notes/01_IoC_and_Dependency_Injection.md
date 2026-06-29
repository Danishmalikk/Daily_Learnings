# Spring Boot Notes — IoC Container & Dependency Injection

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What problem are we even solving?

Imagine a class that needs another class to work:

```java
class OrderService {
    private EmailService email = new EmailService();  // I created my own dependency
}
```

Problem: `OrderService` is **tightly coupled** to `EmailService`. If `EmailService` needs a database, or you want to swap it for `SmsService`, or test it with a fake — you must edit `OrderService` every time. That's rigid and hard to test.

**The fix:** Don't let a class build its own dependencies. Let someone else *give* them to it. That "someone else" is the **Spring IoC Container**.

---

## 2. Inversion of Control (IoC)

*Definition:* **Inversion of Control (IoC)** means flipping who is in charge of creating and connecting objects. Normally *your* class creates the objects it needs. With IoC, you hand that job over to a **container** — it creates the objects, wires them together, and gives them to you ready to use.

Real life: At a restaurant you don't go into the kitchen to cook (control). You order, and the kitchen prepares and serves it (control inverted to the kitchen). The Spring container *is* that kitchen.

```
Normal:   You → create objects yourself → use them
IoC:      Container → creates objects → hands them to you
```

> "Inversion" = the *control of object creation* is inverted — taken away from your class and given to the framework.

**Interview one-liner:** IoC means the framework (not your code) controls the creation and wiring of objects — your class just declares what it needs.

---

## 3. The IoC Container (ApplicationContext)

*Definition:* The **IoC Container** is the part of Spring that creates objects, stores them, connects them to each other, and manages their full life. In Spring these managed objects are called **beans**. The container is represented by the **`ApplicationContext`**.

```
+------------------ IoC Container (ApplicationContext) ------------------+
|   Reads your config/annotations                                       |
|   Creates beans (OrderService, EmailService, Repository...)           |
|   Injects dependencies between them                                   |
|   Manages their lifecycle (create → use → destroy)                    |
+-----------------------------------------------------------------------+
```

- A **bean** = any object the Spring container creates and manages.
- The container scans your code, finds classes marked as beans, builds them, and keeps them in one place (sometimes called the *application context*).

**Interview one-liner:** The IoC container (ApplicationContext) creates, wires, and manages the lifecycle of objects called beans.

---

## 4. Dependency Injection (DI) — how IoC actually happens

*Definition:* **Dependency Injection (DI)** is the technique the container uses to *do* IoC: it **injects** (passes in) the objects a class needs from outside, instead of the class creating them itself.

> IoC is the **idea** (framework is in control). DI is the **way** it's done (it injects dependencies). Don't confuse them — DI is *how* you achieve IoC.

Real life: You (a class) need electricity. You don't build a power plant inside your house — the grid *injects* power through a socket. You just declare "I need a plug here."

### Before DI (bad — tightly coupled)
```java
class OrderService {
    private EmailService email = new EmailService();  // hard-wired
}
```

### After DI (good — loosely coupled)
```java
@Service
class OrderService {
    private final EmailService email;

    OrderService(EmailService email) {   // container injects it here
        this.email = email;
    }
}
```
Now `OrderService` doesn't care *how* `EmailService` is built. The container provides it. Easy to swap, easy to test (inject a fake).

**Interview one-liner:** Dependency Injection is how IoC works — the container passes a class its dependencies from outside instead of the class creating them with `new`.

---

## 5. The 3 Types of Dependency Injection

| Type | How | Recommended? |
|------|-----|--------------|
| **Constructor injection** | Pass dependencies through the constructor | ✅ **Best** (preferred) |
| **Setter injection** | Pass them through setter methods | ⚠️ For optional dependencies |
| **Field injection** | Put `@Autowired` directly on the field | ❌ Avoid (hard to test) |

### Constructor injection (preferred)
```java
@Service
class OrderService {
    private final EmailService email;

    @Autowired   // optional since Spring 4.3 if there's only one constructor
    OrderService(EmailService email) {
        this.email = email;
    }
}
```

### Setter injection
```java
@Service
class OrderService {
    private EmailService email;

    @Autowired
    void setEmail(EmailService email) { this.email = email; }
}
```

### Field injection (avoid)
```java
@Service
class OrderService {
    @Autowired
    private EmailService email;   // looks clean, but hard to test, hides dependencies
}
```

### Why constructor injection wins
- Dependencies can be `final` → **immutable**, can't accidentally change.
- The object is **fully built** the moment it's created (no half-ready state).
- Makes dependencies **obvious** — you can see them all in the constructor.
- **Easy to test** — just `new OrderService(fakeEmail)` in a unit test, no Spring needed.

**Easy rule of thumb:** Always use **constructor injection**. Use setter only for truly optional dependencies. Never use field injection in real code.

**Interview one-liner:** Three DI types — constructor (best, immutable + testable), setter (optional deps), field (avoid). Prefer constructor injection.

---

## 6. How does the container know what to create? (@Component & @Autowired)

You mark classes so the container picks them up during **component scanning** (Spring auto-searches your packages for these annotations):

```java
@Component                 // "Hey container, manage me as a bean"
class EmailService { }

@Service                   // @Service is a @Component (more on this later)
class OrderService {
    private final EmailService email;
    OrderService(EmailService email) { this.email = email; }  // auto-injected
}
```

- `@Component` (and friends `@Service`, `@Repository`, `@Controller`) = "register this class as a bean".
- `@Autowired` = "inject the matching bean here". On a single constructor it's optional.

> By default Spring beans are **singletons** — the container creates **one** shared instance and injects that same one everywhere. (More in the Bean Scopes file.)

**Interview one-liner:** `@Component`/`@Service`/etc. register a class as a bean; component scanning finds them; `@Autowired` injects the needed bean (constructor `@Autowired` is optional).

---

## 7. What if there are two matching beans? (@Qualifier & @Primary)

If two beans match one type, Spring gets confused ("which one?!") and throws an error. Fix it:

```java
interface PaymentService {}

@Service @Primary                       // default winner if ambiguous
class UpiPayment implements PaymentService {}

@Service @Qualifier("card")
class CardPayment implements PaymentService {}

@Service
class Checkout {
    Checkout(@Qualifier("card") PaymentService p) { }  // explicitly pick "card"
}
```

- `@Primary` → "if confused, pick me by default."
- `@Qualifier("name")` → "inject this *specific* named bean."

**Interview one-liner:** When multiple beans match, use `@Primary` to set a default or `@Qualifier` to pick a specific bean by name.

---

## 8. IoC vs DI — the classic interview trap

| | IoC | DI |
|--|-----|-----|
| What is it | A **principle** — framework controls object creation | A **pattern/technique** — inject dependencies from outside |
| Question it answers | *Who* is in control? (the framework) | *How* are dependencies provided? (by injection) |
| Relationship | The broad idea | One concrete way to implement that idea |

> Think: **IoC is the goal, DI is the method.** Spring achieves IoC *through* DI.

**Interview one-liner:** IoC is the principle (framework controls object creation); DI is the technique that implements it by injecting dependencies. DI is one way to achieve IoC.

---

## Quick Revision Cheat Sheet

- **Problem**: `new`-ing dependencies = tight coupling, hard to test.
- **IoC** = framework (not your code) controls object creation & wiring.
- **IoC Container** = `ApplicationContext`; managed objects = **beans**.
- **DI** = the technique that does IoC — dependencies injected from outside.
- **IoC = the idea; DI = how it's done.**
- **3 DI types**: Constructor (✅ best, immutable+testable), Setter (optional), Field (❌ avoid).
- **@Component/@Service/...** register beans; **@Autowired** injects them.
- **@Primary** = default bean; **@Qualifier** = pick a specific named bean.
- Beans are **singletons** by default (one shared instance).
