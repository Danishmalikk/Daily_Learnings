# Spring Boot Notes — Profiles & Environment Config (@Profile, application.yml)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The problem: one app, many environments

Your app runs in different places with different settings:
- **Local (dev)** → local H2 database, debug logging.
- **Test** → test database, mock services.
- **Production (prod)** → real database, error-only logging, real API keys.

You don't want to change code or rebuild for each. You want **the same app** to pick the right settings per environment. That's what **profiles** and **externalized config** are for.

---

## 2. Externalized configuration (application.properties / application.yml)

*Definition:* **Externalized configuration** means keeping settings (DB url, ports, passwords, flags) **outside your code**, in config files or environment variables — so you can change behavior without recompiling.

Two file formats (pick one style):

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost/mydb
app.feature.dark-mode=true
```
```yaml
# application.yml  (same thing, nested & cleaner)
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost/mydb
app:
  feature:
    dark-mode: true
```

| | .properties | .yml |
|--|-------------|------|
| Style | flat `key=value` | nested/indented |
| Readable for nesting | repetitive | ✅ cleaner |
| Lists/maps | clumsy | ✅ natural |

**Interview one-liner:** Externalized config keeps settings outside code in `application.properties`/`application.yml`; YAML is cleaner for nested structures.

---

## 3. Profiles — different config per environment

*Definition:* A **Profile** is a named set of configuration/beans that's only active in a certain environment (like `dev`, `test`, `prod`). You switch profiles to switch the whole app's settings at once.

### Profile-specific files
Boot automatically loads `application-{profile}.yml` when that profile is active:
```
application.yml             ← common/base settings (always loaded)
application-dev.yml         ← only when 'dev' profile active
application-prod.yml        ← only when 'prod' profile active
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb       # local in-memory DB
logging:
  level:
    root: DEBUG
```
```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-server/appdb
logging:
  level:
    root: ERROR
```

> Base `application.yml` is **always** applied; the active profile's file **adds to / overrides** it.

**Interview one-liner:** A profile is an environment-specific config set; Boot auto-loads `application-{profile}.yml` on top of the base `application.yml` when that profile is active.

---

## 4. How to activate a profile

Many ways (later ones override earlier ones):

```yaml
# 1. In application.yml (default profile)
spring:
  profiles:
    active: dev
```
```bash
# 2. Command line (overrides the file)
java -jar app.jar --spring.profiles.active=prod

# 3. Environment variable
export SPRING_PROFILES_ACTIVE=prod

# 4. JVM system property
java -Dspring.profiles.active=prod -jar app.jar
```

> Common production pattern: **don't** hardcode `active` in the file — set it via an **environment variable** on the server, so the same jar runs anywhere.

---

## 5. @Profile — beans that exist only in certain profiles

*Definition:* **`@Profile`** on a bean/class makes it load **only** when the named profile is active. Different environments get different implementations.

```java
@Service
@Profile("dev")
class FakeEmailService implements EmailService {
    public void send(String to) { System.out.println("DEV: pretend-sent to " + to); }
}

@Service
@Profile("prod")
class RealEmailService implements EmailService {
    public void send(String to) { /* call real email provider */ }
}
```
- In `dev` → `FakeEmailService` is used.
- In `prod` → `RealEmailService` is used.
- Your other code just depends on `EmailService` and doesn't care which.

```java
@Profile("!prod")   // active in everything EXCEPT prod
@Profile({"dev", "test"})   // active in dev OR test
```

**Interview one-liner:** `@Profile("dev")` makes a bean load only under that profile, so you can swap implementations (e.g. fake vs real email) per environment.

---

## 6. Reading config values into code

### @Value — single property
*Definition:* **`@Value`** injects one property value into a field.
```java
@Service
class AppService {
    @Value("${server.port}")               private int port;
    @Value("${app.feature.dark-mode:false}") private boolean darkMode;  // :false = default
}
```

### @ConfigurationProperties — group of properties (preferred)
*Definition:* **`@ConfigurationProperties`** binds a whole **group** of related properties to a typed object — cleaner than many `@Value`s.

```yaml
app:
  mail:
    host: smtp.gmail.com
    port: 587
    from: noreply@app.com
```
```java
@Component
@ConfigurationProperties(prefix = "app.mail")
class MailConfig {
    private String host;
    private int port;
    private String from;
    // getters & setters → Spring binds them automatically
}
```

| | @Value | @ConfigurationProperties |
|--|--------|--------------------------|
| Best for | One or two values | A group of related properties |
| Type-safe | Less | ✅ Yes (typed object) |
| Validation | No | ✅ Supports `@Validated` |
| Readability | Clutters with many | ✅ Clean |

**Easy rule of thumb:** A few scattered values → `@Value`. A related group (mail, datasource, feature settings) → `@ConfigurationProperties`.

**Interview one-liner:** `@Value` injects a single property; `@ConfigurationProperties` binds a group of properties to a typed object (cleaner, type-safe, validatable).

---

## 7. Configuration precedence (who wins)

Boot reads config from many sources; later ones **override** earlier. Simplified order (highest priority first):

```
1. Command-line args (--key=value)
2. Environment variables / system properties
3. application-{profile}.yml (active profile)
4. application.yml (base)
5. Default values in code
```

> This is *why* you can ship one jar and override anything per environment from the outside — without rebuilding. Common interview point.

---

## Quick Revision Cheat Sheet

- **Externalized config** = settings outside code (`application.properties`/`.yml`); YAML cleaner for nesting.
- **Profile** = environment-specific config set (`dev`, `test`, `prod`).
- Boot loads **`application-{profile}.yml`** on top of base `application.yml` when active.
- **Activate** via `spring.profiles.active` — file, CLI `--spring.profiles.active`, or env var `SPRING_PROFILES_ACTIVE`.
- **@Profile("dev")** = bean loads only in that profile (swap fake/real implementations).
- **@Value("${key}")** = one property (supports `:default`).
- **@ConfigurationProperties(prefix=...)** = bind a group to a typed object (preferred, type-safe).
- **Precedence**: CLI args > env vars > profile file > base file > code defaults.
