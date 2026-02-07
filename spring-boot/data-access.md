# Data Access (JPA/Hibernate) Best Practices

## 1. Repository Design

### ✅ DO: Use Spring Data JPA effectively

```java
// DO: Extend JpaRepository for standard CRUD
public interface UserRepository extends JpaRepository<User, String> {
    
    // Query methods with clear naming
    Optional<User> findByUsername(String username);
    
    List<User> findByActiveTrue();
    
    Page<User> findByRole(UserRole role, Pageable pageable);
    
    boolean existsByEmail(String email);
    
    long countByActiveTrue();
}

// DO: Custom queries for complex operations
public interface OrderRepository extends JpaRepository<Order, String> {
    
    @Query("SELECT o FROM Order o WHERE o.customer.id = :customerId " +
           "AND o.status = :status ORDER BY o.createdAt DESC")
    List<Order> findCustomerOrders(
        @Param("customerId") String customerId,
        @Param("status") OrderStatus status
    );
    
    // Native query when JPQL is not suitable
    @Query(value = "SELECT * FROM orders WHERE total > :amount " +
                   "AND created_at > :since",
           nativeQuery = true)
    List<Order> findHighValueOrdersSince(
        @Param("amount") BigDecimal amount,
        @Param("since") LocalDateTime since
    );
    
    // Projection for specific fields (avoid loading entire entity)
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.total, o.status) " +
           "FROM Order o WHERE o.customer.id = :customerId")
    List<OrderSummary> findOrderSummaries(@Param("customerId") String customerId);
}

// DO: Use specifications for dynamic queries
public interface UserRepository extends JpaRepository<User, String>, 
                                       JpaSpecificationExecutor<User> {
}

public class UserSpecifications {
    
    public static Specification<User> hasRole(UserRole role) {
        return (root, query, cb) -> 
            role == null ? null : cb.equal(root.get("role"), role);
    }
    
    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }
    
    public static Specification<User> usernameContains(String username) {
        return (root, query, cb) ->
            username == null ? null : cb.like(
                cb.lower(root.get("username")),
                "%" + username.toLowerCase() + "%"
            );
    }
}

// Usage
public Page<User> searchUsers(UserRole role, String username, boolean activeOnly, Pageable pageable) {
    Specification<User> spec = Specification.where(null);
    
    if (role != null) {
        spec = spec.and(UserSpecifications.hasRole(role));
    }
    if (username != null) {
        spec = spec.and(UserSpecifications.usernameContains(username));
    }
    if (activeOnly) {
        spec = spec.and(UserSpecifications.isActive());
    }
    
    return userRepository.findAll(spec, pageable);
}
```

**When to use:**
- JpaRepository: Standard CRUD operations
- Query methods: Simple queries derived from method names
- @Query: Complex queries, joins, projections
- Specifications: Dynamic queries with multiple optional filters
- Native queries: Database-specific features (last resort)

**When NOT to use:** Native queries for portable queries, repositories for business logic.

### ❌ DON'T: Misuse repositories

```java
// DON'T: Business logic in repository
public interface UserRepository extends JpaRepository<User, String> {
    
    // DON'T - business logic belongs in service
    default User createUserWithDefaults(String username, String email) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setRole(UserRole.USER);
        user.setActive(true);
        return save(user);
    }
}

// Better: Keep in service layer
@Service
public class UserService {
    public User create(CreateUserRequest request) {
        User user = new User();
        user.setUsername(request.username());
        user.setEmail(request.email());
        user.setRole(UserRole.USER);
        user.setActive(true);
        return repository.save(user);
    }
}

// DON'T: N+1 query problem
public List<OrderDto> getAllOrders() {
    List<Order> orders = orderRepository.findAll();
    
    return orders.stream()
                 .map(order -> {
                     Customer customer = order.getCustomer();  // Lazy load - N+1!
                     return new OrderDto(order, customer.getName());
                 })
                 .toList();
}

// Better: Use fetch join
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();

// DON'T: Return entities from repository methods with business logic
public interface UserRepository extends JpaRepository<User, String> {
    
    @Query("SELECT u FROM User u WHERE u.email = :email")
    User findByEmailAndUpdateLastLogin(@Param("email") String email);
    // Misleading - doesn't update anything, just queries
}

// Better: Separate query and update
Optional<User> findByEmail(String email);
// Update in service layer
```

**When this appears:** Confusion about layer responsibilities, performance issues.

**Why it's wrong:** N+1 queries, layer mixing, misleading API.

---

## 2. Entity Mapping

### ✅ DO: Map entities efficiently

```java
// DO: Use appropriate fetch types and cascade options
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    
    // Many-to-one: EAGER is OK for single object
    @ManyToOne(fetch = FetchType.LAZY)  // But LAZY is still preferred
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;
    
    // One-to-many: Always LAZY
    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    private List<OrderItem> items = new ArrayList<>();
    
    // Embedded value object
    @Embedded
    private Address shippingAddress;
    
    @Enumerated(EnumType.STRING)  // STRING, not ORDINAL
    @Column(nullable = false)
    private OrderStatus status;
    
    @Column(nullable = false)
    private BigDecimal total;
    
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    // Helper methods for bidirectional relationships
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

// DO: Value objects with @Embeddable
@Embeddable
public class Address {
    
    @Column(name = "address_street")
    private String street;
    
    @Column(name = "address_city")
    private String city;
    
    @Column(name = "address_zip")
    private String zipCode;
    
    @Column(name = "address_country")
    private String country;
}

// DO: Use converters for complex types
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, BigDecimal> {
    
    @Override
    public BigDecimal convertToDatabaseColumn(Money money) {
        return money == null ? null : money.amount();
    }
    
    @Override
    public Money convertToEntityAttribute(BigDecimal amount) {
        return amount == null ? null : new Money(amount, Currency.getInstance("USD"));
    }
}

// DO: Proper index definition
@Entity
@Table(
    name = "users",
    indexes = {
        @Index(name = "idx_username", columnList = "username"),
        @Index(name = "idx_email", columnList = "email"),
        @Index(name = "idx_role_active", columnList = "role, active")
    }
)
public class User {
    // ...
}
```

**When to use:**
- LAZY fetch: Default for collections, most relationships
- EAGER fetch: Rarely, when always needed together
- CascadeType.ALL: Parent owns children lifecycle
- @Embeddable: Value objects without separate table
- EnumType.STRING: Human-readable, migration-safe

**When NOT to use:**
- EAGER fetch for collections (N+1 problem)
- EnumType.ORDINAL (breaks when enum order changes)
- Cascade for shared entities

### ❌ DON'T: Poor entity mapping

```java
// DON'T: EAGER fetch for collections
@Entity
public class Order {
    
    @OneToMany(fetch = FetchType.EAGER)  // BAD - loads all items always
    private List<OrderItem> items;
    
    @ManyToOne(fetch = FetchType.EAGER)  // Could be LAZY
    private Customer customer;
}

// DON'T: Bidirectional relationships without helper methods
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items = new ArrayList<>();
}

// Client code (easy to forget other side):
Order order = new Order();
OrderItem item = new OrderItem();
order.getItems().add(item);  // Only one side set - inconsistent!

// Better: Helper methods ensure consistency
public void addItem(OrderItem item) {
    items.add(item);
    item.setOrder(this);  // Both sides set
}

// DON'T: EnumType.ORDINAL
@Enumerated(EnumType.ORDINAL)  // BAD - stored as 0, 1, 2...
private UserRole role;

// If you add new enum value at beginning:
// enum UserRole { ADMIN, USER, GUEST }  // Original
// enum UserRole { SUPER_ADMIN, ADMIN, USER, GUEST }  // After change
// All existing data now has wrong roles!

// Better: EnumType.STRING
@Enumerated(EnumType.STRING)  // Stored as "ADMIN", "USER", etc.
private UserRole role;

// DON'T: No indexes on frequently queried columns
@Entity
@Table(name = "users")  // No indexes
public class User {
    private String email;  // Frequently queried, but no index
}

// Better: Add indexes
@Table(
    name = "users",
    indexes = @Index(name = "idx_email", columnList = "email")
)

// DON'T: Primitive types for potentially null values
@Entity
public class User {
    private int loginCount;  // Can't be null - always 0 for new users
}

// Better: Use wrapper type if null is meaningful
private Integer loginCount;  // Can be null to indicate "never logged in"
```

**When this appears:** Lack of JPA knowledge, copy-paste without understanding.

**Why it's wrong:** Performance issues, data integrity problems, migration difficulties.

---

## 3. Transaction Management

### ✅ DO: Use transactions properly

```java
// DO: @Transactional on service methods
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    @Transactional  // Default: readOnly=false, propagation=REQUIRED
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(request.toEntity());
        paymentService.processPayment(order);
        return order;
        // Commit happens here if no exception
    }
    
    @Transactional(readOnly = true)  // Optimization for read-only operations
    public Order findById(String id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
    }
    
    @Transactional(readOnly = true)
    public Page<Order> findAll(Pageable pageable) {
        return orderRepository.findAll(pageable);
    }
}

// DO: Propagation for nested transactions
@Service
public class EmailService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendEmail(String to, String subject, String body) {
        // New transaction - commits independently
        // Even if caller transaction rolls back, email is still sent
        emailRepository.save(new EmailLog(to, subject, body));
    }
}

// DO: Isolation levels for specific needs
@Transactional(isolation = Isolation.SERIALIZABLE)
public void updateInventory(String productId, int quantity) {
    // Prevents concurrent modifications
    Product product = productRepository.findById(productId).orElseThrow();
    product.decrementStock(quantity);
    productRepository.save(product);
}

// DO: Explicit rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void processOrder(Order order) throws OrderProcessingException {
    // Rolls back for checked exceptions too
    orderRepository.save(order);
    if (!validate(order)) {
        throw new OrderProcessingException("Invalid order");
    }
}
```

**When to use:**
- @Transactional: Service layer methods (not repositories)
- readOnly=true: Read operations (optimization)
- REQUIRES_NEW: Independent nested operations
- rollbackFor: When checked exceptions should rollback

**When NOT to use:**
- @Transactional on repositories (redundant)
- @Transactional on controllers (wrong layer)
- Transactions for single read operations (overhead)

### ❌ DON'T: Transaction anti-patterns

```java
// DON'T: @Transactional on repository methods
public interface UserRepository extends JpaRepository<User, String> {
    
    @Transactional  // Redundant - Spring Data already handles this
    Optional<User> findByUsername(String username);
}

// DON'T: @Transactional on controller
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @PostMapping
    @Transactional  // Wrong layer!
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }
}

// Better: Transaction in service
@Service
public class OrderService {
    
    @Transactional
    public Order create(CreateOrderRequest request) {
        // Business logic and transaction here
    }
}

// DON'T: Calling transactional method from same class
@Service
public class UserService {
    
    public void processUser(String userId) {
        User user = findById(userId);  // Transaction not started!
        // Process user
    }
    
    @Transactional
    public User findById(String userId) {
        return userRepository.findById(userId).orElseThrow();
    }
}
// Spring AOP doesn't intercept internal calls - no transaction started!

// Better: Call from different bean or make both transactional
@Service
@RequiredArgsConstructor
public class UserService {
    
    @Transactional
    public void processUser(String userId) {  // Transaction here
        User user = findById(userId);
        // Process user
    }
    
    public User findById(String userId) {  // Participates in existing transaction
        return userRepository.findById(userId).orElseThrow();
    }
}

// DON'T: Long-running transactions
@Transactional
public void processAllOrders() {
    List<Order> orders = orderRepository.findAll();  // Could be millions
    
    for (Order order : orders) {
        processOrder(order);  // Slow operation
        // Transaction held for entire loop - locks database
    }
}

// Better: Process in batches with separate transactions
public void processAllOrders() {
    Page<Order> page;
    int pageNumber = 0;
    
    do {
        page = orderRepository.findAll(PageRequest.of(pageNumber++, 100));
        processOrderBatch(page.getContent());  // Separate transaction
    } while (page.hasNext());
}

@Transactional
private void processOrderBatch(List<Order> orders) {
    orders.forEach(this::processOrder);
    // Shorter transaction - releases locks faster
}
```

**When this appears:** Misunderstanding of Spring AOP, transaction scope confusion.

**Why it's wrong:** No transaction started, long-held locks, performance issues.

---

## 4. N+1 Query Problem

### ✅ DO: Avoid N+1 queries with fetch joins

```java
// DO: Fetch join for associations
public interface OrderRepository extends JpaRepository<Order, String> {
    
    @Query("SELECT o FROM Order o JOIN FETCH o.customer")
    List<Order> findAllWithCustomer();
    
    @Query("SELECT DISTINCT o FROM Order o " +
           "LEFT JOIN FETCH o.items " +
           "LEFT JOIN FETCH o.customer")
    List<Order> findAllWithItemsAndCustomer();
    
    @Query("SELECT o FROM Order o " +
           "JOIN FETCH o.customer " +
           "WHERE o.id = :id")
    Optional<Order> findByIdWithCustomer(@Param("id") String id);
}

// DO: Use @EntityGraph
public interface UserRepository extends JpaRepository<User, String> {
    
    @EntityGraph(attributePaths = {"roles", "profile"})
    Optional<User> findWithRolesAndProfileByUsername(String username);
    
    @EntityGraph(attributePaths = {"orders", "orders.items"})
    List<User> findAllWithOrders();
}

// DO: DTO projection to avoid loading entire entity
@Query("SELECT new com.example.dto.OrderSummary(o.id, o.total, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();

// DO: Batch fetching (application.yml)
// spring:
//   jpa:
//     properties:
//       hibernate:
//         default_batch_fetch_size: 10

@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 10)  // Fetches up to 10 customers in one query
    private Customer customer;
}
```

**When to use:**
- JOIN FETCH: When you always need the association
- @EntityGraph: More flexible than JOIN FETCH
- DTO projection: When you only need specific fields
- Batch fetching: Reduces N+1 to N/batch_size + 1

**When NOT to use:** EAGER fetch (loads always, even when not needed).

### ❌ DON'T: Allow N+1 queries

```java
// DON'T: Lazy loading in loop
public List<OrderDto> getAllOrders() {
    List<Order> orders = orderRepository.findAll();  // 1 query
    
    return orders.stream()
                 .map(order -> {
                     String customerName = order.getCustomer().getName();  // N queries!
                     return new OrderDto(order.getId(), order.getTotal(), customerName);
                 })
                 .toList();
}
// Total: 1 + N queries (if 1000 orders, 1001 queries!)

// Better: Fetch join
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();  // 1 query

// DON'T: Multiple separate queries
public OrderDetailDto getOrderDetail(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();  // Query 1
    Customer customer = customerRepository.findById(order.getCustomerId()).orElseThrow();  // Query 2
    List<OrderItem> items = orderItemRepository.findByOrderId(orderId);  // Query 3
    
    return new OrderDetailDto(order, customer, items);
}

// Better: Single query with joins
@Query("SELECT o FROM Order o " +
       "JOIN FETCH o.customer " +
       "LEFT JOIN FETCH o.items " +
       "WHERE o.id = :id")
Optional<Order> findByIdWithDetails(@Param("id") String id);

public OrderDetailDto getOrderDetail(String orderId) {
    Order order = orderRepository.findByIdWithDetails(orderId).orElseThrow();  // 1 query
    return new OrderDetailDto(order, order.getCustomer(), order.getItems());
}
```

**When this appears:** Lazy loading without awareness, lack of query optimization.

**Why it's wrong:** Performance disaster with large datasets, database overload.

---

## 5. Pagination and Sorting

### ✅ DO: Implement pagination for all lists

```java
// DO: Use Pageable for all list endpoints
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    public Page<UserResponse> getUsers(
            @PageableDefault(size = 20, sort = "createdAt", direction = Direction.DESC) 
            Pageable pageable) {
        return userService.findAll(pageable).map(UserResponse::from);
    }
}

// DO: Custom sorting with validation
@GetMapping
public Page<UserResponse> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "DESC") Sort.Direction direction) {
    
    // Validate sort field to prevent SQL injection
    Set<String> allowedFields = Set.of("username", "email", "createdAt", "lastLogin");
    if (!allowedFields.contains(sortBy)) {
        sortBy = "createdAt";
    }
    
    Pageable pageable = PageRequest.of(page, size, Sort.by(direction, sortBy));
    return userService.findAll(pageable).map(UserResponse::from);
}

// DO: Slice for infinite scrolling (doesn't count total)
public interface OrderRepository extends JpaRepository<Order, String> {
    
    Slice<Order> findByCustomerId(String customerId, Pageable pageable);
    // Faster than Page - doesn't execute count query
}

// DO: Cursor-based pagination for real-time data
public interface MessageRepository extends JpaRepository<Message, String> {
    
    @Query("SELECT m FROM Message m WHERE m.createdAt < :cursor ORDER BY m.createdAt DESC")
    List<Message> findBeforeCursor(@Param("cursor") LocalDateTime cursor, Pageable pageable);
}
```

**When to use:**
- Page: When you need total count (classic pagination)
- Slice: When you don't need total count (faster)
- Cursor-based: For real-time feeds, consistent results
- @PageableDefault: Sensible defaults

**When NOT to use:** Returning all records without pagination.

### ❌ DON'T: Return unpaginated lists

```java
// DON'T: Return all records
@GetMapping("/users")
public List<User> getAllUsers() {
    return userRepository.findAll();  // Could be millions of records!
}

// DON'T: Manual pagination
@GetMapping("/users")
public List<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    
    List<User> allUsers = userRepository.findAll();  // Loads ALL users into memory
    int start = page * size;
    int end = Math.min(start + size, allUsers.size());
    return allUsers.subList(start, end);  // Memory waste
}

// Better: Database pagination
public Page<User> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable);  // Database does pagination
}

// DON'T: Unrestricted page size
@GetMapping("/users")
public Page<User> getUsers(
        @RequestParam int page,
        @RequestParam int size) {  // No max limit!
    
    return userRepository.findAll(PageRequest.of(page, size));
    // User can request size=1000000
}

// Better: Limit maximum page size
@GetMapping("/users")
public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    
    if (size > 100) size = 100;  // Cap at 100
    return userRepository.findAll(PageRequest.of(page, size));
}
```

**When this appears:** Small datasets that grew, lack of scalability planning.

**Why it's wrong:** Out of memory errors, slow responses, database overload.
