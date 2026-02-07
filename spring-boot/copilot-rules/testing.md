# Testing Best Practices

## 1. Test Structure

### ✅ DO: Follow Given-When-Then pattern

```java
// DO: Clear test structure with JUnit 5
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("Should create user when valid request is provided")
    void shouldCreateUserWhenValidRequest() {
        // Given
        CreateUserRequest request = new CreateUserRequest("john", "john@example.com", "password123");
        User savedUser = new User("1", "john", "john@example.com", "encodedPassword");

        when(passwordEncoder.encode("password123")).thenReturn("encodedPassword");
        when(userRepository.save(any(User.class))).thenReturn(savedUser);

        // When
        User result = userService.createUser(request);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getUsername()).isEqualTo("john");
        assertThat(result.getEmail()).isEqualTo("john@example.com");

        verify(passwordEncoder).encode("password123");
        verify(userRepository).save(any(User.class));
    }

    @Test
    @DisplayName("Should throw exception when user not found")
    void shouldThrowExceptionWhenUserNotFound() {
        // Given
        String userId = "nonexistent";
        when(userRepository.findById(userId)).thenReturn(Optional.empty());

        // When & Then
        assertThatThrownBy(() -> userService.findById(userId))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("User")
            .hasMessageContaining(userId);
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "ab"})
    @DisplayName("Should reject invalid usernames")
    void shouldRejectInvalidUsernames(String username) {
        // Given
        CreateUserRequest request = new CreateUserRequest(username, "valid@email.com", "password123");

        // When & Then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(ValidationException.class);
    }

    @ParameterizedTest
    @CsvSource({
        "john, john@example.com, true",
        "jane, jane@example.com, false"
    })
    @DisplayName("Should check user activation status")
    void shouldCheckUserActivationStatus(String username, String email, boolean expectedActive) {
        // Given
        User user = new User("1", username, email, "password");
        user.setActive(expectedActive);
        when(userRepository.findByUsername(username)).thenReturn(Optional.of(user));

        // When
        boolean isActive = userService.isUserActive(username);

        // Then
        assertThat(isActive).isEqualTo(expectedActive);
    }
}
```

**When to use:**

- Given-When-Then: All unit tests
- @DisplayName: Describe expected behavior
- @ParameterizedTest: Multiple inputs with same logic

### ❌ DON'T: Write unclear tests

```java
// DON'T: Unclear test names
@Test
void test1() {
    // What does this test?
}

// DON'T: Multiple assertions without context
@Test
void testUser() {
    User user = userService.createUser(request);
    assertNotNull(user);
    assertEquals("john", user.getUsername());
    assertTrue(user.isActive());
    assertNotNull(user.getCreatedAt());
    // What are we actually testing?
}

// DON'T: No Given section - unclear setup
@Test
void shouldFindUser() {
    User result = userService.findById("1");
    assertNotNull(result);
    // Where does user "1" come from? What state?
}
```

---

## 2. Slice Tests

### ✅ DO: Use appropriate slice tests

```java
// DO: @WebMvcTest for controller layer
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @BeforeEach
    void setUp() {
        // Common setup
    }

    @Test
    @DisplayName("GET /users/{id} should return user when found")
    void getUser_shouldReturnUser_whenFound() throws Exception {
        // Given
        User user = new User("1", "john", "john@example.com");
        when(userService.findById("1")).thenReturn(user);

        // When & Then
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("john"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    @DisplayName("POST /users should create user with valid request")
    void createUser_shouldCreateUser_withValidRequest() throws Exception {
        // Given
        String requestBody = """
            {
                "username": "john",
                "email": "john@example.com",
                "password": "password123"
            }
            """;
        User createdUser = new User("1", "john", "john@example.com");
        when(userService.createUser(any())).thenReturn(createdUser);

        // When & Then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("1"))
            .andExpect(jsonPath("$.username").value("john"));
    }

    @Test
    @DisplayName("POST /users should return 400 for invalid email")
    void createUser_shouldReturn400_forInvalidEmail() throws Exception {
        String requestBody = """
            {
                "username": "john",
                "email": "invalid-email",
                "password": "password123"
            }
            """;

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.email").exists());
    }
}

// DO: @DataJpaTest for repository layer
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("Should find user by email")
    void shouldFindUserByEmail() {
        // Given
        User user = new User("john", "john@example.com", "encodedPassword");
        entityManager.persistAndFlush(user);

        // When
        Optional<User> result = userRepository.findByEmail("john@example.com");

        // Then
        assertThat(result).isPresent();
        assertThat(result.get().getUsername()).isEqualTo("john");
    }

    @Test
    @DisplayName("Should find active users by role")
    void shouldFindActiveUsersByRole() {
        // Given
        User activeAdmin = createUser("admin1", UserRole.ADMIN, true);
        User inactiveAdmin = createUser("admin2", UserRole.ADMIN, false);
        User activeUser = createUser("user1", UserRole.USER, true);

        entityManager.persist(activeAdmin);
        entityManager.persist(inactiveAdmin);
        entityManager.persist(activeUser);
        entityManager.flush();

        // When
        List<User> result = userRepository.findByRoleAndActiveTrue(UserRole.ADMIN);

        // Then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getUsername()).isEqualTo("admin1");
    }
}
```

**When to use:**

- @WebMvcTest: Controller tests (HTTP layer only)
- @DataJpaTest: Repository tests (database layer)
- @JsonTest: JSON serialization/deserialization
- @WebFluxTest: WebFlux controller tests

---

## 3. Integration Tests with Testcontainers

### ✅ DO: Use Testcontainers for realistic integration tests

```java
// DO: Full integration test with Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    @DisplayName("Full user lifecycle - create, read, update, delete")
    void userLifecycle() {
        // Create
        CreateUserRequest createRequest = new CreateUserRequest("john", "john@example.com", "password123");
        ResponseEntity<UserResponse> createResponse = restTemplate.postForEntity(
            "/api/v1/users",
            createRequest,
            UserResponse.class
        );

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String userId = createResponse.getBody().id();

        // Read
        ResponseEntity<UserResponse> getResponse = restTemplate.getForEntity(
            "/api/v1/users/" + userId,
            UserResponse.class
        );

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().username()).isEqualTo("john");

        // Update
        UpdateUserRequest updateRequest = new UpdateUserRequest("john_updated", "john.new@example.com");
        restTemplate.put("/api/v1/users/" + userId, updateRequest);

        // Verify update
        UserResponse updated = restTemplate.getForObject("/api/v1/users/" + userId, UserResponse.class);
        assertThat(updated.username()).isEqualTo("john_updated");

        // Delete
        restTemplate.delete("/api/v1/users/" + userId);

        // Verify deletion
        ResponseEntity<UserResponse> deleteCheck = restTemplate.getForEntity(
            "/api/v1/users/" + userId,
            UserResponse.class
        );
        assertThat(deleteCheck.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}

// DO: Reusable container configuration
@TestConfiguration
public class TestContainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    GenericContainer<?> redisContainer() {
        return new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    }
}
```

### ❌ DON'T: Mock everything in integration tests

```java
// DON'T: Mock database in integration test
@SpringBootTest
class BadIntegrationTest {

    @MockBean
    private UserRepository userRepository;  // Defeats purpose of integration test

    @MockBean
    private EmailService emailService;  // Maybe OK if external service

    @Test
    void shouldCreateUser() {
        when(userRepository.save(any())).thenReturn(new User(...));
        // Not testing real database interaction!
    }
}
```

---

## 4. Test Fixtures and Builders

### ✅ DO: Use test builders for clean data setup

```java
// DO: Test data builder
public class UserBuilder {
    private String id = UUID.randomUUID().toString();
    private String username = "testuser";
    private String email = "test@example.com";
    private String password = "encodedPassword";
    private UserRole role = UserRole.USER;
    private boolean active = true;
    private LocalDateTime createdAt = LocalDateTime.now();

    public static UserBuilder aUser() {
        return new UserBuilder();
    }

    public UserBuilder withId(String id) {
        this.id = id;
        return this;
    }

    public UserBuilder withUsername(String username) {
        this.username = username;
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder withRole(UserRole role) {
        this.role = role;
        return this;
    }

    public UserBuilder active() {
        this.active = true;
        return this;
    }

    public UserBuilder inactive() {
        this.active = false;
        return this;
    }

    public User build() {
        User user = new User(id, username, email, password);
        user.setRole(role);
        user.setActive(active);
        user.setCreatedAt(createdAt);
        return user;
    }
}

// Usage in tests
@Test
void shouldFindActiveAdmins() {
    // Given
    User activeAdmin = UserBuilder.aUser()
        .withUsername("admin1")
        .withRole(UserRole.ADMIN)
        .active()
        .build();

    User inactiveAdmin = UserBuilder.aUser()
        .withUsername("admin2")
        .withRole(UserRole.ADMIN)
        .inactive()
        .build();

    // ...
}

// DO: Use records for test constants
public final class TestUsers {
    public static final CreateUserRequest VALID_REQUEST =
        new CreateUserRequest("john", "john@example.com", "password123");

    public static final CreateUserRequest INVALID_EMAIL_REQUEST =
        new CreateUserRequest("john", "invalid-email", "password123");

    private TestUsers() {}
}
```

---

## 5. Testing Async Code

### ✅ DO: Test async operations properly

```java
// DO: Test CompletableFuture with Awaitility
@Test
@DisplayName("Should process order asynchronously")
void shouldProcessOrderAsync() {
    // Given
    Order order = OrderBuilder.anOrder().build();

    // When
    CompletableFuture<ProcessedOrder> future = orderService.processAsync(order);

    // Then
    await()
        .atMost(Duration.ofSeconds(5))
        .untilAsserted(() -> {
            assertThat(future).isCompleted();
            assertThat(future.join().getStatus()).isEqualTo(OrderStatus.PROCESSED);
        });
}

// DO: Test @Async methods
@SpringBootTest
class AsyncServiceTest {

    @Autowired
    private AsyncService asyncService;

    @Test
    @DisplayName("Should execute async method on separate thread")
    void shouldExecuteOnSeparateThread() throws Exception {
        // Given
        String mainThread = Thread.currentThread().getName();

        // When
        CompletableFuture<String> future = asyncService.getExecutingThreadName();
        String asyncThread = future.get(5, TimeUnit.SECONDS);

        // Then
        assertThat(asyncThread).isNotEqualTo(mainThread);
    }
}

// DO: Test scheduled tasks
@SpringBootTest
class ScheduledTaskTest {

    @SpyBean
    private ScheduledTask scheduledTask;

    @Test
    @DisplayName("Should execute scheduled task within expected interval")
    void shouldExecuteScheduledTask() {
        await()
            .atMost(Duration.ofSeconds(10))
            .untilAsserted(() ->
                verify(scheduledTask, atLeast(2)).performTask()
            );
    }
}
```

---

## 6. Testing Best Practices Summary

### Memory-Efficient Testing

```java
// DO: Use @DirtiesContext sparingly
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)  // Not AFTER_EACH_TEST_METHOD

// DO: Share containers across tests
@Testcontainers
class BaseIntegrationTest {

    @Container
    protected static final PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true);  // Reuse across test runs
}

// DO: Clean up test data in @BeforeEach, not by restarting context
@BeforeEach
void cleanUp() {
    userRepository.deleteAll();  // Fast
    // Instead of @DirtiesContext which restarts Spring context
}

// DO: Use lazy bean initialization in tests
@TestConfiguration
public class TestConfig {

    @Bean
    @Lazy
    ExpensiveService expensiveService() {
        return new ExpensiveService();  // Only created when needed
    }
}
```
