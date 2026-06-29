# Spring Boot Notes — Spring AOP (Aspect, Advice, Pointcut, JoinPoint)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What problem does AOP solve?

Some code is needed in *many* places but isn't the *main job* of those places — like **logging**, **security checks**, **transactions**, **performance timing**. If you copy-paste it into every method, you get:
- Duplication everywhere.
- The real business logic buried under noise.

These repeated, scattered concerns are called **cross-cutting concerns** (they "cut across" many classes).

**AOP's idea:** Write that repeated code **once** in a separate place, and tell Spring "automatically run this around these methods." Your business methods stay clean.

---

## 2. Aspect-Oriented Programming (AOP)

*Definition:* **AOP (Aspect-Oriented Programming)** is a way to take "cross-cutting" code (logging, security, transactions) out of your business methods and put it in **one separate place**, then have Spring automatically weave it into the methods that need it — without you editing those methods.

Real life: A security guard at a building. Every person entering gets checked — but you don't put a guard *inside* each office. One guard at the entrance handles everyone. AOP is that guard, applied around your methods.

```
Without AOP:                    With AOP:
method1(){ log(); work(); }     method1(){ work(); }
method2(){ log(); work(); }     method2(){ work(); }    ← clean
method3(){ log(); work(); }     method3(){ work(); }
                                + one Aspect that adds log() around all of them
```

**Interview one-liner:** AOP separates cross-cutting concerns (logging, security, transactions) into reusable aspects that Spring weaves around your methods, keeping business code clean.

---

## 3. The 5 core terms (learn these — interviews love them)

```
@Aspect class LoggingAspect {

   @Before("execution(* com.app.service.*.*(..))")  ← Pointcut (WHERE)
   public void log(JoinPoint jp) {                   ← Advice (WHAT to do)
       // this runs at the JoinPoint (a method execution)
   }
}
```

| Term | Simple meaning |
|------|----------------|
| **Aspect** | The class holding the cross-cutting code (the "module" of concern) |
| **Advice** | The actual action + *when* it runs (before/after/around a method) |
| **Pointcut** | The expression that picks *which* methods to apply advice to |
| **JoinPoint** | A specific point where advice can run (in Spring = a method execution) |
| **Weaving** | The process of plugging the aspect into the target methods (Spring does it at runtime via proxies) |

### How they relate (one sentence)
An **Aspect** contains **Advice**, which runs at matching **JoinPoints** that are selected by a **Pointcut**.

---

## 4. Aspect & JoinPoint

*Definition (Aspect):* An **Aspect** is the class (marked `@Aspect`) that bundles together the cross-cutting logic and the rules for where it applies.

*Definition (JoinPoint):* A **JoinPoint** is a point in your program where an aspect *can* be applied. In Spring AOP this is always a **method execution**. The `JoinPoint` object also gives you info about the intercepted method (its name, arguments, etc.).

```java
@Aspect
@Component                 // must be a bean too
class LoggingAspect {
    @Before("execution(* com.app.service.*.*(..))")
    public void logBefore(JoinPoint jp) {
        System.out.println("Calling: " + jp.getSignature().getName());  // method name
    }
}
```

---

## 5. Advice — the 5 types (WHAT and WHEN)

*Definition:* **Advice** is the code that actually runs, plus the timing of when it runs relative to the target method.

| Advice | Runs… | Use for |
|--------|-------|---------|
| **@Before** | Before the method | Logging, security checks |
| **@AfterReturning** | After the method returns successfully | Log/inspect the result |
| **@AfterThrowing** | When the method throws an exception | Error logging/handling |
| **@After** | After the method, no matter what (like `finally`) | Cleanup |
| **@Around** | Wraps the method — runs **before AND after**, can change result/skip call | Most powerful: timing, caching, transactions |

```java
@Aspect @Component
class TimingAspect {

    @Around("execution(* com.app.service.*.*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();

        Object result = pjp.proceed();   // ← actually run the real method

        long ms = System.currentTimeMillis() - start;
        System.out.println(pjp.getSignature().getName() + " took " + ms + "ms");
        return result;
    }
}
```

> `@Around` uses a **`ProceedingJoinPoint`** and you must call `pjp.proceed()` to run the actual method. If you don't call it, the real method never runs (useful for caching/security blocks).

**Interview one-liner:** Five advice types — @Before, @AfterReturning, @AfterThrowing, @After, and @Around (the most powerful, wraps the method and must call `proceed()`).

---

## 6. Pointcut — WHERE the advice applies

*Definition:* A **Pointcut** is an expression that selects which methods (join points) the advice runs on. It's like a filter: "apply to all methods in the service package", "apply only to methods annotated with @Loggable", etc.

```java
// execution( returnType  package.Class.method(args) )
execution(* com.app.service.*.*(..))   // any method, any class in service package, any args
execution(public * *..*Service.*(..))  // any public method in any *Service class
@annotation(com.app.Loggable)          // any method annotated @Loggable
within(com.app.service..*)             // anything within the service package tree
```

You can name and reuse a pointcut:
```java
@Pointcut("execution(* com.app.service.*.*(..))")
public void serviceMethods() {}

@Before("serviceMethods()")            // reuse it
public void log() { }
```

`execution(* com.app.service.*.*(..))` decoded:
- `*` → any return type
- `com.app.service.*` → any class in that package
- `.*` → any method name
- `(..)` → any number/type of arguments

**Interview one-liner:** A pointcut is an expression that picks which methods advice applies to (e.g. `execution(* com.app.service.*.*(..))`); you can name and reuse it with `@Pointcut`.

---

## 7. Full real example — logging aspect

```java
@Aspect
@Component
public class LoggingAspect {

    @Pointcut("execution(* com.app.service.*.*(..))")
    public void serviceLayer() {}

    @Before("serviceLayer()")
    public void before(JoinPoint jp) {
        System.out.println("➡ Entering: " + jp.getSignature().getName());
    }

    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void afterReturn(JoinPoint jp, Object result) {
        System.out.println("✅ " + jp.getSignature().getName() + " returned " + result);
    }

    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void afterThrow(JoinPoint jp, Exception ex) {
        System.out.println("❌ " + jp.getSignature().getName() + " threw " + ex.getMessage());
    }
}
```
Add the dependency: `spring-boot-starter-aop`. Spring Boot auto-enables AOP.

---

## 8. How does it work under the hood? (Proxies)

*Definition:* Spring applies aspects using **proxies** — instead of giving you the real bean, Spring secretly wraps it in a stand-in object (proxy) that runs the advice, then calls the real method.

```
caller → [Proxy: run @Before → call real method → run @After] → real bean
```

- Uses **JDK dynamic proxy** (if the bean implements an interface) or **CGLIB** (subclass proxy, if no interface).
- **Gotcha:** because it's proxy-based, calling one method of a bean from *another method in the same bean* (a "self-invocation") **bypasses the proxy**, so the advice won't fire. Interviewers love this one.

**Interview one-liner:** Spring AOP works via runtime proxies (JDK dynamic or CGLIB) that wrap the bean; self-invocation within the same bean skips the proxy so advice won't apply.

---

## 9. Where you already use AOP without knowing

- `@Transactional` → transaction management is AOP.
- `@Cacheable` → caching is AOP.
- Spring Security method checks (`@PreAuthorize`) → AOP.

These prove the concept: cross-cutting behavior added without touching your business code.

---

## Quick Revision Cheat Sheet

- **AOP** separates cross-cutting concerns (logging, security, transactions) from business logic.
- **Aspect** = class with the concern (`@Aspect`).
- **Advice** = the action + when: @Before, @AfterReturning, @AfterThrowing, @After, @Around.
- **@Around** is most powerful — wraps method, must call `proceed()`.
- **Pointcut** = expression selecting which methods (e.g. `execution(* com.app.service.*.*(..))`).
- **JoinPoint** = a point where advice runs (in Spring = method execution).
- **Weaving** via runtime **proxies** (JDK dynamic / CGLIB).
- **Self-invocation** within the same bean bypasses the proxy → advice won't fire.
- `@Transactional`, `@Cacheable` are AOP in action.
