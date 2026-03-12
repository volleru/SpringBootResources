# Spring Boot Interview Questions - Real-Time Scenarios

## Core Spring Boot Concepts

### 1. What is Spring Boot and how is it different from Spring Framework?
**Answer:** Spring Boot is an opinionated framework built on top of Spring Framework that simplifies configuration and setup. It provides auto-configuration, embedded servers, and production-ready features out of the box.

### 2. Explain the concept of Auto-Configuration in Spring Boot.
**Answer:** Auto-configuration automatically configures Spring beans based on the dependencies present in the classpath. For example, if H2 database is in classpath, it auto-configures an in-memory database.

### 3. What is the purpose of `@SpringBootApplication` annotation?
**Answer:** It's a combination of three annotations:
- `@Configuration` - Marks the class as a configuration class
- `@EnableAutoConfiguration` - Enables auto-configuration
- `@ComponentScan` - Scans for components in the current package and sub-packages

### 4. How does Spring Boot know which beans to configure automatically?
**Answer:** Spring Boot uses `spring.factories` file (or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in newer versions) which lists all auto-configuration classes. Conditions like `@ConditionalOnClass`, `@ConditionalOnMissingBean` determine which configurations apply.

### 5. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?
**Answer:** All are stereotype annotations for component scanning:
- `@Component` - Generic component
- `@Service` - Business logic layer
- `@Repository` - Data access layer (adds exception translation)
- `@Controller` - Web layer (handles HTTP requests)

---

## Real-Time Scenario Questions

### 6. Your application is taking 2 minutes to start. How would you debug and optimize startup time?
**Answer:**
- Enable startup actuator endpoint to analyze bean initialization times
- Use lazy initialization (`spring.main.lazy-initialization=true`)
- Check for heavy initialization in `@PostConstruct` methods
- Use Spring Boot's startup time analysis with `ApplicationStartup`
- Profile with tools like async-profiler

### 7. How do you handle configuration for different environments (dev, staging, prod)?
**Answer:**
- Use Spring Profiles with `application-{profile}.yml` files
- Set active profile via `spring.profiles.active`
- Use environment variables for sensitive data
- Consider Spring Cloud Config for centralized configuration
- Use `@Profile` annotation for environment-specific beans

### 8. Your REST API is returning 500 errors intermittently. How would you troubleshoot?
**Answer:**
- Check logs for stack traces
- Enable detailed error responses in dev
- Use Spring Boot Actuator for health checks
- Implement proper exception handling with `@ControllerAdvice`
- Add distributed tracing (Sleuth/Zipkin)
- Monitor database connection pool exhaustion

### 9. How do you implement circuit breaker pattern in Spring Boot?
**Answer:**
```java
@CircuitBreaker(name = "serviceA", fallbackMethod = "fallback")
public String callExternalService() {
    return restTemplate.getForObject(url, String.class);
}

public String fallback(Exception e) {
    return "Fallback response";
}
```
Use Resilience4j with Spring Boot for circuit breaker, retry, rate limiter patterns.

### 10. How would you secure sensitive properties like database passwords?
**Answer:**
- Use environment variables
- Spring Cloud Vault for secrets management
- Jasypt for encrypted properties
- AWS Secrets Manager / Azure Key Vault
- Never commit secrets to version control

---

## REST API & Web

### 11. Difference between `@RestController` and `@Controller`?
**Answer:** `@RestController` = `@Controller` + `@ResponseBody`. It automatically serializes return objects to JSON/XML without needing `@ResponseBody` on each method.

### 12. How do you handle validation in Spring Boot REST APIs?
**Answer:**
```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO user) {
    // If validation fails, MethodArgumentNotValidException is thrown
}

public class UserDTO {
    @NotBlank(message = "Name is required")
    private String name;
    
    @Email(message = "Invalid email format")
    private String email;
}
```

### 13. How do you implement global exception handling?
**Answer:**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("Internal server error"));
    }
}
```

### 14. How do you implement API versioning in Spring Boot?
**Answer:**
- URL versioning: `/api/v1/users`, `/api/v2/users`
- Header versioning: `X-API-Version: 1`
- Request parameter: `/api/users?version=1`
- Content negotiation: `Accept: application/vnd.company.v1+json`

### 15. How do you implement pagination in Spring Boot?
**Answer:**
```java
@GetMapping("/users")
public Page<User> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "id") String sortBy) {
    
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
    return userRepository.findAll(pageable);
}
```

---

## Database & JPA

### 16. Explain the difference between `findById()` and `getOne()` in JPA Repository.
**Answer:**
- `findById()` - Returns `Optional<T>`, executes query immediately
- `getOne()`/`getReferenceById()` - Returns a lazy proxy, throws `EntityNotFoundException` if accessed and not found

### 17. How do you handle database connection pool in production?
**Answer:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
```
Monitor pool metrics via Actuator, set appropriate timeouts, and size based on load testing.

### 18. What is N+1 problem and how do you solve it?
**Answer:**
N+1 occurs when fetching parent entities triggers N additional queries for child entities.

Solutions:
- Use `@EntityGraph` for eager fetching
- Use `JOIN FETCH` in JPQL
- Use batch fetching with `@BatchSize`
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
List<Order> findOrdersWithItems(@Param("customerId") Long customerId);
```

### 19. How do you implement database migrations in Spring Boot?
**Answer:**
Use Flyway or Liquibase:
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```
Create migration files like `V1__create_users_table.sql`, `V2__add_email_column.sql`

### 20. Explain transaction management in Spring Boot.
**Answer:**
```java
@Transactional(
    isolation = Isolation.READ_COMMITTED,
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class,
    timeout = 30
)
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    // Atomic operation
}
```
Use `@Transactional` at service layer, understand propagation levels (REQUIRED, REQUIRES_NEW, etc.)

---

## Security

### 21. How do you implement JWT authentication in Spring Boot?
**Answer:**
- Add Spring Security dependency
- Create JWT utility class for token generation/validation
- Implement `UserDetailsService`
- Create JWT filter to intercept requests
- Configure `SecurityFilterChain` to use JWT filter

### 22. How do you secure REST endpoints with role-based access?
**Answer:**
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .build();
    }
}

// Method level
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { }
```

### 23. How do you prevent CSRF attacks in Spring Boot?
**Answer:**
- For stateless REST APIs with JWT, CSRF can be disabled
- For stateful apps, Spring Security enables CSRF by default
- Use `CsrfTokenRepository` to customize token handling
- Include CSRF token in forms and AJAX requests

### 24. How do you implement OAuth2 login in Spring Boot?
**Answer:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
```

### 25. How do you handle CORS in Spring Boot?
**Answer:**
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

---

## Microservices

### 26. How do you implement service discovery in Spring Boot microservices?
**Answer:**
Use Spring Cloud Netflix Eureka or Consul:
```java
@EnableEurekaClient
@SpringBootApplication
public class OrderServiceApplication { }
```
Configure service registration and discovery for load-balanced inter-service communication.

### 27. How do you implement distributed tracing?
**Answer:**
Use Micrometer Tracing (replacement for Spring Cloud Sleuth):
```yaml
management:
  tracing:
    sampling:
      probability: 1.0
```
Integrate with Zipkin/Jaeger for trace visualization.

### 28. How do you handle inter-service communication?
**Answer:**
- Synchronous: RestTemplate, WebClient, OpenFeign
- Asynchronous: Kafka, RabbitMQ
```java
@FeignClient(name = "user-service")
public interface UserServiceClient {
    @GetMapping("/users/{id}")
    UserDTO getUser(@PathVariable Long id);
}
```

### 29. How do you implement API Gateway pattern?
**Answer:**
Use Spring Cloud Gateway:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
```

### 30. How do you implement centralized configuration?
**Answer:**
Use Spring Cloud Config Server:
- Set up config server pointing to Git repo
- Client applications fetch configuration on startup
- Support for refresh scope to update properties without restart

---

## Testing

### 31. How do you write unit tests for Spring Boot services?
**Answer:**
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void shouldReturnUserWhenFound() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User()));
        assertNotNull(userService.getUserById(1L));
    }
}
```

### 32. How do you write integration tests for REST controllers?
**Answer:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldCreateUser() {
        ResponseEntity<User> response = restTemplate
            .postForEntity("/api/users", new UserDTO("John"), User.class);
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

### 33. How do you test with an embedded database?
**Answer:**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldSaveUser() {
        User saved = userRepository.save(new User("John"));
        assertNotNull(saved.getId());
    }
}
```

### 34. How do you mock external API calls in tests?
**Answer:**
Use WireMock:
```java
@WireMockTest(httpPort = 8089)
class ExternalServiceTest {
    
    @Test
    void shouldHandleExternalApiCall() {
        stubFor(get("/external/data")
            .willReturn(okJson("{\"status\": \"success\"}")));
        
        // Test your service that calls the external API
    }
}
```

### 35. What is `@MockBean` vs `@Mock`?
**Answer:**
- `@Mock` (Mockito) - Creates a mock in isolation, used in unit tests
- `@MockBean` (Spring Boot Test) - Replaces a Spring bean in the application context with a mock, used in integration tests

---

## Performance & Monitoring

### 36. How do you implement caching in Spring Boot?
**Answer:**
```java
@EnableCaching
@Configuration
public class CacheConfig { }

@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

### 37. How do you monitor Spring Boot application health?
**Answer:**
Use Spring Boot Actuator:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, info, prometheus
  endpoint:
    health:
      show-details: always
```
Integrate with Prometheus/Grafana for monitoring dashboards.

### 38. How do you implement rate limiting?
**Answer:**
Use Resilience4j or Bucket4j:
```java
@RateLimiter(name = "api", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/data")
public ResponseEntity<Data> getData() {
    return ResponseEntity.ok(dataService.getData());
}
```

### 39. How do you optimize database queries in production?
**Answer:**
- Enable slow query logging
- Use query hints and indexing
- Implement read replicas for read-heavy operations
- Use connection pooling (HikariCP)
- Cache frequently accessed data
- Monitor with APM tools (New Relic, Dynatrace)

### 40. How do you handle memory leaks in Spring Boot?
**Answer:**
- Use profiling tools (VisualVM, JProfiler)
- Monitor heap usage via Actuator
- Check for unclosed resources (streams, connections)
- Review singleton beans holding large data
- Analyze heap dumps
- Check for circular references in caching

---

## Advanced Topics

### 41. Explain the Spring Boot bean lifecycle.
**Answer:**
1. Bean instantiation
2. Populate properties (dependency injection)
3. `BeanNameAware.setBeanName()`
4. `BeanFactoryAware.setBeanFactory()`
5. `ApplicationContextAware.setApplicationContext()`
6. `@PostConstruct` / `InitializingBean.afterPropertiesSet()`
7. Bean is ready for use
8. `@PreDestroy` / `DisposableBean.destroy()`

### 42. How do you create custom auto-configuration?
**Answer:**
```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```
Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

### 43. How do you implement async processing in Spring Boot?
**Answer:**
```java
@EnableAsync
@Configuration
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        return executor;
    }
}

@Service
public class EmailService {
    
    @Async
    public CompletableFuture<Void> sendEmailAsync(String to, String body) {
        // Send email
        return CompletableFuture.completedFuture(null);
    }
}
```

### 44. How do you implement scheduled tasks?
**Answer:**
```java
@EnableScheduling
@Configuration
public class SchedulerConfig { }

@Component
public class ReportGenerator {
    
    @Scheduled(cron = "0 0 2 * * ?") // Every day at 2 AM
    public void generateDailyReport() {
        // Generate report
    }
    
    @Scheduled(fixedRate = 60000) // Every minute
    public void checkHealth() {
        // Health check
    }
}
```

### 45. How do you handle file uploads in Spring Boot?
**Answer:**
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("File is empty");
    }
    
    String fileName = StringUtils.cleanPath(file.getOriginalFilename());
    Path path = Paths.get(uploadDir).resolve(fileName);
    Files.copy(file.getInputStream(), path, StandardCopyOption.REPLACE_EXISTING);
    
    return ResponseEntity.ok("File uploaded: " + fileName);
}
```
Configure max file size in `application.yml`:
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

### 46. How do you implement WebSocket in Spring Boot?
**Answer:**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}
```

### 47. How do you containerize a Spring Boot application?
**Answer:**
```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
Or use Spring Boot's built-in buildpack support:
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest
```

### 48. How do you implement health checks for Kubernetes?
**Answer:**
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```
Kubernetes deployment:
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

### 49. How do you handle graceful shutdown in Spring Boot?
**Answer:**
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```
This ensures in-flight requests complete before shutdown, connection pools are properly closed, and scheduled tasks finish.

### 50. How do you debug production issues without redeployment?
**Answer:**
- Use Spring Boot Actuator endpoints (`/env`, `/configprops`, `/mappings`)
- Enable debug logging dynamically via `/actuator/loggers`
- Use distributed tracing to track request flow
- Analyze thread dumps via `/actuator/threaddump`
- Use heap dumps for memory analysis
- Implement feature flags for toggling functionality
- Use tools like Arthas for live debugging

---

## Quick Reference Cheat Sheet

| Topic | Key Annotations/Classes |
|-------|------------------------|
| Configuration | `@Configuration`, `@Bean`, `@Value`, `@ConfigurationProperties` |
| REST | `@RestController`, `@GetMapping`, `@PostMapping`, `@RequestBody`, `@PathVariable` |
| Validation | `@Valid`, `@NotNull`, `@Size`, `@Email`, `@Pattern` |
| JPA | `@Entity`, `@Repository`, `@Query`, `@Transactional` |
| Security | `@EnableWebSecurity`, `@PreAuthorize`, `SecurityFilterChain` |
| Testing | `@SpringBootTest`, `@MockBean`, `@DataJpaTest`, `@WebMvcTest` |
| Caching | `@EnableCaching`, `@Cacheable`, `@CacheEvict` |
| Async | `@EnableAsync`, `@Async`, `CompletableFuture` |
| Scheduling | `@EnableScheduling`, `@Scheduled` |

---

*Good luck with your interview! 🚀*

