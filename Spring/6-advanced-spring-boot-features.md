# Advanced Spring Boot Features — Interview-Ready Notes

---

## 1. Scheduling Tasks

**What is it?** Spring Boot lets you run a method automatically at fixed intervals — without needing an external cron job or OS-level scheduler. Perfect for background jobs like clearing stale sessions, sending daily digest emails, generating reports, or syncing caches.

**Two things you need:**
1. `@EnableScheduling` on your main/config class — turns on the scheduling infrastructure.
2. `@Scheduled` on the method you want to run periodically — the method must be `void` and take **no parameters**.

```java
@SpringBootApplication
@EnableScheduling
public class ReportingApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReportingApplication.class, args);
    }
}
```

```java
@Component
public class ReportScheduler {

    // Runs at the 0th second of every minute, between 9 AM and 9:59 AM
    @Scheduled(cron = "0 * 9 * * ?")
    public void generateHourlyReport() {
        System.out.println("Generating report at: " + LocalDateTime.now());
    }
}
```

### The Three Ways to Schedule

| Attribute | Meaning | Waits for previous run? |
|---|---|---|
| `cron` | Cron-style expression (second, minute, hour, day-of-month, month, day-of-week) | Follows the schedule regardless |
| `fixedRate` | Runs every N milliseconds, measured from the **start** of the previous invocation | ❌ No — can overlap if a task runs long |
| `fixedDelay` | Runs N milliseconds after the **previous invocation finishes** | ✅ Yes |

```java
// fixedRate: fires every 2 seconds, does NOT wait for the previous run to finish
@Scheduled(fixedRate = 2000)
public void pollQueue() {
    System.out.println("Polling at " + LocalDateTime.now());
}

// fixedDelay + initialDelay: waits 5s before the first run,
// then waits 3s after each run completes before starting the next one
@Scheduled(fixedDelay = 3000, initialDelay = 5000)
public void cleanTempFiles() {
    System.out.println("Cleaning temp files at " + LocalDateTime.now());
}
```

> 💡 **Interview tip:** `fixedRate` is good for tasks that should run at a steady cadence regardless of duration (e.g., health check pings). `fixedDelay` is safer for tasks where overlapping runs could cause race conditions (e.g., a batch job that reads and processes files).

**Common gotcha:** By default, `@Scheduled` methods all run on a **single thread** (one shared scheduler thread) — if one task takes too long, it delays all others. Fix this by configuring a `TaskScheduler` bean with a thread pool:

```java
@Bean
public TaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
    scheduler.setPoolSize(5);   // allows up to 5 scheduled tasks to run concurrently
    scheduler.setThreadNamePrefix("scheduled-task-");
    return scheduler;
}
```

---

## 2. Sending Emails (SMTP)

Spring Boot integrates the JavaMail API through the `spring-boot-starter-mail` dependency, letting you send emails via any SMTP server (Gmail, Outlook, SendGrid SMTP relay, etc.).

### Setup

**Dependency:** `spring-boot-starter-mail`

**`application.properties` (using Gmail SMTP as an example):**
```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your_email@gmail.com
spring.mail.password=your_app_password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

> ⚠️ **Important:** Gmail no longer accepts your regular account password for SMTP. You must enable **2-Step Verification** on the Google account, then generate a 16-character **App Password** and use that instead.

### Core Building Blocks

Spring exposes the `JavaMailSender` interface — Spring Boot auto-configures a bean for it once the mail starter + properties are present.

- **`SimpleMailMessage`** → for plain text emails (no attachments, no HTML).
- **`MimeMessage` + `MimeMessageHelper`** → for rich emails with attachments, inline images, or HTML bodies.

```java
public record EmailRequest(String recipient, String subject, String body, String attachmentPath) {}
```

```java
public interface NotificationMailService {
    String sendPlainMail(EmailRequest request);
    String sendMailWithAttachment(EmailRequest request);
}
```

```java
@Service
public class NotificationMailServiceImpl implements NotificationMailService {

    @Autowired
    private JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String fromAddress;

    @Override
    public String sendPlainMail(EmailRequest request) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(fromAddress);
            message.setTo(request.recipient());
            message.setSubject(request.subject());
            message.setText(request.body());
            mailSender.send(message);
            return "Email sent successfully";
        } catch (Exception ex) {
            return "Failed to send email: " + ex.getMessage();
        }
    }

    @Override
    public String sendMailWithAttachment(EmailRequest request) {
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        try {
            MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
            helper.setFrom(fromAddress);
            helper.setTo(request.recipient());
            helper.setSubject(request.subject());
            helper.setText(request.body());

            FileSystemResource file = new FileSystemResource(new File(request.attachmentPath()));
            helper.addAttachment(file.getFilename(), file);

            mailSender.send(mimeMessage);
            return "Email with attachment sent successfully";
        } catch (MessagingException ex) {
            return "Failed to send email: " + ex.getMessage();
        }
    }
}
```

```java
@RestController
@RequestMapping("/api/mail")
public class MailController {

    @Autowired
    private NotificationMailService mailService;

    @PostMapping("/simple")
    public String sendSimple(@RequestBody EmailRequest request) {
        return mailService.sendPlainMail(request);
    }

    @PostMapping("/attachment")
    public String sendWithAttachment(@RequestBody EmailRequest request) {
        return mailService.sendMailWithAttachment(request);
    }
}
```

> 💡 **Interview tip:** `SimpleMailMessage` is easy but limited to plain text. `MimeMessageHelper` is the go-to for anything real-world — HTML templates (e.g., Thymeleaf-rendered emails), inline images (`helper.addInline(...)`), and multiple attachments.

---

## 3. File Handling & Uploading Files

Spring Boot handles file uploads via the `MultipartFile` abstraction, which represents a file received in a `multipart/form-data` HTTP request.

### Configuration

```properties
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

- `max-file-size` → cap on a single file's size.
- `max-request-size` → cap on the entire multipart request (useful when uploading multiple files at once).

### Upload, List, and Download Endpoints

```java
@RestController
@RequestMapping("/files")
public class FileStorageController {

    private final String uploadDir = System.getProperty("user.dir") + "/uploads";

    // 1. Upload a file
    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
        try {
            Path targetPath = Paths.get(uploadDir, file.getOriginalFilename());
            Files.createDirectories(targetPath.getParent());
            Files.write(targetPath, file.getBytes());
            return ResponseEntity.ok("File uploaded successfully: " + file.getOriginalFilename());
        } catch (IOException e) {
            return ResponseEntity.internalServerError().body("Upload failed: " + e.getMessage());
        }
    }

    // 2. List uploaded files
    @GetMapping("/list")
    public String[] listFiles() {
        File directory = new File(uploadDir);
        return directory.list();
    }

    // 3. Download a file
    @GetMapping("/download/{filename:.+}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String filename) throws IOException {
        Path filePath = Paths.get(uploadDir, filename);

        if (!Files.exists(filePath)) {
            return ResponseEntity.notFound().build();
        }

        Resource resource = new InputStreamResource(new FileInputStream(filePath.toFile()));

        return ResponseEntity.ok()
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
                .body(resource);
    }
}
```

**Key points:**
- `{filename:.+}` in the path is a regex trick — without it, Spring strips everything after the **last dot** (treating it as a file extension for content negotiation), which would cut off the real file extension.
- Always validate uploaded file names/paths in production to prevent **path traversal attacks** (e.g., someone uploading a file named `../../etc/passwd`).
- For large files or production systems, prefer external/object storage (AWS S3, Azure Blob) over storing on the local disk of a stateless container.

> 💡 **Interview tip:** `MultipartFile.getBytes()` loads the entire file into memory — fine for small files, but for large uploads use `transferTo()` or work with the file's `InputStream` directly to avoid `OutOfMemoryError`.

---

## 4. Caching

**What is caching, and why?** A cache is a temporary, fast-access storage layer sitting between your application and a slower resource (a database, an external API). It stores frequently/recently used results so repeated requests don't have to redo the expensive work.

### Types of Caching (conceptual, often asked)

| Type | Where it lives | Example |
|---|---|---|
| **CDN Caching** | Edge servers close to the user | Caching static assets (images, JS/CSS) |
| **Database Caching** | Layer in front of the DB | Query result caching |
| **In-Memory Caching** | Application/server RAM | Redis, Caffeine |
| **Web Server Caching** | Web/proxy server | Caching full rendered HTML pages |

### Enabling Caching in Spring Boot

**Dependency:** `spring-boot-starter-cache`

```java
@SpringBootApplication
@EnableCaching
public class InventoryApplication {
    public static void main(String[] args) {
        SpringApplication.run(InventoryApplication.class, args);
    }
}
```

### The Core Caching Annotations

```java
@Service
public class ProductService {

    // 1. @Cacheable — cache the return value; skip the method body on cache hit
    @Cacheable(value = "products", key = "#id")
    public Product getProductById(Long id) {
        System.out.println("Hitting the database for product " + id);
        return productRepository.findById(id).orElseThrow();
    }

    // 2. @CachePut — ALWAYS runs the method, then updates the cache with the result
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // 3. @CacheEvict — removes an entry (or all entries) from the cache
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // Evict everything, e.g. after a bulk import
    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() {
        System.out.println("Product cache cleared");
    }
}
```

| Annotation | Runs the method body? | Purpose |
|---|---|---|
| `@Cacheable` | Only on cache miss | Read-through caching |
| `@CachePut` | Always | Update cache + DB together (e.g. on an update operation) |
| `@CacheEvict` | Always | Remove stale entries (e.g. on delete) |
| `@Caching` | — | Combine multiple cache annotations of the **same type** on one method (Spring doesn't allow repeating an annotation type directly) |
| `@CacheConfig` | — | Class-level shortcut to set a common cache name so you don't repeat `value="..."` on every method |

```java
// @Caching example — needed because you can't repeat @CacheEvict directly on one method
@Caching(evict = {
    @CacheEvict(value = "products", key = "#id"),
    @CacheEvict(value = "productSummary", allEntries = true)
})
public void removeProductEverywhere(Long id) {
    productRepository.deleteById(id);
}
```

### Conditional Caching — `condition` and `unless`

```java
// condition: decide BEFORE the method runs, based on the input arguments
@Cacheable(value = "products", condition = "#id > 0")
public Product getProduct(Long id) { ... }

// unless: decide AFTER the method runs, based on the returned result
@Cacheable(value = "products", unless = "#result == null")
public Product findProduct(Long id) { ... }
```

> 💡 **Interview one-liner:** `condition` filters on the **input**; `unless` filters on the **output**. Both accept SpEL (Spring Expression Language).

---

## 5. Caching with Other Providers

By default, Spring Boot's `@EnableCaching` uses a **simple in-memory `ConcurrentHashMap`-based cache** — fine for demos, but it has no eviction policy, no TTL (time-to-live), and doesn't survive a restart or scale across multiple instances. For production, you plug in a real **cache provider**.

Spring's caching abstraction is built on two interfaces:
- **`Cache`** — represents one actual cache (a named bucket of key-value pairs).
- **`CacheManager`** — creates/manages `Cache` instances; Spring Boot auto-detects which `CacheManager` implementation to configure based on what's on the classpath.

### Common Providers

| Provider | Best for | Notes |
|---|---|---|
| **Redis** | Distributed caching across multiple app instances/microservices | Supports TTL, persistence, pub-sub |
| **Caffeine** | High-performance local (in-JVM) caching | Modern replacement for Guava's cache |
| **EhCache** | Local caching with XML-based fine-grained config | Long-standing, widely used in enterprise Java |
| **Hazelcast** | Distributed, in-memory data grid | Good for clustering + real-time computation |
| **Infinispan** | Large-scale distributed caching | Used heavily in the JBoss/WildFly ecosystem |
| **JCache (JSR-107)** | Standard Java caching spec | Portable across compliant providers |
| **Guava** | Legacy local caching | Superseded by Caffeine in most new projects |

### Example: Redis as the Cache Provider

**Dependency:** `spring-boot-starter-data-redis`

```properties
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
spring.cache.redis.time-to-live=60000
spring.cache.cache-names=products,productSummary
```

With this configuration, the exact same `@Cacheable`/`@CachePut`/`@CacheEvict` code shown earlier now stores its data in Redis instead of local memory — **no code changes needed**, only configuration. This is the whole point of Spring's cache abstraction: swap the provider without touching business logic.

### Example: Caffeine as the Cache Provider

**Dependency:** `com.github.ben-manes.caffeine:caffeine`

```properties
spring.cache.type=caffeine
spring.cache.cache-names=products
spring.cache.caffeine.spec=maximumSize=500,expireAfterAccess=600s
```

> 💡 **Interview tip:** A classic question is *"Local cache (Caffeine/EhCache) vs Distributed cache (Redis) — when do you use which?"*
> - **Local cache** → fastest (no network hop), but each app instance has its own copy → can go stale/inconsistent across instances.
> - **Distributed cache** → one shared source of truth across all instances/microservices, survives individual app restarts, but adds network latency and an extra infrastructure dependency.

---

## 6. Transaction Management (`@Transactional`)

### Why transactions matter — ACID

| Property | Meaning |
|---|---|
| **Atomicity** | All operations in a transaction succeed, or none do |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere with each other |
| **Durability** | Once committed, changes survive crashes/restarts |

**Classic example:** A money transfer needs to (1) debit account A, (2) credit account B. If the app crashes between steps 1 and 2, you've lost money — unless both steps are wrapped in a single transaction that rolls back entirely on failure.

### `@Transactional` in Action

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryService inventoryService;

    @Transactional(rollbackFor = Exception.class)
    public Order placeOrder(Order order) throws Exception {
        Order savedOrder = orderRepository.save(order);

        // If this throws, the order save above is ALSO rolled back
        inventoryService.reduceStock(order.getProductId(), order.getQuantity());

        if (order.getQuantity() > 1000) {
            throw new RuntimeException("Suspiciously large order — flagged for review");
        }

        return savedOrder;
    }
}
```

**What `@Transactional` does under the hood:**
- Starts a transaction before the method executes.
- Commits automatically if the method completes normally.
- Rolls back automatically if a **runtime (unchecked) exception** is thrown.
- By default, it does **NOT** roll back on checked exceptions — use `rollbackFor = Exception.class` to override this and roll back on any exception type.

### Without `@Transactional` — the problem it solves

If you skip `@Transactional`, a failure partway through a multi-step operation leaves the database in an inconsistent state — e.g., the `Order` row gets saved, but the inventory update fails with a `NullPointerException`, leaving stock counts wrong forever. Wrapping both operations in `@Transactional` ensures **all-or-nothing**: if the inventory update fails, the order save is undone too.

### Enabling It

If you're using `spring-boot-starter-data-jpa`, transaction management is **auto-configured** — you don't need to do anything extra. `@EnableTransactionManagement` is only required if you're managing transactions manually (no JPA starter) or need custom transaction manager behavior.

```java
@SpringBootApplication
@EnableTransactionManagement   // optional when spring-boot-starter-data-jpa is present
public class OrderManagementApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderManagementApplication.class, args);
    }
}
```

### Important `@Transactional` Attributes

| Attribute | Purpose |
|---|---|
| `rollbackFor` | Specify which exception types trigger a rollback (default: only unchecked exceptions) |
| `noRollbackFor` | Exceptions that should NOT trigger a rollback |
| `propagation` | How this transaction relates to an existing one (see below) |
| `isolation` | Isolation level (READ_COMMITTED, SERIALIZABLE, etc.) |
| `readOnly` | Marks the transaction as read-only — allows the DB/ORM to optimize |
| `timeout` | Max time (in seconds) before the transaction is forced to roll back |

### Propagation — Frequently Asked

Propagation defines what happens when a `@Transactional` method calls another `@Transactional` method.

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Joins the existing transaction, or creates a new one if none exists |
| `REQUIRES_NEW` | Always creates a new, independent transaction — suspends the caller's transaction |
| `NESTED` | Creates a savepoint within the existing transaction; can roll back to that point without aborting the whole transaction |
| `MANDATORY` | Must run inside an existing transaction, or throws an exception |
| `NOT_SUPPORTED` | Runs without a transaction, suspending any existing one |

> 💡 **Interview tip:** A very common gotcha — calling a `@Transactional` method **from within the same class** (self-invocation) **bypasses the proxy** Spring uses to manage transactions, so the annotation is silently ignored. `@Transactional` only works through Spring-managed proxies, which means the call must come from **outside the class** (e.g., another `@Service` calling it), or you need AspectJ weaving instead of proxy-based AOP.

---

## 7. DTO Mapping (Entity ↔ DTO with ModelMapper)

### Why DTOs?

A **DTO (Data Transfer Object)** is a plain class used to move data between layers (e.g., service → controller → client) **without exposing your JPA entity directly**.

**Reasons to use DTOs instead of returning entities directly:**
- **Security:** Hide sensitive fields (passwords, internal IDs, audit columns) from API responses.
- **Decoupling:** Your API contract doesn't break every time you add/rename a database column.
- **Flexibility:** Combine/reshape data from multiple entities into one response object.
- **Avoids lazy-loading issues:** Prevents accidental serialization of lazy JPA associations, which can trigger extra queries or `LazyInitializationException`.

```java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    private String password;   // should NEVER leave the server as-is
    // getters/setters omitted for brevity
}
```

```java
// DTO exposes only what the client actually needs
public class UserResponseDto {
    private String name;
    private String email;
    // getters/setters omitted for brevity
}
```

### Manual Mapping (no library)

```java
UserResponseDto dto = new UserResponseDto();
dto.setName(user.getName());
dto.setEmail(user.getEmail());
```

This works, but becomes tedious and error-prone as objects grow — every new field means touching every mapping call site.

### Automatic Mapping with ModelMapper

**Dependency:**
```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.1.1</version>
</dependency>
```

**Register it as a bean:**
```java
@Configuration
public class MapperConfig {
    @Bean
    public ModelMapper modelMapper() {
        return new ModelMapper();
    }
}
```

**Use it in the service layer:**
```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private ModelMapper modelMapper;

    public UserResponseDto getUser(Long userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        // ModelMapper matches fields by name/type automatically
        return modelMapper.map(user, UserResponseDto.class);
    }

    public User createUser(User user) {
        return userRepository.save(user);   // full entity saved, password included
    }
}
```

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        return new ResponseEntity<>(userService.createUser(user), HttpStatus.CREATED);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUser(id));   // password NOT returned
    }
}
```

**How ModelMapper decides what maps to what:** By default it matches fields by **name and type** between the source and destination classes. For mismatched names (e.g., `firstName` in the entity vs `fname` in the DTO), you configure an explicit `PropertyMap` or `TypeMap`.

```java
// Custom mapping example when field names differ
modelMapper.typeMap(User.class, UserResponseDto.class).addMappings(mapper ->
    mapper.map(User::getEmail, UserResponseDto::setContactEmail)
);
```

> 💡 **Interview tip:** ModelMapper (and similar libraries like MapStruct) exist to eliminate repetitive `dto.setX(entity.getX())` boilerplate. **MapStruct** is often preferred over ModelMapper in high-performance systems because it generates mapping code at **compile time** (via annotation processing) rather than using **reflection at runtime**, making it noticeably faster — a nuance worth mentioning if asked to compare the two.

---

## 8. Rapid-Fire Interview Q&A

**Q1: Difference between `fixedRate` and `fixedDelay`?**
`fixedRate` starts counting from when the previous execution **started** (can overlap). `fixedDelay` starts counting only after the previous execution **finished** (never overlaps).

**Q2: How would you send an HTML email with an embedded logo image?**
Use `MimeMessageHelper` with `helper.setText(htmlContent, true)` to mark the body as HTML, and `helper.addInline("logo", imageResource)` to embed an image referenced via `cid:logo` in the HTML.

**Q3: What's wrong with storing uploaded files on local disk in a cloud-native app?**
Containers/pods are typically ephemeral and can be scaled horizontally — a file saved to one instance's disk won't be visible from another instance, and it's lost on restart. Use object storage (S3, GCS, Azure Blob) or a shared network volume instead.

**Q4: Difference between `@Cacheable` and `@CachePut`?**
`@Cacheable` skips the method entirely on a cache hit (read path). `@CachePut` always executes the method and then refreshes the cache with the new result (write/update path) — used so a cached value doesn't go stale after an update.

**Q5: Why doesn't `@Transactional` work when called from within the same class?**
Spring's proxy-based AOP wraps the **bean**, not the raw class instance. A call like `this.someTransactionalMethod()` from inside the same class bypasses the proxy entirely, so no transaction is started. Fix: move the method to another Spring bean, or use `AopContext.currentProxy()` / switch to AspectJ compile-time weaving.

**Q6: Does Spring roll back a transaction for checked exceptions by default?**
No — by default, Spring only rolls back on unchecked (`RuntimeException`) exceptions. To roll back on checked exceptions too, explicitly set `@Transactional(rollbackFor = Exception.class)`.

**Q7: Why use a DTO instead of returning the JPA entity directly from a REST controller?**
To avoid leaking sensitive fields, prevent tight coupling between your DB schema and API contract, and avoid serialization issues with lazy-loaded relationships.

**Q8: Local cache vs distributed cache — how do you choose?**
Local (Caffeine/EhCache) is faster since there's no network round trip, but each instance keeps its own copy, risking staleness in a multi-instance deployment. Distributed (Redis/Hazelcast) keeps one consistent shared cache across all instances at the cost of network latency and an extra moving part in your infrastructure.

**Q9: What does `@EnableCaching` actually do, and is it enough on its own?**
It turns on Spring's caching annotation processing (`@Cacheable`, `@CachePut`, `@CacheEvict`). On its own it defaults to an in-memory, non-expiring `ConcurrentHashMap`-backed cache — for production use you typically also add a real provider (Redis, Caffeine, etc.) and its configuration.

**Q10: What is transaction propagation, and give a real use case for `REQUIRES_NEW`?**
Propagation controls how a `@Transactional` method behaves when called from within another transaction. `REQUIRES_NEW` is useful for things like audit logging: you want the audit record to persist **even if the main business transaction later rolls back**, so you run the audit-write in its own independent transaction.

---

## 9. Quick Revision Cheat-Sheet

- **Scheduling** → `@EnableScheduling` + `@Scheduled(cron/fixedRate/fixedDelay)`
- **Email** → `spring-boot-starter-mail` + `JavaMailSender` → `SimpleMailMessage` (text) / `MimeMessageHelper` (attachments/HTML)
- **File handling** → `MultipartFile` for uploads, `Resource`/`InputStreamResource` for downloads; watch multipart size limits & path traversal
- **Caching** → `@EnableCaching` + `@Cacheable` (read), `@CachePut` (update), `@CacheEvict` (remove)
- **Cache providers** → swap `CacheManager` implementation (Redis, Caffeine, EhCache...) via config — code stays the same
- **Transactions** → `@Transactional` for atomic multi-step DB operations; know `rollbackFor` and `propagation`; beware self-invocation bypassing the proxy
- **DTO Mapping** → never expose entities directly; ModelMapper (runtime/reflection) or MapStruct (compile-time) to avoid manual boilerplate

---

*End of notes. For each feature, be ready to explain not just "how" (the annotation/config) but "why" (the real-world problem it solves) — that's what separates a memorized answer from a confident one in interviews.*