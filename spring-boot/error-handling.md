# Error Handling Best Practices

## 1. Exception Hierarchy

### ✅ DO: Create meaningful exception hierarchies

```java
// DO: Custom exception hierarchy
public class ApplicationException extends RuntimeException {
    private final String errorCode;
    
    public ApplicationException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public ApplicationException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
}

public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resource, String id) {
        super(String.format("%s not found with id: %s", resource, id), "RESOURCE_NOT_FOUND");
    }
}

public class ValidationException extends ApplicationException {
    private final Map<String, String> fieldErrors;
    
    public ValidationException(Map<String, String> fieldErrors) {
        super("Validation failed", "VALIDATION_ERROR");
        this.fieldErrors = Map.copyOf(fieldErrors);
    }
    
    public Map<String, String> getFieldErrors() {
        return fieldErrors;
    }
}

public class BusinessRuleException extends ApplicationException {
    public BusinessRuleException(String message) {
        super(message, "BUSINESS_RULE_VIOLATION");
    }
}
```

**When to use:**
- Checked exceptions: Recovery is possible and expected (rare in modern Java)
- Unchecked exceptions: Programming errors, business rule violations (most cases)
- Custom exceptions: Domain-specific errors that need special handling

**When NOT to use:** Generic Exception or RuntimeException directly in business logic.

### ❌ DON'T: Use generic exceptions or poor hierarchy

```java
// DON'T: Throw generic Exception
public User findUser(String id) throws Exception {  // Too generic
    if (id == null) {
        throw new Exception("ID is null");  // Doesn't convey intent
    }
    return repository.findById(id);
}

// DON'T: Multiple unrelated exception types for same scenario
public class UserNotFoundException extends RuntimeException {}
public class UserNotFound extends RuntimeException {}
public class NoSuchUserException extends RuntimeException {}
// Three exceptions for same thing - confusing!

// DON'T: Checked exceptions for business logic
public void processOrder(Order order) throws OrderProcessingException {  // Checked
    // Forces all callers to handle or declare
}

// Better: Unchecked exception
public void processOrder(Order order) {  // Unchecked RuntimeException
    if (!order.isValid()) {
        throw new BusinessRuleException("Invalid order");
    }
}

// DON'T: Swallow exception information
public void processData(String data) {
    try {
        parse(data);
    } catch (ParseException e) {
        throw new RuntimeException("Error");  // Lost original exception!
    }
}

// Better: Chain exceptions
public void processData(String data) {
    try {
        parse(data);
    } catch (ParseException e) {
        throw new DataProcessingException("Failed to parse data", e);  // Preserve cause
    }
}
```

**When this appears:** Quick fixes, lack of exception design, legacy code patterns.

**Why it's wrong:** Poor error context, forces unnecessary exception handling, loses information.

---

## 2. Try-with-Resources

### ✅ DO: Use try-with-resources for AutoCloseable

```java
// DO: Single resource
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
    }  // Both closed in reverse order
}

// DO: Custom AutoCloseable resources
public class DatabaseConnection implements AutoCloseable {
    private final Connection connection;
    
    public DatabaseConnection(String url) throws SQLException {
        this.connection = DriverManager.getConnection(url);
    }
    
    public void executeQuery(String sql) throws SQLException {
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            // Process results
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
try (DatabaseConnection db = new DatabaseConnection(url)) {
    db.executeQuery("SELECT * FROM users");
}  // Automatically closed

// DO: Effectively final variables (Java 9+)
BufferedReader reader = Files.newBufferedReader(path);
try (reader) {  // Can use existing variable if effectively final
    return reader.readLine();
}
```

**When to use:** Always for resources that implement AutoCloseable (files, streams, connections).

**When NOT to use:** For objects that don't need cleanup or aren't AutoCloseable.

### ❌ DON'T: Manual resource management

```java
// DON'T: Manual try-finally
public String readFile(Path path) throws IOException {
    BufferedReader reader = null;
    try {
        reader = Files.newBufferedReader(path);
        return reader.lines().collect(Collectors.joining("\n"));
    } finally {
        if (reader != null) {
            try {
                reader.close();
            } catch (IOException e) {
                // What to do here?
            }
        }
    }
}

// Better: try-with-resources handles all this
public String readFile(Path path) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        return reader.lines().collect(Collectors.joining("\n"));
    }
}

// DON'T: Forget to close resources
public void processFile(Path path) {
    BufferedReader reader = Files.newBufferedReader(path);  // Never closed - resource leak!
    reader.lines().forEach(this::process);
}

// DON'T: Close in wrong order
Connection conn = null;
Statement stmt = null;
try {
    conn = getConnection();
    stmt = conn.createStatement();
    // ...
} finally {
    conn.close();  // Wrong order! Close stmt first
    stmt.close();  // May never execute if conn.close() throws
}

// Better: try-with-resources handles order
try (Connection conn = getConnection();
     Statement stmt = conn.createStatement()) {
    // ...
}  // Closed in reverse order: stmt, then conn
```

**When this appears:** Old code, unfamiliarity with try-with-resources (Java 7+).

**Why it's wrong:** Resource leaks, verbose code, error-prone cleanup logic.

---

## 3. Exception Handling in Spring Boot

### ✅ DO: Use @ControllerAdvice for global exception handling

```java
// DO: Global exception handler
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return new ErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            LocalDateTime.now()
        );
    }
    
    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidation(ValidationException ex) {
        log.warn("Validation error: {}", ex.getMessage());
        return new ValidationErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            ex.getFieldErrors(),
            LocalDateTime.now()
        );
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> Objects.requireNonNullElse(error.getDefaultMessage(), "Invalid value")
            ));
        
        return new ValidationErrorResponse("VALIDATION_ERROR", "Validation failed", errors, LocalDateTime.now());
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex) {
        log.error("Unexpected error", ex);
        return new ErrorResponse(
            "INTERNAL_ERROR",
            "An unexpected error occurred",  // Don't expose internal details
            LocalDateTime.now()
        );
    }
}

// DO: Structured error responses
public record ErrorResponse(
    String errorCode,
    String message,
    LocalDateTime timestamp
) {}

public record ValidationErrorResponse(
    String errorCode,
    String message,
    Map<String, String> fieldErrors,
    LocalDateTime timestamp
) {}
```

**When to use:**
- All REST API exception handling
- Converting exceptions to HTTP responses
- Logging errors consistently

**When NOT to use:** For exceptions that should propagate (very rare).

### ❌ DON'T: Handle exceptions in each controller

```java
// DON'T: Duplicate exception handling in controllers
@RestController
@RequestMapping("/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable String id) {
        try {
            User user = userService.findById(id);
            return ResponseEntity.ok(user);
        } catch (ResourceNotFoundException e) {
            return ResponseEntity.notFound().build();  // Duplicate logic
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();  // Duplicate logic
        }
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody UserDto dto) {
        try {
            User user = userService.create(dto);
            return ResponseEntity.ok(user);
        } catch (ValidationException e) {
            return ResponseEntity.badRequest().build();  // Duplicate logic
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();  // Duplicate logic
        }
    }
}

// Better: Let GlobalExceptionHandler handle it
@RestController
@RequestMapping("/users")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        return userService.findById(id);  // Exception propagates to handler
    }
    
    @PostMapping
    public User createUser(@Valid @RequestBody UserDto dto) {
        return userService.create(dto);  // Clean, focused on business logic
    }
}

// DON'T: Expose internal exception details
@ExceptionHandler(SQLException.class)
public ErrorResponse handleSQLException(SQLException ex) {
    return new ErrorResponse(
        "DB_ERROR",
        ex.getMessage(),  // Exposes DB schema, SQL queries - security risk!
        LocalDateTime.now()
    );
}

// Better: Generic message
@ExceptionHandler(SQLException.class)
public ErrorResponse handleSQLException(SQLException ex) {
    log.error("Database error", ex);  // Log details
    return new ErrorResponse(
        "INTERNAL_ERROR",
        "A database error occurred",  // Generic message to client
        LocalDateTime.now()
    );
}
```

**When this appears:** Controllers without @ControllerAdvice, security oversight.

**Why it's wrong:** Code duplication, inconsistent error responses, security risks.

---

## 4. Fail-Fast Principle

### ✅ DO: Validate early and fail fast

```java
// DO: Validate constructor parameters
public class Order {
    private final String orderId;
    private final List<OrderItem> items;
    private final Customer customer;
    
    public Order(String orderId, List<OrderItem> items, Customer customer) {
        this.orderId = Objects.requireNonNull(orderId, "Order ID cannot be null");
        this.items = Objects.requireNonNull(items, "Items cannot be null");
        this.customer = Objects.requireNonNull(customer, "Customer cannot be null");
        
        if (items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
    }
}

// DO: Use Objects utility methods
public void processUser(User user, String action) {
    Objects.requireNonNull(user, "User cannot be null");
    Objects.requireNonNull(action, "Action cannot be null");
    
    if (action.isBlank()) {
        throw new IllegalArgumentException("Action cannot be blank");
    }
    
    // Process
}

// DO: Validate in service methods
@Service
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request) {
        validateOrderRequest(request);  // Fail fast
        
        // Business logic
        return saveOrder(request);
    }
    
    private void validateOrderRequest(CreateOrderRequest request) {
        if (request.items().isEmpty()) {
            throw new ValidationException(Map.of("items", "At least one item required"));
        }
        
        if (request.items().stream().anyMatch(item -> item.quantity() <= 0)) {
            throw new ValidationException(Map.of("quantity", "Quantity must be positive"));
        }
    }
}

// DO: Use Bean Validation (JSR-380)
public record CreateUserRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    String username,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be at least 18 years old")
    Integer age
) {}

@RestController
public class UserController {
    
    @PostMapping("/users")
    public User createUser(@Valid @RequestBody CreateUserRequest request) {
        // Validation happens before method execution
        return userService.create(request);
    }
}
```

**When to use:**
- Constructor parameters
- Public method parameters
- Method preconditions
- Business rule validation

**When NOT to use:** For performance-critical paths where validation has been done upstream.

### ❌ DON'T: Defer validation or silently handle invalid state

```java
// DON'T: Accept null and deal with it later
public class OrderService {
    public Order createOrder(CreateOrderRequest request) {
        // No validation
        Order order = new Order();
        
        if (request != null) {  // Defensive programming gone wrong
            if (request.items() != null) {
                order.setItems(request.items());
            } else {
                order.setItems(new ArrayList<>());  // Empty order - invalid state!
            }
        }
        
        return repository.save(order);  // Saving invalid order
    }
}

// Better: Fail fast
public Order createOrder(CreateOrderRequest request) {
    Objects.requireNonNull(request, "Request cannot be null");
    
    if (request.items().isEmpty()) {
        throw new ValidationException(Map.of("items", "At least one item required"));
    }
    
    return repository.save(new Order(request));
}

// DON'T: Return null or default on invalid input
public User findUser(String id) {
    if (id == null || id.isBlank()) {
        return null;  // Hides the problem
    }
    return repository.findById(id);
}

// Better: Fail fast
public User findUser(String id) {
    if (id == null || id.isBlank()) {
        throw new IllegalArgumentException("ID cannot be null or blank");
    }
    return repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User", id));
}

// DON'T: Catch and ignore validation errors
public void processOrder(Order order) {
    try {
        validator.validate(order);
    } catch (ValidationException e) {
        // Ignore and continue - invalid state persists!
    }
    
    repository.save(order);  // Saving invalid order
}

// Better: Let exception propagate
public void processOrder(Order order) {
    validator.validate(order);  // Throws if invalid
    repository.save(order);
}
```

**When this appears:** Defensive programming, trying to "handle" all cases.

**Why it's wrong:** Invalid state propagates, errors surface far from root cause, hard to debug.

---

## 5. Logging Exceptions

### ✅ DO: Log exceptions appropriately

```java
// DO: Log at appropriate levels
@Service
@Slf4j
public class PaymentService {
    
    public Payment processPayment(PaymentRequest request) {
        try {
            return paymentGateway.charge(request);
        } catch (PaymentDeclinedException e) {
            // Expected business exception - INFO or WARN
            log.info("Payment declined for order {}: {}", 
                request.orderId(), e.getMessage());
            throw e;
        } catch (PaymentGatewayException e) {
            // External system error - ERROR with stack trace
            log.error("Payment gateway error for order {}", 
                request.orderId(), e);
            throw new PaymentProcessingException("Payment failed", e);
        }
    }
}

// DO: Log before rethrowing
@Service
@Slf4j
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request) {
        try {
            return processOrder(request);
        } catch (ValidationException e) {
            log.warn("Order validation failed: {}", e.getMessage());
            throw e;  // Rethrow after logging
        } catch (Exception e) {
            log.error("Unexpected error creating order", e);
            throw new OrderCreationException("Failed to create order", e);
        }
    }
}

// DO: Use parameterized logging
log.info("User {} logged in from {}", username, ipAddress);  // Efficient
log.error("Failed to process order {} for customer {}", orderId, customerId, exception);

// DO: Log with context in catch blocks
@Service
@Slf4j
public class FileProcessor {
    
    public void processFile(String filename) {
        try {
            String content = Files.readString(Path.of(filename));
            process(content);
        } catch (IOException e) {
            log.error("Failed to read file: {}", filename, e);  // Include filename in log
            throw new FileProcessingException("Cannot read file: " + filename, e);
        }
    }
}

// DO: Don't log and rethrow at every level
// Service layer - log and rethrow
@Service
@Slf4j
public class UserService {
    public User createUser(UserDto dto) {
        try {
            return repository.save(dto.toEntity());
        } catch (DataAccessException e) {
            log.error("Failed to create user: {}", dto.username(), e);
            throw new UserCreationException("Cannot create user", e);
        }
    }
}

// Controller layer - let @ControllerAdvice handle logging
@RestController
public class UserController {
    @PostMapping("/users")
    public User createUser(@RequestBody UserDto dto) {
        return userService.createUser(dto);  // Don't log here
    }
}
```

**When to use:**
- ERROR: System errors, unexpected exceptions, failures requiring attention
- WARN: Recoverable errors, business rule violations, degraded functionality
- INFO: Expected exceptions with business meaning (e.g., payment declined)
- DEBUG: Detailed flow for troubleshooting

**When NOT to use:** Excessive logging at every catch block (log once at appropriate level).

### ❌ DON'T: Log incorrectly

```java
// DON'T: String concatenation in logging
log.info("User " + username + " logged in");  // Always executes concatenation
log.error("Error: " + e.getMessage());  // Loses stack trace!

// Better:
log.info("User {} logged in", username);
log.error("Error processing request", e);  // Includes stack trace

// DON'T: Log and rethrow at every level
@RestController
public class UserController {
    @PostMapping("/users")
    public User createUser(@RequestBody UserDto dto) {
        try {
            return userService.createUser(dto);
        } catch (Exception e) {
            log.error("Error in controller", e);  // Logged here
            throw e;
        }
    }
}

@Service
public class UserService {
    public User createUser(UserDto dto) {
        try {
            return repository.save(dto.toEntity());
        } catch (Exception e) {
            log.error("Error in service", e);  // Logged again
            throw e;
        }
    }
}

// Result: Same exception logged multiple times with different messages

// Better: Log once at appropriate level (@ControllerAdvice)

// DON'T: Log sensitive information
log.info("User login: username={}, password={}", username, password);  // NEVER!
log.error("Credit card processing failed: {}", creditCardNumber);  // NEVER!

// Better:
log.info("User login: username={}", username);
log.error("Payment processing failed for user: {}", userId);

// DON'T: Swallow exceptions with just logging
public void processData(Data data) {
    try {
        process(data);
    } catch (ProcessingException e) {
        log.error("Processing failed", e);
        // Exception swallowed - caller doesn't know it failed!
    }
}

// Better: Log and rethrow, or use specific recovery strategy
public void processData(Data data) {
    try {
        process(data);
    } catch (ProcessingException e) {
        log.error("Processing failed", e);
        throw new DataProcessingException("Failed to process data", e);
    }
}
```

**When this appears:** Over-logging, security oversights, unclear error handling strategy.

**Why it's wrong:** Log pollution, security risks, lost exception context, hidden failures.
