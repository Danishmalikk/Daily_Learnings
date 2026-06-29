# Spring Boot Notes — Testing (@SpringBootTest, @WebMvcTest, @DataJpaTest)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. The testing pyramid (why different test types)

*Definition:* The **testing pyramid** says: write **many small fast tests** (unit), **fewer medium tests** (integration of a slice), and **few big slow tests** (full app). Spring Boot gives you an annotation for each level.

```
        /\        Few   → @SpringBootTest (whole app, slow)
       /  \
      /----\      Some  → slice tests (@WebMvcTest, @DataJpaTest) — one layer
     /      \
    /--------\    Many  → plain unit tests (no Spring, fastest)
```

> Key principle: **load only what you need.** Starting the whole Spring context for every test is slow. "Slice" annotations load just one layer.

**Interview one-liner:** Prefer many fast unit tests, some slice tests for one layer, and few full-context tests — Spring Boot provides `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest` for these levels.

---

## 2. Plain unit test (no Spring at all — fastest)

For pure logic, you don't need Spring. Just JUnit + Mockito.

```java
class OrderServiceTest {
    @Test
    void calculatesTotal() {
        OrderRepository repo = mock(OrderRepository.class);   // fake dependency
        OrderService service = new OrderService(repo);        // plain constructor

        assertEquals(200, service.totalFor(2, 100));
    }
}
```
- `mock(...)` from **Mockito** = a fake object you control.
- No Spring context started → milliseconds fast.

> This is why **constructor injection** matters (see IoC file): you can `new` the class with fakes, no Spring needed.

**Interview one-liner:** For pure logic, write plain JUnit + Mockito tests with no Spring context — fastest and simplest.

---

## 3. @SpringBootTest — full integration test

*Definition:* **`@SpringBootTest`** starts the **entire** Spring application context (all beans, real wiring) for a test. Use it for end-to-end / integration tests where multiple layers work together.

```java
@SpringBootTest
class OrderIntegrationTest {

    @Autowired OrderService orderService;   // real beans, fully wired

    @Test
    void placesOrderEndToEnd() {
        Order o = orderService.place(new Order(100));
        assertNotNull(o.getId());
    }
}
```

To also test HTTP endpoints, add a web environment + a test client:
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiTest {
    @Autowired TestRestTemplate rest;       // or MockMvc / WebTestClient

    @Test
    void getsUser() {
        ResponseEntity<User> res = rest.getForEntity("/users/1", User.class);
        assertEquals(200, res.getStatusCodeValue());
    }
}
```

> **Trade-off:** `@SpringBootTest` is the most realistic but the **slowest** (loads everything). Use it sparingly — for real integration scenarios, not for testing one class.

**Interview one-liner:** `@SpringBootTest` boots the full application context for integration tests — realistic but slow; use it for end-to-end scenarios, not single-class tests.

---

## 4. @WebMvcTest — test only the web layer (controllers)

*Definition:* **`@WebMvcTest`** loads **only the web layer** (controllers, filters, JSON converters) — not services or repositories. It's fast and focused on testing controller behavior, request mapping, and JSON.

```java
@WebMvcTest(UserController.class)            // load ONLY this controller's web slice
class UserControllerTest {

    @Autowired MockMvc mockMvc;              // simulates HTTP without a real server

    @MockBean UserService userService;       // service is mocked, not loaded

    @Test
    void returnsUser() throws Exception {
        when(userService.find(1L)).thenReturn(new User(1L, "Danish"));

        mockMvc.perform(get("/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Danish"));
    }
}
```

- **`MockMvc`** = lets you send fake HTTP requests to controllers without starting a real server.
- **`@MockBean`** = replaces a real bean (the service) with a Mockito mock inside the Spring context.

> Because services/repositories aren't loaded, you **must mock** them with `@MockBean`. That's the whole point — test the controller in isolation.

**Interview one-liner:** `@WebMvcTest` loads only the web layer and uses `MockMvc` to test controllers fast; dependencies like services are provided via `@MockBean`.

---

## 5. @DataJpaTest — test only the persistence layer (repositories)

*Definition:* **`@DataJpaTest`** loads **only the JPA/repository layer** (entities, repositories, an embedded test database). Perfect for testing custom queries and repository methods.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;        // helper to set up test data

    @Test
    void findsByEmail() {
        em.persist(new User("Danish", "d@app.com"));

        User found = repo.findByEmail("d@app.com");

        assertEquals("Danish", found.getName());
    }
}
```

Key behaviors:
- Uses an **embedded in-memory DB** (H2) by default — no real DB needed.
- Each test runs in a **transaction that rolls back** afterward → tests don't pollute each other.
- Only repository/JPA beans load (not your services/controllers) → fast.

**Interview one-liner:** `@DataJpaTest` loads only the JPA layer with an embedded H2 DB and rolls back after each test — ideal for testing repositories and custom queries in isolation.

---

## 6. The comparison table (the money table)

| Annotation | Loads | Speed | Use for |
|------------|-------|-------|---------|
| **(none)** + Mockito | Nothing (plain) | ⚡ Fastest | Pure logic / unit tests |
| **@WebMvcTest** | Web layer only | Fast | Controllers (with `MockMvc` + `@MockBean`) |
| **@DataJpaTest** | JPA layer + H2 | Fast | Repositories & queries |
| **@SpringBootTest** | Whole app context | 🐢 Slowest | Full integration / end-to-end |

---

## 7. Key testing helpers (vocabulary)

| Tool | What it does |
|------|--------------|
| **JUnit 5** | The test framework (`@Test`, `assertEquals`) |
| **Mockito** | Creates fake objects (`mock()`, `when().thenReturn()`) |
| **@Mock / @MockBean** | `@Mock` = plain Mockito; `@MockBean` = mock inside Spring context |
| **MockMvc** | Fake HTTP requests to controllers (no real server) |
| **TestEntityManager** | Set up DB data in `@DataJpaTest` |
| **TestRestTemplate / WebTestClient** | Real HTTP calls in `@SpringBootTest` |
| **AssertJ** | Fluent assertions (`assertThat(x).isEqualTo(y)`) |

> `@Mock` vs `@MockBean`: **`@Mock`** is a plain Mockito mock (no Spring). **`@MockBean`** puts a mock *into the Spring context*, replacing the real bean — used in slice/`@SpringBootTest` tests. Common interview distinction.

All of this comes with one dependency: `spring-boot-starter-test`.

---

## Quick Revision Cheat Sheet

- **Pyramid**: many unit tests, some slice tests, few full-context tests. Load only what you need.
- **Plain unit** (JUnit + Mockito): no Spring, fastest — works because of constructor injection.
- **@SpringBootTest**: full app context; integration/end-to-end; slowest — use sparingly.
- **@WebMvcTest**: only web layer; test controllers with **MockMvc** + **@MockBean**.
- **@DataJpaTest**: only JPA layer + **embedded H2**; rolls back each test; for repositories/queries.
- **MockMvc** = fake HTTP to controllers; **TestEntityManager** = set up test data.
- **@Mock** = plain Mockito; **@MockBean** = mock inside the Spring context.
- Everything via **`spring-boot-starter-test`** (JUnit 5, Mockito, AssertJ, Spring Test).
