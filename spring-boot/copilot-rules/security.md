# Security Best Practices

## 1. Spring Security 6.x Configuration

### ✅ DO: Use lambda DSL and component-based security

```java
// DO: Modern Spring Security 6.x configuration
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfiguration {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // Disable for stateless API
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider)
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler(new CustomAccessDeniedHandler())
            )
            .build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://myapp.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        configuration.setExposedHeaders(List.of("X-Total-Count"));
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}

// DO: Method-level security
@Service
@RequiredArgsConstructor
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(String userId) {
        userRepository.deleteById(userId);
    }

    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public User getUser(String userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", userId));
    }

    @PostAuthorize("returnObject.email == authentication.principal.email or hasRole('ADMIN')")
    public User getCurrentUser() {
        return userRepository.findById(getCurrentUserId()).orElseThrow();
    }
}
```

**When to use:**

- Lambda DSL: Always in Spring Security 6.x
- @PreAuthorize: Before method execution
- @PostAuthorize: After method execution (when return value needed)
- @Secured: Simple role checks (less flexible)

### ❌ DON'T: Use deprecated configuration patterns

```java
// DON'T: Deprecated WebSecurityConfigurerAdapter
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {  // DEPRECATED
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // Old pattern
    }
}

// DON'T: antMatchers (use requestMatchers in Spring Security 6)
.authorizeRequests()  // Deprecated
    .antMatchers("/api/**").authenticated()  // Deprecated

// Better: Use requestMatchers
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/**").authenticated()
)

// DON'T: Disable CSRF without understanding implications
http.csrf().disable();  // Only OK for stateless REST APIs

// DON'T: Permit all by default
.anyRequest().permitAll()  // Security hole!
```

---

## 2. JWT Authentication

### ✅ DO: Implement secure JWT handling

```java
// DO: JWT service with proper validation
@Service
@RequiredArgsConstructor
public class JwtService {

    private final JwtProperties jwtProperties;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtProperties.secret());
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        return generateToken(Map.of(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + jwtProperties.expiration().toMillis()))
            .signWith(getSigningKey())
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            final String username = extractUsername(token);
            return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
        } catch (JwtException e) {
            return false;
        }
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
}

// DO: JWT filter with proper error handling
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            final String jwt = authHeader.substring(7);
            final String username = jwtService.extractUsername(jwt);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            // Invalid token - continue without authentication
            log.debug("Invalid JWT token: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

### ❌ DON'T: Common JWT mistakes

```java
// DON'T: Store sensitive data in JWT payload
public String generateToken(User user) {
    return Jwts.builder()
        .claim("password", user.getPassword())  // NEVER!
        .claim("ssn", user.getSsn())            // NEVER!
        .build();
}

// DON'T: Use weak signing key
private static final String SECRET = "secret";  // Too short, predictable

// DON'T: Skip token validation
public String extractUsername(String token) {
    return Jwts.parser()
        .build()
        .parseSignedClaims(token)  // No signature verification!
        .getPayload()
        .getSubject();
}

// DON'T: Long-lived tokens without refresh
@Value("${jwt.expiration}")
private long expiration = 86400000 * 30;  // 30 days - too long!

// Better: Short access tokens + refresh tokens
private Duration accessTokenExpiration = Duration.ofMinutes(15);
private Duration refreshTokenExpiration = Duration.ofDays(7);
```

---

## 3. Password Security

### ✅ DO: Use strong password encoding

```java
// DO: BCrypt with strength
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // Work factor 12
}

// DO: Password validation
public record ChangePasswordRequest(
    @NotBlank String currentPassword,

    @NotBlank
    @Size(min = 8, max = 100)
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain uppercase, lowercase, number, and special character"
    )
    String newPassword
) {}

// DO: Use Argon2 for highest security
@Bean
public PasswordEncoder passwordEncoder() {
    return Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
}
```

### ❌ DON'T: Weak password handling

```java
// DON'T: Store plain text passwords
user.setPassword(rawPassword);  // NEVER!

// DON'T: MD5 or SHA1 for passwords
String hash = DigestUtils.md5Hex(password);  // Crackable!

// DON'T: Compare passwords with equals
if (user.getPassword().equals(inputPassword)) {  // Timing attack!
    // authenticate
}

// Better: Use encoder.matches()
if (passwordEncoder.matches(inputPassword, user.getPassword())) {
    // authenticate
}
```

---

## 4. Input Validation

### ✅ DO: Validate all inputs thoroughly

```java
// DO: Request validation with Bean Validation
public record CreateUserRequest(
    @NotBlank
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers, and underscores")
    String username,

    @NotBlank
    @Email
    String email,

    @NotBlank
    @Size(min = 8, max = 100)
    String password,

    @NotNull
    @Past
    LocalDate birthDate
) {}

// DO: Sanitize HTML content
@Service
@RequiredArgsConstructor
public class ContentService {

    private final PolicyFactory sanitizer = Sanitizers.FORMATTING.and(Sanitizers.LINKS);

    public String sanitizeHtml(String untrustedHtml) {
        return sanitizer.sanitize(untrustedHtml);
    }
}

// DO: Validate path parameters
@GetMapping("/files/{filename}")
public Resource getFile(
        @PathVariable
        @Pattern(regexp = "^[a-zA-Z0-9._-]+$")
        String filename) {
    // Filename is validated - no path traversal
    return fileService.getFile(filename);
}

// DO: Validate file uploads
public void uploadFile(MultipartFile file) {
    // Check content type
    String contentType = file.getContentType();
    if (!ALLOWED_TYPES.contains(contentType)) {
        throw new ValidationException("Invalid file type");
    }

    // Check file size
    if (file.getSize() > MAX_FILE_SIZE) {
        throw new ValidationException("File too large");
    }

    // Check actual content (magic bytes)
    if (!isValidFileContent(file.getBytes())) {
        throw new ValidationException("Invalid file content");
    }
}
```

### ❌ DON'T: Trust user input

```java
// DON'T: Path traversal vulnerability
@GetMapping("/files")
public Resource getFile(@RequestParam String path) {
    return new FileSystemResource(basePath + path);  // Path traversal!
    // Attacker can request: ?path=../../etc/passwd
}

// DON'T: SQL injection
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = '" + username + "'";
    // SQL injection!
}

// Better: Parameterized query
@Query("SELECT u FROM User u WHERE u.username = :username")
Optional<User> findByUsername(@Param("username") String username);

// DON'T: Log injection
log.info("User logged in: " + userInput);  // Log injection!

// Better: Use parameterized logging
log.info("User logged in: {}", userInput);

// DON'T: Trust Content-Type header for file uploads
if (file.getContentType().equals("image/jpeg")) {
    // Content-Type can be spoofed!
}
```

---

## 5. HTTPS and Headers

### ✅ DO: Configure security headers

```java
// DO: Security headers configuration
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'"))
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
            .contentTypeOptions(Customizer.withDefaults())
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicyHeader(permissions -> permissions
                .policy("geolocation=(), microphone=(), camera=()"))
        )
        .build();
}

// DO: Force HTTPS in production
@Bean
@Profile("prod")
public SecurityFilterChain httpsFilterChain(HttpSecurity http) throws Exception {
    return http
        .requiresChannel(channel -> channel
            .anyRequest().requiresSecure())
        .headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)))
        .build();
}
```

**application.yml for production:**

```yaml
server:
  ssl:
    enabled: true
    key-store: ${SSL_KEYSTORE_PATH}
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
  http2:
    enabled: true
```

---

## 6. Rate Limiting

### ✅ DO: Implement rate limiting

```java
// DO: Rate limiting with Bucket4j
@Configuration
public class RateLimitConfiguration {

    @Bean
    public CacheManager<String, BucketConfiguration> bucketCacheManager() {
        return Caffeine.newBuilder()
            .maximumSize(100_000)
            .build(this::buildDefaultBucket);
    }

    private BucketConfiguration buildDefaultBucket(String key) {
        return BucketConfiguration.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
            .build();
    }
}

@Component
@RequiredArgsConstructor
public class RateLimitFilter extends OncePerRequestFilter {

    private final ProxyManager<String> buckets;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain chain) throws ServletException, IOException {
        String key = getClientIdentifier(request);
        Bucket bucket = buckets.builder().build(key, () -> defaultConfiguration());

        if (bucket.tryConsume(1)) {
            response.addHeader("X-Rate-Limit-Remaining",
                String.valueOf(bucket.getAvailableTokens()));
            chain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded");
        }
    }

    private String getClientIdentifier(HttpServletRequest request) {
        String forwarded = request.getHeader("X-Forwarded-For");
        return forwarded != null ? forwarded.split(",")[0] : request.getRemoteAddr();
    }
}
```

### ❌ DON'T: No rate limiting on sensitive endpoints

```java
// DON'T: Unprotected login endpoint
@PostMapping("/login")
public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest request) {
    // No rate limiting - brute force attack possible!
    return authService.authenticate(request);
}

// DON'T: Unprotected password reset
@PostMapping("/reset-password")
public ResponseEntity<Void> resetPassword(@RequestBody ResetRequest request) {
    // No rate limiting - email bombing possible!
    return passwordService.resetPassword(request);
}
```
