# Java Exception Handling — Interview Ready Notes (3+ YOE)

---

## 1. Exception Hierarchy

Everything starts at `Throwable`. Understanding this hierarchy is the #1 thing interviewers probe.

```
                Throwable
               /         \
          Error          Exception
         /                /       \
  (OOM, StackOverflow) RuntimeException   (Checked Exceptions)
                        /        |        \
                IllegalArgument  NullPointer  ArrayIndexOutOfBounds
                Exception          Exception    Exception
```

```java
Object
 └── Throwable
       ├── Error                         // JVM-level, unrecoverable (don't catch)
       │     ├── OutOfMemoryError
       │     ├── StackOverflowError
       │     └── NoClassDefFoundError
       └── Exception
             ├── RuntimeException        // Unchecked — programming errors
             │     ├── NullPointerException
             │     ├── ArrayIndexOutOfBoundsException
             │     ├── IllegalArgumentException
             │     ├── IllegalStateException
             │     ├── ClassCastException
             │     ├── NumberFormatException
             │     └── ArithmeticException
             └── (Checked Exceptions)    // Must be declared or caught
                   ├── IOException
                   ├── SQLException
                   ├── ClassNotFoundException
                   └── InterruptedException
```

**Key point:** `RuntimeException` and its subclasses, plus `Error`, are **unchecked**. Everything else under `Exception` is **checked**.

---

## 2. Checked vs Unchecked Exceptions

| Aspect | Checked | Unchecked |
|---|---|---|
| Compile-time enforcement | Must be caught or declared (`throws`) | No enforcement |
| Extends | `Exception` (not `RuntimeException`) | `RuntimeException` or `Error` |
| Represents | Recoverable, external conditions (file missing, network down) | Programming bugs / logic errors |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `ArrayIndexOutOfBoundsException` |
| Philosophy | "Caller must handle or acknowledge" | "This shouldn't happen if code is correct" |

```java
// Checked — compiler forces you to handle it
public void readFile() throws IOException {
    FileReader fr = new FileReader("data.txt");
}

// Unchecked — compiles fine even if you ignore the risk
public void divide(int a, int b) {
    int result = a / b; // ArithmeticException possible, but no forced handling
}
```

**Common debate question — "Are checked exceptions a good design?"**
Many senior engineers (including parts of the Java community) consider checked exceptions controversial — they can lead to boilerplate `try/catch` or exception-swallowing (`catch(Exception e) {}`), and get in the way of functional-style code (lambdas, streams don't play well with checked exceptions). This is why most modern frameworks (Spring, Hibernate) wrap checked exceptions into unchecked ones (e.g., `DataAccessException` wraps `SQLException`).

---

## 3. try-catch-finally Mechanics

```java
try {
    riskyOperation();
} catch (IOException e) {
    log.error("IO failed", e);
} catch (SQLException e) {
    log.error("DB failed", e);
} finally {
    cleanup(); // ALWAYS runs, even if try/catch has return/throw
}
```

### Multi-catch (Java 7+)
```java
try {
    riskyOperation();
} catch (IOException | SQLException e) {
    // Handle both the same way; e is effectively final, common supertype
    log.error("Operation failed", e);
}
```
Note: In multi-catch, the exception types must not be related by inheritance (can't combine a class with its subclass) — compiler error otherwise.

### finally execution order — classic trap question

```java
public static int test() {
    try {
        return 1;
    } finally {
        System.out.println("finally runs");
    }
}
// Output: "finally runs" printed, then returns 1
```

**Trickier version (interview favorite):**
```java
public static int test() {
    try {
        return 1;
    } finally {
        return 2; // overrides the try's return value!
    }
}
// Returns 2 — finally's return SWALLOWS the try's return
```

**Rule:** `finally` always executes before the method actually returns/throws — and if `finally` itself has a `return` or `throw`, it **overrides/swallows** whatever the `try`/`catch` was going to do. This is considered bad practice — **never put `return` inside `finally`**.

### When does `finally` NOT run?
- `System.exit()` is called inside `try`.
- JVM crashes (e.g., `kill -9`, power failure).
- Infinite loop / thread killed abruptly inside `try`.

---

## 4. try-with-resources (Java 7+) — Very High Frequency Topic

Replaces manual `finally`-based resource closing. Any class implementing `AutoCloseable` (or `Closeable`) qualifies.

```java
// Old way — verbose and error-prone
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("file.txt"));
    return br.readLine();
} finally {
    if (br != null) br.close();
}

// Modern way — auto-closed, even on exception
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    return br.readLine();
} // br.close() called automatically here
```

### Multiple resources (closed in reverse order of declaration)
```java
try (FileInputStream fis = new FileInputStream("in.txt");
     FileOutputStream fos = new FileOutputStream("out.txt")) {
    // use both
} // fos closed first, then fis
```

### Custom AutoCloseable
```java
class DbConnection implements AutoCloseable {
    public void query() { System.out.println("querying..."); }
    @Override
    public void close() {
        System.out.println("closing connection");
    }
}

try (DbConnection conn = new DbConnection()) {
    conn.query();
}
// Output:
// querying...
// closing connection
```

### Suppressed Exceptions (important nuance)
If an exception is thrown in the `try` block **and** another exception is thrown while closing the resource, the close-exception is **suppressed**, not lost — it's attached to the primary exception and retrievable via `getSuppressed()`.

```java
try (AutoCloseable r = () -> { throw new RuntimeException("close failed"); }) {
    throw new RuntimeException("main failure");
}
// Thrown: "main failure"
// Suppressed: "close failed"  (visible via e.getSuppressed())
```

---

## 5. Custom Exceptions

```java
// Checked custom exception
public class InsufficientFundsException extends Exception {
    private final double shortfall;

    public InsufficientFundsException(String message, double shortfall) {
        super(message);
        this.shortfall = shortfall;
    }

    public double getShortfall() {
        return shortfall;
    }
}

// Unchecked custom exception (more common in modern APIs / Spring services)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
    public ResourceNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**Best practice guidelines:**
- Always provide constructors that accept `message` and `message + cause` — preserves the original stack trace for debugging.
- Name custom exceptions ending in `Exception`.
- Prefer extending `RuntimeException` for business/service-layer errors in modern backend code (especially with Spring) — avoids `throws` clutter through every layer.
- Include only relevant context fields (like `shortfall` above), not entire domain objects — keep exceptions lightweight and serializable.

```java
throw new ResourceNotFoundException("User not found with id: " + userId);
```

---

## 6. Exception Chaining (Cause Preservation) — Frequently Missed in Real Code

When you catch one exception and throw another, **always preserve the original cause** — otherwise you lose the root-cause stack trace, making production debugging painful.

```java
// BAD — swallows the original stack trace
try {
    parseConfig();
} catch (IOException e) {
    throw new RuntimeException("Config parsing failed"); // original 'e' lost!
}

// GOOD — chains the cause
try {
    parseConfig();
} catch (IOException e) {
    throw new RuntimeException("Config parsing failed", e); // preserved
}
```

Access the original cause later via `getCause()`:
```java
catch (RuntimeException e) {
    Throwable root = e.getCause(); // the original IOException
}
```

---

## 7. Common Runtime Exceptions — Root Causes

| Exception | Typical Cause |
|---|---|
| `NullPointerException` | Calling a method/field on a null reference |
| `ArrayIndexOutOfBoundsException` | Accessing array index outside `[0, length-1]` |
| `ClassCastException` | Invalid downcast, e.g., `(String) obj` when obj is an Integer |
| `NumberFormatException` | `Integer.parseInt("abc")` |
| `ArithmeticException` | Division by zero (`int` only — floating point gives `Infinity`/`NaN` instead) |
| `ConcurrentModificationException` | Modifying a collection while iterating with a for-each/Iterator |
| `IllegalArgumentException` | Passing an invalid/unexpected argument to a method |
| `IllegalStateException` | Calling a method when the object is in the wrong state (e.g., `Iterator.remove()` before `next()`) |
| `UnsupportedOperationException` | Modifying an immutable collection, e.g., `List.of(1,2).add(3)` |

```java
int[] arr = {1, 2, 3};
System.out.println(10 / 0);          // ArithmeticException
System.out.println(10.0 / 0);         // Infinity — no exception!
System.out.println(arr[5]);            // ArrayIndexOutOfBoundsException

List<Integer> list = List.of(1, 2, 3);
list.add(4);                              // UnsupportedOperationException

for (Integer i : list) {
    list.remove(i);                        // ConcurrentModificationException
}
```

---

## 8. NullPointerException Deep-Dive (Very Common in Real Interviews)

### Helpful NPEs (Java 14+, JEP 358)
Modern JVMs now print **exactly which variable was null**, drastically improving debuggability.

```java
String name = person.getAddress().getCity().toUpperCase();
// Java 14+ error message:
// Cannot invoke "String.toUpperCase()" because the return value of
// "Address.getCity()" is null
```

### Defensive strategies
```java
// 1. Objects.requireNonNull — fail fast with clear message
public void setName(String name) {
    this.name = Objects.requireNonNull(name, "name must not be null");
}

// 2. Optional — express "may be absent" explicitly in the type system
public Optional<User> findUser(String id) {
    return Optional.ofNullable(repository.find(id));
}
User user = findUser("123").orElseThrow(() -> new ResourceNotFoundException("User not found"));

// 3. Objects.equals / Objects.isNull — null-safe utility comparisons
Objects.equals(a, b);   // true if both null, or a.equals(b)
```

---

## 9. Best Practices (Senior-Level Talking Points)

1. **Never swallow exceptions silently:**
```java
// BAD
try { doWork(); } catch (Exception e) { } // silent failure — debugging nightmare

// GOOD — at minimum, log it
try { doWork(); } catch (Exception e) { log.error("doWork failed", e); }
```

2. **Catch specific exceptions, not generic `Exception`/`Throwable`** — avoids masking bugs unrelated to the intended failure.
```java
// BAD — too broad, could hide NullPointerException, bugs, etc.
try { ... } catch (Exception e) { ... }

// GOOD
try { ... } catch (SQLException e) { ... } catch (IOException e) { ... }
```

3. **Don't use exceptions for normal control flow** — exceptions are expensive (stack trace capture) and hurt readability.
```java
// BAD
try {
    return list.get(index);
} catch (IndexOutOfBoundsException e) {
    return null;
}

// GOOD
return (index < list.size()) ? list.get(index) : null;
```

4. **Fail fast** — validate inputs early and throw immediately, rather than letting bad state propagate.

5. **Always clean up resources** — use try-with-resources instead of manual `finally` blocks.

6. **Preserve the stack trace** when wrapping exceptions (pass `cause`).

7. **Don't catch `Throwable`/`Error`** unless you have a very specific reason (e.g., top-level framework code) — `Error`s like `OutOfMemoryError` usually mean the JVM itself is in trouble; catching and continuing is dangerous.

8. **Log once, at the right layer** — don't log the same exception at every layer as it propagates up (log spam); log with full context at the point where you handle it or at the top-level boundary.

---

## 10. Spring Boot / Backend-Specific Exception Handling (High-Value for 3+ YOE)

### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return new ResponseEntity<>(new ErrorResponse(400, message), HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        ErrorResponse error = new ErrorResponse(500, "Internal server error");
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

Why this matters in interviews: it shows you know how to centralize exception-to-HTTP-status mapping instead of scattering `try/catch` across every controller — a hallmark of clean REST API design.

### Transaction rollback nuance (classic Spring gotcha)
By default, `@Transactional` only rolls back on **unchecked exceptions** (`RuntimeException`/`Error`), NOT on checked exceptions, unless explicitly configured:

```java
@Transactional(rollbackFor = Exception.class) // forces rollback even for checked exceptions
public void transferMoney() throws InsufficientFundsException {
    ...
}
```
This is a **very popular real-world interview question** for candidates with Spring experience — many engineers get bitten by this default behavior in production.

---

## 11. Stack Trace & printStackTrace Internals

```java
try {
    throw new RuntimeException("boom");
} catch (RuntimeException e) {
    e.printStackTrace();          // prints to System.err — avoid in production code
    log.error("Something failed", e); // preferred — goes through logging framework
}
```

- `Throwable.fillInStackTrace()` captures the stack at the point of **construction** (not where it's thrown/caught) — this is why re-throwing the same exception object across layers can show a misleading origin unless you're careful.
- Constructing exceptions (especially with stack trace capture) is relatively expensive — a known reason to avoid using exceptions for routine control flow in hot paths.

---

## 12. Tricky Interview Questions & Answers

**Q: Can you catch an `Error`?**
**A:** Technically yes (`Error` extends `Throwable`), but it's discouraged — Errors like `OutOfMemoryError` or `StackOverflowError` indicate the JVM itself is compromised; recovery is usually not meaningful.

**Q: What happens if an exception is thrown in a `catch` block?**
**A:** If there's a `finally` block, it still executes. Then the new exception propagates up (the original exception, if not chained, is lost unless the new one wraps it as a cause).

**Q: Can a `finally` block throw an exception that hides the original?**
**A:** Yes — if `finally` itself throws, it **replaces** any exception from `try`/`catch`. This is a classic source of silently lost errors.

**Q: What's the difference between `throw` and `throws`?**
**A:** `throw` actually raises an exception instance at runtime (`throw new IOException()`); `throws` is a method-signature declaration telling callers "this method might raise this checked exception."

**Q: Why can't you override a method and add a broader checked exception than the parent declares?**
**A:** Because it would break the Liskov Substitution Principle — code calling the method via the parent type/interface wouldn't be prepared to catch the new checked exception at compile time.
```java
class Parent {
    void doWork() throws IOException { }
}
class Child extends Parent {
    // COMPILE ERROR: SQLException is not a subtype of IOException
    void doWork() throws SQLException { }
}
```
(Note: You CAN narrow to a subtype, or throw unchecked exceptions freely, or throw no exception at all.)

**Q: Is it good practice to have a method throw `Exception` broadly?**
**A:** No — it forces every caller to catch a generic `Exception`, hiding what can actually go wrong, and discourages specific handling. Declare the narrowest checked exception type.

**Q: What's the difference between `Exception(String message)` and `Exception(String message, Throwable cause)`?**
**A:** The second constructor variant chains the original exception as the `cause`, preserving its stack trace — critical for root-cause debugging in layered applications.

**Q: How does try-with-resources guarantee closing even when an exception occurs mid-resource-creation (multiple resources)?**
**A:** Each resource is closed in reverse declaration order as the try block unwinds; if an earlier resource's constructor throws, only the resources successfully opened before it get closed.

---

## 13. Quick-Fire Revision Table

| Concept | One-liner |
|---|---|
| `Throwable` | Root of Error + Exception hierarchy |
| Checked exception | Must be caught/declared; represents recoverable external conditions |
| Unchecked exception | `RuntimeException`/`Error`; represents programming bugs |
| `finally` | Always runs; a `return`/`throw` inside it overrides try/catch's outcome |
| try-with-resources | Auto-closes `AutoCloseable` resources, reverse declaration order |
| Suppressed exceptions | Close-time exceptions attached to the primary exception, not lost |
| Exception chaining | Always pass `cause` when wrapping exceptions |
| `@Transactional` default | Rolls back only on unchecked exceptions unless `rollbackFor` is set |
| Best practice | Catch specific types, don't swallow, don't use for control flow |
| Java 14+ Helpful NPE | Pinpoints exact null variable in the message |

---

### Suggested Follow-Up Topics to Prep Next
Collections Framework (HashMap internals), Multithreading & Concurrency, JVM Memory Model & Garbage Collection, Streams API, Spring Core/Boot Annotations.