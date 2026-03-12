# Spring Boot Interview Questions & Answers — Part 2
## Advanced Topics for Senior/Architect Level

---

## Section 1: Spring Boot Internals & Auto-Configuration

### Q1. How does Spring Boot Auto-Configuration work internally?

**Answer:**

Auto-configuration is the core of Spring Boot magic — it conditionally creates beans based on classpath, properties, and existing beans.

```
Startup sequence:
  1. SpringApplication.run() starts
  2. SpringFactoriesLoader reads:
       META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
       (Spring Boot 2.7+, was META-INF/spring.factories before)
  3. Loads all AutoConfiguration classes
  4. Each class evaluated with @Conditional annotations
  5. Only conditions that pass → beans created
```

**Example — DataSource auto-config:**
```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration {}

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class,
              DataSourceConfiguration.Tomcat.class, ... })
    protected static class PooledDataSourceConfiguration {}
}
```

**Key conditionals:**
| Annotation | Condition |
|-----------|-----------|
| `@ConditionalOnClass` | Class present on classpath |
| `@ConditionalOnMissingBean` | Bean NOT already defined |
| `@ConditionalOnProperty` | Property equals specified value |
| `@ConditionalOnBean` | Specific bean exists |
| `@ConditionalOnWebApplication` | Running as web app |
| `@ConditionalOnResource` | File/resource exists |

**Debug auto-configuration:**
```bash
# Add to application.properties
debug=true
# Or run with:
java -jar app.jar --debug
# Shows CONDITIONS EVALUATION REPORT
```

---

### Q2. What is the difference between `@SpringBootApplication` and its constituent annotations?

**Answer:**

`@SpringBootApplication` is a composed annotation combining three:

```java
@SpringBootApplication
// is equivalent to:
@SpringBootConfiguration   // = @Configuration (marks as config class)
@EnableAutoConfiguration   // triggers auto-configuration
@ComponentScan             // scans current package + sub-packages
public class MyApp {}
```

**When to split them:**
```java
// Exclude specific auto-configurations
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})

// Scan specific packages (useful for modular architecture)
@SpringBootApplication
@ComponentScan(basePackages = {"com.company.app", "com.company.shared"})
public class MyApp {}

// In tests — disable auto-config for faster tests
@SpringBootTest
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
class MyServiceTest {}
```

---

### Q3. Explain Spring Boot's condition evaluation order and how to create a custom `@Conditional`.

**Answer:**

**Condition evaluation order matters** — Spring processes conditions in phases:

```
Phase 1: PARSE_CONFIGURATION
  @ConditionalOnClass / @ConditionalOnMissingClass
  (classpath check — before any beans are created)

Phase 2: REGISTER_BEAN
  @ConditionalOnBean / @ConditionalOnMissingBean
  (after other @Configuration classes processed)

Phase 3: (always)
  @ConditionalOnProperty
  @ConditionalOnResource
  @ConditionalOnExpression
```

**Custom @Conditional example:**
```java
// Step 1: Create condition
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String os = context.getEnvironment().getProperty("os.name");
        return os != null && os.toLowerCase().contains("linux");
    }
}

// Step 2: Create annotation
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnLinuxCondition.class)
public @interface ConditionalOnLinux {}

// Step 3: Use it
@Bean
@ConditionalOnLinux
public FileSystemService linuxFileSystemService() {
    return new LinuxFileSystemService();
}
```

---

### Q4. How does Spring Boot's `Environment` and `PropertySource` hierarchy work?

**Answer:**

Spring Boot loads properties in a specific priority order (higher overrides lower):

```
Priority (highest to lowest):
  1.  Command-line arguments       --server.port=9090
  2.  @TestPropertySource          (tests only)
  3.  SPRING_APPLICATION_JSON      (env var with inline JSON)
  4.  ServletConfig/Context params (web apps)
  5.  JNDI java:comp/env
  6.  System.getProperties()       (JVM -D flags)
  7.  OS environment variables     SERVER_PORT=9090
  8.  Profile-specific files       application-prod.properties
  9.  application.properties/yaml  inside jar
  10. application.properties/yaml  outside jar (./config/ or ./)
  11. @PropertySource annotations  on @Configuration classes
  12. Default properties           SpringApplication.setDefaultProperties()
```

**Custom PropertySource:**
```java
@Configuration
public class VaultPropertySourceConfig
    implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(
        ConfigurableEnvironment environment,
        SpringApplication application) {

        // Load secrets from Vault
        Map<String, Object> secrets = vaultClient.getSecrets();
        MapPropertySource vaultSource =
            new MapPropertySource("vault", secrets);

        // Add with highest priority
        environment.getPropertySources().addFirst(vaultSource);
    }
}

// Register in META-INF/spring.factories:
// org.springframework.boot.env.EnvironmentPostProcessor=com.example.VaultPropertySourceConfig
```

---

### Q5. What is Spring Boot's `ApplicationContext` refresh lifecycle and what events are fired?

**Answer:**

```
SpringApplication.run() lifecycle:

1. ApplicationStartingEvent
   └── App starting, no context yet

2. ApplicationEnvironmentPreparedEvent
   └── Environment ready, before context created
   └── EnvironmentPostProcessors run here

3. ApplicationContextInitializedEvent
   └── Context created, initializers called, not yet refreshed

4. ApplicationPreparedEvent
   └── Context loaded (bean definitions), not yet refreshed

5. ContextRefreshedEvent (standard Spring)
   └── All beans instantiated and wired

6. ApplicationStartedEvent
   └── Context refreshed, before runners called

7. ApplicationReadyEvent
   └── App ready to serve requests
   └── CommandLineRunner / ApplicationRunner execute here

8. ApplicationFailedEvent (if startup fails)
   └── Exception during startup
```

**Listening to events:**
```java
@Component
public class StartupListener implements ApplicationListener<ApplicationReadyEvent> {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("App is ready! Start serving traffic.");
    }
}

// Or with @EventListener (Spring 4.2+)
@Component
public class EventListeners {
    @EventListener
    public void onReady(ApplicationReadyEvent event) {
        // warm up caches, validate connections, etc.
    }

    @EventListener
    public void onContextRefresh(ContextRefreshedEvent event) {
        // runs after every context refresh (including child contexts)
    }
}
```

---

## Section 2: Spring Data & Database

### Q6. How does Spring Data JPA's query derivation work and what are its limits?

**Answer:**

Spring Data parses method names at startup to build JPQL/SQL queries:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // SELECT u FROM User u WHERE u.email = ?1
    Optional<User> findByEmail(String email);

    // SELECT u FROM User u WHERE u.firstName = ?1 AND u.lastName = ?2
    List<User> findByFirstNameAndLastName(String first, String last);

    // SELECT u FROM User u WHERE u.age > ?1 ORDER BY u.lastName ASC
    List<User> findByAgeGreaterThanOrderByLastNameAsc(int age);

    // SELECT COUNT(u) FROM User u WHERE u.active = true
    long countByActiveTrue();

    // DELETE FROM User u WHERE u.status = ?1
    @Modifying
    @Transactional
    void deleteByStatus(String status);

    // EXISTS query
    boolean existsByEmail(String email);
}
```

**Supported keywords:**

```
Comparison:   Is, Equals, Not, LessThan, GreaterThan, Between
Null checks:  IsNull, IsNotNull
Like:         Like, NotLike, StartingWith, EndingWith, Containing
Boolean:      True, False
Collection:   In, NotIn
Order:        OrderByXxxAsc, OrderByXxxDesc
Limiting:     findFirst3By..., findTop5By...
```

**Limits of query derivation:**
- Method names become unreadable for complex queries
- No aggregation (SUM, AVG, GROUP BY)
- No JOIN across multiple entities easily
- No native SQL

**Solution — use `@Query`:**
```java
@Query("SELECT u FROM User u WHERE u.department.name = :dept AND u.salary > :minSalary")
List<User> findByDeptAndMinSalary(@Param("dept") String dept, @Param("minSalary") BigDecimal min);

// Native SQL
@Query(value = "SELECT * FROM users WHERE MATCH(bio) AGAINST (:term)", nativeQuery = true)
List<User> fullTextSearch(@Param("term") String term);
```

---

### Q7. Explain N+1 query problem in JPA and all the ways to solve it.

**Answer:**

**The problem:**
```java
// Order has a List<OrderItem> (LAZY)
List<Order> orders = orderRepo.findAll();  // 1 query: SELECT * FROM orders
for (Order o : orders) {
    o.getItems().size();  // N queries: SELECT * FROM order_items WHERE order_id = ?
}
// Total: 1 + N queries → N+1 problem
```

**Solutions:**

**1. JPQL JOIN FETCH:**
```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findWithItems(@Param("status") String status);
```

**2. `@EntityGraph`:**
```java
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByStatus(String status);
```

**3. Batch size (for collections):**
```java
@Entity
public class Order {
    @OneToMany
    @BatchSize(size = 25)  // SELECT WHERE order_id IN (25 IDs at a time)
    private List<OrderItem> items;
}

# Or globally in application.properties:
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

**4. DTO projection:**
```java
// Only fetch what you need
@Query("SELECT new com.example.OrderSummary(o.id, o.status, COUNT(i)) " +
       "FROM Order o LEFT JOIN o.items i GROUP BY o.id, o.status")
List<OrderSummary> findOrderSummaries();
```

**5. Hibernate `@Fetch(FetchMode.SUBSELECT)`:**
```java
@OneToMany
@Fetch(FetchMode.SUBSELECT)
// Uses: SELECT * FROM order_items WHERE order_id IN (SELECT id FROM orders WHERE ...)
private List<OrderItem> items;
```

**Detect N+1 in testing:**
```java
// Use datasource-proxy or Hibernate statistics
@Test
void shouldNotHaveNPlus1() {
    // Enable SQL logging in test
    // Count queries and assert
}

# In application.properties (for dev)
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.generate_statistics=true
```

---

### Q8. How do you implement optimistic and pessimistic locking in Spring Data JPA?

**Answer:**

**Optimistic Locking** — no DB lock, detects conflicts at commit time:
```java
@Entity
public class BankAccount {
    @Id
    private Long id;
    private BigDecimal balance;

    @Version  // Hibernate manages this automatically
    private Long version;  // or Timestamp
}

// Usage — Spring Data handles version check automatically
// If two transactions read version=5, both increment balance,
// second commit throws OptimisticLockException (version mismatch)
@Transactional
public void transfer(Long fromId, BigDecimal amount) {
    BankAccount account = repo.findById(fromId).orElseThrow();
    account.debit(amount);
    repo.save(account);  // fails with OptimisticLockException if stale
}

// Retry on conflict
@Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void transferWithRetry(Long fromId, BigDecimal amount) { ... }
```

**Pessimistic Locking** — acquires DB lock immediately:
```java
public interface AccountRepository extends JpaRepository<BankAccount, Long> {

    // SELECT ... FOR UPDATE (blocks other transactions)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<BankAccount> findById(Long id);

    // SELECT ... FOR SHARE (allows concurrent reads)
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT a FROM BankAccount a WHERE a.id = :id")
    Optional<BankAccount> findByIdForRead(@Param("id") Long id);

    // With timeout to avoid indefinite blocking
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "3000"))
    Optional<BankAccount> findByIdWithTimeout(Long id);
}
```

**When to use which:**
| | Optimistic | Pessimistic |
|-|------------|-------------|
| Conflict rate | Low | High |
| Lock duration | None (check at commit) | Throughout transaction |
| Throughput | High | Lower |
| Risk | Lost updates | Deadlocks |
| Use case | Read-heavy, occasional writes | Financial transactions, inventory |

---

### Q9. How does Spring's `@Transactional` work and what are common pitfalls?

**Answer:**

**How it works:**
```
@Transactional uses Spring AOP proxy:

External call → Proxy intercepts → begins transaction → calls real method → commits/rollbacks → returns
Internal call (this.method()) → bypasses proxy → NO transaction!
```

**Propagation types:**
```java
@Transactional(propagation = Propagation.REQUIRED)        // default: join existing or create new
@Transactional(propagation = Propagation.REQUIRES_NEW)    // always new (suspends outer)
@Transactional(propagation = Propagation.SUPPORTS)        // join if exists, none if not
@Transactional(propagation = Propagation.NOT_SUPPORTED)   // suspend outer, run without tx
@Transactional(propagation = Propagation.MANDATORY)       // must have outer tx (else exception)
@Transactional(propagation = Propagation.NEVER)           // must NOT have tx (else exception)
@Transactional(propagation = Propagation.NESTED)          // savepoint within outer tx
```

**Common pitfalls:**

```java
// PITFALL 1: Self-invocation (most common!)
@Service
public class OrderService {
    public void processOrders(List<Long> ids) {
        ids.forEach(id -> this.processOne(id));  // ← bypasses proxy, NO transaction!
    }

    @Transactional
    public void processOne(Long id) { ... }
}

// FIX: Inject self or extract to separate bean
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // self-injection via Spring proxy

    public void processOrders(List<Long> ids) {
        ids.forEach(id -> self.processOne(id));  // ← goes through proxy ✓
    }
}

// PITFALL 2: Wrong exception type for rollback
@Transactional  // Only rolls back on RuntimeException by default!
public void save(Order order) throws IOException {
    // IOException is checked → NO rollback unless specified
}

// FIX:
@Transactional(rollbackFor = Exception.class)
// or
@Transactional(rollbackFor = { IOException.class, SQLException.class })

// PITFALL 3: @Transactional on private methods
@Transactional  // IGNORED — Spring AOP can't proxy private methods
private void internalSave() { ... }

// PITFALL 4: Transaction isolation mismatch
@Transactional(isolation = Isolation.SERIALIZABLE)  // may cause deadlocks at scale
```

---

### Q10. What is the difference between Spring Data's `save()`, `saveAndFlush()`, and `saveAll()`?

**Answer:**

```java
// save(entity)
// - If entity is new (null/0 id): calls persist()
// - If entity exists: calls merge()
// - Does NOT immediately flush to DB (waits for transaction commit or explicit flush)
User saved = userRepo.save(user);

// saveAndFlush(entity)
// - Same as save() but immediately executes SQL
// - Useful when you need DB-generated values (triggers, sequences) before tx ends
// - Or when subsequent queries in same tx need to see the change
User saved = userRepo.saveAndFlush(user);

// saveAll(Iterable)
// - Calls save() for each entity
// - In a loop — still has N+1 if not batched
List<User> saved = userRepo.saveAll(users);

// For bulk inserts — configure batch:
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

**Determining if entity is "new":**
```java
// Spring Data uses Persistable<ID> or checks if ID is null
// Custom logic:
@Entity
public class Order implements Persistable<String> {
    @Id
    private String id = UUID.randomUUID().toString();  // Always set

    @Transient
    private boolean isNew = true;

    @PostPersist
    @PostLoad
    void markNotNew() { this.isNew = false; }

    @Override
    public boolean isNew() { return isNew; }
}
```

---

## Section 3: Spring Security

### Q11. How does Spring Security's filter chain work?

**Answer:**

Every HTTP request passes through a chain of security filters before reaching your controller:

```
Request
  ↓
DisableEncodeUrlFilter
  ↓
WebAsyncManagerIntegrationFilter
  ↓
SecurityContextHolderFilter  ← loads SecurityContext from session/JWT
  ↓
HeaderWriterFilter           ← adds security headers (X-Frame-Options, etc.)
  ↓
CorsFilter                   ← handles CORS preflight
  ↓
CsrfFilter                   ← validates CSRF token
  ↓
LogoutFilter                 ← handles /logout
  ↓
UsernamePasswordAuthenticationFilter  ← processes /login (form login)
  ↓
BearerTokenAuthenticationFilter       ← validates JWT (if OAuth2 resource server)
  ↓
BasicAuthenticationFilter            ← HTTP Basic auth
  ↓
RequestCacheAwareFilter
  ↓
SecurityContextHolderAwareRequestFilter
  ↓
AnonymousAuthenticationFilter        ← sets anonymous auth if not yet authenticated
  ↓
ExceptionTranslationFilter           ← converts Spring Security exceptions to HTTP responses
  ↓
AuthorizationFilter                  ← checks permissions (replaces FilterSecurityInterceptor)
  ↓
Controller / Resource
```

**Multiple filter chains (different rules per path):**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

---

### Q12. How do you implement JWT authentication in Spring Boot?

**Answer:**

```java
// Dependencies: spring-boot-starter-security, spring-boot-starter-oauth2-resource-server

// application.yml (for external JWKS endpoint)
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://auth-server/.well-known/jwks.json
          # OR for symmetric key:
          # secret: my-256-bit-secret

// Security config
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter())));
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtConverter() {
        JwtGrantedAuthoritiesConverter authConverter = new JwtGrantedAuthoritiesConverter();
        authConverter.setAuthoritiesClaimName("roles");
        authConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authConverter);
        return converter;
    }
}

// Generate JWT (for self-issued tokens)
@Service
public class JwtService {
    @Value("${jwt.secret}")
    private String secret;

    public String generateToken(UserDetails user) {
        return Jwts.builder()
            .setSubject(user.getUsername())
            .claim("roles", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).collect(Collectors.toList()))
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(1, ChronoUnit.HOURS)))
            .signWith(Keys.hmacShaKeyFor(secret.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }
}
```

---

### Q13. What is the difference between `@PreAuthorize`, `@Secured`, and `@RolesAllowed`?

**Answer:**

| Annotation | Source | SpEL support | Flexibility |
|-----------|--------|-------------|-------------|
| `@Secured` | Spring Security | No | Basic role check |
| `@RolesAllowed` | JSR-250 (javax) | No | Standard Java EE |
| `@PreAuthorize` | Spring Security | Yes | Full expression power |
| `@PostAuthorize` | Spring Security | Yes | Check return value |
| `@PreFilter` | Spring Security | Yes | Filter input collection |
| `@PostFilter` | Spring Security | Yes | Filter output collection |

```java
// Enable method security
@Configuration
@EnableMethodSecurity  // Spring Boot 3.x (replaces @EnableGlobalMethodSecurity)
public class MethodSecurityConfig {}

// @Secured — simple role check
@Secured("ROLE_ADMIN")
public void deleteUser(Long id) {}

@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
public List<User> getAllUsers() {}

// @PreAuthorize — SpEL expressions
@PreAuthorize("hasRole('ADMIN')")
public void adminOnly() {}

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER') and #user.department == authentication.principal.department")
public void updateUser(User user) {}

// Check method parameter
@PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
public User getUser(Long userId) {}

// @PostAuthorize — check return value
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) {}

// @PreFilter / @PostFilter — filter collections
@PostFilter("filterObject.owner == authentication.name")
public List<Document> getAllDocuments() {}

// Custom security expression
@PreAuthorize("@documentSecurityService.canRead(#id, authentication)")
public Document getDocument(Long id) {}
```

---

### Q14. How do you implement OAuth2 login (social login) in Spring Boot?

**Answer:**

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/error").permitAll()
                .anyRequest().authenticated())
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())))
            .logout(logout -> logout.logoutSuccessUrl("/"));
        return http.build();
    }

    @Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
}

// Custom user service — map OAuth2 user to your domain user
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    @Autowired private UserRepository userRepo;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest request) throws OAuth2AuthenticationException {
        OAuth2User oauth2User = super.loadUser(request);

        String email = oauth2User.getAttribute("email");
        String provider = request.getClientRegistration().getRegistrationId(); // "google" / "github"

        User user = userRepo.findByEmail(email)
            .orElseGet(() -> createNewUser(oauth2User, provider));

        return new CustomUserPrincipal(user, oauth2User.getAttributes());
    }
}
```

---

## Section 4: Spring Boot Testing

### Q15. What is the difference between `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, and `@JsonTest`?

**Answer:**

```
@SpringBootTest
  - Loads FULL application context
  - All beans, auto-configurations, embedded server (optional)
  - Use for: integration tests, end-to-end tests
  - Slow startup
  - webEnvironment options: MOCK (default), RANDOM_PORT, DEFINED_PORT, NONE

@WebMvcTest(MyController.class)
  - Loads ONLY web layer (controllers, filters, @ControllerAdvice)
  - No service beans, no DB — must mock them with @MockBean
  - Fast — only web slice
  - Use for: controller unit tests

@DataJpaTest
  - Loads ONLY JPA layer (entities, repositories)
  - Uses in-memory DB (H2) by default
  - @Transactional — each test rolled back
  - No web, no services
  - Use for: repository tests

@JsonTest
  - Loads ONLY Jackson / JSON configuration
  - Tests JSON serialization/deserialization
  - JacksonTester, GsonTester, BasicJsonTester available

@DataRedisTest       — Redis repositories only
@DataMongoTest       — MongoDB repositories only
@RestClientTest      — RestTemplate / RestClient tests
@WebFluxTest         — WebFlux controllers only
```

**Examples:**
```java
// Controller test
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;  // must mock all deps

    @Test
    void shouldReturn200() throws Exception {
        when(userService.findById(1L)).thenReturn(new User("Alice"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}

// Repository test
@DataJpaTest
class UserRepositoryTest {
    @Autowired TestEntityManager em;
    @Autowired UserRepository repo;

    @Test
    void shouldFindByEmail() {
        em.persistAndFlush(new User("alice@example.com", "Alice"));
        Optional<User> found = repo.findByEmail("alice@example.com");
        assertThat(found).isPresent();
    }
}

// Integration test
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {
    @Autowired TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/orders", new CreateOrderRequest("item1", 2), Order.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

---

### Q16. How do you test async methods and scheduled tasks in Spring Boot?

**Answer:**

**Testing `@Async` methods:**
```java
// Problem: @Async returns CompletableFuture — need to await
@Service
public class EmailService {
    @Async
    public CompletableFuture<String> sendEmail(String to) {
        // ... send email
        return CompletableFuture.completedFuture("sent");
    }
}

@SpringBootTest
class EmailServiceTest {
    @Autowired EmailService emailService;

    @Test
    void shouldSendEmail() throws Exception {
        CompletableFuture<String> result = emailService.sendEmail("test@example.com");
        // Wait for async result
        String status = result.get(5, TimeUnit.SECONDS);
        assertThat(status).isEqualTo("sent");
    }
}

// For unit testing async (disable @Async)
@SpringBootTest
@EnableAsync(mode = AdviceMode.PROXY)
// Override async executor to run synchronously
@TestConfiguration
class TestConfig {
    @Bean
    public Executor taskExecutor() {
        return Runnable::run;  // synchronous executor for tests
    }
}
```

**Testing `@Scheduled`:**
```java
// Option 1: Test the underlying method directly (not the schedule)
@SpringBootTest
class ReportJobTest {
    @Autowired ReportScheduler scheduler;

    @Test
    void shouldGenerateReport() {
        scheduler.generateDailyReport();  // call directly
        // assert report was generated
    }
}

// Option 2: Use Awaitility for timing-based tests
@SpringBootTest
class ScheduledTaskTest {
    @Autowired ReportRepository reportRepo;

    @Test
    void shouldRunEvery5Seconds() {
        await().atMost(10, SECONDS)
               .until(() -> reportRepo.count() > 0);
    }
}
```

---

### Q17. How do you use `@MockBean` vs `@SpyBean` vs Mockito's `@Mock`?

**Answer:**

| Annotation | Context | Behavior |
|-----------|---------|----------|
| `@Mock` (Mockito) | Unit test (no Spring) | Creates mock, no Spring context |
| `@MockBean` (Spring) | Spring context test | Creates mock, replaces bean in Spring context |
| `@SpyBean` (Spring) | Spring context test | Wraps real bean, can stub specific methods |
| `@Spy` (Mockito) | Unit test (no Spring) | Wraps real object, partial mocking |

```java
// @MockBean — completely replaces the bean
@SpringBootTest
class OrderServiceTest {
    @Autowired OrderService orderService;
    @MockBean PaymentService paymentService;  // real PaymentService replaced

    @Test
    void shouldHandlePaymentFailure() {
        when(paymentService.charge(any())).thenThrow(new PaymentException("declined"));
        assertThrows(OrderException.class, () -> orderService.createOrder(order));
    }
}

// @SpyBean — real bean but can stub specific methods
@SpringBootTest
class EmailServiceTest {
    @SpyBean EmailService emailService;  // real EmailService, but can override

    @Test
    void shouldNotSendInTestEnv() {
        doNothing().when(emailService).sendEmail(anyString());  // stub specific method
        // rest of the bean works normally
        emailService.processNewUser(user);
        verify(emailService).sendEmail(user.getEmail());
    }
}

// @Mock — no Spring context, fastest
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository userRepo;
    @InjectMocks UserService userService;

    @Test
    void shouldCreateUser() {
        when(userRepo.save(any())).thenReturn(new User("Alice"));
        User result = userService.create("Alice");
        assertThat(result.getName()).isEqualTo("Alice");
    }
}
```

---

### Q18. How do you test Spring Boot applications with Testcontainers?

**Answer:**

**Testcontainers** spins up real Docker containers for integration tests.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// Spring Boot 3.1+ — @ServiceConnection (automatic config)
@SpringBootTest
@Testcontainers
class UserRepositoryIT {

    @Container
    @ServiceConnection  // auto-configures spring.datasource.*
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15-alpine");

    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer("redis:7-alpine");

    @Autowired UserRepository userRepo;

    @Test
    void shouldPersistUser() {
        User user = userRepo.save(new User("alice@example.com"));
        assertThat(user.getId()).isNotNull();
        assertThat(userRepo.findByEmail("alice@example.com")).isPresent();
    }
}

// Reuse container across tests (faster)
@SpringBootTest
class BaseIntegrationTest {
    static PostgreSQLContainer<?> postgres;

    static {
        postgres = new PostgreSQLContainer<>("postgres:15")
            .withReuse(true);  // reuse if container already running
        postgres.start();
    }

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

## Section 5: Reactive Programming

### Q19. What is the difference between `Mono` and `Flux` in Project Reactor?

**Answer:**

```
Mono<T>  = 0 or 1 element (async Optional)
Flux<T>  = 0 to N elements (async Stream)
```

```java
// Mono examples
Mono<User> userMono = userRepo.findById(1L);           // 0 or 1 user
Mono<Void> deleteMono = userRepo.deleteById(1L);       // completion signal
Mono<String> justMono = Mono.just("hello");            // create with value
Mono<String> emptyMono = Mono.empty();                 // no value
Mono<String> errorMono = Mono.error(new RuntimeException("oops"));

// Flux examples
Flux<User> allUsers = userRepo.findAll();              // many users
Flux<Integer> numbers = Flux.range(1, 100);            // 1 to 100
Flux<String> names = Flux.just("Alice", "Bob", "Charlie");
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1)); // infinite stream

// Common operators
userMono
    .map(user -> user.getName().toUpperCase())          // transform
    .filter(name -> name.startsWith("A"))               // filter
    .flatMap(name -> fetchDetails(name))                // async transform
    .switchIfEmpty(Mono.just("Anonymous"))              // default if empty
    .onErrorReturn("Error occurred")                    // error handling
    .timeout(Duration.ofSeconds(5))                     // timeout
    .subscribeOn(Schedulers.boundedElastic());          // which thread

// Chain Mono and Flux
Flux<User> activeUsers = userRepo.findAll()
    .filter(User::isActive)
    .flatMap(user -> enrichWithProfile(user))          // parallel async enrichment
    .collectList()                                      // Flux<User> → Mono<List<User>>
    .flatMapMany(Flux::fromIterable);                  // back to Flux
```

---

### Q20. How do you handle backpressure in Spring WebFlux?

**Answer:**

**Backpressure** = consumer controls the rate it receives data from producer.

```java
// Subscriber requesting data in chunks
Flux.range(1, 1000)
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(10);  // initially request 10
        }

        @Override
        protected void hookOnNext(Integer value) {
            process(value);
            if (valueCount % 10 == 0) {
                request(10);  // request next batch
            }
        }
    });

// Backpressure strategies for overflow
Flux.range(1, 10000)
    .onBackpressureBuffer(500)      // buffer up to 500, then error
    .onBackpressureDrop()           // drop items if consumer is slow
    .onBackpressureLatest()         // keep only latest item
    .onBackpressureError();         // throw OverflowException immediately

// Rate limiting with limitRate
Flux.range(1, 1000)
    .limitRate(50)   // upstream sends max 50 at a time (75% replenish threshold)
    .subscribe(this::processItem);

// WebFlux SSE (Server-Sent Events) with backpressure
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamEvents() {
    return Flux.interval(Duration.ofMillis(100))
        .map(seq -> ServerSentEvent.<String>builder()
            .id(String.valueOf(seq))
            .data("Event-" + seq)
            .build())
        .onBackpressureDrop();  // drop if client is slow
}
```

---

## Section 6: Microservices Patterns

### Q21. How do you implement the Circuit Breaker pattern with Resilience4j in Spring Boot?

**Answer:**

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10           # last 10 calls
        failure-rate-threshold: 50        # open if 50% fail
        wait-duration-in-open-state: 30s  # stay open 30s
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-rate-threshold: 80      # slow calls also count as failures
        slow-call-duration-threshold: 2s
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - feign.FeignException
  timelimiter:
    instances:
      paymentService:
        timeout-duration: 3s
```

```java
@Service
public class OrderService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
            paymentClient.charge(request));
    }

    // Fallback — same signature + Throwable
    public CompletableFuture<PaymentResult> fallbackPayment(
        PaymentRequest request, Throwable ex) {
        log.warn("Payment service unavailable, queuing for retry: {}", ex.getMessage());
        paymentQueue.add(request);  // async queue for retry
        return CompletableFuture.completedFuture(PaymentResult.PENDING);
    }
}

// Monitor circuit breaker state
@EventListener
public void onStateChange(CircuitBreakerOnStateTransitionEvent event) {
    log.warn("Circuit breaker {} state: {} → {}",
        event.getCircuitBreakerName(),
        event.getStateTransition().getFromState(),
        event.getStateTransition().getToState());
    // alert on OPEN state
}
```

---

### Q22. How do you implement distributed tracing with Spring Boot and OpenTelemetry?

**Answer:**

```xml
<!-- Spring Boot 3.x uses Micrometer Tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (use 0.1 for 10% in prod)
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces  # or Cloud Trace endpoint

spring:
  application:
    name: order-service  # appears as service name in traces
```

```java
// Automatic: Spring Boot auto-instruments:
//   - HTTP requests (incoming + outgoing via RestTemplate/WebClient)
//   - @Transactional methods
//   - Scheduled tasks
//   - Kafka/RabbitMQ messages

// Manual span creation
@Service
public class InventoryService {

    private final Tracer tracer;

    public InventoryService(Tracer tracer) {
        this.tracer = tracer;
    }

    public void reserveStock(String itemId, int qty) {
        Span span = tracer.nextSpan().name("reserve-stock").start();
        span.tag("item.id", itemId);
        span.tag("quantity", String.valueOf(qty));

        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            // business logic
            doReserveStock(itemId, qty);
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}

// MDC integration — traceId in logs automatically
// log output: 2024-01-01 [order-service,abc123def456,abc123def456] INFO ...
```

---

### Q23. How do you implement the Saga pattern for distributed transactions in Spring Boot?

**Answer:**

**Saga** = sequence of local transactions, each publishes event or message triggering next step. On failure, compensating transactions undo completed steps.

**Two styles:**
```
Choreography: each service publishes events, others react
  OrderService → publishes OrderCreated
    → PaymentService listens → charges card → publishes PaymentCompleted
      → InventoryService listens → reserves stock → publishes StockReserved
        → ShippingService listens → creates shipment

Orchestration: central coordinator drives the saga
  OrderSaga (orchestrator) → calls PaymentService
    → success: calls InventoryService
      → success: calls ShippingService
        → failure: calls InventoryService.release(), PaymentService.refund()
```

**Choreography with Spring Events + Kafka:**
```java
// Order Service
@Service
@Transactional
public class OrderService {
    @Autowired KafkaTemplate<String, Object> kafka;

    public Order createOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));  // local TX
        kafka.send("order-created", new OrderCreatedEvent(order.getId(), order.getTotal()));
        return order;
    }

    // Compensating transaction
    @KafkaListener(topics = "payment-failed")
    public void onPaymentFailed(PaymentFailedEvent event) {
        orderRepo.updateStatus(event.getOrderId(), OrderStatus.FAILED);
    }
}

// Payment Service listens and reacts
@KafkaListener(topics = "order-created")
@Transactional
public void onOrderCreated(OrderCreatedEvent event) {
    try {
        paymentGateway.charge(event.getOrderId(), event.getAmount());
        kafka.send("payment-completed", new PaymentCompletedEvent(event.getOrderId()));
    } catch (Exception e) {
        kafka.send("payment-failed", new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
    }
}
```

---

### Q24. What is the Outbox Pattern and how do you implement it in Spring Boot?

**Answer:**

**Problem:** Write to DB and publish message atomically. If service crashes between DB commit and Kafka send, the message is lost.

**Outbox Pattern:**
```
1. Write business data + outbox message in SAME DB transaction
2. Separate relay process reads outbox → publishes to Kafka → marks as published

Application TX:
  INSERT INTO orders (...)
  INSERT INTO outbox_events (aggregate_type, aggregate_id, type, payload)
  -- Both in same transaction -- guaranteed atomicity

Outbox Relay (separate process/thread):
  SELECT * FROM outbox_events WHERE published = false
  → Publish to Kafka
  → UPDATE outbox_events SET published = true
```

**Implementation with Debezium (CDC-based):**
```yaml
# Debezium connector watches outbox table changes → publishes to Kafka
# Zero polling, real-time via DB transaction log
```

**Manual polling implementation:**
```java
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue
    private UUID id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;
    private boolean published = false;
    private LocalDateTime createdAt;
}

@Service
@Transactional
public class OrderService {
    @Autowired OrderRepository orderRepo;
    @Autowired OutboxRepository outboxRepo;

    public Order createOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        // In same TX — guaranteed atomicity
        outboxRepo.save(new OutboxEvent(
            "Order", order.getId().toString(), "OrderCreated",
            objectMapper.writeValueAsString(order)));
        return order;
    }
}

@Component
@Scheduled(fixedDelay = 1000)
public class OutboxRelay {
    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> kafka;

    @Transactional
    public void publishPendingEvents() {
        outboxRepo.findTop100ByPublishedFalseOrderByCreatedAtAsc()
            .forEach(event -> {
                kafka.send(event.getEventType(), event.getAggregateId(), event.getPayload());
                event.setPublished(true);
            });
    }
}
```

---

## Section 7: Performance & Optimization

### Q25. How do you tune the Spring Boot embedded Tomcat for high throughput?

**Answer:**

```yaml
server:
  tomcat:
    # Thread pool
    threads:
      max: 400          # max worker threads (default 200)
      min-spare: 20     # always keep 20 threads ready
    # Connection settings
    max-connections: 8192    # max simultaneous TCP connections
    accept-count: 200        # queue size when all threads busy
    connection-timeout: 20000  # 20s connection timeout

    # Performance
    max-http-form-post-size: 2MB
    keep-alive-timeout: 65000   # 65s keep-alive (match load balancer)
    max-keep-alive-requests: 100

  # HTTP compression
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html
    min-response-size: 1024  # only compress responses > 1KB
```

**Thread pool sizing formula:**
```
Threads = (CPU cores) / (1 - blocking coefficient)
  I/O-bound app (DB, HTTP calls): coefficient = 0.9 → 10 cores = 100 threads
  CPU-bound app (computation): coefficient = 0.1 → 10 cores = 11 threads
```

**Connection pool (HikariCP):**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20       # rule: (CPU cores * 2) + spindle count
      minimum-idle: 5
      connection-timeout: 30000   # 30s wait for connection
      idle-timeout: 600000        # 10min idle eviction
      max-lifetime: 1800000       # 30min max connection age
      leak-detection-threshold: 60000  # warn if connection held > 60s
```

---

### Q26. How do you implement caching in Spring Boot with multiple cache providers?

**Answer:**

```java
// Enable caching
@SpringBootApplication
@EnableCaching
public class App {}

// Service with caching
@Service
public class ProductService {

    // Cache result — key = productId
    @Cacheable(value = "products", key = "#id",
               condition = "#id > 0",
               unless = "#result == null")
    public Product findById(Long id) {
        return productRepo.findById(id).orElse(null);
    }

    // Update cache when product changes
    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepo.save(product);
    }

    // Evict from cache
    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        productRepo.deleteById(id);
    }

    // Evict all products cache
    @CacheEvict(value = "products", allEntries = true)
    @Scheduled(cron = "0 0 * * * *")  // every hour
    public void clearProductsCache() {}

    // Multiple caches
    @Caching(
        cacheable = @Cacheable("products"),
        evict = @CacheEvict(value = "productSearch", allEntries = true)
    )
    public Product createProduct(Product product) { ... }
}
```

**Redis cache configuration:**
```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));
    }

    @Bean
    public RedisCacheManagerBuilderCustomizer cacheManagerCustomizer() {
        return builder -> builder
            .withCacheConfiguration("products",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(60)))
            .withCacheConfiguration("sessions",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(10)));
    }
}
```

---

### Q27. How do you implement pagination and sorting efficiently in Spring Boot?

**Answer:**

```java
// Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Spring Data handles SQL: SELECT ... LIMIT ? OFFSET ?
    Page<Product> findByCategory(String category, Pageable pageable);

    // Slice — no COUNT query (cheaper for large tables)
    Slice<Product> findByPriceGreaterThan(BigDecimal price, Pageable pageable);

    // With Specification (dynamic filters)
    Page<Product> findAll(Specification<Product> spec, Pageable pageable);
}

// Controller
@GetMapping("/products")
public ResponseEntity<Page<ProductDto>> getProducts(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String direction) {

    // Validate sort field to prevent injection
    Set<String> allowedSortFields = Set.of("name", "price", "createdAt");
    if (!allowedSortFields.contains(sortBy)) {
        throw new BadRequestException("Invalid sort field: " + sortBy);
    }

    Sort sort = Sort.by(Sort.Direction.fromString(direction), sortBy);
    Pageable pageable = PageRequest.of(page, Math.min(size, 100), sort);

    Page<Product> products = productRepo.findAll(pageable);
    return ResponseEntity.ok(products.map(productMapper::toDto));
}

// Cursor-based pagination (better performance for large datasets)
// Instead of OFFSET (scans all preceding rows):
public List<Product> findNextPage(Long lastId, int size) {
    return productRepo.findByIdGreaterThanOrderByIdAsc(lastId, Pageable.ofSize(size));
}
```

---

## Section 8: Spring Boot Actuator & Observability

### Q28. How do you create custom Actuator endpoints and health indicators?

**Answer:**

```java
// Custom health indicator
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    @Autowired ExternalApiClient apiClient;

    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            apiClient.ping();
            long responseTime = System.currentTimeMillis() - start;

            if (responseTime > 2000) {
                return Health.degraded()
                    .withDetail("responseTime", responseTime + "ms")
                    .withDetail("status", "SLOW")
                    .build();
            }
            return Health.up()
                .withDetail("responseTime", responseTime + "ms")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .withException(e)
                .build();
        }
    }
}

// Custom Actuator endpoint
@Component
@Endpoint(id = "feature-flags")
public class FeatureFlagsEndpoint {

    @Autowired FeatureFlagService flagService;

    @ReadOperation  // GET /actuator/feature-flags
    public Map<String, Boolean> flags() {
        return flagService.getAllFlags();
    }

    @ReadOperation  // GET /actuator/feature-flags/{name}
    public Boolean flag(@Selector String name) {
        return flagService.isEnabled(name);
    }

    @WriteOperation  // POST /actuator/feature-flags
    public void setFlag(@Selector String name, boolean enabled) {
        flagService.setFlag(name, enabled);
    }
}

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, feature-flags
  endpoint:
    health:
      show-details: when-authorized
      show-components: always
      group:
        readiness:
          include: db, redis, diskSpace
        liveness:
          include: ping
```

---

### Q29. How do you integrate Spring Boot Actuator with Prometheus and Grafana?

**Answer:**

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health, info
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
    distribution:
      percentiles-histogram:
        http.server.requests: true   # enables latency histogram
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
      sla:
        http.server.requests: 100ms, 500ms, 1s
```

```java
// Custom metrics
@Service
public class OrderService {
    private final Counter ordersCreated;
    private final Timer orderProcessingTimer;
    private final Gauge pendingOrdersGauge;

    public OrderService(MeterRegistry registry, OrderRepository orderRepo) {
        this.ordersCreated = Counter.builder("orders.created")
            .tag("type", "standard")
            .description("Total orders created")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("order.processing.time")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        // Gauge reads value lazily
        Gauge.builder("orders.pending", orderRepo, OrderRepository::countPending)
            .description("Pending orders count")
            .register(registry);
    }

    public Order createOrder(CreateOrderRequest req) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(req);
            ordersCreated.increment();
            return order;
        });
    }
}
```

**Prometheus scrape config:**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
    scrape_interval: 15s
```

---

## Section 9: Spring Boot Configuration Advanced

### Q30. How does `@ConfigurationProperties` work and what are its advantages over `@Value`?

**Answer:**

```java
// @Value — fine for simple cases
@Value("${app.timeout:30}")
private int timeout;

// @ConfigurationProperties — structured, validated, IDE-friendly
@ConfigurationProperties(prefix = "app.email")
@Validated  // enables Bean Validation
public class EmailProperties {
    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port = 587;

    @NotBlank
    private String username;

    private boolean ssl = true;
    private Duration connectionTimeout = Duration.ofSeconds(5);
    private List<String> adminEmails = new ArrayList<>();
    private Map<String, String> headers = new HashMap<>();

    // getters and setters (or use Lombok @Data)
}

// Register (Spring Boot 2.2+)
@SpringBootApplication
@ConfigurationPropertiesScan  // auto-discovers all @ConfigurationProperties
public class App {}

// OR explicit:
@EnableConfigurationProperties(EmailProperties.class)
```

```yaml
# application.yml
app:
  email:
    host: smtp.gmail.com
    port: 587
    username: service@company.com
    ssl: true
    connection-timeout: 10s   # Duration auto-conversion
    admin-emails:
      - admin1@company.com
      - admin2@company.com
    headers:
      X-Source: spring-app
      X-Priority: "1"
```

**Advantages of @ConfigurationProperties:**
1. Type-safe (no runtime cast errors)
2. IDE auto-completion (with spring-configuration-processor)
3. Validation with JSR-303
4. Relaxed binding (camelCase, kebab-case, UPPER_CASE all work)
5. Grouping related properties in one class

---

### Q31. How do you implement dynamic configuration refresh at runtime without restart?

**Answer:**

**Option 1: Spring Cloud Config + `@RefreshScope`**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```java
@RefreshScope  // bean re-created on /actuator/refresh
@Service
public class FeatureService {
    @Value("${feature.newUI.enabled:false}")
    private boolean newUiEnabled;

    public boolean isNewUiEnabled() { return newUiEnabled; }
}

// Trigger refresh
// POST /actuator/refresh  → re-reads config from Config Server
```

**Option 2: Watch `@ConfigurationProperties` bean directly (no cloud needed)**
```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String mode = "standard";
    // getters/setters
}

// ApplicationContext refresh triggers re-binding
```

**Option 3: Custom file watcher (zero dependencies)**
```java
@Component
public class DynamicConfigWatcher implements InitializingBean {
    @Autowired Environment env;

    @Scheduled(fixedDelay = 30000)
    public void checkConfigFile() {
        Path configPath = Paths.get("/etc/app/config.properties");
        if (Files.getLastModifiedTime(configPath) > lastLoaded) {
            reloadConfig(configPath);
        }
    }
}
```

---

## Section 10: Spring Boot in Production

### Q32. What is Spring Boot's Graceful Shutdown and how do you configure it?

**Answer:**

```yaml
server:
  shutdown: graceful          # default is "immediate"

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # wait up to 30s for in-flight requests
```

```
Graceful shutdown flow:

1. SIGTERM received (Kubernetes sends this before killing pod)
2. Spring marks app as "not accepting new requests"
   → Kubernetes readiness probe fails → LB stops sending new traffic
3. Spring waits for in-flight requests to complete (up to timeout)
4. ApplicationContext closes (beans destroyed in reverse order)
5. DataSource closed, Kafka consumers stopped, schedulers halted
6. JVM exits with code 0
```

**Custom shutdown hook:**
```java
@Component
public class GracefulShutdownHandler implements DisposableBean {

    @Autowired KafkaListenerEndpointRegistry kafkaRegistry;

    @Override
    public void destroy() throws Exception {
        log.info("Starting graceful shutdown...");

        // Stop Kafka consumers first
        kafkaRegistry.stop();

        // Wait for in-flight messages to complete
        Thread.sleep(5000);

        log.info("Graceful shutdown complete");
    }
}

// Kubernetes deployment config (for proper graceful shutdown)
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # must be > spring timeout
      containers:
      - lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]  # wait for LB to deregister
```

---

### Q33. How do you implement multi-tenancy in a Spring Boot application?

**Answer:**

**Three approaches:**

**1. Separate database per tenant (highest isolation):**
```java
@Component
public class TenantDataSourceRouter extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();  // ThreadLocal
    }
}

@Configuration
public class MultiTenantDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        Map<Object, Object> dataSources = new HashMap<>();
        tenantList.forEach(tenant ->
            dataSources.put(tenant.getId(), createDataSourceFor(tenant)));

        TenantDataSourceRouter router = new TenantDataSourceRouter();
        router.setTargetDataSources(dataSources);
        router.setDefaultTargetDataSource(dataSources.get("default"));
        return router;
    }
}

// Extract tenant from JWT/header
@Component
public class TenantFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        String tenantId = extractTenantId((HttpServletRequest) req);
        TenantContext.setCurrentTenant(tenantId);
        try {
            chain.doFilter(req, res);
        } finally {
            TenantContext.clear();  // critical! prevent ThreadLocal leak
        }
    }
}
```

**2. Schema per tenant:**
```java
// Hibernate multi-tenancy with schema separation
@Configuration
public class HibernateMultiTenancyConfig {
    @Bean
    public MultiTenantConnectionProvider connectionProvider(DataSource ds) {
        return new SchemaBasedMultiTenantConnectionProvider(ds);
    }

    @Bean
    public CurrentTenantIdentifierResolver tenantResolver() {
        return () -> TenantContext.getCurrentTenant();
    }
}
```

**3. Row-level (discriminator column):**
```java
@Entity
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Order {
    @Column(name = "tenant_id")
    private String tenantId;
    // ...
}

// Enable filter automatically
@Aspect
@Component
public class TenantFilterAspect {
    @Autowired EntityManager em;

    @Before("execution(* com.example..repository.*.*(..))")
    public void enableTenantFilter() {
        Session session = em.unwrap(Session.class);
        session.enableFilter("tenantFilter")
            .setParameter("tenantId", TenantContext.getCurrentTenant());
    }
}
```

---

### Q34. How do you handle database schema migrations in Spring Boot CI/CD?

**Answer:**

**Flyway (SQL-based, most common):**
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true   # for existing DBs
    out-of-order: false         # strict ordering
    validate-on-migrate: true   # validate checksums
```

```
src/main/resources/db/migration/
  V1__create_users_table.sql
  V2__add_email_index.sql
  V3__create_orders_table.sql
  R__create_views.sql            # repeatable migration (R__ prefix)
```

```sql
-- V3__create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total DECIMAL(19,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

**Zero-downtime migration strategies:**
```
Expand → Migrate Data → Contract

V4__expand_add_full_name.sql:
  ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

V5__migrate_full_name_data.sql:
  UPDATE users SET full_name = first_name || ' ' || last_name;

-- Deploy new code that writes to both old and new columns

V6__contract_make_full_name_required.sql:
  ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Deploy code that only uses full_name

V7__contract_remove_old_columns.sql:
  ALTER TABLE users DROP COLUMN first_name;
  ALTER TABLE users DROP COLUMN last_name;
```

---

### Q35. How do you profile and find memory leaks in a Spring Boot application?

**Answer:**

```bash
# 1. JVM flags for profiling
java -jar app.jar \
  -Xms512m -Xmx2g \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -XX:+PrintGCDetails \
  -Xlog:gc*:file=/tmp/gc.log:time,tags

# 2. Trigger heap dump manually via Actuator
curl -X POST http://localhost:8080/actuator/heapdump > heap.hprof

# 3. Analyze with Eclipse MAT or VisualVM
# Look for: retained heap size, dominator tree, leak suspects

# 4. JFR (Java Flight Recorder) — zero overhead profiling
java -jar app.jar \
  -XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr

# 5. Async-profiler (wall-clock profiling)
./profiler.sh -d 30 -f /tmp/flamegraph.svg <PID>
```

**Common Spring Boot memory leaks:**

```java
// LEAK 1: ThreadLocal not cleared
// Filter/interceptor sets ThreadLocal but doesn't clear in finally
public class TenantFilter implements Filter {
    public void doFilter(...) {
        TenantContext.set(tenant);
        try {
            chain.doFilter(req, res);
        } finally {
            TenantContext.clear();  // MUST clear!
        }
    }
}

// LEAK 2: Event listener prevents GC
@Component
public class LeakyComponent {
    @Autowired ApplicationEventPublisher publisher;

    // Anonymous listener registered but never removed
    publisher.publishEvent(new Object() {
        @EventListener
        public void handle(MyEvent e) { ... }
    });
}

// LEAK 3: Hibernate first-level cache growing in long session
// Batch processing: clear after each batch
for (List<Entity> batch : batches) {
    batch.forEach(entityManager::persist);
    entityManager.flush();
    entityManager.clear();  // prevent first-level cache bloat
}

// LEAK 4: Cache without eviction
@Cacheable("users")  // cache grows forever without TTL!
// FIX: configure TTL in cache manager
```

---

## Section 11: Design Patterns in Spring Boot

### Q36. How do you implement the Strategy Pattern in Spring Boot?

**Answer:**

```java
// Strategy interface
public interface PaymentStrategy {
    PaymentResult process(PaymentRequest request);
    String type();  // discriminator
}

// Implementations — all annotated as Spring beans
@Component
public class CreditCardPayment implements PaymentStrategy {
    public PaymentResult process(PaymentRequest req) { /* charge card */ }
    public String type() { return "CREDIT_CARD"; }
}

@Component
public class CryptoPayment implements PaymentStrategy {
    public PaymentResult process(PaymentRequest req) { /* broadcast tx */ }
    public String type() { return "CRYPTO"; }
}

// Strategy registry — Spring injects all implementations
@Service
public class PaymentService {
    private final Map<String, PaymentStrategy> strategies;

    // Spring injects all PaymentStrategy beans into the list
    public PaymentService(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(PaymentStrategy::type, Function.identity()));
    }

    public PaymentResult processPayment(String type, PaymentRequest req) {
        PaymentStrategy strategy = strategies.get(type);
        if (strategy == null) {
            throw new UnsupportedPaymentTypeException(type);
        }
        return strategy.process(req);
    }
}
```

---

### Q37. How do you implement the Decorator Pattern using Spring AOP?

**Answer:**

```java
// Business method to decorate
@Service
public class ProductService {
    public Product findById(Long id) {
        return productRepo.findById(id).orElseThrow();
    }
}

// Decoration via AOP
@Aspect
@Component
public class CachingAspect {
    @Autowired CacheManager cacheManager;

    @Around("@annotation(Cacheable)")
    public Object cacheResult(ProceedingJoinPoint pjp) throws Throwable {
        // Decorate with caching
        String key = generateKey(pjp);
        Cache cache = cacheManager.getCache("products");

        Cache.ValueWrapper cached = cache.get(key);
        if (cached != null) return cached.get();

        Object result = pjp.proceed();
        cache.put(key, result);
        return result;
    }
}

@Aspect
@Component
public class AuditAspect {
    @AfterReturning(
        pointcut = "execution(* com.example..service.*Service.*(..)) && @annotation(Audited)",
        returning = "result")
    public void audit(JoinPoint jp, Object result) {
        // Decorate with audit logging
        auditLog.record(jp.getSignature().getName(),
                        jp.getArgs(), result,
                        SecurityContextHolder.getContext().getAuthentication().getName());
    }
}
```

---

### Q38. How do you implement an event-driven architecture within a single Spring Boot app?

**Answer:**

```java
// Domain event
public record OrderPlacedEvent(Long orderId, BigDecimal amount, String customerId) {}

// Publisher
@Service
@Transactional
public class OrderService {
    @Autowired ApplicationEventPublisher eventPublisher;

    public Order createOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));

        // Publish synchronously (in same thread, same transaction)
        eventPublisher.publishEvent(new OrderPlacedEvent(
            order.getId(), order.getTotal(), req.getCustomerId()));

        return order;
    }
}

// Synchronous listener (same thread, same transaction by default)
@Component
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class InventoryEventHandler {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.reserveStock(event.orderId());
    }
}

// Async listener (separate thread, separate transaction)
@Component
@Async
public class NotificationEventHandler {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

// Transactional listener — runs AFTER transaction commits
@Component
public class AuditEventHandler {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Safe to read committed data here
        auditService.logOrderCreated(event.orderId());
    }
}
```

---

## Section 12: Spring Boot with Cloud Native

### Q39. How do you build a 12-factor app with Spring Boot?

**Answer:**

```
The 12 Factors and Spring Boot implementation:

I. Codebase
   One repo, multiple deploys via Git + profiles/env vars

II. Dependencies
   pom.xml / build.gradle — no system-level deps, fat jar bundles all

III. Config
   application.yml for defaults, env vars override
   spring.config.import=optional:configserver:
   NEVER commit secrets to git → use Secret Manager / Vault

IV. Backing Services
   Treat as attached resources via URLs
   spring.datasource.url=${DATABASE_URL}
   spring.redis.host=${REDIS_HOST}

V. Build, release, run
   mvn package → Docker image (build)
   Image + env config (release)
   java -jar app.jar (run)

VI. Processes
   @Async, stateless REST controllers
   Session state → Redis (Memorystore), not in-process

VII. Port binding
   server.port=${PORT:8080}  ← self-contained, no app server needed

VIII. Concurrency
   Scale out with multiple instances (K8s replicas)
   Thread pools for vertical scaling within instance

IX. Disposability
   Graceful shutdown (server.shutdown=graceful)
   Fast startup (use spring.jpa.open-in-view=false, lazy init)

X. Dev/prod parity
   Same Docker image dev → staging → prod
   Testcontainers for local DB (same Postgres as prod)

XI. Logs
   Log to stdout/stderr (Spring defaults to console)
   NO file logging in containers
   logging.pattern.console=%d{ISO8601} [%thread] %-5level %logger - %msg%n

XII. Admin processes
   Spring Batch / @CommandLineRunner for one-off tasks
   Separate job pods in Kubernetes
```

---

### Q40. How does Spring Boot GraalVM Native Image work and what are the trade-offs?

**Answer:**

**Native Image compilation:**
```
Traditional JVM:               GraalVM Native Image:
  Source → bytecode              Source → native binary
  JIT compiled at runtime        AOT compiled at build time
  JVM startup: ~2-5 seconds      Native startup: ~50ms
  Memory: 256+ MB                Memory: ~50-100 MB
  Peak perf: very high           Peak perf: slightly lower (no JIT)
```

```bash
# Build native image
mvn -Pnative native:compile

# Or with Docker (no GraalVM install needed)
mvn -Pnative spring-boot:build-image
```

**Challenges with native image:**
```
1. Reflection — must be declared at build time
   Spring Boot 3.x + GraalVM auto-generates hints for Spring beans
   Custom reflection: register in reflect-config.json or use @RegisterReflectionForBinding

2. Dynamic class loading — not supported
   No runtime bytecode generation (Javassist, CGLIB)
   Spring Boot 3.x switched from CGLIB to interface proxies

3. Build time — 3–10 minutes (vs seconds for JVM)
   Use layered caching in CI/CD

4. Debugging — harder (no JIT, different stack traces)

5. Third-party library compatibility
   Check GraalVM reachability metadata repo
```

**When to use:**
```
Use Native Image for:
  - Serverless / Cloud Functions (cold start matters)
  - CLI tools
  - High-density container deployments
  - Kubernetes sidecar containers

Stick with JVM for:
  - Long-running services (JIT warms up and outperforms)
  - Apps with heavy reflection/dynamic proxies
  - When build time is a bottleneck
```

---

## Section 13: Advanced REST API Design

### Q41. How do you implement API versioning in Spring Boot?

**Answer:**

**4 strategies:**

```java
// Strategy 1: URI versioning (most common, highly cacheable)
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller {}

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller {}

// Strategy 2: Request Header versioning
@GetMapping(value = "/users", headers = "API-Version=1")
public List<UserV1Dto> getUsersV1() {}

@GetMapping(value = "/users", headers = "API-Version=2")
public List<UserV2Dto> getUsersV2() {}

// Strategy 3: Accept header (content negotiation)
@GetMapping(value = "/users",
    produces = "application/vnd.company.app-v1+json")
public List<UserV1Dto> getUsersV1() {}

@GetMapping(value = "/users",
    produces = "application/vnd.company.app-v2+json")
public List<UserV2Dto> getUsersV2() {}

// Strategy 4: Query parameter (simple but pollutes URL)
@GetMapping(value = "/users", params = "version=1")
public List<UserV1Dto> getUsersV1() {}

// Best practice: URI versioning for external APIs
// Header versioning for internal microservices
```

---

### Q42. How do you implement rate limiting in Spring Boot?

**Answer:**

```java
// Using Resilience4j RateLimiter
@Configuration
public class RateLimiterConfig {
    @Bean
    public RateLimiterRegistry rateLimiterRegistry() {
        io.github.resilience4j.ratelimiter.RateLimiterConfig config =
            io.github.resilience4j.ratelimiter.RateLimiterConfig.custom()
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .limitForPeriod(100)         // 100 requests per second
                .timeoutDuration(Duration.ofMillis(100))
                .build();
        return RateLimiterRegistry.of(config);
    }
}

@RestController
public class ApiController {
    @RateLimiter(name = "default", fallbackMethod = "rateLimitFallback")
    @GetMapping("/data")
    public ResponseEntity<Data> getData() { ... }

    public ResponseEntity<Data> rateLimitFallback(RequestNotPermitted ex) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", "1")
            .build();
    }
}

// Per-user rate limiting with Redis (Bucket4j)
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    @Autowired RedisTemplate<String, Long> redis;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ... {
        String userId = extractUserId(req);
        String key = "rate_limit:" + userId;

        Long requests = redis.opsForValue().increment(key);
        if (requests == 1) {
            redis.expire(key, 1, TimeUnit.MINUTES);
        }
        if (requests > 60) {  // 60 req/min per user
            res.setStatus(429);
            return;
        }
        chain.doFilter(req, res);
    }
}
```

---

### Q43. How do you handle file uploads efficiently in Spring Boot?

**Answer:**

```yaml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 100MB
      max-request-size: 110MB
      file-size-threshold: 2KB   # buffer to disk after 2KB
```

```java
@RestController
public class FileUploadController {

    // Single file
    @PostMapping("/upload")
    public ResponseEntity<String> upload(@RequestParam MultipartFile file) {
        if (file.isEmpty()) throw new BadRequestException("Empty file");

        String filename = StringUtils.cleanPath(file.getOriginalFilename());
        // Validate file type
        String contentType = file.getContentType();
        if (!ALLOWED_TYPES.contains(contentType)) {
            throw new BadRequestException("File type not allowed: " + contentType);
        }

        // Stream to GCS (don't load entire file into memory)
        BlobInfo blobInfo = BlobInfo.newBuilder("my-bucket", filename).build();
        storage.createFrom(blobInfo, file.getInputStream());

        return ResponseEntity.ok("Uploaded: " + filename);
    }

    // Streaming large files (avoid OOM for huge uploads)
    @PostMapping("/upload/stream")
    public ResponseEntity<String> uploadStream(HttpServletRequest req) throws IOException {
        // Read raw request body as stream
        try (InputStream is = req.getInputStream()) {
            storage.createFrom(blobInfo, is);
        }
        return ResponseEntity.ok("Uploaded");
    }

    // Multipart mixed (file + JSON metadata)
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<Document> uploadWithMetadata(
        @RequestPart("file") MultipartFile file,
        @RequestPart("metadata") @Valid DocumentMetadata metadata) {
        // Process both together
    }
}
```

---

## Section 14: Final Scenario Questions

### Q44. How do you migrate a Spring Boot app from Spring Security 5.x to 6.x (Spring Boot 2 → 3)?

**Answer:**

**Major breaking changes:**

```java
// Old (Spring Boot 2.x)
@Configuration
public class OldSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
            .and()
            .formLogin();
    }
}

// New (Spring Boot 3.x) — WebSecurityConfigurerAdapter REMOVED
@Configuration
@EnableWebSecurity
public class NewSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()  // antMatchers → requestMatchers
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        // Return UserDetailsService bean instead of overriding configureAuthentication()
        return username -> userRepo.findByUsername(username)
            .map(CustomUserDetails::new)
            .orElseThrow(() -> new UsernameNotFoundException(username));
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**Other breaking changes checklist:**
```
✓ javax.* → jakarta.* (entire package rename)
✓ spring.security.oauth2.* properties restructured
✓ AuthorizationFilter replaces FilterSecurityInterceptor
✓ @EnableGlobalMethodSecurity → @EnableMethodSecurity
✓ HttpMethod.GET.name() → HttpMethod.GET (as constant)
✓ SecurityContextHolder stored in RequestAttributeSecurityContextRepository by default
✓ HTTPS redirect by default changed
```

---

### Q45. Your Spring Boot app has high GC pauses. How do you diagnose and fix it?

**Answer:**

```bash
# Step 1: Enable GC logging
java -jar app.jar \
  -Xlog:gc*:file=/tmp/gc.log:time,uptime,tags \
  -XX:+PrintGCDateStamps

# Step 2: Analyze GC log
# Use GCEasy.io or GCViewer to visualize
# Look for: Full GC frequency, pause times, promotion failures

# Step 3: Check heap usage trend
curl http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap
```

**Common causes and fixes:**

```
Cause 1: Heap too small → frequent GC
  Fix: Increase -Xmx (ensure < 75% container limit)
  Fix: Analyze what's using heap (heap dump → MAT)

Cause 2: Long-lived objects in old gen → Full GC
  Fix: Check for memory leaks (unbounded caches, ThreadLocal, static collections)
  Fix: Use weak references for caches (Caffeine handles this automatically)

Cause 3: Too many short-lived allocations
  Fix: Increase -Xmn (young gen size)
  Fix: Object pooling for frequent allocations
  Fix: Review hot code paths with async profiler

Cause 4: G1GC mixed collections too long
  Fix: -XX:MaxGCPauseMillis=200  (G1 will try to meet this target)
  Fix: -XX:G1HeapRegionSize=16m  (for apps with large objects)
  Fix: Consider ZGC (< 1ms pauses) for latency-sensitive apps:
       -XX:+UseZGC (Java 15+ production ready)

Cause 5: Finalization overhead
  Fix: Avoid finalize() — use try-with-resources instead
  Fix: Check for unclosed streams/connections
```

**Production GC settings for Spring Boot:**
```bash
java -jar app.jar \
  -XX:+UseG1GC \
  -Xms1g -Xmx1g \                    # fixed heap = no resizing GC
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -XX:+DisableExplicitGC \            # prevent System.gc() calls
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heap.hprof
```

---

### Q46. How do you implement idempotency for REST APIs in Spring Boot?

**Answer:**

**Idempotency** = calling an API multiple times produces the same result (safe retries).

```java
// Idempotency key approach (Stripe-style)
@RestController
public class PaymentController {

    @PostMapping("/payments")
    public ResponseEntity<Payment> createPayment(
        @RequestHeader("Idempotency-Key") @NotBlank String idempotencyKey,
        @RequestBody @Valid PaymentRequest request) {

        // Check if we've seen this key before
        Optional<Payment> existing = idempotencyRepo.findByKey(idempotencyKey);
        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get());  // return cached response
        }

        // Process payment
        Payment payment = paymentService.process(request);

        // Store result with idempotency key
        idempotencyRepo.save(new IdempotencyRecord(
            idempotencyKey, payment, Instant.now().plus(24, HOURS)));

        return ResponseEntity.status(201).body(payment);
    }
}

// Idempotency store
@Entity
public class IdempotencyRecord {
    @Id
    private String key;
    @Column(columnDefinition = "jsonb")
    private String responseBody;
    private LocalDateTime expiresAt;
}

// With Redis for distributed idempotency (prevents duplicate in race condition)
@Service
public class IdempotencyService {
    @Autowired RedisTemplate<String, String> redis;

    public <T> T executeIdempotent(String key, Supplier<T> operation) {
        String lockKey = "idem:lock:" + key;
        String resultKey = "idem:result:" + key;

        // Check cached result
        String cached = redis.opsForValue().get(resultKey);
        if (cached != null) return deserialize(cached);

        // Lock to prevent concurrent execution with same key
        Boolean locked = redis.opsForValue()
            .setIfAbsent(lockKey, "locked", Duration.ofSeconds(30));

        if (!Boolean.TRUE.equals(locked)) {
            throw new ConcurrentRequestException("Duplicate request in flight");
        }

        try {
            T result = operation.get();
            redis.opsForValue().set(resultKey, serialize(result), Duration.ofHours(24));
            return result;
        } finally {
            redis.delete(lockKey);
        }
    }
}
```

---

### Q47. How do you implement health check endpoints for Kubernetes liveness and readiness probes?

**Answer:**

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true          # enables /actuator/health/liveness and /actuator/health/readiness
      show-details: always
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

```java
// Custom readiness indicator — not ready if critical dependency is down
@Component
public class DatabaseReadinessIndicator implements HealthIndicator {

    @Autowired DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up().withDetail("db", "reachable").build();
            }
        } catch (SQLException e) {
            return Health.down().withDetail("db", e.getMessage()).build();
        }
        return Health.down().withDetail("db", "invalid connection").build();
    }
}

// Custom liveness indicator — app is alive but might be degraded
@Component
public class DeadlockLivenessIndicator implements LivenessStateHealthIndicator {
    @Override
    public Health health() {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        long[] deadlocked = bean.findDeadlockedThreads();
        if (deadlocked != null && deadlocked.length > 0) {
            return Health.broken().withDetail("deadlockedThreads", deadlocked.length).build();
        }
        return Health.correct().build();
    }
}

// Programmatically change readiness state
@Service
public class WarmupService implements ApplicationRunner {
    @Autowired ApplicationContext context;

    @Override
    public void run(ApplicationArguments args) {
        AvailabilityChangeEvent.publish(context, ReadinessState.REFUSING_TRAFFIC);
        try {
            warmUpCaches();
            warmUpConnections();
        } finally {
            AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
        }
    }
}
```

```yaml
# Kubernetes probe config
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3
```

---

### Q48. How would you design a Spring Boot application for 1 million concurrent users?

**Answer:**

```
Architecture for 1M concurrent users:

1. Non-blocking I/O — use Spring WebFlux (Netty)
   Traditional Tomcat: 1 thread per request → 1M threads = OOM
   WebFlux: event loop (few threads handle millions of concurrent requests)

2. Stateless — scale horizontally
   No in-memory session (use Redis via Spring Session)
   No instance-level cache (use Redis / Memcached)

3. Async everywhere
   WebClient (non-blocking HTTP) instead of RestTemplate
   Reactive Spring Data (R2DBC) instead of JPA
   Reactive Kafka consumer

4. Caching layers
   L1: Caffeine (in-process, < 1ms)
   L2: Redis (distributed, < 5ms)
   CDN: CloudFlare / Cloud CDN for static + cacheable API responses

5. Database scaling
   Read replicas for read-heavy endpoints
   Connection pool tuning (R2DBC pool for reactive)
   Query optimization (indexes, avoid N+1)
   Eventual consistency where acceptable

6. Infrastructure
   Kubernetes HPA (Horizontal Pod Autoscaler)
   Cloud SQL with read replicas
   Cloud Spanner for write-heavy global scale
   Pub/Sub for async operations

7. Rate limiting at the edge
   Cloud Armor / API Gateway rate limiting
   Per-user limits in Redis

8. Load test before launch
   Gatling or k6 to simulate 1M concurrent
   Monitor: CPU, memory, GC, DB connections, Kafka lag
```

---

### Q49. How do you implement blue-green and canary deployments for a Spring Boot app on Kubernetes?

**Answer:**

**Blue-Green Deployment:**
```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    version: blue
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
      version: blue

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  labels:
    version: green
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
      version: green

---
# Service — switch between blue/green by changing selector
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: my-app
    version: blue   # ← change to "green" to switch
  ports:
  - port: 80

# Switch command:
kubectl patch svc app-service -p '{"spec":{"selector":{"version":"green"}}}'
# Rollback:
kubectl patch svc app-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

**Canary Deployment (gradual traffic shift):**
```yaml
# Stable deployment: 9 replicas = 90% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9  # 90% traffic

---
# Canary deployment: 1 replica = 10% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1  # 10% traffic
  template:
    metadata:
      labels:
        app: my-app
        version: canary

# Service selects both (no version label)
spec:
  selector:
    app: my-app  # selects all 10 pods, Kube distributes evenly

# Gradually scale up canary:
kubectl scale deployment app-canary --replicas=3  # 30%
kubectl scale deployment app-stable --replicas=7
# ... eventually:
kubectl scale deployment app-canary --replicas=10
kubectl delete deployment app-stable
```

---

### Q50. How do you implement a CQRS pattern in Spring Boot?

**Answer:**

**CQRS** = Command Query Responsibility Segregation — separate read and write models.

```java
// Command side — optimized for writes
@RestController
@RequestMapping("/commands/orders")
public class OrderCommandController {

    @PostMapping
    public ResponseEntity<UUID> createOrder(@RequestBody @Valid CreateOrderCommand cmd) {
        UUID orderId = commandBus.dispatch(cmd);
        return ResponseEntity.accepted()
            .header("Location", "/queries/orders/" + orderId)
            .body(orderId);
    }

    @PutMapping("/{id}/cancel")
    public ResponseEntity<Void> cancelOrder(@PathVariable UUID id) {
        commandBus.dispatch(new CancelOrderCommand(id));
        return ResponseEntity.accepted().build();
    }
}

// Command handler — writes to normalized relational DB
@Component
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand> {

    @Override
    @Transactional
    public UUID handle(CreateOrderCommand cmd) {
        Order order = new Order(cmd.getCustomerId(), cmd.getItems());
        order = orderRepo.save(order);

        // Publish domain event to update read model
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
        return order.getId();
    }
}

// Query side — optimized for reads (denormalized view)
@RestController
@RequestMapping("/queries/orders")
public class OrderQueryController {

    @GetMapping("/{id}")
    public OrderDetailView getOrder(@PathVariable UUID id) {
        return orderViewRepo.findById(id).orElseThrow();
    }

    @GetMapping("/customer/{customerId}")
    public Page<OrderSummaryView> getCustomerOrders(
        @PathVariable String customerId, Pageable pageable) {
        return orderViewRepo.findByCustomerId(customerId, pageable);
    }
}

// Read model — denormalized, pre-joined, optimized for queries
@Entity
@Table(name = "order_detail_view")
public class OrderDetailView {
    @Id private UUID orderId;
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private String status;
    private List<OrderItemView> items;
    // All data pre-joined — single query to read
}

// Update read model when write model changes
@Component
@Async
public class OrderProjection {

    @EventListener
    @Transactional
    public void on(OrderCreatedEvent event) {
        Order order = event.getOrder();
        Customer customer = customerRepo.findById(order.getCustomerId()).orElseThrow();

        OrderDetailView view = new OrderDetailView();
        view.setOrderId(order.getId());
        view.setCustomerName(customer.getFullName());
        view.setCustomerEmail(customer.getEmail());
        view.setTotalAmount(order.getTotal());
        view.setItems(mapItems(order.getItems()));

        orderViewRepo.save(view);
    }

    @EventListener
    @Transactional
    public void on(OrderCancelledEvent event) {
        orderViewRepo.findById(event.getOrderId())
            .ifPresent(view -> {
                view.setStatus("CANCELLED");
                orderViewRepo.save(view);
            });
    }
}
```

---

## Quick Reference: Spring Boot Key Annotations

| Annotation | Purpose |
|-----------|---------|
| `@SpringBootApplication` | Entry point: config + scan + auto-config |
| `@ConfigurationProperties` | Type-safe configuration binding |
| `@ConditionalOnProperty` | Conditional bean creation |
| `@RefreshScope` | Dynamic config reload |
| `@Transactional` | Transaction management |
| `@Async` | Async method execution |
| `@Scheduled` | Scheduled task |
| `@EventListener` | Application event listener |
| `@TransactionalEventListener` | Event listener after TX commit |
| `@Cacheable` / `@CacheEvict` | Method-level caching |
| `@CircuitBreaker` | Resilience4j circuit breaker |
| `@PreAuthorize` | Method-level security with SpEL |
| `@RestControllerAdvice` | Global exception handling |
| `@EntityGraph` | Eager load JPA associations |
| `@Lock` | Pessimistic/optimistic DB locking |
| `@Version` | Optimistic locking version field |

---

