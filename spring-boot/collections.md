# Collections Best Practices

## 1. Choose the Right Collection Type

### ✅ DO: Use appropriate collection for the use case

```java
// DO: ArrayList for random access and iteration
@Service
public class ProductService {
    public List<Product> getActiveProducts() {
        // ArrayList: good for indexed access, iteration
        return new ArrayList<>(repository.findByActiveTrue());
    }
}

// DO: LinkedHashSet for ordered uniqueness
public class OrderProcessor {
    public Set<String> getProcessingSteps() {
        Set<String> steps = new LinkedHashSet<>();  // Maintains insertion order
        steps.add("validation");
        steps.add("payment");
        steps.add("fulfillment");
        return steps;
    }
}

// DO: HashMap for key-value lookups
public class ProductCache {
    private final Map<String, Product> cache = new HashMap<>();
    
    public Optional<Product> findById(String id) {
        return Optional.ofNullable(cache.get(id));  // O(1) lookup
    }
}

// DO: EnumMap for enum keys (memory efficient)
public class OrderStateMachine {
    private final Map<OrderStatus, List<OrderStatus>> transitions = 
        new EnumMap<>(OrderStatus.class);
    
    public OrderStateMachine() {
        transitions.put(OrderStatus.PENDING, List.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED));
        transitions.put(OrderStatus.CONFIRMED, List.of(OrderStatus.SHIPPED));
    }
}
```

**When to use:**
- **ArrayList**: Default list choice, random access, frequent iteration
- **LinkedList**: Frequent insertions/deletions at beginning/middle (rare in practice)
- **HashSet**: Uniqueness, no order needed, fast contains()
- **LinkedHashSet**: Uniqueness + insertion order
- **TreeSet**: Uniqueness + natural ordering
- **HashMap**: Default map choice, key-value lookup
- **LinkedHashMap**: Key-value + insertion order (e.g., LRU cache)
- **TreeMap**: Key-value + sorted keys
- **EnumMap**: Enum keys (fastest, most memory efficient)

**When NOT to use:** Wrong collection for the access pattern (e.g., LinkedList for random access).

### ❌ DON'T: Use wrong collection type

```java
// DON'T: LinkedList for random access
public class ReportService {
    public String generateReport() {
        List<String> lines = new LinkedList<>();  // BAD: O(n) for get(i)
        
        for (int i = 0; i < 1000; i++) {
            lines.add("Line " + i);
        }
        
        // This is O(n²) with LinkedList!
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < lines.size(); i++) {
            sb.append(lines.get(i));  // O(n) for each get()
        }
        return sb.toString();
    }
}

// DON'T: ArrayList for frequent middle insertions
public class EventQueue {
    private final List<Event> events = new ArrayList<>();
    
    public void addPriorityEvent(Event event) {
        events.add(0, event);  // O(n) - shifts all elements
    }
}

// DON'T: Using synchronized collections in modern code
public class UserCache {
    private final Map<String, User> cache = new Hashtable<>();  // Legacy, use ConcurrentHashMap
    private final List<User> users = new Vector<>();             // Legacy, use ArrayList
}
```

**When this appears:** Copy-paste code, lack of understanding collection performance.

**Why it's wrong:** Performance degradation, especially at scale.

---

## 2. Immutable Collections (Java 21)

### ✅ DO: Use immutable collections when data won't change

```java
// DO: Factory methods for immutable collections
public class ProductCatalog {
    private static final List<String> CATEGORIES = 
        List.of("Electronics", "Books", "Clothing");  // Immutable, memory efficient
    
    private static final Set<String> SUPPORTED_CURRENCIES = 
        Set.of("USD", "EUR", "GBP");  // Immutable set
    
    private static final Map<String, BigDecimal> TAX_RATES = Map.of(
        "CA", new BigDecimal("0.0725"),
        "NY", new BigDecimal("0.0875"),
        "TX", new BigDecimal("0.0625")
    );  // Immutable map
    
    public List<String> getCategories() {
        return CATEGORIES;  // Safe to return, can't be modified
    }
}

// DO: Use List.copyOf() for defensive copies
@Service
public class OrderService {
    public Order createOrder(List<OrderItem> items) {
        return new Order(List.copyOf(items));  // Immutable defensive copy
    }
}

// DO: Records with immutable collections
public record Order(
    String orderId,
    List<OrderItem> items,  // Should be immutable
    LocalDateTime createdAt
) {
    public Order {
        items = List.copyOf(items);  // Ensure immutability in compact constructor
    }
}
```

**When to use:** Constants, configuration, DTOs, method return values that shouldn't be modified.

**When NOT to use:** When collection needs to be modified by caller.

### ❌ DON'T: Return mutable collections or use old immutability patterns

```java
// DON'T: Return mutable internal collections
public class ProductCatalog {
    private final List<String> categories = new ArrayList<>();
    
    public List<String> getCategories() {
        return categories;  // Caller can modify internal state!
    }
}

// DON'T: Use old Collections.unmodifiableXxx() for new code
public class OldStyle {
    private static final List<String> CATEGORIES = 
        Collections.unmodifiableList(Arrays.asList("A", "B", "C"));  // Verbose
    
    // List.of() is better: more concise, less memory overhead
}

// DON'T: Unnecessary defensive copying
public class WastefulService {
    public void process(List<String> items) {
        List<String> copy = new ArrayList<>(items);  // Unnecessary if items won't change
        // Just use items directly if you won't modify it
    }
}
```

**When this appears:** Legacy code, unfamiliarity with Java 9+ collection factories.

**Why it's wrong:** Memory waste, potential bugs from unintended modifications.

---

## 3. Collection Initialization

### ✅ DO: Use modern initialization patterns

```java
// DO: Factory methods for small collections
List<String> names = List.of("Alice", "Bob", "Charlie");
Set<Integer> ids = Set.of(1, 2, 3, 4, 5);
Map<String, Integer> scores = Map.of("Alice", 100, "Bob", 85);

// DO: Builder pattern for complex collections (Guava)
ImmutableMap<String, List<String>> userGroups = ImmutableMap.<String, List<String>>builder()
    .put("admin", List.of("user1", "user2"))
    .put("viewer", List.of("user3", "user4"))
    .build();

// DO: Collection instantiation with capacity for known size
public List<User> loadUsers(int expectedCount) {
    List<User> users = new ArrayList<>(expectedCount);  // Avoid resizing
    // Load users
    return users;
}

// DO: Stream to collection
List<String> activeUsernames = users.stream()
    .filter(User::isActive)
    .map(User::getUsername)
    .toList();  // Java 16+ immutable list
```

**When to use:**
- `List.of()`: Fixed small lists (<10 elements)
- `new ArrayList<>(capacity)`: Known size, will be modified
- `.toList()`: Stream terminal operation (immutable result)
- `.collect(Collectors.toList())`: Stream to mutable list (rare need)

**When NOT to use:** `new ArrayList<>()` without capacity when size is known.

### ❌ DON'T: Use outdated initialization patterns

```java
// DON'T: Double brace initialization (creates anonymous class)
List<String> names = new ArrayList<String>() {{
    add("Alice");
    add("Bob");
    add("Charlie");
}};  // Memory leak risk, creates anonymous inner class

// DON'T: Arrays.asList() for modification
List<String> names = Arrays.asList("Alice", "Bob");
names.add("Charlie");  // UnsupportedOperationException!

// DON'T: No capacity hint when size is known
public List<User> loadUsers(ResultSet rs) throws SQLException {
    List<User> users = new ArrayList<>();  // Will resize multiple times
    while (rs.next()) {
        users.add(mapUser(rs));  // Could be 10,000 rows
    }
    return users;
}

// Better:
rs.last();
int count = rs.getRow();
rs.beforeFirst();
List<User> users = new ArrayList<>(count);
```

**When this appears:** Old tutorials, Java 7- code, lack of awareness of modern APIs.

**Why it's wrong:** Performance, memory waste, unexpected behavior.

---

## 4. Null-Safety with Collections

### ✅ DO: Use empty collections instead of null

```java
// DO: Return empty collections, never null
@Service
public class UserService {
    public List<User> findUsersByRole(String role) {
        List<User> users = repository.findByRole(role);
        return users != null ? users : List.of();  // Never null
    }
    
    // DO: Even better with Optional
    public List<User> searchUsers(String query) {
        return Optional.ofNullable(repository.search(query))
                       .orElse(List.of());
    }
}

// DO: Collections.emptyXxx() for mutable empty collections
public List<Order> getRecentOrders(String userId) {
    if (userId == null) {
        return Collections.emptyList();  // Immutable empty list
    }
    return repository.findRecentOrders(userId);
}

// DO: Default to empty in constructors
public class ShoppingCart {
    private final List<CartItem> items;
    
    public ShoppingCart() {
        this.items = new ArrayList<>();  // Never null
    }
    
    public ShoppingCart(List<CartItem> items) {
        this.items = items != null ? new ArrayList<>(items) : new ArrayList<>();
    }
}
```

**When to use:** Always for collection return types. Null collections violate the Null Object pattern.

**When NOT to use:** Never return null collections from methods.

### ❌ DON'T: Return or allow null collections

```java
// DON'T: Return null collections
@Service
public class ProductService {
    public List<Product> findProducts(String category) {
        if (category == null) {
            return null;  // Forces caller to null-check
        }
        return repository.findByCategory(category);
    }
}

// DON'T: Null checks everywhere
public void processOrders(List<Order> orders) {
    if (orders != null) {  // Shouldn't be necessary
        for (Order order : orders) {
            process(order);
        }
    }
}

// DON'T: Nullable collection fields
public class User {
    private List<Address> addresses;  // Can be null, bug-prone
    
    public List<Address> getAddresses() {
        return addresses;  // NullPointerException waiting to happen
    }
}
```

**When this appears:** C-style thinking, defensive programming gone wrong.

**Why it's wrong:** Forces null checks everywhere, NullPointerExceptions, violates fail-fast principle.

---

## 5. Performance: Avoid Unnecessary Copies

### ✅ DO: Minimize copying and object creation

```java
// DO: Use views instead of copies
public class ReportService {
    public List<String> getTopProducts(List<Product> products, int limit) {
        return products.stream()
                      .sorted(Comparator.comparing(Product::getSales).reversed())
                      .limit(limit)
                      .map(Product::getName)
                      .toList();  // Single materialization
    }
}

// DO: Reuse collections in loops
public void processOrders(List<Order> orders) {
    List<String> results = new ArrayList<>(orders.size());  // Pre-sized
    
    for (Order order : orders) {
        results.add(process(order));
    }
}

// DO: Use primitive collections for performance (Eclipse Collections, Trove)
import org.eclipse.collections.api.list.primitive.IntList;
import org.eclipse.collections.impl.list.mutable.primitive.IntArrayList;

public class ScoreCalculator {
    public int calculateTotal(IntList scores) {  // No boxing overhead
        return scores.sum();  // Primitive operations
    }
}

// DO: Sublist views for slicing (Java 21 sequenced collections)
public List<User> getTopUsers(List<User> rankedUsers) {
    return rankedUsers.subList(0, Math.min(10, rankedUsers.size()));  // View, not copy
    // NOTE: Don't modify original list while using subList view
}
```

**When to use:** Hot paths, large collections, memory-constrained environments.

**When NOT to use:** Premature optimization. Profile first.

### ❌ DON'T: Unnecessary copying and inefficient operations

```java
// DON'T: Multiple intermediate collections
public List<String> processUsers(List<User> users) {
    List<User> activeUsers = new ArrayList<>();
    for (User user : users) {
        if (user.isActive()) {
            activeUsers.add(user);
        }
    }
    
    List<String> names = new ArrayList<>();
    for (User user : activeUsers) {
        names.add(user.getName());
    }
    
    List<String> uppercaseNames = new ArrayList<>();
    for (String name : names) {
        uppercaseNames.add(name.toUpperCase());
    }
    
    return uppercaseNames;
}

// Better: Single stream pipeline
public List<String> processUsers(List<User> users) {
    return users.stream()
                .filter(User::isActive)
                .map(User::getName)
                .map(String::toUpperCase)
                .toList();
}

// DON'T: Boxing overhead for primitives
public class ScoreProcessor {
    public int sum(List<Integer> scores) {  // Boxing/unboxing overhead
        int total = 0;
        for (Integer score : scores) {  // Unbox on every iteration
            total += score;
        }
        return total;
    }
}

// DON'T: Repeated contains() on List
public List<String> findUnique(List<String> items) {
    List<String> unique = new ArrayList<>();
    for (String item : items) {
        if (!unique.contains(item)) {  // O(n) for each item = O(n²) total
            unique.add(item);
        }
    }
    return unique;
}

// Better: Use Set
public List<String> findUnique(List<String> items) {
    return new ArrayList<>(new LinkedHashSet<>(items));  // O(n)
}
```

**When this appears:** Imperative style, unfamiliarity with streams, premature optimization avoidance.

**Why it's wrong:** Memory waste, CPU cycles, slower performance.

---

## 6. Thread-Safe Collections

### ✅ DO: Use concurrent collections for multi-threaded access

```java
// DO: ConcurrentHashMap for thread-safe caching
@Service
public class ProductCache {
    private final Map<String, Product> cache = new ConcurrentHashMap<>();
    
    public Product getOrLoad(String productId) {
        return cache.computeIfAbsent(productId, id -> {
            return productRepository.findById(id)
                                   .orElseThrow();
        });  // Atomic operation
    }
}

// DO: CopyOnWriteArrayList for rare writes, frequent reads
@Component
public class EventListeners {
    private final List<EventListener> listeners = new CopyOnWriteArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);  // Rare operation
    }
    
    public void notifyAll(Event event) {
        listeners.forEach(listener -> listener.onEvent(event));  // Frequent, thread-safe
    }
}

// DO: BlockingQueue for producer-consumer
@Service
public class TaskProcessor {
    private final BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(100);
    
    public void submitTask(Task task) throws InterruptedException {
        taskQueue.put(task);  // Blocks if full
    }
    
    @Async
    public void processTasksAsync() throws InterruptedException {
        while (true) {
            Task task = taskQueue.take();  // Blocks if empty
            process(task);
        }
    }
}
```

**When to use:**
- **ConcurrentHashMap**: Shared caches, multi-threaded lookups
- **CopyOnWriteArrayList**: Event listeners, rarely modified lists
- **BlockingQueue**: Producer-consumer patterns
- **ConcurrentSkipListMap/Set**: Concurrent sorted collections

**When NOT to use:** Single-threaded code (overhead), use Collections.synchronizedXxx() instead.

### ❌ DON'T: Use synchronized wrappers or manual synchronization

```java
// DON'T: Collections.synchronizedXxx() in new code
public class UserCache {
    private final Map<String, User> cache = 
        Collections.synchronizedMap(new HashMap<>());  // Coarse-grained locking
    
    public User getOrLoad(String userId) {
        synchronized (cache) {  // Manual sync needed for compound operations
            if (!cache.containsKey(userId)) {
                cache.put(userId, loadUser(userId));
            }
            return cache.get(userId);
        }
    }
}

// Better: ConcurrentHashMap
private final Map<String, User> cache = new ConcurrentHashMap<>();
public User getOrLoad(String userId) {
    return cache.computeIfAbsent(userId, this::loadUser);  // Atomic, fine-grained locking
}

// DON'T: Manual synchronization on collection
public class OrderProcessor {
    private final List<Order> orders = new ArrayList<>();
    
    public synchronized void addOrder(Order order) {  // Method-level sync
        orders.add(order);
    }
    
    public synchronized List<Order> getOrders() {
        return new ArrayList<>(orders);  // Defensive copy under lock
    }
}

// Better: Use thread-safe collection
private final List<Order> orders = new CopyOnWriteArrayList<>();
```

**When this appears:** Old concurrency patterns, lack of knowledge of java.util.concurrent.

**Why it's wrong:** Poor performance (coarse locks), hard to use correctly, error-prone.

---

## 7. Collection Streams Efficiency

### ✅ DO: Use streams efficiently

```java
// DO: Short-circuit operations
public Optional<User> findFirstAdmin(List<User> users) {
    return users.stream()
                .filter(User::isAdmin)
                .findFirst();  // Stops after first match
}

// DO: Parallel streams for CPU-intensive operations on large datasets
public List<ProcessedData> processLargeDataset(List<RawData> data) {
    if (data.size() < 1000) {
        return data.stream()
                   .map(this::expensiveProcessing)
                   .toList();
    }
    
    return data.parallelStream()  // Parallel for large datasets
               .map(this::expensiveProcessing)
               .toList();
}

// DO: Collect to specific collection types
public LinkedHashSet<String> getUniqueUsernames(List<User> users) {
    return users.stream()
                .map(User::getUsername)
                .collect(Collectors.toCollection(LinkedHashSet::new));
}

// DO: Use primitive streams to avoid boxing
public int sumScores(List<Score> scores) {
    return scores.stream()
                 .mapToInt(Score::getValue)  // IntStream, no boxing
                 .sum();
}
```

**When to use:**
- Streams: Transformation pipelines, filtering, mapping
- Parallel streams: Large datasets (>10k), CPU-intensive operations
- Primitive streams: Numeric operations to avoid boxing

**When NOT to use:** Small lists with simple operations (for-loop is clearer).

### ❌ DON'T: Misuse streams

```java
// DON'T: Stream for side effects
public void updateUsers(List<User> users) {
    users.stream()
         .forEach(user -> {
             user.setLastAccessed(LocalDateTime.now());  // Side effect
             repository.save(user);  // Side effect
         });
}

// Better: Use regular loop for side effects
public void updateUsers(List<User> users) {
    for (User user : users) {
        user.setLastAccessed(LocalDateTime.now());
        repository.save(user);
    }
}

// DON'T: Parallel streams with small data or I/O operations
public List<User> loadUsers(List<String> userIds) {
    return userIds.parallelStream()  // BAD: I/O bound, overhead > benefit
                  .map(repository::findById)
                  .filter(Optional::isPresent)
                  .map(Optional::get)
                  .toList();
}

// DON'T: Multiple terminal operations (stream consumed after first)
Stream<User> stream = users.stream().filter(User::isActive);
long count = stream.count();  // OK
List<User> list = stream.toList();  // IllegalStateException: stream already operated upon

// DON'T: Boxing overhead
public int sum(List<Integer> numbers) {
    return numbers.stream()
                  .reduce(0, Integer::sum);  // Boxing overhead on every element
}

// Better:
public int sum(List<Integer> numbers) {
    return numbers.stream()
                  .mapToInt(Integer::intValue)
                  .sum();
}
```

**When this appears:** Stream overuse, unfamiliarity with stream lifecycle.

**Why it's wrong:** Performance issues, unexpected exceptions, unclear intent.

---

## 8. Java 21 Sequenced Collections

### ✅ DO: Use new sequenced collection methods

```java
// DO: Use reversed() for reverse iteration
public List<Order> getLatestOrders(List<Order> orders) {
    return orders.reversed().stream()  // Java 21: efficient reverse view
                 .limit(10)
                 .toList();
}

// DO: getFirst() and getLast()
public Optional<User> getLatestUser(List<User> users) {
    return users.isEmpty() ? Optional.empty() : Optional.of(users.getLast());
}

// DO: addFirst() and addLast() for deques
public void addPriorityTask(Deque<Task> tasks, Task task) {
    tasks.addFirst(task);  // More readable than offerFirst()
}

// DO: removeFirst() and removeLast()
public Task getNextTask(Deque<Task> tasks) {
    return tasks.isEmpty() ? null : tasks.removeFirst();
}
```

**When to use:** When working with ordered collections and need first/last access or reversal.

**When NOT to use:** On Sets (unordered) or when order doesn't matter.

### ❌ DON'T: Use old patterns for first/last access

```java
// DON'T: Manual index access for first/last
public User getFirstUser(List<User> users) {
    return users.get(0);  // IndexOutOfBoundsException if empty
}

// Better:
public User getFirstUser(List<User> users) {
    return users.getFirst();  // Throws NoSuchElementException, clearer intent
}

// DON'T: Manual reversal
public List<String> reverseList(List<String> items) {
    List<String> reversed = new ArrayList<>(items);
    Collections.reverse(reversed);  // Mutates, creates copy
    return reversed;
}

// Better:
public List<String> reverseList(List<String> items) {
    return items.reversed();  // Immutable view (Java 21)
}

// DON'T: get(size() - 1) for last element
public Order getLatestOrder(List<Order> orders) {
    return orders.get(orders.size() - 1);  // Verbose, error-prone
}

// Better:
public Order getLatestOrder(List<Order> orders) {
    return orders.getLast();  // Clear, concise
}
```

**When this appears:** Pre-Java 21 code, habit.

**Why it's wrong:** More verbose, less clear intent, potential errors.
