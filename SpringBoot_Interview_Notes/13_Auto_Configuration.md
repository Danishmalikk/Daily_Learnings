# Spring Boot Notes — Auto Configuration (how it works internally)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What problem does Auto Configuration solve?

In old Spring, you had to **manually configure everything** — DataSource, EntityManager, DispatcherServlet, view resolvers — pages of XML/Java config before your app even ran.

Spring Boot's promise: **"It just works."** You add a dependency, and Boot **automatically configures** sensible defaults for you.

*Definition:* **Auto Configuration** is Spring Boot's feature that automatically sets up beans and configuration **based on what's on your classpath** (the libraries you added). Add a library → Boot guesses what you need and wires it up.

Real life: A smart home that detects the devices you plug in and configures them automatically — plug in a smart bulb, it just connects. You only step in to change defaults.

**Interview one-liner:** Auto Configuration automatically creates and configures beans based on the libraries present on the classpath, so you get working defaults with zero manual setup.

---

## 2. The trigger: @SpringBootApplication

Every Boot app starts here:

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

*Definition:* **`@SpringBootApplication`** is a convenience annotation that bundles three annotations:

```
@SpringBootApplication =
    @SpringBootConfiguration   (this class holds config / is a @Configuration)
  + @EnableAutoConfiguration   (TURN ON auto configuration)   ← the key one
  + @ComponentScan             (scan this package for beans)
```

The one that powers magic = **`@EnableAutoConfiguration`**.

**Interview one-liner:** `@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`; the `@EnableAutoConfiguration` part triggers auto-configuration.

---

## 3. How it works internally (step by step)

```
1. @EnableAutoConfiguration kicks in at startup
2. Boot reads a list of auto-configuration classes from the libraries
       (file: META-INF/spring/...AutoConfiguration.imports inside each jar)
3. For EACH auto-config class, it checks @Conditional rules:
       "Should I apply this? Only IF certain conditions are true."
4. Conditions met → beans created.  Not met → skipped.
5. Your app ends up with exactly the beans your classpath needs.
```

### Where the list lives
- **Spring Boot 2.7+ / 3.x:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- **Older (≤2.6):** `META-INF/spring.factories`

> Each starter jar (e.g. `spring-boot-starter-web`) ships these auto-config classes. Boot loads them and conditionally applies each.

**Interview one-liner:** At startup, `@EnableAutoConfiguration` loads a list of auto-configuration classes from each jar's `AutoConfiguration.imports` file, then applies each one only if its `@Conditional` checks pass.

---

## 4. The secret sauce: @Conditional annotations

*Definition:* **`@Conditional`** annotations make a bean/config apply **only if** certain conditions are true. This is *how* Boot decides what to configure — "only configure a DataSource IF a database driver is on the classpath."

| Conditional | Applies the config only if… |
|-------------|------------------------------|
| `@ConditionalOnClass` | A certain class is on the classpath (library present) |
| `@ConditionalOnMissingBean` | You haven't already defined that bean yourself |
| `@ConditionalOnProperty` | A property is set (e.g. `feature.enabled=true`) |
| `@ConditionalOnBean` | Another specific bean exists |
| `@ConditionalOnWebApplication` | It's a web app |

### Example (simplified from Boot's own code)
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)            // only if JDBC is on classpath
class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                     // only if YOU didn't define one
    DataSource dataSource() {
        return new HikariDataSource(/* defaults */);
    }
}
```
Reading it: "If a `DataSource` class exists on the classpath **and** the user hasn't created their own DataSource bean, then create a default one."

> **`@ConditionalOnMissingBean` is the politeness rule:** Boot only provides a default if *you* didn't. The moment you define your own bean, Boot backs off. This is why your custom config always wins.

**Interview one-liner:** Auto-config classes use `@Conditional` annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`) to apply only when appropriate — and `@ConditionalOnMissingBean` lets your own beans override the defaults.

---

## 5. Starters — the dependency bundles

*Definition:* A **Starter** is a ready-made bundle of dependencies for a feature, so you add **one** dependency instead of hunting for many compatible versions.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>   <!-- brings Spring MVC, Tomcat, Jackson... -->
</dependency>
```

| Starter | Brings in |
|---------|-----------|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson (JSON) |
| `spring-boot-starter-data-jpa` | Spring Data JPA, Hibernate |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit, Mockito, Spring Test |

> Starters + Auto Configuration work together: the starter **puts libraries on the classpath**, and auto-config **detects them and wires beans**. That's the full "just works" magic.

**Interview one-liner:** Starters are curated dependency bundles; they put libraries on the classpath, and auto-configuration detects those libraries and configures the matching beans.

---

## 6. How to see what got auto-configured (debugging)

Run with debug enabled and Boot prints a **Conditions Evaluation Report**:
```properties
# application.properties
debug=true
```
It lists:
- **Positive matches** → auto-configs that were applied (and why).
- **Negative matches** → auto-configs that were skipped (and which condition failed).

Great interview answer to "how do you know what Boot configured?" → enable `debug=true` and read the report.

---

## 7. Overriding / disabling auto-config

```java
// Disable a specific auto-config
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApp { }
```
Or in properties:
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
And remember: defining your **own bean** of the same type usually overrides the default (thanks to `@ConditionalOnMissingBean`).

---

## Quick Revision Cheat Sheet

- **Auto Configuration** = Boot auto-sets up beans based on **classpath** libraries (working defaults, no manual config).
- **@SpringBootApplication** = `@SpringBootConfiguration` + **`@EnableAutoConfiguration`** + `@ComponentScan`.
- **@EnableAutoConfiguration** loads auto-config classes from each jar's `AutoConfiguration.imports` (old: `spring.factories`).
- Each is applied conditionally via **@Conditional** (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`).
- **@ConditionalOnMissingBean** → your own bean overrides Boot's default.
- **Starters** = dependency bundles; they add libraries, auto-config detects & wires them.
- **`debug=true`** → Conditions Evaluation Report (what matched / didn't).
- Disable with `@SpringBootApplication(exclude=...)` or `spring.autoconfigure.exclude`.
