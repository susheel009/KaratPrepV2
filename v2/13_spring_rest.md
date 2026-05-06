# 13 — Spring REST & Web

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/13_spring_rest_doubts.md) · [← Prev: 12 Spring Boot](./12_spring_boot.md) · [Next: 14 Spring AOP & Data →](./14_spring_aop_data.md)
>
> **Priority:** 🟡 High · **Related topics:** [12 Spring Boot](./12_spring_boot.md) · [04 Exceptions](./04_exceptions.md) · [10 Testing](./10_testing.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — turning a method call into an HTTP API
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — request lifecycle
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — controller + JSON in 20 lines
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 REST principles](#41-rest-principles--idempotency)
   - [4.2 The dispatch pipeline](#42-the-dispatch-pipeline)
   - [4.3 Controller patterns](#43-controller-patterns--paths-params-body)
   - [4.4 Validation](#44-validation--bean-validation--valid)
   - [4.5 Global exception handling](#45-global-exception-handling)
   - [4.6 ResponseEntity](#46-responseentity--full-http-control)
   - [4.7 Jackson](#47-jackson--customising-json)
   - [4.8 HTTP clients](#48-http-clients-resttemplate-webclient-restclient)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 CORS](#51-cors--what-it-is-when-needed)
   - [5.2 Spring Security basics](#52-spring-security-basics)
   - [5.3 API versioning](#53-api-versioning-strategies)
   - [5.4 Pagination & sorting](#54-pagination-and-sorting)
   - [5.5 Idempotency keys](#55-idempotency-keys-for-payment-style-posts)
   - [5.6 HATEOAS / OpenAPI](#56-hateoas-and-openapi)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

You've written a Java method `getUser(123)` that returns a `User` object. It works on your machine. Now your iPhone app and your React frontend, neither of which can call Java methods directly, both want to call it.

Without a framework, you'd have to: spin up an HTTP server, parse the URL, route `/users/123` to your method, extract `123` as a number, marshal `User` into JSON, set the right `Content-Type`, handle `?` query params, deal with bad input (400), missing IDs (404), errors (500), CORS preflight requests, authentication, rate limiting…

A REST API is the standard way to expose a Java method to the network. Spring's web stack does the boring 90 % so you can focus on the method body:

- **`@RestController` + `@GetMapping`** — declare which method handles which URL.
- **`@PathVariable`, `@RequestParam`, `@RequestBody`** — declare how to extract data from the request.
- **Jackson** — automatic Java ↔ JSON conversion.
- **`@RestControllerAdvice`** — central place for converting exceptions to HTTP responses.
- **Bean Validation (`@Valid`)** — declarative input validation.
- **`MockMvc` / WebTestClient** — test the endpoints without a real server.

> Once you see "REST controller = a method with HTTP wiring annotations", every Spring Web feature becomes a refinement of one of those wiring concerns: how URLs map to methods, how to read input, how to write output, how to handle errors.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

Trace what happens when a browser sends `GET /api/users/42` to your Spring Boot app:

| Step | Component | What |
|---|---|---|
| 1 | Embedded Tomcat | Receives the TCP request, parses HTTP headers, hands the request to Spring's `DispatcherServlet`. |
| 2 | `DispatcherServlet` | The single front controller. Runs all the registered filters first (Spring Security, CORS, logging). |
| 3 | `HandlerMapping` | Finds the controller method matching `GET /api/users/{id}`. |
| 4 | `HandlerInterceptor.preHandle` | Any registered interceptors run here (auth checks, MDC setup). |
| 5 | Controller method | Spring extracts `42` from the path → calls `getUser(42)`. The method's return value (a `User`) is the result. |
| 6 | Jackson | Serialises the `User` object to JSON: `{"id":42,"name":"Alice"}`. |
| 7 | `HandlerInterceptor.postHandle` / `afterCompletion` | Logging, metrics. |
| 8 | DispatcherServlet | Writes the JSON to the response, closes the request. |

Two takeaways:

1. **One front controller (`DispatcherServlet`) routes everything.** Filters and interceptors hook into it; URL patterns dispatch to your `@*Mapping` methods.
2. **JSON ↔ Java is automatic.** `@RequestBody` deserialises incoming JSON into a method parameter; the return value is serialised back. Jackson is doing the work; Spring just configures it.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
@RestController                                       // = @Controller + @ResponseBody on every method
@RequestMapping("/api/users")                         // base path for every method below
public class UserController {

    private final UserService service;
    public UserController(UserService service) { this.service = service; }   // constructor injection

    @GetMapping("/{id}")                              // GET /api/users/{id}
    public User getUser(@PathVariable Long id) {       // {id} → method parameter
        return service.findById(id)                    // Optional<User>
            .orElseThrow(() -> new ResourceNotFoundException("user " + id));   // → 404 via advice
    }

    @PostMapping                                      // POST /api/users
    public ResponseEntity<User> create(@Valid @RequestBody CreateUserRequest req) {  // validate input
        var created = service.create(req);
        var location = URI.create("/api/users/" + created.id());
        return ResponseEntity.created(location).body(created);                       // 201 + Location header
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)             // 204 — no body
    public void delete(@PathVariable Long id) { service.delete(id); }
}

public record CreateUserRequest(
    @NotBlank String name,
    @Email   String email
) {}
```

What just happened: three endpoints, full input validation, explicit 201 with `Location` header, automatic JSON serialisation in and out. About 25 lines of actual code. The runtime work — TCP, HTTP parsing, routing, JSON, validation — is entirely in Spring.

---

## 4. Build Up — Practical Patterns

### 4.1 REST principles + idempotency

REST is an architectural style with a few constraints:

- **Resource-oriented URLs.** `/users/42`, not `/getUser?id=42`.
- **Standard HTTP verbs.** `GET` to read, `POST` to create, `PUT` to replace, `PATCH` to update, `DELETE` to delete.
- **Stateless.** Each request carries everything needed (auth tokens, headers); no server-side session affinity required.
- **Standard status codes.** `200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `401 Unauthorized`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `500 Internal Server Error`, `503 Service Unavailable`.

| Method | Idempotent? | Safe? | Body? | Standard status |
|---|:-:|:-:|:-:|:-:|
| GET | ✅ | ✅ | No | 200 / 404 |
| POST | ❌ | ❌ | Yes | 201 |
| PUT | ✅ | ❌ | Yes | 200 / 204 |
| PATCH | ❌ | ❌ | Yes | 200 / 204 |
| DELETE | ✅ | ❌ | No | 204 |

> 💡 **Idempotency** is critical in payments and finance. If a `POST /payments` is retried after a network timeout, two payments could be created. Either use `PUT` with a client-generated ID, or accept an `Idempotency-Key` header (see [§5.5](#55-idempotency-keys-for-payment-style-posts)).

### 4.2 The dispatch pipeline

```
HTTP request
   │
   ▼
[ Servlet container (Tomcat) ]
   │
   ▼
[ Filter chain ]                     (Security, CORS, request-logging, auth)
   │
   ▼
DispatcherServlet
   │
   ▼
HandlerMapping       finds handler method matching URL + method
   │
   ▼
HandlerInterceptor.preHandle()       (cross-cutting, may short-circuit)
   │
   ▼
Controller method                    @RequestBody → Jackson deserialise; @Valid → validate
   │  return value
   ▼
HandlerInterceptor.postHandle()
   │
   ▼
HttpMessageConverter                 Jackson serialises Java → JSON
   │
   ▼
HTTP response
```

Filters run *outside* Spring (servlet level), interceptors run *inside* (Spring MVC level). Use a filter for cross-cutting concerns that don't need the matched controller (request logging, CORS); use an interceptor when you need controller info (per-method auth, MDC enrichment based on path).

### 4.3 Controller patterns — paths, params, body

```java
@RestController
@RequestMapping("/api/trades")
public class TradeController {

    // GET /api/trades?symbol=AAPL&page=0&size=20
    @GetMapping
    public Page<TradeDTO> list(
            @RequestParam(required = false) String symbol,        // query param, optional
            @PageableDefault(size = 20) Pageable pageable) {        // pagination via Spring Data
        return service.find(symbol, pageable);
    }

    // GET /api/trades/{id}
    @GetMapping("/{id}")
    public TradeDTO get(@PathVariable String id) {                 // path variable
        return service.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(id));
    }

    // POST /api/trades  (Content-Type: application/json)
    @PostMapping
    public ResponseEntity<TradeDTO> create(@Valid @RequestBody CreateTradeRequest req) {
        var created = service.create(req);
        return ResponseEntity.created(URI.create("/api/trades/" + created.id())).body(created);
    }

    // PUT /api/trades/{id}
    @PutMapping("/{id}")
    public TradeDTO replace(@PathVariable String id, @Valid @RequestBody UpdateTradeRequest req) {
        return service.replace(id, req);
    }

    // PATCH /api/trades/{id}
    @PatchMapping("/{id}")
    public TradeDTO patch(@PathVariable String id, @RequestBody Map<String, Object> patch) {
        return service.patch(id, patch);
    }

    // DELETE /api/trades/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable String id) { service.delete(id); }
}
```

Common annotations summary:

| Annotation | What it binds |
|---|---|
| `@PathVariable` | A `{...}` segment from the URL path |
| `@RequestParam` | A query parameter (`?key=value`) |
| `@RequestBody` | The request body (JSON → Java via Jackson) |
| `@RequestHeader` | A header value |
| `@CookieValue` | A cookie value |
| `@RequestPart` | A part of a multipart upload |
| `@SessionAttribute` / `@RequestAttribute` | Servlet-level session/request attribute |

### 4.4 Validation — Bean Validation + `@Valid`

```java
public record CreateTradeRequest(
    @NotBlank(message = "Symbol is required")
    String symbol,

    @NotNull @Min(1) @Max(1_000_000)
    Integer quantity,

    @NotNull @DecimalMin("0.01")
    BigDecimal price,

    @Email
    String traderEmail,

    @Pattern(regexp = "^(BUY|SELL)$", message = "Must be BUY or SELL")
    String side
) {}

// trigger validation with @Valid on the controller parameter
@PostMapping
public TradeDTO create(@Valid @RequestBody CreateTradeRequest req) { /* ... */ }

// On failure: Spring throws MethodArgumentNotValidException → handled by advice (§4.5)
```

Common annotations: `@NotNull`, `@NotBlank`, `@NotEmpty`, `@Size(min,max)`, `@Min`, `@Max`, `@DecimalMin`, `@DecimalMax`, `@Email`, `@Pattern(regexp=...)`, `@Past`, `@Future`. From `jakarta.validation.constraints` (Bean Validation 3.0 / Jakarta Validation).

### 4.5 Global exception handling

```java
@RestControllerAdvice                                     // applies to all @RestControllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> notFound(ResourceNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> business(BusinessException e) {
        return ResponseEntity.status(422).body(new ErrorResponse(e.code(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)         // @Valid failure
    public ResponseEntity<Map<String, String>> validation(MethodArgumentNotValidException e) {
        var errors = e.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage, (a, b) -> a));
        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(Exception.class)                              // catch-all — must log
    public ResponseEntity<ErrorResponse> unexpected(Exception e) {
        log.error("unexpected", e);
        return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
        // ⚠ never leak internal details (stack traces, SQL) to clients
    }
}

public record ErrorResponse(String code, String message) {}
```

### 4.6 `ResponseEntity` — full HTTP control

When you need control beyond just the body:

```java
return ResponseEntity.ok(data);                                          // 200 + body
return ResponseEntity.status(201).body(created);                         // 201 + body
return ResponseEntity.created(locationUri).body(created);                // 201 + Location header + body
return ResponseEntity.noContent().build();                                // 204, no body
return ResponseEntity.notFound().build();                                 // 404, no body
return ResponseEntity.badRequest().body(errors);                          // 400 + body

return ResponseEntity.ok()
    .header("X-Request-Id", requestId)
    .header("Cache-Control", "max-age=300")
    .body(data);
```

### 4.7 Jackson — customising JSON

```java
public class Trade {
    @JsonProperty("trade_id") private String id;                       // rename in JSON
    @JsonIgnore             private String internalCode;              // exclude
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private LocalDateTime timestamp;
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String notes;                                              // omit if null
}
```

Global Jackson config:

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false       # ISO-8601 strings instead of epoch numbers
    default-property-inclusion: non_null      # omit nulls from output
    deserialization:
      fail-on-unknown-properties: false       # extra fields in input → ignore (don't 400)
```

> 💡 **`fail-on-unknown-properties: false`** is the right default for forward compatibility: clients might send fields you don't know about yet. Pair with strict `@Valid` for the fields you do care about.

### 4.8 HTTP clients: `RestTemplate`, `WebClient`, `RestClient`

Three options for calling another API from within Spring:

| Client | Style | Status |
|---|---|---|
| `RestTemplate` | Synchronous, blocking | **Maintenance mode** — don't use for new code |
| `WebClient` | Async, reactive (`Mono`/`Flux`) | Production-ready, works in both reactive and servlet stacks |
| `RestClient` (Spring 6.1+) | Synchronous fluent API | **The new replacement for `RestTemplate`** for sync code |

```java
// RestClient — synchronous fluent
RestClient client = RestClient.create("https://api.example.com");
User u = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .body(User.class);

// WebClient — reactive non-blocking
WebClient wc = WebClient.create("https://api.example.com");
Mono<User> userMono = wc.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);
```

For new synchronous code, use `RestClient`. For reactive code or when you need backpressure, use `WebClient`. Avoid `RestTemplate` in new code.

---

## 5. Going Deep — Interview-Level Material

### 5.1 CORS — what it is, when needed

**Cross-Origin Resource Sharing.** When a browser makes a request from one origin (`https://app.example.com`) to a different origin (`https://api.example.com`), the browser blocks it by default. The server must opt in via headers.

For "non-simple" requests (anything other than GET/POST with simple headers and content types), the browser sends a **preflight `OPTIONS`** request first to check permissions. The server must respond with the right headers.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override public void addCorsMappings(CorsRegistry r) {
        r.addMapping("/api/**")
         .allowedOrigins("https://app.example.com")
         .allowedMethods("GET", "POST", "PUT", "DELETE")
         .allowedHeaders("*")
         .allowCredentials(true)
         .maxAge(3600);
    }
}
```

CORS is enforced **only by browsers**. Server-to-server calls (curl, another service) don't apply. **CORS is not a security feature for the server** — it's a browser sandbox. Same-origin policy still requires server-side authn/authz.

### 5.2 Spring Security basics

Spring Security adds an additional filter chain to the Spring MVC pipeline. Configuration is via `SecurityFilterChain`:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                     // disable for stateless APIs
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))   // JWT bearer
            .build();
    }
}
```

Two common authentication models:

- **JWT bearer** (typical for REST APIs) — client sends `Authorization: Bearer <jwt>`. Server validates signature, extracts roles from claims. **Stateless** — no server session.
- **Session cookie** (typical for browser-rendered apps) — server creates a session, sends `JSESSIONID` cookie, validates on each request. CSRF protection required.

Authorisation by role: `hasRole("ADMIN")` (`ROLE_ADMIN` authority), `hasAuthority("READ_TRADES")` (raw authority string). Method-level: `@PreAuthorize("hasRole('ADMIN')")`.

### 5.3 API versioning strategies

| Strategy | Example | Trade-offs |
|---|---|---|
| **URI versioning** | `/api/v1/users` | Most common, easiest to route, easiest to cache. Cosmetically duplicates the URL space. |
| **Header versioning** | `Accept: application/vnd.example.v1+json` | Cleaner URLs; harder to debug, harder to cache. |
| **Query-param versioning** | `/api/users?version=1` | Easiest for clients to test ad-hoc; mixes data with metadata. |
| **No versioning, evolve compatibly** | Add new fields, never remove | Cleanest in theory; hardest in practice once breaking changes accumulate. |

Most production APIs use **URI versioning** (`/v1/`, `/v2/`) — it's explicit, debuggable, and cache-friendly.

### 5.4 Pagination and sorting

Spring Data provides `Pageable` and `Page<T>` out of the box:

```java
@GetMapping
public Page<TradeDTO> list(
        @RequestParam(required = false) String symbol,
        @PageableDefault(size = 20, sort = "timestamp,desc") Pageable pageable) {
    return service.find(symbol, pageable);
}

// Client: GET /api/trades?symbol=AAPL&page=0&size=20&sort=timestamp,desc
// Response: { content: [...], totalElements: 1234, totalPages: 62, number: 0, size: 20 }
```

For very large data sets, prefer **cursor-based pagination** (stateless, immune to inserts during pagination) over **offset-based** (`page=10000` is expensive — DB seeks 200 000 rows).

### 5.5 Idempotency keys for payment-style POSTs

A genuine `POST` is non-idempotent — retrying creates duplicates. For operations that *must* be safe to retry (payments, transfers), accept an `Idempotency-Key` header from the client:

```java
@PostMapping
public PaymentDTO charge(
        @RequestHeader("Idempotency-Key") String key,
        @Valid @RequestBody ChargeRequest req) {
    return idempotencyStore.executeOnce(key, () -> service.charge(req));
    // executeOnce: if key seen before, return the same response. Else execute and cache.
}
```

The server's idempotency store maps key → cached response (for some retention window, e.g. 24 h). Stripe's API is the reference design.

### 5.6 HATEOAS and OpenAPI

**HATEOAS** (Hypermedia as the Engine of Application State) — REST responses include links to related actions, so clients navigate by following links rather than hard-coding URLs. Spring HATEOAS supports this with `EntityModel`/`CollectionModel`. In practice, mostly used for internal APIs and rarely "discovered" by clients — most APIs document URLs out-of-band.

**OpenAPI / Swagger** — describe your API in a standard schema (YAML/JSON), generate clients, generate documentation. Spring's `springdoc-openapi` library auto-generates the schema from your controllers + DTOs and serves Swagger UI:

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.x.x</version>
</dependency>
```

Then `http://localhost:8080/swagger-ui.html` gives interactive API docs for free.

---

## 6. Memory Aids

### Decision tree: "what status code do I return?"

```
Resource was created?                          → 201 Created (+ Location header pointing to it)
Resource was updated/replaced?                  → 200 OK (with body) or 204 No Content (without)
Operation succeeded, no body to return?         → 204 No Content
Resource doesn't exist?                         → 404 Not Found
Authentication missing/invalid?                  → 401 Unauthorized
Authenticated but not allowed?                   → 403 Forbidden
Request body fails validation?                   → 400 Bad Request (or 422 if semantically wrong)
State conflict (e.g. duplicate)?                 → 409 Conflict
Server-side bug?                                 → 500 Internal Server Error
Downstream temporarily unavailable?              → 503 Service Unavailable
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "@Controller vs @RestController?" | View vs body | "@RestController = @Controller + @ResponseBody on every method. Returns data, not view names." |
| "Idempotent methods?" | GET, PUT, DELETE | "POST and PATCH are not. Idempotency matters for retries — payments need an Idempotency-Key header." |
| "How do I validate input?" | @Valid + Bean Validation | "Annotate the DTO; @Valid on the controller param triggers it. Errors → MethodArgumentNotValidException." |
| "How do I return 404?" | Throw, handle in advice | "Throw a domain exception, map it in @RestControllerAdvice." |
| "RestTemplate or WebClient?" | Sync vs reactive | "For new sync code use RestClient (Spring 6.1+). RestTemplate is in maintenance." |
| "What's CORS?" | Browser cross-origin sandbox | "Server opts in via headers. Not a server-side security mechanism." |
| "API versioning?" | URI is most common | "/v1/, /v2/. Header-based and query-based exist; URI is most debuggable." |
| "How do I secure /admin?" | SecurityFilterChain | "hasRole('ADMIN') in authorizeHttpRequests. JWT bearer for REST." |

### Three anchor pictures

1. **`@RestController` = method + URL annotations.** Spring handles the HTTP plumbing.
2. **JSON ↔ Java is automatic** via Jackson. `@RequestBody` in, return value out.
3. **Errors → `@RestControllerAdvice`.** One place to map exceptions to status codes.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: `@Controller` vs `@RestController`?
**A:** `@Controller` returns view names (Thymeleaf, JSP). `@RestController` = `@Controller` + `@ResponseBody` on every method — return values are serialised directly to the response body (JSON or XML). Use `@RestController` for REST APIs.

### Q2: Request mapping — key annotations?
**A:** `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` — shortcuts for `@RequestMapping(method=...)`. Path variables: `@PathVariable`. Query params: `@RequestParam`. Body: `@RequestBody`. Headers: `@RequestHeader`. Cookies: `@CookieValue`.

### Q3: HTTP methods — which are idempotent?
**A:** GET (safe + idempotent), PUT (idempotent), DELETE (idempotent). POST and PATCH are NOT. Idempotency matters for retries — for payment-style POSTs, accept an `Idempotency-Key` header.

### Q4: How does Spring convert Java ↔ JSON?
**A:** Jackson `ObjectMapper` is auto-configured by `spring-boot-starter-web`. `@RequestBody` deserialises JSON → Java; the return value is serialised → JSON automatically (`@ResponseBody`/`@RestController`). Customise via `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`, `@JsonInclude`, or in `application.yml` under `spring.jackson.*`.

### Q5: How do you validate request input?
**A:** Bean Validation annotations (`@NotBlank`, `@NotNull`, `@Min/@Max`, `@Email`, `@Pattern`) on the DTO. `@Valid` on the controller parameter triggers it. On failure: `MethodArgumentNotValidException` thrown — handle in `@RestControllerAdvice` to return 400 with a structured error body.

### Q6: Global exception handling?
**A:** `@RestControllerAdvice` class with `@ExceptionHandler` methods. Each handler maps an exception type to a `ResponseEntity` with the right status and body. Hierarchy: most specific handler wins; an `Exception`-typed handler is the catch-all.

### Q7: What's `ResponseEntity`?
**A:** Represents the full HTTP response — status, headers, body. `ResponseEntity.ok(data)`, `ResponseEntity.created(uri).body(data)`, `ResponseEntity.notFound().build()`, custom headers via `.header(...)`. More control than a bare return.

### Q8: API versioning approaches?
**A:** URI (`/api/v1/users`) — most common, easiest. Header (`Accept: application/vnd.example.v1+json`) — cleaner URLs but harder to debug. Query param (`?version=1`) — flexible but unconventional. URI versioning dominates production.

### Q9: `RestTemplate` vs `WebClient` vs `RestClient`?
**A:** `RestTemplate` — synchronous, **maintenance mode**, don't use for new code. `WebClient` — reactive non-blocking (`Mono`/`Flux`), works in both stacks. `RestClient` (Spring 6.1+) — synchronous fluent API, the modern replacement for `RestTemplate`.

### Q10: How do you secure REST endpoints?
**A:** Spring Security `SecurityFilterChain`. `authorizeHttpRequests(auth -> auth.requestMatchers("/admin/**").hasRole("ADMIN").anyRequest().authenticated())`. JWT bearer (`oauth2ResourceServer(oauth2 -> oauth2.jwt(...))`) for stateless REST APIs. Disable CSRF for pure-REST APIs (`.csrf(csrf -> csrf.disable())`).

### Q11: What's CORS and when is it needed?
**A:** Cross-Origin Resource Sharing — browser-enforced sandbox preventing JS from one origin calling another. Server opts in via headers. Configure in Spring with `WebMvcConfigurer.addCorsMappings()` or `@CrossOrigin`. **Browser-only** — server-to-server calls aren't subject to CORS. Not a server security mechanism.

### Q12: Content negotiation?
**A:** Server picks the response format based on the client's `Accept` header. Spring returns JSON by default (Jackson on classpath). Add Jackson XML for XML support. `produces` on the mapping restricts output formats; `consumes` restricts input. `Accept: application/json` is the default for most REST clients.

### Q13: Pagination?
**A:** Spring Data's `Pageable` parameter and `Page<T>` return. `@PageableDefault(size = 20)` sets defaults. Client: `?page=0&size=20&sort=field,desc`. Response: content + totalElements + totalPages + number + size. For very large data, consider cursor-based pagination instead of offset.

### Q14: Idempotency for payment POSTs?
**A:** Accept an `Idempotency-Key` header from the client. Server caches `key → response`; on retry with the same key, return the cached response without re-executing. Stripe's design is the canonical reference.

### Q15: Filter vs Interceptor vs Aspect?
**A:** **Filter** — servlet-level, before Spring's dispatcher; doesn't know about controllers. **HandlerInterceptor** — Spring MVC level, knows the matched handler. **Aspect (AOP)** — bean method level, deepest hook. Use the outermost level that has the info you need.

### Q16: How do you test a controller?
**A:** `@WebMvcTest(MyController.class)` slice. Inject `MockMvc`. `@MockBean` the service dependencies. `mockMvc.perform(get("/api/...")).andExpect(status().isOk()).andExpect(jsonPath("$.field").value(...))`. See [10 Testing §4.6](./10_testing.md#46-spring-boot-test-slices).

---

### Key Code Patterns

**Standard CRUD controller**
```java
@RestController @RequestMapping("/api/x")
public class XController {
    @GetMapping("/{id}")    public X get(@PathVariable Long id) { ... }
    @PostMapping            public ResponseEntity<X> create(@Valid @RequestBody CreateX req) { ... }
    @PutMapping("/{id}")    public X update(@PathVariable Long id, @Valid @RequestBody UpdateX req) { ... }
    @DeleteMapping("/{id}") @ResponseStatus(NO_CONTENT) public void delete(@PathVariable Long id) { ... }
}
```

**Validation request record**
```java
public record CreateUserRequest(@NotBlank String name, @Email String email, @Min(18) int age) {}
```

**Global error handler**
```java
@RestControllerAdvice
class Errors {
    @ExceptionHandler(NotFoundException.class)
    ResponseEntity<?> nf(NotFoundException e) { return ResponseEntity.status(404).body(Map.of("error", e.getMessage())); }
}
```

**Security filter chain (JWT)**
```java
@Bean SecurityFilterChain chain(HttpSecurity http) throws Exception {
    return http.csrf(c -> c.disable())
               .authorizeHttpRequests(a -> a.requestMatchers("/api/public/**").permitAll().anyRequest().authenticated())
               .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
               .build();
}
```

---

## 8. Self-Test

**Easy**
- [ ] Difference between `@Controller` and `@RestController`?
- [ ] Which HTTP methods are idempotent?
- [ ] What status code for "resource created"?
- [ ] What does `@Valid` do?

**Medium**
- [ ] Trace a `GET /api/users/{id}` from Tomcat to JSON response.
- [ ] Show a `@RestControllerAdvice` mapping three exception types to three status codes.
- [ ] Why is `RestClient` preferred over `RestTemplate` for new sync code?
- [ ] How would you handle pagination?

**Hard**
- [ ] Explain CORS and write a config that allows `https://app.example.com` for the API.
- [ ] Implement an idempotency-key flow for `POST /payments`.
- [ ] Compare URI / header / query-param API versioning. When would each win?
- [ ] Why disable CSRF for REST APIs but not for browser-rendered apps?
- [ ] How do you test a controller without booting the full Spring context?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **REST** | Architectural style: resource URLs + standard HTTP methods + stateless. |
| **`@RestController`** | Class whose methods produce response bodies (vs view names). |
| **`@RequestMapping` family** | Annotations mapping HTTP method + path to controller methods. |
| **DispatcherServlet** | Spring's front controller — every request goes through it. |
| **HandlerMapping** | Component that picks the controller method for a request. |
| **HandlerInterceptor** | Spring-level pre/post hook around controller invocation. |
| **Filter** | Servlet-level pre/post hook (before DispatcherServlet). |
| **Bean Validation** | Annotation-driven input validation (`@NotBlank`, `@Min`, …). |
| **`@RestControllerAdvice`** | Cross-controller error handler. |
| **`ResponseEntity`** | Full HTTP response (status + headers + body). |
| **CORS** | Browser sandbox for cross-origin AJAX; server opts in via headers. |
| **CSRF** | Cross-Site Request Forgery — relevant for cookie-auth, not for token-auth REST. |
| **Idempotency** | Same operation, same result, no matter how many times you call it. |
| **Pageable / Page** | Spring Data's pagination abstraction. |
| **Idempotency-Key** | A client-supplied header to deduplicate retried POSTs. |
| **OpenAPI / Swagger** | Standard API description format; tools generate clients and docs. |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/13_spring_rest_doubts.md) · [← Prev: 12 Spring Boot](./12_spring_boot.md) · [Next: 14 Spring AOP & Data →](./14_spring_aop_data.md)

[↑ Back to top](#13--spring-rest--web)
