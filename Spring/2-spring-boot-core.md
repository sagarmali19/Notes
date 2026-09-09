# Spring Boot Core Features — Interview-Ready Notes
### For Java Backend Developer Interviews

---

## 1. Spring Boot Architecture

### What Spring Boot Solves
Traditional Spring required extensive **XML/manual configuration** — defining `DispatcherServlet`, view resolvers, data sources, transaction managers, etc. by hand. Spring Boot removes this burden through **convention over configuration**, **auto-configuration**, and an **embedded server**, letting you build a production-ready app with minimal setup.

### Layered Architecture (Conceptual)
Spring Boot apps are typically organized in layers:

```
┌─────────────────────────────────────┐
│   Presentation Layer (Controller)    │  @RestController / @Controller
├─────────────────────────────────────┤
│   Business/Service Layer             │  @Service
├─────────────────────────────────────┤
│   Data Access Layer (Repository)     │  @Repository / Spring Data JPA
├─────────────────────────────────────┤
│   Database                           │  MySQL / PostgreSQL / MongoDB...
└─────────────────────────────────────┘
```

```java
@RestController
@RequestMapping("/api/products")
class ProductController {
    private final ProductService service;
    ProductController(ProductService service) { this.service = service; }

    @GetMapping("/{id}")
    public ProductDto getProduct(@PathVariable Long id) {
        return service.findById(id); // delegates to service layer
    }
}

@Service
class ProductService {
    private final ProductRepository repository;
    ProductService(ProductRepository repository) { this.repository = repository; }

    public ProductDto findById(Long id) {
        Product p = repository.findById(id).orElseThrow();
        return new ProductDto(p.getId(), p.getName());
    }
}

@Repository
interface ProductRepository extends JpaRepository<Product, Long> {}
```

### Internal Architecture — Core Building Blocks

1. **Spring Boot Starters** — curated dependency bundles (see Section 4).
2. **Auto-Configuration** — automatically configures beans based on classpath contents (see Section 3).
3. **Embedded Server** — Tomcat (default), Jetty, or Undertow packaged **inside** the JAR — no external app server (WAR + separate Tomcat install) needed.
4. **`SpringApplication` class** — bootstraps the application: creates the `ApplicationContext`, triggers auto-configuration, starts the embedded server.
5. **Actuator** — production-monitoring endpoints (see Section 8).
6. **Externalized Configuration** — `application.properties`/`.yml`, environment variables, command-line args (see Sections 6–7).

### The `main()` Entry Point
```java
@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class ShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```

**What happens on `SpringApplication.run()`:**
1. Determines application type (Servlet web / Reactive web / none).
2. Creates the appropriate `ApplicationContext` (e.g., `AnnotationConfigServletWebServerApplicationContext`).
3. Loads `ApplicationListener`s and `ApplicationContextInitializer`s.
4. Processes `@SpringBootApplication` → triggers component scanning + auto-configuration.
5. Starts the embedded servlet container (Tomcat by default) on the configured port (default `8080`).
6. Publishes `ApplicationReadyEvent` once fully started.

### `@SpringBootApplication` Breakdown
```java
@SpringBootConfiguration   // specialization of @Configuration
@EnableAutoConfiguration   // triggers auto-configuration mechanism
@ComponentScan             // scans current package + sub-packages for @Component, @Service, etc.
public @interface SpringBootApplication { }
```
**Interview tip:** because `@ComponentScan` scans from the package of the annotated class **downward**, the convention is to place your main application class in the **root package** (e.g., `com.example.shopapp`) so all sub-packages (`controller`, `service`, `repository`) get scanned automatically.

### Packaging: JAR vs WAR
- **Fat/Executable JAR** (default, preferred) — contains your code + all dependencies + embedded Tomcat, run via `java -jar app.jar`.
- **WAR** — for deployment to an external servlet container (legacy enterprise setups); requires extending `SpringBootServletInitializer`.

```java
public class ShopApplication extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(ShopApplication.class);
    }
}
```

---

## 2. Spring Boot Annotations

Spring Boot builds on core Spring annotations and adds its own. Group these mentally by purpose for interviews.

### Bootstrapping
| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | Entry-point annotation: config + auto-config + component scan |
| `@EnableAutoConfiguration` | Enables classpath-based auto-configuration |
| `@SpringBootConfiguration` | Marks the class as the primary configuration source |

### Stereotype / Component
| Annotation | Purpose |
|---|---|
| `@Component` | Generic Spring-managed bean |
| `@Service` | Business/service layer bean |
| `@Repository` | Persistence layer bean; adds exception translation |
| `@Controller` | MVC controller returning view names |
| `@RestController` | `@Controller` + `@ResponseBody` — returns data (JSON) directly |
| `@ComponentScan` | Configures base packages to scan for components |

### Dependency Injection
| Annotation | Purpose |
|---|---|
| `@Autowired` | Injects a bean by type |
| `@Qualifier` | Disambiguates between multiple beans of the same type |
| `@Primary` | Marks a bean as the default choice when ambiguous |
| `@Value` | Injects a value from properties/environment |

```java
@Component
class MailService {
    @Value("${mail.from.address}")
    private String fromAddress;
}
```

### Configuration
| Annotation | Purpose |
|---|---|
| `@Configuration` | Marks a class as a source of bean definitions |
| `@Bean` | Declares a bean, method-level, inside a `@Configuration` class |
| `@PropertySource` | Loads a specific `.properties` file |
| `@ConfigurationProperties` | Binds a whole group of properties to a POJO (type-safe config) |

```java
@Component
@ConfigurationProperties(prefix = "app.mail")
class MailProperties {
    private String host;
    private int port;
    // getters/setters — Spring Boot binds app.mail.host, app.mail.port automatically
}
```

### Web / MVC
| Annotation | Purpose |
|---|---|
| `@RequestMapping` | Maps HTTP requests to handler methods/classes |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` | Shortcuts for specific HTTP verbs |
| `@PathVariable` | Binds a URI template variable to a method param |
| `@RequestParam` | Binds a query parameter |
| `@RequestBody` | Deserializes the HTTP request body into a Java object |
| `@ResponseBody` | Serializes the return value directly into the HTTP response body |
| `@ResponseStatus` | Sets the HTTP status code of the response |
| `@ExceptionHandler` | Handles exceptions thrown within a controller |
| `@ControllerAdvice` / `@RestControllerAdvice` | Global exception handling / cross-cutting controller logic |

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ProductNotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }
}
```

### Testing
| Annotation | Purpose |
|---|---|
| `@SpringBootTest` | Loads the full application context for integration tests |
| `@WebMvcTest` | Loads only the MVC layer (controllers) — faster, sliced test |
| `@DataJpaTest` | Loads only JPA-related components |
| `@MockBean` | Injects a Mockito mock into the Spring context |

### Conditional (used heavily in auto-configuration internally)
| Annotation | Purpose |
|---|---|
| `@ConditionalOnClass` | Bean registered only if a specific class is on the classpath |
| `@ConditionalOnMissingBean` | Bean registered only if no other bean of that type exists |
| `@ConditionalOnProperty` | Bean registered only if a property has a given value |

---

## 3. Auto-Configuration

### What it is
Auto-configuration is Spring Boot's mechanism for **automatically configuring beans based on what's on the classpath and what's already defined**, so you don't hand-write configuration for common scenarios.

**Example:** if `spring-boot-starter-data-jpa` and an H2/MySQL driver are on the classpath, Spring Boot auto-configures a `DataSource`, `EntityManagerFactory`, and `TransactionManager` — no manual bean definitions needed.

### How It Works Internally
1. `@EnableAutoConfiguration` (part of `@SpringBootApplication`) triggers the mechanism.
2. Spring Boot reads a list of auto-configuration classes registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (in older versions: `spring.factories`).
3. Each auto-configuration class is annotated with **conditional annotations** — it only "activates" if its conditions are satisfied.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)              // only if javax.sql.DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)         // only if user hasn't defined their own DataSource
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

### Key Conditional Annotations
| Annotation | Trigger Condition |
|---|---|
| `@ConditionalOnClass` | A specified class is present on the classpath |
| `@ConditionalOnMissingClass` | A specified class is **absent** |
| `@ConditionalOnBean` | A specified bean already exists in context |
| `@ConditionalOnMissingBean` | No bean of that type exists — **critical for letting user overrides win** |
| `@ConditionalOnProperty` | A property is set to a specific value (or just present) |
| `@ConditionalOnWebApplication` | App is a web application |
| `@ConditionalOnResource` | A specific resource file exists |

### You Can Always Override Auto-Configuration
Because auto-configured beans use `@ConditionalOnMissingBean`, **your own `@Bean` definition always wins**:
```java
@Configuration
class MyDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // Your custom DataSource takes priority; Spring Boot's default auto-config backs off
        return DataSourceBuilder.create()
                .url("jdbc:mysql://localhost:3306/shopdb")
                .username("root")
                .password("secret")
                .build();
    }
}
```

### Debugging Auto-Configuration
Run with `--debug` flag (or set `debug=true` in properties) to print an **auto-configuration report** showing which auto-configs were applied ("Positive matches") and which were skipped ("Negative matches") and why:
```bash
java -jar app.jar --debug
```

### Excluding Specific Auto-Configurations
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class ShopApplication { }
```
or in properties:
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**Interview one-liner:** *"Auto-configuration inspects the classpath and existing bean definitions via `@Conditional...` annotations, and registers sensible-default beans only when they're needed and not already user-defined — this is what makes Spring Boot 'opinionated but overridable.'"*

---

## 4. Spring Boot — Dependency Management (Starters)

### The Problem It Solves
In plain Spring, you had to manually pick compatible versions of dozens of libraries (Spring Core, Jackson, Hibernate, Tomcat, logging, etc.) — version mismatches caused frequent, hard-to-debug conflicts.

### Spring Boot Starters
**Starters** are curated dependency descriptors — a single dependency that pulls in everything commonly needed for a feature, with **compatible, tested versions**.

| Starter | Brings In |
|---|---|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson (JSON) |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, JDBC support |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Spring Test |
| `spring-boot-starter-actuator` | Production monitoring endpoints |
| `spring-boot-starter-validation` | Hibernate Validator (Bean Validation / JSR-380) |

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- no <version> needed! managed by parent BOM -->
</dependency>
```

### The `spring-boot-starter-parent` (Maven)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
```
This parent POM:
- Manages **default versions** for hundreds of common dependencies (via its own parent, `spring-boot-dependencies`, which acts as a **BOM — Bill Of Materials**).
- Sets sensible plugin defaults (compiler source/target, packaging plugin).
- You omit `<version>` tags for anything covered by the BOM — Spring Boot guarantees they're mutually compatible.

### If You Can't Extend `spring-boot-starter-parent` (e.g., your org has its own parent POM)
Use dependency management import instead:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Gradle Equivalent
```groovy
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5' // handles BOM-style version management
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web' // no version needed
}
```

### Overriding a Managed Version
```xml
<properties>
    <jackson.version>2.17.0</jackson.version> <!-- overrides the BOM-managed Jackson version -->
</properties>
```

**Interview tip:** Be ready to explain **BOM (Bill of Materials)** — a POM whose sole job is to declare a consistent, tested set of dependency versions (via `<dependencyManagement>`), which child projects "import" rather than inherit directly.

---

## 5. Best Practices for Spring Boot Applications

### Project Structure
Organize by **feature/layer**, keep the main class in the root package:
```
com.example.shopapp
 ├── ShopApplication.java
 ├── controller/
 ├── service/
 ├── repository/
 ├── model/ (entities)
 ├── dto/
 ├── config/
 └── exception/
```

### Configuration & Secrets
- **Never hardcode secrets** (DB passwords, API keys) — externalize via environment variables or a secrets manager (Vault, AWS Secrets Manager).
- Use **Spring Profiles** to separate environment-specific config:
```properties
# application-dev.properties
spring.datasource.url=jdbc:h2:mem:devdb

# application-prod.properties
spring.datasource.url=jdbc:mysql://prod-db:3306/shopdb
```
```java
@Profile("dev")
@Configuration
class DevConfig { }
```
Activate via: `spring.profiles.active=prod` (in properties, env var, or `-D` flag).

### Dependency Injection
- Prefer **constructor injection** (immutability, testability, fail-fast).
- Avoid field injection in production code.

### Exception Handling
- Centralize with `@RestControllerAdvice` + `@ExceptionHandler` rather than try/catch in every controller.
- Return consistent, structured error responses (don't leak stack traces to clients).

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex) {
        return new ErrorResponse("Something went wrong");
    }
}
```

### Layered Architecture Discipline
- Controllers stay thin — delegate to services.
- Business logic lives in `@Service` classes, not in controllers or repositories.
- Use **DTOs** at API boundaries — never expose JPA entities directly (avoids leaking persistence details, lazy-loading issues, and unwanted serialization of relationships).

### Validation
```java
class ProductRequest {
    @NotBlank
    private String name;

    @Positive
    private double price;
}

@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody ProductRequest request) { ... }
```

### Logging
- Use SLF4J (`Logger`), not `System.out.println`.
- Use structured, leveled logging (`DEBUG`/`INFO`/`WARN`/`ERROR`) and avoid logging sensitive data (PII, passwords).

```java
private static final Logger log = LoggerFactory.getLogger(ProductService.class);
log.info("Created product with id={}", product.getId());
```

### Performance & Reliability
- Use connection pooling (HikariCP — Spring Boot's default) tuned for expected load.
- Enable caching (`@Cacheable`) for expensive/read-heavy operations.
- Use pagination (`Pageable`) for large result sets — never return unbounded lists.
- Set timeouts on outbound HTTP calls (`RestTemplate`/`WebClient`) — don't let a downstream hang forever.

### Testing
- Unit test services with mocked dependencies (Mockito).
- Use `@WebMvcTest`/`@DataJpaTest` for fast, sliced integration tests instead of always loading the full context with `@SpringBootTest`.
- Use **Testcontainers** for realistic DB integration tests instead of only H2 in-memory (behavior parity with production DB).

### Observability
- Enable **Actuator** endpoints for health checks and metrics (Section 8).
- Integrate structured logging + metrics (Micrometer) with your monitoring stack (Prometheus/Grafana).

### Security
- Never trust client input — validate everything server-side.
- Use HTTPS, proper CORS config, and Spring Security for authn/authz rather than rolling your own.
- Keep dependencies updated (`mvn versions:display-dependency-updates`) to avoid known CVEs.

---

## 6. Application Properties

### What It Is
`src/main/resources/application.properties` is Spring Boot's default **externalized configuration** file — key-value pairs that configure the app without touching code.

```properties
# Server config
server.port=8081
server.servlet.context-path=/api

# Datasource config
spring.datasource.url=jdbc:mysql://localhost:3306/shopdb
spring.datasource.username=root
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Logging
logging.level.org.springframework=INFO
logging.level.com.example=DEBUG

# Custom app properties
app.mail.host=smtp.example.com
app.mail.port=587
```

### Reading Values in Code

**Option A — `@Value` (single property)**
```java
@Component
class MailSender {
    @Value("${app.mail.host}")
    private String host;

    @Value("${app.mail.port:25}") // ':25' = default value if property is absent
    private int port;
}
```

**Option B — `@ConfigurationProperties` (grouped, type-safe — preferred for related properties)**
```java
@Component
@ConfigurationProperties(prefix = "app.mail")
class MailProperties {
    private String host;
    private int port;
    // getters and setters required for binding
}
```
```java
@Configuration
@EnableConfigurationProperties(MailProperties.class) // if MailProperties isn't itself a @Component
class MailConfig { }
```

### Profile-Specific Properties
Spring Boot automatically picks up `application-{profile}.properties`:
```
application.properties          (common/default config)
application-dev.properties       (dev overrides)
application-prod.properties      (prod overrides)
```
Activate with:
```properties
spring.profiles.active=dev
```
or:
```bash
java -jar app.jar --spring.profiles.active=prod
```

### Property Precedence (Highest to Lowest — commonly asked)
1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` env variable
3. JVM system properties (`-Dserver.port=9090`)
4. OS environment variables
5. Profile-specific properties files (`application-{profile}.properties`)
6. `application.properties` / `application.yml`
7. `@PropertySource` annotated sources
8. Default properties set in code

**Interview tip:** the higher-precedence source **always wins** — this lets you override a base config (baked into the JAR) with environment-specific values at deploy time without rebuilding.

---

## 7. YAML Configuration

### Why YAML
YAML (`application.yml`) offers the same configuration capability as `.properties` but with a **hierarchical, more readable** structure — especially valuable for deeply nested config (great for lists/maps too).

### Equivalent Config in YAML
```yaml
server:
  port: 8081
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/shopdb
    username: root
    password: secret
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

logging:
  level:
    org.springframework: INFO
    com.example: DEBUG

app:
  mail:
    host: smtp.example.com
    port: 587
```

### YAML Lists
```yaml
app:
  allowed-origins:
    - https://example.com
    - https://admin.example.com
```
```java
@ConfigurationProperties(prefix = "app")
class CorsProperties {
    private List<String> allowedOrigins;
}
```

### Multiple Profiles in a Single YAML File
YAML supports **document separators (`---`)** so you can define all profiles in one file:
```yaml
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8080

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### YAML vs Properties — Comparison

| Aspect | `.properties` | `.yml` |
|---|---|---|
| Structure | Flat, dot-separated keys | Hierarchical/nested |
| Readability for nested config | Poor (repetitive prefixes) | Excellent |
| Native list/map support | Awkward (`app.list[0]=x`) | Clean (`- item`) |
| Multi-profile in one file | Not directly supported (needs separate files) | Supported via `---` separators |
| Comments | `#` | `#` |
| Common pitfall | — | **Indentation-sensitive** — a misplaced space breaks the whole file; also can't mix tabs and spaces |

**Interview tip:** Functionally, `.properties` and `.yml` are interchangeable — Spring Boot's `PropertySource` abstraction treats them the same after loading. Choice is largely a readability/team-preference decision. **You cannot use `@PropertySource` to load a `.yml` file directly** — it only supports `.properties`; for YAML you'd use `YamlPropertySourceFactory` as a workaround.

---

## 8. Actuator

### What It Is
`spring-boot-starter-actuator` adds **production-ready monitoring and management endpoints** to your application — health checks, metrics, environment info, and more — without writing any code.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Key Built-In Endpoints
| Endpoint | Purpose |
|---|---|
| `/actuator/health` | Application health status (UP/DOWN), including DB/disk/custom checks |
| `/actuator/info` | Arbitrary app info (build version, git commit, etc.) |
| `/actuator/metrics` | JVM, HTTP, and custom metrics (via Micrometer) |
| `/actuator/env` | All environment properties and their sources |
| `/actuator/beans` | Every Spring bean currently loaded in the context |
| `/actuator/mappings` | All `@RequestMapping` endpoints in the app |
| `/actuator/loggers` | View/change log levels **at runtime**, without redeploying |
| `/actuator/threaddump` | JVM thread dump — useful for debugging deadlocks/hangs |
| `/actuator/heapdump` | Downloadable heap dump for memory analysis |
| `/actuator/shutdown` | Gracefully shuts down the app (disabled by default — dangerous if exposed) |

### Enabling/Exposing Endpoints
By default, only `/health` and `/info` are exposed over HTTP for security. You must explicitly opt in to more:
```properties
management.endpoints.web.exposure.include=health,info,metrics,loggers
# Or expose everything (use with caution, typically behind auth):
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```

### Custom Health Indicator
```java
@Component
class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean gatewayUp = checkGatewayConnectivity();
        if (gatewayUp) {
            return Health.up().withDetail("gateway", "reachable").build();
        }
        return Health.down().withDetail("gateway", "unreachable").build();
    }

    private boolean checkGatewayConnectivity() {
        // custom logic, e.g. ping an external service
        return true;
    }
}
```
This automatically contributes to the aggregate status returned by `/actuator/health`.

### Custom Metrics with Micrometer
```java
@Component
class OrderMetrics {
    private final Counter orderCounter;

    OrderMetrics(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
                .description("Total orders created")
                .register(registry);
    }

    public void recordOrder() {
        orderCounter.increment();
    }
}
```

### Securing Actuator
Actuator endpoints can leak sensitive internals (`/env`, `/beans`, `/heapdump`) — **always secure them** in production:
```java
@Configuration
class ActuatorSecurityConfig {
    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health").permitAll()
            .requestMatchers("/actuator/**").hasRole("ADMIN")
        );
        return http.build();
    }
}
```

**Interview one-liner:** *"Actuator exposes operational metadata — health, metrics, env, loggers — as HTTP endpoints or JMX beans, integrating cleanly with monitoring tools like Prometheus/Grafana; sensitive endpoints must be explicitly exposed and secured."*

---

## 9. DevTools

### What It Is
`spring-boot-devtools` is a set of **developer-productivity tools** meant for local development only — automatically excluded from production builds (Spring Boot detects a "packaged" JAR/WAR and disables DevTools features then).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

### Key Features

**1. Automatic Restart**
When files on the classpath change (e.g., you edit a `.java` file and recompile), DevTools automatically restarts the application — much faster than a full JVM restart because it uses **two classloaders**:
- **Base classloader** — loads unchanging classes (third-party JARs) — stays alive.
- **Restart classloader** — loads your application's own classes — gets thrown away and reloaded on each change.

This makes restarts noticeably faster than a cold start.

**2. LiveReload**
DevTools starts an embedded **LiveReload server** (default port `35729`). With the LiveReload browser extension installed, your browser **automatically refreshes** when a static resource or template changes — no manual F5 needed.

**3. Property Defaults for Development**
DevTools automatically disables template caching (Thymeleaf, FreeMarker) so template edits show up immediately without a restart.

**4. Global Settings (Across Projects)**
You can set DevTools preferences globally in `~/.spring-boot-devtools.properties`, applying to every project on your machine.

### Configuration Options
```properties
# Exclude certain paths from triggering a restart (e.g., static resources)
spring.devtools.restart.exclude=static/**,public/**

# Disable restart entirely (keep only LiveReload) — useful for large apps where classloader restarts are noisy
spring.devtools.restart.enabled=true

# Add extra paths to watch for restart triggers
spring.devtools.restart.additional-paths=src/main/other

# Disable LiveReload if unwanted
spring.devtools.livereload.enabled=true
```

### Remote Debugging Support
DevTools can be used with a **remote application** too (e.g., a container running in a dev/staging environment) via `spring.devtools.remote.secret`, allowing remote restart/LiveReload — but this is a niche/less-commonly-asked feature.

### Important Interview Notes
- DevTools is a **development-time-only** dependency (`<optional>true</optional>` in Maven; `developmentOnly` configuration in Gradle) — Spring Boot automatically disables its restart features if the app is run via `java -jar` from a fully-packaged, "production" artifact, or if `spring.devtools.restart.enabled=false`.
- It does **not** affect production behavior/performance since it's excluded from the final artifact if configured properly (`developmentOnly` in Gradle ensures it's not included in a `bootJar`).
- Restart is not the same as **hot code replace (HCR)** — DevTools restarts the Spring context (fast, seconds), whereas JVM-level HCR (via IDE debugger) reloads bytecode inside a running JVM instantly but has limitations on structural changes (e.g., adding new methods).

```groovy
// Gradle — ensures devtools is dev-only, excluded from the final artifact
dependencies {
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
}
```

**Interview one-liner:** *"DevTools speeds up the local dev loop via fast application restarts (dual classloader trick), LiveReload for browser auto-refresh, and dev-friendly property defaults — it's automatically excluded from production packaging."*

---

## Quick-Fire Interview Summary Table

| Topic | One-Line Answer |
|---|---|
| Architecture | Layered app (Controller→Service→Repository) bootstrapped by `SpringApplication`, powered by starters + auto-config + embedded server |
| Annotations | Grouped by purpose: bootstrapping, stereotypes, DI, config, web/MVC, testing, conditional |
| Auto-configuration | Classpath + `@Conditional...` driven bean registration; always overridable via your own `@Bean` |
| Dependency Management | Starters + BOM (`spring-boot-dependencies`) guarantee compatible versions without manual `<version>` tags |
| Best Practices | Constructor injection, DTOs at boundaries, centralized exception handling, profiles for env config, secure secrets, observability |
| Application Properties | `application.properties` — externalized key-value config; precedence: CLI args > env vars > profile files > base file |
| YAML Configuration | Same config, hierarchical syntax; supports multi-profile docs via `---`; indentation-sensitive |
| Actuator | Exposes `/health`, `/metrics`, `/env`, etc. for ops monitoring; secure and selectively expose in production |
| DevTools | Dev-only: fast restarts (dual classloader), LiveReload, disabled template caching; excluded from prod builds |