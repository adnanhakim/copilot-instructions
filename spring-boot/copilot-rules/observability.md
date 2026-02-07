# Observability Best Practices

## 1. Structured Logging

### ✅ DO: Use structured logging with SLF4J and Logback

```java
// DO: Structured logging with MDC
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain chain) throws ServletException, IOException {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }

        MDC.put("traceId", traceId);
        MDC.put("userId", extractUserId(request));
        MDC.put("path", request.getRequestURI());
        MDC.put("method", request.getMethod());

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// DO: Use parameterized logging
@Service
@Slf4j
@RequiredArgsConstructor
public class OrderService {

    public Order createOrder(CreateOrderRequest request) {
        log.info("Creating order for customer: {}", request.customerId());

        try {
            Order order = processOrder(request);
            log.info("Order created successfully: orderId={}, amount={}",
                order.getId(), order.getTotalAmount());
            return order;
        } catch (InsufficientInventoryException e) {
            log.warn("Order failed due to insufficient inventory: productId={}, requested={}, available={}",
                e.getProductId(), e.getRequestedQuantity(), e.getAvailableQuantity());
            throw e;
        } catch (Exception e) {
            log.error("Unexpected error creating order for customer: {}",
                request.customerId(), e);
            throw e;
        }
    }
}

// DO: Log important business events
@Service
@Slf4j
public class PaymentService {

    public PaymentResult processPayment(Payment payment) {
        log.info("Processing payment: paymentId={}, amount={}, currency={}",
            payment.getId(), payment.getAmount(), payment.getCurrency());

        Instant start = Instant.now();
        PaymentResult result = paymentGateway.process(payment);
        Duration duration = Duration.between(start, Instant.now());

        if (result.isSuccess()) {
            log.info("Payment successful: paymentId={}, transactionId={}, durationMs={}",
                payment.getId(), result.getTransactionId(), duration.toMillis());
        } else {
            log.warn("Payment failed: paymentId={}, reason={}, durationMs={}",
                payment.getId(), result.getFailureReason(), duration.toMillis());
        }

        return result;
    }
}
```

**logback-spring.xml with JSON format:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProfile name="prod">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>path</includeMdcKeyName>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>

        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

### ❌ DON'T: Poor logging practices

```java
// DON'T: String concatenation in logging
log.debug("Processing order for customer " + customerId + " with amount " + amount);

// Better: Parameterized logging (lazy evaluation)
log.debug("Processing order for customer {} with amount {}", customerId, amount);

// DON'T: Log sensitive data
log.info("User login: username={}, password={}", username, password);  // NEVER!
log.debug("Payment: cardNumber={}", cardNumber);  // NEVER!

// DON'T: Catch and log without context
try {
    process();
} catch (Exception e) {
    log.error("Error");  // Useless!
    log.error(e.getMessage());  // Missing stack trace!
}

// Better: Include context and exception
try {
    process();
} catch (Exception e) {
    log.error("Failed to process order: orderId={}", orderId, e);
}

// DON'T: Log at wrong level
log.info("Entering method processOrder");  // Too verbose for INFO
log.error("User not found");  // Not an error, use WARN

// DON'T: Excessive logging in loops
for (Item item : items) {
    log.debug("Processing item: {}", item);  // Thousands of log lines!
}

// Better: Summary logging
log.debug("Processing {} items", items.size());
```

---

## 2. Metrics with Micrometer

### ✅ DO: Collect meaningful metrics

```java
// DO: Custom business metrics
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final Counter ordersCreatedCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderAmountSummary;

    public OrderService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        this.ordersCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(meterRegistry);

        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time to process orders")
            .publishPercentileHistogram()
            .register(meterRegistry);

        this.orderAmountSummary = DistributionSummary.builder("orders.amount")
            .description("Order amounts")
            .baseUnit("dollars")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);

            ordersCreatedCounter.increment();
            orderAmountSummary.record(order.getTotalAmount().doubleValue());

            // Tag-based counters for different categories
            Counter.builder("orders.by.category")
                .tag("category", order.getCategory())
                .register(meterRegistry)
                .increment();

            return order;
        });
    }
}

// DO: Gauge for current state
@Component
@RequiredArgsConstructor
public class QueueMetrics {

    private final BlockingQueue<Task> taskQueue;

    @PostConstruct
    public void registerMetrics(MeterRegistry registry) {
        Gauge.builder("queue.size", taskQueue, Queue::size)
            .description("Current queue size")
            .tag("queue", "task-queue")
            .register(registry);
    }
}

// DO: Database connection pool metrics
@Configuration
public class DataSourceMetricsConfig {

    @Bean
    public HikariDataSource dataSource(MeterRegistry registry, DataSourceProperties props) {
        HikariDataSource dataSource = props.initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();

        dataSource.setMetricRegistry(registry);
        return dataSource;
    }
}

// DO: Cache metrics
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(MeterRegistry registry) {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .recordStats());

        // Bind cache stats to Micrometer
        return cacheManager;
    }
}
```

**application.yml for metrics:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
```

### ❌ DON'T: Metric anti-patterns

```java
// DON'T: High cardinality tags
Counter.builder("requests")
    .tag("userId", userId)  // Millions of unique values!
    .tag("requestId", requestId)  // Every request is unique!
    .register(meterRegistry)
    .increment();

// DON'T: Create new meters inside loops
for (Order order : orders) {
    Counter.builder("orders.processed")
        .register(meterRegistry)  // Creates duplicate meter!
        .increment();
}

// Better: Reuse meter or use tags appropriately
private final Counter ordersProcessed = Counter.builder("orders.processed")
    .register(meterRegistry);

for (Order order : orders) {
    ordersProcessed.increment();
}
```

---

## 3. Health Checks

### ✅ DO: Implement custom health indicators

```java
// DO: Custom health indicator for external dependencies
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient paymentGateway;

    public PaymentGatewayHealthIndicator(PaymentGatewayClient paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    @Override
    public Health health() {
        try {
            boolean healthy = paymentGateway.healthCheck();
            if (healthy) {
                return Health.up()
                    .withDetail("provider", "stripe")
                    .withDetail("responseTime", "50ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("error", "Health check returned false")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("provider", "stripe")
                .build();
        }
    }
}

// DO: Readiness vs Liveness probes
@Component
public class DatabaseReadinessIndicator implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up().build();
            }
        } catch (SQLException e) {
            return Health.down(e).build();
        }
        return Health.down().withDetail("reason", "Connection invalid").build();
    }
}

// DO: Health groups for different probes
// application.yml
management:
  endpoint:
    health:
      show-details: when_authorized
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis, paymentGateway
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

---

## 4. Distributed Tracing

### ✅ DO: Implement distributed tracing with Micrometer Tracing

```java
// DO: Configure tracing with Micrometer + OpenTelemetry
// build.gradle
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # Sample 100% in dev, lower in prod
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces

// DO: Custom spans for important operations
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;

    public Order processOrder(CreateOrderRequest request) {
        Span span = tracer.nextSpan().name("processOrder");
        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            span.tag("customerId", request.customerId());
            span.tag("itemCount", String.valueOf(request.items().size()));

            Order order = createOrder(request);

            span.tag("orderId", order.getId());
            span.tag("totalAmount", order.getTotalAmount().toString());

            return order;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}

// DO: Use @Observed for automatic spans
@Service
@Observed(name = "inventory-service")
public class InventoryService {

    @Observed(name = "check-availability")
    public boolean checkAvailability(String productId, int quantity) {
        return inventoryRepository.getAvailableQuantity(productId) >= quantity;
    }
}

// DO: Propagate trace context to async operations
@Service
@RequiredArgsConstructor
public class AsyncOrderProcessor {

    private final Tracer tracer;

    public void processAsync(Order order) {
        Span currentSpan = tracer.currentSpan();

        CompletableFuture.runAsync(() -> {
            try (Tracer.SpanInScope ws = tracer.withSpan(currentSpan)) {
                // Trace context preserved in async thread
                processOrderAsync(order);
            }
        });
    }
}
```

---

## 5. Actuator Configuration

### ✅ DO: Configure Actuator endpoints securely

```java
// DO: Secure actuator endpoints
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/liveness").permitAll()
                .requestMatchers("/actuator/health/readiness").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .requestMatchers("/actuator/prometheus").permitAll()  // For Prometheus scraping
                .requestMatchers("/actuator/**").hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

**application.yml:**

```yaml
management:
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health, info, metrics, prometheus, env, loggers
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
    env:
      show-values: when_authorized
  info:
    env:
      enabled: true
    git:
      mode: full

# Kubernetes probes
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  shutdown: graceful

# Custom info
info:
  app:
    name: ${spring.application.name}
    version: ${app.version:unknown}
    description: Order Processing Service
```

---

## 6. Alerting Guidelines

### ✅ DO: Define SLIs and SLOs

```yaml
# Prometheus alerting rules example
groups:
  - name: order-service-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: High error rate detected
          description: Error rate is {{ $value | humanizePercentage }}

      # Slow response time
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) 
          by (le)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: P95 latency above 1 second

      # Database connection pool exhaustion
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Database connection pool nearly exhausted
```

```java
// DO: Define SLI/SLO in code comments
/**
 * Order creation endpoint
 *
 * SLO Targets:
 * - Availability: 99.9%
 * - Latency P99: < 500ms
 * - Error rate: < 0.1%
 */
@PostMapping
public ResponseEntity<Order> createOrder(@Valid @RequestBody CreateOrderRequest request) {
    // Implementation
}
```
