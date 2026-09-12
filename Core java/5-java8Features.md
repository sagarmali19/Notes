# Java 8 Features — Interview Ready Notes
### For Backend Developer (3+ YOE)

---

## 1. Lambda Expressions

### What & Why
A lambda expression is an anonymous function — a block of code with parameters that you can pass around like a value. Java 8 introduced them to enable **functional-style programming** and reduce boilerplate (especially for anonymous inner classes).

### Syntax
```java
(parameters) -> expression
(parameters) -> { statements; }
```

### Examples
```java
// No parameters
Runnable r = () -> System.out.println("Running...");

// One parameter (parentheses optional)
Consumer<String> printer = name -> System.out.println("Hello " + name);

// Multiple parameters
Comparator<Integer> comp = (a, b) -> a - b;

// With explicit types
BinaryOperator<Integer> add = (Integer a, Integer b) -> a + b;

// Block body
Function<Integer, Integer> square = (x) -> {
    int result = x * x;
    return result;
};
```

### Key interview points
- Lambdas **can only be used where a functional interface is expected** (target typing).
- Lambdas don't have their own `this` — `this` refers to the enclosing class instance (unlike anonymous classes).
- Lambdas can access **effectively final** local variables (variables not reassigned after initialization).
- No lambda expression exists as a standalone type at runtime — the compiler infers the functional interface type from context.

```java
// Effectively final example
int factor = 10; // effectively final
Function<Integer, Integer> multiplier = x -> x * factor;
// factor = 20; // ❌ would cause compile error if uncommented above
```

---

## 2. Functional Interface

### Definition
An interface with **exactly one abstract method** (SAM — Single Abstract Method). Can have any number of `default` and `static` methods.

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);

    default void printResult(int a, int b) {
        System.out.println("Result: " + calculate(a, b));
    }
}
```

- `@FunctionalInterface` is **optional** but recommended — the compiler throws an error if you accidentally add a second abstract method.
- Lambdas and method references are only assignable to functional interfaces.

```java
Calculator addition = (a, b) -> a + b;
addition.printResult(3, 4); // Result: 7
```

**Common gotcha asked in interviews:** Can a functional interface extend another interface?
- Yes, if the parent interface has no abstract methods of its own, or if it only re-declares the same abstract method (doesn't add a new one).

---

## 3. Anonymous Class vs Lambda

Anonymous classes were the pre-Java 8 way to implement functional interfaces (or any interface/abstract class) inline.

```java
// Anonymous class
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Anonymous class running");
    }
};

// Lambda equivalent
Runnable r2 = () -> System.out.println("Lambda running");
```

### Key Differences (frequently asked)

| Aspect | Anonymous Class | Lambda |
|---|---|---|
| `this` keyword | Refers to the anonymous class instance | Refers to the enclosing class instance |
| Compiled to | A separate `.class` file (e.g., `Outer$1.class`) | Uses `invokedynamic`, no separate class file generated at compile time |
| Can implement | Any interface (multiple abstract methods) or abstract class | Only functional interfaces (single abstract method) |
| Verbosity | More boilerplate | Concise |
| Instantiation | Creates a new object with its own state | Doesn't necessarily create a new object each time (can be cached by JVM) |
| Variable shadowing | Can shadow outer class variables | Cannot shadow — same scope as enclosing method |

```java
class Demo {
    int x = 10;
    void test() {
        Runnable anon = new Runnable() {
            int x = 20;
            public void run() {
                System.out.println(this.x); // 20 (anonymous class's own x)
            }
        };
        Runnable lambda = () -> System.out.println(this.x); // 10 (Demo's x)
    }
}
```

---

## 4. Core Functional Interfaces (`java.util.function`)

### 4.1 `Predicate<T>` — takes T, returns boolean
Used for **testing/filtering** conditions.

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4)); // true

// Combining predicates
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEvenAndPositive = isEven.and(isPositive);
Predicate<Integer> isEvenOrNegative = isEven.or(n -> n < 0);
Predicate<Integer> isOdd = isEven.negate();

// Common use: filtering a stream
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);
List<Integer> evens = nums.stream().filter(isEven).collect(Collectors.toList());
```
**Method signature:** `boolean test(T t)`
Default methods: `and()`, `or()`, `negate()`. Static: `Predicate.isEqual(Object)`.

### 4.2 `Function<T, R>` — takes T, returns R
Used for **transforming** data.

```java
Function<String, Integer> stringLength = String::length;
System.out.println(stringLength.apply("Hello")); // 5

// Chaining
Function<Integer, Integer> addTwo = n -> n + 2;
Function<Integer, Integer> multiplyByThree = n -> n * 3;

Function<Integer, Integer> combined1 = addTwo.andThen(multiplyByThree); // (n+2)*3
Function<Integer, Integer> combined2 = addTwo.compose(multiplyByThree); // (n*3)+2

System.out.println(combined1.apply(2)); // (2+2)*3 = 12
System.out.println(combined2.apply(2)); // (2*3)+2 = 8
```
**Method signature:** `R apply(T t)`
Default methods: `andThen()`, `compose()`. Static: `Function.identity()`.

### 4.3 `Consumer<T>` — takes T, returns nothing
Used for **performing an action** (side effects), like printing, saving, logging.

```java
Consumer<String> print = s -> System.out.println("Value: " + s);
print.accept("Java 8");

// Chaining with andThen
Consumer<String> logIt = s -> System.out.println("Logging: " + s);
Consumer<String> combined = print.andThen(logIt);
combined.accept("Test"); // prints both

// Common use: forEach
List<String> names = List.of("A", "B", "C");
names.forEach(print);
```
**Method signature:** `void accept(T t)`
Default method: `andThen()`.

### 4.4 `Supplier<T>` — takes nothing, returns T
Used for **lazy generation/supplying** of values (factories, lazy initialization).

```java
Supplier<Double> randomValue = () -> Math.random();
System.out.println(randomValue.get());

Supplier<List<String>> listSupplier = ArrayList::new;
List<String> list = listSupplier.get();

// Common use: lazy evaluation in Optional
Optional<String> opt = Optional.empty();
String result = opt.orElseGet(() -> "Default from Supplier");
```
**Method signature:** `T get()`
No default methods.

### 4.5 `BiPredicate<T, U>` — takes two args, returns boolean
```java
BiPredicate<String, Integer> lengthCheck = (str, len) -> str.length() == len;
System.out.println(lengthCheck.test("Hello", 5)); // true

BiPredicate<Integer, Integer> isGreater = (a, b) -> a > b;
```
**Method signature:** `boolean test(T t, U u)`
Default methods: `and()`, `or()`, `negate()` (same as Predicate).

### 4.6 `BiFunction<T, U, R>` — takes two args, returns R
```java
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
System.out.println(add.apply(3, 4)); // 7

BiFunction<String, String, String> concat = (s1, s2) -> s1 + s2;

// andThen (no compose available for BiFunction)
BiFunction<Integer, Integer, Integer> addThenDouble =
        add.andThen(result -> result * 2);
System.out.println(addThenDouble.apply(2, 3)); // (2+3)*2 = 10
```
**Method signature:** `R apply(T t, U u)`
Default method: `andThen()` only (no `compose`, since it needs 2 inputs).

### 4.7 `BiConsumer<T, U>` — takes two args, returns nothing
```java
BiConsumer<String, Integer> printEntry = (name, age) ->
        System.out.println(name + " is " + age + " years old");
printEntry.accept("Alice", 30);

// Common use: Map.forEach
Map<String, Integer> map = Map.of("A", 1, "B", 2);
map.forEach((k, v) -> System.out.println(k + "=" + v));
```
**Method signature:** `void accept(T t, U u)`
Default method: `andThen()`.

### 4.8 `UnaryOperator<T>` — extends `Function<T, T>`
Input and output are the **same type**.

```java
UnaryOperator<Integer> square = x -> x * x;
System.out.println(square.apply(5)); // 25

UnaryOperator<String> toUpper = String::toUpperCase;

// Common use: List.replaceAll
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));
nums.replaceAll(n -> n * 10); // uses UnaryOperator
System.out.println(nums); // [10, 20, 30]
```
**Method signature:** `T apply(T t)` (inherited from Function)
Static: `UnaryOperator.identity()`.

### 4.9 `BinaryOperator<T>` — extends `BiFunction<T, T, T>`
Both inputs and output are the **same type**. Very commonly used in `reduce()`.

```java
BinaryOperator<Integer> sum = (a, b) -> a + b;
System.out.println(sum.apply(5, 10)); // 15

// Common use: Stream.reduce
List<Integer> nums = List.of(1, 2, 3, 4, 5);
int total = nums.stream().reduce(0, sum); // 15

// Static helper methods
BinaryOperator<Integer> maxOp = BinaryOperator.maxBy(Comparator.naturalOrder());
BinaryOperator<Integer> minOp = BinaryOperator.minBy(Comparator.naturalOrder());
System.out.println(maxOp.apply(10, 20)); // 20
```
**Method signature:** `T apply(T t1, T t2)` (inherited from BiFunction)
Static: `minBy()`, `maxBy()`.

### Quick Reference Table

| Interface | Input | Output | Method | Typical Use |
|---|---|---|---|---|
| `Predicate<T>` | T | boolean | `test(T)` | Filtering |
| `Function<T,R>` | T | R | `apply(T)` | Transformation |
| `Consumer<T>` | T | void | `accept(T)` | Side effects (printing, saving) |
| `Supplier<T>` | none | T | `get()` | Lazy/factory creation |
| `BiPredicate<T,U>` | T, U | boolean | `test(T,U)` | Two-arg condition |
| `BiFunction<T,U,R>` | T, U | R | `apply(T,U)` | Two-arg transformation |
| `BiConsumer<T,U>` | T, U | void | `accept(T,U)` | Two-arg side effect (Map.forEach) |
| `UnaryOperator<T>` | T | T | `apply(T)` | Same-type transform |
| `BinaryOperator<T>` | T, T | T | `apply(T,T)` | reduce/aggregate (sum, max) |

---

## 5. Method References

A method reference is **shorthand for a lambda that just calls an existing method**. Syntax uses `::`.

### 4 Types

**1. Reference to a static method**
```java
// Lambda: (a, b) -> Integer.compare(a, b)
BiFunction<Integer, Integer, Integer> compare = Integer::compare;

Function<String, Integer> parse = Integer::parseInt; // s -> Integer.parseInt(s)
```

**2. Reference to an instance method of a particular object**
```java
String prefix = "Hello, ";
// Lambda: s -> prefix.concat(s)
Function<String, String> greeter = prefix::concat;
System.out.println(greeter.apply("World")); // Hello, World

List<String> list = new ArrayList<>();
Consumer<String> adder = list::add; // s -> list.add(s)
```

**3. Reference to an instance method of an arbitrary object of a particular type**
```java
// Lambda: (s1, s2) -> s1.compareTo(s2)
Comparator<String> comp = String::compareTo;

Function<String, Integer> len = String::length; // s -> s.length()

// Common use in streams
List<String> names = List.of("bob", "alice", "eve");
names.stream().map(String::toUpperCase).forEach(System.out::println);
```

**4. Reference to a constructor**
```java
Supplier<ArrayList<String>> listSupplier = ArrayList::new; // () -> new ArrayList<>()
ArrayList<String> newList = listSupplier.get();

Function<String, StringBuilder> sbCreator = StringBuilder::new; // s -> new StringBuilder(s)

// Common use: converting stream of data to objects
Function<String, Person> personCreator = Person::new; // constructor taking a String name
List<Person> people = names.stream().map(Person::new).collect(Collectors.toList());
```

### Comparison Table

| Type | Syntax | Lambda Equivalent |
|---|---|---|
| Static method | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` |
| Instance method (specific object) | `object::instanceMethod` | `(args) -> object.instanceMethod(args)` |
| Instance method (arbitrary object) | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` |
| Constructor | `ClassName::new` | `(args) -> new ClassName(args)` |

**Interview tip:** Method references and constructor references are just **syntactic sugar** over lambdas — the compiler converts them to the same target functional interface. They improve readability when the lambda body is literally just delegating to an existing method.

---

## 6. Common Interview Questions (with crisp answers)

**Q1: Why were lambdas introduced in Java 8?**
To support functional programming, reduce boilerplate code for single-method interfaces, and enable the Stream API for declarative, parallelizable data processing.

**Q2: Can a lambda expression throw a checked exception?**
Only if the functional interface's abstract method declares that checked exception in its `throws` clause. Standard interfaces like `Function`, `Consumer` don't declare checked exceptions, so you must wrap them in try-catch or use a custom functional interface.

**Q3: Difference between `Function` and `UnaryOperator`?**
`UnaryOperator<T>` extends `Function<T,T>` — used specifically when input and output types are the same. Prefer `UnaryOperator` for semantic clarity (e.g., `List.replaceAll` uses it).

**Q4: Why does `BiFunction` have `andThen()` but not `compose()`?**
`compose()` would require accepting a function first and applying it to produce two inputs — not logically composable the same way as `Function.compose()`, which takes one input. `andThen()` works because it just transforms the single result.

**Q5: Are lambdas objects at runtime?**
Yes conceptually — the JVM uses `invokedynamic` and generates the implementation lazily at runtime via `LambdaMetafactory`, rather than generating a class file at compile time (unlike anonymous classes). This can be more memory-efficient.

**Q6: Can you use a lambda to implement an interface with 2 abstract methods?**
No. Lambdas require a functional interface (exactly one abstract method). Use an anonymous class instead.

**Q7: What is target typing?**
The Java compiler infers the type of a lambda expression from the context it's used in (the functional interface it's being assigned to or passed as). The same lambda `(a, b) -> a + b` could be a `BinaryOperator<Integer>` or a custom interface, depending on context.

**Q8: How do you write a custom functional interface with 3 parameters (since BiFunction only supports 2)?**
```java
@FunctionalInterface
interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}
TriFunction<Integer, Integer, Integer, Integer> add3 = (a, b, c) -> a + b + c;
```

**Q9: Practical difference: Predicate vs Function<T, Boolean>?**
Both can express a boolean check, but `Predicate<T>` is primitive (no autoboxing) via its `test()` method and comes with `and()/or()/negate()` for combining conditions — designed specifically for filtering logic. `Function<T, Boolean>` involves boxing overhead and lacks these combinators.

**Q10: Real-world usage combining several of these interfaces:**
```java
List<String> names = List.of("john", "jane", "bob", "alice");

names.stream()
     .filter(name -> name.length() > 3)       // Predicate
     .map(String::toUpperCase)                // Function / Method reference
     .sorted(String::compareTo)                // Comparator via method reference
     .forEach(System.out::println);            // Consumer / Method reference
```

---

## 7. Quick Cheat Sheet Summary

```
Predicate<T>        : T -> boolean         test()
Function<T,R>       : T -> R               apply()
Consumer<T>         : T -> void            accept()
Supplier<T>         : () -> T              get()
BiPredicate<T,U>     : (T,U) -> boolean     test()
BiFunction<T,U,R>    : (T,U) -> R           apply()
BiConsumer<T,U>      : (T,U) -> void        accept()
UnaryOperator<T>     : T -> T               apply()   (extends Function<T,T>)
BinaryOperator<T>    : (T,T) -> T           apply()   (extends BiFunction<T,T,T>)
```

Good luck with your interviews!