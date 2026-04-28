# Spring Boot — 20 Easy Scenario Interview Questions & Answers

> Covers auto-configuration, beans, REST, profiles, dependency injection, actuator, and common pitfalls.

---

## Q1. Your Spring Boot app fails to start with "No qualifying bean of type" error. What do you check first?

**Answer:**
Spring cannot find a bean to inject. Common causes:

1. Missing `@Component` / `@Service` / `@Repository` on the class
2. Class is outside the component scan package
3. Missing `@Bean` method in a `@Configuration` class

```java
// BAD — missing @Service, Spring won't scan it
public class UserService {
    public User findById(Long id) { ... }
}

// GOOD
@Service
public class UserService {
    public User findById(Long id) { ... }
}

// Also check your main class package — scan covers main package + sub-packages
@SpringBootApplication  // scans com.example and below
package com.example;    // UserService must be in com.example.** to be found
```

---

## Q2. What does `@SpringBootApplication` actually do?

**Answer:**
It is a shortcut for three annotations combined:

| Annotation | Purpose |
|---|---|
| `@SpringBootConfiguration` | Marks this as a configuration class (same as `@Configuration`) |
| `@EnableAutoConfiguration` | Tells Spring Boot to auto-configure beans based on classpath |
| `@ComponentScan` | Scans the current package and sub-packages for components |

```java
// These two are equivalent:

@SpringBootApplication
public class MyApp { ... }

// Same as:
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(basePackages = "com.example")
public class MyApp { ... }
```

---

## Q3. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

**Answer:**
All four are specializations of `@Component` and register the class as a Spring bean. The difference is **semantic** — they tell developers (and Spring) what role the class plays.

| Annotation | Layer | Extra Behavior |
|---|---|---|
| `@Component` | Generic | None |
| `@Service` | Business logic | None (marker only) |
| `@Repository` | Data access | Translates DB exceptions to Spring's `DataAccessException` |
| `@Controller` | Web layer (MVC) | Handles HTTP requests, returns view names |
| `@RestController` | Web layer (REST) | `@Controller` + `@ResponseBody` — returns JSON/XML |

```java
@Repository  // DB exceptions auto-translated
public class UserRepository { ... }

@Service     // business logic
public class UserService { ... }

@RestController  // REST API
public class UserController { ... }
```

---

## Q4. You have two beans of the same type. Spring throws `NoUniqueBeanDefinitionException`. How do you fix it?

**Answer:**
Use `@Primary` to mark the default bean, or `@Qualifier` to specify exactly which bean to inject.

```java
@Service
@Primary  // used by default when type matches
public class EmailNotificationService implements NotificationService { ... }

@Service
public class SmsNotificationService implements NotificationService { ... }

// Inject specific one with @Qualifier
@Service
public class OrderService {

    @Autowired
    @Qualifier("smsNotificationService")
    private NotificationService notificationService;
}
```

---

## Q5. What is the difference between `application.properties` and `application.yml`?

**Answer:**
Both configure the same properties — `yml` is just a more readable hierarchical format. Spring Boot supports both; pick one per project.

```properties
# application.properties
server.port=8081
spring.datasource.url=jdbc:mysql://localhost/mydb
spring.datasource.username=root
```

```yaml
# application.yml — same config, cleaner for nested properties
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost/mydb
    username: root
```

> If both files exist, `application.properties` takes precedence.

---

## Q6. How do you run different configurations for dev, test, and prod environments?

**Answer:**
Use Spring Profiles. Create profile-specific property files and activate the right profile at runtime.

```
application.yml           ← shared defaults
application-dev.yml       ← dev overrides
application-prod.yml      ← prod overrides
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb  # in-memory for dev

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-server/mydb
```

```java
// Activate via command line
// java -jar app.jar --spring.profiles.active=prod

// Or in application.yml
spring:
  profiles:
    active: dev

// Bean active only in a specific profile
@Service
@Profile("dev")
public class MockPaymentService implements PaymentService { ... }

@Service
@Profile("prod")
public class RealPaymentService implements PaymentService { ... }
```

---

## Q7. What is auto-configuration and how does Spring Boot decide what to configure?

**Answer:**
Spring Boot scans the classpath and automatically configures beans based on what it finds. For example, if `spring-boot-starter-data-jpa` is on the classpath, it auto-configures a `DataSource`, `EntityManagerFactory`, and `JpaTransactionManager`.

It works through `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.

```java
// Spring Boot's internal auto-config (simplified example)
@Configuration
@ConditionalOnClass(DataSource.class)          // only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)    // only if user hasn't defined their own
public class DataSourceAutoConfiguration {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(); // auto-creates connection pool
    }
}
```

> You can exclude specific auto-configurations:
```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

---

## Q8. What is the difference between `@Bean` and `@Component`?

**Answer:**

| | `@Component` | `@Bean` |
|---|---|---|
| Where | On the class | On a method inside `@Configuration` |
| Use case | Your own classes | Third-party classes you can't annotate |
| Detection | Component scan | Explicit method call |

```java
// @Component — annotate your own class
@Component
public class EmailService { ... }

// @Bean — register a third-party class you don't own
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

---

## Q9. How do you inject values from `application.properties` into a bean?

**Answer:**
Use `@Value` for single values or `@ConfigurationProperties` for grouped config.

```java
// application.properties
app.name=MyApp
app.max-retries=3

// Option 1: @Value — simple, one-off
@Service
public class AppService {

    @Value("${app.name}")
    private String appName;

    @Value("${app.max-retries:5}") // default = 5 if not set
    private int maxRetries;
}

// Option 2: @ConfigurationProperties — better for grouped config
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int maxRetries;
    // getters and setters required
}
```

---

## Q10. What is the difference between `@GetMapping` and `@RequestMapping(method = GET)`?

**Answer:**
`@GetMapping` is a shortcut (composed annotation) for `@RequestMapping(method = RequestMethod.GET)`. Both do the same thing — `@GetMapping` is just cleaner.

```java
// Verbose — old style
@RequestMapping(value = "/users", method = RequestMethod.GET)
public List<User> getUsers() { ... }

// Clean — preferred
@GetMapping("/users")
public List<User> getUsers() { ... }

// Other shortcuts
@PostMapping("/users")
@PutMapping("/users/{id}")
@DeleteMapping("/users/{id}")
@PatchMapping("/users/{id}")
```

---

## Q11. How do you handle exceptions globally in a Spring Boot REST API?

**Answer:**
Use `@ControllerAdvice` + `@ExceptionHandler` to catch exceptions across all controllers in one place.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleBadRequest(IllegalArgumentException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Something went wrong");
    }
}
```

---

## Q12. What is the difference between `@PathVariable` and `@RequestParam`?

**Answer:**

| | `@PathVariable` | `@RequestParam` |
|---|---|---|
| Location | Part of the URL path | Query string `?key=value` |
| Required? | Yes (by default) | Optional (can have default) |
| Example URL | `/users/42` | `/users?id=42` |

```java
// @PathVariable — /users/42
@GetMapping("/users/{id}")
public User getById(@PathVariable Long id) { ... }

// @RequestParam — /users?name=John&page=1
@GetMapping("/users")
public List<User> search(
    @RequestParam String name,
    @RequestParam(defaultValue = "0") int page
) { ... }
```

---

## Q13. What is Spring Boot Actuator and what can it tell you about your app?

**Answer:**
Actuator exposes built-in endpoints to monitor and manage a running Spring Boot application — health, metrics, environment info, beans, etc.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml — expose all endpoints
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

| Endpoint | URL | What it shows |
|---|---|---|
| Health | `/actuator/health` | App health status |
| Info | `/actuator/info` | App info (version, etc.) |
| Metrics | `/actuator/metrics` | JVM, HTTP, DB metrics |
| Beans | `/actuator/beans` | All registered Spring beans |
| Env | `/actuator/env` | All environment properties |
| Loggers | `/actuator/loggers` | Change log levels at runtime |

---

## Q14. What is the difference between `@RequestBody` and `@ResponseBody`?

**Answer:**

| | Direction | Purpose |
|---|---|---|
| `@RequestBody` | Incoming | Deserializes HTTP request body (JSON → Java object) |
| `@ResponseBody` | Outgoing | Serializes Java object → JSON response body |

```java
@RestController  // @RestController already includes @ResponseBody on all methods
public class UserController {

    // @RequestBody — reads JSON from request body and maps to User object
    @PostMapping("/users")
    public User createUser(@RequestBody User user) {
        return userService.save(user); // returned object auto-serialized to JSON
    }
}

// Without @RestController, you need @ResponseBody explicitly
@Controller
public class UserController {

    @PostMapping("/users")
    @ResponseBody
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

---

## Q15. How do you change the default port of a Spring Boot application?

**Answer:**
Three ways — property file, command line, or programmatically.

```yaml
# application.yml
server:
  port: 9090
```

```bash
# Command line — overrides application.yml
java -jar app.jar --server.port=9090
```

```java
// Programmatically
@SpringBootApplication
public class MyApp implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);
    }
}
```

> Use port `0` to let Spring Boot pick a **random available port** (useful in tests).

---

## Q16. What is the Bean lifecycle in Spring Boot?

**Answer:**
Spring manages bean creation, initialization, and destruction through a defined lifecycle.

```
1. Instantiation     — Spring creates the object (calls constructor)
2. Dependency Inject — @Autowired fields and setters are populated
3. @PostConstruct    — Your init method is called (setup logic)
4. Bean in use       — Application uses the bean
5. @PreDestroy       — Your cleanup method is called before shutdown
6. Destruction       — Bean is garbage collected
```

```java
@Service
public class CacheService {

    @PostConstruct
    public void init() {
        // runs once after bean is fully created and injected
        System.out.println("Loading cache...");
    }

    @PreDestroy
    public void cleanup() {
        // runs before app shuts down
        System.out.println("Clearing cache...");
    }
}
```

---

## Q17. What is the difference between `@Autowired` on a field vs constructor injection?

**Answer:**
Constructor injection is preferred — it makes dependencies explicit, supports immutability, and makes the class easier to test without Spring.

```java
// Field injection — easy but hides dependencies, hard to test
@Service
public class OrderService {

    @Autowired
    private UserRepository userRepo; // hidden dependency

    @Autowired
    private PaymentService paymentService;
}

// Constructor injection — preferred
@Service
public class OrderService {

    private final UserRepository userRepo;
    private final PaymentService paymentService;

    // @Autowired optional if only one constructor (Spring 4.3+)
    public OrderService(UserRepository userRepo, PaymentService paymentService) {
        this.userRepo = userRepo;
        this.paymentService = paymentService;
    }
}

// With Lombok — cleanest
@Service
@RequiredArgsConstructor
public class OrderService {
    private final UserRepository userRepo;
    private final PaymentService paymentService;
}
```

---

## Q18. How do you validate request body fields in a Spring Boot REST API?

**Answer:**
Use Bean Validation (`@Valid` + validation annotations) on the request body.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
// DTO with validation constraints
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Age must be at least 18")
    private int age;
}

// Controller — @Valid triggers validation
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
    return ResponseEntity.ok(userService.create(request));
}

// Handle validation errors globally
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }
}
```

---

## Q19. What is the difference between `@SpringBootTest` and `@WebMvcTest`?

**Answer:**

| | `@SpringBootTest` | `@WebMvcTest` |
|---|---|---|
| Context loaded | Full application context | Web layer only (controllers) |
| Speed | Slow | Fast |
| Use for | Integration tests | Unit tests for controllers |
| Beans loaded | All beans | Controller + `@ControllerAdvice` only |

```java
// @WebMvcTest — fast, tests only the controller layer
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;

    @MockBean UserService userService; // mock the service layer

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Alice"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}

// @SpringBootTest — full context, use for integration tests
@SpringBootTest
@AutoConfigureMockMvc
class UserIntegrationTest {

    @Autowired MockMvc mockMvc;

    @Test
    void shouldCreateAndFetchUser() throws Exception {
        mockMvc.perform(post("/users").contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"Alice\",\"email\":\"alice@test.com\",\"age\":25}"))
            .andExpect(status().isOk());
    }
}
```

---

## Q20. Your REST API returns a 500 error but you see no logs. How do you debug it?

**Answer:**
Enable debug logging and use Actuator to inspect the app state.

```yaml
# application.yml — increase log level
logging:
  level:
    root: DEBUG
    org.springframework.web: DEBUG
    com.example: DEBUG

# Show full error details in responses (only in dev!)
server:
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: on_param   # add ?trace=true to URL
```

```java
// Add global exception handler to log and return meaningful errors
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAll(Exception ex, HttpServletRequest request) {
        log.error("Unhandled error on {} {}", request.getMethod(), request.getRequestURI(), ex);
        return ResponseEntity.internalServerError().body("Internal error: " + ex.getMessage());
    }
}
```

```bash
# Check actuator health and logs endpoint
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/loggers/com.example
```

---

## Quick Reference Cheat Sheet

| Topic | Key Point |
|---|---|
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@Service` vs `@Repository` | `@Repository` translates DB exceptions; `@Service` is a marker |
| Two beans, same type | Use `@Primary` or `@Qualifier` |
| Properties vs YAML | Both work; `.properties` wins if both exist |
| Profiles | `application-{profile}.yml` + `--spring.profiles.active=prod` |
| `@Bean` vs `@Component` | `@Bean` for third-party classes; `@Component` for your own |
| `@PathVariable` | From URL path `/users/{id}` |
| `@RequestParam` | From query string `/users?name=Alice` |
| Constructor injection | Preferred over field injection |
| `@WebMvcTest` | Fast controller tests — no full context |
| `@SpringBootTest` | Full integration tests — slow but complete |
| Validation | `@Valid` + `@NotBlank`, `@Email`, `@Min` on DTO |
| Global error handling | `@RestControllerAdvice` + `@ExceptionHandler` |
| Actuator | `/actuator/health`, `/actuator/metrics`, `/actuator/beans` |
