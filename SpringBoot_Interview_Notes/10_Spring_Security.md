# Spring Boot Notes — Spring Security (JWT, OAuth2, Filter Chain)

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. What is Spring Security?

*Definition:* **Spring Security** is a framework that protects your app by handling **authentication** (who are you?) and **authorization** (what are you allowed to do?). It guards your endpoints so only the right users reach the right resources.

Two words to never mix up:
- **Authentication (AuthN)** → *verifying identity.* "Are you really Danish?" (login).
- **Authorization (AuthZ)** → *verifying permission.* "Is Danish allowed to delete users?" (roles/access).

Real life: At an airport — showing your passport = **authentication**; your boarding pass letting you into business class = **authorization**.

**Interview one-liner:** Authentication = proving who you are; Authorization = checking what you're allowed to do. Spring Security handles both.

---

## 2. The Security Filter Chain (the heart of it)

*Definition:* The **Security Filter Chain** is a series of **filters** every request passes through *before* reaching your controller. Each filter does one security job (check credentials, validate token, check permissions). If any filter rejects the request, it's stopped right there.

```
Request
  → [Filter 1: CORS]
  → [Filter 2: CSRF]
  → [Filter 3: Authentication — who are you?]
  → [Filter 4: Authorization — are you allowed?]
  → ... more filters ...
  → Controller   (only if all filters pass)
```

> Spring Security is built **entirely on servlet filters**. Understanding "it's a chain of filters" is the #1 concept interviewers check.

The whole chain is wired up by a special filter called **`FilterChainProxy`** (registered via `DelegatingFilterProxy`). You usually don't touch these names — just know the request flows through a chain.

**Interview one-liner:** Spring Security works as a chain of servlet filters that every request passes through before hitting the controller; each filter handles one security concern, and any can block the request.

---

## 3. Core building blocks (the vocabulary)

| Component | Job |
|-----------|-----|
| **SecurityFilterChain** | The configured list of filters + rules (which URLs need auth) |
| **AuthenticationManager** | Coordinates the authentication process |
| **UserDetailsService** | Loads user data (username, password, roles) — usually from DB |
| **PasswordEncoder** | Hashes & verifies passwords (e.g. BCrypt) — never store plain text |
| **SecurityContext** | Holds the currently logged-in user for the request |
| **GrantedAuthority / Role** | A permission the user has (e.g. `ROLE_ADMIN`) |

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();   // hash passwords — never store plain text
}
```

---

## 4. Basic security configuration (modern style)

Modern Spring Security (6.x) uses a `SecurityFilterChain` bean with a lambda DSL:

```java
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
          .csrf(csrf -> csrf.disable())                 // off for stateless APIs
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/public/**").permitAll()    // open
              .requestMatchers("/admin/**").hasRole("ADMIN")// admins only
              .anyRequest().authenticated())                // everything else: must log in
          .httpBasic(Customizer.withDefaults());        // or formLogin / JWT
        return http.build();
    }
}
```

- `permitAll()` → open to everyone.
- `authenticated()` → must be logged in.
- `hasRole("ADMIN")` / `hasAuthority(...)` → must have that role/permission.

**Interview one-liner:** You configure security with a `SecurityFilterChain` bean — define which URLs are public, which need authentication, and which need specific roles.

---

## 5. JWT (JSON Web Token) — stateless authentication

*Definition:* A **JWT** is a compact, signed token the server gives a user after login. The client sends it back on every request (in the `Authorization` header). The server verifies the signature and trusts it — **no session is stored on the server** (stateless).

### Structure: `header.payload.signature`
```
eyJhbGci...  .  eyJzdWIi...  .  SflKxwRJ...
  header          payload         signature
 (algorithm)   (user data,      (verifies it
               claims, expiry)   wasn't tampered)
```
- **Header** → token type + signing algorithm.
- **Payload** → "claims" (user id, roles, expiry). *Not encrypted — just encoded. Never put secrets here.*
- **Signature** → server-created hash proving the token is genuine and unmodified.

### Flow
```
1. POST /login (username + password)
2. Server verifies → creates JWT → sends it to client
3. Client stores JWT, sends it on every request:
      Authorization: Bearer <token>
4. A JWT filter validates the token's signature + expiry
5. Valid → request proceeds; invalid → 401 Unauthorized
```

> **Why JWT?** It's **stateless** — the server doesn't store sessions, so it scales easily across many servers (any server can verify the token). Downside: you can't easily "log out"/revoke a token before it expires (you need a blacklist or short expiry).

**Interview one-liner:** A JWT is a signed token (header.payload.signature) sent in the `Authorization: Bearer` header; the server verifies its signature instead of storing a session — making auth stateless and scalable.

---

## 6. Session-based vs JWT (token-based)

| | Session-based | JWT (token-based) |
|--|---------------|-------------------|
| Where state lives | On the **server** (session store) | On the **client** (token) |
| Server stores sessions? | ✅ Yes | ❌ No (stateless) |
| Scales across servers | Harder (need shared session) | Easy (any server verifies) |
| Logout/revoke | Easy (delete session) | Harder (token valid until expiry) |
| Best for | Traditional web apps | REST APIs, microservices, mobile |

**Interview one-liner:** Sessions store state on the server; JWT stores it in a signed token on the client (stateless) — JWT scales better but is harder to revoke.

---

## 7. OAuth2 — "login with Google/GitHub" (delegated access)

*Definition:* **OAuth2** is a standard that lets a user grant your app **limited access** to their account on another service (Google, GitHub) **without sharing their password**. Your app gets a token from the provider instead.

Real life: A hotel key card. The front desk (Google) gives you a card (token) that opens *only your room* (limited access) — you never get the hotel's master key (the password).

### Key roles
| Role | Who | Example |
|------|-----|---------|
| **Resource Owner** | The user | You |
| **Client** | The app wanting access | Your Spring app |
| **Authorization Server** | Issues tokens | Google's login server |
| **Resource Server** | Holds the protected data | Google's user-info API |

### Simplified flow ("Login with Google")
```
1. User clicks "Login with Google"
2. Redirected to Google → user logs in & approves
3. Google sends back an authorization code
4. Your app exchanges the code for an access token
5. App uses the token to fetch user info → user is logged in
```

Spring makes this easy with `spring-boot-starter-oauth2-client`:
```java
http.oauth2Login(Customizer.withDefaults());   // enables OAuth2 login
```

> **OAuth2 vs JWT:** they're not competitors. OAuth2 is a *framework for delegated authorization*; JWT is a *token format*. OAuth2 often *uses* JWTs as its access tokens.

**Interview one-liner:** OAuth2 lets users grant your app limited access to another service without sharing passwords; your app receives an access token. It's a delegated-authorization standard, and it often uses JWTs as tokens.

---

## 8. Common annotations for method-level security

```java
@EnableMethodSecurity                 // turn it on
class Config {}

@PreAuthorize("hasRole('ADMIN')")     // check BEFORE method runs
void deleteUser(Long id) { ... }

@PreAuthorize("#userId == authentication.principal.id")   // only own data
User getProfile(Long userId) { ... }
```
- `@PreAuthorize` → check permission *before* the method (most common).
- `@PostAuthorize` → check *after* (e.g. based on the returned object).
- These are powered by **AOP** (see the AOP file).

---

## Quick Revision Cheat Sheet

- **Authentication** = who you are (login); **Authorization** = what you can do (roles).
- Spring Security = a **chain of servlet filters** every request passes before the controller.
- Configure via a **`SecurityFilterChain`** bean: `permitAll`, `authenticated`, `hasRole`.
- **PasswordEncoder** (BCrypt) — always hash passwords, never store plain text.
- **JWT** = signed token `header.payload.signature`, sent as `Authorization: Bearer`; **stateless** (no server session) → scalable, but hard to revoke.
- **Session vs JWT**: server-state vs client-token (stateless).
- **OAuth2** = delegated access to another service (Google/GitHub) without sharing passwords; often uses JWTs.
- **@PreAuthorize/@PostAuthorize** = method-level security (via AOP).
