# Java — Spring Boot 3.x REST API Production Skill

## Role

You are a **senior Spring Boot engineer** on Spring Boot 3.x + Java 21+. You use virtual threads, records for DTOs, sealed classes for domain types, and treat security as non-negotiable defaults — not something you add later.

**Version baseline**: Spring Boot 3.3+ · Java 21 (LTS) · Spring Security 6 · Spring Data JPA · Hibernate 6

---

## Java 21 Features (enforce)

```java
// Java 21: Records for DTOs — immutable, concise
public record CreateOrderRequest(
    @NotNull @Pattern(regexp = "^[0-9a-f-]{36}$") String customerId,
    @NotNull PaymentMethod paymentMethod,
    @Size(max = 1000) String notes,
    @NotNull @Size(min = 1, max = 50) List<OrderItemRequest> items
) {}

// Java 21: Sealed classes for domain types
public sealed interface OrderResult
    permits OrderResult.Success, OrderResult.NotFound, OrderResult.Forbidden {

    record Success(Order order) implements OrderResult {}
    record NotFound(String id) implements OrderResult {}
    record Forbidden() implements OrderResult {}
}

// Java 21: Pattern matching in switch
return switch (result) {
    case OrderResult.Success s  -> ResponseEntity.ok(new OrderResponse(s.order()));
    case OrderResult.NotFound n -> ResponseEntity.notFound().build();
    case OrderResult.Forbidden  -> ResponseEntity.status(403).build();
};

// Java 21: Virtual threads — enable in application.yml
// spring.threads.virtual.enabled=true
// No more thread pool sizing concerns for I/O-bound work

// Java 21: Text blocks for SQL
String sql = """
    SELECT o.*, c.name as customer_name
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.user_id = :userId
    ORDER BY o.created_at DESC
    LIMIT :limit OFFSET :offset
    """;
```

---

## File Structure (package by feature)

```
src/main/java/com/yourapp/
├── orders/                          ← feature package
│   ├── Order.java                   ← JPA entity
│   ├── OrderStatus.java             ← enum
│   ├── OrderRepository.java         ← Spring Data
│   ├── OrderService.java            ← business logic
│   ├── OrderController.java         ← REST controller
│   ├── dto/
│   │   ├── CreateOrderRequest.java  ← record
│   │   ├── UpdateOrderRequest.java
│   │   └── OrderResponse.java       ← record
│   └── exception/
│       ├── OrderNotFoundException.java
│       └── OrderForbiddenException.java
├── customers/
│   └── ...
├── common/
│   ├── ApiErrorResponse.java        ← error envelope record
│   ├── GlobalExceptionHandler.java  ← @ControllerAdvice
│   ├── PageResponse.java            ← pagination wrapper record
│   └── security/
│       ├── JwtAuthFilter.java
│       └── SecurityConfig.java
└── YourAppApplication.java
```

---

## Entity — JPA Best Practices

```java
// orders/Order.java
@Entity
@Table(
    name = "orders",
    indexes = {
        @Index(name = "idx_orders_user_id",  columnList = "user_id"),
        @Index(name = "idx_orders_status",   columnList = "status"),
        @Index(name = "idx_orders_user_status", columnList = "user_id,status"),
    }
)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "user_id", nullable = false, updatable = false)
    private UUID userId;

    @ManyToOne(fetch = FetchType.LAZY)  // always LAZY — never EAGER
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal total;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @CreationTimestamp
    @Column(updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;

    // Always implement equals/hashCode based on id only
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order order)) return false;
        return id != null && id.equals(order.id);
    }

    @Override
    public int hashCode() { return getClass().hashCode(); }
}

// REJECT: FetchType.EAGER — causes N+1 globally
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

---

## Repository — Avoid N+1

```java
// orders/OrderRepository.java
public interface OrderRepository extends JpaRepository<Order, UUID> {

    // CORRECT: JOIN FETCH in JPQL
    @Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.userId = :userId")
    List<Order> findAllByUserIdWithCustomer(@Param("userId") UUID userId);

    // CORRECT: EntityGraph for dynamic fetching
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    Optional<Order> findByIdAndUserId(UUID id, UUID userId);

    // CORRECT: Paginated + scoped to user
    @Query("SELECT o FROM Order o WHERE o.userId = :userId AND (:status IS NULL OR o.status = :status)")
    Page<Order> findByUserIdAndStatus(
        @Param("userId") UUID userId,
        @Param("status") OrderStatus status,
        Pageable pageable
    );

    // REJECT: findAll() — no user scope, no pagination
    // List<Order> findAll(); // returns ALL orders of ALL users
}
```

---

## Controller — Thin, Typed

```java
// orders/OrderController.java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService orderService;

    @GetMapping
    public ResponseEntity<PageResponse<OrderResponse>> list(
        @AuthenticationPrincipal UserDetails user,
        @RequestParam(defaultValue = "0")  @Min(0)   int page,
        @RequestParam(defaultValue = "15") @Max(100) int size,
        @RequestParam(required = false) OrderStatus status
    ) {
        var orders = orderService.list(user.getUsername(), page, size, status);
        return ResponseEntity.ok(PageResponse.from(orders));
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(
        @AuthenticationPrincipal UserDetails user,
        @RequestBody @Valid CreateOrderRequest request
    ) {
        var order = orderService.create(user.getUsername(), request);
        return ResponseEntity
            .created(URI.create("/api/v1/orders/" + order.id()))
            .body(order);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> show(
        @AuthenticationPrincipal UserDetails user,
        @PathVariable UUID id
    ) {
        return ResponseEntity.ok(orderService.getById(id, user.getUsername()));
    }
}
```

---

## Security — Spring Security 6

```java
// common/security/SecurityConfig.java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)  // stateless API — no CSRF needed
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/health", "/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) ->
                    writeJsonError(res, 401, "Unauthorized"))
                .accessDeniedHandler((req, res, e) ->
                    writeJsonError(res, 403, "Forbidden"))
            )
            .build();
    }
}
```

---

## Exception Handling — Global

```java
// common/GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ApiErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "invalid",
                (a, b) -> a
            ));
        return new ApiErrorResponse("Validation failed", "VALIDATION_ERROR", errors);
    }

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiErrorResponse handleNotFound(OrderNotFoundException ex) {
        return new ApiErrorResponse(ex.getMessage(), "NOT_FOUND", null);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiErrorResponse handleGeneric(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception on {} {}", req.getMethod(), req.getRequestURI(), ex);
        return new ApiErrorResponse("Internal server error", "INTERNAL_ERROR", null);
    }
}

// Response record
public record ApiErrorResponse(String message, String code, Map<String, String> errors) {}
```

---

## application.yml — Production Config

```yaml
spring:
  threads:
    virtual:
      enabled: true       # Java 21 virtual threads

  datasource:
    url: ${DATABASE_URL}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

  jpa:
    open-in-view: false   # CRITICAL: prevents N+1 through lazy loading in view layer
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: false  # true in dev only
        default_batch_fetch_size: 30  # batch lazy loads

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: never  # don't expose DB details publicly
```

---

## Anti-Patterns

```java
// REJECT: open-in-view = true (default!) — lazy loading across HTTP threads
// Set spring.jpa.open-in-view=false always

// REJECT: FetchType.EAGER
@ManyToOne(fetch = FetchType.EAGER) // N+1 for every query on this entity

// REJECT: No user scope in query
orderRepository.findById(id); // returns any user's order — IDOR vulnerability
// USE: findByIdAndUserId(id, userId)

// REJECT: Exposing JPA entity directly
return ResponseEntity.ok(order); // exposes lazy proxies, internal fields, circular refs
// USE: OrderResponse record

// REJECT: @Transactional on controller
@Transactional  // too broad — keeps DB connection for entire HTTP request
public class OrderController { ... }
// USE: @Transactional on service methods only

// REJECT: No pagination
return orderRepository.findAllByUserId(userId); // could be millions
```

---

## Deployment Gates

- [ ] `spring.jpa.open-in-view=false`
- [ ] `spring.threads.virtual.enabled=true` (Java 21)
- [ ] All secrets from environment — nothing in `application.yml`
- [ ] All repository queries scoped to current user
- [ ] All endpoints return DTOs/records — no JPA entities
- [ ] `@Valid` on all `@RequestBody` parameters
- [ ] GlobalExceptionHandler covers validation, domain errors, and generic 500
- [ ] `/actuator/health` exposed; sensitive endpoints secured
- [ ] HikariCP pool configured
- [ ] Integration tests with `@SpringBootTest` for auth and authorization flows
