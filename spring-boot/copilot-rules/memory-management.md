# Memory Management Best Practices

## 1. Object Creation and Reuse

### ✅ DO: Minimize unnecessary object creation

```java
// DO: Reuse immutable objects
public class DateUtils {
    private static final DateTimeFormatter ISO_DATE_FORMATTER =
        DateTimeFormatter.ISO_LOCAL_DATE;  // Reuse, thread-safe

    public static String formatDate(LocalDate date) {
        return ISO_DATE_FORMATTER.format(date);  // No new formatter each time
    }
}

// DO: Use StringBuilder for string concatenation in loops
public String buildReport(List<String> items) {
    StringBuilder sb = new StringBuilder(items.size() * 50);  // Estimate capacity

    for (String item : items) {
        sb.append(item).append("\n");  // Efficient
    }

    return sb.toString();
}

// DO: String.format alternatives for simple concatenation
String message = "User %s logged in".formatted(username);  // Java 15+
String path = "/users/" + userId;  // Simple concatenation is fine

// DO: Object pooling for expensive objects (when appropriate)
@Configuration
public class DatabaseConfiguration {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(10);  // Connection pool
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}

// DO: Use primitives in hot paths
public class ScoreCalculator {
    public int calculateTotal(int[] scores) {  // Primitive array, no boxing
        int total = 0;  // Primitive, no object creation
        for (int score : scores) {
            total += score;  // No boxing
        }
        return total;
    }
}
```

**When to use:**

- Primitive types: Performance-critical code, large arrays
- StringBuilder: String concatenation in loops
- Object pools: Database connections, thread pools
- Static final: Immutable constants

**When NOT to use:** Premature optimization. Profile first.

### ❌ DON'T: Create unnecessary objects

```java
// DON'T: String concatenation in loops
public String buildReport(List<String> items) {
    String report = "";
    for (String item : items) {
        report += item + "\n";  // Creates new String each iteration!
    }
    return report;
}
// For 1000 items: 1000 intermediate String objects created

// Better: StringBuilder
public String buildReport(List<String> items) {
    StringBuilder sb = new StringBuilder();
    for (String item : items) {
        sb.append(item).append("\n");  // Modifies existing buffer
    }
    return sb.toString();
}

// DON'T: Create new formatter every time
public String formatDate(LocalDate date) {
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");  // New object
    return formatter.format(date);
}

// Better: Reuse formatter
private static final DateTimeFormatter DATE_FORMATTER =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");

public String formatDate(LocalDate date) {
    return DATE_FORMATTER.format(date);
}

// DON'T: Boxing in performance-critical code
public Integer sum(List<Integer> numbers) {  // Boxed
    Integer total = 0;  // Boxed
    for (Integer num : numbers) {  // Unbox
        total += num;  // Unbox, add, box
    }
    return total;
}

// Better: Primitives
public int sum(int[] numbers) {  // Primitive array
    int total = 0;  // Primitive
    for (int num : numbers) {
        total += num;  // No boxing/unboxing
    }
    return total;
}

// DON'T: Create objects in loops
public void processRecords(List<Record> records) {
    for (Record record : records) {
        DateTimeFormatter formatter = DateTimeFormatter.ISO_DATE_TIME;  // Created 1000x!
        String formatted = formatter.format(record.getTimestamp());
        // Process
    }
}

// Better: Create once
public void processRecords(List<Record> records) {
    DateTimeFormatter formatter = DateTimeFormatter.ISO_DATE_TIME;  // Once
    for (Record record : records) {
        String formatted = formatter.format(record.getTimestamp());
        // Process
    }
}
```

**When this appears:** Tight loops, lack of awareness, copying patterns without understanding.

**Why it's wrong:** Excessive garbage collection, memory pressure, poor performance.

---

## 2. Collections and Data Structures

### ✅ DO: Choose memory-efficient collections

```java
// DO: Right-size collections
public List<User> loadUsers(int expectedCount) {
    List<User> users = new ArrayList<>(expectedCount);  // Avoid resizing
    // Load users
    return users;
}

// DO: Use immutable collections to save memory
public class ConfigurationService {
    // These won't change after initialization
    private static final List<String> ALLOWED_ROLES =
        List.of("ADMIN", "USER", "GUEST");  // Compact, immutable

    private static final Set<String> SUPPORTED_CURRENCIES =
        Set.of("USD", "EUR", "GBP");

    private static final Map<String, String> ERROR_CODES = Map.of(
        "001", "Invalid input",
        "002", "Not found",
        "003", "Unauthorized"
    );
}

// DO: Clear collections when done
@Service
public class BatchProcessor {

    public void processBatch(List<Item> items) {
        List<Result> results = new ArrayList<>(items.size());

        for (Item item : items) {
            results.add(process(item));
        }

        sendResults(results);
        results.clear();  // Help GC, especially if this object is reused
    }
}

// DO: Use EnumMap/EnumSet for enum keys
public class StateMachine {
    private final Map<State, List<State>> transitions =
        new EnumMap<>(State.class);  // More efficient than HashMap for enums
}

// DO: Stream for one-time iteration of large datasets
public void processLargeFile(Path file) throws IOException {
    try (Stream<String> lines = Files.lines(file)) {
        lines.filter(line -> !line.isEmpty())
             .map(this::process)
             .forEach(this::save);
        // Lines not held in memory, processed one by one
    }
}
```

**When to use:**

- ArrayList with capacity: Known or estimated size
- Immutable collections: Constants, unmodifiable data
- EnumMap/EnumSet: Enum keys/values
- Streams: Large datasets, one-time iteration

**When NOT to use:** Over-optimization. Measure first.

### ❌ DON'T: Waste memory with collections

```java
// DON'T: Load entire dataset into memory
public List<User> getAllUsers() {
    return userRepository.findAll();  // Could be millions of users!
}

// Better: Stream or paginate
public Stream<User> streamAllUsers() {
    return userRepository.streamAll();  // Process one by one
}

public Page<User> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable);  // Load in chunks
}

// DON'T: No capacity hint when size is known
public List<String> generateIds(int count) {
    List<String> ids = new ArrayList<>();  // Starts at 10, grows to 15, 22, 33...
    for (int i = 0; i < count; i++) {
        ids.add(UUID.randomUUID().toString());
    }
    return ids;
}

// Better: Pre-size
public List<String> generateIds(int count) {
    List<String> ids = new ArrayList<>(count);  // Exactly right size
    for (int i = 0; i < count; i++) {
        ids.add(UUID.randomUUID().toString());
    }
    return ids;
}

// DON'T: Hold references to large collections unnecessarily
@Service
public class CacheService {
    private List<User> allUsers;  // Held in memory always!

    public void initialize() {
        allUsers = userRepository.findAll();  // Millions of users
    }

    public User findUser(String id) {
        return allUsers.stream()
                      .filter(u -> u.getId().equals(id))
                      .findFirst()
                      .orElse(null);
    }
}

// Better: Use proper cache or database queries
@Service
public class CacheService {

    @Cacheable("users")
    public User findUser(String id) {
        return userRepository.findById(id).orElse(null);
        // Caches individual users, not entire list
    }
}

// DON'T: Keep collections of deleted data
public class OrderProcessor {
    private final List<Order> processedOrders = new ArrayList<>();

    public void process(Order order) {
        // Process order
        processedOrders.add(order);  // Grows forever!
    }
}

// Better: Clear periodically or don't keep
public class OrderProcessor {
    public void process(Order order) {
        // Process order
        // Don't keep in memory
    }
}
```

**When this appears:** Loading entire tables, no pagination, unbounded collections.

**Why it's wrong:** OutOfMemoryError, GC pressure, poor scalability.

---

## 3. Resource Management

### ✅ DO: Close resources properly

```java
// DO: Try-with-resources for AutoCloseable
public String readFile(Path path) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        return reader.lines().collect(Collectors.joining("\n"));
    }  // Automatically closed
}

// DO: Multiple resources
public void copyFile(Path source, Path target) throws IOException {
    try (InputStream in = Files.newInputStream(source);
         OutputStream out = Files.newOutputStream(target)) {
        in.transferTo(out);
    }  // Both closed automatically, even if exception thrown
}

// DO: Custom AutoCloseable for resource cleanup
public class DatabaseSession implements AutoCloseable {
    private final Connection connection;

    public DatabaseSession() throws SQLException {
        this.connection = DriverManager.getConnection(dbUrl);
    }

    public void execute(String sql) throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            stmt.execute(sql);
        }
    }

    @Override
    public void close() throws SQLException {
        if (connection != null && !connection.isClosed()) {
            connection.close();
        }
    }
}

// Usage
try (DatabaseSession session = new DatabaseSession()) {
    session.execute("UPDATE users SET active = true");
}  // Connection closed automatically

// DO: @PreDestroy for Spring beans
@Component
public class CacheManager {
    private final Cache cache;

    @PreDestroy
    public void cleanup() {
        if (cache != null) {
            cache.clear();
            cache.close();
        }
    }
}
```

**When to use:**

- Try-with-resources: Always for AutoCloseable resources
- @PreDestroy: Spring bean cleanup
- Custom AutoCloseable: Resources needing cleanup

**When NOT to use:** Spring-managed beans don't need try-with-resources.

### ❌ DON'T: Leak resources

```java
// DON'T: Forget to close resources
public String readFile(Path path) throws IOException {
    BufferedReader reader = Files.newBufferedReader(path);
    return reader.lines().collect(Collectors.joining("\n"));
    // reader never closed - resource leak!
}

// DON'T: Close only on success path
public void processFile(Path path) throws IOException {
    InputStream in = Files.newInputStream(path);

    if (validate(in)) {
        process(in);
        in.close();  // Only closed if validate() returns true!
    }
    // If validate() returns false, resource leaked
}

// Better: Try-with-resources
public void processFile(Path path) throws IOException {
    try (InputStream in = Files.newInputStream(path)) {
        if (validate(in)) {
            process(in);
        }
    }  // Always closed
}

// DON'T: Store streams/readers as fields
@Service
public class FileProcessor {
    private BufferedReader reader;  // BAD - when is this closed?

    public void initialize(Path file) throws IOException {
        reader = Files.newBufferedReader(file);
    }

    public String readLine() throws IOException {
        return reader.readLine();
    }
}

// Better: Open and close in same scope
@Service
public class FileProcessor {

    public List<String> processFile(Path file) throws IOException {
        try (BufferedReader reader = Files.newBufferedReader(file)) {
            return reader.lines()
                        .map(this::processLine)
                        .toList();
        }
    }
}
```

**When this appears:** Manual resource management, forgetting cleanup.

**Why it's wrong:** Resource leaks, file handle exhaustion, memory leaks.

---

## 4. Caching Strategy

### ✅ DO: Cache strategically

```java
// DO: Spring Cache for expensive operations
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(String id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        // Clears entire product cache
    }
}

// DO: Caffeine cache for fine-grained control
@Configuration
public class CacheConfiguration {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("products", "users");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)  // Limit memory usage
            .expireAfterWrite(10, TimeUnit.MINUTES)  // Eviction policy
            .recordStats());  // Monitor cache performance
        return cacheManager;
    }
}

// DO: Local cache for frequently accessed data
@Service
public class ConfigService {
    private final LoadingCache<String, Configuration> cache;

    public ConfigService() {
        this.cache = Caffeine.newBuilder()
            .maximumSize(100)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .build(key -> loadConfiguration(key));  // Load on miss
    }

    public Configuration getConfig(String key) {
        return cache.get(key);  // Load from cache or source
    }
}

// DO: Weak references for memory-sensitive caches
public class ImageCache {
    private final Map<String, SoftReference<BufferedImage>> cache =
        new ConcurrentHashMap<>();

    public BufferedImage get(String key) {
        SoftReference<BufferedImage> ref = cache.get(key);
        if (ref != null) {
            BufferedImage image = ref.get();
            if (image != null) {
                return image;  // Still in memory
            }
        }
        // Load and cache
        BufferedImage image = loadImage(key);
        cache.put(key, new SoftReference<>(image));  // GC can reclaim if needed
        return image;
    }
}
```

**When to use:**

- @Cacheable: Expensive database queries, external API calls
- Caffeine: Fine-grained control, eviction policies
- SoftReference: Large objects, memory-sensitive caches
- Time-based eviction: Data that becomes stale

**When NOT to use:** Caching everything, data that changes frequently.

### ❌ DON'T: Cache incorrectly

```java
// DON'T: Unbounded cache
@Service
public class UserCache {
    private final Map<String, User> cache = new ConcurrentHashMap<>();

    public User getUser(String id) {
        return cache.computeIfAbsent(id, this::loadUser);
        // Grows forever! Memory leak
    }
}

// Better: Use cache with size limit
private final LoadingCache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(1000)  // Limit
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(this::loadUser);

// DON'T: Cache user-specific data globally
@Service
public class OrderService {

    @Cacheable("orders")  // BAD - cached across all users!
    public List<Order> getUserOrders(String userId) {
        return orderRepository.findByUserId(userId);
    }
}
// User A's orders might be returned to User B!

// Better: Include userId in cache key
@Cacheable(value = "orders", key = "#userId")
public List<Order> getUserOrders(String userId) {
    return orderRepository.findByUserId(userId);
}

// DON'T: Cache without eviction
@Service
public class ProductCache {
    private final Map<String, Product> cache = new HashMap<>();

    public Product getProduct(String id) {
        return cache.computeIfAbsent(id, productRepository::findById);
        // Never evicted - stale data forever
    }
}

// Better: Set expiration
@Cacheable(value = "products", key = "#id")
public Product getProduct(String id) {
    return productRepository.findById(id);
}

// Cache config with expiration
Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();

// DON'T: Cache mutable objects
@Cacheable("users")
public User getUser(String id) {
    return userRepository.findById(id);  // Entity with setters!
}

// Client code:
User user = userService.getUser("123");
user.setEmail("new@email.com");  // Modifies cached object!

// Better: Cache immutable DTOs
@Cacheable("users")
public UserDto getUser(String id) {
    User user = userRepository.findById(id);
    return UserDto.from(user);  // Immutable record
}
```

**When this appears:** No cache strategy, memory not considered, security oversight.

**Why it's wrong:** Memory leaks, stale data, security vulnerabilities.

---

## 5. Lazy Initialization

### ✅ DO: Lazy initialization when appropriate

```java
// DO: Lazy initialization for expensive objects
public class ReportGenerator {
    private HeavyResource resource;

    private HeavyResource getResource() {
        if (resource == null) {
            resource = new HeavyResource();  // Created only when needed
        }
        return resource;
    }

    public Report generate() {
        return getResource().createReport();
    }
}

// DO: Double-checked locking for thread-safe lazy init
public class ConfigurationManager {
    private volatile Configuration config;

    public Configuration getConfig() {
        if (config == null) {  // First check (no lock)
            synchronized (this) {
                if (config == null) {  // Second check (with lock)
                    config = loadConfiguration();
                }
            }
        }
        return config;
    }
}

// DO: Lazy initialization holder class (best for singletons)
public class DatabaseConnection {

    private DatabaseConnection() {}

    private static class Holder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE;  // Lazy, thread-safe, efficient
    }
}

// DO: Spring @Lazy for beans
@Service
@Lazy  // Not created until first use
public class HeavyService {

    public HeavyService() {
        // Expensive initialization
    }
}
```

**When to use:**

- Expensive objects not always needed
- Singletons (initialization holder pattern)
- Optional dependencies
- Performance optimization

**When NOT to use:** Simple objects, always-needed dependencies.

### ❌ DON'T: Misuse lazy initialization

```java
// DON'T: Broken lazy init in multi-threaded environment
public class ResourceManager {
    private HeavyResource resource;

    public HeavyResource getResource() {
        if (resource == null) {  // Race condition!
            resource = new HeavyResource();  // Could be created multiple times
        }
        return resource;
    }
}

// Better: Use synchronization or holder pattern

// DON'T: Lazy init for Spring beans manually
@Service
public class UserService {
    private EmailService emailService;  // Lazy field

    private EmailService getEmailService() {
        if (emailService == null) {
            emailService = new EmailService();  // DON'T - bypasses Spring!
        }
        return emailService;
    }
}

// Better: Use Spring @Lazy
@Service
public class UserService {
    private final EmailService emailService;

    public UserService(@Lazy EmailService emailService) {
        this.emailService = emailService;  // Spring manages lazy init
    }
}

// DON'T: Over-complicate simple cases
public class SimpleService {
    private String message;

    public String getMessage() {
        if (message == null) {  // Unnecessary lazy init
            message = "Hello";  // Simple String
        }
        return message;
    }
}

// Better: Just initialize
public class SimpleService {
    private final String message = "Hello";

    public String getMessage() {
        return message;
    }
}
```

**When this appears:** Premature optimization, threading issues.

**Why it's wrong:** Thread-safety issues, unnecessary complexity, bypasses framework features.

---

## 6. Virtual Threads (Java 21) - Memory Benefits

### ✅ DO: Use virtual threads to reduce memory per connection

```java
// DO: Virtual threads for high-concurrency with low memory footprint
// Platform threads: ~1MB stack each (1000 threads = ~1GB memory)
// Virtual threads: ~KB each (1,000,000+ threads possible)

@Configuration
public class VirtualThreadConfiguration {

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}

// DO: Virtual threads for batch processing
@Service
public class BulkEmailService {

    public void sendBulkEmails(List<Email> emails) {
        // Instead of thread pool limiting concurrency
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<Void>> futures = emails.stream()
                .map(email -> executor.submit(() -> {
                    sendEmail(email);
                    return null;
                }))
                .toList();

            // Wait for all
            for (Future<Void> f : futures) {
                try { f.get(); } catch (Exception e) { log.warn("Email failed", e); }
            }
        }
        // Memory: ~KB per email vs ~1MB per platform thread
    }
}

// DO: Spring Boot 3.2+ native virtual thread support
# application.yml
spring:
  threads:
    virtual:
      enabled: true  # Enables virtual threads for web requests
```

**Memory comparison:**
| Scenario | Platform Threads | Virtual Threads |
|----------|------------------|-----------------|
| 1,000 concurrent requests | ~1 GB | ~10 MB |
| 10,000 concurrent requests | ~10 GB (impossible) | ~100 MB |

### ❌ DON'T: Use virtual threads for CPU-bound work

```java
// DON'T: Virtual threads for computation
public List<Result> compute(List<Data> items) {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        return items.stream()
            .map(item -> executor.submit(() -> heavyCpuCalculation(item)))
            .map(Future::join)
            .toList();
    }
    // Virtual threads don't help CPU-bound work - no memory benefit
}

// Better: Use parallel streams or ForkJoinPool for CPU work
public List<Result> compute(List<Data> items) {
    return items.parallelStream()
        .map(this::heavyCpuCalculation)
        .toList();
}
```

---

## 7. JVM Tuning Tips

### ✅ DO: Configure JVM for memory efficiency

```bash
# String deduplication for apps with many duplicate strings
java -XX:+UseStringDeduplication -jar app.jar

# Compact strings (enabled by default in Java 9+)
java -XX:+CompactStrings -jar app.jar

# G1GC for large heaps (default in Java 11+)
java -XX:+UseG1GC -Xmx4g -jar app.jar

# ZGC for low-latency (Java 21+)
java -XX:+UseZGC -Xmx4g -jar app.jar
```

```java
// DO: Use trimToSize() for finalized ArrayLists
public List<User> loadUsers() {
    ArrayList<User> users = new ArrayList<>(1000);
    // Load users...
    users.trimToSize();  // Release unused capacity
    return Collections.unmodifiableList(users);
}

// DO: Use records for value objects (more GC-friendly)
// Records have compact memory layout and simpler object graph
public record Point(int x, int y) {}  // Smaller than class equivalent
public record Money(BigDecimal amount, String currency) {}
```
