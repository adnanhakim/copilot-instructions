# Streams & Lambdas Best Practices

## 1. Lambda Expression Clarity

### ✅ DO: Write clear, concise lambdas

```java
// DO: Use method references when possible
public List<String> getUsernames(List<User> users) {
    return users.stream()
                .map(User::getUsername)  // Method reference
                .toList();
}

// DO: Extract complex lambdas to named methods
public List<Order> getHighValueOrders(List<Order> orders) {
    return orders.stream()
                 .filter(this::isHighValue)  // Named method for clarity
                 .toList();
}

private boolean isHighValue(Order order) {
    return order.getTotal().compareTo(new BigDecimal("1000")) > 0
        && order.getStatus() == OrderStatus.CONFIRMED
        && order.getCustomer().isPremium();
}

// DO: Use var for complex lambda parameters (Java 10+)
public void processUsers(List<User> users) {
    users.forEach((var user) -> {
        // Type inference with ability to add annotations
        log.info("Processing: {}", user.getUsername());
        user.setLastProcessed(LocalDateTime.now());
    });
}

// DO: Single expression lambdas without braces
Predicate<String> notEmpty = s -> !s.isEmpty();
Function<User, String> toEmail = user -> user.getEmail();
```

**When to use:**
- Method references: When lambda just calls a method
- Named methods: Complex logic (>2 lines)
- Inline lambdas: Simple, single expressions

**When NOT to use:** Complex multi-line lambdas inline.

### ❌ DON'T: Write unclear or overly complex lambdas

```java
// DON'T: Complex inline lambda
public List<Order> getHighValueOrders(List<Order> orders) {
    return orders.stream()
                 .filter(order -> {
                     BigDecimal total = order.getTotal();
                     OrderStatus status = order.getStatus();
                     Customer customer = order.getCustomer();
                     boolean isHighValue = total.compareTo(new BigDecimal("1000")) > 0;
                     boolean isConfirmed = status == OrderStatus.CONFIRMED;
                     boolean isPremium = customer != null && customer.isPremium();
                     return isHighValue && isConfirmed && isPremium;
                 })
                 .toList();
}

// DON'T: Lambda instead of method reference
list.forEach(item -> System.out.println(item));  // Bad
list.forEach(System.out::println);  // Good

// DON'T: Unnecessarily verbose lambda
Function<String, Integer> toLength = (String s) -> { return s.length(); };  // Bad
Function<String, Integer> toLength = String::length;  // Good
```

**When this appears:** Inline business logic, unfamiliarity with method references.

**Why it's wrong:** Hard to read, test, and reuse.

---

## 2. Stream Pipeline Structure

### ✅ DO: Build readable, efficient pipelines

```java
// DO: Logical ordering of operations
public List<String> getTopCustomerNames(List<Order> orders, int limit) {
    return orders.stream()
                 .filter(order -> order.getStatus() == OrderStatus.COMPLETED)  // Filter early
                 .map(Order::getCustomer)  // Transform after filtering
                 .distinct()  // Remove duplicates
                 .sorted(Comparator.comparing(Customer::getTotalSpent).reversed())
                 .limit(limit)  // Limit late for efficiency
                 .map(Customer::getName)
                 .toList();
}

// DO: Use intermediate variables for readability
public BigDecimal calculateTotalRevenue(List<Order> orders) {
    var completedOrders = orders.stream()
                                .filter(order -> order.getStatus() == OrderStatus.COMPLETED);
    
    var revenue = completedOrders
                    .map(Order::getTotal)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    return revenue;
}

// DO: Use collectors for complex aggregations
public Map<String, List<Order>> groupOrdersByCustomer(List<Order> orders) {
    return orders.stream()
                 .collect(Collectors.groupingBy(
                     order -> order.getCustomer().getId()
                 ));
}

// DO: Custom collector for complex logic
public OrderStatistics calculateStatistics(List<Order> orders) {
    return orders.stream().collect(Collector.of(
        OrderStatistics::new,
        (stats, order) -> {
            stats.incrementCount();
            stats.addRevenue(order.getTotal());
            stats.updateAverage();
        },
        OrderStatistics::combine
    ));
}
```

**When to use:**
- Filter early: Reduce elements before expensive operations
- Limit late: After sorting but before final collection
- Collectors: Grouping, partitioning, summarizing

**When NOT to use:** Streams for simple iterations with side effects.

### ❌ DON'T: Build inefficient or confusing pipelines

```java
// DON'T: Mapping before filtering
public List<String> getActiveUsernames(List<User> users) {
    return users.stream()
                .map(User::getUsername)  // Maps all users first
                .filter(username -> isActive(username))  // Then filters (inefficient)
                .toList();
}

// Better: Filter first
public List<String> getActiveUsernames(List<User> users) {
    return users.stream()
                .filter(User::isActive)  // Filter first (fewer elements to map)
                .map(User::getUsername)
                .toList();
}

// DON'T: Multiple terminal operations
public void processOrders(List<Order> orders) {
    Stream<Order> stream = orders.stream().filter(Order::isPending);
    
    long count = stream.count();  // First terminal operation
    List<Order> list = stream.toList();  // IllegalStateException!
}

// Better: Collect once
public void processOrders(List<Order> orders) {
    List<Order> pending = orders.stream()
                                .filter(Order::isPending)
                                .toList();
    
    long count = pending.size();
    processList(pending);
}

// DON'T: Overly long single pipeline
public Result complexProcessing(List<Data> data) {
    return data.stream()
               .filter(d -> d.isValid())
               .map(d -> transform(d))
               .filter(d -> d.getScore() > 50)
               .sorted(Comparator.comparing(Data::getPriority))
               .map(d -> enrich(d))
               .filter(d -> d.getCategory().equals("A"))
               .map(d -> convert(d))
               .collect(Collectors.toList())
               .stream()  // Stream again! Bad sign
               .limit(10)
               .collect(Collectors.groupingBy(Data::getType))
               .entrySet().stream()  // Stream again!
               .map(entry -> process(entry))
               .collect(Collectors.toList());
}

// Better: Break into logical steps
public Result complexProcessing(List<Data> data) {
    var validData = filterAndTransformData(data);
    var enrichedData = enrichAndFilterByCategory(validData);
    var topResults = getTopResults(enrichedData, 10);
    return aggregateResults(topResults);
}
```

**When this appears:** Stream overuse, trying to do everything in one pipeline.

**Why it's wrong:** Hard to debug, understand, and maintain. Performance issues.

---

## 3. Primitive Streams

### ✅ DO: Use primitive streams for numeric operations

```java
// DO: IntStream for integer operations
public int calculateTotalQuantity(List<OrderItem> items) {
    return items.stream()
                .mapToInt(OrderItem::getQuantity)  // IntStream, no boxing
                .sum();
}

// DO: DoubleStream for decimal operations
public double calculateAveragePrice(List<Product> products) {
    return products.stream()
                   .mapToDouble(p -> p.getPrice().doubleValue())
                   .average()
                   .orElse(0.0);
}

// DO: LongStream for long operations
public long countActiveUsers(List<User> users) {
    return users.stream()
                .filter(User::isActive)
                .count();  // Returns primitive long
}

// DO: Range operations
public List<Integer> generateIds(int start, int end) {
    return IntStream.range(start, end)  // Exclusive end
                    .boxed()
                    .toList();
}

public void processInBatches(int totalItems, int batchSize) {
    IntStream.iterate(0, i -> i < totalItems, i -> i + batchSize)  // Java 9+
             .forEach(offset -> processBatch(offset, batchSize));
}

// DO: Primitive stream statistics
public IntSummaryStatistics getScoreStats(List<Student> students) {
    return students.stream()
                   .mapToInt(Student::getScore)
                   .summaryStatistics();  // min, max, avg, sum, count in one pass
}
```

**When to use:**
- IntStream: Integer arithmetic, ranges, indexing
- LongStream: Large numbers, counts
- DoubleStream: Decimal calculations (when BigDecimal not needed)

**When NOT to use:** When precision matters (use BigDecimal), when boxing overhead is negligible.

### ❌ DON'T: Use object streams for primitives

```java
// DON'T: Boxing overhead in numeric operations
public int sumScores(List<Student> students) {
    return students.stream()
                   .map(Student::getScore)  // Stream<Integer> - boxing
                   .reduce(0, Integer::sum);  // Unboxing/boxing on each operation
}

// Better:
public int sumScores(List<Student> students) {
    return students.stream()
                   .mapToInt(Student::getScore)  // IntStream - no boxing
                   .sum();
}

// DON'T: Multiple passes for statistics
public class ScoreAnalyzer {
    public void analyzeScores(List<Student> students) {
        int min = students.stream().mapToInt(Student::getScore).min().orElse(0);
        int max = students.stream().mapToInt(Student::getScore).max().orElse(0);
        double avg = students.stream().mapToInt(Student::getScore).average().orElse(0);
        long count = students.stream().mapToInt(Student::getScore).count();
    }
}

// Better: Single pass
public void analyzeScores(List<Student> students) {
    IntSummaryStatistics stats = students.stream()
                                         .mapToInt(Student::getScore)
                                         .summaryStatistics();
    
    int min = stats.getMin();
    int max = stats.getMax();
    double avg = stats.getAverage();
    long count = stats.getCount();
}

// DON'T: List.of() when IntStream is better
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
int sum = numbers.stream().mapToInt(Integer::intValue).sum();

// Better:
int sum = IntStream.rangeClosed(1, 10).sum();
```

**When this appears:** Unfamiliarity with primitive streams, copying patterns from object streams.

**Why it's wrong:** Boxing/unboxing overhead, especially in large datasets or hot paths.

---

## 4. Optional Best Practices

### ✅ DO: Use Optional appropriately

```java
// DO: Optional as return type for potentially absent values
public Optional<User> findUserById(String id) {
    return Optional.ofNullable(repository.findById(id));
}

// DO: Optional operations instead of isPresent() checks
public String getUsernameOrDefault(String userId) {
    return findUserById(userId)
            .map(User::getUsername)
            .orElse("Guest");
}

// DO: orElseThrow for required values
public User getUserOrThrow(String id) {
    return findUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
}

// DO: flatMap for nested Optionals
public Optional<String> getUserEmail(String userId) {
    return findUserById(userId)
            .flatMap(User::getEmail);  // User::getEmail returns Optional<String>
}

// DO: ifPresentOrElse (Java 9+)
public void processUser(String userId) {
    findUserById(userId)
        .ifPresentOrElse(
            user -> sendNotification(user),
            () -> log.warn("User not found: {}", userId)
        );
}

// DO: or() for alternative Optional (Java 9+)
public Optional<User> findUser(String id) {
    return findUserInCache(id)
            .or(() -> findUserInDatabase(id))
            .or(() -> findUserInBackup(id));
}

// DO: Stream from Optional (Java 9+)
public List<String> getActiveUserEmails(List<String> userIds) {
    return userIds.stream()
                  .map(this::findUserById)
                  .flatMap(Optional::stream)  // Converts Optional to Stream
                  .filter(User::isActive)
                  .map(User::getEmail)
                  .flatMap(Optional::stream)
                  .toList();
}
```

**When to use:**
- Method return types that may not have a value
- Chaining operations on potentially absent values
- Avoiding null checks

**When NOT to use:**
- Collections (use empty collections)
- Fields (use null or lazy initialization)
- Method parameters (use overloading or null checks)

### ❌ DON'T: Misuse Optional

```java
// DON'T: Optional fields
public class User {
    private Optional<String> middleName;  // DON'T - use String with null check
    
    public Optional<String> getMiddleName() {
        return middleName;  // Optional of Optional - confusing
    }
}

// Better:
public class User {
    private String middleName;  // Can be null
    
    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}

// DON'T: Optional parameters
public void updateUser(String id, Optional<String> email) {  // DON'T
    email.ifPresent(e -> setEmail(e));
}

// Better: Overloading or null parameter
public void updateUser(String id, String email) {
    if (email != null) {
        setEmail(email);
    }
}

// DON'T: isPresent() + get() pattern
public String getUsername(String userId) {
    Optional<User> user = findUserById(userId);
    if (user.isPresent()) {  // DON'T - defeats the purpose
        return user.get().getUsername();
    } else {
        return "Guest";
    }
}

// Better:
public String getUsername(String userId) {
    return findUserById(userId)
            .map(User::getUsername)
            .orElse("Guest");
}

// DON'T: Optional.get() without checking
public User getUser(String id) {
    return findUserById(id).get();  // NoSuchElementException if empty!
}

// Better:
public User getUser(String id) {
    return findUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
}

// DON'T: Optional.of(null)
public Optional<User> findUser(String id) {
    User user = repository.findById(id);
    return Optional.of(user);  // NullPointerException if user is null!
}

// Better:
public Optional<User> findUser(String id) {
    User user = repository.findById(id);
    return Optional.ofNullable(user);
}

// DON'T: Creating Optional just to check null
public void process(String value) {
    Optional.ofNullable(value).ifPresent(this::doSomething);  // Overkill
}

// Better:
public void process(String value) {
    if (value != null) {
        doSomething(value);
    }
}
```

**When this appears:** Optional overuse, treating Optional like a null wrapper.

**Why it's wrong:** Complexity without benefit, performance overhead, confusing semantics.

---

## 5. Parallel Streams

### ✅ DO: Use parallel streams wisely

```java
// DO: Parallel for CPU-intensive operations on large datasets
public List<ProcessedImage> processImages(List<Image> images) {
    if (images.size() < 100) {
        return images.stream()
                     .map(this::processImage)  // Sequential for small datasets
                     .toList();
    }
    
    return images.parallelStream()
                 .map(this::processImage)  // CPU-bound, benefits from parallelism
                 .toList();
}

// DO: Use ForkJoinPool for custom parallelism
public List<Result> processWithCustomPool(List<Data> data) {
    ForkJoinPool customPool = new ForkJoinPool(4);  // Custom thread count
    
    try {
        return customPool.submit(() ->
            data.parallelStream()
                .map(this::process)
                .toList()
        ).join();
    } finally {
        customPool.shutdown();
    }
}

// DO: Ensure thread safety in parallel operations
public Map<String, Long> countWordOccurrences(List<String> documents) {
    return documents.parallelStream()
                    .flatMap(doc -> Arrays.stream(doc.split("\\s+")))
                    .collect(Collectors.groupingByConcurrent(  // Thread-safe collector
                        Function.identity(),
                        Collectors.counting()
                    ));
}

// DO: Check if parallelism helps with benchmarking
@Benchmark
public List<Integer> sequentialProcessing() {
    return data.stream().map(this::transform).toList();
}

@Benchmark
public List<Integer> parallelProcessing() {
    return data.parallelStream().map(this::transform).toList();
}
```

**When to use:**
- Large datasets (>10k elements)
- CPU-intensive transformations
- Independent operations (no shared state)

**When NOT to use:**
- Small datasets (overhead > benefit)
- I/O-bound operations (use virtual threads instead)
- Operations with side effects or shared mutable state

### ❌ DON'T: Misuse parallel streams

```java
// DON'T: Parallel for I/O operations
public List<User> loadUsers(List<String> userIds) {
    return userIds.parallelStream()  // DON'T - I/O bound
                  .map(id -> repository.findById(id))  // Database call
                  .filter(Optional::isPresent)
                  .map(Optional::get)
                  .toList();
}

// Better: Use virtual threads or CompletableFuture
public List<User> loadUsers(List<String> userIds) {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        List<CompletableFuture<Optional<User>>> futures = userIds.stream()
            .map(id -> CompletableFuture.supplyAsync(
                () -> repository.findById(id), 
                executor
            ))
            .toList();
        
        return futures.stream()
                      .map(CompletableFuture::join)
                      .flatMap(Optional::stream)
                      .toList();
    }
}

// DON'T: Shared mutable state in parallel streams
private int count = 0;  // Shared mutable state

public void countElements(List<String> items) {
    items.parallelStream()
         .forEach(item -> count++);  // Race condition!
}

// Better: Use reduction
public long countElements(List<String> items) {
    return items.parallelStream().count();  // Or just items.size()
}

// DON'T: Parallel for small datasets
public List<Integer> doubleNumbers(List<Integer> numbers) {
    return numbers.parallelStream()  // Overhead > benefit for small lists
                  .map(n -> n * 2)
                  .toList();
}

// Better: Sequential for small datasets
public List<Integer> doubleNumbers(List<Integer> numbers) {
    return numbers.stream()
                  .map(n -> n * 2)
                  .toList();
}

// DON'T: Order-dependent operations in parallel
public List<String> processInOrder(List<String> items) {
    return items.parallelStream()
                .map(this::transform)
                .toList();  // Order not guaranteed!
}

// Better: Use forEachOrdered or sequential stream
public List<String> processInOrder(List<String> items) {
    return items.stream()  // Sequential maintains order
                .map(this::transform)
                .toList();
}
```

**When this appears:** "Parallel is always faster" mindset, unfamiliarity with parallel stream overhead.

**Why it's wrong:** Thread contention, race conditions, overhead exceeding benefits.

---

## 6. Custom Collectors

### ✅ DO: Create custom collectors for complex aggregations

```java
// DO: Custom collector for immutable results
public class ImmutableListCollector<T> implements Collector<T, List<T>, List<T>> {
    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new;
    }
    
    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add;
    }
    
    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }
    
    @Override
    public Function<List<T>, List<T>> finisher() {
        return List::copyOf;  // Immutable copy
    }
    
    @Override
    public Set<Characteristics> characteristics() {
        return Set.of();  // No characteristics
    }
}

// Usage
List<String> immutable = stream.collect(new ImmutableListCollector<>());

// DO: Use Collector.of() for simple custom collectors
public static <T> Collector<T, ?, String> joining(String delimiter, String prefix, String suffix) {
    return Collector.of(
        StringBuilder::new,  // Supplier
        (sb, element) -> {   // Accumulator
            if (sb.length() > 0) sb.append(delimiter);
            sb.append(element);
        },
        (sb1, sb2) -> {      // Combiner
            if (sb1.length() > 0 && sb2.length() > 0) {
                sb1.append(delimiter);
            }
            return sb1.append(sb2);
        },
        sb -> prefix + sb + suffix  // Finisher
    );
}

// DO: Downstream collectors
public Map<String, Long> countUsersByRole(List<User> users) {
    return users.stream()
                .collect(Collectors.groupingBy(
                    User::getRole,
                    Collectors.counting()  // Downstream collector
                ));
}

public Map<String, Set<String>> getUsernamesByRole(List<User> users) {
    return users.stream()
                .collect(Collectors.groupingBy(
                    User::getRole,
                    Collectors.mapping(      // Downstream collector
                        User::getUsername,
                        Collectors.toSet()   // Further downstream
                    )
                ));
}
```

**When to use:**
- Complex aggregation logic not covered by standard collectors
- Custom immutable collection types
- Performance-critical aggregations with specific requirements

**When NOT to use:** When standard collectors suffice.

### ❌ DON'T: Reinvent standard collectors

```java
// DON'T: Manual collection in forEach
public List<String> getActiveUsernames(List<User> users) {
    List<String> usernames = new ArrayList<>();
    users.stream()
         .filter(User::isActive)
         .forEach(user -> usernames.add(user.getUsername()));  // Side effect
    return usernames;
}

// Better: Use collectors
public List<String> getActiveUsernames(List<User> users) {
    return users.stream()
                .filter(User::isActive)
                .map(User::getUsername)
                .toList();
}

// DON'T: Manual grouping
public Map<String, List<User>> groupUsersByRole(List<User> users) {
    Map<String, List<User>> grouped = new HashMap<>();
    users.forEach(user -> {
        String role = user.getRole();
        grouped.computeIfAbsent(role, k -> new ArrayList<>()).add(user);
    });
    return grouped;
}

// Better: Use groupingBy
public Map<String, List<User>> groupUsersByRole(List<User> users) {
    return users.stream()
                .collect(Collectors.groupingBy(User::getRole));
}

// DON'T: Complex manual aggregation
public String joinUsernames(List<User> users) {
    StringBuilder sb = new StringBuilder();
    users.stream()
         .map(User::getUsername)
         .forEach(name -> {
             if (sb.length() > 0) {
                 sb.append(", ");
             }
             sb.append(name);
         });
    return sb.toString();
}

// Better: Use joining collector
public String joinUsernames(List<User> users) {
    return users.stream()
                .map(User::getUsername)
                .collect(Collectors.joining(", "));
}
```

**When this appears:** Unfamiliarity with built-in collectors, imperative programming habits.

**Why it's wrong:** Reinventing the wheel, less efficient, more error-prone.
