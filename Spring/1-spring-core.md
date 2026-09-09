# Spring Core — Interview-Ready Notes
### For Java Backend Developer Interviews

---

## 1. Inversion of Control (IoC)

### What it is
IoC is a **design principle** where the control of object creation, configuration, and lifecycle management is transferred from the application code to a **container/framework**. Instead of your code doing `new Service()`, the container creates and hands you the object.

**Traditional (without IoC):**
```java
class OrderService {
    private PaymentProcessor processor = new CreditCardProcessor(); // tight coupling
}
```
`OrderService` controls the creation of its dependency → hard to swap, hard to test.

**With IoC:**
```java
class OrderService {
    private PaymentProcessor processor;
    // Container "injects" the dependency — OrderService no longer creates it
    OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }
}
```
Now the **Spring IoC Container** creates `OrderService`, resolves that it needs a `PaymentProcessor`, creates/finds that bean, and injects it. Control has been "inverted" — hence Inversion of Control.

### Why it matters
- **Loose coupling** — classes depend on abstractions (interfaces), not concrete implementations.
- **Testability** — you can inject a mock `PaymentProcessor` in unit tests.
- **Centralized configuration** — object graph is managed in one place (XML, annotations, Java config).
- **Reduced boilerplate** — no manual wiring code scattered across the app.

### The Spring IoC Container
Spring's IoC container is responsible for:
1. Instantiating beans
2. Configuring them (wiring dependencies)
3. Assembling the object graph
4. Managing the full lifecycle (init → use → destroy)

The container reads **metadata** — annotations (`@Component`, `@Bean`), XML config, or Java `@Configuration` classes — to know what objects to create and how to wire them.

There are two core container implementations:
- `BeanFactory` — the basic, lightweight container
- `ApplicationContext` — the advanced container (superset of BeanFactory) — covered in detail in Section 3

```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = context.getBean(OrderService.class); // container-managed instance
```

**Key interview point:** IoC is the *principle*; **Dependency Injection (DI)** is the most common *pattern* used to implement IoC in Spring. Other IoC implementation styles include the Service Locator pattern (less common, less preferred because it hides dependencies).

---

## 2. Dependency Injection (DI)

### Definition
DI is a design pattern that implements IoC, where an object's dependencies are **supplied ("injected") by an external entity** (the Spring container) rather than the object creating them itself.

### Types of Dependency Injection in Spring

#### a) Constructor Injection (Recommended)
Dependencies are provided via the class constructor. Ensures the object is always created in a fully-initialized, valid state, and supports **immutability** (`final` fields).

```java
@Component
class OrderService {
    private final PaymentProcessor processor;

    @Autowired // optional on single-constructor classes since Spring 4.3
    public OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }
}
```

#### b) Setter Injection
Dependencies are provided via setter methods. Useful for **optional** dependencies.

```java
@Component
class OrderService {
    private NotificationService notificationService;

    @Autowired
    public void setNotificationService(NotificationService ns) {
        this.notificationService = ns;
    }
}
```

#### c) Field Injection
Dependency is injected directly into the field via reflection.

```java
@Component
class OrderService {
    @Autowired
    private PaymentProcessor processor;
}
```
⚠️ **Not recommended** for production: can't make fields `final`, hides dependencies (no constructor signature to reveal them), harder to unit test without Spring/reflection, and circular dependencies fail silently at runtime instead of failing fast at startup.

### Comparison Table

| Aspect | Constructor | Setter | Field |
|---|---|---|---|
| Immutability | ✅ Yes (`final`) | ❌ No | ❌ No |
| Mandatory deps | ✅ Best fit | ❌ Awkward | ❌ N/A |
| Optional deps | ❌ Awkward | ✅ Best fit | ✅ Works |
| Testability (plain JUnit, no Spring) | ✅ Easy | ⚠️ OK | ❌ Hard |
| Circular dependency detection | ✅ Fails fast at startup | ⚠️ Works (lazy) | ⚠️ Works (lazy) |
| Spring team recommendation | ✅ Preferred | Secondary | Avoid |

### Example: full DI wiring
```java
interface PaymentProcessor {
    void pay(double amount);
}

@Component
class CreditCardProcessor implements PaymentProcessor {
    public void pay(double amount) {
        System.out.println("Paid " + amount + " via Credit Card");
    }
}

@Component
class OrderService {
    private final PaymentProcessor processor;

    public OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }

    public void checkout(double amount) {
        processor.pay(amount);
    }
}

@Configuration
@ComponentScan(basePackages = "com.example")
class AppConfig {}

public class Main {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        OrderService service = ctx.getBean(OrderService.class);
        service.checkout(500.0); // Spring auto-wired CreditCardProcessor into OrderService
    }
}
```

### Common Interview Questions
- **Q: Why prefer constructor injection?** → Immutability, fail-fast on missing deps, easier to test without a Spring context, avoids partially-constructed objects, and naturally prevents circular dependency issues (Spring throws `BeanCurrentlyInCreationException` at startup rather than an NPE at runtime).
- **Q: What happens if two beans implement the same interface?** → Spring throws `NoUniqueBeanDefinitionException` unless you disambiguate using `@Primary` or `@Qualifier`.

```java
@Component @Primary
class CreditCardProcessor implements PaymentProcessor { ... }

@Component
class UpiProcessor implements PaymentProcessor { ... }

@Component
class OrderService {
    public OrderService(@Qualifier("upiProcessor") PaymentProcessor processor) { ... }
}
```
### Spring Beans & Dependency Injection

### What is a Bean?
A **bean** is an object whose lifecycle (creation, configuration, wiring, destruction) is managed by the Spring IoC container instead of by your own code with `new`.

### Declaring Beans — Component Annotations
Spring finds classes to turn into beans via **stereotype annotations**:

| Annotation | Purpose |
|---|---|
| `@Component` | Generic bean, base annotation for the others |
| `@Service` | Business/service layer (semantic marker) |
| `@Repository` | Persistence layer; also enables exception translation (DB exceptions → Spring's `DataAccessException`) |
| `@RestController` | `@Controller` + `@ResponseBody`; web layer, returns JSON/XML directly |

```java
@Service
public class OrderService {
    // becomes a Spring-managed bean
}
```

### Component Scanning
Spring needs to know **where to look** for these annotated classes — that's **component scanning**.

- **Spring Boot**: automatic. `@SpringBootApplication` includes `@ComponentScan` on the package it's in (and sub-packages), so no extra config needed as long as your classes are under the main application's package.
- **Classic Spring**: manual. You need:
```java
  @Configuration
  @ComponentScan(basePackages = "com.example.app")
  public class AppConfig { }
```
or the XML equivalent (`<context:component-scan base-package="..."/>`).

---

### Dependency Injection — Preferred Approach: Constructor Injection

```java
@Service
public class OrderService {

    private final PaymentGateway paymentGateway; // interface, not impl

    // With one constructor, @Autowired is optional (Spring 4.3+)
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

**Why constructor injection is preferred:**
- Allows fields to be `final` → immutability.
- Makes dependencies explicit and required.
- **Fails fast**: if a dependency is missing/misconfigured, the app won't even start (fails at context startup), instead of throwing a `NullPointerException` at runtime deep in some call path.
- Easy to unit test — just call `new OrderService(mockGateway)`, no container needed.

---

### Autowiring by Type

By default, `@Autowired` resolves by **type**.

```java
public interface PaymentGateway { }

@Component
public class StripeGateway implements PaymentGateway { }
```

If there's exactly **one** implementation, Spring injects it automatically.

### Multiple Implementations → `@Qualifier`

If there are **multiple beans of the same type**, Spring can't pick one — you disambiguate with `@Qualifier`.

```java
@Component("stripeGateway")
public class StripeGateway implements PaymentGateway { }

@Component("paypalGateway")
public class PaypalGateway implements PaymentGateway { }

@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(@Qualifier("stripeGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

### Default Choice → `@Primary`

Mark one implementation as the **default** when multiple exist, so callers don't need `@Qualifier` unless they want a non-default one.

```java
@Component
@Primary
public class StripeGateway implements PaymentGateway { }

@Component
public class PaypalGateway implements PaymentGateway { }
```

### By Name → `@Resource`

`@Resource` (JSR-250, `javax.annotation` / `jakarta.annotation`) injects **by name first**, falling back to type.

```java
@Resource(name = "paypalGateway")
private PaymentGateway paymentGateway;
```
Contrast: `@Autowired` = by type (+ `@Qualifier` for name); `@Resource` = by name (+ type fallback).

### Injecting Properties → `@Value`

```java
@Value("${payment.api-key}")
private String apiKey;
```
Pulls from `application.properties` / `application.yml`, env vars, etc.

---

### Inject Interfaces, Not Implementations

```java
// Good — depends on abstraction
private final PaymentGateway gateway;

// Bad — tightly coupled to concrete class
private final StripeGateway gateway;
```
**Why:** loose coupling (swap implementations without touching consumers), easier mocking in unit tests, aligns with Dependency Inversion Principle.

---

### Circular Dependencies

Happens when Bean A needs Bean B, and Bean B needs Bean A.

```java
@Component
class A { A(B b) {} }

@Component
class B { B(A a) {} }
```

- With **constructor injection**, this **fails fast at startup** with `BeanCurrentlyInCreationException` — Spring literally cannot construct either bean first.
- With **field/setter injection**, Spring can sometimes resolve this by injecting a half-initialized bean — which **hides a design problem** instead of surfacing it.

**Correct fix:** refactor — usually means:
- Extract shared logic into a third bean that both A and B depend on.
- Merge A and B if they're really one responsibility.
- Use events (`ApplicationEventPublisher`) to decouple the interaction.

**Anti-pattern to avoid:** using `@Lazy` on one of the constructor params just to "make the circular dependency go away.” This papers over bad design instead of fixing it — the coupling issue still exists, just deferred to first use.

```java
// Avoid this "fix"
public A(@Lazy B b) { this.b = b; }
```

**Interview soundbite:**
> "Constructor injection is preferred because it enforces required dependencies, enables immutability, and fails fast at startup rather than at runtime. If you hit a circular dependency, that's a design smell — the right move is to refactor (extract a shared collaborator, or use events), not to reach for `@Lazy` or field injection to mask it."

---

## 3. BeanFactory vs. ApplicationContext

Both are implementations of the Spring **IoC container**, but they differ in capability and use case.

### BeanFactory
- Root interface: `org.springframework.beans.factory.BeanFactory`
- The **most basic** container — provides fundamental DI support.
- **Lazy initialization by default** — beans are created only when `getBean()` is called.
- Lightweight — suited for memory-constrained environments (rarely used directly today).

### ApplicationContext
- Interface: `org.springframework.context.ApplicationContext`
- **Extends `BeanFactory`** (a superset) and adds enterprise-level features.
- **Eagerly initializes singleton beans by default** at container startup (fail-fast — configuration errors surface immediately rather than on first use).
- This is the container used in virtually all real-world Spring/Spring Boot apps.

### Feature Comparison

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation | Lazy (on `getBean()`) | Eager (singletons at startup) |
| Internationalization (i18n) | ❌ No | ✅ `MessageSource` support |
| Event publishing (`ApplicationEvent`) | ❌ No | ✅ Yes |
| AOP integration (auto-proxying) | Limited | ✅ Full support |
| Annotation-based config (`@Autowired`, `@Component`) | Needs manual `BeanPostProcessor` registration | ✅ Built-in |
| Environment abstraction (profiles, properties) | ❌ No | ✅ Yes |
| Common implementations | `XmlBeanFactory` (deprecated) | `AnnotationConfigApplicationContext`, `ClassPathXmlApplicationContext`, `GenericWebApplicationContext` |

### Code Example
```java
// BeanFactory — rarely used in practice
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml")); // deprecated in modern Spring
MyBean bean = (MyBean) factory.getBean("myBean"); // triggers creation now (lazy)

// ApplicationContext — standard, modern approach
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// All singleton beans are already created by this point (eager)
MyBean bean2 = context.getBean(MyBean.class);
```

**Interview one-liner:** *"BeanFactory is the basic container providing DI; ApplicationContext is a superset adding enterprise features like event handling, AOP, i18n, and eager singleton instantiation — it's what Spring Boot uses under the hood (e.g., `AnnotationConfigApplicationContext`)."*

---

## 4. Spring Bean Lifecycle

Understanding this is a very common interview topic — know the **order of phases**.

### Lifecycle Phases (High Level)
1. **Instantiation** — container creates the bean instance (constructor call).
2. **Populate properties** — dependencies injected (setter/field injection resolved).
3. **`BeanNameAware.setBeanName()`** — if bean implements it, Spring sets the bean's name.
4. **`BeanFactoryAware` / `ApplicationContextAware`** — container references injected if implemented.
5. **`BeanPostProcessor.postProcessBeforeInitialization()`** — runs for all beans.
6. **`@PostConstruct`** method invoked (or `InitializingBean.afterPropertiesSet()`, or custom `init-method`).
7. **`BeanPostProcessor.postProcessAfterInitialization()`** — runs for all beans (this is where Spring AOP proxies are typically created).
8. **Bean is ready** — fully initialized, in use by the application.
9. **`@PreDestroy`** (or `DisposableBean.destroy()`, or custom `destroy-method`) — called when the container shuts down (only for singleton beans by default; prototype beans are **not** managed through destruction by Spring).

### Diagram (textual)
```
Instantiate → Populate Properties → Aware Interfaces →
BeanPostProcessor(before) → @PostConstruct / afterPropertiesSet() →
BeanPostProcessor(after) → [ Bean Ready / In Use ] →
@PreDestroy / destroy() → Bean Destroyed
```

### Code Example — All Hooks
```java
@Component
class Engine implements InitializingBean, DisposableBean, BeanNameAware {

    public Engine() {
        System.out.println("1. Constructor called");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("2. BeanNameAware: bean name = " + name);
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct called");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("4. InitializingBean.afterPropertiesSet() called");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("5. @PreDestroy called");
    }

    @Override
    public void destroy() {
        System.out.println("6. DisposableBean.destroy() called");
    }
}
```

### Custom `init-method` / `destroy-method` (Java Config style, no annotations needed on the bean class)
```java
class Database {
    public void connect() { System.out.println("Connecting to DB..."); }
    public void disconnect() { System.out.println("Disconnecting DB..."); }
}

@Configuration
class AppConfig {
    @Bean(initMethod = "connect", destroyMethod = "disconnect")
    public Database database() {
        return new Database();
    }
}
```

### BeanPostProcessor Example (used internally for things like AOP proxy creation)
```java
@Component
class LoggingBeanPostProcessor implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean;
    }
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After init: " + beanName);
        return bean;
    }
}
```

**Key interview point:** `@PostConstruct`/`@PreDestroy` are the **recommended, modern approach** (JSR-250 standard annotations, framework-agnostic). `InitializingBean`/`DisposableBean` couple your code to Spring APIs. Also: **destroy callbacks only fire for singleton-scoped beans** managed by the container — prototype beans are handed off to the client and Spring does not track/destroy them.

---

## 5. Singleton and Prototype Bean Scopes

Bean **scope** defines how many instances of a bean the container creates and how it's shared.

### Singleton (Default Scope)
- **Exactly one instance** per Spring IoC container, shared across the entire application.
- Cached in the container; every `getBean()` call or injection returns the **same object reference**.
- Best for **stateless** beans (services, DAOs, repositories).

```java
@Component // default scope = singleton
class CounterService {
    private int count = 0;
    public void increment() { count++; }
    public int getCount() { return count; }
}
```
```java
CounterService s1 = context.getBean(CounterService.class);
CounterService s2 = context.getBean(CounterService.class);
s1.increment();
System.out.println(s1 == s2);        // true — same instance
System.out.println(s2.getCount());   // 1 — shared state!
```

### Prototype
- **A new instance is created every time** the bean is requested (`getBean()` or injected).
- Best for **stateful** beans where each caller needs its own instance.

```java
@Component
@Scope("prototype")
class ShoppingCart {
    private List<String> items = new ArrayList<>();
    public void addItem(String item) { items.add(item); }
    public List<String> getItems() { return items; }
}
```
```java
ShoppingCart cart1 = context.getBean(ShoppingCart.class);
ShoppingCart cart2 = context.getBean(ShoppingCart.class);
cart1.addItem("Book");
System.out.println(cart1 == cart2);            // false — different instances
System.out.println(cart2.getItems().size());   // 0 — independent state
```

### Key Differences

| Aspect | Singleton | Prototype |
|---|---|---|
| Instances per container | 1 | New instance every request |
| Default scope | ✅ Yes | ❌ Must specify `@Scope("prototype")` |
| State | Should be stateless | Can be stateful |
| Lifecycle managed fully by Spring | ✅ (init + destroy) | ⚠️ Only creation + init; **not** destruction |
| Memory | Efficient (1 object) | More objects created — use carefully |
| Cached in container | ✅ Yes | ❌ No |

### ⚠️ Common Gotcha: Injecting a Prototype into a Singleton
If a singleton bean has a prototype dependency injected normally, it only gets **one** instance of the prototype (injected once, at singleton creation time) — defeating the purpose!

```java
@Component
class ReportGenerator { // singleton
    @Autowired
    private ShoppingCart cart; // prototype — but only injected ONCE!
}
```

**Fix 1 — `ObjectProvider` / `Provider`:**
```java
@Component
class ReportGenerator {
    @Autowired
    private ObjectProvider<ShoppingCart> cartProvider;

    public void generate() {
        ShoppingCart cart = cartProvider.getObject(); // fresh instance each call
    }
}
```

**Fix 2 — Lookup method injection:**
```java
@Component
abstract class ReportGenerator {
    @Lookup
    public abstract ShoppingCart createCart(); // Spring overrides this at runtime
}
```

**Fix 3 — `proxyMode` on the prototype bean:**
```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
class ShoppingCart { ... }
```

---

## 6. Custom Bean Scope

Besides `singleton`, `prototype`, and the web-aware scopes (`request`, `session`, `application`), Spring lets you define your **own custom scope** by implementing the `Scope` interface.

### Steps to Create a Custom Scope
1. Implement `org.springframework.beans.factory.config.Scope`.
2. Register it with `ConfigurableBeanFactory.registerScope(name, scope)`.
3. Use `@Scope("yourScopeName")` on beans.

### Example: A Simple "Thread" Scope (one instance per thread)
```java
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.beans.factory.config.Scope;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

class ThreadScope implements Scope {

    private final ThreadLocal<Map<String, Object>> threadLocalScope =
            ThreadLocal.withInitial(ConcurrentHashMap::new);

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = threadLocalScope.get();
        return scope.computeIfAbsent(name, key -> objectFactory.getObject());
    }

    @Override
    public Object remove(String name) {
        return threadLocalScope.get().remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        // Optional: store callback to invoke on thread cleanup
    }

    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    public String getConversationId() {
        return Thread.currentThread().getName();
    }
}
```

### Registering the Custom Scope
```java
@Configuration
class ScopeConfig {

    @Bean
    public static CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer configurer = new CustomScopeConfigurer();
        configurer.addScope("thread", new ThreadScope());
        return configurer;
    }
}
```

### Using the Custom Scope
```java
@Component
@Scope("thread")
class ThreadLocalCache {
    private final Map<String, Object> cache = new HashMap<>();
    public void put(String key, Object val) { cache.put(key, val); }
    public Object get(String key) { return cache.get(key); }
}
```

**Why this matters in interviews:** shows deep understanding that Spring's scope mechanism is **pluggable** — not limited to the 4-5 built-in scopes. Good example use cases: tenant-scoped beans in multi-tenant SaaS apps, thread-scoped beans, or scopes tied to a custom business "conversation" concept.

---

## 7. Creating a Spring Bean (3 Ways)

### Way 1 — XML Configuration (Legacy but still asked about)
```xml
<!-- beans.xml -->
<beans xmlns="http://www.springframework.org/schema/beans" ...>
    <bean id="engine" class="com.example.Engine"/>
    <bean id="car" class="com.example.Car">
        <constructor-arg ref="engine"/>
    </bean>
</beans>
```
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
Car car = (Car) ctx.getBean("car");
```

### Way 2 — Annotation-Based (Stereotype Annotations + Component Scanning)
Mark classes with `@Component` (or specialized stereotypes: `@Service`, `@Repository`, `@Controller`), and enable scanning.

```java
@Component
class Engine {}

@Component
class Car {
    private final Engine engine;
    public Car(Engine engine) { this.engine = engine; }
}

@Configuration
@ComponentScan(basePackages = "com.example")
class AppConfig {}
```
```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
Car car = ctx.getBean(Car.class);
```

**Stereotype annotations** — all meta-annotated with `@Component`, but convey semantic intent:

| Annotation | Layer |
|---|---|
| `@Component` | Generic bean |
| `@Service` | Service/business logic layer |
| `@Repository` | DAO/persistence layer (adds automatic exception translation for `DataAccessException`) |
| `@Controller` / `@RestController` | Web layer (MVC) |

### Way 3 — Java-Based Configuration Using `@Bean` (most flexible — full manual control)
Useful for wiring third-party classes you can't annotate (e.g., library classes).

```java
class Engine {
    public Engine(String type) { this.type = type; }
    private String type;
}

class Car {
    private Engine engine;
    public Car(Engine engine) { this.engine = engine; }
}

@Configuration
class AppConfig {

    @Bean
    public Engine engine() {
        return new Engine("V8");
    }

    @Bean
    public Car car() {
        return new Car(engine()); // explicit wiring
    }
}
```
```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
Car car = ctx.getBean(Car.class);
```

### Comparison

| Method | Pros | Cons |
|---|---|---|
| XML | Decoupled from code, no recompilation for config changes | Verbose, no compile-time checking, largely legacy today |
| Annotation + Component Scan | Concise, colocated with class, minimal boilerplate | Requires modifying the class source (can't use for 3rd-party classes) |
| `@Bean` in `@Configuration` | Full control, works for any class (including 3rd party libs), type-safe, refactor-friendly (IDE support) | Slightly more verbose than stereotype annotations |

**Interview tip:** Spring Boot apps typically **combine** annotation scanning (`@Component`/`@Service`) for your own classes with `@Bean` methods in `@Configuration` classes for third-party beans (e.g., configuring a `RestTemplate`, `ObjectMapper`, or a `DataSource`).

---

## 8. Spring Autowiring

Autowiring lets Spring **automatically resolve and inject** collaborating beans, without explicit wiring in config.

### `@Autowired` — the primary mechanism
Can be applied to constructors, setters, or fields (constructor preferred — see Section 2).

```java
@Component
class OrderService {
    private final PaymentProcessor processor;

    @Autowired
    public OrderService(PaymentProcessor processor) {
        this.processor = processor;
    }
}
```

### Autowiring Modes (Historically, in XML — good to know conceptually)
| Mode | Behavior |
|---|---|
| `no` | Default; no autowiring, explicit wiring required |
| `byName` | Matches a bean whose **id** matches the property name |
| `byType` | Matches a bean whose **type** matches the property type |
| `constructor` | Like `byType`, but applied to constructor arguments |

With annotations, Spring effectively does **byType** matching by default via `@Autowired`.

### Resolving Ambiguity: Multiple Beans of the Same Type
```java
interface Notifier {}

@Component("emailNotifier")
class EmailNotifier implements Notifier {}

@Component("smsNotifier")
class SmsNotifier implements Notifier {}

@Component
class AlertService {
    private final Notifier notifier;

    @Autowired
    public AlertService(@Qualifier("smsNotifier") Notifier notifier) {
        this.notifier = notifier; // explicitly picks SmsNotifier
    }
}
```

Or mark one bean as the default:
```java
@Component
@Primary
class EmailNotifier implements Notifier {} // chosen when no @Qualifier given
```

### Optional Dependencies
```java
@Autowired(required = false)
private AuditLogger auditLogger; // won't fail startup if no bean exists
```

Or with `Optional`:
```java
@Autowired
private Optional<AuditLogger> auditLogger;
```

### Autowiring Collections
Spring can inject **all** beans of a type as a `List`, `Set`, or `Map`:
```java
@Component
class NotificationDispatcher {
    private final List<Notifier> notifiers;

    @Autowired
    public NotificationDispatcher(List<Notifier> notifiers) {
        this.notifiers = notifiers; // injects EmailNotifier + SmsNotifier, in order (use @Order to control)
    }
}
```

### `@Resource` vs `@Autowired` vs `@Inject`
| Annotation | Source | Matching Strategy |
|---|---|---|
| `@Autowired` | Spring | byType first, then `@Qualifier`/name as tiebreaker |
| `@Resource` | JSR-250 (`javax`/`jakarta.annotation`) | byName first, then byType |
| `@Inject` | JSR-330 (`javax`/`jakarta.inject`) | byType, like `@Autowired` (needs `@Named` for qualifying) — framework-agnostic |

**Interview tip:** `@Resource` and `@Inject` are useful to mention if asked about writing framework-agnostic code, since they're Java standard annotations, not Spring-specific.

---

## 9. DispatcherServlet

### What it is
`DispatcherServlet` is the **front controller** of Spring MVC — the single entry point for all incoming HTTP requests in a Spring web application. It implements the **Front Controller design pattern**.

### Why it exists
Instead of each URL mapping to a different servlet (old Java EE style), **all** requests go through one servlet (`DispatcherServlet`), which then delegates to the appropriate handler (controller method) based on configured mappings. This centralizes cross-cutting request-processing logic (routing, exception handling, view resolution).

### Request Processing Flow
```
1. Client sends HTTP request
2. DispatcherServlet receives it (registered against "/" or a URL pattern in web.xml / auto-configured by Spring Boot)
3. DispatcherServlet consults HandlerMapping to find the right Controller + method
4. HandlerAdapter invokes the actual controller method
5. Controller executes business logic, returns a ModelAndView (or @ResponseBody data)
6. If a view name is returned, ViewResolver resolves it to an actual View (JSP, Thymeleaf, etc.)
7. View is rendered with the Model data
8. DispatcherServlet sends the final HTTP response back to the client
```

### Diagram (textual)
```
Client
  |
  v
DispatcherServlet  <-- Front Controller
  |
  |--> HandlerMapping        (which controller handles this URL?)
  |--> HandlerAdapter        (invoke the controller method)
  |--> Controller             (@Controller / @RestController)
  |--> ViewResolver           (resolve logical view name -> actual View, e.g. JSP)
  |--> View                   (render response)
  |
  v
HTTP Response
```

### Traditional web.xml Registration (pre-Boot)
```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-mvc-config.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

### In Spring Boot
Spring Boot **auto-configures `DispatcherServlet`** for you via `DispatcherServletAutoConfiguration` — no manual registration needed. You just write controllers:

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        Order order = new Order(id, "Book");
        return ResponseEntity.ok(order); // DispatcherServlet routes here automatically
    }

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        // process order
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
}
```
When this app starts, `DispatcherServlet` is registered against `/` by the embedded servlet container (Tomcat by default), and Spring builds a `HandlerMapping` that maps `GET /api/orders/{id}` and `POST /api/orders` to these methods.

### `@Controller` vs `@RestController`
- `@Controller` — returns **view names**; typically used with server-rendered views (JSP/Thymeleaf).
- `@RestController` = `@Controller` + `@ResponseBody` — return values are serialized (usually to JSON) directly into the HTTP response body — standard for REST APIs.

**Interview one-liner:** *"DispatcherServlet is Spring MVC's front controller — every request funnels through it, and it orchestrates handler mapping, controller invocation, and view resolution. In Spring Boot it's auto-configured, so developers just write `@RestController` classes."*

---

## 10. Build Tools — Maven vs Gradle

### What a build tool does
Automates the software build lifecycle:
- **Dependency management** — download and manage third-party libraries (and their transitive dependencies) from repositories (e.g., Maven Central).
- **Compilation** — compile source code.
- **Testing** — run unit/integration tests.
- **Packaging** — bundle into JAR/WAR.
- **Deployment** — publish artifacts to a repository or deploy to a server.

Without a build tool, you'd manually download JARs, manage classpath, and script compilation — error-prone at any real scale.

### Maven
Uses **declarative XML** (`pom.xml`) and a fixed, convention-driven lifecycle.

```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Maven Build Lifecycle (default, in order):**
```
validate → compile → test → package → verify → install → deploy
```
Running `mvn install` executes every phase up to and including `install` in sequence.

Common commands:
```bash
mvn clean install     # clean previous build, compile, test, package, install to local repo
mvn spring-boot:run   # run a Spring Boot app directly
mvn dependency:tree   # view the full dependency graph (useful for resolving version conflicts)
```

### Gradle
Uses a **Groovy or Kotlin DSL** (`build.gradle` / `build.gradle.kts`) — more concise and programmable than XML.

```groovy
// build.gradle (Groovy DSL)
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'java'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```

Common commands:
```bash
gradle build           # compile, test, package
gradle bootRun         # run a Spring Boot app
gradle dependencies    # view the dependency graph
```

### Maven vs Gradle Comparison

| Aspect | Maven | Gradle |
|---|---|---|
| Config format | XML (`pom.xml`) — verbose, declarative | Groovy/Kotlin DSL (`build.gradle`) — concise, programmable |
| Build model | Fixed lifecycle phases | Flexible, task-based (define custom tasks easily) |
| Performance | Slower (no built-in incremental/parallel build by default in older versions) | Faster — incremental builds, build caching, parallel execution |
| Dependency management | Central `<dependencies>` block; conflict resolution via "nearest wins" | More flexible conflict resolution strategies |
| Learning curve | Simpler to read/predict (rigid conventions) | Steeper (more powerful, more ways to do things) |
| IDE/tooling support | Extremely mature | Excellent, especially in Android & modern Spring Boot projects |
| Multi-module projects | Verbose but standard support | Cleaner support via `settings.gradle` |
| Spring Boot default | Both fully supported — `start.spring.io` lets you pick either | Both fully supported |

**Interview tip:** Be ready to explain the Maven lifecycle phases in order, and know that Gradle's speed advantage (incremental builds + caching) is the #1 reason large projects (e.g., many Android and modern microservices) prefer it, while Maven's rigidity is often preferred for **standardized, predictable** enterprise builds.

---

## Quick-Fire Interview Summary Table

| Topic | One-Line Answer |
|---|---|
| IoC | Principle: control of object creation/wiring is handed to a container, not the app code |
| DI | Pattern implementing IoC: container injects dependencies via constructor/setter/field |
| BeanFactory vs ApplicationContext | ApplicationContext = BeanFactory + eager init + events + AOP + i18n; used everywhere in practice |
| Bean Lifecycle | Instantiate → populate props → Aware callbacks → BeanPostProcessor(before) → init (`@PostConstruct`) → BeanPostProcessor(after) → ready → destroy (`@PreDestroy`) |
| Singleton scope | One shared instance per container; default scope |
| Prototype scope | New instance every `getBean()`/injection call; Spring doesn't manage destruction |
| Custom scope | Implement `Scope` interface + register via `CustomScopeConfigurer` |
| Creating a bean | XML / `@Component` + component-scan / `@Bean` in `@Configuration` |
| Autowiring | `@Autowired` resolves by type, disambiguated with `@Qualifier`/`@Primary` |
| DispatcherServlet | Front controller for Spring MVC — routes all requests through HandlerMapping → Controller → ViewResolver |
| Maven/Gradle | Build tools automating dependency management, compilation, testing, packaging; Maven=XML/fixed lifecycle, Gradle=DSL/flexible tasks, faster incremental builds |