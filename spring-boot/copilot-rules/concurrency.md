# Concurrency Best Practices

## 1. Virtual Threads (Java 21)

### ✅ DO: Use virtual threads for I/O-bound operations

```java
// DO: Virtual threads for concurrent I/O
@Service
@RequiredArgsConstructor
public class NotificationService {
    private final EmailClient emailClient;

    public void sendBulkNotifications(List<User> users) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<Void>> futures = users.stream()
                .map(user -> executor.submit(() -> {
                    emailClient.send(user.getEmail(), "Notification");
                    return null;
                }))
                .toList();

            // Wait for all to complete
            for (Future<Void> future : futures) {
                try {
                    future.get();
                } catch (ExecutionException e) {
                    log.error("Notification failed", e.getCause());
                }
            }
        }
    }
}

// DO: Virtual threads with Spring's @Async
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}

@Service
public class AsyncService {

    @Async
    public CompletableFuture<Result> processAsync(Data data) {
        // Runs on virtual thread
        return CompletableFuture.completedFuture(process(data));
    }
}

// DO: Virtual threads for HTTP clients
public class HttpClientService {
    private final HttpClient client = HttpClient.newBuilder()
        .executor(Executors.newVirtualThreadPerTaskExecutor())
        .build();

    public List<String> fetchAll(List<String> urls) throws Exception {
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> client.sendAsync(
                HttpRequest.newBuilder().uri(URI.create(url)).build(),
                HttpResponse.BodyHandlers.ofString()
            ))
            .map(cf -> cf.thenApply(HttpResponse::body))
            .toList();

        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
}
```

**When to use:**

- I/O-bound operations (HTTP calls, database queries, file I/O)
- High concurrency with many blocking operations
- Replacing thread pools for simple concurrent tasks

**When NOT to use:**

- CPU-bound operations (use parallel streams or ForkJoinPool)
- Operations requiring thread-local state (use Scoped Values)

### ❌ DON'T: Misuse virtual threads

```java
// DON'T: Virtual threads for CPU-bound work
public List<Integer> computeIntensive(List<Data> data) {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        return data.stream()
            .map(d -> executor.submit(() -> heavyCpuComputation(d)))
            .map(Future::join)
            .toList();
    }
    // Virtual threads don't help CPU-bound work!
}

// Better: Use parallel streams for CPU-bound
public List<Integer> computeIntensive(List<Data> data) {
    return data.parallelStream()
        .map(this::heavyCpuComputation)
        .toList();
}

// DON'T: Synchronized blocks in virtual threads (pinning)
public class BadVirtualThreadUsage {
    private final Object lock = new Object();

    public void process() {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                synchronized (lock) {  // BAD: Pins virtual thread to carrier
                    doBlockingIO();
                }
            });
        }
    }
}

// Better: Use ReentrantLock
public class GoodVirtualThreadUsage {
    private final ReentrantLock lock = new ReentrantLock();

    public void process() {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                lock.lock();
                try {
                    doBlockingIO();
                } finally {
                    lock.unlock();
                }
            });
        }
    }
}

// DON'T: ThreadLocal with virtual threads
private static final ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();

// Better: Use Scoped Values (Java 21)
private static final ScopedValue<Connection> CONNECTION = ScopedValue.newInstance();
```

**Why it's wrong:** Virtual threads are designed for I/O, not CPU work. Synchronized blocks pin virtual threads to carrier threads, negating their benefits.

---

## 2. Scoped Values (Java 21)

### ✅ DO: Use Scoped Values for request context

```java
// DO: Define scoped values for request context
public class RequestContext {
    public static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
    public static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();
}

// DO: Bind scoped values in filter
@Component
public class RequestContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String userId = extractUserId(httpRequest);
        String traceId = httpRequest.getHeader("X-Trace-Id");

        ScopedValue.where(RequestContext.USER_ID, userId)
            .where(RequestContext.TRACE_ID, traceId)
            .run(() -> {
                try {
                    chain.doFilter(request, response);
                } catch (IOException | ServletException e) {
                    throw new RuntimeException(e);
                }
            });
    }
}

// DO: Access scoped values in service
@Service
public class AuditService {

    public void logAction(String action) {
        String userId = RequestContext.USER_ID.orElse("anonymous");
        String traceId = RequestContext.TRACE_ID.orElse("unknown");
        log.info("[{}] User {} performed: {}", traceId, userId, action);
    }
}

// DO: Pass scoped values to child virtual threads
public class ParallelProcessor {

    public List<Result> processInParallel(List<Data> items) {
        String userId = RequestContext.USER_ID.get();
        String traceId = RequestContext.TRACE_ID.get();

        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            return items.stream()
                .map(item -> executor.submit(() ->
                    ScopedValue.where(RequestContext.USER_ID, userId)
                        .where(RequestContext.TRACE_ID, traceId)
                        .call(() -> processItem(item))
                ))
                .map(Future::join)
                .toList();
        }
    }
}
```

**When to use:**

- Request context (user ID, trace ID, tenant ID)
- Replacing ThreadLocal in virtual thread environments
- Immutable context that flows through call stack

**When NOT to use:**

- Mutable state (use explicit parameters)
- Long-lived context (use dependency injection)

### ❌ DON'T: Use ThreadLocal with virtual threads

```java
// DON'T: ThreadLocal with virtual threads
public class UserContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();

    public static void setUserId(String id) {
        userId.set(id);  // Stored on carrier thread, not virtual thread!
    }
}

// DON'T: Forget to clean up ThreadLocal
public void processRequest(String userId) {
    UserContext.setUserId(userId);
    try {
        doWork();
    } finally {
        // Forgot to remove - memory leak!
    }
}

// Better: Scoped Values are automatically cleaned up
ScopedValue.where(USER_ID, userId).run(() -> doWork());
```

---

## 3. Thread Safety

### ✅ DO: Use concurrent collections and atomic operations

```java
// DO: ConcurrentHashMap for thread-safe caching
@Service
public class CacheService {
    private final Map<String, Product> cache = new ConcurrentHashMap<>();

    public Product getOrLoad(String id) {
        return cache.computeIfAbsent(id, this::loadFromDatabase);
    }

    public void invalidate(String id) {
        cache.remove(id);
    }
}

// DO: AtomicReference for lock-free updates
public class Counter {
    private final AtomicLong count = new AtomicLong(0);
    private final AtomicReference<Instant> lastUpdate = new AtomicReference<>(Instant.now());

    public void increment() {
        count.incrementAndGet();
        lastUpdate.set(Instant.now());
    }

    public long getCount() {
        return count.get();
    }
}

// DO: CopyOnWriteArrayList for rare writes, frequent reads
@Component
public class EventListenerRegistry {
    private final List<EventListener> listeners = new CopyOnWriteArrayList<>();

    public void register(EventListener listener) {
        listeners.add(listener);  // Rare
    }

    public void notifyAll(Event event) {
        listeners.forEach(l -> l.onEvent(event));  // Frequent, no sync needed
    }
}

// DO: BlockingQueue for producer-consumer
@Service
public class TaskQueue {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

    public void submit(Task task) throws InterruptedException {
        queue.put(task);  // Blocks if full
    }

    @Scheduled(fixedDelay = 100)
    public void processQueue() throws InterruptedException {
        Task task = queue.poll(100, TimeUnit.MILLISECONDS);
        if (task != null) {
            process(task);
        }
    }
}
```

**When to use:**

- ConcurrentHashMap: Thread-safe maps, caches
- AtomicXxx: Counters, flags, single-value updates
- CopyOnWriteArrayList: Listeners, rarely modified lists
- BlockingQueue: Producer-consumer patterns

### ❌ DON'T: Use manual synchronization incorrectly

```java
// DON'T: Shared mutable state without synchronization
@Service
public class UnsafeCounter {
    private int count = 0;  // Not thread-safe!

    public void increment() {
        count++;  // Race condition
    }
}

// DON'T: Double-checked locking without volatile
public class BrokenSingleton {
    private static BrokenSingleton instance;

    public static BrokenSingleton getInstance() {
        if (instance == null) {  // First check (no sync)
            synchronized (BrokenSingleton.class) {
                if (instance == null) {  // Second check
                    instance = new BrokenSingleton();  // Might be partially constructed!
                }
            }
        }
        return instance;
    }
}

// Better: Use volatile or initialization holder
private static volatile Singleton instance;  // volatile fixes it

// Or use holder pattern
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}
public static Singleton getInstance() {
    return Holder.INSTANCE;
}

// DON'T: Lock on String or boxed primitives
synchronized ("lock") {  // String interning causes issues
    doWork();
}

synchronized (Integer.valueOf(1)) {  // Integer cache causes issues
    doWork();
}

// Better: Use dedicated lock object
private final Object lock = new Object();
synchronized (lock) {
    doWork();
}
```

---

## 4. CompletableFuture Patterns

### ✅ DO: Chain async operations effectively

```java
// DO: Chain transformations
public CompletableFuture<OrderSummary> processOrder(String orderId) {
    return fetchOrder(orderId)
        .thenApply(this::validateOrder)
        .thenCompose(this::processPayment)  // Returns another CompletableFuture
        .thenApply(this::createSummary)
        .exceptionally(this::handleError);
}

// DO: Combine multiple futures
public CompletableFuture<Dashboard> loadDashboard(String userId) {
    CompletableFuture<User> userFuture = fetchUser(userId);
    CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);
    CompletableFuture<Stats> statsFuture = fetchStats(userId);

    return CompletableFuture.allOf(userFuture, ordersFuture, statsFuture)
        .thenApply(v -> new Dashboard(
            userFuture.join(),
            ordersFuture.join(),
            statsFuture.join()
        ));
}

// DO: Use virtual threads with CompletableFuture
public CompletableFuture<Result> processAsync(Data data) {
    return CompletableFuture.supplyAsync(
        () -> expensiveIoOperation(data),
        Executors.newVirtualThreadPerTaskExecutor()
    );
}

// DO: Timeout handling
public CompletableFuture<Result> fetchWithTimeout(String id) {
    return fetchData(id)
        .orTimeout(5, TimeUnit.SECONDS)
        .exceptionally(ex -> {
            if (ex.getCause() instanceof TimeoutException) {
                return Result.timeout();
            }
            throw new CompletionException(ex);
        });
}
```

### ❌ DON'T: Block unnecessarily or ignore exceptions

```java
// DON'T: Block immediately after creating future
public Result processSync(Data data) {
    return processAsync(data).join();  // Defeats the purpose of async!
}

// DON'T: Ignore exceptions
CompletableFuture<Result> future = processAsync(data);
future.thenAccept(this::handleResult);  // Exception silently swallowed!

// Better: Always handle exceptions
future.thenAccept(this::handleResult)
      .exceptionally(ex -> {
          log.error("Processing failed", ex);
          return null;
      });

// DON'T: Use common ForkJoinPool for blocking operations
CompletableFuture.supplyAsync(() -> {
    return blockingDatabaseCall();  // Starves common pool!
});

// Better: Use dedicated executor
CompletableFuture.supplyAsync(
    () -> blockingDatabaseCall(),
    Executors.newVirtualThreadPerTaskExecutor()
);
```

---

## 5. Spring Async Configuration

### ✅ DO: Configure async properly with virtual threads

```java
// DO: Configure virtual thread executor
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method {} failed", method.getName(), ex);
        };
    }
}

// DO: Use @Async with return types
@Service
public class EmailService {

    @Async
    public CompletableFuture<SendResult> sendEmailAsync(Email email) {
        SendResult result = emailClient.send(email);
        return CompletableFuture.completedFuture(result);
    }

    @Async
    public void sendNotification(String userId, String message) {
        // Fire and forget, but exceptions are logged
        notificationClient.send(userId, message);
    }
}

// DO: Name executors for different purposes
@Configuration
@EnableAsync
public class AsyncConfiguration {

    @Bean("ioExecutor")
    public Executor ioExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    @Bean("cpuExecutor")
    public Executor cpuExecutor() {
        return Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    }
}

@Service
public class ProcessingService {

    @Async("ioExecutor")
    public CompletableFuture<Data> fetchData(String id) {
        return CompletableFuture.completedFuture(client.fetch(id));
    }

    @Async("cpuExecutor")
    public CompletableFuture<Result> computeResult(Data data) {
        return CompletableFuture.completedFuture(heavyComputation(data));
    }
}
```

### ❌ DON'T: Misconfigure Spring async

```java
// DON'T: Call @Async method from same class
@Service
public class BrokenAsyncService {

    public void process() {
        asyncMethod();  // NOT async! Proxy not invoked
    }

    @Async
    public void asyncMethod() {
        // This won't run asynchronously when called from process()
    }
}

// Better: Inject self or use separate class
@Service
@RequiredArgsConstructor
public class FixedAsyncService {
    private final AsyncWorker asyncWorker;  // Separate bean

    public void process() {
        asyncWorker.asyncMethod();  // Correctly invokes proxy
    }
}

// DON'T: Unbounded thread pool with no monitoring
@Bean
public Executor taskExecutor() {
    return Executors.newCachedThreadPool();  // Unbounded, dangerous
}

// Better: Use virtual threads or bounded pool
@Bean
public Executor taskExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();  // Virtual threads
    // Or use ThreadPoolTaskExecutor with bounds
}
```
