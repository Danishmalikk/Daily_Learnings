# Spring Boot Notes — Exception Handling (@ControllerAdvice, @ExceptionHandler)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Why centralized exception handling?

When something goes wrong (user not found, invalid input), you want to send the client a **clean, consistent error response** — not an ugly stack trace or a random 500 error.

Bad approach: wrapping every controller method in `try-catch`. That's repetitive and messy.

```java
// ❌ try-catch in every method = duplication everywhere
@GetMapping("/users/{id}")
User get(@PathVariable Long id) {
    try { ... } catch (Exception e) { ... }   // repeated 100 times
}
```

**The Spring way:** Write error-handling logic **once** in a central place, and Spring routes all exceptions there. Clean controllers, consistent responses.

---

## 2. @ExceptionHandler — handle a specific exception

*Definition:* **`@ExceptionHandler`** marks a method that handles a particular exception type. When that exception is thrown by a controller, Spring calls this method instead of crashing — and you return a proper response.

### Local (inside one controller)
```java
@RestController
class UserController {

    @GetMapping("/users/{id}")
    User get(@PathVariable Long id) {
        return repo.findById(id)
                   .orElseThrow(() -> new UserNotFoundException(id));  // throw
    }

    @ExceptionHandler(UserNotFoundException.class)   // catches it here
    ResponseEntity<String> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404).body(ex.getMessage());
    }
}
```
Problem: this handler only works for **this one controller**. If 10 controllers throw `UserNotFoundException`, you'd repeat it. That's where `@ControllerAdvice` comes in.

**Interview one-liner:** `@ExceptionHandler` is a method that handles a specific exception type and returns a proper response instead of letting the app crash.

---

## 3. @ControllerAdvice — handle exceptions globally (for all controllers)

*Definition:* **`@ControllerAdvice`** is a special class that applies across **all** controllers. Put your `@ExceptionHandler` methods here, and they handle exceptions thrown anywhere in the app — one central place for all error handling.

```java
@RestControllerAdvice            // = @ControllerAdvice + @ResponseBody (returns JSON)
class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    ResponseEntity<ErrorResponse> handleBadInput(IllegalArgumentException ex) {
        return ResponseEntity.badRequest().body(new ErrorResponse(400, ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)            // catch-all fallback
    ResponseEntity<ErrorResponse> handleAll(Exception ex) {
        return ResponseEntity.status(500).body(new ErrorResponse(500, "Something went wrong"));
    }
}
```

> **`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.** Use it for REST APIs so handlers return JSON. Use plain `@ControllerAdvice` if you return error *views* (HTML pages).

**Interview one-liner:** `@ControllerAdvice` centralizes exception handling across all controllers; combined with `@ExceptionHandler` methods it gives one consistent place for error responses. `@RestControllerAdvice` adds `@ResponseBody` for JSON.

---

## 4. A clean error response object

Return a structured error, not a plain string. Clients love consistency.

```java
record ErrorResponse(int status, String message, LocalDateTime timestamp) {
    public ErrorResponse(int status, String message) {
        this(status, message, LocalDateTime.now());
    }
}
```
Client receives:
```json
{ "status": 404, "message": "User 5 not found", "timestamp": "2026-06-29T10:15:00" }
```

---

## 5. Custom exceptions

Define meaningful exceptions for your domain instead of throwing generic ones.

```java
class UserNotFoundException extends RuntimeException {
    UserNotFoundException(Long id) {
        super("User " + id + " not found");
    }
}
```

> Tip: You can also put `@ResponseStatus(HttpStatus.NOT_FOUND)` on the exception class itself, and Spring will return 404 automatically — but a `@ControllerAdvice` gives more control (custom body, logging).

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
class UserNotFoundException extends RuntimeException { ... }
```

---

## 6. Handling validation errors (very common)

With `@Valid`, when input fails validation Spring throws `MethodArgumentNotValidException`. Handle it globally:

```java
// DTO with validation rules
record CreateUser(@NotBlank String name, @Email String email) {}

@PostMapping("/users")
User create(@Valid @RequestBody CreateUser dto) { ... }   // @Valid triggers checks

// In @RestControllerAdvice:
@ExceptionHandler(MethodArgumentNotValidException.class)
ResponseEntity<Map<String,String>> handleValidation(MethodArgumentNotValidException ex) {
    Map<String,String> errors = new HashMap<>();
    ex.getBindingResult().getFieldErrors()
      .forEach(err -> errors.put(err.getField(), err.getDefaultMessage()));
    return ResponseEntity.badRequest().body(errors);
}
```
Client gets: `{ "email": "must be a valid email", "name": "must not be blank" }`.

---

## 7. The hierarchy: how Spring picks the handler

When an exception is thrown, Spring looks for the **most specific** matching `@ExceptionHandler`:
```
UserNotFoundException thrown
  → is there a handler for UserNotFoundException?  ✅ use it
  → else handler for its parent (RuntimeException)?  
  → else handler for Exception (catch-all)?
  → else default Spring error (500 / whitelabel page)
```

> Order of precedence: **local `@ExceptionHandler` in the controller** wins over the **global `@ControllerAdvice`** one for the same exception.

**Interview one-liner:** Spring matches the most specific `@ExceptionHandler` by exception type; local controller handlers take precedence over global `@ControllerAdvice` handlers.

---

## 8. Modern alternative (good to mention): ResponseEntityExceptionHandler & ProblemDetail

- Extending **`ResponseEntityExceptionHandler`** gives you ready-made handling for many built-in Spring MVC exceptions.
- **`ProblemDetail`** (Spring 6 / Boot 3) is a standardized error format (RFC 7807) — worth name-dropping in interviews as the modern way to model errors.

---

## Quick Revision Cheat Sheet

- Avoid `try-catch` in every method — handle errors **centrally**.
- **@ExceptionHandler**: method that handles a specific exception type and returns a proper response.
- **@ControllerAdvice**: global class applying handlers across **all** controllers.
- **@RestControllerAdvice** = `@ControllerAdvice` + `@ResponseBody` (returns JSON) — use for REST APIs.
- Return a **structured error object** (status, message, timestamp) for consistency.
- **Custom exceptions** = clearer domain errors; `@ResponseStatus` can set the HTTP code automatically.
- **Validation**: `@Valid` → handle `MethodArgumentNotValidException` to return field errors.
- Spring picks the **most specific** handler; **local** beats **global**.
- Modern: `ResponseEntityExceptionHandler`, `ProblemDetail` (RFC 7807).
