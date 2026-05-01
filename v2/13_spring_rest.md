# 13 — Spring REST & Web

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 🟢 Start Here — REST APIs in Plain English

### What is a REST API?

A REST API is how **two programs talk to each other over the internet**. Your phone app talks to a server, your frontend talks to a backend — they use REST APIs.

Think of it like a restaurant:
- **You** (the client) = order food from the menu
- **The waiter** (the API) = takes your order, brings your food
- **The kitchen** (the server) = prepares the food

You don't walk into the kitchen yourself — you use the waiter (API) as the interface.

### HTTP Methods — the four basic actions

| Action | HTTP Method | Restaurant analogy | Example |
|--------|:----------:|-------------------|---------|
| **Read** | `GET` | "What's on the menu?" | `GET /api/users/123` |
| **Create** | `POST` | "I'd like to order this" | `POST /api/users` (with data) |
| **Update** | `PUT` | "Change my order" | `PUT /api/users/123` (with new data) |
| **Delete** | `DELETE` | "Cancel my order" | `DELETE /api/users/123` |

### A simple Spring REST controller

```java
@RestController                          // "this class handles web requests"
@RequestMapping("/api/users")            // base URL for all methods below
public class UserController {
    
    @GetMapping("/{id}")                 // GET /api/users/123
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);  // returns JSON automatically
    }
    
    @PostMapping                         // POST /api/users
    public User createUser(@RequestBody User user) {  // reads JSON from request body
        return userService.save(user);
    }
}
// Spring automatically converts Java objects ↔ JSON using Jackson
```

### What's JSON?

JSON is the **language** APIs use to send data. It looks like this:

```json
{
    "id": 123,
    "name": "Alice",
    "email": "alice@citi.com"
}
```

Spring automatically converts your Java objects to JSON (and back) — you don't have to do it manually.

> Key takeaway: A REST API = a menu of URLs that clients can call to read, create, update, or delete data. Spring handles the conversion between Java objects and JSON. You just define the endpoints.

---

## 📚 Study Material

### 1. REST Principles

```
REST = Representational State Transfer — architectural style for web APIs

Key constraints:
1. Client-Server — separation of concerns
2. Stateless — each request contains all info needed (no server-side session)
3. Uniform Interface — standard HTTP methods on resources
4. Layered System — client doesn't know if it's talking to server or proxy
5. Cacheable — responses declare cacheability
```

**Resource-oriented design:**
```
GET    /api/trades              → list all trades (200)
GET    /api/trades/{id}         → get one trade (200 or 404)
POST   /api/trades              → create a trade (201 Created)
PUT    /api/trades/{id}         → full replace (200 or 204)
PATCH  /api/trades/{id}         → partial update (200)
DELETE /api/trades/{id}         → delete (204 No Content)
```

| Method | Idempotent? | Safe? | Body? | Use |
|--------|:-----------:|:-----:|:-----:|-----|
| GET | ✅ | ✅ | No | Read |
| POST | ❌ | ❌ | Yes | Create (non-idempotent) |
| PUT | ✅ | ❌ | Yes | Full replace |
| PATCH | ❌* | ❌ | Yes | Partial update |
| DELETE | ✅ | ❌ | No | Remove |

💡 **Idempotency matters in banking:** If a payment POST is retried (network timeout), it could create duplicates. Use idempotency keys or PUT for critical operations.

### 2. Spring MVC Request Lifecycle

```
HTTP Request
    │
    ▼
DispatcherServlet (front controller)
    │
    ▼
HandlerMapping (find which controller handles this URL)
    │
    ▼
Filter chain (Spring Security, CORS, logging)
    │
    ▼
HandlerInterceptor.preHandle()
    │
    ▼
Controller method (@GetMapping, @PostMapping, etc.)
    │  ├─ @RequestBody → Jackson deserialises JSON → Java object
    │  ├─ @Valid → Bean Validation runs
    │  └─ Method executes → returns object
    │
    ▼
HandlerInterceptor.postHandle()
    │
    ▼
Jackson serialises Java → JSON (@ResponseBody)
    │
    ▼
HTTP Response
```

### 3. Controller Patterns

```java
@RestController                               // @Controller + @ResponseBody on every method
@RequestMapping("/api/trades")                // base path for all methods
public class TradeController {
    
    private final TradeService tradeService;
    
    public TradeController(TradeService tradeService) {
        this.tradeService = tradeService;
    }
    
    // GET /api/trades?symbol=AAPL&page=0&size=20
    @GetMapping
    public Page<TradeDTO> listTrades(
            @RequestParam(required = false) String symbol,    // query parameter
            @PageableDefault(size = 20) Pageable pageable) {  // pagination
        return tradeService.findTrades(symbol, pageable);
    }
    
    // GET /api/trades/{id}
    @GetMapping("/{id}")
    public TradeDTO getTrade(@PathVariable String id) {       // path variable
        return tradeService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Trade not found: " + id));
    }
    
    // POST /api/trades
    @PostMapping
    public ResponseEntity<TradeDTO> createTrade(
            @Valid @RequestBody CreateTradeRequest request) {  // validated request body
        TradeDTO created = tradeService.create(request);
        URI location = URI.create("/api/trades/" + created.getId());
        return ResponseEntity.created(location).body(created);  // 201 + Location header
    }
    
    // PUT /api/trades/{id}
    @PutMapping("/{id}")
    public TradeDTO updateTrade(
            @PathVariable String id,
            @Valid @RequestBody UpdateTradeRequest request) {
        return tradeService.update(id, request);
    }
    
    // DELETE /api/trades/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)                    // 204
    public void deleteTrade(@PathVariable String id) {
        tradeService.delete(id);
    }
}
```

### 4. Request Validation

```java
public class CreateTradeRequest {
    @NotBlank(message = "Symbol is required")
    private String symbol;
    
    @NotNull @Min(1) @Max(1_000_000)
    private Integer quantity;
    
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal price;
    
    @Email
    private String traderEmail;
    
    @Pattern(regexp = "^(BUY|SELL)$", message = "Must be BUY or SELL")
    private String side;
}

// Validation triggered by @Valid on controller parameter
// If validation fails → MethodArgumentNotValidException → 400 Bad Request

// Custom error response:
@RestControllerAdvice
public class ValidationAdvice {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage,
                (e1, e2) -> e1));
        return ResponseEntity.badRequest().body(errors);
    }
}
// Response: { "symbol": "Symbol is required", "quantity": "must be greater than 0" }
```

### 5. Global Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity.status(404).body(
            new ErrorResponse("NOT_FOUND", e.getMessage(), Instant.now()));
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        return ResponseEntity.status(422).body(
            new ErrorResponse(e.getCode(), e.getMessage(), Instant.now()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(500).body(
            new ErrorResponse("INTERNAL_ERROR", "Something went wrong", Instant.now()));
    }
}

// Consistent error body for ALL errors:
public record ErrorResponse(String code, String message, Instant timestamp) {}
```

### 6. ResponseEntity — Full HTTP Control

```java
// Return with status + headers + body
return ResponseEntity.ok(data);                              // 200 + body
return ResponseEntity.status(201).body(created);             // 201 + body
return ResponseEntity.created(locationUri).body(created);    // 201 + Location header + body
return ResponseEntity.noContent().build();                   // 204, no body
return ResponseEntity.notFound().build();                    // 404, no body
return ResponseEntity.badRequest().body(errors);             // 400 + error body

// Custom headers
return ResponseEntity.ok()
    .header("X-Request-Id", requestId)
    .header("Cache-Control", "max-age=300")
    .body(data);
```

### 7. Jackson — JSON Serialisation

```java
// Jackson auto-configured by spring-boot-starter-web
// Customise with annotations:
public class Trade {
    @JsonProperty("trade_id")                  // rename in JSON
    private String id;
    
    @JsonIgnore                                // exclude from JSON
    private String internalCode;
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")  // date format
    private LocalDateTime timestamp;
    
    @JsonInclude(JsonInclude.Include.NON_NULL)   // omit if null
    private String notes;
}

// Global configuration in application.yml:
// spring:
//   jackson:
//     serialization:
//       write-dates-as-timestamps: false    # ISO-8601 instead of epoch
//     default-property-inclusion: non_null   # omit nulls globally
```

### 8. REST Client — RestTemplate vs WebClient vs RestClient

```java
// RestTemplate — synchronous, legacy (maintenance mode)
RestTemplate rt = new RestTemplate();
ResponseEntity<User> response = rt.getForEntity("/api/users/{id}", User.class, 1);

// WebClient — reactive, non-blocking (works in both reactive and servlet stacks)
WebClient client = WebClient.create("http://api.example.com");
Mono<User> user = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

// RestClient (Spring 6.1) — synchronous fluent API (REPLACEMENT for RestTemplate)
RestClient client = RestClient.create("http://api.example.com");
User user = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .body(User.class);

// Use: RestClient for new synchronous code, WebClient for reactive
```

### 9. CORS & Security Basics

```java
// CORS — allow frontend (different origin) to call your API
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.citi.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .maxAge(3600);
    }
}

// Spring Security — basic SecurityFilterChain
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                              // disable for APIs
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

---

## Rapid-Fire Q&A

### Q1: `@Controller` vs `@RestController`?
**A:** `@Controller` returns view names (for MVC/Thymeleaf). `@RestController` = `@Controller` + `@ResponseBody` — every method returns data directly (JSON/XML), no view resolution.

### Q2: Request mapping — key annotations?
**A:** `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`. All are shortcuts for `@RequestMapping(method=...)`. Path variables: `@PathVariable`. Query params: `@RequestParam`. Request body: `@RequestBody`.

### Q3: HTTP methods — which are idempotent?
**A:** `GET` — safe + idempotent. `PUT` — idempotent (same request = same result). `DELETE` — idempotent (deleting again = same state). `POST` — NOT idempotent (can create duplicates). `PATCH` — not guaranteed idempotent. Idempotency is critical in payment systems — double-submit safety.

### Q4: How does Spring convert Java objects to JSON?
**A:** Jackson `ObjectMapper` auto-configured by Spring Boot. `@RequestBody` deserialises JSON → Java. `@ResponseBody` serialises Java → JSON. Customise with `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`. Register custom serialisers via `@JsonSerialize`.

### Q5: How do you validate request input?
**A:** `@Valid` or `@Validated` on the `@RequestBody` parameter. Bean Validation annotations on the POJO: `@NotNull`, `@NotBlank`, `@Size(min,max)`, `@Email`, `@Min`, `@Max`, `@Pattern`. Validation errors return 400 by default. Customise error response via `@ControllerAdvice`.

### Q6: Global exception handling?
**A:** `@RestControllerAdvice` class with `@ExceptionHandler` methods. E.g., `@ExceptionHandler(ResourceNotFoundException.class)` returns 404 with custom body. Centralises error formatting. `ResponseStatusException` for one-off throws.

### Q7: What's `ResponseEntity`?
**A:** Represents the full HTTP response: status code + headers + body. `return ResponseEntity.ok(data)`, `ResponseEntity.status(201).body(created)`, `ResponseEntity.notFound().build()`. More control than just returning the body.

### Q8: API versioning — approaches?
**A:** URI: `/api/v1/users`. Header: `Accept: application/vnd.company.v1+json`. Query param: `?version=1`. URI versioning is simplest and most common.

### Q9: `RestTemplate` vs `WebClient` vs `RestClient`?
**A:** `RestTemplate`: synchronous, legacy (maintenance mode). `WebClient`: reactive, non-blocking, works in both reactive and servlet stacks. `RestClient` (Spring 6.1): synchronous fluent API replacing `RestTemplate`. Use `RestClient` for new synchronous code; `WebClient` for reactive.

### Q10: How do you secure REST endpoints?
**A:** Spring Security with `SecurityFilterChain`. `http.authorizeHttpRequests(auth -> auth.requestMatchers("/admin/**").hasRole("ADMIN").anyRequest().authenticated())`. JWT for stateless auth. OAuth2 for delegated auth. CORS configuration for frontend access.

### Q11: What's CORS and how do you configure it?
**A:** Cross-Origin Resource Sharing — browser security. Backend must explicitly allow other origins. Configure with `@CrossOrigin` per controller, or globally via `WebMvcConfigurer.addCorsMappings()`. In Spring Security: `http.cors(cors -> cors.configurationSource(...))`.

### Q12: Content negotiation?
**A:** Spring returns JSON by default (Jackson on classpath). Can support XML (add Jackson XML). Client sends `Accept: application/xml`. `produces` attribute on mapping restricts output. `consumes` restricts input.

---

## Can you answer these cold?

- [ ] `@RestController` — what it combines
- [ ] Idempotent HTTP methods — which ones and why it matters
- [ ] Request validation — `@Valid` + Bean Validation annotations
- [ ] Global exception handling — `@RestControllerAdvice` + `@ExceptionHandler`
- [ ] `RestTemplate` vs `WebClient` vs `RestClient` — when each
- [ ] Spring Security basics — `SecurityFilterChain`, JWT

[← Back to Index](./00_INDEX.md)
