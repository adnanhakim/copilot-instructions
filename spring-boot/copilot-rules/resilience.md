# Resilience & Fault Tolerance Best Practices

## 1. Circuit Breaker Pattern

### ✅ DO: Implement circuit breakers for external calls

```java
// DO: Use Resilience4j circuit breaker
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentGateway paymentGateway;
    private final CircuitBreaker circuitBreaker;

    public PaymentResult processPayment(PaymentRequest request) {
        return circuitBreaker.executeSupplier(
            () -> paymentGateway.process(request)
        );
    }
}

// DO: Configure circuit breaker with appropriate thresholds
@Configuration
public class ResilienceConfig {

    @Bean
    public CircuitBreaker paymentCircuitBreaker(CircuitBreakerRegistry registry) {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                    // Open at 50% failure
            .slowCallRateThreshold(50)                   // Open at 50% slow calls
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .slidingWindowSize(10)
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .build();

        return registry.circuitBreaker("payment", config);
    }
}

// DO: Use annotations for cleaner code
@Service
public class ExternalApiService {

    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallback")
    public ApiResponse callExternalApi(ApiRequest request) {
        return externalApiClient.call(request);
    }

    private ApiResponse fallback(ApiRequest request, Throwable t) {
        log.warn("External API failed, using fallback", t);
        return ApiResponse.cached();  // Return cached or default response
    }
}

// DO: Combine with fallback for graceful degradation
@CircuitBreaker(name = "inventory", fallbackMethod = "getInventoryFallback")
public InventoryStatus getInventory(String productId) {
    return inventoryService.check(productId);
}

private InventoryStatus getInventoryFallback(String productId, Throwable t) {
    return InventoryStatus.unknown();  // Allow order with manual verification
}
```

**When to use:** External API calls, database connections, any remote service call.

**When NOT to use:** Local method calls, in-memory operations.

### ❌ DON'T: Ignore circuit breaker best practices

```java
// DON'T: No fallback strategy
@CircuitBreaker(name = "payment")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentGateway.process(request);
    // When circuit opens, exception propagates with no graceful handling
}

// DON'T: Circuit breaker on non-idempotent operations without care
@CircuitBreaker(name = "order")
public Order createOrder(OrderRequest request) {
    return orderService.create(request);
    // If circuit flips during retry, duplicate orders possible
}

// DON'T: Same circuit breaker for unrelated services
@CircuitBreaker(name = "external")  // Too generic!
public void callPaymentGateway() { }

@CircuitBreaker(name = "external")  // Same breaker!
public void callShippingService() { }
// One failing service opens circuit for all

// Better: Separate circuit breakers per service
@CircuitBreaker(name = "paymentGateway")
public void callPaymentGateway() { }

@CircuitBreaker(name = "shippingService")
public void callShippingService() { }
```

---

## 2. Retry Pattern

### ✅ DO: Implement smart retries

```java
// DO: Configure retries with exponential backoff
@Configuration
public class RetryConfig {

    @Bean
    public Retry paymentRetry(RetryRegistry registry) {
        io.github.resilience4j.retry.RetryConfig config =
            io.github.resilience4j.retry.RetryConfig.custom()
                .maxAttempts(3)
                .waitDuration(Duration.ofMillis(500))
                .exponentialBackoff(Duration.ofMillis(500), 2.0)
                .retryOnException(e -> e instanceof TransientException)
                .retryExceptions(TimeoutException.class, IOException.class)
                .ignoreExceptions(ValidationException.class)
                .build();

        return registry.retry("payment", config);
    }
}

// DO: Use annotations
@Service
public class OrderService {

    @Retry(name = "orderCreation", fallbackMethod = "createOrderFallback")
    public Order createOrder(OrderRequest request) {
        return externalOrderService.submit(request);
    }

    private Order createOrderFallback(OrderRequest request, Throwable t) {
        log.error("Order creation failed after retries", t);
        return Order.pending(request);  // Queue for manual processing
    }
}

// DO: Combine retry with circuit breaker
@CircuitBreaker(name = "inventory")
@Retry(name = "inventory")
public InventoryStatus checkInventory(String productId) {
    return inventoryClient.check(productId);
}

// DO: Retry only transient failures
@Retry(name = "database")
public User findUser(String id) {
    try {
        return userRepository.findById(id).orElseThrow();
    } catch (TransientDataAccessException e) {
        throw e;  // Retry
    } catch (DataAccessException e) {
        throw new ServiceException("Database error", e);  // Don't retry
    }
}

// DO: Add jitter to prevent thundering herd
RetryConfig.custom()
    .waitDuration(Duration.ofMillis(500))
    .exponentialBackoffWithJitter(Duration.ofMillis(500), 2.0, 0.5)
    .build();
```

**When to use:** Network failures, transient database issues, rate limiting responses.

**When NOT to use:** Validation errors, business logic failures, non-idempotent operations.

### ❌ DON'T: Retry blindly

```java
// DON'T: Retry validation failures
@Retry(name = "order")
public Order createOrder(OrderRequest request) {
    if (request.amount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new ValidationException("Invalid amount");
        // Will retry 3 times - pointless!
    }
}

// DON'T: Retry non-idempotent operations
@Retry(name = "payment")
public PaymentResult chargeCustomer(String customerId, BigDecimal amount) {
    return paymentGateway.charge(customerId, amount);
    // If first call succeeds but times out, retry charges twice!
}

// Better: Use idempotency key
@Retry(name = "payment")
public PaymentResult chargeCustomer(String idempotencyKey, String customerId, BigDecimal amount) {
    return paymentGateway.charge(idempotencyKey, customerId, amount);
}

// DON'T: Infinite retries
RetryConfig.custom()
    .maxAttempts(Integer.MAX_VALUE)  // Never give up = stuck forever
    .build();

// DON'T: No backoff
RetryConfig.custom()
    .maxAttempts(10)
    .waitDuration(Duration.ZERO)  // Hammer the failing service
    .build();

// DON'T: Fixed delay under load
RetryConfig.custom()
    .waitDuration(Duration.ofSeconds(1))  // All retries at same time
    .build();

// Better: Exponential backoff with jitter
RetryConfig.custom()
    .exponentialBackoffWithJitter(Duration.ofMillis(100), 2.0, 0.5)
    .build();
```

---

## 3. Timeout Pattern

### ✅ DO: Set appropriate timeouts

```java
// DO: Configure timeouts at multiple levels
@Configuration
public class TimeoutConfig {

    // HTTP client timeout
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .requestFactory(new HttpComponentsClientHttpRequestFactory(
                HttpClientBuilder.create()
                    .setConnectionManager(PoolingHttpClientConnectionManagerBuilder.create()
                        .setDefaultConnectionConfig(ConnectionConfig.custom()
                            .setConnectTimeout(Timeout.ofSeconds(5))
                            .setSocketTimeout(Timeout.ofSeconds(30))
                            .build())
                        .build())
                    .build()))
            .build();
    }

    // Resilience4j timeout
    @Bean
    public TimeLimiter paymentTimeLimiter(TimeLimiterRegistry registry) {
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(10))
            .cancelRunningFuture(true)
            .build();

        return registry.timeLimiter("payment", config);
    }
}

// DO: Use @TimeLimiter annotation
@Service
public class ExternalApiService {

    @TimeLimiter(name = "externalApi")
    @CircuitBreaker(name = "externalApi")
    public CompletableFuture<ApiResponse> callApi(ApiRequest request) {
        return CompletableFuture.supplyAsync(() ->
            apiClient.call(request)
        );
    }
}

// DO: Database query timeout
public interface OrderRepository extends JpaRepository<Order, String> {

    @QueryHints(@QueryHint(name = "jakarta.persistence.query.timeout", value = "5000"))
    List<Order> findByStatus(OrderStatus status);
}

// DO: Transaction timeout
@Transactional(timeout = 30)
public void processLargeOrder(Order order) {
    // Times out after 30 seconds
}
```

**When to use:** All external calls, database queries, any I/O operation.

### ❌ DON'T: Ignore timeouts

```java
// DON'T: No timeout on HTTP calls
RestClient restClient = RestClient.create();  // Default: infinite timeout!

// DON'T: Timeout too long
TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofMinutes(30))  // Thread blocked for 30 min
    .build();

// DON'T: Timeout too short
TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofMillis(100))  // Normal calls fail
    .build();

// DON'T: Ignore timeout exceptions
try {
    result = externalService.call();
} catch (TimeoutException e) {
    // Silently ignore - caller waits forever
}

// Better: Handle timeout gracefully
try {
    return externalService.call();
} catch (TimeoutException e) {
    log.warn("External service timeout", e);
    return cachedResult();
}
```

---

## 4. Bulkhead Pattern

### ✅ DO: Isolate resources

```java
// DO: Thread pool bulkhead for different operations
@Configuration
public class BulkheadConfig {

    @Bean
    public ThreadPoolBulkhead paymentBulkhead(ThreadPoolBulkheadRegistry registry) {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(25)
            .keepAliveDuration(Duration.ofSeconds(20))
            .build();

        return registry.bulkhead("payment", config);
    }

    @Bean
    public Bulkhead inventoryBulkhead(BulkheadRegistry registry) {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(20)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();

        return registry.bulkhead("inventory", config);
    }
}

// DO: Use annotations
@Service
public class OrderService {

    @Bulkhead(name = "orderProcessing", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<Order> processOrder(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> doProcess(request));
    }

    @Bulkhead(name = "orderQuery", type = Bulkhead.Type.SEMAPHORE)
    public List<Order> findOrders(OrderFilter filter) {
        return orderRepository.findAll(filter);
    }
}

// DO: Separate bulkheads for critical vs non-critical
@Bulkhead(name = "criticalPayment")  // Dedicated resources
public PaymentResult processPayment(PaymentRequest request) { }

@Bulkhead(name = "notifications")  // Separate pool
public void sendNotification(Notification notification) { }
```

**When to use:** Isolate slow/unreliable dependencies, protect critical resources.

### ❌ DON'T: Shared resource pools

```java
// DON'T: One pool for everything
@Bulkhead(name = "shared")
public void processPayment() { }

@Bulkhead(name = "shared")
public void sendEmail() { }

@Bulkhead(name = "shared")
public void generateReport() { }
// Slow reports exhaust pool, blocking payments

// DON'T: Unlimited concurrency
BulkheadConfig.custom()
    .maxConcurrentCalls(Integer.MAX_VALUE)  // No protection!
    .build();

// DON'T: Queue too large
ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .queueCapacity(10000)  // Memory issues, long wait times
    .build();
```

---

## 5. Rate Limiting

### ✅ DO: Implement rate limiting

```java
// DO: Configure rate limiter
@Configuration
public class RateLimitConfig {

    @Bean
    public RateLimiter apiRateLimiter(RateLimiterRegistry registry) {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(100)  // 100 calls
            .limitRefreshPeriod(Duration.ofSeconds(1))  // per second
            .timeoutDuration(Duration.ofMillis(500))  // wait time
            .build();

        return registry.rateLimiter("api", config);
    }
}

// DO: Use annotations
@RestController
public class ApiController {

    @RateLimiter(name = "api", fallbackMethod = "rateLimitExceeded")
    @GetMapping("/data")
    public Data getData() {
        return dataService.fetch();
    }

    private Data rateLimitExceeded(Throwable t) {
        throw new TooManyRequestsException("Rate limit exceeded");
    }
}

// DO: Per-user rate limiting
@Service
public class UserRateLimitedService {

    private final RateLimiterRegistry registry;

    public Data fetchDataForUser(String userId) {
        RateLimiter limiter = registry.rateLimiter(
            "user-" + userId,
            RateLimiterConfig.custom()
                .limitForPeriod(10)
                .limitRefreshPeriod(Duration.ofMinutes(1))
                .build()
        );

        return limiter.executeSupplier(() -> dataService.fetch(userId));
    }
}
```

### ❌ DON'T: Ignore rate limits

```java
// DON'T: No client-side rate limiting
public void callExternalApi() {
    // 1000 requests/second to API with 100 req/sec limit
    externalApi.call();  // Gets rate limited, errors pile up
}

// DON'T: Same limit for all operations
@RateLimiter(name = "global")  // Read and write same limit
public Data read() { }

@RateLimiter(name = "global")  // Admin gets same limit as user
public void adminOperation() { }
```

---

## 6. Health Checks and Monitoring

### ✅ DO: Implement comprehensive health checks

```java
// DO: Custom health indicator
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGateway gateway;

    @Override
    public Health health() {
        try {
            boolean isUp = gateway.ping();
            return isUp ? Health.up()
                .withDetail("gateway", "Available")
                .build()
                : Health.down()
                .withDetail("gateway", "Unavailable")
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

// DO: Circuit breaker health
@Component
@RequiredArgsConstructor
public class CircuitBreakerHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry registry;

    @Override
    public Health health() {
        Map<String, String> details = new HashMap<>();
        boolean allClosed = true;

        for (CircuitBreaker cb : registry.getAllCircuitBreakers()) {
            details.put(cb.getName(), cb.getState().name());
            if (cb.getState() == State.OPEN) {
                allClosed = false;
            }
        }

        return allClosed ? Health.up().withDetails(details).build()
            : Health.status("DEGRADED").withDetails(details).build();
    }
}

// DO: Expose metrics
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,retries
  health:
    circuitbreakers:
      enabled: true
```

**When to use:** All production services, external dependencies.
