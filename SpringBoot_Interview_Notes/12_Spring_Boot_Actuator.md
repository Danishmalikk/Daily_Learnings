# Spring Boot Notes — Spring Boot Actuator (Health, Metrics, Endpoints)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Actuator?

*Definition:* **Spring Boot Actuator** is a built-in feature that exposes ready-made **endpoints** to **monitor and manage** your running application — its health, metrics, configuration, environment, and more. You get production-grade monitoring without writing any code.

Real life: A car dashboard — fuel gauge, engine temperature, speed, warning lights. You don't build these sensors; the car comes with them. Actuator is your app's dashboard.

> The key idea: once your app is **live in production**, you need to *see inside it* (Is it healthy? How much memory? How many requests?). Actuator gives you that visibility out of the box.

**Interview one-liner:** Actuator exposes built-in endpoints to monitor and manage a running Spring Boot app — health, metrics, environment, and more — without custom code.

---

## 2. Setup (one dependency)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

That's it. Endpoints become available under `/actuator`.

---

## 3. The common endpoints

*Definition:* An **Actuator endpoint** is a special built-in URL (under `/actuator`) that reports or controls some aspect of your app.

| Endpoint | Shows | Exposed by default? |
|----------|-------|---------------------|
| `/actuator/health` | App health (UP/DOWN), DB/disk status | ✅ Yes |
| `/actuator/info` | Custom app info (version, name) | ✅ Yes |
| `/actuator/metrics` | Metrics (memory, CPU, requests, GC) | ❌ No (must enable) |
| `/actuator/env` | Environment properties & config | ❌ No |
| `/actuator/beans` | All Spring beans in the context | ❌ No |
| `/actuator/mappings` | All URL → controller mappings | ❌ No |
| `/actuator/loggers` | View & change log levels at runtime | ❌ No |
| `/actuator/threaddump` | Snapshot of all threads | ❌ No |
| `/actuator/heapdump` | Downloads a heap dump file | ❌ No |

> **Security note:** By default only `health` (and `info`) are exposed over HTTP, because the others (`env`, `beans`, `heapdump`) can leak sensitive info. You must explicitly enable the rest.

---

## 4. The /health endpoint (most used)

*Definition:* The **health endpoint** reports whether the app and its dependencies (database, disk space, message brokers) are working. Returns `UP`, `DOWN`, or `OUT_OF_SERVICE`.

```
GET /actuator/health
```
```json
{ "status": "UP" }
```
With details enabled, it breaks down each component:
```json
{
  "status": "UP",
  "components": {
    "db":        { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "ping":      { "status": "UP" }
  }
}
```

> Load balancers and Kubernetes use `/health` for **liveness/readiness probes** — to decide whether to send traffic to your app or restart it. That's the real-world reason it matters.

**Interview one-liner:** `/actuator/health` reports UP/DOWN status of the app and its dependencies; orchestrators like Kubernetes use it for liveness/readiness probes.

---

## 5. Custom health check

You can add your own health logic by implementing `HealthIndicator`:

```java
@Component
class PaymentServiceHealth implements HealthIndicator {
    @Override
    public Health health() {
        boolean ok = checkPaymentProvider();
        return ok ? Health.up().withDetail("provider", "reachable").build()
                  : Health.down().withDetail("provider", "unreachable").build();
    }
}
```
Now `/actuator/health` includes your `paymentServiceHealth` component.

---

## 6. Metrics & Micrometer

*Definition:* The **metrics endpoint** exposes numeric measurements about your app (memory used, CPU, HTTP request counts, response times, GC activity). Spring Boot uses a library called **Micrometer** to collect them.

```
GET /actuator/metrics                       → list of all metric names
GET /actuator/metrics/jvm.memory.used       → a specific metric
GET /actuator/metrics/http.server.requests  → request count & timings
```

> **Micrometer** is like "SLF4J but for metrics" — one API that can ship metrics to many monitoring tools (**Prometheus**, Grafana, Datadog, New Relic). Mention Prometheus + Grafana in interviews — that's the common production stack: Actuator → Micrometer → Prometheus (stores) → Grafana (dashboards).

**Interview one-liner:** `/actuator/metrics` exposes app metrics via Micrometer (a vendor-neutral metrics API), commonly scraped by Prometheus and visualized in Grafana.

---

## 7. Configuration (exposing & securing endpoints)

In `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers   # which to expose over HTTP
        # include: "*"                          # expose ALL (dev only!)
  endpoint:
    health:
      show-details: always                      # show component breakdown
  info:
    env:
      enabled: true
```

> **Securing actuator** is an interview favorite: in production you should (1) expose only what you need, (2) protect endpoints with Spring Security, and/or (3) move them to a **separate port** (`management.server.port`) not reachable publicly. Never expose `heapdump`/`env` openly.

---

## 8. Custom info & custom endpoints

### Custom info
```yaml
info:
  app:
    name: Order Service
    version: 1.2.0
```
→ shows at `/actuator/info`.

### Custom endpoint (advanced, just be aware)
```java
@Component
@Endpoint(id = "release")
class ReleaseEndpoint {
    @ReadOperation
    public Map<String, String> release() {
        return Map.of("release", "2026-Q2", "owner", "Danish");
    }
}
```
→ available at `/actuator/release`.

---

## Quick Revision Cheat Sheet

- **Actuator** = built-in endpoints to **monitor & manage** a live app (the app's dashboard).
- Add `spring-boot-starter-actuator`; endpoints live under `/actuator`.
- **Default exposed**: `health`, `info`. Others (`metrics`, `env`, `beans`, `heapdump`) must be enabled.
- **/health** = UP/DOWN of app + dependencies; used by Kubernetes liveness/readiness probes.
- Custom health via **`HealthIndicator`**.
- **/metrics** via **Micrometer** (vendor-neutral) → commonly **Prometheus + Grafana**.
- Configure exposure with `management.endpoints.web.exposure.include`.
- **Secure** actuator in prod: expose minimally, protect with Spring Security, or use a separate management port.
