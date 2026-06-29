# Spring Boot Notes — Spring MVC Request Lifecycle

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Spring MVC?

*Definition:* **Spring MVC** is Spring's framework for building web applications, based on the **MVC pattern**:
- **Model** → the data.
- **View** → how the data is shown (HTML page, or JSON).
- **Controller** → handles the request and decides what to do.

The whole point of this note: understand the **journey of one HTTP request** from the browser to your code and back.

---

## 2. The star of the show: DispatcherServlet (Front Controller)

*Definition:* The **`DispatcherServlet`** is the single entry point that receives **every** incoming HTTP request and coordinates everything needed to handle it. This design is called the **Front Controller pattern** — one gatekeeper routes all traffic.

Real life: A hotel front desk. Every guest request goes to the front desk first, which then directs it to the right department (housekeeping, kitchen, etc.) and brings the response back.

```
Browser → DispatcherServlet (front desk) → right Controller → response → Browser
```

> Spring Boot **auto-configures** the DispatcherServlet for you. You never create it manually.

**Interview one-liner:** The DispatcherServlet is the front controller — the single entry point that receives every request and delegates it to the right handler.

---

## 3. The full request lifecycle (step by step)

```
   ┌─────────┐  1.request   ┌──────────────────┐
   │ Browser │ ───────────► │ DispatcherServlet│
   └─────────┘              └──────────────────┘
        ▲                      │  2. which handler?
        │                      ▼
        │              ┌──────────────────┐
        │              │  HandlerMapping  │  finds the right controller method
        │              └──────────────────┘
        │                      │  3. invoke via
        │                      ▼
        │              ┌──────────────────┐
        │              │ HandlerAdapter   │ ─► calls your @Controller method
        │              └──────────────────┘        (runs business logic)
        │                      │  4. returns Model + View name
        │                      ▼
        │              ┌──────────────────┐
        │              │ ViewResolver     │  maps view name → actual view (e.g. home.html)
        │              └──────────────────┘
        │                      │  5. render with model data
        │   6. HTTP response   ▼
        └────────────── View renders HTML/JSON
```

### The steps in words
1. **Request arrives** at the `DispatcherServlet`.
2. **HandlerMapping** figures out *which* controller method handles this URL (e.g. `GET /orders/5` → `OrderController.get()`).
3. **HandlerAdapter** actually invokes that controller method. Your code runs, returns a **Model** (data) + a **view name** (or data for JSON).
4. For web pages: the **ViewResolver** turns the view name (`"home"`) into a real view file (`home.html`).
5. The **View** is rendered using the model data → HTML produced.
6. The **response** goes back through the DispatcherServlet to the browser.

**Interview one-liner:** Request → DispatcherServlet → HandlerMapping (find controller) → HandlerAdapter (call it) → controller returns model+view → ViewResolver → View renders → response back.

---

## 4. The key players (cheat table)

| Component | Job |
|-----------|-----|
| **DispatcherServlet** | Front controller; receives all requests, orchestrates the flow |
| **HandlerMapping** | Maps a URL to the correct controller method |
| **HandlerAdapter** | Actually calls the chosen controller method |
| **Controller** | Your code — runs logic, returns model + view (or data) |
| **ViewResolver** | Converts a view name into an actual view file |
| **View** | Renders the final output (HTML, etc.) |

---

## 5. What about REST APIs (JSON)? (no view needed)

When you use `@RestController` (or `@ResponseBody`), steps 4-5 change. There's **no ViewResolver / View**. Instead:

*Definition:* An **`HttpMessageConverter`** takes the object your method returns and converts it directly into the response body — usually **JSON** (via the Jackson library).

```
@RestController flow:
Request → DispatcherServlet → HandlerMapping → HandlerAdapter
       → your method returns an object
       → HttpMessageConverter turns it into JSON
       → response body
```

```java
@RestController
class OrderApi {
    @GetMapping("/orders/{id}")
    Order get(@PathVariable Long id) {
        return new Order(id, 500);   // → Jackson converts to {"id":5,"amount":500}
    }
}
```

> **Web pages** use **ViewResolver + View**. **REST APIs** use **HttpMessageConverter** (object → JSON). This contrast is a common interview question.

**Interview one-liner:** For REST, there's no view — an HttpMessageConverter (Jackson) serializes the returned object straight to JSON in the response body.

---

## 6. Where do filters and interceptors fit?

Two ways to run code *around* requests:

| | Filter | Interceptor (HandlerInterceptor) |
|--|--------|----------------------------------|
| Belongs to | Servlet level (before Spring) | Spring MVC level |
| Runs | Before/after the DispatcherServlet | Around the controller (`preHandle`, `postHandle`, `afterCompletion`) |
| Knows about controller? | No | Yes (has handler info) |
| Use for | Logging, auth, CORS, compression | Auth checks, modifying model, timing per controller |

```
Browser → Filter → DispatcherServlet → Interceptor.preHandle → Controller
        → Interceptor.postHandle → View → Interceptor.afterCompletion → Filter → Browser
```

**Interview one-liner:** Filters run at servlet level (around DispatcherServlet); interceptors run at Spring MVC level around the controller (preHandle/postHandle/afterCompletion).

---

## 7. Minimal controller to tie it together

```java
@Controller
class HomeController {

    @GetMapping("/welcome")
    String welcome(@RequestParam String name, Model model) {
        model.addAttribute("name", name);   // Model = data
        return "welcome";                    // View name → ViewResolver → welcome.html
    }
}
```
URL `GET /welcome?name=Danish` → method runs → model gets `name=Danish` → `welcome.html` rendered with it.

---

## Quick Revision Cheat Sheet

- **Spring MVC** = Model (data) + View (display) + Controller (logic).
- **DispatcherServlet** = front controller, single entry point for all requests (auto-configured).
- **Flow**: Request → DispatcherServlet → **HandlerMapping** (find controller) → **HandlerAdapter** (invoke) → Controller returns model+view → **ViewResolver** → **View** renders → response.
- **REST/JSON**: no view — **HttpMessageConverter** (Jackson) turns the object into JSON.
- **Filters** = servlet-level (around DispatcherServlet); **Interceptors** = MVC-level (around controller).
