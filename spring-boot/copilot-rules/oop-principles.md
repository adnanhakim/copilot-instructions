# OOP Principles Best Practices

## 1. Single Responsibility Principle (SRP)

### ✅ DO: One class, one responsibility

```java
// DO: Separate concerns into focused classes
public class UserService {
    private final UserRepository repository;
    private final UserValidator validator;
    private final UserNotifier notifier;

    public User createUser(UserDto dto) {
        validator.validate(dto);
        User user = repository.save(dto.toEntity());
        notifier.sendWelcomeEmail(user);
        return user;
    }
}

public class UserValidator {
    public void validate(UserDto dto) {
        // Validation logic only
    }
}

public class UserNotifier {
    public void sendWelcomeEmail(User user) {
        // Notification logic only
    }
}
```

**When to use:** Always. Each class should have one reason to change.

**When NOT to use:** Never violate this. Even small utilities should be focused.

### ❌ DON'T: Mix multiple responsibilities

```java
// DON'T: God class with multiple responsibilities
public class UserService {
    public User createUser(UserDto dto) {
        // Validation
        if (dto.email() == null || !dto.email().contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }

        // Persistence
        Connection conn = DriverManager.getConnection(url);
        PreparedStatement stmt = conn.prepareStatement("INSERT INTO users...");
        // ... JDBC code

        // Email sending
        Session session = Session.getInstance(props);
        MimeMessage message = new MimeMessage(session);
        // ... Email code

        // Logging
        FileWriter fw = new FileWriter("users.log");
        // ... File I/O

        return user;
    }
}
```

**When this appears:** Legacy code, rapid prototypes that weren't refactored.

**Why it's wrong:** Hard to test, maintain, and reuse. Changes ripple everywhere.

---

## 2. Open/Closed Principle (OCP)

### ✅ DO: Use abstraction for extension

```java
// DO: Strategy pattern for extensibility
public interface DiscountStrategy {
    BigDecimal calculateDiscount(Order order);
}

@Component
public class SeasonalDiscountStrategy implements DiscountStrategy {
    @Override
    public BigDecimal calculateDiscount(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.10"));
    }
}

@Component
public class LoyaltyDiscountStrategy implements DiscountStrategy {
    @Override
    public BigDecimal calculateDiscount(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.15"));
    }
}

@Service
public class OrderService {
    private final Map<String, DiscountStrategy> strategies;

    public BigDecimal applyDiscount(Order order, String strategyType) {
        DiscountStrategy strategy = strategies.get(strategyType);
        return strategy.calculateDiscount(order);
    }
}
```

**When to use:** When you anticipate variations in behavior (payment methods, pricing strategies, validation rules).

**When NOT to use:** For truly one-off, unique logic that will never vary.

### ❌ DON'T: Modify existing code for new features

```java
// DON'T: Adding if/else for every new discount type
public class OrderService {
    public BigDecimal applyDiscount(Order order, String type) {
        if ("SEASONAL".equals(type)) {
            return order.getTotal().multiply(new BigDecimal("0.10"));
        } else if ("LOYALTY".equals(type)) {
            return order.getTotal().multiply(new BigDecimal("0.15"));
        } else if ("CLEARANCE".equals(type)) {  // New requirement = modification
            return order.getTotal().multiply(new BigDecimal("0.30"));
        } else if ("BLACK_FRIDAY".equals(type)) {  // Another modification
            return order.getTotal().multiply(new BigDecimal("0.50"));
        }
        return BigDecimal.ZERO;
    }
}
```

**When this appears:** Quick fixes, tight deadlines, lack of design foresight.

**Why it's wrong:** Violates OCP. Every new discount requires modifying tested code, increasing bug risk.

---

## 3. Liskov Substitution Principle (LSP)

### ✅ DO: Ensure substitutability

```java
// DO: Subtypes honor base class contracts
public abstract class Document {
    public abstract void save();
    public abstract String getContent();

    public final void process() {
        validate();
        save();
        log();
    }

    protected abstract void validate();
    protected void log() {
        // Default logging
    }
}

public class PdfDocument extends Document {
    @Override
    public void save() {
        // Save as PDF
    }

    @Override
    public String getContent() {
        return extractPdfContent();
    }

    @Override
    protected void validate() {
        // PDF-specific validation
    }
}

// Can substitute anywhere Document is used
public void processDocument(Document doc) {
    doc.process();  // Works for any Document subtype
}
```

**When to use:** Always when using inheritance. Subtypes must be drop-in replacements.

**When NOT to use:** When behavior fundamentally differs, use composition instead.

### ❌ DON'T: Break base class contracts

```java
// DON'T: Subtype changes expected behavior
public abstract class Bird {
    public abstract void fly();
}

public class Sparrow extends Bird {
    @Override
    public void fly() {
        System.out.println("Flying high!");
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
        // BREAKS LSP - callers expect fly() to work
    }
}

// This will fail for Penguin
public void makeBirdFly(Bird bird) {
    bird.fly();  // Boom! Runtime exception for Penguin
}
```

**When this appears:** Modeling real-world relationships without considering behavior.

**Why it's wrong:** Code using base type (Bird) breaks with subtypes. Use interfaces instead:

```java
// Better: Separate flying behavior
public interface Flyable {
    void fly();
}

public class Sparrow implements Flyable { /*...*/ }
public class Penguin { /* No fly method */ }
```

---

## 4. Interface Segregation Principle (ISP)

### ✅ DO: Small, focused interfaces

```java
// DO: Segregated interfaces for different clients
public interface Readable {
    String read();
}

public interface Writable {
    void write(String content);
}

public interface Deletable {
    void delete();
}

// Implement only what's needed
public class ReadOnlyDocument implements Readable {
    @Override
    public String read() {
        return content;
    }
}

public class FullAccessDocument implements Readable, Writable, Deletable {
    @Override
    public String read() { return content; }

    @Override
    public void write(String content) { this.content = content; }

    @Override
    public void delete() { this.content = null; }
}
```

**When to use:** Always. Clients should only depend on methods they use.

**When NOT to use:** For truly cohesive operations that always go together.

### ❌ DON'T: Fat interfaces

```java
// DON'T: Force implementations of unused methods
public interface DocumentOperations {
    String read();
    void write(String content);
    void delete();
    void encrypt();
    void compress();
    void share();
}

public class ReadOnlyDocument implements DocumentOperations {
    @Override
    public String read() {
        return content;
    }

    @Override
    public void write(String content) {
        throw new UnsupportedOperationException();  // Forced to implement
    }

    @Override
    public void delete() {
        throw new UnsupportedOperationException();  // Not needed
    }

    @Override
    public void encrypt() {
        throw new UnsupportedOperationException();  // Not needed
    }

    @Override
    public void compress() {
        throw new UnsupportedOperationException();  // Not needed
    }

    @Override
    public void share() {
        throw new UnsupportedOperationException();  // Not needed
    }
}
```

**When this appears:** Over-generalization, designing interfaces without considering specific use cases.

**Why it's wrong:** Forces implementations to stub methods, violates ISP and LSP.

---

## 5. Dependency Inversion Principle (DIP)

### ✅ DO: Depend on abstractions

```java
// DO: High-level depends on abstraction
public interface NotificationService {
    void send(String recipient, String message);
}

@Component
public class EmailNotificationService implements NotificationService {
    @Override
    public void send(String recipient, String message) {
        // Email implementation
    }
}

@Component
public class SmsNotificationService implements NotificationService {
    @Override
    public void send(String recipient, String message) {
        // SMS implementation
    }
}

@Service
public class OrderService {
    private final NotificationService notificationService;

    // Depends on abstraction, not concrete class
    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        // Business logic
        notificationService.send(order.getCustomerEmail(), "Order placed");
    }
}
```

**When to use:** Always for dependencies that might change or have multiple implementations.

**When NOT to use:** For stable, framework classes (String, LocalDate, etc.).

### ❌ DON'T: Depend on concrete implementations

```java
// DON'T: High-level module depends on low-level details
@Service
public class OrderService {
    private final EmailNotificationService emailService;  // Concrete dependency

    public OrderService() {
        this.emailService = new EmailNotificationService();  // Direct instantiation
    }

    public void placeOrder(Order order) {
        // Business logic
        emailService.sendEmail(order.getCustomerEmail(), "Order placed");
        // Now we can't switch to SMS without modifying OrderService
    }
}
```

**When this appears:** Quick implementations, lack of abstraction planning.

**Why it's wrong:** Tight coupling. Can't test, can't swap implementations, violates DIP.

---

## 6. Favor Composition Over Inheritance

### ✅ DO: Use composition for flexibility

```java
// DO: Compose behaviors
public class Logger {
    public void log(String message) {
        System.out.println(message);
    }
}

public class Validator {
    public void validate(Object obj) {
        // Validation logic
    }
}

@Service
public class UserService {
    private final Logger logger;
    private final Validator validator;
    private final UserRepository repository;

    public UserService(Logger logger, Validator validator, UserRepository repository) {
        this.logger = logger;
        this.validator = validator;
        this.repository = repository;
    }

    public User createUser(UserDto dto) {
        validator.validate(dto);
        logger.log("Creating user: " + dto.username());
        return repository.save(dto.toEntity());
    }
}
```

**When to use:** When you need to combine behaviors from multiple sources.

**When NOT to use:** For true "is-a" relationships with shared behavior (rare).

### ❌ DON'T: Deep inheritance hierarchies

```java
// DON'T: Deep, rigid inheritance
public abstract class BaseService {
    protected void log(String msg) { }
}

public abstract class ValidatingService extends BaseService {
    protected void validate(Object obj) { }
}

public abstract class CachingValidatingService extends ValidatingService {
    protected void cache(Object obj) { }
}

public class UserService extends CachingValidatingService {
    // Inherits baggage from 3 levels
    // Can't reuse logging without validation
    // Can't swap caching strategy
}
```

**When this appears:** Trying to reuse code through inheritance, modeling complex taxonomies.

**Why it's wrong:** Rigid, hard to change, violates SRP. Changes in base classes affect all descendants.

---

## 7. Encapsulation

### ✅ DO: Hide internal state

```java
// DO: Proper encapsulation with Java 21 records
public record Money(BigDecimal amount, Currency currency) {
    // Compact constructor for validation
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        Objects.requireNonNull(currency, "Currency required");
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// DO: Encapsulation in regular classes
public class BankAccount {
    private final String accountNumber;
    private BigDecimal balance;  // Private, controlled access

    public BankAccount(String accountNumber, BigDecimal initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit must be positive");
        }
        this.balance = this.balance.add(amount);
    }

    public BigDecimal getBalance() {
        return balance;  // Returns copy (BigDecimal is immutable)
    }
}
```

**When to use:** Always. Protect invariants and hide implementation details.

**When NOT to use:** Never expose mutable internal state directly.

### ❌ DON'T: Expose internal state

```java
// DON'T: Public mutable fields
public class BankAccount {
    public String accountNumber;  // Anyone can modify
    public BigDecimal balance;     // No validation possible

    // Getter exposes mutable collection
    private List<Transaction> transactions = new ArrayList<>();

    public List<Transaction> getTransactions() {
        return transactions;  // Caller can modify internal list!
    }
}

// Usage - breaks encapsulation
BankAccount account = new BankAccount();
account.balance = new BigDecimal("-1000");  // No validation!
account.getTransactions().clear();  // Oops, deleted all transactions
```

**When this appears:** DTOs (but use records instead), legacy code, quick prototypes.

**Why it's wrong:** No control over state, invariants can't be enforced, internal changes break clients.

**Fix:**

```java
public List<Transaction> getTransactions() {
    return Collections.unmodifiableList(transactions);  // Or List.copyOf(transactions)
}
```

---

## 8. Sealed Classes and Pattern Matching (Java 17+)

### ✅ DO: Use sealed classes for controlled hierarchies

```java
// DO: Sealed interface for known implementations
public sealed interface PaymentMethod
    permits CreditCard, PayPal, BankTransfer, Crypto {

    String processPayment(BigDecimal amount);
}

public record CreditCard(String cardNumber, String cvv, YearMonth expiry)
    implements PaymentMethod {

    @Override
    public String processPayment(BigDecimal amount) {
        return processCardPayment(amount);
    }
}

public record PayPal(String email) implements PaymentMethod {
    @Override
    public String processPayment(BigDecimal amount) {
        return processPayPalPayment(amount);
    }
}

public record BankTransfer(String iban, String bic) implements PaymentMethod {
    @Override
    public String processPayment(BigDecimal amount) {
        return processBankTransfer(amount);
    }
}

public record Crypto(String walletAddress, CryptoType type) implements PaymentMethod {
    @Override
    public String processPayment(BigDecimal amount) {
        return processCryptoPayment(amount);
    }
}

// DO: Pattern matching with exhaustive switch (Java 21)
@Service
public class PaymentService {

    public PaymentResult process(PaymentMethod method, BigDecimal amount) {
        return switch (method) {
            case CreditCard card -> processCreditCard(card, amount);
            case PayPal paypal -> processPayPal(paypal, amount);
            case BankTransfer transfer -> processBankTransfer(transfer, amount);
            case Crypto crypto -> processCrypto(crypto, amount);
            // No default needed - compiler ensures exhaustiveness!
        };
    }

    // Guarded patterns
    public BigDecimal calculateFee(PaymentMethod method, BigDecimal amount) {
        return switch (method) {
            case CreditCard card when card.cardNumber().startsWith("4") ->
                amount.multiply(new BigDecimal("0.015"));  // Visa: 1.5%
            case CreditCard card when card.cardNumber().startsWith("5") ->
                amount.multiply(new BigDecimal("0.02"));   // Mastercard: 2%
            case CreditCard card ->
                amount.multiply(new BigDecimal("0.025"));  // Other: 2.5%
            case PayPal _ -> amount.multiply(new BigDecimal("0.029"));
            case BankTransfer _ -> new BigDecimal("0.50");  // Flat fee
            case Crypto _ -> amount.multiply(new BigDecimal("0.01"));
        };
    }
}

// DO: Sealed classes for state machines
public sealed interface OrderState
    permits Pending, Confirmed, Shipped, Delivered, Cancelled {
}

public record Pending(LocalDateTime createdAt) implements OrderState {}
public record Confirmed(LocalDateTime confirmedAt, String paymentId) implements OrderState {}
public record Shipped(LocalDateTime shippedAt, String trackingNumber) implements OrderState {}
public record Delivered(LocalDateTime deliveredAt) implements OrderState {}
public record Cancelled(LocalDateTime cancelledAt, String reason) implements OrderState {}

public class Order {
    private OrderState state = new Pending(LocalDateTime.now());

    public boolean canCancel() {
        return switch (state) {
            case Pending _, Confirmed _ -> true;
            case Shipped _, Delivered _, Cancelled _ -> false;
        };
    }
}
```

**When to use:**

- Modeling finite state machines
- Payment/notification types
- API response variants
- Domain objects with known subtypes

### ❌ DON'T: Use if/else chains for type checking

```java
// DON'T: Manual type checking
public BigDecimal calculateFee(Object payment) {
    if (payment instanceof CreditCard) {
        CreditCard card = (CreditCard) payment;
        return processCreditCardFee(card);
    } else if (payment instanceof PayPal) {
        return processPayPalFee((PayPal) payment);
    } else if (payment instanceof BankTransfer) {
        return processBankTransferFee((BankTransfer) payment);
    }
    throw new IllegalArgumentException("Unknown payment type");
}
// Problems: Verbose, no compile-time exhaustiveness, can miss new types

// Better: Pattern matching with sealed types
public BigDecimal calculateFee(PaymentMethod payment) {
    return switch (payment) {
        case CreditCard card -> processCreditCardFee(card);
        case PayPal paypal -> processPayPalFee(paypal);
        case BankTransfer transfer -> processBankTransferFee(transfer);
        case Crypto crypto -> processCryptoFee(crypto);
        // Compile error if you forget a type!
    };
}
```

**Why it's wrong:** No compile-time safety, easy to forget cases when adding new types.
