# Spring Boot Notes — @RequestMapping, @GetMapping, @PostMapping

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is mapping?

*Definition:* **Mapping** = connecting a **URL + HTTP method** to a specific Java method in your controller. So when a request like `GET /users/5` comes in, Spring knows to call *your* `getUser()` method.

Real life: A phone switchboard. A call to extension "users" gets routed to the users department. The mapping annotations are the switchboard wiring.

```
GET  /users      → getAllUsers()
POST /users      → createUser()
GET  /users/5    → getUser(5)
```

---

## 2. @RequestMapping — the original, general-purpose one

*Definition:* **`@RequestMapping`** maps a URL to a method and can specify the HTTP method, headers, params, and content types. It's the **parent** of all the shortcut annotations.

```java
@RestController
@RequestMapping("/users")            // base path for the whole class
class UserController {

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    User getUser(@PathVariable Long id) { ... }
}
```

- On a **class** → sets a base path for all methods (`/users`).
- On a **method** → maps a specific path + HTTP method.
- Without `method`, it matches **all** HTTP methods (usually you don't want that).

**Interview one-liner:** `@RequestMapping` is the general mapping annotation (URL + HTTP method + more); it's the parent of the `@GetMapping`/`@PostMapping` shortcuts.

---

## 3. The shortcut annotations (cleaner, preferred)

Since Spring 4.3, each HTTP method has its own shortcut. They're just `@RequestMapping` pre-set to one HTTP method.

| Shortcut | Equivalent to | HTTP method | Used for |
|----------|---------------|-------------|----------|
| **@GetMapping** | `@RequestMapping(method=GET)` | GET | Read/fetch data |
| **@PostMapping** | `@RequestMapping(method=POST)` | POST | Create new data |
| **@PutMapping** | `@RequestMapping(method=PUT)` | PUT | Update/replace data |
| **@DeleteMapping** | `@RequestMapping(method=DELETE)` | DELETE | Delete data |
| **@PatchMapping** | `@RequestMapping(method=PATCH)` | PATCH | Partial update |

```java
@RestController
@RequestMapping("/users")
class UserController {

    @GetMapping            User[] all()                  { ... }   // GET /users
    @GetMapping("/{id}")   User one(@PathVariable Long id){ ... }  // GET /users/5
    @PostMapping           User create(@RequestBody User u){ ... } // POST /users
    @PutMapping("/{id}")   User update(@PathVariable Long id, @RequestBody User u){ ... }
    @DeleteMapping("/{id}") void delete(@PathVariable Long id){ ... }
}
```

**Easy rule of thumb:** Use the **specific shortcut** (`@GetMapping`, `@PostMapping`, …). It's shorter and shows intent. Use `@RequestMapping` mainly for the **class-level base path**.

**Interview one-liner:** `@GetMapping`/`@PostMapping`/etc. are shortcuts for `@RequestMapping` with a fixed HTTP method — cleaner and preferred over the general form.

---

## 4. Reading data from the request — the 3 key annotations

When a request arrives, you often need data *from* it. Three main sources:

### @PathVariable — value from the URL path
*Definition:* **`@PathVariable`** pulls a value out of the **URL path** itself.
```java
@GetMapping("/users/{id}")
User get(@PathVariable Long id) { ... }   // GET /users/5 → id = 5
```

### @RequestParam — value from query string
*Definition:* **`@RequestParam`** reads a **query parameter** (the part after `?`).
```java
@GetMapping("/users")
List<User> search(@RequestParam String name,
                  @RequestParam(defaultValue = "0") int page) { ... }
// GET /users?name=Danish&page=2  → name="Danish", page=2
```

### @RequestBody — value from the request body
*Definition:* **`@RequestBody`** takes the **JSON body** of the request and converts it into a Java object (via Jackson).
```java
@PostMapping("/users")
User create(@RequestBody User user) { ... }
// body {"name":"Danish"} → User object with name="Danish"
```

| Annotation | Reads from | Example source |
|------------|-----------|----------------|
| **@PathVariable** | URL path | `/users/**5**` |
| **@RequestParam** | Query string | `/users?name=**Danish**` |
| **@RequestBody** | Request body | JSON `{"name":"Danish"}` |

**Interview one-liner:** `@PathVariable` reads from the URL path, `@RequestParam` from the query string (`?key=value`), and `@RequestBody` converts the JSON body into a Java object.

---

## 5. Putting it together — a full REST controller

```java
@RestController
@RequestMapping("/api/users")
class UserController {

    @GetMapping                          // GET /api/users?role=admin
    List<User> list(@RequestParam(required = false) String role) { ... }

    @GetMapping("/{id}")                 // GET /api/users/5
    User get(@PathVariable Long id) { ... }

    @PostMapping                         // POST /api/users  (JSON body)
    User create(@RequestBody User user) { ... }

    @PutMapping("/{id}")                 // PUT /api/users/5  (JSON body)
    User update(@PathVariable Long id, @RequestBody User user) { ... }

    @DeleteMapping("/{id}")              // DELETE /api/users/5
    void delete(@PathVariable Long id) { ... }
}
```

This is the standard **CRUD** REST pattern (Create=POST, Read=GET, Update=PUT, Delete=DELETE).

---

## 6. Handy extras (good to know)

- **Narrow by params/headers:**
```java
@GetMapping(value = "/users", params = "type=vip")     // only if ?type=vip
@PostMapping(value = "/users", consumes = "application/json", produces = "application/json")
```
- `consumes` → what content type the method **accepts** (request body).
- `produces` → what content type it **returns** (response).
- **Rename a path variable:** `@PathVariable("id") Long userId` if names differ.
- **Optional param:** `@RequestParam(required = false)` or give a `defaultValue`.

---

## Quick Revision Cheat Sheet

- **Mapping** = connect a URL + HTTP method to a controller method.
- **@RequestMapping**: general-purpose parent; great for the **class-level base path**.
- **Shortcuts**: `@GetMapping` (read), `@PostMapping` (create), `@PutMapping` (update), `@DeleteMapping` (delete), `@PatchMapping` (partial) — preferred over `@RequestMapping`.
- **@PathVariable** → from URL path (`/users/5`).
- **@RequestParam** → from query string (`?name=Danish`); supports `defaultValue`, `required`.
- **@RequestBody** → JSON body → Java object (via Jackson).
- Standard CRUD: POST=create, GET=read, PUT=update, DELETE=delete.
- `consumes` = accepted request type; `produces` = returned response type.
