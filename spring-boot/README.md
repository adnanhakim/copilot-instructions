# Spring Boot 3.5 & Java 21 Best Practices

A comprehensive collection of instruction files for GitHub Copilot (and other AI coding assistants) to generate high-quality, production-ready Spring Boot code following industry best practices.

## 📋 Overview

This repository contains curated best practices for building enterprise-grade Spring Boot applications. Each file provides DO/DON'T patterns with examples, focusing on:

- **Memory Efficiency** – Minimize allocations, use appropriate collection sizes
- **Readability** – Clear, self-documenting code following Google Java Style
- **Maintainability** – SOLID principles, clean architecture patterns
- **Performance** – Efficient queries, caching, virtual threads

## 🛠 Technology Stack

| Technology       | Version  |
| ---------------- | -------- |
| Java             | 21 (LTS) |
| Spring Boot      | 3.5.x    |
| Spring Framework | 6.x      |
| Spring Security  | 6.x      |
| Spring Data JPA  | 3.x      |

## 📁 Structure

```
spring-boot/
├── .github-copilot-instructions.md  # Main instructions file (entry point)
├── README.md                         # This file
└── copilot-rules/
    ├── collections.md                # Lists, Sets, Maps, immutability
    ├── concurrency.md                # Virtual threads, async, thread safety
    ├── configuration.md              # @ConfigurationProperties, profiles
    ├── data-access.md                # JPA, repositories, transactions
    ├── dependency-injection.md       # DI patterns, bean scopes
    ├── error-handling.md             # Exceptions, @ControllerAdvice
    ├── memory-management.md          # Resource cleanup, optimization
    ├── observability.md              # Logging, metrics, tracing
    ├── oop-principles.md             # SOLID, composition, encapsulation
    ├── rest-api-design.md            # Controllers, DTOs, versioning
    ├── security.md                   # Spring Security 6.x, JWT, CORS
    ├── streams-lambdas.md            # Functional patterns, Optional
    └── testing.md                    # JUnit 5, Testcontainers, slices
```

## 🚀 Quick Start

### Using with GitHub Copilot

1. **Copy the main instructions file** to your project:

   ```bash
   # Copy to your repository root
   cp .github-copilot-instructions.md /path/to/your/project/
   ```

2. **Copy specific rule files** you need:

   ```bash
   # Copy all rules
   cp -r copilot-rules /path/to/your/project/

   # Or copy specific files
   cp copilot-rules/data-access.md /path/to/your/project/copilot-rules/
   ```

3. **Update file paths** in `.github-copilot-instructions.md` if you placed files in different locations.

### Using with Cursor

1. Copy the files to your project's `.cursor/rules/` directory:
   ```bash
   mkdir -p .cursor/rules
   cp .github-copilot-instructions.md .cursor/rules/
   cp -r copilot-rules .cursor/rules/
   ```

### Using with Other AI Assistants

Simply include the relevant markdown files in your project or paste the content into your assistant's context/instructions.

## 📖 Topics Covered

### Core Java & OOP

- **[OOP Principles](./copilot-rules/oop-principles.md)** – SOLID, sealed classes, pattern matching
- **[Collections](./copilot-rules/collections.md)** – Right collection types, immutability, null safety
- **[Streams & Lambdas](./copilot-rules/streams-lambdas.md)** – Functional patterns, primitive streams
- **[Error Handling](./copilot-rules/error-handling.md)** – Exception hierarchies, @ControllerAdvice
- **[Concurrency](./copilot-rules/concurrency.md)** – Virtual threads, scoped values, CompletableFuture
- **[Memory Management](./copilot-rules/memory-management.md)** – Resource cleanup, GC-friendly code

### Spring Boot

- **[Dependency Injection](./copilot-rules/dependency-injection.md)** – Constructor injection, scopes
- **[REST API Design](./copilot-rules/rest-api-design.md)** – Controllers, DTOs, RFC 7807 errors
- **[Data Access](./copilot-rules/data-access.md)** – JPA, repositories, N+1 prevention
- **[Configuration](./copilot-rules/configuration.md)** – Type-safe config, profiles
- **[Security](./copilot-rules/security.md)** – Spring Security 6.x, JWT, CORS
- **[Testing](./copilot-rules/testing.md)** – JUnit 5, Testcontainers, slice tests
- **[Observability](./copilot-rules/observability.md)** – Logging, metrics, tracing

## 💡 Example Patterns

### Constructor Injection (Preferred)

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;
    private final PaymentService paymentService;
}
```

### Records for DTOs

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<OrderItem> items
) {}
```

### Virtual Threads for I/O

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Result>> futures = tasks.stream()
        .map(task -> executor.submit(() -> process(task)))
        .toList();
}
```

## 🔧 Customization

Feel free to:

- Add project-specific patterns to any file
- Remove files for technologies you don't use
- Adjust examples to match your coding conventions
- Add new topic files for additional frameworks

## 📝 File Format

Each instruction file follows this structure:

```markdown
# Topic Name

## 1. Category Name

### ✅ DO: Brief description

\`\`\`java
// Example code
\`\`\`

**When to use:**

- Guidance on when to apply this pattern

### ❌ DON'T: Anti-pattern description

\`\`\`java
// Anti-pattern code
\`\`\`

**Why it's wrong:** Explanation
```

## 📜 References

- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Effective Java (3rd Edition)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)

## 🤝 Contributing

Contributions are welcome! Please:

1. Follow the existing file format
2. Include both DO and DON'T examples
3. Add "When to use" guidance
4. Test code examples conceptually

## 📄 License

MIT License - feel free to use and modify for your projects.
