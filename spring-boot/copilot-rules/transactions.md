# Transaction Management Best Practices

## 1. @Transactional Basics

### ✅ DO: Use @Transactional correctly

```java
// DO: Apply @Transactional at the service layer
@Service
@RequiredArgsConstructor
public class OrderService {

    @Transactional  // Default: propagation=REQUIRED, readOnly=false
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        inventoryService.reserve(order.getItems());
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)  // Optimization for read operations
    public Order findById(String id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
    }

    @Transactional(timeout = 30)  // 30 second timeout
    public void processLargeDataSet(List<DataRecord> records) {
        records.forEach(this::processRecord);
    }
}
```

**When to use:** Service layer methods, multiple repository calls, readOnly=true for reads.

**When NOT to use:** Repository methods, controller methods, private methods.

### ❌ DON'T: Common @Transactional mistakes

```java
// DON'T: @Transactional on private methods (ignored by Spring AOP)
@Transactional
private void updateUserInternal(User user) { /* IGNORED! */ }

// DON'T: Self-invocation (bypasses proxy)
public void processOrder(String orderId) {
    findById(orderId);  // @Transactional NOT applied!
}

@Transactional
public Order findById(String orderId) { return orderRepository.findById(orderId).orElseThrow(); }

// DON'T: @Transactional on repository (redundant)
public interface UserRepository extends JpaRepository<User, String> {
    @Transactional  // Spring Data already handles this
    Optional<User> findByEmail(String email);
}

// DON'T: @Transactional on controller (wrong layer)
@PostMapping("/users")
@Transactional  // Keep transactions in service
public UserResponse createUser(@RequestBody CreateUserRequest request) { }

// DON'T: Catching exceptions inside transaction
@Transactional
public void updateUser(User user) {
    try {
        userRepository.save(user);
        externalService.notify(user);
    } catch (Exception e) {
        log.error("Failed", e);  // Transaction still commits!
    }
}
```

---

## 2. Propagation Levels

### ✅ DO: Choose appropriate propagation

```java
// REQUIRED (default) - Join existing or create new
@Transactional(propagation = Propagation.REQUIRED)
public void createOrder(Order order) { orderRepository.save(order); }

// REQUIRES_NEW - Independent transaction (audit logs, notifications)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAuditEvent(AuditEvent event) {
    auditRepository.save(event);  // Commits even if caller rolls back
}

// NESTED - Savepoint within existing transaction
@Transactional(propagation = Propagation.NESTED)
public void processItem(OrderItem item) {
    itemRepository.save(item);  // Can rollback just this item
}

// MANDATORY - Must run within existing transaction
@Transactional(propagation = Propagation.MANDATORY)
public void updateInventory(String productId, int quantity) {
    // Throws exception if no active transaction
}
```

### ❌ DON'T: Misuse propagation

```java
// DON'T: REQUIRES_NEW everywhere (loses atomicity, poor performance)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public Order findById(String id) { }  // Unnecessary!

// DON'T: NOT_SUPPORTED for writes
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void updateUser(User user) { }  // No transaction - inconsistent!
```

---

## 3. Rollback Behavior

### ✅ DO: Configure rollback correctly

```java
// DO: Rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void processPayment(PaymentRequest request) throws PaymentException { }

// DO: Specific exception handling
@Transactional(
    rollbackFor = {DataAccessException.class, PaymentException.class},
    noRollbackFor = {NotificationException.class}
)
public void createOrder(CreateOrderRequest request) { }

// DO: Use unchecked exceptions for business errors
public class InsufficientBalanceException extends RuntimeException { }

@Transactional
public void withdraw(String accountId, BigDecimal amount) {
    if (account.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException("Insufficient balance");
        // RuntimeException triggers automatic rollback
    }
}
```

### ❌ DON'T: Incorrect rollback handling

```java
// DON'T: Assume checked exceptions rollback (they don't by default!)
@Transactional
public void transfer(TransferRequest request) throws InsufficientFundsException {
    throw new InsufficientFundsException("Not enough");  // Transaction COMMITS!
}

// DON'T: Swallow exceptions
@Transactional
public void processOrder(Order order) {
    try {
        orderRepository.save(order);
        inventoryService.reserve(order.getItems());
    } catch (InventoryException e) {
        log.error("Failed", e);  // Order saved despite inventory failure!
    }
}
```

---

## 4. Isolation Levels

### ✅ DO: Choose appropriate isolation

```java
// READ_COMMITTED (default) - Good balance
@Transactional(isolation = Isolation.READ_COMMITTED)
public Order findOrder(String id) { }

// REPEATABLE_READ - Consistent reads within transaction
@Transactional(isolation = Isolation.REPEATABLE_READ)
public OrderReport generateReport(String customerId) { }

// SERIALIZABLE - Critical financial operations
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferMoney(String from, String to, BigDecimal amount) { }

// Optimistic locking - Alternative to high isolation
@Entity
public class Account {
    @Version
    private Long version;  // Throws OptimisticLockException on conflict
}
```

### ❌ DON'T: Ignore isolation implications

```java
// DON'T: SERIALIZABLE everywhere (high overhead)
@Transactional(isolation = Isolation.SERIALIZABLE)
public List<Product> getProducts() { }  // Overkill!

// DON'T: Ignore lost updates
@Transactional
public void incrementCounter(String id) {
    Counter c = counterRepository.findById(id).orElseThrow();
    c.setValue(c.getValue() + 1);  // Lost update possible!
}

// Better: Atomic update
@Query("UPDATE Counter c SET c.value = c.value + 1 WHERE c.id = :id")
void incrementCounter(@Param("id") String id);
```

---

## 5. Transaction Scope and Performance

### ✅ DO: Optimize transaction boundaries

```java
// DO: Keep transactions short
@Transactional
public Order createOrder(CreateOrderRequest request) {
    return orderRepository.save(Order.from(request));
}

public void createOrderWithNotification(CreateOrderRequest request) {
    Order order = createOrder(request);  // Transaction complete
    notificationService.sendConfirmation(order);  // Outside transaction
}

// DO: Batch operations
@Transactional
public void createOrders(List<CreateOrderRequest> requests) {
    orderRepository.saveAll(requests.stream().map(Order::from).toList());
}

// DO: Eager load to avoid lazy loading issues
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") String id);
```

### ❌ DON'T: Bloated transactions

```java
// DON'T: External calls inside transaction
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(Order.from(request));
    paymentGateway.processPayment(order);  // Blocks transaction!
    emailService.sendConfirmation(order);  // Could fail, causing rollback!
    return order;
}

// DON'T: Process large collections in single transaction
@Transactional
public void updateAllUsers(List<UserUpdate> updates) {
    for (UserUpdate update : updates) {  // Could be millions
        userRepository.save(user);  // Long locks, memory issues
    }
}

// Better: Chunk processing
public void updateAllUsers(List<UserUpdate> updates) {
    Lists.partition(updates, 100).forEach(this::updateChunk);
}

@Transactional
public void updateChunk(List<UserUpdate> chunk) { /* Smaller transaction */ }
```

---

## 6. Programmatic Transactions

### ✅ DO: Use TransactionTemplate when needed

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final TransactionTemplate transactionTemplate;

    public Order createWithFallback(CreateOrderRequest request) {
        try {
            return transactionTemplate.execute(status -> {
                Order order = orderRepository.save(Order.from(request));
                inventoryService.reserve(order.getItems());
                return order;
            });
        } catch (InventoryException e) {
            return createBackorder(request);
        }
    }

    // Manual rollback
    public void processWithValidation(Order order) {
        transactionTemplate.executeWithoutResult(status -> {
            orderRepository.save(order);
            if (!validate(order)) {
                status.setRollbackOnly();
            }
        });
    }
}
```

### ❌ DON'T: Mix declarative and programmatic

```java
// DON'T: Both at same time
@Transactional
public void mixedApproach(Order order) {
    transactionTemplate.execute(status -> { /* Confusing! */ });
}

// DON'T: Create EntityManager manually with @Transactional
@Transactional
public void wrongApproach(User user) {
    EntityManager em = entityManagerFactory.createEntityManager();
    em.getTransaction().begin();  // Different transaction!
}
```
