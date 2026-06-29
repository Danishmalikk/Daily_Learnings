# Spring Boot Notes — @RestController vs @Controller

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The one-line difference

*Definition:*
- **`@Controller`** is for traditional web apps that return **views** (HTML pages). What you return is treated as a **view name**.
- **`@RestController`** is for **REST APIs** that return **data** (usually JSON). What you return is sent **directly as the response body**.

The formula to memorize:
```
@RestController = @Controller + @ResponseBody
```

Real life:
- `@Controller` = a waiter who brings you a **plated dish** (a full rendered HTML page).
- `@RestController` = a takeaway counter that hands you **raw ingredients in a box** (raw JSON data) for the client to use.

---

## 2. @Controller — returns a view (HTML)

With `@Controller`, the String you return is a **view name**. The ViewResolver finds the matching HTML template and renders it.

```java
@Controller
class PageController {

    @GetMapping("/home")
    String home(Model model) {
        model.addAttribute("user", "Danish");
        return "home";      // → looks for home.html template, renders it
    }
}
```
Response sent to browser = a full **HTML page**.

> If you want a `@Controller` method to return *data* instead of a view, you add `@ResponseBody` on that method:
```java
@Controller
class MixController {
    @GetMapping("/api/user")
    @ResponseBody                      // now returns data, not a view
    User user() { return new User("Danish"); }   // → JSON
}
```

**Interview one-liner:** `@Controller` returns view names that get rendered into HTML; add `@ResponseBody` to a method if you want it to return data instead.

---

## 3. @RestController — returns data (JSON)

With `@RestController`, every method automatically has `@ResponseBody`. The returned object is **serialized to JSON** (by Jackson) and put in the response body. No views involved.

```java
@RestController
class UserApi {

    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id) {
        return new User(id, "Danish");   // → {"id":1,"name":"Danish"}
    }
}
```
Response sent to client = **JSON data**.

**Interview one-liner:** `@RestController` = `@Controller` + `@ResponseBody`; every method returns data (JSON) directly in the body — built for REST APIs.

---

## 4. @ResponseBody — the key that explains both

*Definition:* **`@ResponseBody`** tells Spring: "Don't treat my return value as a view name — convert it directly into the response body." That conversion (object → JSON) is done by an **HttpMessageConverter** (Jackson for JSON).

```
With @ResponseBody:     return user  →  Jackson  →  {"name":"Danish"}  (body)
Without it (@Controller): return "user" →  ViewResolver  →  user.html  (page)
```

So:
- `@Controller` alone → view name.
- `@Controller` + `@ResponseBody` on a method → data.
- `@RestController` → `@ResponseBody` already applied to **all** methods.

---

## 5. Side-by-side comparison

| Feature | @Controller | @RestController |
|---------|-------------|-----------------|
| Returns | View name (HTML) | Data (JSON/XML) |
| `@ResponseBody` needed? | Yes, per method (if returning data) | No — built in for all methods |
| Used for | Server-rendered web pages (Thymeleaf, JSP) | REST APIs |
| Needs ViewResolver? | Yes | No |
| Typical return | `"home"` → home.html | `new User(...)` → JSON |

---

## 6. How does the object become JSON?

You return a Java object; the client gets JSON. Behind the scenes:
1. Method returns `new User(1, "Danish")`.
2. Spring picks an **HttpMessageConverter** (`MappingJackson2HttpMessageConverter`).
3. Jackson serializes the object → `{"id":1,"name":"Danish"}`.
4. That JSON goes into the HTTP response body.

> Jackson is included automatically with `spring-boot-starter-web`. That's why JSON "just works" with no setup.

---

## 7. Which should a beginner use?

- Building a **REST API / backend for a frontend (React, Angular, mobile)** → **`@RestController`** (almost always your choice today).
- Building a **server-rendered website** (HTML pages via Thymeleaf) → **`@Controller`**.

**Easy rule of thumb:** Returning JSON to a client app → `@RestController`. Returning HTML pages → `@Controller`.

---

## Quick Revision Cheat Sheet

- **@RestController = @Controller + @ResponseBody.**
- **@Controller**: returns **view names** → rendered as HTML (needs ViewResolver).
- **@RestController**: returns **data** → serialized to JSON (every method has `@ResponseBody`).
- **@ResponseBody**: "convert my return value to the response body" via HttpMessageConverter (Jackson).
- Add `@ResponseBody` to a `@Controller` method to return data instead of a view.
- Use **@RestController** for REST APIs, **@Controller** for server-rendered web pages.
