# 13 — Spring REST & Web

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

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
