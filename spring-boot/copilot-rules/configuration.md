# Configuration Best Practices

## 1. Configuration Properties

### ✅ DO: Use type-safe configuration with records

```java
// DO: @ConfigurationProperties with record (Java 16+)
@ConfigurationProperties(prefix = "app.security")
public record SecurityProperties(
    boolean enabled,
    String jwtSecret,
    Duration jwtExpiration,
    List<String> allowedOrigins
) {
    public SecurityProperties {
        // Compact constructor for validation
        if (enabled && (jwtSecret == null || jwtSecret.isBlank())) {
            throw new IllegalArgumentException("JWT secret required when security is enabled");
        }
        jwtExpiration = jwtExpiration != null ? jwtExpiration : Duration.ofHours(1);
        allowedOrigins = allowedOrigins != null ? List.copyOf(allowedOrigins) : List.of();
    }
}

// DO: Nested configuration with validation
@ConfigurationProperties(prefix = "app")
@Validated
public record ApplicationProperties(
    @NotBlank String name,
    @NotBlank String version,
    @Valid DatabaseProperties database,
    @Valid CacheProperties cache
) {
    public record DatabaseProperties(
        @NotBlank String url,
        @NotBlank String username,
        @NotBlank String password,
        @Min(1) @Max(100) int maxPoolSize,
        Duration connectionTimeout
    ) {
        public DatabaseProperties {
            maxPoolSize = maxPoolSize > 0 ? maxPoolSize : 10;
            connectionTimeout = connectionTimeout != null ? connectionTimeout : Duration.ofSeconds(30);
        }
    }

    public record CacheProperties(
        boolean enabled,
        Duration ttl,
        int maxSize
    ) {
        public CacheProperties {
            ttl = ttl != null ? ttl : Duration.ofMinutes(10);
            maxSize = maxSize > 0 ? maxSize : 1000;
        }
    }
}

// Enable in main class
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**application.yml:**

```yaml
app:
  name: my-application
  version: 1.0.0
  database:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    max-pool-size: 20
    connection-timeout: 30s
  cache:
    enabled: true
    ttl: 1h
    max-size: 5000
```

**When to use:**

- Groups of related properties
- Type conversion needed (Duration, DataSize, List)
- Validation required
- Properties used in multiple places

### ❌ DON'T: Scatter @Value annotations

```java
// DON'T: Multiple @Value annotations
@Service
public class DatabaseService {

    @Value("${app.database.url}")
    private String url;

    @Value("${app.database.username}")
    private String username;

    @Value("${app.database.password}")
    private String password;

    @Value("${app.database.max-pool-size:10}")
    private int maxPoolSize;

    // No type safety, no validation, scattered
}

// Better: Use @ConfigurationProperties
@Service
@RequiredArgsConstructor
public class DatabaseService {
    private final ApplicationProperties.DatabaseProperties dbProps;

    public Connection getConnection() {
        // Type-safe, validated access
        return createConnection(dbProps.url(), dbProps.username());
    }
}
```

---

## 2. Profiles

### ✅ DO: Use profiles for environment-specific configuration

```java
// DO: Profile-specific beans
@Configuration
@Profile("dev")
public class DevConfiguration {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfiguration {

    @Bean
    public DataSource dataSource(DatabaseProperties props) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(props.url());
        config.setUsername(props.username());
        config.setPassword(props.password());
        config.setMaximumPoolSize(props.maxPoolSize());
        return new HikariDataSource(config);
    }
}

// DO: Profile expressions
@Configuration
@Profile("!prod")  // Any non-production
public class NonProdConfiguration {
    @Bean
    public MockEmailService emailService() {
        return new MockEmailService();
    }
}

@Configuration
@Profile("cloud & !local")  // Cloud but not local
public class CloudConfiguration {
    // Cloud-specific beans
}
```

**application-dev.yml:**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    org.springframework: DEBUG
```

**application-prod.yml:**

```yaml
spring:
  datasource:
    url: ${DATABASE_URL}
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: WARN
    com.myapp: INFO
```

### ❌ DON'T: Hardcode environment checks

```java
// DON'T: Check profile in code
@Service
public class NotificationService {

    @Value("${spring.profiles.active}")
    private String activeProfile;

    public void send(Notification notification) {
        if ("prod".equals(activeProfile)) {
            realEmailService.send(notification);
        } else {
            log.info("Would send: {}", notification);
        }
    }
}

// Better: Profile-specific beans
@Service
@Profile("prod")
public class RealNotificationService implements NotificationService {
    public void send(Notification notification) {
        emailService.send(notification);
    }
}

@Service
@Profile("!prod")
public class MockNotificationService implements NotificationService {
    public void send(Notification notification) {
        log.info("Mock notification: {}", notification);
    }
}
```

---

## 3. External Configuration

### ✅ DO: Use environment variables and secrets properly

```yaml
# application.yml - Reference environment variables
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD} # No default for sensitive values

app:
  security:
    jwt-secret: ${JWT_SECRET} # Required, no default
    allowed-origins: ${ALLOWED_ORIGINS:http://localhost:3000}
```

```java
// DO: Fail fast on missing required config
@ConfigurationProperties(prefix = "app.security")
@Validated
public record SecurityProperties(
    @NotBlank(message = "JWT secret must be configured via JWT_SECRET env variable")
    String jwtSecret,

    @NotEmpty
    List<String> allowedOrigins
) {}

// DO: Use Spring Cloud Config for distributed config
// application.yml
spring:
  config:
    import: optional:configserver:http://config-server:8888

// DO: Use Spring Vault for secrets
// bootstrap.yml
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      authentication: KUBERNETES
      kubernetes:
        role: my-app
```

### ❌ DON'T: Hardcode secrets or sensitive configuration

```java
// DON'T: Hardcoded secrets
@Configuration
public class SecurityConfig {
    private static final String JWT_SECRET = "my-secret-key-123";  // NEVER!
}

// DON'T: Secrets in application.yml committed to git
spring:
  datasource:
    password: actual-password-here  # NEVER!

// DON'T: Log sensitive configuration
@PostConstruct
public void init() {
    log.info("Database password: {}", dbPassword);  // NEVER!
}
```

---

## 4. Configuration Validation

### ✅ DO: Validate configuration at startup

```java
// DO: Bean validation on properties
@ConfigurationProperties(prefix = "app.api")
@Validated
public record ApiProperties(
    @NotBlank String baseUrl,

    @Min(1) @Max(60)
    int timeoutSeconds,

    @Min(1) @Max(10)
    int retryAttempts,

    @Pattern(regexp = "^[A-Za-z0-9-]+$")
    String apiKey
) {}

// DO: Custom validation with @AssertTrue
@ConfigurationProperties(prefix = "app.feature")
@Validated
public record FeatureProperties(
    boolean enableCache,
    Duration cacheTtl,
    int cacheMaxSize
) {
    @AssertTrue(message = "Cache TTL and max size required when cache is enabled")
    private boolean isCacheConfigValid() {
        if (enableCache) {
            return cacheTtl != null && cacheMaxSize > 0;
        }
        return true;
    }
}

// DO: Fail-fast configuration check
@Component
@RequiredArgsConstructor
public class ConfigurationValidator {
    private final ApplicationProperties props;

    @EventListener(ApplicationReadyEvent.class)
    public void validateConfiguration() {
        if (props.database().maxPoolSize() < 5) {
            log.warn("Database pool size {} is low, consider increasing",
                props.database().maxPoolSize());
        }
    }
}
```

---

## 5. Property Sources and Precedence

### ✅ DO: Understand and use property precedence

```java
// Property precedence (highest to lowest):
// 1. Command line arguments: --server.port=8081
// 2. SPRING_APPLICATION_JSON environment variable
// 3. OS environment variables: SERVER_PORT=8081
// 4. application-{profile}.yml
// 5. application.yml
// 6. @PropertySource annotations
// 7. Default properties

// DO: Use relaxed binding
// All of these work for app.database.max-pool-size:
// - app.database.max-pool-size (kebab-case, recommended)
// - app.database.maxPoolSize (camelCase)
// - APP_DATABASE_MAX_POOL_SIZE (environment variable)

// DO: Custom property source for additional files
@Configuration
@PropertySource("classpath:custom.properties")
@PropertySource(value = "file:${app.config.path}/override.properties",
                ignoreResourceNotFound = true)
public class PropertySourceConfig {}

// DO: Use Spring Boot's config import
# application.yml
spring:
  config:
    import:
      - optional:file:./config/local.yml
      - optional:configserver:
      - optional:vault://
```

**When to use:**

- Environment variables: Secrets, environment-specific values
- Profile-specific YAML: Environment configurations
- External files: Local development overrides

### ❌ DON'T: Override properties in confusing ways

```java
// DON'T: Same property in multiple places without documentation
// application.yml
server:
  port: 8080

// application-dev.yml
server:
  port: 8081

// application-local.yml
server:
  port: 8082

// Plus environment variable: SERVER_PORT=8083
// What port will it run on? Confusion!

// Better: Document and use clear precedence
// application.yml
server:
  port: ${SERVER_PORT:8080}  # Clear that env var takes precedence
```
