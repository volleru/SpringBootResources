# Spring Boot Interview Questions & Answers

---

## 1. What is Spring Boot and how is it different from Spring Framework?

**Answer:**
Spring Boot is an opinionated, convention-over-configuration extension of the Spring Framework that simplifies the setup and development of Spring applications.

Key differences:
- **Spring Framework** requires manual configuration (XML or Java-based) for almost everything — beans, data sources, MVC setup, etc.
- **Spring Boot** provides auto-configuration, embedded servers (Tomcat, Jetty), and starter dependencies, so you can run a production-ready app with minimal setup.
- Spring Boot apps can run as standalone JARs without deploying to an external server.

---

## 2. What is Spring Boot Auto-Configuration?

**Answer:**
Auto-configuration is Spring Boot's mechanism to automatically configure your application based on the dependencies present on the classpath. For example:
- If `spring-boot-starter-data-jpa` is on the classpath and a `DataSource` bean is available, Spring Boot auto-configures Hibernate and a `JpaTransactionManager`.
- It uses `@EnableAutoConfiguration` (included in `@SpringBootApplication`) which scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` to load configuration classes conditionally using `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.

You can exclude specific auto-configurations using:
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

---

## 3. What is `@SpringBootApplication`?

**Answer:**
`@SpringBootApplication` is a convenience annotation that combines three annotations:
- `@Configuration` — marks the class as a source of bean definitions.
- `@EnableAutoConfiguration` — enables Spring Boot's auto-configuration mechanism.
- `@ComponentScan` — enables component scanning in the current package and sub-packages.

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

---

## 4. What are Spring Boot Starters?

**Answer:**
Starters are pre-configured dependency descriptors that bundle commonly used libraries together so you don't have to hunt for compatible versions. Examples:
- `spring-boot-starter-web` — includes Spring MVC, embedded Tomcat, Jackson.
- `spring-boot-starter-data-jpa` — includes Hibernate, Spring Data JPA.
- `spring-boot-starter-security` — includes Spring Security.
- `spring-boot-starter-test` — includes JUnit, Mockito, AssertJ.

They follow the naming convention `spring-boot-starter-*`.

---

## 5. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

**Answer:**
All four are specializations of `@Component` and are used for component scanning. The difference is semantic and functional:

| Annotation | Layer | Extra Behavior |
|---|---|---|
| `@Component` | Generic | None |
| `@Service` | Business/Service | None (semantic) |
| `@Repository` | Data Access | Translates persistence exceptions to `DataAccessException` |
| `@Controller` | Web/Presentation | Handles HTTP requests, works with `@RequestMapping` |
| `@RestController` | REST API | `@Controller` + `@ResponseBody` |

---

## 6. What is Spring Boot Actuator?

**Answer:**
Spring Boot Actuator provides production-ready features to monitor and manage your application. It exposes endpoints over HTTP or JMX.

Common endpoints:
- `/actuator/health` — shows application health status.
- `/actuator/metrics` — shows application metrics.
- `/actuator/env` — shows environment properties.
- `/actuator/info` — shows custom app info.
- `/actuator/beans` — lists all Spring beans.

Add it with:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

## 7. How do you configure properties in Spring Boot?

**Answer:**
Spring Boot supports multiple ways to configure properties:

1. **`application.properties`** or **`application.yml`** in `src/main/resources`.
2. **Environment variables** — override properties at runtime.
3. **Command-line arguments** — `--server.port=9090`
4. **`@Value`** annotation — inject individual properties.
5. **`@ConfigurationProperties`** — bind a group of properties to a POJO.

```java
@ConfigurationProperties(prefix = "app.datasource")
public class DataSourceConfig {
    private String url;
    private String username;
    // getters and setters
}
```

Priority order (highest to lowest): command-line args > environment variables > `application.properties`.

---

## 8. What are Spring Boot Profiles?

**Answer:**
Profiles allow you to have different configurations for different environments (dev, test, prod).

Define profile-specific files:
- `application-dev.properties`
- `application-prod.properties`

Activate a profile:
```properties
spring.profiles.active=dev
```

Use `@Profile` to conditionally load beans:
```java
@Bean
@Profile("dev")
public DataSource devDataSource() { ... }
```

---

## 9. What is the difference between `@Bean` and `@Component`?

**Answer:**
| | `@Bean` | `@Component` |
|---|---|---|
| Defined in | `@Configuration` class method | Class level |
| Control | Full control over instantiation | Spring controls instantiation |
| Use case | Third-party classes you can't annotate | Your own classes |
| Detection | Explicit declaration | Component scanning |

Example of `@Bean`:
```java
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}
```

---

## 10. How does Spring Boot handle exceptions?

**Answer:**
Spring Boot provides several mechanisms:

1. **`@ExceptionHandler`** — handles exceptions in a specific controller.
2. **`@ControllerAdvice` / `@RestControllerAdvice`** — global exception handling across all controllers.
3. **`BasicErrorController`** — Spring Boot's default error handling that maps to `/error`.
4. **`@ResponseStatus`** — maps an exception to an HTTP status code.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

---

## 11. What is Spring Data JPA and how does it work with Spring Boot?

**Answer:**
Spring Data JPA is an abstraction over JPA that eliminates boilerplate DAO code. You define a repository interface extending `JpaRepository` or `CrudRepository`, and Spring Data generates the implementation at runtime.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}
```

Spring Boot auto-configures the `EntityManagerFactory`, `DataSource`, and transaction manager when `spring-boot-starter-data-jpa` is on the classpath.

---

## 12. What is the difference between `CrudRepository`, `JpaRepository`, and `PagingAndSortingRepository`?

**Answer:**
| Interface | Extends | Extra Features |
|---|---|---|
| `CrudRepository` | `Repository` | Basic CRUD: `save`, `findById`, `delete`, `findAll` |
| `PagingAndSortingRepository` | `CrudRepository` | `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository` | `PagingAndSortingRepository` | `flush()`, `saveAndFlush()`, `deleteInBatch()`, `getOne()` |

Use `JpaRepository` when you need JPA-specific features; `CrudRepository` for simpler use cases.

---

## 13. How do you secure a Spring Boot application?

**Answer:**
Add `spring-boot-starter-security` to get basic security auto-configured. For REST APIs:

1. **JWT (JSON Web Token)** — stateless authentication. Generate a token on login and validate on each request using a filter.
2. **OAuth2 / OpenID Connect** — using `spring-boot-starter-oauth2-resource-server`.
3. **Method-level security** — using `@PreAuthorize`, `@PostAuthorize` with `@EnableMethodSecurity`.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        ).sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
}
```

---

## 14. What is `@Transactional` and how does it work?

**Answer:**
`@Transactional` demarcates a transaction boundary. Spring uses AOP proxy to begin a transaction before the method and commit/rollback after.

Key attributes:
- `propagation` — defines how transactions relate (e.g., `REQUIRED`, `REQUIRES_NEW`).
- `isolation` — defines isolation level (e.g., `READ_COMMITTED`).
- `rollbackFor` — specifies which exceptions trigger rollback (default: unchecked exceptions only).
- `readOnly` — optimization hint for read-only operations.

```java
@Transactional(rollbackFor = Exception.class)
public void transferFunds(Long from, Long to, BigDecimal amount) {
    // database operations
}
```

**Important:** `@Transactional` only works on public methods and when called from outside the bean (due to proxy-based AOP).

---

## 15. What is the difference between `application.properties` and `application.yml`?

**Answer:**
Both serve the same purpose — externalizing configuration. The difference is format:

`application.properties`:
```properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
```

`application.yml`:
```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
```

YAML is more readable for hierarchical config and supports lists more naturally. Both can coexist; `application.properties` takes precedence if both define the same key.

---

## 16. How does Spring Boot embedded server work?

**Answer:**
Spring Boot bundles an embedded servlet container (Tomcat by default, or Jetty/Undertow) directly into the fat JAR. When you run `java -jar app.jar`, the embedded server starts and serves your application without any external server installation.

To switch to Jetty:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

## 17. What is Dependency Injection and what types does Spring support?

**Answer:**
Dependency Injection (DI) is a design pattern where dependencies are provided to a class rather than the class creating them itself.

Spring supports three types:
1. **Constructor Injection** (recommended) — dependencies injected via constructor; supports immutability and easy testing.
2. **Setter Injection** — dependencies injected via setter methods; useful for optional dependencies.
3. **Field Injection** — using `@Autowired` on fields; discouraged as it hides dependencies and makes testing harder.

```java
// Constructor injection (preferred)
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

---

## 18. What is Spring Boot DevTools?

**Answer:**
`spring-boot-devtools` is a developer productivity tool that provides:
- **Automatic restart** — restarts the application when classpath files change.
- **LiveReload** — triggers browser refresh when resources change.
- **Disabled caching** — disables template caching (Thymeleaf, FreeMarker) so changes reflect immediately.
- **H2 Console** — enables the H2 web console automatically.

It is automatically disabled in production (when running a fat JAR).

---

## 19. How do you write unit tests and integration tests in Spring Boot?

**Answer:**

**Unit Test** — test a single class in isolation using mocks:
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserById() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User()));
        assertNotNull(userService.getUserById(1L));
    }
}
```

**Integration Test** — loads the full Spring context:
```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerIntegrationTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturn200ForHealthEndpoint() throws Exception {
        mockMvc.perform(get("/actuator/health"))
               .andExpect(status().isOk());
    }
}
```

---

## 20. What is the difference between `@SpringBootTest` and `@WebMvcTest`?

**Answer:**
| Annotation | Loads | Use Case |
|---|---|---|
| `@SpringBootTest` | Full application context | Integration tests, end-to-end |
| `@WebMvcTest` | Only web layer (Controllers, Filters, etc.) | Testing controllers in isolation |
| `@DataJpaTest` | Only JPA layer | Testing repositories with in-memory DB |
| `@JsonTest` | Only JSON serialization | Testing Jackson serialization |

`@WebMvcTest` is faster than `@SpringBootTest` since it doesn't load service or repository beans — you mock them with `@MockBean`.

---

## 21. What is a Spring Boot fat JAR?

**Answer:**
A fat JAR (also called uber JAR or executable JAR) is a self-contained JAR file that includes:
- Your application's compiled classes.
- All dependency JARs nested inside.
- The embedded server.
- Spring Boot's launcher classes.

Built using the Spring Boot Maven/Gradle plugin:
```bash
mvn clean package
java -jar target/myapp-1.0.0.jar
```

The `spring-boot-maven-plugin` repackages the regular JAR into this format.

---

## 22. How do you handle circular dependencies in Spring Boot?

**Answer:**
Circular dependencies occur when Bean A depends on Bean B and Bean B depends on Bean A. Spring Boot 2.6+ fails fast on circular dependencies by default.

Solutions:
1. **Refactor** — extract common logic into a third bean (best approach).
2. **`@Lazy`** — inject one dependency lazily to break the cycle.
3. **Setter injection** — instead of constructor injection, allows Spring to create beans first and inject later.
4. **Allow circular references** (not recommended):
   ```properties
   spring.main.allow-circular-references=true
   ```

---

## 23. What is Spring Boot's externalized configuration order of precedence?

**Answer:**
From highest to lowest priority:
1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` env variable
3. OS environment variables
4. Java system properties (`-Dserver.port=9090`)
5. `application.properties` / `application.yml` outside the JAR
6. Profile-specific properties (`application-{profile}.properties`)
7. `application.properties` / `application.yml` inside the JAR
8. `@PropertySource` annotations
9. Default properties (`SpringApplication.setDefaultProperties`)

---

## 24. What is `CommandLineRunner` and `ApplicationRunner`?

**Answer:**
Both are interfaces used to execute code after the Spring application context is fully loaded.

```java
@Component
public class DataLoader implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // runs after startup — good for loading initial data
        System.out.println("Application started with args: " + Arrays.toString(args));
    }
}
```

`ApplicationRunner` is similar but receives `ApplicationArguments` instead of a raw `String[]`, giving more structured access to named and non-named arguments.

---

## 25. How do you implement caching in Spring Boot?

**Answer:**
Spring Boot provides cache abstraction via `@EnableCaching` and JSR-107 annotations.

```java
@SpringBootApplication
@EnableCaching
public class MyApplication { ... }

@Service
public class ProductService {
    @Cacheable("products")
    public Product getProductById(Long id) {
        // this method result is cached after first call
        return productRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}
```

Default cache provider is `ConcurrentHashMap`. For production, use Redis or Hazelcast by adding the appropriate starter.

---
