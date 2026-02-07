# REST API Design Best Practices

## 1. Controller Structure

### ✅ DO: Design clean, RESTful controllers

```java
// DO: Use @RestController and proper HTTP methods
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    public Page<UserResponse> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String search) {
        return userService.findAll(PageRequest.of(page, size), search)
                         .map(UserResponse::from);
    }
    
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable String id) {
        return UserResponse.from(userService.findById(id));
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        return UserResponse.from(user);
    }
    
    @PutMapping("/{id}")
    public UserResponse updateUser(
            @PathVariable String id,
            @Valid @RequestBody UpdateUserRequest request) {
        User user = userService.update(id, request);
        return UserResponse.from(user);
    }
    
    @PatchMapping("/{id}")
    public UserResponse partialUpdate(
            @PathVariable String id,
            @Valid @RequestBody PatchUserRequest request) {
        User user = userService.patch(id, request);
        return UserResponse.from(user);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable String id) {
        userService.delete(id);
    }
}
```

**When to use:**
- @RestController for REST APIs (combines @Controller + @ResponseBody)
- @RequestMapping for base path
- Specific HTTP method annotations (@GetMapping, @PostMapping, etc.)
- @Valid for request validation
- @ResponseStatus for non-200 success codes

**When NOT to use:** @Controller (use @RestController), @RequestMapping without specific HTTP method.

### ❌ DON'T: Poor controller design

```java
// DON'T: Use generic @RequestMapping
@RestController
public class UserController {
    
    @RequestMapping("/users")  // DON'T - which HTTP method?
    public List<User> users() {
        return userService.findAll();
    }
    
    @RequestMapping(value = "/user", method = RequestMethod.POST)  // Verbose
    public User create(@RequestBody User user) {  // DON'T - expose entity directly
        return userService.save(user);
    }
}

// DON'T: Return entities directly
@GetMapping("/{id}")
public User getUser(@PathVariable String id) {  // Entity exposed
    return userService.findById(id);
}

// Better: Use DTOs
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable String id) {
    return UserResponse.from(userService.findById(id));
}

// DON'T: Mix responsibilities in controller
@RestController
@RequestMapping("/users")
public class UserController {
    
    @Autowired
    private UserRepository repository;  // Direct repository access - BAD
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        User user = repository.findById(id).orElse(null);
        if (user != null) {
            // Business logic in controller - BAD
            user.setLastAccessed(LocalDateTime.now());
            repository.save(user);
        }
        return user;
    }
}

// Better: Use service layer
@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;  // Service layer
    
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable String id) {
        return UserResponse.from(userService.findById(id));
    }
}

// DON'T: No pagination for lists
@GetMapping
public List<User> getAllUsers() {  // Could return millions of records!
    return userService.findAll();
}

// Better: Always paginate
@GetMapping
public Page<UserResponse> getUsers(Pageable pageable) {
    return userService.findAll(pageable).map(UserResponse::from);
}
```

**When this appears:** Quick implementations, lack of REST knowledge, old Spring patterns.

**Why it's wrong:** Poor API design, performance issues, tight coupling, security risks.

---

## 2. Request/Response DTOs

### ✅ DO: Use dedicated DTOs with validation

```java
// DO: Request DTOs with validation (Java 21 records)
public record CreateUserRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    String username,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password,
    
    @NotNull(message = "Role is required")
    UserRole role
) {
    public User toEntity(PasswordEncoder encoder) {
        return User.builder()
                  .username(username)
                  .email(email)
                  .password(encoder.encode(password))
                  .role(role)
                  .build();
    }
}

// DO: Response DTOs that expose only needed fields
public record UserResponse(
    String id,
    String username,
    String email,
    UserRole role,
    LocalDateTime createdAt,
    boolean active
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getUsername(),
            user.getEmail(),  // OK to expose
            user.getRole(),
            user.getCreatedAt(),
            user.isActive()
            // password NOT included - security
        );
    }
}

// DO: Patch request with all fields optional
public record PatchUserRequest(
    Optional<@Size(min = 3, max = 50) String> username,
    Optional<@Email String> email,
    Optional<Boolean> active
) {}

// DO: Nested DTOs for complex structures
public record CreateOrderRequest(
    @NotBlank String customerId,
    
    @NotEmpty(message = "Order must have at least one item")
    @Valid List<OrderItemRequest> items,
    
    @Valid ShippingAddressRequest shippingAddress
) {
    public record OrderItemRequest(
        @NotBlank String productId,
        @Min(1) int quantity
    ) {}
    
    public record ShippingAddressRequest(
        @NotBlank String street,
        @NotBlank String city,
        @NotBlank String zipCode,
        @NotBlank String country
    ) {}
}
```

**When to use:**
- Always for API requests and responses
- Records for immutable DTOs (Java 14+)
- Bean Validation annotations
- Factory methods for entity conversion

**When NOT to use:** Entities directly in API (security risk, tight coupling).

### ❌ DON'T: Expose entities or skip validation

```java
// DON'T: Expose entity directly
@PostMapping
public User createUser(@RequestBody User user) {  // Entity exposed - BAD
    return userService.save(user);
    // Problems:
    // 1. Clients can set internal fields (id, createdAt, etc.)
    // 2. Password exposed in responses
    // 3. Database schema leaks into API
    // 4. Changes to entity break API
}

// DON'T: No validation
public record CreateUserRequest(
    String username,  // No @NotBlank - could be null or empty
    String email,     // No @Email - could be invalid
    String password   // No @Size - could be "123"
) {}

// DON'T: Use Map for structured data
@PostMapping
public User createUser(@RequestBody Map<String, Object> userData) {  // NO!
    String username = (String) userData.get("username");  // Type unsafe
    String email = (String) userData.get("email");        // No validation
    // Error-prone, no IDE support, no documentation
}

// DON'T: Reuse request DTO for response
public record UserDto(
    String id,
    String username,
    String email,
    String password,  // DON'T return password!
    LocalDateTime createdAt
) {}

@PostMapping
public UserDto createUser(@RequestBody UserDto dto) {  // Same DTO for both
    return userService.create(dto);  // Returns password - security issue!
}

// Better: Separate request and response DTOs
public record CreateUserRequest(
    @NotBlank String username,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password
) {}

public record UserResponse(
    String id,
    String username,
    String email,
    LocalDateTime createdAt
    // No password
) {}
```

**When this appears:** Rapid prototyping, lack of security awareness, convenience over correctness.

**Why it's wrong:** Security vulnerabilities, validation bypassed, API instability.

---

## 3. Error Responses

### ✅ DO: Return consistent error responses

```java
// DO: Standardized error response
public record ErrorResponse(
    String errorCode,
    String message,
    LocalDateTime timestamp,
    String path
) {
    public static ErrorResponse of(String errorCode, String message, String path) {
        return new ErrorResponse(errorCode, message, LocalDateTime.now(), path);
    }
}

public record ValidationErrorResponse(
    String errorCode,
    String message,
    Map<String, String> fieldErrors,
    LocalDateTime timestamp,
    String path
) {}

// DO: Global exception handler
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {
        return ErrorResponse.of(
            "RESOURCE_NOT_FOUND",
            ex.getMessage(),
            request.getRequestURI()
        );
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {
        
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> Objects.requireNonNullElse(error.getDefaultMessage(), "Invalid value"),
                (existing, replacement) -> existing  // Keep first error per field
            ));
        
        return new ValidationErrorResponse(
            "VALIDATION_ERROR",
            "Validation failed",
            errors,
            LocalDateTime.now(),
            request.getRequestURI()
        );
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleConstraintViolation(
            ConstraintViolationException ex,
            HttpServletRequest request) {
        
        Map<String, String> errors = ex.getConstraintViolations()
            .stream()
            .collect(Collectors.toMap(
                violation -> violation.getPropertyPath().toString(),
                ConstraintViolation::getMessage
            ));
        
        return new ValidationErrorResponse(
            "CONSTRAINT_VIOLATION",
            "Constraint violation",
            errors,
            LocalDateTime.now(),
            request.getRequestURI()
        );
    }
}
```

**When to use:**
- All REST APIs
- Consistent error structure
- Appropriate HTTP status codes
- Error codes for client-side handling

**When NOT to use:** Different error formats for different endpoints.

### ❌ DON'T: Inconsistent or unsafe error responses

```java
// DON'T: Return different error formats
@RestController
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        try {
            return userService.findById(id);
        } catch (Exception e) {
            return null;  // Inconsistent - sometimes User, sometimes null
        }
    }
    
    @PostMapping
    public ResponseEntity<?> createUser(@RequestBody UserDto dto) {
        try {
            return ResponseEntity.ok(userService.create(dto));
        } catch (ValidationException e) {
            return ResponseEntity.badRequest().body(e.getMessage());  // Just string
        } catch (Exception e) {
            return ResponseEntity.status(500).body(Map.of("error", "Error"));  // Map
        }
    }
}

// DON'T: Expose internal error details
@ExceptionHandler(Exception.class)
public ErrorResponse handleException(Exception ex) {
    return new ErrorResponse(
        "ERROR",
        ex.getMessage(),  // Could expose SQL queries, file paths, etc.
        LocalDateTime.now(),
        ""
    );
}

// Better: Generic message for internal errors
@ExceptionHandler(Exception.class)
public ErrorResponse handleException(Exception ex) {
    log.error("Unexpected error", ex);  // Log full details
    return new ErrorResponse(
        "INTERNAL_ERROR",
        "An unexpected error occurred",  // Generic message to client
        LocalDateTime.now(),
        ""
    );
}

// DON'T: Return stack traces to clients
@ExceptionHandler(Exception.class)
public Map<String, Object> handleException(Exception ex) {
    return Map.of(
        "error", ex.getMessage(),
        "stackTrace", Arrays.toString(ex.getStackTrace())  // Security risk!
    );
}
```

**When this appears:** Ad-hoc error handling, lack of security awareness.

**Why it's wrong:** Inconsistent client experience, security vulnerabilities, poor error handling.

---

## 4. Versioning

### ✅ DO: Version your API

```java
// DO: URL path versioning (most common)
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserResponseV1 getUser(@PathVariable String id) {
        return userService.findById(id);
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable String id) {
        // V2 with additional fields or different structure
        return userService.findByIdV2(id);
    }
}

// DO: Header versioning (alternative)
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserResponseV1 getUserV1(@PathVariable String id) {
        return userService.findById(id);
    }
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserResponseV2 getUserV2(@PathVariable String id) {
        return userService.findByIdV2(id);
    }
}

// DO: Content negotiation versioning
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", produces = "application/vnd.api.v1+json")
    public UserResponseV1 getUserV1(@PathVariable String id) {
        return userService.findById(id);
    }
    
    @GetMapping(value = "/{id}", produces = "application/vnd.api.v2+json")
    public UserResponseV2 getUserV2(@PathVariable String id) {
        return userService.findByIdV2(id);
    }
}
```

**When to use:**
- Breaking API changes
- URL path versioning: Simple, visible, easy to test
- Header versioning: Cleaner URLs, more flexible
- Content negotiation: RESTful, complex

**When NOT to use:** For non-breaking changes (add optional fields to responses, new endpoints).

### ❌ DON'T: Break API without versioning

```java
// DON'T: Change existing endpoints without versioning
// Version 1
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable String id) {
    return new UserResponse(id, username, email);
}

// Later, changed to (BREAKS CLIENTS):
@GetMapping("/{id}")
public UserDetailResponse getUser(@PathVariable String id) {
    return new UserDetailResponse(id, username, email, phone, address);
    // Different response structure - breaking change!
}

// Better: Create new version
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable String id) {
        return new UserResponse(id, username, email);
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserDetailResponse getUser(@PathVariable String id) {
        return new UserDetailResponse(id, username, email, phone, address);
    }
}

// DON'T: Version in query parameters
@GetMapping("/users/{id}")
public UserResponse getUser(
        @PathVariable String id,
        @RequestParam(defaultValue = "1") int version) {  // BAD
    if (version == 1) {
        return getUserV1(id);
    } else {
        return getUserV2(id);
    }
}
```

**When this appears:** No versioning strategy, underestimating API changes.

**Why it's wrong:** Breaks existing clients, forces simultaneous updates.

---

## 5. Response Compression and Caching

### ✅ DO: Use HTTP caching and compression

```java
// DO: Enable compression in application.yml
// server:
//   compression:
//     enabled: true
//     mime-types: application/json,application/xml,text/html,text/xml,text/plain

// DO: Cache-Control headers
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable String id) {
        ProductResponse product = productService.findById(id);
        
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS)
                             .cachePublic())
                .eTag(String.valueOf(product.hashCode()))
                .body(product);
    }
    
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getProducts(Pageable pageable) {
        Page<ProductResponse> products = productService.findAll(pageable);
        
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
                .body(products);
    }
}

// DO: Conditional requests with ETags
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProduct(
        @PathVariable String id,
        @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {
    
    ProductResponse product = productService.findById(id);
    String etag = String.valueOf(product.hashCode());
    
    if (etag.equals(ifNoneMatch)) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                .eTag(etag)
                .build();
    }
    
    return ResponseEntity.ok()
            .eTag(etag)
            .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))
            .body(product);
}

// DO: No-cache for sensitive data
@GetMapping("/profile")
public ResponseEntity<UserProfile> getProfile() {
    UserProfile profile = userService.getCurrentUserProfile();
    
    return ResponseEntity.ok()
            .cacheControl(CacheControl.noStore())  // Never cache
            .body(profile);
}
```

**When to use:**
- Compression: Always (minimal overhead, significant bandwidth savings)
- Caching: Static or infrequently changing data
- ETags: For conditional requests, bandwidth optimization
- No-cache: Sensitive or user-specific data

**When NOT to use:** Caching for real-time or user-specific data (unless sophisticated strategy).

### ❌ DON'T: Ignore caching and compression

```java
// DON'T: No cache headers for cacheable data
@GetMapping("/products")
public List<Product> getProducts() {
    return productService.findAll();
    // No cache headers - fetched from server every time
}

// DON'T: Cache user-specific data
@GetMapping("/cart")
public ResponseEntity<ShoppingCart> getCart() {
    ShoppingCart cart = cartService.getCurrentUserCart();
    
    return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))  // BAD!
            .body(cart);
    // User A might get User B's cart from cache!
}

// DON'T: Compression disabled by default
// Check application.yml - ensure compression is enabled

// DON'T: Return large responses without pagination
@GetMapping("/orders")
public List<Order> getAllOrders() {
    return orderService.findAll();  // Could be millions of records
}

// Better:
@GetMapping("/orders")
public Page<OrderResponse> getOrders(Pageable pageable) {
    return orderService.findAll(pageable).map(OrderResponse::from);
}
```

**When this appears:** No performance optimization, lack of HTTP knowledge.

**Why it's wrong:** Poor performance, high bandwidth usage, scalability issues.
