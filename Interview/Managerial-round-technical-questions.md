# SAP Fioneer — Technical Interview Revision Notes (Java Backend)

**Interview:** Final technical + managerial round, Friday, 20 November 2026  
**Candidate context:** ~3 years of experience in Java/Spring financial trade-processing systems at Barclays  
**Purpose:** A stand-alone, interview-ready technical study handbook — explanations, runnable-style examples, likely questions, short model answers, traps, and practical engineering decisions.

> **How to use this guide:** For each topic, first read the **60-second answer**, understand the code and **pitfalls**, and then speak the Q&A answers aloud. Don't claim that a suggested design was deployed at Barclays unless you actually implemented it. Project-specific facts are marked **[YOUR WORK]**; proposals and generic code are **[ILLUSTRATIVE]**.
>
> **Interview response pattern:** **Define → Explain mechanism → Show example → Explain a trade-off → Relate to production.** Aim for 45–90 seconds on ordinary concepts and 2–4 minutes on architecture questions.

## Priority map and table of contents

| Priority | Topic | Why it matters |
|---|---|---|
| **P0** | [1. OOP, SOLID and patterns](#1-object-oriented-programming-oop-solid-and-design-patterns--p0) | Mandatory foundation; modeling trading products |
| **P0** | [2. Core Java, collections, JVM](#2-core-java-collections-and-jvm--p0) | Java language and backend reliability |
| **P0** | [3. Multithreading and concurrency](#3-multithreading-and-concurrency--p0) | Directly tied to your booking optimization |
| **P0** | [4. Spring Framework and Spring Boot](#4-spring-framework-and-spring-boot--p0) | Primary production stack |
| **P0** | [5. REST, microservices and messaging](#5-rest-microservices-and-messaging--p0) | Financial integrations and reliability |
| **P0** | [6. SQL, transactions and performance](#6-sql-transactions-and-performance--p0) | 18.5M-trade archival and correctness |
| **P0** | [7. Caching and distributed systems](#7-caching-distributed-systems-and-system-design--p0) | Oracle Coherence and architecture decisions |
| **P1** | [8. Production, testing and DevOps](#8-production-troubleshooting-testing-and-devops--p1) | Managerial ownership and incidents |
| **P2** | [9. Docker, Kubernetes, cloud, security, JPA, modern Java](#9-additional-engineering-topics--p2) | Useful follow-up breadth |
| **P0** | [10. Combined technical–managerial scenarios](#10-high-probability-technical-managerial-scenarios) | Tests engineering judgment |
| **Review** | [11. Rapid-fire Q&A and cheat sheet](#11-rapid-fire-questions-and-final-cheat-sheet) | Final interview warm-up |

---

# 1. Object-oriented programming (OOP), SOLID and design patterns — P0

## 1.1 Four pillars of OOP

| Pillar | Meaning | Java mechanism | Finance example |
|---|---|---|---|
| **Encapsulation** | Keep state valid and hide implementation details | `private` fields, controlled methods | Trade status can change only via `approve()` / `cancel()` rules |
| **Abstraction** | Expose *what* something does, not *how* | interfaces, abstract classes | `TradeValidator.validate(trade)` hides rule details |
| **Inheritance** | Reuse/extend behavior via an **is-a** relationship | `extends`, `implements` | A specialized validator can extend an abstract base validator |
| **Polymorphism** | Same contract, different implementation at runtime | overriding, interface dispatch | FX, equity-swap and loan validators selected by trade type |

**60-second answer:** "OOP organizes software around objects that hold state and behavior. The four pillars are encapsulation, abstraction, inheritance and polymorphism. Encapsulation protects invariants, abstraction separates contract from implementation, inheritance allows specialization where an is-a relationship really exists, and polymorphism lets clients depend on a common interface. In financial services, I would use a common trade validation contract and separate implementations for different products, so the booking flow remains stable when a new trade type is added. I usually prefer composition and interfaces over deep inheritance because they are easier to test and change."

### Encapsulation: protect a business invariant

```java
import java.math.BigDecimal;

public final class Trade {
    private final String tradeId;
    private BigDecimal quantity;
    private Status status;

    public Trade(String tradeId, BigDecimal quantity) {
        if (tradeId == null || tradeId.isBlank()) throw new IllegalArgumentException("tradeId");
        if (quantity == null || quantity.signum() <= 0) throw new IllegalArgumentException("quantity");
        this.tradeId = tradeId;
        this.quantity = quantity;
        this.status = Status.NEW;
    }

    public void approve() {
        if (status != Status.NEW) throw new IllegalStateException("Only NEW trades can be approved");
        status = Status.APPROVED;
    }

    public String tradeId() { return tradeId; }
    public Status status() { return status; }
    public BigDecimal quantity() { return quantity; }
    public enum Status { NEW, APPROVED, CANCELLED }
}
```

`private` is not sufficient by itself: the real benefit comes from methods that enforce invariants. Use `BigDecimal` for money/prices rather than floating-point types when exact decimal behavior matters.

### Abstraction + polymorphism (with composition)

```java
import java.util.Map;

interface TradeValidator {
    void validate(TradeRequest request);
}
record TradeRequest(String productType, String counterparty) {}

final class FxTradeValidator implements TradeValidator {
    @Override public void validate(TradeRequest request) {
        if (request.counterparty() == null) throw new IllegalArgumentException("Counterparty required");
    }
}
final class EquitySwapValidator implements TradeValidator {
    @Override public void validate(TradeRequest request) {
        // Equity-swap-specific rules
    }
}

final class TradeValidationService {
    private final Map<String, TradeValidator> validators;
    TradeValidationService(Map<String, TradeValidator> validators) {
        this.validators = Map.copyOf(validators);
    }
    void validate(TradeRequest request) {
        TradeValidator validator = validators.get(request.productType());
        if (validator == null) throw new IllegalArgumentException("Unsupported product");
        validator.validate(request);
    }
}
```

The caller is unchanged when another validator is added. This is the **Strategy pattern**: behavior chosen through a common contract. This illustrative snippet is a simplified design, not your exact Atlas code.

## 1.2 Interface vs abstract class

| Interface | Abstract class |
|---|---|
| Contract/capability; a class can implement many interfaces | Shared base implementation/state; only one class can be extended |
| Supports abstract, `default`, `static`, and private helper methods | Can have constructors, instance fields, protected methods |
| Good for `TradeValidator`, `Publisher`, `Repository` | Good when related implementations genuinely share behavior/state |

**Q: Can interfaces contain method implementations?** **A:** Yes — `default` and `static` methods (and private helper methods in modern Java). They are not the same as abstract classes with instance state and constructors.

**Q: Overloading vs overriding?** **A:** Overloading means same name, different parameter lists, resolved at compile time; overriding means a subclass provides a new implementation of an inherited instance method, selected dynamically. Static methods are hidden, not overridden.

**Q: Is Java pass-by-reference?** **A:** No. Java is always pass-by-value. For object variables, the *reference value* is copied, so a method can mutate the referenced object but cannot rebind the caller's variable.

**Q: Can we override a `private`, `final` or `static` method?** **A:** Private methods aren't inherited; final instance methods can't be overridden; static methods are hidden, not dynamically overridden.

**Q: Composition vs inheritance?** **A:** Prefer composition for flexible "has-a" relationships and variable behavior. Use inheritance only where the Liskov substitution principle holds; otherwise adding behavior via injected collaborators is cleaner.

## 1.3 SOLID principles

1. **S — Single Responsibility:** A class should have one reason to change. Separate validation, trade persistence and publishing.
2. **O — Open/Closed:** Extend through new strategies instead of modifying a giant `switch` for every product.
3. **L — Liskov Substitution:** A subtype must preserve expected base behavior. Do not let a specialized validator silently skip mandatory checks promised by its contract.
4. **I — Interface Segregation:** Clients should depend only on operations they need; don't force a read-only query service to implement `cancelTrade()`.
5. **D — Dependency Inversion:** Core booking logic depends on `TradeRepository` / `Publisher` interfaces rather than concrete clients; inject implementations with Spring.

**Q: Which SOLID principle do you use most?** **A:** "SRP and dependency inversion are particularly practical in microservices. They help isolate business validation from transport, persistence and external clients, making unit tests cheaper and changes safer."

## 1.4 Design patterns to know

| Pattern | Idea | Example | Watch-out |
|---|---|---|---|
| Strategy | Swap algorithm behind interface | Product-specific validation | Avoid a huge if/else factory |
| Factory | Centralize object selection/construction | Create handler for product type | Don't hide complex side effects |
| Builder | Construct complex objects readably | Request with 40+ fields | Builder doesn't validate automatically |
| Template Method | Skeleton flow with customizable steps | Common booking pipeline | Tight inheritance can make change harder |
| Observer / Pub-Sub | Notify subscribers on event | Trade booked → event consumers | Delivery may be duplicated/out-of-order |
| Adapter | Translate one interface/schema to another | Delta One request → Atlas DTO | Preserve semantic meaning, not just field names |
| Decorator | Add behavior without changing target | Metrics/logging around remote client | Avoid deep wrapper chains |
| Singleton | One instance per relevant boundary | Stateless Spring service bean | Not automatically thread-safe |

**[YOUR WORK]** Your Delta One STP mapping of 40+ fields and your CCLT/TRAC validation flows are natural examples for **Adapter**, **Strategy** and **separation of concerns**. Use these as analogies; confirm whether a specific named pattern was intentionally implemented.

### OOP interview traps

- An interface is not "100% abstract" in modern Java.
- A Spring singleton is **one bean per container / bean definition**, not necessarily one instance across the entire application cluster.
- OOP doesn't mean every database table must map to an elaborate inheritance hierarchy.
- Favor immutable DTOs and stateless singleton services; mutable fields in singleton service classes are a concurrency hazard.

---

# 2. Core Java, collections and JVM — P0

## 2.1 `==` vs `equals()` vs `hashCode()`

**60-second answer:** "`==` compares primitive values or whether object references are identical; `equals()` compares logical equality if overridden. Equal objects must produce equal hash codes. `HashMap` uses a key's hash to locate a bucket and then uses equality checks to match keys. If I use a custom cache or map key, I implement both `equals()` and `hashCode()`, and keep the key immutable so it remains retrievable after insertion."

```java
import java.util.Objects;

final class NexusKey {
    private final String traderBrid;
    private final String salespersonBrid;

    NexusKey(String traderBrid, String salespersonBrid) {
        this.traderBrid = Objects.requireNonNull(traderBrid);
        this.salespersonBrid = Objects.requireNonNull(salespersonBrid);
    }
    @Override public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof NexusKey)) return false;
        NexusKey that = (NexusKey) other;
        return traderBrid.equals(that.traderBrid)
            && salespersonBrid.equals(that.salespersonBrid);
    }
    @Override public int hashCode() {
        return Objects.hash(traderBrid, salespersonBrid);
    }
}
```

In Java 17, a `record NexusKey(String traderBrid, String salespersonBrid) {}` is a concise immutable *shallow* value carrier whose generated equality/hash methods suit this example.

**Q: What if `equals()` is overridden but `hashCode()` isn't?** **A:** Two equal keys may land in different buckets, breaking lookups and violating the contract.

**Q: Can unequal objects share a hash code?** **A:** Yes. Collisions are allowed and resolved using equality comparison.

## 2.2 `HashMap` internals vs `ConcurrentHashMap`

- `HashMap` uses a bucket array; hashes determine bucket index. Collisions form chains and can become balanced trees under thresholds and sufficient capacity. Average `get/put` is O(1), worst cases can be worse.
- Default load factor is commonly `0.75`; resize generally increases capacity as threshold is crossed.
- Ordinary `HashMap` is **not safe for unsynchronized concurrent mutation**; using `Collections.synchronizedMap` serializes access through synchronization.
- `ConcurrentHashMap` supports concurrent access with fine-grained coordination; it **does not allow null keys/values** and offers atomic APIs such as `putIfAbsent`, `compute`, `merge`.
- A single `get()` followed by `put()` is still a multi-step race even with `ConcurrentHashMap`. Use an atomic map operation.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

ConcurrentHashMap<String, AtomicInteger> failures = new ConcurrentHashMap<>();
failures.computeIfAbsent("job-42", ignored -> new AtomicInteger()).incrementAndGet();
```

**Q: Is `ConcurrentHashMap` always better than `HashMap`?** **A:** No. Use `HashMap` for local, single-threaded state. A concurrent map adds concurrency-related overhead; it also does not make arbitrary multi-step business logic atomic.

**Q: Why should map keys be immutable?** **A:** Mutating fields used by `hashCode()` changes the logical bucket, so subsequent lookups can fail even though the object is still stored.

**Collections refresher:**

| Type | Ordering | Duplicates | Typical usage |
|---|---|---|---|
| `ArrayList` | Insertion order | Yes | Indexed retrieval; amortized append |
| `LinkedList` | Insertion order | Yes | Queue/deque operations; often worse cache locality |
| `HashSet` | No guaranteed order | No | Uniqueness |
| `LinkedHashSet` | Insertion order | No | Uniqueness + iteration order |
| `TreeSet` | Sorted | No under comparator | Ordered ranges |
| `HashMap` | No guaranteed order | Unique keys | Fast key-value lookup |
| `LinkedHashMap` | Predictable iteration order | Unique keys | Order-aware maps / LRU building blocks |
| `TreeMap` | Sorted by key | Unique keys | Range queries |
| `CopyOnWriteArrayList` | Ordered snapshot iteration | Yes | Read-heavy, rare writes |

**Q: `ArrayList` vs `LinkedList`?** **A:** ArrayList is usually faster for iteration/random access due to locality; LinkedList can efficiently insert/remove at already-located nodes but pays memory and traversal overhead. Don't assume LinkedList is better for all insertions.

## 2.3 Streams, lambdas and `Optional`

**60-second answer:** "A stream expresses a pipeline over data: source, intermediate operations like `filter`/`map`, then a terminal operation like `collect` or `sum`. Intermediate operations are lazy. I use streams for readable transformations, not to conceal side effects or database operations. `Optional` is useful for an absent return value but isn't a substitute for validating required inputs."

```java
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

record Booking(String product, BigDecimal amount, boolean active) {}
List<Booking> bookings = List.of(
    new Booking("FX", new BigDecimal("100"), true),
    new Booking("FX", new BigDecimal("50"), false),
    new Booking("EQ", new BigDecimal("75"), true));

Map<String, BigDecimal> totals = bookings.stream()
    .filter(Booking::active)
    .collect(Collectors.groupingBy(
        Booking::product,
        Collectors.reducing(BigDecimal.ZERO, Booking::amount, BigDecimal::add)));
```

- **`map` vs `flatMap`:** One input → one output vs one input → zero/many flattened outputs.
- **`findFirst` vs `findAny`:** First respects encounter order where applicable; `findAny` is free to select any matching element.
- **`reduce` vs `collect`:** Reduce combines values; collect builds mutable result containers or grouped structures.
- **`orElse` vs `orElseGet`:** `orElse(expensive())` computes the argument eagerly; `orElseGet(() -> expensive())` runs lazily when empty.
- **`parallelStream()`:** Not a blanket performance optimization; uses shared fork-join resources by default, may cause contention and unpredictable order of side effects.

**Q: Functional interface?** **A:** Interface with exactly one abstract method, such as `Predicate<T>`, `Function<T,R>`, `Consumer<T>` or `Supplier<T>`. Default methods don't count toward the single-abstract-method total.

**Q: `map` or `flatMap` with `Optional`?** **A:** `map` when the mapping returns a plain value; `flatMap` when it returns an `Optional`, preventing nested `Optional<Optional<T>>`.

## 2.4 Java exceptions, immutability, generics

- **Checked exceptions:** Compiler generally requires catch or declaration; **unchecked exceptions** inherit `RuntimeException`. `Error` generally represents JVM/system problems and should not be used for application validation.
- **`throw` vs `throws`:** Actually throw an exception vs declare a method's exception contract.
- **`finally`:** Typically executes even after return, but not guaranteed for abnormal JVM termination.
- **Try-with-resources:** Prefer for connections, streams, statements; closes resources in reverse declaration order; preserves suppressed exceptions.
- **Immutability:** Make fields final, don't expose mutable collections, use defensive copies; `final` object reference alone doesn't make its fields immutable.
- **Generics:** Compile-time type safety; type erasure means many generic type parameters are not retained at runtime. `List<? extends Trade>` is generally for reading; `List<? super Trade>` for adding Trade values (**PECS**: Producer Extends, Consumer Super).
- **String vs StringBuilder:** `String` immutable; StringBuilder mutable and normally faster for repeated concatenation in a single thread; `StringBuffer` synchronizes operations.

**Q: How is `String` immutable?** **A:** Its contents cannot be mutated through the API; operations yield new values. Immutability supports safe sharing and hash caching.

## 2.5 JVM memory and garbage collection

| Component | Holds / purpose | Typical symptom |
|---|---|---|
| Heap | Objects/arrays, GC managed | `OutOfMemoryError: Java heap space`, long GC pauses |
| Thread stack | Frames, local primitives/references | `StackOverflowError` from deep recursion |
| Metaspace | Class metadata | Metaspace pressure with classloader leaks |
| Direct/off-heap memory | NIO buffers/native allocations | OOM outside Java heap; container memory pressure |
| GC | Reclaims unreachable managed objects | CPU or pause spikes |

**60-second answer:** "The JVM stores most objects on the heap, while each thread has a stack for call frames. Garbage collection finds objects that are no longer reachable and reclaims their memory. Under production pressure I would inspect heap usage, GC pause times, allocation rate and live object retention rather than simply increasing `-Xmx`. Common leaks are unbounded caches, static collections, ThreadLocals that aren't cleared, or listeners never unregistered."

**Q: Is Java free from memory leaks?** **A:** No. Objects can remain reachable unintentionally, so GC cannot reclaim them.

**Q: How do you diagnose high memory?** **A:** Observe memory trend, heap/non-heap, GC logs; capture and inspect a heap dump if operationally safe; identify dominators/retaining paths; fix retention, capacity and cleanup. Check container memory limits too.

**Q: What is stop-the-world?** **A:** Periods where application threads pause for certain GC phases. Collector design changes pause behavior; not all collection work is necessarily fully stop-the-world.

---
# 3. Multithreading and concurrency — P0

## 3.1 Threads, executors, and real-world trade processing

**60-second answer:** "A thread is an execution path within a process. Multithreading helps when independent tasks can run concurrently, but it introduces shared-state risks and overhead. Rather than creating a thread per booking group, I would use a bounded `ExecutorService`, isolate data for each task, preserve necessary request context, collect results, handle timeouts and exceptions, and control database/downstream load. I select the pool size using benchmarks, not simply the CPU count."

| Concept | Meaning | Typical use |
|---|---|---|
| `Thread` | Low-level execution unit | Rarely manage directly in server code |
| `Runnable` | Task with no result | Fire-and-join work |
| `Callable<T>` | Task returning result / throwing exception | Book a group and return status |
| `ExecutorService` | Manage task execution, futures and shutdown | Bounded booking workers |
| `Future<T>` | Retrieve outcome, cancellation request | Wait for each booking result |
| `CompletableFuture<T>` | Compose non-blocking callbacks/async workflows | Job orchestration and aggregation |
| `BlockingQueue` | Backpressure between producers/consumers | Worker pipelines |
| `Semaphore` | Limit concurrent access to resource | Rate-limit downstream TRAC calls |
| `CountDownLatch` | Wait for a fixed number of completions | Integration test coordination |

### Bounded pool example [ILLUSTRATIVE]

```java
import java.util.*;
import java.util.concurrent.*;

record BookingGroup(String id) {}
record BookingResult(String id, boolean succeeded) {}

final class BookingBatchProcessor {
    private final ExecutorService pool = new ThreadPoolExecutor(
        5, 10, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(200),
        new ThreadPoolExecutor.CallerRunsPolicy()); // Illustrative policy; not always right for REST

    List<BookingResult> process(List<BookingGroup> groups) throws Exception {
        List<Future<BookingResult>> futures = new ArrayList<>();
        for (BookingGroup group : groups) {
            futures.add(pool.submit(() -> bookIndependently(group)));
        }
        List<BookingResult> results = new ArrayList<>();
        for (Future<BookingResult> future : futures) {
            results.add(future.get(30, TimeUnit.SECONDS)); // per-future wait, NOT a global 30s deadline
        }
        return results;
    }

    private BookingResult bookIndependently(BookingGroup group) {
        // New per-task business context; no mutation of shared request/DTO
        return new BookingResult(group.id(), true);
    }
    void shutdown() { pool.shutdown(); }
}
```

**Important design critique:** This sample shows bounded workers but may block the HTTP request via `CallerRunsPolicy`; it also has a per-future rather than overall batch timeout, does not cancel remaining tasks on failure, and depends on a service lifecycle that shuts the executor down. In a production interview, say you would choose the rejection policy deliberately (often reject with a 429/503 or persist queued work) and enforce a **single batch deadline**.

## 3.2 `CompletableFuture` essentials

```java
import java.util.concurrent.*;

ExecutorService executor = Executors.newFixedThreadPool(8);
try {
    CompletableFuture<String> first = CompletableFuture.supplyAsync(() -> "group-A", executor);
    CompletableFuture<String> second = CompletableFuture.supplyAsync(() -> "group-B", executor);

    CompletableFuture<String> combined = first.thenCombine(second, (a, b) -> a + "," + b);
    String result = combined.orTimeout(10, TimeUnit.SECONDS).join();
    System.out.println(result);
} finally {
    executor.shutdown();
}
```

- `supplyAsync`: returns value; `runAsync`: void work.
- `thenApply`: transform a result; `thenCompose`: flatten dependent async stage; `thenCombine`: combine two independent stages.
- `allOf`: wait for all futures; *does not itself expose individual typed results*.
- `exceptionally`, `handle`, `whenComplete`: recover, transform errors or observe completion (semantics differ).
- `join()` throws unchecked `CompletionException`; `get()` exposes checked exceptions.
- `orTimeout()` completes a future exceptionally; **does not guarantee interrupting or stopping underlying work**.

**Q: `Future` vs `CompletableFuture`?** **A:** `Future` mainly blocks or polls for completion; CompletableFuture supports dependent operations, error handling and composition without manually waiting after every step.

**Q: Can I add `@Transactional` to a method and execute parts in worker threads?** **A:** Spring transactions are usually bound to the calling thread. They don't automatically propagate to newly created threads. Each worker's independent DB work needs an appropriate transaction boundary; don't assume the parent transaction covers them.

## 3.3 Thread safety, `volatile`, `synchronized`, locks, atomics

| Mechanism | Guarantees | Does **not** guarantee |
|---|---|---|
| `volatile` | Visibility and ordering guarantees for reads/writes | Atomic increment or compound check-then-act |
| `synchronized` | Mutual exclusion + happens-before visibility | High throughput under long critical sections |
| `ReentrantLock` | Explicit lock/unlock, `tryLock`, timed acquisition | Automatic release unless you use `finally` |
| `AtomicInteger` | Atomic individual operations via CAS mechanisms | Multi-field business invariants |
| `ConcurrentHashMap` | Thread-safe map operations + atomic utilities | Atomic database booking or entire workflow |
| `ThreadLocal` | Per-thread value isolation | Automatic propagation across pools or auto-cleanup |

```java
import java.util.concurrent.atomic.AtomicInteger;

final class Counters {
    private final AtomicInteger completed = new AtomicInteger();
    void markComplete() { completed.incrementAndGet(); }
    int completedCount() { return completed.get(); }
}
```

**Q: Why doesn't `volatile int count` make `count++` safe?** **A:** `count++` is a read–modify–write operation, not one atomic operation; two threads can read the same old value. Use `AtomicInteger.incrementAndGet()` or synchronization.

**Q: What is happens-before?** **A:** Java's memory model ordering relationship: actions before a monitor unlock happen-before actions after a subsequent lock of that monitor; volatile writes happen-before later volatile reads; future submission/completion also establishes defined visibility relationships.

**Q: `synchronized` vs `ReentrantLock`?** **A:** Both can enforce mutual exclusion. ReentrantLock adds timed/interruptible acquisition and other capabilities but requires explicit `unlock()` in `finally`.

**Q: Can two threads modify different entries in ConcurrentHashMap?** **A:** Yes, generally with high concurrency; but `map.get(key)` then `map.put(key)` is not a safe atomic compound operation. Use `compute`, `putIfAbsent`, or external synchronization for cross-entry invariants.

## 3.4 Race conditions, deadlocks, starvation, livelock

- **Race condition:** Correctness changes depending on interleaving. Example: two workers both check "trade absent" and both insert.
- **Deadlock:** A waits on lock held by B, while B waits on lock held by A.
- **Starvation:** A task rarely gets resources because other tasks dominate.
- **Livelock:** Tasks stay active changing state but make no progress.

**Prevent deadlock:** consistent lock ordering, shorter lock duration, avoid nested distributed locks, lock timeouts, analyze thread dumps. For DB deadlocks use short transactions, stable row order, appropriate indexes, bounded retries with jitter on recognized transient errors.

### The key trading correctness question: "How do you avoid duplicate trades?"

**60-second answer:** "Application-level synchronization helps only within one JVM. For correctness across multiple service instances or retries, I need a durable idempotency key or unique business identifier protected by a database unique constraint. The service should atomically reserve the key and record processing state, then return or reconcile the existing result for duplicate requests. If booking and publishing span systems, I'd use transactional outbox and idempotent consumers to avoid inconsistency. A failed or timed-out request must not blindly create a second booking."

**Illustrative state machine:** `RECEIVED → IN_PROGRESS → COMPLETED` or `FAILED_RETRYABLE` / `FAILED_PERMANENT`; keyed by unique external request ID (or a canonical business key agreed with upstream).

**Pseudo-SQL (adapt for MS SQL Server):**

```sql
CREATE TABLE booking_request (
    idempotency_key VARCHAR(128) NOT NULL PRIMARY KEY,
    status VARCHAR(24) NOT NULL,
    trade_id VARCHAR(64) NULL,
    created_at DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
-- Attempt INSERT for a new key in a transaction.
-- If unique-key conflict: read existing row and return existing outcome
-- or 202/409 depending on request status and API contract.
```

**Do not promise "exactly once" just because you use a lock.** A node can crash after DB commit but before HTTP response. Retried requests must discover the committed result. Also decide what to do if the same key arrives with *different payload*: store/compare a payload hash and reject mismatches.

## 3.5 How to size a thread pool

For a CPU-bound workload, begin around available CPU capacity; for I/O-heavy work, additional threads can hide waiting, but downstream/database connection limits cap useful concurrency. A frequently quoted rough heuristic is `threads ≈ CPU cores × (1 + wait time / compute time)`; **benchmark rather than treat it as a formula guaranteeing optimality**. Observe queue length, CPU, DB connections, remote service rate limits and P95/P99 latency.

**[YOUR WORK]** Your parallel booking-group design used **deep-cloned per-group contexts**, propagated `EpicUserSession` and `RequestSession`, separated parallel per-group steps from sequential final steps, and used a **configurable pool (1–20; default 10)** with **360-second timeout protection**. In testing, **1,710 groups** improved from about **439.182 s to 186.697 s** with **15 threads (~57.49%)**. Benchmarking suggested benefits tapered with excessive threads. Distinguish measured end-to-end improvements from general JVM theory.

**Q: Why deep clone each booking-group context?** **A:** To avoid cross-group modifications to shared mutable request/workflow objects. Cloning does have memory/CPU cost, so I would assess object graph size and consider immutable per-group context where practical.

**Q: What about session propagation?** **A:** Thread-bound context such as security or request sessions typically does not move automatically into executor workers. Capture only required context, bind it in each task, and clear it in `finally` to prevent one user's context leaking into another task on a reused thread.

**Q: What if a task fails after some other groups succeed?** **A:** Define partial-success semantics first. Capture each group's outcome, cancel pending work where safe, reconcile committed bookings and return a truthful aggregate response. Never promise automatic rollback of independent transactions.

**Q: Why not just parallel streams?** **A:** Parallel streams use a shared pool by default and give less explicit control over queueing, naming, saturation, request context and lifecycle. For production booking, a dedicated bounded executor is easier to operate.

**Concurrency practice questions:** Explain thread states; daemon vs user thread; `wait()` vs `sleep()`; `notify` vs `notifyAll`; `Callable` vs `Runnable`; interruption; `shutdown` vs `shutdownNow`; bounded vs unbounded queues; task cancellation; `ThreadLocal` leaks; why ThreadPoolExecutor queue/rejection design matters.

---

# 4. Spring Framework and Spring Boot — P0

## 4.1 IoC, DI and bean lifecycle

**60-second answer:** "Spring's IoC container creates and wires managed objects called beans. Dependency Injection is how collaborators are supplied rather than being constructed inside a class. I prefer constructor injection because it makes required dependencies explicit, supports immutability and makes unit tests straightforward. The container also provides bean scopes, lifecycle callbacks and infrastructure features such as transactions and auto-configuration."

```java
import org.springframework.stereotype.Service;

interface TradeRepository { void save(String tradeId); }

@Service
public class TradeService {
    private final TradeRepository repository;
    public TradeService(TradeRepository repository) {
        this.repository = repository;
    }
    public void book(String tradeId) { repository.save(tradeId); }
}
```

- `@Component`: generic bean; `@Service`: service stereotype; `@Repository`: persistence stereotype with exception translation in suitable arrangements; `@Controller` / `@RestController`: web layer.
- `@Bean`: declares a bean via method in configuration class, often for third-party classes.
- `@Primary` / `@Qualifier`: resolve multiple matching candidates.
- Lifecycle: instantiate → inject dependencies → `BeanPostProcessor`/initialization → use → destruction for managed scope. `@PostConstruct` / `@PreDestroy` are hooks (Jakarta imports for modern Spring).

**Q: Circular dependencies?** **A:** Constructor-cycle A→B→A cannot be resolved cleanly. Usually this means responsibilities should be refactored, not bypassed with field injection.

## 4.2 Bean scopes (very likely Spring question)

| Scope | Instance lifecycle | Common usage | Key pitfall |
|---|---|---|---|
| `singleton` (default) | One bean instance per Spring container/definition | Stateless service, repository | Mutable shared fields must be thread-safe |
| `prototype` | New instance on each explicit retrieval/injection resolution | Stateful short-lived object | Injecting into singleton ordinarily creates only one prototype at singleton construction |
| `request` | One per HTTP request | Request-specific state | Requires web-aware context |
| `session` | One per HTTP session | Session data | State/memory and horizontal scaling |
| `application` | One per `ServletContext` | Application web context | Not equivalent to cross-cluster singleton |
| `websocket` | One per WebSocket session | WebSocket state | Web-aware infrastructure required |

```java
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@Component
@Scope("prototype")
class ValidationContext { /* request-specific mutable state */ }

@Service
class ValidationRunner {
    private final ObjectProvider<ValidationContext> contexts;
    ValidationRunner(ObjectProvider<ValidationContext> contexts) { this.contexts = contexts; }
    void run() { ValidationContext fresh = contexts.getObject(); /* use fresh context */ }
}
```

**Q: If I inject a prototype into a singleton, do I get a new prototype for each method call?** **A:** Not with ordinary direct injection; dependency resolution happens when the singleton is created. Use `ObjectProvider`, a scoped proxy, or lookup injection for fresh instances when necessary.

**Q: Are singleton beans thread-safe?** **A:** No automatic guarantee. Stateless services usually are safe if dependencies are safe; storing per-request `tradeId` in a singleton instance field is dangerous.

**Q: Does Spring run `@PreDestroy` for prototypes?** **A:** Not normally — the container instantiates and injects a prototype but does not manage its full destruction lifecycle.

## 4.3 How Spring Boot starts and auto-configuration

`@SpringBootApplication` combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

**Startup path:** `SpringApplication.run()` builds application context → reads environment/profiles → scans/registers beans and auto-configuration → creates and wires eligible beans → runs lifecycle initialization → starts embedded server for web apps → invokes startup runners.

**Q: How does auto-configuration work?** **A:** Boot registers configuration conditionally using classpath presence, defined properties and existing beans. When web/data libraries appear, Boot can configure appropriate defaults; explicitly defined beans can override many auto-configured ones. It is conditional, not magic.

**Q: `@Configuration` vs `@Component`?** **A:** Both can register bean definitions. `@Configuration` is intended for configuration classes and allows full processing of `@Bean` methods (including interception under default proxyBeanMethods behavior); a generic component isn't necessarily equivalent for inter-bean method calls.

## 4.4 `@Transactional`: practical behavior and traps

**60-second answer:** "`@Transactional` marks a logical transaction boundary around database operations invoked through Spring's transactional proxy. On success it commits; by default runtime exceptions and errors cause rollback, while checked exceptions need configuration when rollback is desired. Important pitfalls are self-invocation bypassing the proxy, long transactions with external calls, and assuming transaction state carries to asynchronous worker threads. I use it around a short unit of database consistency and complement it with idempotency for distributed retries."

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
class BookingWriter {
    private final BookingRepository repository;
    BookingWriter(BookingRepository repository) { this.repository = repository; }

    @Transactional
    public String createTrade(String key, String payloadHash) {
        // Illustrative: use database uniqueness on key and atomic state changes.
        // Do not assume this alone solves duplicate remote calls or event publication.
        return repository.insertOnce(key, payloadHash);
    }
}
interface BookingRepository { String insertOnce(String key, String payloadHash); }
```

| Propagation | Meaning / use |
|---|---|
| `REQUIRED` (default) | Join existing transaction or create one |
| `REQUIRES_NEW` | Suspend existing, start separate transaction (e.g., independent audit; mind separate connection needs) |
| `SUPPORTS` | Join if there is one; otherwise run non-transactionally |
| `MANDATORY` | Must already have transaction |
| `NOT_SUPPORTED` | Suspend an existing transaction, run without one |
| `NEVER` | Error if there is a transaction |
| `NESTED` | Savepoint-like nested behavior when supported by transaction manager |

**Common traps:**

1. **Self-invocation:** `this.innerTransactionalMethod()` bypasses normal proxy interception.
2. **Exception swallowed:** Catching error without marking rollback can allow commit.
3. **Checked exception:** Default rollback typically does not include it; use `rollbackFor = ...` if required.
4. **Thread boundary:** `@Async` / executor work starts separately; transactions don't flow automatically to worker threads.
5. **Distributed transaction:** Local `@Transactional` doesn't atomically commit a database and Kafka/outside HTTP API.
6. **Long transaction:** Locks and connections held while waiting on network may cause contention.
7. **Visibility:** In proxy mode, method visibility and proxy type matter; prefer external calls to public service methods.

**Q: If an inner `REQUIRED` method throws a runtime exception but the outer method catches it, will it commit?** **A:** If the inner transactional interceptor marks the shared transaction rollback-only, the outer "successful" return can still result in rollback and potentially `UnexpectedRollbackException`. This depends on actual proxy/transaction boundaries.

**Q: How to do database+Kafka reliably?** **A:** Write trade change and an **outbox event** in the same DB transaction. A separate publisher reliably sends the outbox event. Consumers must be idempotent because duplicates remain possible.

## 4.5 Spring configuration and exceptions

- **`@Value` vs `@ConfigurationProperties`:** Fine-grained single value vs typed grouping/validation of related config.
- **Profiles:** `spring.profiles.active` selects environment-specific beans/properties; don't embed production secrets in source control.
- **Spring exception handling:** `@RestControllerAdvice` + `@ExceptionHandler` maps domain errors to consistent HTTP responses.
- **Validation:** `@Valid`, `@NotNull`, `@NotBlank`, `@Size` for boundary input; these do not replace business validation and database constraints.
- **`@Async`:** Enables method execution through async infrastructure when invoked via the managed proxy; choose bounded executor and make error/context propagation explicit.

**Q: What happens if multiple beans match an interface injection?** **A:** Resolve using `@Qualifier`, `@Primary`, or inject a collection/map of beans where appropriate.

**Q: `@Component` vs `@Bean`?** **A:** Component scanning detects annotated classes; `@Bean` explicitly returns an instance managed by Spring, especially useful with third-party classes or nontrivial construction.

**[YOUR WORK]** You contributed to a Spring Boot **2.6.8 → 2.7.5** upgrade and H2/SQL Server driver compatibility work across multiple services. Real talking points include H2 schema syntax differences and the change to SQL Server driver's encryption defaults. **Security nuance:** `encrypt=true;trustServerCertificate=true` enables encrypted transport but skips server certificate validation; production-grade trust should generally use certificate verification and a trusted certificate chain, according to organizational policy. Don't describe the trust-bypass flag as full identity verification.

---
# 5. REST, microservices and messaging — P0

## 5.1 REST and HTTP design

**60-second answer:** "A REST-style service models resources through HTTP URLs and methods, uses appropriate status codes, and separates transport DTOs from domain logic. For a trade API I validate the request, use correlation and idempotency identifiers, persist the result consistently and return a stable response contract. A timeout must not lead the caller to assume the trade was never booked."

| Method | Meaning | Idempotent by HTTP semantics? | Example |
|---|---|---|---|
| GET | Read resource | Yes | `GET /trades/T123` |
| POST | Create / process | Not inherently | `POST /trades` with `Idempotency-Key` |
| PUT | Replace/upsert a resource at known URI | Yes | `PUT /trade-configs/key` |
| PATCH | Partial update | Not inherently | `PATCH /trades/T123` status change |
| DELETE | Delete resource | Yes at semantic state level | `DELETE /jobs/J123` (only if supported) |

**Status codes to know:** `200 OK` read/update; `201 Created` new resource; `202 Accepted` accepted for async processing; `204 No Content` no response body; `400 Bad Request` syntactic validation; `401 Unauthorized` missing/invalid authentication; `403 Forbidden` authenticated but not permitted; `404 Not Found`; `409 Conflict` state/idempotency conflict; `422 Unprocessable Content` well-formed business validation failure (contract dependent); `429 Too Many Requests`; `500` internal unexpected error; `502/503/504` gateway, unavailable, timeout patterns.

### API skeleton [ILLUSTRATIVE, Spring Boot 3 / Jakarta]

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

record CreateTradeRequest(@NotBlank String upstreamTradeId, @NotBlank String productType) {}
record CreateTradeResponse(String tradeId, String status) {}

@RestController
@RequestMapping("/trades")
class TradeController {
    private final BookingApplicationService service;
    TradeController(BookingApplicationService service) { this.service = service; }

    @PostMapping
    ResponseEntity<CreateTradeResponse> create(
            @RequestHeader("Idempotency-Key") String key,
            @Valid @RequestBody CreateTradeRequest body) {
        CreateTradeResponse result = service.create(key, body);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
        // Real API: repeat same key may return 200 or 201 consistently per contract.
    }
}
interface BookingApplicationService {
    CreateTradeResponse create(String key, CreateTradeRequest body);
}
```

**Q: `@RestController` vs `@Controller`?** **A:** `@RestController` is effectively `@Controller` + `@ResponseBody`, so response data is serialized rather than interpreted as a view name.

**Q: `@PathVariable` vs `@RequestParam` vs `@RequestBody`?** **A:** Resource path element, query value, serialized request payload.

**Q: How do you version APIs?** **A:** Prefer backward-compatible additive changes; when behavior/contracts break, introduce a documented versioning scheme (URI/header/media type), support migration, and deprecate safely.

**Q: How to make POST idempotent?** **A:** Agree on key semantics, persist key/payload hash and outcome with a unique constraint, return/reconcile the original result on retries, and define concurrent `IN_PROGRESS` behavior.

**Q: What is pagination?** **A:** Return bounded pages for large result sets. Offset pagination is simple but can be unstable/slow at scale; cursor/keyset pagination is useful for frequently changing or large ordered datasets.

## 5.2 Microservices vs monolith and service boundaries

**60-second answer:** "Microservices divide an application into independently deployable services aligned to business capabilities, but introduce network latency, distributed failures, contract evolution and observability complexity. I would choose service boundaries based on ownership, data consistency and release independence, not create a service for every entity. A modular monolith can be better until scale and organizational needs justify distribution."

**Trade-offs:**

- **Microservices benefit:** Independent deployment/scaling and team ownership; easier isolation of some failure domains.
- **Cost:** Network calls, eventual consistency, duplicate messages, security, data ownership, monitoring, test complexity.
- **Monolith benefit:** Simpler deployments and local transactions; strong consistency easier within one database.
- **Cost:** Tight coupling unless modular, larger blast radius, harder independent scaling at large size.

**Q: How should two services communicate?** **A:** Synchronous REST/gRPC when caller needs immediate response, asynchronous messaging for decoupling, buffering and eventual consistency. Choose based on latency, delivery semantics, data consistency and recovery needs.

**Q: What is service discovery?** **A:** Clients find service endpoints via orchestrator, DNS or registry rather than hardcoded host:port values.

**Q: What is a circuit breaker?** **A:** Tracks failures and temporarily stops sending requests to an unhealthy dependency, allowing recovery and preventing cascading load. It should be combined with appropriate timeouts and safe fallbacks.

## 5.3 Apache Kafka, Solace and delivery semantics

| Concept | Concise explanation |
|---|---|
| Topic / partition | Kafka topic stores records in partitions; order guaranteed **within** a partition, not across all partitions |
| Key | Drives partition selection; key by trade ID when per-trade ordering matters |
| Consumer group | Multiple instances share partitions; each partition is handled by at most one group member at a time (under normal assignment) |
| Offset | Position within partition; commit strategy determines replay behavior |
| Producer retries | May resend; idempotent producer avoids some broker-side duplicates, not arbitrary application duplication |
| At-most-once | May lose records, typically no duplicates |
| At-least-once | Retry favors no loss but can deliver duplicates |
| Exactly-once | Requires narrowly defined broker/transactional guarantees; not automatic across external DB/API side effects |
| DLQ / DLT | Persist poison events for investigation / controlled replay |
| Schema evolution | Compatible message changes, e.g., Avro/Protobuf conventions |

**60-second answer:** "Kafka is a partitioned event log useful for decoupling services and replayable consumption. In an at-least-once consumer, a crash after business commit but before offset commit causes redelivery, so consumers should process events idempotently. I would choose a stable trade identifier as the partition key if updates for a trade require order. For cross-database/event consistency I favor an outbox pattern."

**Q: Kafka vs Solace?** **A:** Both move messages, but operational models and routing/delivery features differ. Kafka is designed around durable, replayable partitioned logs; Solace supports enterprise pub/sub, queues and routing patterns. Compare requirements (ordering, replay, fan-out, protocol, operations) rather than declaring one universally better.

**Q: What if a consumer crashes after saving a trade but before committing the offset?** **A:** The message is redelivered. Guard with a durable message/trade idempotency key and only acknowledge/commit after safe processing.

**Q: How do you handle a poison message?** **A:** Distinguish transient vs permanent failures; use bounded retries with backoff, route unrecoverable cases to DLQ with context, alert and build a safe replay process.

**Q: How do you preserve order?** **A:** Partition by the required ordering key, avoid side-effects that reorder asynchronously, and design for rebalances/retries. Global order across partitions is not guaranteed.

**Q: What is the transactional outbox pattern?** **A:** Commit the business state and an outbox row in one local DB transaction; a publisher sends the event later. Event consumers must still be idempotent.

## 5.4 Retry, timeout, circuit breaker, bulkhead, rate limiting

**60-second answer:** "Every remote dependency needs a timeout. Retry only failures likely to be transient, cap attempts and use exponential backoff with jitter. For non-idempotent calls, retries must be protected by idempotency. A circuit breaker reduces repeated pressure on an unhealthy dependency, while a bulkhead limits resource consumption so one dependency does not exhaust the entire service. I monitor retries, P95 latency, failure rate and queue depth."

**Failure rules:**

- **Timeout** bounds how long a call waits; distinguish connect, read and overall end-to-end deadline.
- **Retries** multiply load; do not retry a permanent business validation error.
- **Exponential backoff + jitter** avoids synchronized retry storms.
- **Circuit breaker** fails fast after repeated dependency failure, with half-open probing during recovery.
- **Bulkhead** isolates thread/connection pool capacity per dependency.
- **Rate limit** protects downstream and can return `429`/backpressure.
- **Fallback** must be **business-approved**. For financial controls, an unverified "allow booking" fallback may be a compliance breach; assess business rules and audit requirements.

**Q: What if upstream timed out but backend committed trade?** **A:** A retry with the same idempotency key should return the existing trade rather than create a second. A status/reconciliation API helps callers resolve uncertain outcomes.

**Q: How to prevent cascading failures?** **A:** Bound queues/concurrency, set deadlines, apply bulkheads and breakers, avoid endless retries, and degrade only where correctness allows it.

**[YOUR WORK]** Your Delta One STP work involved GCAM → Delta One → Atlas mapping, L1/L2 acknowledgements and correlation IDs; discuss **traceability, schema validation and duplicate detection**. For CCLT/TRAC, your notes describe a **soft-block** override design, feature toggles and continued operation during integration errors; present this as a **business-approved control policy with compensating checks** rather than a universal recommendation for regulatory dependencies.

---

# 6. SQL, transactions and performance — P0

## 6.1 Joins, indexes, query plans, tuning

**60-second answer:** "I start by measuring the slow query and reading its actual execution plan. I look for scans on large tables, missing/ineffective indexes, expensive sorts/joins, incorrect cardinality estimates, blocking and unnecessary data transfer. I then validate a change with representative data, logical reads, elapsed time and query-plan evidence. I don't create indexes indiscriminately because they increase write cost and storage."

| SQL concept | What to say |
|---|---|
| `INNER JOIN` | Keep matching rows on both sides |
| `LEFT JOIN` | Keep all left rows and matching right rows; unmatched right fields null |
| `UNION` vs `UNION ALL` | UNION removes duplicates (can require sort/hash); ALL retains duplicates, often cheaper |
| `GROUP BY` vs `HAVING` | Grouping and aggregate filtering after grouping |
| Index | Data structure speeding lookups, usually B-tree/B+tree style for rowstore indexes |
| Composite index | Key order matters; equality, ranges, sort and covering behavior drive usefulness |
| Covering index | Contains needed query columns via key/INCLUDE, reducing lookups |
| Scan vs seek | Scan reads broader range; seek navigates to qualifying portion; neither is universally bad |
| Statistics | Help optimizer estimate cardinality and choose plan |
| SARGable predicate | Enables efficient index access; avoid wrapping indexed column in functions unnecessarily |

```sql
-- Example: common recent-trade retrieval (adapt for your actual schema)
SELECT trade_id, product_type, status, booking_date
FROM dbo.trades
WHERE counterparty_id = @counterparty
  AND booking_date >= @fromDate
ORDER BY booking_date DESC;

-- Illustrative SQL Server covering index; validate writes/selectivity/plan first.
CREATE INDEX IX_trades_counterparty_bookingdate
ON dbo.trades(counterparty_id, booking_date DESC)
INCLUDE (trade_id, product_type, status);
```

**Q: Why might an index not be used?** **A:** Very low selectivity, small table, stale statistics, expression on indexed column, implicit conversions, non-leading column of composite index, optimizer cost trade-offs.

**Q: What is the N+1 query problem?** **A:** Loading N parent entities then triggering one child query per parent; solve via suitable join fetch, entity graphs, batching or DTO projections, watching for cartesian explosion.

**Q: `DELETE` vs `TRUNCATE` vs `DROP`?** **A:** DELETE removes selected rows and can have WHERE; TRUNCATE removes all rows using different logging/allocation behavior and restrictions; DROP removes the object. Transaction/rollback capabilities depend on DBMS; SQL Server supports transactional rollback for suitable DDL/truncate scenarios. Always check referential constraints and operational approvals.

## 6.2 ACID, isolation and locking

| ACID | Definition | Trade impact |
|---|---|---|
| Atomicity | All-or-nothing unit of work | Trade status and its DB detail rows committed together |
| Consistency | DB constraints preserve invariants across committed transactions | Uniqueness / foreign keys / check constraints |
| Isolation | Concurrent transactions behave according to isolation contract | Prevent invalid parallel updates |
| Durability | Committed data survives appropriate failure scenarios | Booked trades not lost after process crash |

| Isolation level | Important idea |
|---|---|
| READ UNCOMMITTED | Allows dirty reads under applicable DB behavior |
| READ COMMITTED | Prevents dirty reads; nonrepeatable reads can occur; SQL Server behavior also depends on `READ_COMMITTED_SNAPSHOT` |
| REPEATABLE READ | Retains relevant read stability; phantoms still possible in conventional models |
| SERIALIZABLE | Strongest standard isolation; restricts phantom anomalies, at cost of contention |
| SNAPSHOT / MVCC modes | Read consistent versions; behavior differs across databases and specific isolation mode |

**Key concurrency anomalies:** dirty read, non-repeatable read, phantom read, lost update, write skew (depending on isolation).

**Optimistic locking:** version column; update where `id = ? AND version = ?`; if no row updated, detect conflict and retry/reject. Useful when contention is modest. **Pessimistic locking:** lock rows before mutation; useful in controlled contentious flows, but increases blocking/deadlock risk.

**Q: Is READ COMMITTED enough for duplicate prevention?** **A:** Not by itself. A pre-check `SELECT` then `INSERT` races across concurrent transactions. Use a DB unique constraint or appropriate atomic locking/insert semantics.

**Q: Difference between deadlock and blocking?** **A:** Blocking is waiting on a lock; deadlock is a cycle of mutually waiting transactions requiring one to be aborted (or some resolution).

**Q: How do you prevent lost updates?** **A:** Version checks, atomic SQL updates, locking as required, and isolation aligned to invariants.

## 6.3 Archive millions of trades without downtime

**60-second answer:** "For a large archival I first define retention and retrieval requirements with business/compliance. Then I identify safe candidates, archive in controlled batches with checkpoints and reconciliation, protect data integrity through transactional or recoverable movement, and keep an unarchive path. I monitor log growth, locks and query plans, validate row counts/checksums and only delete source records after verifying the destination."

**[YOUR WORK]** Listed trade archive covered **18.5 million trades (~340 GB) across six releases**, with older than 18 months / book-and-instrument targeting according to your contribution notes. Retrieval improved from about **45 seconds to 8 seconds** (82%), and backup/recovery window improved from ~12 hours to ~5 hours. You also implemented an **unarchival mechanism** and pre/post validation.

**Safe illustrative batching approach:**

1. Agree retention criteria (e.g., before cutoff date), compliance hold conditions and archive schema/indexes.
2. Pick a **stable** batch of primary keys and record checkpoint/batch ID.
3. In a transaction or recoverable mechanism, copy/verify target and remove source where safe; avoid making a long multi-million-row transaction.
4. Record counts and checksums; monitor log, lock duration, replication/backup and execution times.
5. Repeat with throttling during quieter periods; make each batch restartable and idempotent.
6. Reconcile complete move and test unarchive path before decommissioning old access patterns.

**Q: The archival fails after moving 10 million records. How recover?** **A:** I wouldn't restart from scratch. I'd read durable checkpoints and reconcile source/target keys for the last incomplete batch. Rerun idempotently with uniqueness protection, investigate failed batch and ensure no source deletion occurred without confirmed archive copy.

**Q: Why did query latency improve?** **A:** Less active table/index volume reduces data scanned and cache pressure and can improve optimizer plans; I would validate with execution plans/reads because a smaller table alone doesn't guarantee a specific improvement.

**Q: Why not just add an index?** **A:** Index tuning may help some queries, but retaining unnecessary data affects backups, maintenance, storage and more general query performance. Archiving and indexing address different aspects of the problem.

---

# 7. Caching, distributed systems and system design — P0

## 7.1 Cache patterns and correct key design

**60-second answer:** "Caching reduces repeated expensive lookups, especially remote dependencies. In a cache-aside approach, we check cache first, call the authoritative service on a miss, then populate the cache. I choose keys based on *all inputs that determine the response*, set TTL based on correctness requirements, monitor hit rate and failures, and plan for stale data and cache outages. A distributed cache is useful when instances need to share results; local caches are simpler but not shared across nodes."

| Pattern | Workflow | Trade-off |
|---|---|---|
| Cache-aside | Application fetches on miss and writes cache | Simple; first miss slow; stale possible |
| Read-through | Cache fetches from source via loader | Centralized loader; more coupling to cache |
| Write-through | Write data to cache/source in controlled path | Potential write latency; consistency design |
| Write-behind | Cache buffers and writes later | Fast writes but greater loss/durability risk |
| TTL | Expire by age | Limits staleness but creates cold misses |
| Explicit invalidation | Evict on source change | More correctness but event coordination |
| Versioned key | Include policy/schema version | Safer updates, extra management |

**[YOUR WORK]** TRAC Nexus lookup originally made cross-region calls to London of about **956 ms**. You added **Oracle Coherence** distributed `NexusCache`, keyed by **Trader BRID + Salesperson BRID**, with **24-hour expiry**, an enable/disable **feature toggle** and logging. **Cache hits** took about **2 ms**, and a 20-booking test improved from ~70s to ~19s. This is **cache-aside-like** behavior.

**Be ready for the strongest architecture probe:** "You cache a regulatory determination for 24 hours. What if the regulatory mapping changes ten minutes later?"

**Model answer:** "Staleness is the main trade-off of this implementation. The business needs to decide the acceptable freshness window. If a determination depends on more than the two key fields, we must ensure the cache key captures those inputs or that the remote result is valid for reuse. For stronger freshness we can reduce TTL, invalidate affected entries on policy changes, or version the cache by rule set. We would validate the chosen mitigation with control owners, as hit rate must not take priority over regulatory correctness."

**Q: What is cache stampede?** **A:** Many simultaneous requests miss for the same key and all call the source; use per-key request coalescing, locking where safe, jittered expiry, or controlled warm-up.

**Q: What is cache penetration?** **A:** Repeated requests for nonexistent keys bypass cache; use short-lived negative caching if business-safe, validation and rate limits.

**Q: What is cache avalanche?** **A:** Many entries expire together causing load spike; jitter TTLs / spread expiry and protect source capacity.

**Q: What if Coherence fails?** **A:** Treat it as an optimization failure according to approved control policy; attempt authoritative source or safe degrade, but do not silently assume stale/default regulatory answers are valid. Monitor failures, latency and circuit-breaker behavior.

## 7.2 Consistency, CAP, distributed locks, availability

- **Strong consistency:** A read reflects the latest relevant committed write under defined guarantees; coordination can raise latency.
- **Eventual consistency:** Replicas/consumers converge when no further updates occur; design for intermediate discrepancies.
- **CAP in a network partition:** A distributed system cannot guarantee both availability for all requests and strongly consistent responses under partition. The theorem does not mean every ordinary system must simply choose “C or A” in all circumstances.
- **Distributed locking:** Can coordinate across nodes, but needs expiry/fencing and careful failure semantics; for trade creation a DB unique constraint is usually a stronger final guarantee than a best-effort distributed lock.
- **Idempotency:** Same logical request can be repeated without changing final effect beyond first successful processing. It does not necessarily mean the exact same response status every time; define contract.
- **Sagas:** Coordinate multi-service business transactions using compensating actions or orchestration. Compensation is not the same as an ACID rollback.

**Q: How would you identify an already booked trade across two pods?** **A:** Durable, uniquely constrained external trade ID/idempotency key in shared data store; atomic state transition and request hash. In-memory `synchronized` only protects one JVM.

**Q: How to design high availability?** **A:** Redundant instances across failure domains, health checks, durable stores, failover, bounded retries, graceful degradation, observability, load tests and tested recovery. Data consistency requirements determine permissible failover behavior.

## 7.3 System design: reliable trade-booking service [ILLUSTRATIVE]

**Requirements to clarify first:** 500 TPS or 5 TPS? Latency SLO? Allowed duplicates? Ordering by trade ID? Partial success? Regulatory validations? Synchronous response required? Multi-tenant? RTO/RPO? Data ownership?

```text
Upstream API / message
        |
        v
API gateway/auth / validation / request ID
        |
        v
Trade Booking Service -----> Control/Reference Data Service (timeout, retries, policy)
        |
        v
DB transaction: idempotency key + trade state + outbox event
        |
        v
HTTP result / booking ID
        |
        +--> Outbox Publisher --> Kafka/Solace --> Idempotent downstream consumers
        |
        +--> Metrics / logs / distributed traces / alerting
```

**Step-by-step 2-minute answer:**

1. Validate request and authenticate/authorize; reject missing mandatory fields.
2. Derive upstream idempotency key and request hash; claim key with unique constraint.
3. Record processing state and apply required booking/eligibility rules.
4. Persist trade + outbox atomically if possible; return committed result or tracked status.
5. Publish downstream asynchronously; consumers dedupe and maintain per-trade ordering where needed.
6. Handle timeouts as uncertain outcomes; expose `GET /trades/by-request/{id}` or equivalent.
7. Monitor success/failure, P95/P99, queues, DB pool, retry and reconciliation lag.

**Failure questions:**

| Failure point | What do you do? |
|---|---|
| Before DB insert | Safe retry with same key |
| After insert, before HTTP response | Return/reconcile original result from persistent key |
| After trade commit, before event publish | Outbox replay publishes eventually |
| After consumer DB update, before acknowledgement | Consumer idempotency prevents duplicate effect |
| Regulatory service unavailable | Apply explicitly approved fail-open/fail-closed/override control, audit and alert |
| Two simultaneous duplicate requests | Unique constraint ensures only one winner; loser looks up state |

## 7.4 System design: 50,000-instrument bulk expiry service

**[YOUR WORK]** A **JMX file-based bulk update** accepted CSV/TXT instrument IDs, used `CompletableFuture` + `ExecutorService` **20 threads**, **batches of 10**, and a `ConcurrentHashMap<String, JobProgress>` with UUID job IDs. It tracked counters with `AtomicInteger`; each instrument persisted individually to avoid shadow-booking conflict. It ran ~**1,000 items in 2 minutes rather than 1 hour**, handled **50,000+** and isolated per-instrument errors.

**Architecture-level improvement you can propose honestly:** In-memory job metadata is lost on process restart, and "no per-call limit" doesn't mean infinite capacity. Persist job state and checkpoints in a database, implement resumable processing, enforce quotas, stream input rather than loading unbounded files, use durable queue/worker architecture if multi-node processing is needed, and add idempotent per-instrument updates.

**Q: Why use `AtomicInteger` and `ConcurrentHashMap`?** **A:** They make counters and individual map operations safe across worker threads. They don't make the whole workflow transactional; I'd use persistent checkpoints for crash recovery if required.

**Q: How to prevent partial corruption?** **A:** Each instrument gets independent validation and persistence; failed items are recorded and replayed deliberately; reconciliation checks final expiry dates and expected counts. For cross-record invariants, need explicit transaction/coordination.

---
# 8. Production troubleshooting, testing and DevOps — P1

## 8.1 Production API latency: a repeatable diagnostic framework

**Scenario:** "A trade-booking API normally takes 200 ms; now it takes 5 seconds. What do you do?"

**Interview answer:** "I first assess customer impact and whether there was a recent release or a dependency incident. I inspect the request path with tracing and compare P50/P95/P99, error rates, throughput, saturation and thread/connection pool usage. I isolate whether the delay comes from code, database, GC, cache misses or a downstream call. If user impact is significant, I use a safe rollback, traffic shaping or feature toggle where appropriate. Once recovered, I perform root cause analysis and add prevention such as alerts, regression performance tests or tighter timeouts."

**Diagnosis by symptom:**

| Symptom | Likely areas | Evidence to obtain |
|---|---|---|
| Only one endpoint slow | Query/index, logic regression, external call | Endpoint traces + SQL plan |
| All endpoints slow, CPU high | GC, loop, compute overload | CPU profile, thread dump, GC logs |
| CPU low, requests stuck | DB/HTTP pool exhaustion, locks, timeouts | Pool metrics, thread stacks, DB blocking |
| P99 high, average normal | Tail latency, uneven nodes, GC pauses | Node-wise histogram and traces |
| Only after deployment | Dependency changes, config, rollout | Change diff and comparison of versions |
| Downstream timeout spikes | Network/service outage or overload | Dependency status, breaker/retry metrics |
| Random trade mismatches | Race, stale cache, reorder | Correlation IDs, version/state logs |

**Q: How to investigate high CPU?** **A:** Confirm process/container CPU, compare recent traffic/deployments, inspect thread dumps/CPU profiles and GC. Look for loops, serialization, excessive context switching or hot code paths; mitigate before deeper analysis where needed.

**Q: How to investigate memory leak?** **A:** Watch heap after full collections, allocation rate and retained object graph; capture a heap dump safely, inspect large maps, long-lived references and thread locals; fix root cause, not just heap size.

**Q: Thread dump vs heap dump?** **A:** Thread dump shows thread states and stack traces; heap dump shows object graph and memory retention.

**Q: How do you know whether the database is the bottleneck?** **A:** Trace duration + pool metrics + actual query timings, execution plans, waits/locks and index usage. A slow HTTP response alone is not proof of slow SQL.

## 8.2 Incidents, logging, metrics, tracing

**60-second answer:** "I approach incidents with two goals: restore safe service and identify evidence-based root cause. I use a correlation ID to follow a request through service logs, database and downstream messages. I communicate impact and mitigation clearly, preserve diagnostic evidence, and then document causes and prevention actions without blaming individuals."

- **Logs:** Structured JSON with correlation ID, trade/job ID where permitted, operation and error category; avoid secrets, credentials and unnecessary sensitive trade data.
- **Metrics:** Counters (bookings, failures), gauges (queue size, in-flight), histograms (latency), cache hit/miss, DB pool saturation, GC.
- **Traces:** Distributed spans across HTTP, DB and messaging; trace ID correlates cross-service operations.
- **SLI / SLO:** Indicator such as 99th-percentile latency or booking success rate; objective target agreed with stakeholders.
- **Alerting:** Alert on user-impacting symptoms and actionable trends, not just noisy CPU thresholds.
- **RCA / postmortem:** Timeline, impact, trigger, root/contributing causes, mitigation, detection gaps and follow-up actions.

**[YOUR WORK]** In an EDW/POG trade publishing incident, you compared Atlas/Sophis/POG timestamps and established that Atlas published promptly while the delay was further downstream. Use this as a practical example of **boundary-by-boundary evidence**, not a claim that you fixed the downstream service itself. You also investigated Reuters market-data fixing delays and loan approval workflow blocks.

**Q: What do you communicate to management during an incident?** **A:** Business impact, start time, services/products affected, current mitigation, next investigation step, risk and next update. Avoid unexplained code-level detail unless asked.

## 8.3 CI/CD, deployment, release and rollback

**Typical GitLab pipeline:**

```text
Commit / merge request
    ↓
Compile + unit tests
    ↓
Static analysis + dependency/security checks
    ↓
Integration/contract tests
    ↓
Build and version immutable artifact
    ↓
Deploy DEV → QA/UAT → approval → PROD
    ↓
Smoke tests / monitors / rollback decision
```

**60-second answer:** "CI ensures code changes are validated consistently. CD automates release delivery subject to required gates. For a financial service I would require tested artifacts, static analysis, regression coverage, controlled configuration/secrets, release approval, post-deployment smoke tests and a rollback strategy. A feature toggle is useful for isolating behavior changes but isn't always a substitute for a safe DB rollback or forward fix."

**Q: Blue-green vs canary deployment?** **A:** Blue-green switches traffic between two production environments; canary directs a small share to the new version and gradually expands. Both require compatibility and observability.

**Q: How do you handle a backward-incompatible DB migration?** **A:** Use expand-and-contract where possible: add compatible schema first, deploy app that can handle old/new shape, backfill, switch reads/writes, then remove old fields in a later controlled release.

**Q: Git rebase vs merge?** **A:** Merge preserves topology and creates merge commit; rebase rewrites branch commits onto a new base for linear history. Don't casually rebase shared published history.

**Q: What does SonarQube do?** **A:** Static analysis for code quality, code smells, certain bugs and vulnerabilities and quality gates; complements, not replaces, tests/security review. Your contribution notes also describe GitLab integration, but don't claim you personally configured company-wide infrastructure unless you did.

## 8.4 Unit tests, integration tests, contract tests and regression

| Test type | Scope | Example |
|---|---|---|
| Unit | One class/component, dependencies mocked or faked | Validate trade request required fields |
| Integration | Multiple real components | Spring repository with test database |
| Contract | Producer/consumer API or event compatibility | Delta One request schema → Atlas contract |
| End-to-end | User/workflow across systems | Request → validate → book → downstream ACK |
| Regression | Existing functionality remains working | FX/NDF/Equity Swap old test scenarios |
| Load/performance | Latency, throughput, saturation under load | 1,710 booking groups with 5/10/15 threads |
| Security | Auth, injection, data access, secrets | Unauthorized request, traversal protection |

### JUnit + Mockito example [ILLUSTRATIVE]

```java
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
import org.junit.jupiter.api.Test;

class TradeServiceTest {
    @Test
    void persistsTradeExactlyOnceForOneServiceInvocation() {
        TradeRepository repository = mock(TradeRepository.class);
        TradeService service = new TradeService(repository);
        service.book("T100");
        verify(repository, times(1)).save("T100");
    }
}
```

**Crucial caveat:** This test verifies *one method invocation* and does **not** prove exactly-once behavior under retries, concurrency, crashes or multiple pods. Test those invariants with DB integration/concurrency/recovery tests.

**Q: Mockito `@Mock` vs `@InjectMocks`?** **A:** `@Mock` creates a mock collaborator; `@InjectMocks` attempts to instantiate the class under test and inject mocks. It does not start a Spring application context.

**Q: `@SpringBootTest` vs Mockito unit test?** **A:** SpringBootTest starts application context and is slower but tests actual Spring wiring; a plain unit test isolates business logic quickly.

**Q: What test cases for an idempotent booking API?** **A:** New key, duplicate same payload, same key different payload, two concurrent identical requests, crash after commit, timeout before response, DB unique violation, downstream duplicate delivery, partial-batch failure and recovery.

**Q: How do you test multithreading?** **A:** Beyond load testing, create deterministic coordination using barriers/latches, expose race conditions, test cancellations/timeouts and verify invariants at persistence layer. Sleeping fixed durations makes brittle concurrency tests.

**[YOUR WORK]** BATHS automation covered FX Spot, NDF, Swaptions, cross-currency swaps and equity-swap flows. In interviews, describe how regression suites protect broad business workflows and how you worked with QA on failures and scenario coverage.

---

# 9. Additional engineering topics — P2

These may not be central to your experience, but you should be able to discuss them at a conceptual level. **Clearly distinguish what you have studied from what you have operated in production.**

## 9.1 Docker and Kubernetes fundamentals

**Docker:** Package an application plus dependencies into a versioned **image**, run as a **container** with isolated process/filesystem/network environment. Images are built in layers; use minimal base images, non-root processes, health checks and no embedded secrets.

```dockerfile
# Illustrative; use a vetted runtime image matching your Java version.
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/service.jar app.jar
USER 10001
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Kubernetes:** `Pod` (smallest schedulable unit), `Deployment` (desired replicas and rollout), `Service` (stable network access), `ConfigMap` (non-secret config), `Secret` (sensitive configuration object; protect with RBAC/encryption), `Ingress`/gateway (external traffic), horizontal pod autoscaling and resource requests/limits.

**Q: Liveness vs readiness probe?** **A:** Liveness asks whether to restart the container; readiness asks whether to route traffic to it. Don't make liveness depend on every remote service or you may trigger cascading restarts.

**Q: What happens if a pod crashes?** **A:** Controller recreates it to reach desired state, but in-memory job status is lost unless persisted externally; requests must be retry/idempotency safe.

**Q: Horizontal vs vertical scaling?** **A:** Add replicas vs give more resources to a replica. Horizontally scaled services need stateless app instances or externalized session/state.

## 9.2 Cloud/distributed infrastructure fundamentals

- **Load balancer:** Routes traffic and checks health; does not fix stateful/session issues automatically.
- **Managed database:** Simplifies operations but needs backups, failover testing, access controls and capacity planning.
- **Object storage:** Large durable files/archive objects; use checksums, encryption, retention and access policies.
- **Message queue:** Buffers and decouples work; consumers need idempotency/retries.
- **Multi-zone resilience:** Deploy across failure domains; databases and state stores require appropriate replication/failover design.
- **Observability:** Logs + traces + metrics and actionable dashboards/alerts.
- **Zero-downtime release:** API/schema backward compatibility, health checking, phased rollout, rollback strategy.

**Q: What does cloud-native mean?** **A:** Architecture and operations emphasizing automation, resilience, scaling, observability and portable deployment; it does not require microservices everywhere.

## 9.3 Spring Security, authentication and authorization

**Authentication:** Who are you? **Authorization:** What can you do? A valid login does not imply permission to book/cancel a particular trade.

- **OAuth 2.0:** Framework for delegated authorization; access token authorizes specific resource access.
- **OpenID Connect (OIDC):** Authentication identity layer on OAuth 2.0.
- **JWT:** Token format, not equivalent to OAuth; may carry claims and signature, but validation must check signature, issuer, audience, expiry and policy.
- **Spring Security filter chain:** Applies authentication/authorization and other HTTP security rules before controllers.
- **RBAC vs ABAC:** Role-based permissions vs decisions using attributes (role, book, product, region, etc.).
- **Principle of least privilege:** Service accounts and users get minimum permissions.
- **CSRF vs CORS:** CSRF protects against cross-site request forgery under browser credential behavior; CORS is a browser cross-origin access policy, **not** authentication.
- **SQL injection:** Use bound parameters/prepared statements rather than concatenating user input.

**Q: How would you secure a trade booking API?** **A:** TLS, identity validation, authorization per trade/book/account, input validation, audit, rate limiting, idempotency, secrets management and carefully controlled logs. Domain permissions may be finer than generic role checks.

## 9.4 Hibernate/JPA essentials

| Concept | Interview answer |
|---|---|
| `@Entity` | Object mapped to persistent relational data |
| Persistence context | Tracks managed entities and dirty state within scope |
| Lazy vs eager | Fetch association on use vs immediately; careless lazy access can cause N+1 or exception out of session |
| `save` / `persist` / `merge` | Semantics depend on repository/API and entity state; don't equate blindly |
| Dirty checking | JPA compares managed state and synchronizes changes at flush/commit |
| `flush` vs commit | Flush sends pending SQL but transaction may still roll back |
| N+1 | Additional query per parent; fix through fetch joins/entity graphs/projections/batching |
| Optimistic locking | `@Version` detects conflicting updates |
| Transactions | Determine lazy access, locks and atomicity |

**Q: `FetchType.EAGER` to solve N+1?** **A:** Not automatically; may cause excessive loading or additional selects. Prefer query-specific fetch plans and verify SQL.

**Q: Why avoid returning entities directly in a REST API?** **A:** Can leak internal fields, trigger lazy loading/recursion, couple API to schema and complicate versioning. Return explicit DTOs.

## 9.5 Java 17 and Spring Boot 3 migration fundamentals

**Java 17 topics:** records, text blocks, sealed classes, enhanced `switch` expressions, pattern matching for `instanceof`. Records are concise value carriers; they are not a substitute for rich mutable JPA entity lifecycles. `var` is local variable type inference, not dynamic typing.

```java
record TradeAck(String tradeId, String state) {}

sealed interface BookingOutcome permits Success, Failure {}
record Success(String id) implements BookingOutcome {}
record Failure(String message) implements BookingOutcome {}
```

**Spring Boot 2.7 → 3.x concepts:** minimum **Java 17** for Spring Boot 3.0; `javax.*` EE APIs generally migrate to `jakarta.*`; Spring Framework 6/Spring Security compatibility, third-party dependencies, observability and integration tests need review. Test driver/framework versions and your codebase; don't imply a no-risk one-line dependency change.

**Q: How would you migrate 6 services safely?** **A:** Inventory dependencies and endpoints, check baseline Java/runtime, automate tests, migrate one low-risk service first, run contract/regression/security checks, roll out gradually, monitor and retain rollback plan. Schema/transport changes need backward compatibility.

**[YOUR WORK]** Your actual completed upgrade described in your notes was **Boot 2.6.8 → 2.7.5**, not a production Boot 3 upgrade. You can correctly say it created a stepping stone toward a Java 17/Boot 3 modernization effort.

---

# 10. High-probability technical–managerial scenarios

For each, use: **clarify requirement → risks → technical design → fallback/recovery → communicate impact**.

## Scenario 1. "Same trade received twice from upstream; how do you avoid two bookings?"

**Answer:** "I need a stable unique upstream trade/request ID. I validate and persist it with a unique DB constraint, using an atomic insert/claim. On retry, the same ID returns the previous outcome, or 'processing' while in progress. I also store a payload hash to detect conflicting reuse. A thread lock isn't sufficient across multiple pods or after a crash. If we publish events, I use outbox and downstream deduplication to prevent secondary side effects."

**Follow-up:** "What if request A commits but reply is lost?" → Query by idempotency key and return committed trade; **never** assume timeout means failure.

## Scenario 2. "Parallel booking improved throughput but corrupts shared fields."

**Answer:** "Stop or disable the unsafe parallel path via feature flag if needed. Reproduce with concurrency tests and inspect shared mutable request state, thread-local session handling and non-thread-safe dependencies. Give each worker isolated context, make shared collaborators stateless/thread-safe, protect DB invariants, revalidate correctness and then slowly restore concurrency with monitoring."

## Scenario 3. "Cache hit rate is 99%, but users occasionally receive incorrect Nexus results."

**Answer:** "Correctness outranks hit rate. I would correlate incorrect values to cache key, TTL, source changes and business-input completeness. Compare cached and authoritative responses, temporarily disable caching if necessary, and involve compliance/control owners to set safe TTL/invalidation or key changes. Add tests for rule version change and negative cases."

## Scenario 4. "Production trading service is down during market hours."

**Answer:** "Start incident response, assess products/users affected, check health/errors/changes and use a safe rollback or controlled fallback to restore service. Give support and business stakeholders concise status updates. In parallel, gather logs/metrics/traces and avoid risky speculative changes. After recovery, lead a timeline-based RCA and add prevention."

## Scenario 5. "An API degrades from 200ms to 5s."

**Answer:** "Segment end-to-end latency by service, database and downstream spans. Check recent releases, traffic, pool saturation, GC, locking and cache hit rates. Mitigate if significant impact, fix root bottleneck and validate P95/P99 under representative load. Share impact and next update clearly with business."

## Scenario 6. "Release tomorrow, but you find a compliance defect."

**Answer:** "Explain severity and affected booking scenarios, reproduce with evidence, propose fix and test scope, and escalate to product/control/release owners. I would not bypass a mandatory control silently to meet a deadline. Consider feature isolation or delaying affected scope with documented authorization; prioritize safe customer and compliance outcomes."

## Scenario 7. "Trade archival moved 10M records and failed."

**Answer:** "Pause further deletion, inspect checkpoints, reconcile target/source by stable keys, restart only incomplete batches, and verify counts/checksums. Keep transactions bounded and ensure unarchive path. Communicate impact and revised plan without promising data integrity until reconciliation completes."

## Scenario 8. "Process 50,000 instruments but support requests restart on node failure."

**Answer:** "Decouple submission from execution: accept a file and return job ID, persist job and per-batch checkpoints, place work in a durable queue, use bounded workers and idempotent per-instrument writes. Progress APIs read durable state. On restart requeue unfinished work. Keep per-item error details and reconciled counts."

## Scenario 9. "Kafka trade event delivered twice or out of order."

**Answer:** "Key by trade ID where per-trade ordering is required, retain a unique event ID and event/version sequence. Consumers use durable deduplication and validate allowed state transitions. If a later event arrives first, buffer/retry or reconcile against source of truth according to domain rule. DLQ irrecoverable cases with alerting."

## Scenario 10. "Junior developer proposes creating 100 threads to improve throughput."

**Answer:** "I would encourage the idea but benchmark it against 5, 10, 15 and 20 threads. We inspect CPU, DB connections, downstream rate limits, queueing and latency. More threads can create contention and slower throughput. I would show evidence from the tests, agree on a bounded safe configuration and document reasoning."

## Scenario 11. "You and your lead disagree on synchronous vs asynchronous response."

**Answer:** "Clarify the upstream contract: do they need completed booking IDs before return, or can they accept job IDs and poll? For booking flows requiring immediate final state, async fire-and-forget changes contract and failure semantics. Present options, latency/reliability implications and a prototype if useful. Align on decision and support the team."

## Scenario 12. "Two systems disagree about trade booking status."

**Answer:** "Trace correlation and message IDs through each service, examine timestamps, retry/acknowledgment, DB transaction boundary and event consumers. Establish source of truth and safe reconciliation plan; do not manually overwrite a trade status without controls and audit. Fix propagation or state machine and add monitoring for divergence."

## Scenario 13. "How would you handle a security vulnerability during a Spring upgrade?"

**Answer:** "Assess exposure and severity with security owners, choose a supported compatible version, test H2/MSSQL driver and application behavior, run automated regression and dependency scans, stage rollout and preserve rollback. Verify TLS certificate trust; encrypted connections and verified server identity are different guarantees."

## Scenario 14. "What happens if a database connection pool is exhausted?"

**Answer:** "Requests wait for connections, increasing latency/timeouts. Check long transactions, leaks, slow queries, maximum pool size and DB capacity. Fix the cause and set sensible timeouts; simply increasing pool size could overload the database. Alert on active/idle/waiting metrics."

## Scenario 15. "A teammate's code works but has no regression tests. Deadline is close."

**Answer:** "Focus on the highest-risk paths: positive, negative, duplicate, failure and compatibility scenarios. Explain production risk and propose a limited but defensible test plan with time estimate. If risk remains high, escalate and adjust release scope. Help the teammate rather than just rejecting their work."

## Scenario 16. "A remote eligibility service is unavailable. Block trades or continue?"

**Answer:** "That is a business/compliance decision, not only a technical one. I would identify which products/rules require strict failure, whether an authorized override and audit trail exist, and how to alert/reconcile later. Implement the approved mode using feature controls and make each exception visible. Unconditional fail-open is unsafe for mandatory controls."

## Scenario 17. "How would you handle a deadlock in production?"

**Answer:** "Identify lock victims and queries/transactions involved; preserve DB evidence, mitigate safely, then shorten transactions, make table/row access order consistent and fix missing indexes causing broad locks. Add bounded retries for known transient deadlock errors with idempotency protection."

## Scenario 18. "How do you take ownership of a feature end to end?"

**Answer:** "Clarify business outcome and edge cases, document contract/risks, propose design and estimates, implement with tests, coordinate QA/UAT, monitor rollout and support production. I consider it complete when the business outcome is verified and operational support can own it, not only when my PR merges."

---
# 11. Rapid-fire questions and final cheat sheet

## 11.1 Forty-five one-minute questions

Use these as an oral flashcard deck. Try to answer without looking at the right-hand column first.

| # | Question | Interview-ready short answer |
|---:|---|---|
| 1 | Four pillars of OOP? | Encapsulation, abstraction, inheritance, polymorphism; use them to protect invariants and vary behavior cleanly. |
| 2 | Encapsulation vs abstraction? | Hide/control internal state vs expose a simplified contract independent of implementation. |
| 3 | Interface vs abstract class? | Contract/multiple interface implementation vs related base class with shared state and behavior. |
| 4 | Overloading vs overriding? | Same-name methods with different signatures vs dynamic override of inherited instance behavior. |
| 5 | Why composition over inheritance? | Less coupling, no fragile hierarchy, easier unit testing and replaceable collaborators. |
| 6 | SOLID? | SRP, Open/Closed, Liskov, Interface Segregation, Dependency Inversion. |
| 7 | `==` vs `equals()`? | Reference identity for objects vs overridable logical equality. |
| 8 | `equals()` and `hashCode()` rule? | Equal objects must have equal hashes; immutable map keys preferred. |
| 9 | HashMap vs ConcurrentHashMap? | Ordinary map not safe for concurrent mutation; concurrent map supports thread-safe operations and no null keys/values. |
| 10 | HashMap collisions? | Bucket collision is resolved by equality comparison, chains/tree bins depending on implementation. |
| 11 | `ArrayList` vs `LinkedList`? | Contiguous indexed array usually faster for reads/iteration; linked nodes have traversal/memory cost. |
| 12 | Java streams lazy? | Intermediate operators are lazy; pipeline runs on a terminal operation. |
| 13 | `map()` vs `flatMap()`? | One-to-one transform vs flattening nested results. |
| 14 | `Optional.orElse` vs `orElseGet`? | Argument eager vs supplier lazily evaluated when empty. |
| 15 | Checked vs unchecked exceptions? | Checked must be handled/declared; unchecked RuntimeExceptions need not. |
| 16 | `final` makes object immutable? | No; reference cannot be reassigned, mutable object may still change. |
| 17 | Thread vs process? | Threads share address space/resources of process, each has its own stack. |
| 18 | Runnable vs Callable? | Runnable returns no value; Callable produces result and can throw checked exception. |
| 19 | `volatile` vs `synchronized`? | Visibility/order vs mutual exclusion plus visibility. |
| 20 | Why `count++` race? | Read-modify-write is not atomic; use an atomic operation/lock. |
| 21 | Deadlock? | Circular wait among threads/transactions; avoid inconsistent lock ordering. |
| 22 | Future vs CompletableFuture? | Simple result/cancellation vs composable async stages and error paths. |
| 23 | Is `CompletableFuture.orTimeout` guaranteed to stop work? | No. Future can time out while task continues unless cancellation is cooperative. |
| 24 | Why dedicated executor? | Control thread/queue capacity, naming, overload, shutdown and isolation. |
| 25 | Singleton vs prototype? | Singleton one bean per container/definition; prototype new instance per retrieval. |
| 26 | Prototype injected into singleton? | Normally one prototype injected on singleton creation; ObjectProvider for fresh instances. |
| 27 | Is a Spring singleton thread-safe? | Only if designed to be; avoid mutable per-request instance fields. |
| 28 | Constructor vs field injection? | Constructor makes required dependencies explicit and improves testability/immutability. |
| 29 | What does Spring auto-configuration do? | Registers eligible defaults conditioned on classpath, properties and existing beans. |
| 30 | `@Transactional` self-invocation issue? | `this.method()` skips the proxy, so advice may not execute. |
| 31 | Default Spring rollback? | Generally on RuntimeException/Error; checked exceptions need explicit rules. |
| 32 | `REQUIRED` vs `REQUIRES_NEW`? | Join/create current transaction vs suspend and create independent transaction. |
| 33 | Can `@Transactional` span worker threads? | Not automatically; each worker needs appropriate transaction boundary. |
| 34 | GET vs POST vs PUT? | Read, create/process, replace/upsert; HTTP idempotency differs. |
| 35 | `200` vs `201` vs `202`? | OK response, new resource created, async request accepted. |
| 36 | At-least-once message delivery? | Redelivery possible; durable idempotency required. |
| 37 | Kafka ordering? | Per partition; choose key aligned with ordering requirements. |
| 38 | What is outbox? | Transactional DB event record published later to avoid lost side effects. |
| 39 | Unique constraint and duplicate booking? | Constraint enforces uniqueness across application instances even under races. |
| 40 | ACID? | Atomicity, Consistency, Isolation, Durability. |
| 41 | Isolation prevents all anomalies by default? | No; depends on level/DB-specific concurrency model. |
| 42 | How to tune slow SQL? | Inspect actual plan, indexes, selectivity, statistics, blocking, execution metrics. |
| 43 | Cache TTL drawback? | Stale values until expiry; correctness/freshness must be agreed with business. |
| 44 | Liveness vs readiness? | Restart health vs ability to receive traffic. |
| 45 | Java 17 / Boot 3 migration key point? | Boot 3.0 baseline Java 17 and Jakarta package migration; test dependent libraries. |

## 11.2 The five answers you should practice verbatim *in structure*, not as memorized scripts

1. **OOP:** "OOP gives me encapsulation for invariants, abstraction for contract, polymorphism for extensible product-specific behavior, and inheritance only where a true is-a relationship exists. In trade processing, I prefer validators behind interfaces and composition."
2. **Spring scopes:** "Singleton is the default bean scope, one per container and bean definition; prototype creates a new instance on retrieval; request/session/application/websocket are web-associated scopes. A singleton isn't automatically thread-safe, and a prototype directly injected into a singleton won't be recreated for every call."
3. **Concurrency:** "I parallelize independent booking groups using bounded executors, isolate mutable context per worker, propagate/clear request context safely, and measure throughput versus contention. The database remains the source of truth for uniqueness."
4. **Transactions and idempotency:** "A local transaction protects DB atomicity; a unique idempotency key protects repeated requests; outbox + consumer dedupe protect cross-system event delivery. No single application lock solves all failure windows."
5. **Caching:** "Cache hits improve latency but may return stale data. I choose key/TTL/invalidation from business correctness requirements, measure hit/miss and tail latency, and maintain a safe fallback."

## 11.3 Engineering trade-off cheat sheet

| Interview decision | Good answer starts with | Bad answer to avoid |
|---|---|---|
| Synchronous vs async API | Upstream completion contract and failure semantics | "Async is always faster, so use it everywhere" |
| Cache vs direct lookup | Freshness, reuse, cache correctness and fallback | "Cache makes everything fast" |
| More worker threads | CPU vs I/O, DB/downstream limits and tests | "More threads always mean more throughput" |
| Distributed lock vs unique constraint | Durable correctness after crash/across pods | "synchronized guarantees once-only booking" |
| Retry vs fail fast | Error type, side effects, idempotency and backoff | "Retry every error infinitely" |
| Microservices vs monolith | Deployment and bounded-context needs vs complexity | "Microservices are always modern and better" |
| Strong vs eventual consistency | Business invariants and acceptable interim state | "Eventual consistency is fine for every trade field" |
| Data archival | Compliance retention, traceability, restart safety | "Delete all old trades in one transaction" |
| Spring scope | Lifecycle and state/thread safety | "Singleton is shared globally by all pods" |
| Security | Least privilege, identity validation, audit | "Encryption with trust-all certificate validates identity" |

## 11.4 Two-day revision plan

**Wednesday evening: build core confidence**

- [ ] 35 min: OOP + SOLID + Strategy/Factory + interview flashcards.
- [ ] 45 min: Java collections, `equals/hashCode`, streams, exception fundamentals.
- [ ] 70 min: Multithreading: pools, deep cloning, context propagation, CompletableFuture, duplicate booking.
- [ ] 45 min: Spring IoC, scopes, lifecycle, `@Transactional` and self-invocation.
- [ ] 20 min: Deliver 2-minute explanations of caching and parallel booking without reading.

**Thursday: architecture and production judgment**

- [ ] 55 min: REST, Kafka/Solace, retries and transactional outbox.
- [ ] 45 min: SQL indexing, isolation, archived-trade recovery.
- [ ] 50 min: Coherence cache staleness, trade-service high-level design, 50k expiry service.
- [ ] 45 min: Production diagnostics, testing, GitLab CI/CD, rollout/rollback.
- [ ] 35 min: Docker/K8s/security/JPA/Java 17 overview.
- [ ] 45 min: Answer 10 technical–managerial scenarios aloud with a timer.

**Friday before the interview: 20–30 minutes only**

- [ ] Recite the five core answers above and first 20 flashcards.
- [ ] Recall your actual performance figures and distinguish production facts from proposed enhancements.
- [ ] Review 3 failure scenarios: duplicate trade, stale cache, partial bulk failure.
- [ ] Check laptop, camera, microphone, connectivity and meeting invite.

## 11.5 Final self-assessment: answer aloud before looking

1. Explain all four OOP pillars using a trade validator.
2. Build an immutable key with correct `equals` and `hashCode`.
3. Why is mutable state in a Spring singleton dangerous?
4. Why does `@Transactional` not work through `this.someMethod()` in normal proxy mode?
5. In what circumstances is a prototype bean **not** created per method call?
6. Explain at-most-once / at-least-once delivery and the limits of "exactly once".
7. Two instances get the same trade request at the same moment. Walk through safe DB state changes.
8. Crash occurs after trade DB commit but before response. How does the client recover?
9. Why might 15 booking threads be faster than 50?
10. How do you pass request/security context safely to worker threads?
11. How do you detect a JVM deadlock or DB deadlock?
12. Why does a cache with a 24-hour TTL create a compliance question?
13. How did you prove 956 ms → 2 ms was a cache-hit measurement?
14. How would you implement job restart for a 50,000-item bulk process?
15. How do you archive 18.5 million trades without losing records?
16. How would you debug 95% CPU plus an API timeout?
17. Which metrics would you add to a high-volume booking service?
18. What are five test scenarios a trade booking service must pass before release?
19. What are the major compatibility issues moving from Spring Boot 2.7 to 3?
20. If a critical compliance validation dependency fails, who decides fail-open vs fail-closed and why?

**Passing standard:** You can explain **why** a design is safe, which failure modes it handles, what it does **not** guarantee and how you would verify it. That is what separates an experienced backend engineer from a candidate who has only memorized definitions.

---

# Notes on provenance and further reading

**Your project-specific examples** are drawn from the supplied **Sagar_Mali_Resume_3YOE.pdf** and **Atlas Trade Contributions, Technical Changes and Business Impact** file. The examples attributed to your work reflect those documents; this guide deliberately calls out proposed improvements (such as persisted bulk-job state and transactional outbox) as *illustrative*, not claimed implementation.

**Official technical references** useful for accurate follow-up reading:

- [Spring Framework — Bean scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Spring Framework — Transactional annotations and proxy behavior](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Boot 3.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)
- [Java SE 17 — java.util.concurrent reference](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Java SE 17 — ExecutorService reference](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ExecutorService.html)

**Practical interview reminder:** Many code snippets here are illustrative interview examples, sometimes abbreviated for readability. They are not reviewed production implementations. When asked about your Barclays work, distinguish the actual design and metrics from what you *would improve next*.
