# 11 — Spring Core & Dependency Injection

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## Rapid-Fire Q&A

### Q1: What's IoC (Inversion of Control)?
**A:** The framework controls object creation and wiring — not your code. You declare dependencies, the container provides them. Spring's IoC container manages the full bean lifecycle: creation → dependency injection → initialisation → destruction.

### Q2: What's Dependency Injection? Three types?
**A:** Container provides dependencies to a class. Three types: (1) **Constructor injection** (preferred — makes dependencies explicit, supports immutability, required deps). (2) **Setter injection** (optional deps). (3) **Field injection** (`@Autowired` on field — discouraged, can't use in tests without reflection).

### Q3: Why is constructor injection preferred?
**A:** (1) Dependencies are explicit — no hidden coupling. (2) Fields can be `final` — immutable. (3) Easier to test — pass mocks via constructor. (4) Circular dependencies fail fast at startup instead of at runtime. Since Spring 4.3, single constructor doesn't even need `@Autowired`.

### Q4: What are bean scopes?
**A:** `singleton` (default — one instance per container), `prototype` (new instance per injection), `request` (per HTTP request), `session` (per HTTP session), `application` (per ServletContext).

### Q5: How do you inject a Prototype bean into a Singleton?
**A:** Direct injection gives you one instance forever. Solutions: (1) `@Lookup` method — Spring overrides it via CGLIB to return fresh prototype. (2) `ObjectFactory<T>` or `Provider<T>` injection — call `.getObject()` each time. (3) `ApplicationContext.getBean()` — service locator (less clean).

### Q6: What's `@Qualifier`?
**A:** Disambiguates when multiple beans of the same type exist. `@Autowired @Qualifier("stripe")` selects the bean named "stripe". Alternative: `@Primary` marks a default bean.

### Q7: Bean lifecycle — what hooks exist?
**A:** Construction → `@PostConstruct` (or `InitializingBean.afterPropertiesSet()`) → ready → `@PreDestroy` (or `DisposableBean.destroy()`). Also: `BeanPostProcessor` for cross-cutting init logic. `@PostConstruct` runs after all dependencies are injected.

### Q8: What are Spring profiles?
**A:** Environment-specific configuration. `@Profile("dev")` activates bean only in dev. `application-dev.yml` loaded when `spring.profiles.active=dev`. Common: dev, staging, prod profiles with different datasource, logging, feature flags.

### Q9: `@ConfigurationProperties` vs `@Value`?
**A:** `@Value("${app.timeout}")` — single property, no type safety. `@ConfigurationProperties(prefix="app")` — binds a whole group to a POJO. Preferred for structured config — type-safe, IDE support, validation with `@Validated`.

### Q10: How does Spring resolve circular dependencies?
**A:** With setter/field injection: Spring creates a partially initialised bean, injects it, then completes initialisation (three-level cache). With constructor injection: fails at startup — which is actually good (design smell). Fix: break the cycle with `@Lazy` or restructure.

### Q11: What's `@Component` vs `@Service` vs `@Repository` vs `@Controller`?
**A:** All are `@Component` stereotypes — Spring auto-detects them via component scan. Semantic difference: `@Service` → business logic, `@Repository` → data access (adds exception translation), `@Controller` → web layer. Use the right one for clarity.

### Q12: What's `@Configuration` and `@Bean`?
**A:** `@Configuration` class contains `@Bean` factory methods. Each `@Bean` method returns an object registered in the container. Used for third-party classes you can't annotate with `@Component`. Methods are proxied — calling a `@Bean` method again returns the same singleton.

---

## Can you answer these cold?

- [ ] Three DI types — constructor (preferred), setter, field
- [ ] Bean scopes — singleton, prototype, request, session
- [ ] Prototype into singleton problem — three solutions
- [ ] Bean lifecycle — `@PostConstruct` / `@PreDestroy`
- [ ] Circular dependency — how Spring resolves it, why constructor injection fails
- [ ] `@ConfigurationProperties` — why preferred over `@Value`

[← Back to Index](./00_INDEX.md)
