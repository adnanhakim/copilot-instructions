# Date and Time Best Practices

## 1. Java Time API (java.time)

### ✅ DO: Use java.time classes exclusively

```java
// DO: Use appropriate type for your use case
public class DateTimeExamples {

    // Date only (no time, no zone) - birthdays, holidays
    private LocalDate birthDate = LocalDate.of(1990, Month.JANUARY, 15);

    // Time only (no date, no zone) - store hours, recurring times
    private LocalTime openingTime = LocalTime.of(9, 0);

    // Date + time (no zone) - local appointments
    private LocalDateTime appointmentTime = LocalDateTime.of(2024, 3, 15, 14, 30);

    // Instant - timestamps, logs, database storage (always UTC)
    private Instant createdAt = Instant.now();

    // ZonedDateTime - user-facing times with timezone
    private ZonedDateTime meetingTime = ZonedDateTime.now(ZoneId.of("America/New_York"));

    // Duration - time-based amounts (hours, minutes, seconds)
    private Duration timeout = Duration.ofMinutes(30);

    // Period - date-based amounts (years, months, days)
    private Period subscriptionLength = Period.ofMonths(12);
}

// DO: Store as Instant/UTC in database, convert for display
@Entity
public class Order {

    @Column(name = "created_at")
    private Instant createdAt;  // Always UTC in database

    @Column(name = "delivery_date")
    private LocalDate deliveryDate;  // No time component needed

    public ZonedDateTime getCreatedAtInTimezone(ZoneId userZone) {
        return createdAt.atZone(userZone);  // Convert for display
    }
}

// DO: Use Clock for testability
@Service
public class SubscriptionService {
    private final Clock clock;

    public SubscriptionService() {
        this.clock = Clock.systemUTC();  // Production
    }

    // For testing
    SubscriptionService(Clock clock) {
        this.clock = clock;
    }

    public boolean isExpired(Subscription subscription) {
        return Instant.now(clock).isAfter(subscription.getExpiresAt());
    }
}

// Test with fixed clock
@Test
void shouldDetectExpiredSubscription() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-06-15T10:00:00Z"),
        ZoneOffset.UTC
    );
    SubscriptionService service = new SubscriptionService(fixedClock);

    Subscription expired = new Subscription(Instant.parse("2024-06-01T00:00:00Z"));
    assertThat(service.isExpired(expired)).isTrue();
}
```

**When to use:**

- **Instant**: Timestamps, audit logs, database storage
- **LocalDate**: Birthdays, holidays, business dates
- **LocalTime**: Store hours, opening times
- **LocalDateTime**: Calendar appointments (user's local time)
- **ZonedDateTime**: Scheduling across timezones, display times
- **OffsetDateTime**: API responses (ISO 8601)

### ❌ DON'T: Use legacy date classes or ignore timezone (in multi-timezone systems)

```java
// DON'T: Legacy date classes
Date now = new Date();  // Mutable, confusing API
Calendar cal = Calendar.getInstance();  // Verbose, error-prone
java.sql.Date sqlDate = new java.sql.Date(System.currentTimeMillis());

// CAUTION: LocalDateTime in multi-timezone systems
@Entity
public class Event {
    private LocalDateTime scheduledTime;  // What timezone is this?
    // If user in NYC schedules "3pm", stored as 15:00
    // User in LA sees 15:00 but thinks it's their local 3pm!
}

// ✅ OK: LocalDateTime when entire system operates in one timezone
// If your DB, servers, and users are all in EST, LocalDateTime is fine
@Entity
public class InternalReport {
    // Document the assumption!
    /** All times in EST - internal reporting system only */
    private LocalDateTime generatedAt;
    private LocalDate reportDate;  // Date-only fields don't need timezone
}

// ✅ OK: LocalDate for dates that are timezone-independent
@Entity
public class Employee {
    private LocalDate birthDate;       // Birthdays are dates, not moments
    private LocalDate hireDate;        // Company-local date
    private LocalDate terminationDate; // Company-local date
}

// For multi-timezone: Store with timezone info
@Entity
public class GlobalEvent {
    private Instant scheduledTime;  // UTC
    private String scheduledTimezone;  // "America/New_York"

    public ZonedDateTime getScheduledTimeInOriginalZone() {
        return scheduledTime.atZone(ZoneId.of(scheduledTimezone));
    }
}

// DON'T: String manipulation for dates
String date = "2024-03-15";
String year = date.substring(0, 4);  // Fragile!

// Better: Parse properly
LocalDate date = LocalDate.parse("2024-03-15");
int year = date.getYear();
```

**When LocalDateTime is appropriate:**

- Internal systems where all users/servers are in the same timezone
- Database configured for a specific timezone (e.g., EST-only)
- Scheduled tasks that run at "business hours" in one location
- **Document the timezone assumption clearly!**

**When to use Instant + timezone:**

- Multi-timezone users (SaaS, global apps)
- Audit logs that need exact moment in time
- APIs consumed by external systems

---

## 2. Formatting and Parsing

### ✅ DO: Reuse formatters, use ISO standards

```java
// DO: Static final formatters (thread-safe)
public class DateTimeUtils {

    // ISO standards - best for APIs and storage
    public static final DateTimeFormatter ISO_DATE = DateTimeFormatter.ISO_LOCAL_DATE;
    public static final DateTimeFormatter ISO_INSTANT = DateTimeFormatter.ISO_INSTANT;

    // Custom formatters for display
    public static final DateTimeFormatter DISPLAY_DATE =
        DateTimeFormatter.ofPattern("MMM d, yyyy");  // "Mar 15, 2024"

    public static final DateTimeFormatter DISPLAY_DATE_TIME =
        DateTimeFormatter.ofPattern("MMM d, yyyy 'at' h:mm a");  // "Mar 15, 2024 at 2:30 PM"

    // Locale-aware formatter
    public static final DateTimeFormatter LOCALIZED_DATE =
        DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM);

    public static String formatForDisplay(LocalDate date) {
        return DISPLAY_DATE.format(date);
    }

    public static String formatForApi(Instant instant) {
        return ISO_INSTANT.format(instant);  // "2024-03-15T14:30:00Z"
    }
}

// DO: Parse with specific formatter
public LocalDate parseDate(String input) {
    try {
        return LocalDate.parse(input, DateTimeFormatter.ISO_LOCAL_DATE);
    } catch (DateTimeParseException e) {
        throw new ValidationException("Invalid date format. Expected: YYYY-MM-DD");
    }
}

// DO: Handle multiple input formats gracefully
public LocalDate parseFlexible(String input) {
    List<DateTimeFormatter> formatters = List.of(
        DateTimeFormatter.ISO_LOCAL_DATE,           // 2024-03-15
        DateTimeFormatter.ofPattern("MM/dd/yyyy"),  // 03/15/2024
        DateTimeFormatter.ofPattern("d-MMM-yyyy")   // 15-Mar-2024
    );

    for (DateTimeFormatter formatter : formatters) {
        try {
            return LocalDate.parse(input, formatter);
        } catch (DateTimeParseException ignored) {}
    }
    throw new ValidationException("Unrecognized date format: " + input);
}
```

### ❌ DON'T: Create formatters repeatedly or use ambiguous formats

```java
// DON'T: Create formatter on every call
public String formatDate(LocalDate date) {
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");  // Created every time!
    return formatter.format(date);
}

// DON'T: Ambiguous date formats
DateTimeFormatter BAD_FORMAT = DateTimeFormatter.ofPattern("dd/MM/yyyy");
// Is 01/02/2024 January 2nd or February 1st? Depends on locale!

// DON'T: SimpleDateFormat (not thread-safe!)
private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");  // Shared mutable state!

public String format(Date date) {
    return sdf.format(date);  // Thread-unsafe - can corrupt in concurrent access
}
```

---

## 3. Timezone Handling

### ✅ DO: Handle timezones explicitly

```java
// DO: Always be explicit about timezone
@Service
public class MeetingService {

    public ZonedDateTime scheduleMeeting(LocalDateTime userLocalTime, String userTimezone) {
        ZoneId zone = ZoneId.of(userTimezone);
        return userLocalTime.atZone(zone);  // Explicit conversion
    }

    public String getTimeForParticipant(ZonedDateTime meetingTime, String participantTimezone) {
        ZonedDateTime participantTime = meetingTime.withZoneSameInstant(
            ZoneId.of(participantTimezone)
        );
        return DateTimeFormatter.ofPattern("h:mm a z").format(participantTime);
    }
}

// DO: Handle DST transitions
public boolean isValidLocalTime(LocalDateTime time, ZoneId zone) {
    try {
        ZonedDateTime zdt = time.atZone(zone);
        // Check if the time exists (not in DST gap)
        return zdt.toLocalDateTime().equals(time);
    } catch (DateTimeException e) {
        return false;
    }
}

// DO: Use IANA timezone IDs
ZoneId newYork = ZoneId.of("America/New_York");  // IANA ID - handles DST
ZoneId tokyo = ZoneId.of("Asia/Tokyo");

// DO: Store user's timezone preference
@Entity
public class UserPreferences {

    @Column(name = "timezone")
    private String timezone = "UTC";  // Default to UTC

    public ZoneId getZoneId() {
        return ZoneId.of(timezone);
    }
}

// DO: Validate timezone IDs
public void setTimezone(String timezone) {
    try {
        ZoneId.of(timezone);  // Validate
        this.timezone = timezone;
    } catch (ZoneRulesException e) {
        throw new ValidationException("Invalid timezone: " + timezone);
    }
}
```

### ❌ DON'T: Ignore timezone or use offsets instead of zones

```java
// DON'T: Assume local timezone
LocalDateTime now = LocalDateTime.now();  // Server's timezone - varies by deployment!

// Better: Be explicit
Instant now = Instant.now();  // Always UTC
ZonedDateTime nowNY = ZonedDateTime.now(ZoneId.of("America/New_York"));

// DON'T: Use fixed offsets for locations
ZoneOffset pst = ZoneOffset.ofHours(-8);  // Doesn't handle DST!
// California is -8 in winter, -7 in summer

// Better: Use zone ID
ZoneId california = ZoneId.of("America/Los_Angeles");  // Handles DST

// DON'T: Abbreviations (ambiguous)
// "PST" = Pacific Standard Time? Philippine Standard Time? Pakistan Standard Time?
// "CST" = Central Standard Time? China Standard Time?

// DON'T: Store times without context
@Column private LocalDateTime eventTime;  // What zone is this?!
```

---

## 4. Database and JPA

### ✅ DO: Configure proper column types and converters

```java
// DO: Use appropriate column types
@Entity
public class Order {

    // TIMESTAMP WITH TIME ZONE in database
    @Column(name = "created_at", columnDefinition = "TIMESTAMP WITH TIME ZONE")
    private Instant createdAt;

    // DATE only in database
    @Column(name = "delivery_date", columnDefinition = "DATE")
    private LocalDate deliveryDate;

    // TIME only in database
    @Column(name = "preferred_delivery_time", columnDefinition = "TIME")
    private LocalTime preferredDeliveryTime;

    @PrePersist
    protected void onCreate() {
        createdAt = Instant.now();  // Always UTC
    }
}

// DO: Configure Hibernate for UTC
// application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          time_zone: UTC

// DO: Use Instant for audit columns
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

---

## 5. REST APIs

### ✅ DO: Use ISO 8601 format in APIs

```java
// DO: Use OffsetDateTime or Instant for API responses
public record OrderResponse(
    String id,
    Instant createdAt,  // "2024-03-15T14:30:00Z"
    LocalDate deliveryDate,  // "2024-03-20"
    OffsetDateTime estimatedArrival  // "2024-03-20T14:00:00-04:00"
) {}

// DO: Configure Jackson for ISO 8601
@Configuration
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
        return builder -> {
            builder.serializers(new LocalDateSerializer(DateTimeFormatter.ISO_LOCAL_DATE));
            builder.serializers(new InstantSerializer(InstantSerializer.INSTANCE, false,
                DateTimeFormatter.ISO_INSTANT));
            builder.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        };
    }
}

// application.yml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
    date-format: "yyyy-MM-dd'T'HH:mm:ss.SSSXXX"
    time-zone: UTC

// DO: Accept flexible input formats
public record CreateEventRequest(
    String title,

    @JsonFormat(pattern = "yyyy-MM-dd")
    LocalDate eventDate,

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ssXXX")
    OffsetDateTime startTime
) {}

// DO: Document timezone expectations in OpenAPI
@Schema(description = "Event start time in ISO 8601 format with timezone offset",
        example = "2024-03-15T14:30:00-04:00")
private OffsetDateTime startTime;
```

---

## 6. Common Calculations

### ✅ DO: Use built-in methods for date math

```java
// DO: Use plus/minus methods
LocalDate today = LocalDate.now();
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);

Instant now = Instant.now();
Instant oneHourLater = now.plus(Duration.ofHours(1));

// DO: Calculate differences
Period age = Period.between(birthDate, LocalDate.now());
int years = age.getYears();

Duration elapsed = Duration.between(startTime, endTime);
long minutes = elapsed.toMinutes();

// DO: Business day calculations
public LocalDate addBusinessDays(LocalDate start, int days) {
    LocalDate result = start;
    int addedDays = 0;

    while (addedDays < days) {
        result = result.plusDays(1);
        if (result.getDayOfWeek() != DayOfWeek.SATURDAY &&
            result.getDayOfWeek() != DayOfWeek.SUNDAY) {
            addedDays++;
        }
    }
    return result;
}

// DO: Check if date is in range
public boolean isWithinRange(LocalDate date, LocalDate start, LocalDate end) {
    return !date.isBefore(start) && !date.isAfter(end);
}

// DO: Get start/end of day
public Instant getStartOfDay(LocalDate date, ZoneId zone) {
    return date.atStartOfDay(zone).toInstant();
}

public Instant getEndOfDay(LocalDate date, ZoneId zone) {
    return date.atTime(LocalTime.MAX).atZone(zone).toInstant();
}
```

### ❌ DON'T: Manual calculations or magic numbers

```java
// DON'T: Manual milliseconds calculations
long oneDay = 24 * 60 * 60 * 1000;  // Error-prone, ignores DST!
Date tomorrow = new Date(System.currentTimeMillis() + oneDay);

// Better:
Instant tomorrow = Instant.now().plus(Duration.ofDays(1));

// DON'T: Magic numbers for months/years
int daysInYear = 365;  // What about leap years?

// Better: Let the API handle it
LocalDate nextYear = today.plusYears(1);
```
