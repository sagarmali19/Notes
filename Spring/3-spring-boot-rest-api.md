# Spring Boot with REST API — Interview Ready Notes

> Self-contained notes. Every concept has a definition, why it matters, a code example, and common interview Q&A / gotchas.

---

## Table of Contents
1. [Introduction to RESTful Web Services](#1-introduction-to-restful-web-services)
2. [@RestController](#2-restcontroller)
3. [@RequestMapping](#3-requestmapping)
4. [@GetMapping and @PostMapping](#4-getmapping-and-postmapping)
5. [@PutMapping and @DeleteMapping](#5-putmapping-and-deletemapping)
6. [@PathVariable and @RequestParam](#6-pathvariable-and-requestparam)
7. [@RequestBody](#7-requestbody)
8. [Building a Complete REST API (CRUD)](#8-building-a-complete-rest-api-crud)
9. [JSON Serialization/Deserialization (Jackson)](#9-json-serializationdeserialization-jackson)
10. [Exception Handling](#10-exception-handling)
11. [Validation](#11-validation)
12. [Quick-Fire Interview Q&A Cheat Sheet](#12-quick-fire-interview-qa-cheat-sheet)

---

## 1. Introduction to RESTful Web Services

**REST (Representational State Transfer)** is an architectural style (not a protocol) for building web services that communicate over HTTP using stateless, resource-oriented operations.

### Core REST Principles
| Principle | Meaning |
|---|---|
| **Statelessness** | Each request contains all info needed; server stores no client session state. |
| **Client-Server** | UI/client and data storage/server are separated and can evolve independently. |
| **Uniform Interface** | Resources identified by URIs; manipulated via standard HTTP verbs. |
| **Resource-Based** | Everything is a "resource" (e.g., `/users/5`), identified by a URI. |
| **Representation** | Resources are exchanged as representations — usually JSON or XML. |
| **Cacheable** | Responses must define themselves as cacheable or not. |
| **Layered System** | Client can't tell if it's connected directly to the server or through an intermediary (proxy/load balancer). |

### HTTP Methods Mapped to CRUD
| HTTP Method | CRUD Operation | Idempotent? | Safe? |
|---|---|---|---|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Update (full replace) | Yes | No |
| PATCH | Update (partial) | No (typically) | No |
| DELETE | Delete | Yes | No |

> **Idempotent** = calling it multiple times has the same effect as calling it once. **Safe** = doesn't modify server state.

### Why Spring Boot for REST?
- Auto-configuration (no XML, minimal boilerplate)
- Embedded servers (Tomcat/Jetty/Undertow) — run as a standalone JAR, no need to deploy a WAR
- Built-in JSON conversion via Jackson
- `spring-boot-starter-web` gives you everything needed (Spring MVC + Jackson + embedded Tomcat) in one dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### Minimal Spring Boot REST App
```java
@SpringBootApplication
public class RestDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(RestDemoApplication.class, args);
    }
}

@RestController
class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello, REST!";
    }
}
```

**Interview Q:** *Is REST a protocol?*
**A:** No — it's an architectural style built on top of HTTP (a protocol). REST doesn't mandate implementation details, just constraints.

---

## 2. @RestController

`@RestController` = `@Controller` + `@ResponseBody`.

- **`@Controller`** — marks a class as a Spring MVC controller (returns view names by default, e.g., for Thymeleaf/JSP).
- **`@ResponseBody`** — tells Spring to serialize the return value directly into the HTTP response body (JSON/XML) instead of resolving it as a view name.
- **`@RestController`** combines both — every method's return value is written directly to the response body. This is what you use for REST APIs.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public List<Product> getAllProducts() {
        return productService.findAll(); // serialized to JSON automatically
    }
}
```

### @Controller vs @RestController (classic interview question)
```java
@Controller
public class WebController {
    @GetMapping("/home")
    public String home() {
        return "home"; // resolves to home.html/home.jsp (a VIEW)
    }

    @GetMapping("/data")
    @ResponseBody              // needed here to return raw data, not a view
    public String data() {
        return "raw data";
    }
}

@RestController
public class ApiController {
    @GetMapping("/data")
    public String data() {
        return "raw data"; // @ResponseBody is implicit — always returns raw data/JSON
    }
}
```

| | `@Controller` | `@RestController` |
|---|---|---|
| Purpose | MVC web apps (returns views) | REST APIs (returns data) |
| Needs `@ResponseBody`? | Yes, per method | No, implicit on all methods |
| Typical return type | View name (String) | Object/JSON/ResponseEntity |

**Interview Q:** *How does Spring know to convert a Java object to JSON?*
**A:** Via `HttpMessageConverter` (specifically `MappingJackson2HttpMessageConverter`), auto-configured when Jackson is on the classpath (included in `spring-boot-starter-web`).

---

## 3. @RequestMapping

The general-purpose annotation to map HTTP requests to handler methods/classes. Can be applied at **class level** (base path) and **method level** (specific path), and supports specifying HTTP method, headers, params, and content types.

```java
@RestController
@RequestMapping(value = "/api/users", produces = "application/json")
public class UserController {

    @RequestMapping(method = RequestMethod.GET)
    public List<User> getUsers() {
        return userService.getAll();
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }

    @RequestMapping(method = RequestMethod.POST)
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

### Key attributes
| Attribute | Purpose |
|---|---|
| `value` / `path` | URI path to map |
| `method` | HTTP method (GET/POST/PUT/DELETE/PATCH) |
| `params` | Requires specific request params to be present |
| `headers` | Requires specific headers |
| `consumes` | Content-Type the method accepts (e.g., `application/json`) |
| `produces` | Content-Type the method returns |

```java
@RequestMapping(value = "/search", params = "type=admin")
public List<User> searchAdmins() { ... }
```

**Why the shortcut annotations exist:** `@RequestMapping(method = RequestMethod.GET)` is verbose, so Spring 4.3+ introduced `@GetMapping`, `@PostMapping`, etc. as composed annotations — same thing, less code.

**Interview Q:** *What happens if two methods map to the same URL + HTTP method?*
**A:** Spring throws an `IllegalStateException: Ambiguous mapping` at startup.

---

## 4. @GetMapping and @PostMapping

Shortcut (composed) annotations equivalent to `@RequestMapping(method = RequestMethod.GET/POST)`.

- **`@GetMapping`** — fetch/read data. Should be **safe** and **idempotent**. No request body typically.
- **`@PostMapping`** — create a new resource / submit data. Not idempotent (calling twice creates two resources).

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    // GET /api/books
    @GetMapping
    public List<Book> getAllBooks() {
        return bookService.findAll();
    }

    // GET /api/books/5
    @GetMapping("/{id}")
    public Book getBookById(@PathVariable Long id) {
        return bookService.findById(id);
    }

    // POST /api/books
    @PostMapping
    public ResponseEntity<Book> createBook(@RequestBody Book book) {
        Book saved = bookService.save(book);
        return new ResponseEntity<>(saved, HttpStatus.CREATED); // 201
    }
}
```

**Best practice:** POST that creates a resource should return **`201 Created`** with a `Location` header pointing to the new resource, not `200 OK`.

```java
@PostMapping
public ResponseEntity<Book> createBook(@RequestBody Book book) {
    Book saved = bookService.save(book);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(saved.getId())
            .toUri();
    return ResponseEntity.created(location).body(saved);
}
```

**Interview Q:** *Why shouldn't GET have a request body?*
**A:** HTTP spec doesn't forbid it, but it's semantically wrong and many servers/proxies/caches ignore or strip GET bodies. Use query params or path variables instead.

---

## 5. @PutMapping and @DeleteMapping

- **`@PutMapping`** — full update/replace of a resource. **Idempotent** — calling it 5 times with the same body leaves the resource in the same final state.
- **`@DeleteMapping`** — removes a resource. Also idempotent (deleting an already-deleted resource has the same net effect, though the response code may differ — e.g., 404 on the second call).

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    // PUT /api/books/5  — full replace
    @PutMapping("/{id}")
    public ResponseEntity<Book> updateBook(@PathVariable Long id, @RequestBody Book bookDetails) {
        Book book = bookService.findById(id);
        if (book == null) {
            return ResponseEntity.notFound().build(); // 404
        }
        book.setTitle(bookDetails.getTitle());
        book.setAuthor(bookDetails.getAuthor());
        book.setPrice(bookDetails.getPrice());
        Book updated = bookService.save(book);
        return ResponseEntity.ok(updated); // 200
    }

    // DELETE /api/books/5
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteBook(@PathVariable Long id) {
        if (!bookService.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        bookService.deleteById(id);
        return ResponseEntity.noContent().build(); // 204
    }
}
```

### PUT vs PATCH (common follow-up)
| | PUT | PATCH |
|---|---|---|
| Replaces | Entire resource | Only specified fields |
| Idempotent | Yes | Not guaranteed |
| Missing fields | Get nulled/reset | Left untouched |

```java
@PatchMapping("/{id}")
public ResponseEntity<Book> partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
    Book book = bookService.findById(id);
    updates.forEach((key, value) -> {
        switch (key) {
            case "title" -> book.setTitle((String) value);
            case "price" -> book.setPrice((Double) value);
        }
    });
    return ResponseEntity.ok(bookService.save(book));
}
```

**Interview Q:** *What status code should DELETE return on success?*
**A:** `204 No Content` (no body) is most common; `200 OK` if you return the deleted resource.

---

## 6. @PathVariable and @RequestParam

Both extract data from the incoming HTTP request, but from different places.

| | `@PathVariable` | `@RequestParam` |
|---|---|---|
| Extracts from | URI path segment | Query string (`?key=value`) |
| Example URL | `/users/101` | `/users?id=101` |
| Typical use | Identifying a specific resource | Filtering, sorting, pagination, optional params |
| Required by default | Yes | Yes (can set `required = false`) |

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET /api/users/101
    @GetMapping("/{id}")
    public User getUser(@PathVariable("id") Long userId) {
        return userService.findById(userId);
    }

    // GET /api/users/101/orders/55
    @GetMapping("/{userId}/orders/{orderId}")
    public Order getOrder(@PathVariable Long userId, @PathVariable Long orderId) {
        return orderService.find(userId, orderId);
    }

    // GET /api/users?page=0&size=10&sort=name
    @GetMapping
    public List<User> searchUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String sort) {
        return userService.search(page, size, sort);
    }

    // GET /api/users/search?name=John  (optional param)
    @GetMapping("/search")
    public List<User> searchByName(@RequestParam(required = false) String name) {
        if (name == null) return userService.getAll();
        return userService.findByName(name);
    }
}
```

**Note:** If the variable name matches the parameter name exactly, you can omit the value: `@PathVariable Long id` works for `/{id}` without `@PathVariable("id")` — but this requires compiling with `-parameters` flag; otherwise, always specify the name explicitly to be safe.

**Interview Q:** *What happens if a required `@RequestParam` is missing?*
**A:** Spring throws `MissingServletRequestParameterException` → results in `400 Bad Request` by default.

---

## 7. @RequestBody

Binds the **HTTP request body** (typically JSON) to a Java object. Spring uses `HttpMessageConverter`s (Jackson) to deserialize the body into the target type.

```java
public class User {
    private String name;
    private String email;
    // getters, setters, no-arg constructor required for Jackson
}

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public User createUser(@RequestBody User user) {
        System.out.println(user.getName()); // Jackson auto-mapped JSON -> object
        return userService.save(user);
    }
}
```

**Sample request:**
```http
POST /api/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

### Common gotchas
1. **No-arg constructor required** on the target class (or use `@JsonCreator` for constructor binding).
2. **`Content-Type: application/json`** header must be set on the request, or Spring throws `HttpMediaTypeNotSupportedException` (415).
3. Malformed JSON → `HttpMessageNotReadableException` (400).
4. Combine with `@Valid` to trigger bean validation (see [Validation](#11-validation)).

```java
@PostMapping
public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(user));
}
```

**Interview Q:** *Can you use `@RequestBody` with GET?*
**A:** Technically possible but discouraged — GET is meant to be safe/cacheable, and many HTTP clients/servers don't support bodies on GET reliably.

---

## 8. Building a Complete REST API (CRUD)

Putting it all together — a full layered REST API: **Controller → Service → Repository → Entity**.

```java
// Entity
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Invalid email format")
    private String email;

    private Double salary;

    // constructors, getters, setters
}

// Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByNameContainingIgnoreCase(String name);
}

// Service
@Service
public class EmployeeService {
    private final EmployeeRepository repository;

    public EmployeeService(EmployeeRepository repository) {
        this.repository = repository; // constructor injection (preferred over @Autowired field injection)
    }

    public List<Employee> getAll() { return repository.findAll(); }

    public Employee getById(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Employee not found: " + id));
    }

    public Employee create(Employee employee) { return repository.save(employee); }

    public Employee update(Long id, Employee details) {
        Employee employee = getById(id);
        employee.setName(details.getName());
        employee.setEmail(details.getEmail());
        employee.setSalary(details.getSalary());
        return repository.save(employee);
    }

    public void delete(Long id) {
        Employee employee = getById(id);
        repository.delete(employee);
    }
}

// Controller
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {
    private final EmployeeService service;

    public EmployeeController(EmployeeService service) { this.service = service; }

    @GetMapping
    public List<Employee> getAll() { return service.getAll(); }

    @GetMapping("/{id}")
    public ResponseEntity<Employee> getById(@PathVariable Long id) {
        return ResponseEntity.ok(service.getById(id));
    }

    @PostMapping
    public ResponseEntity<Employee> create(@Valid @RequestBody Employee employee) {
        Employee saved = service.create(employee);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Employee> update(@PathVariable Long id, @Valid @RequestBody Employee employee) {
        return ResponseEntity.ok(service.update(id, employee));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Why layering matters (interview favorite):**
- **Controller** — HTTP concerns only (request/response mapping).
- **Service** — business logic, transactions (`@Transactional`).
- **Repository** — data access (Spring Data JPA abstracts SQL/JPQL).
- Keeps concerns separated → testable, maintainable, swappable persistence layer.

**Interview Q:** *Why constructor injection over field injection (`@Autowired` on field)?*
**A:** Enables immutability (`final` fields), makes dependencies explicit/required, easier to unit test (no need for reflection-based mocking frameworks), and fails fast at startup if a bean is missing.

---

## 9. JSON Serialization/Deserialization (Jackson)

Spring Boot uses **Jackson** (`ObjectMapper`) by default to convert Java objects ↔ JSON.

- **Serialization**: Java object → JSON (on response)
- **Deserialization**: JSON → Java object (on `@RequestBody`)

### Common Jackson Annotations
```java
public class Employee {

    private Long id;

    @JsonProperty("full_name")           // renames field in JSON
    private String name;

    @JsonIgnore                          // excluded from JSON entirely
    private String password;

    @JsonFormat(pattern = "yyyy-MM-dd")  // date formatting
    private LocalDate joiningDate;

    @JsonInclude(JsonInclude.Include.NON_NULL) // omit if null
    private String middleName;

    // getters/setters
}
```

**Resulting JSON:**
```json
{
  "id": 1,
  "full_name": "Alice",
  "joiningDate": "2024-01-15"
}
```
(note: `password` is missing, `name` became `full_name`)

### Handling Circular References (Parent-Child)
```java
public class Department {
    private Long id;
    @JsonManagedReference
    private List<Employee> employees;
}

public class Employee {
    private Long id;
    @JsonBackReference
    private Department department; // won't cause infinite recursion
}
```

### Customizing the Global ObjectMapper
```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());        // for LocalDate/LocalDateTime
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
        return mapper;
    }
}
```

### Manual usage (rarely needed — Spring does it automatically)
```java
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(employee);      // serialize
Employee emp = mapper.readValue(json, Employee.class);  // deserialize
```

**Interview Q:** *What if incoming JSON has a field that doesn't exist in your Java class?*
**A:** By default Jackson throws `UnrecognizedPropertyException`. Suppress with:
```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class Employee { ... }
```

**Interview Q:** *Jackson vs Gson vs Moshi?*
**A:** Jackson is Spring Boot's default (auto-configured, feature-rich, annotation-driven). Gson (Google) is simpler but less powerful with generics/streaming. Not usually asked in depth for Spring interviews — know Jackson is the default.

---

## 10. Exception Handling

Goal: return **consistent, meaningful error responses** (with proper HTTP status codes) instead of leaking stack traces.

### Approach 1: `@ExceptionHandler` (single controller)
```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    @GetMapping("/{id}")
    public Employee getById(@PathVariable Long id) {
        return service.getById(id); // throws ResourceNotFoundException if missing
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

### Approach 2: `@ControllerAdvice` / `@RestControllerAdvice` (global — the standard, interview-expected answer)
```java
// Custom exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}

// Standard error response shape
public class ErrorResponse {
    private int status;
    private String message;
    private LocalDateTime timestamp;
    private Map<String, String> fieldErrors; // for validation errors

    // constructors, getters, setters
}

@RestControllerAdvice   // = @ControllerAdvice + @ResponseBody, applies globally to all controllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage(), LocalDateTime.now(), null);
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class) // from @Valid failures
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(fe -> fieldErrors.put(fe.getField(), fe.getDefaultMessage()));
        ErrorResponse error = new ErrorResponse(400, "Validation failed", LocalDateTime.now(), fieldErrors);
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)  // catch-all fallback
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(500, "Internal server error", LocalDateTime.now(), null);
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

**Sample error JSON response:**
```json
{
  "status": 404,
  "message": "Employee not found: 99",
  "timestamp": "2026-09-07T10:15:30"
}
```

### Why `@RestControllerAdvice` is preferred
- Centralizes error handling across **all** controllers (DRY — don't repeat try/catch everywhere).
- Keeps controllers clean, focused only on the happy path.
- Consistent error response contract for API consumers.

**Interview Q:** *Difference between `@ControllerAdvice` and `@RestControllerAdvice`?*
**A:** `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody` — responses are serialized directly to the body (JSON), same relationship as `@RestController` vs `@Controller`.

**Interview Q:** *What's the default Spring Boot error response if you do nothing?*
**A:** The built-in `/error` whitelabel error page / a generic JSON via `BasicErrorController`, e.g.:
```json
{ "timestamp": "...", "status": 500, "error": "Internal Server Error", "path": "/api/employees/1" }
```

---

## 11. Validation

Spring Boot integrates **Bean Validation (JSR-380 / Hibernate Validator)** to validate request data declaratively.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Common Annotations
| Annotation | Rule |
|---|---|
| `@NotNull` | Value must not be null |
| `@NotBlank` | String must not be null/empty/whitespace-only |
| `@NotEmpty` | Collection/String must not be null or empty |
| `@Size(min=, max=)` | String/collection length bounds |
| `@Min` / `@Max` | Numeric bounds |
| `@Email` | Valid email format |
| `@Pattern(regexp=)` | Must match regex |
| `@Positive` / `@Negative` | Numeric sign |
| `@Past` / `@Future` | Date must be in the past/future |

```java
public class Employee {

    @NotBlank(message = "Name cannot be blank")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;

    @Email(message = "Email should be valid")
    @NotBlank(message = "Email is required")
    private String email;

    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 65, message = "Age must not exceed 65")
    private int age;

    @Positive(message = "Salary must be positive")
    private Double salary;
}
```

### Triggering validation in the controller
```java
@PostMapping
public ResponseEntity<Employee> create(@Valid @RequestBody Employee employee, BindingResult result) {
    if (result.hasErrors()) {
        // manual handling option — usually skip this and let @RestControllerAdvice handle it
        return ResponseEntity.badRequest().build();
    }
    return ResponseEntity.status(HttpStatus.CREATED).body(service.create(employee));
}
```

> **Note:** `@Valid` (from `jakarta.validation`) triggers the validation. If `BindingResult` is **not** present as the next parameter, Spring throws `MethodArgumentNotValidException` automatically — which is exactly what the `@RestControllerAdvice` example in [Section 10](#10-exception-handling) catches. This is the cleaner, more common approach in interviews.

### Validating @RequestParam / @PathVariable (needs class-level annotation)
```java
@RestController
@RequestMapping("/api/employees")
@Validated   // required for method-parameter-level validation
public class EmployeeController {

    @GetMapping("/{id}")
    public Employee getById(@PathVariable @Min(1) Long id) {
        return service.getById(id);
    }
}
```

### Custom Validator (advanced — sometimes asked)
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired
    private EmployeeRepository repository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return !repository.existsByEmail(email);
    }
}
```

**Interview Q:** *`@Valid` vs `@Validated`?*
**A:** `@Valid` (standard Java/Jakarta Bean Validation) triggers validation on a single object, works on method parameters and nested objects. `@Validated` (Spring-specific) adds support for **validation groups** and enables validation on simple parameters (like `@PathVariable`/`@RequestParam`) at the class level.

**Interview Q:** *What HTTP status should invalid input return?*
**A:** `400 Bad Request`.

---

## 12. Quick-Fire Interview Q&A Cheat Sheet

**Q: What's the difference between REST and SOAP?**
A: REST is lightweight, uses HTTP verbs + JSON/XML, stateless. SOAP is a strict protocol using XML only, has built-in error handling (WS-Security, ACID transactions), heavier and more rigid — used in enterprise/banking systems needing strict contracts.

**Q: What does `ResponseEntity` give you that a plain return type doesn't?**
A: Full control over status code, headers, and body — e.g., `ResponseEntity.status(201).header("Location", uri).body(obj)`. Returning a plain object always gives `200 OK`.

**Q: How do you version a REST API?**
A: Common strategies: URI versioning (`/api/v1/users`), request param (`/api/users?version=1`), header versioning (`Accept: application/vnd.company.v1+json`), or media type versioning. URI versioning is most common/simplest.

**Q: What is HATEOAS?**
A: Hypermedia As The Engine Of Application State — REST responses include links to related actions/resources, so clients can navigate the API dynamically (Spring HATEOAS library). Rarely fully implemented in practice, but often asked conceptually.

**Q: How does Spring Boot auto-configure things like Jackson and Tomcat?**
A: Via `@EnableAutoConfiguration` (part of `@SpringBootApplication`), which scans the classpath and conditionally registers beans (`@ConditionalOnClass`, `@ConditionalOnMissingBean`) based on what dependencies are present.

**Q: Difference between `@Component`, `@Service`, `@Repository`, `@Controller`?**
A: All are stereotypes of `@Component` (so all are auto-detected by component scanning). They differ semantically: `@Service` = business logic layer, `@Repository` = persistence layer (also enables automatic exception translation to `DataAccessException`), `@Controller`/`@RestController` = web layer.

**Q: How do you test a REST controller?**
A: `@WebMvcTest` + `MockMvc` for slice testing (controller layer only, mocks service), or `@SpringBootTest` + `TestRestTemplate`/`WebTestClient` for full integration tests.
```java
@WebMvcTest(EmployeeController.class)
class EmployeeControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean EmployeeService service;

    @Test
    void shouldReturnEmployee() throws Exception {
        Mockito.when(service.getById(1L)).thenReturn(new Employee(1L, "Alice"));
        mockMvc.perform(get("/api/employees/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

**Q: What is `@Transactional` and where does it belong?**
A: Marks a method/class as running within a database transaction (commits on success, rolls back on `RuntimeException` by default). Belongs on the **Service** layer, not the controller — controllers shouldn't know about persistence details.

**Q: What common HTTP status codes should you know?**

| Code | Meaning |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request (validation/malformed input) |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (authenticated but not allowed) |
| 404 | Not Found |
| 405 | Method Not Allowed |
| 409 | Conflict (e.g., duplicate resource) |
| 415 | Unsupported Media Type |
| 500 | Internal Server Error |

**Q: How would you secure a REST API?**
A: Spring Security with JWT/OAuth2 for stateless authentication (fits REST's statelessness), method-level security (`@PreAuthorize`), HTTPS, CORS configuration (`@CrossOrigin` or a global `CorsConfigurationSource` bean).

**Q: What's the purpose of `consumes` and `produces` in mappings?**
A: `consumes` restricts what `Content-Type` the endpoint accepts in the request; `produces` restricts what `Content-Type`/`Accept` the endpoint returns. Useful when supporting both JSON and XML, or versioned media types.

---

### Final Prep Tips
- Be ready to **live-code a small CRUD controller** from memory — it's the single most common practical exercise.
- Know the **HTTP status code table** cold.
- Be able to explain **why** you'd choose `@RestControllerAdvice` over try/catch in every method.
- Understand the **request lifecycle**: Client → DispatcherServlet → HandlerMapping → Controller → Service → Repository → DB, and back with Jackson serializing the response.