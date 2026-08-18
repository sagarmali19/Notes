# Java Generics — Interview-Ready Notes

## 1. Why Generics? (The Problem Before Java 5)

Before Generics (introduced in **Java 5**), collections like `ArrayList` stored everything as `Object`:

```java
ArrayList list = new ArrayList();   // raw type
list.add("Hello");
list.add(10);
list.add(3.14);

String s = (String) list.get(0);    // manual cast required
```

**Two core problems this caused:**

| Problem | Explanation |
|---|---|
| **No type safety** | Any type of object could be added — Strings, Integers, Doubles all mixed together with no compile-time check. |
| **Manual casting required** | Since `get()` returns `Object`, you must cast it back to the real type yourself. If you cast wrong (e.g., cast an `Integer` to `String`), you get a `ClassCastException` — but only **at runtime**, not compile time. |

```java
list.add(10);                       // Integer added
String s2 = (String) list.get(0);   // compiles fine...
// Runtime: java.lang.ClassCastException: Integer cannot be cast to String
```

👉 **Generics solve this by moving the type-check from runtime to compile time**, and by removing the need for manual casting.

```java
ArrayList<String> list = new ArrayList<>();
list.add("Hello");
// list.add(10);   ❌ compile-time error now, not runtime
String s = list.get(0);  // no cast needed
```

**Interview one-liner:** *"Generics provide compile-time type safety and eliminate the need for explicit casting, catching type mismatches early instead of throwing `ClassCastException` at runtime."*

---

## 2. What Are Generics?

> Generics allow you to define a **class, interface, or method** with **type parameters** (placeholders) for the data types they will operate on. The actual type is supplied when the class/interface is used or the method is called.

Syntax uses **angle brackets `< >`**.

---

## 3. Generic Classes

### Syntax
```java
class ClassName<T> {
    // T acts as a placeholder for a type, decided at object creation
}
```

### Example — without Generics (problem)
```java
class Box {
    private Object value;
    public void setValue(Object value) { this.value = value; }
    public Object getValue() { return value; }
}

Box box = new Box();
box.setValue(10);
String s = (String) box.getValue();  // compiles, but throws ClassCastException at runtime
```

### Example — with Generics (solution)
```java
class Box<T> {
    private T value;
    public void setValue(T value) { this.value = value; }
    public T getValue() { return value; }
}

Box<Integer> box = new Box<>();
box.setValue(10);
int i = box.getValue();     // no cast needed, type-safe
// box.setValue("Hello");   ❌ compile-time error
```

**Naming convention (not mandatory, just convention):**
- `T` → Type
- `E` → Element (used in collections)
- `K`, `V` → Key, Value (used in maps/pairs)
- `N` → Number

### Multiple type parameters — Pair example
```java
class Pair<K, V> {
    private K key;
    private V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }
    public K getKey()   { return key; }
    public V getValue() { return value; }
}

Pair<String, Integer> p = new Pair<>("age", 30);
System.out.println(p.getKey() + " " + p.getValue());  // age 30
```

---

## 4. Generic Interfaces

### Syntax
```java
interface Container<T> {
    T get();
    void set(T item);
}
```

Two ways to implement a generic interface:

**a) Implementing class specifies a concrete type:**
```java
class StringContainer implements Container<String> {
    private String item;
    public void set(String item) { this.item = item; }
    public String get() { return item; }
}
```

**b) Implementing class stays generic (propagates the type parameter):**
```java
class GenericContainer<T> implements Container<T> {
    private T item;
    public void set(T item) { this.item = item; }
    public T get() { return item; }
}
```

---

## 5. Bounded Type Parameters

Restrict what types `T` can accept using `extends` (works for both classes **and** interfaces — Java uses `extends` for both in bounds, not `implements`).

### Single bound
```java
class Box<T extends Number> {   // T must be Number or a subclass (Integer, Double, Float...)
    private T value;
    public void setValue(T value) { this.value = value; }
    public T getValue() { return value; }
}

Box<Integer> b1 = new Box<>();   // ✅ OK
// Box<String> b2 = new Box<>(); ❌ compile error — String doesn't extend Number
```

### Multiple bounds
```java
// Rule: class must come first, then interfaces — separated by &
class Box<T extends Number & Printable> {
    T item;
}
```
- First bound → must be a **class**
- Remaining bounds → must be **interfaces**
- Even for interface bounds, keyword used is still `extends` (never `implements`)

**Example:**
```java
interface Printable {
    void print();
}

class MyNumber extends Number implements Printable {
    private double val;
    MyNumber(double val) { this.val = val; }
    public void print() { System.out.println("MyNumber " + val); }
    public int intValue()    { return (int) val; }
    public long longValue()  { return (long) val; }
    public float floatValue(){ return (float) val; }
    public double doubleValue(){ return val; }
}

class Box<T extends Number & Printable> {
    T item;
    Box(T item) { this.item = item; }
}

Box<MyNumber> box = new Box<>(new MyNumber(12));
```

---

## 6. Generics in Enums

Enums are **inherently type-safe** already — you cannot assign a random string/int to an enum variable.

```java
enum Day { MONDAY, TUESDAY, WEDNESDAY }

Day d = Day.MONDAY;   // ✅ OK
// Day d = "MONDAY";  ❌ compile-time error automatically — no generics needed here
```
> Note: While enums *can* be combined with generics in advanced designs (e.g., `Enum<E extends Enum<E>>` internally), for interview purposes the key point is: **enums achieve type safety by design, without you writing any generic syntax.**

---

## 7. Generic Constructors

A class itself doesn't have to be generic — only its **constructor** can be generic.

```java
public class Box2 {                       // class is NOT generic
    public <T> Box2(T value) {            // constructor IS generic
        System.out.println("Constructor received: " + value);
    }
}

Box2 b1 = new Box2("Hello");   // T = String
Box2 b2 = new Box2(123);       // T = Integer
```
The type parameter `<T>` is placed **between the access modifier and the constructor name**, scoping it only to the constructor.

---

## 8. Generic Methods

### Syntax
```java
modifier <T> returnType methodName(T param) { ... }
```
The `<T>` **must** appear before the return type, or it's a compile error.

### Example
```java
class Test {
    static <T> void printArray(T[] array) {
        for (T item : array) {
            System.out.println(item);
        }
    }

    public static void main(String[] args) {
        Integer[] intArr = {1, 2, 3};
        String[] strArr = {"Hello", "World"};
        printArray(intArr);
        printArray(strArr);
    }
}
```

### Generic method + bound (real use case)
```java
enum Operation {
    ADD, SUBTRACT, MULTIPLY, DIVIDE;

    public <T extends Number> double apply(T a, T b) {
        double x = a.doubleValue();
        double y = b.doubleValue();
        switch (this) {
            case ADD: return x + y;
            case SUBTRACT: return x - y;
            case MULTIPLY: return x * y;
            case DIVIDE: return x / y;
        }
        return 0;
    }
}

double result = Operation.ADD.apply(10, 20);       // 30.0
double result2 = Operation.MULTIPLY.apply(10, 20); // 200.0
```

### Method overloading using generics
```java
static <T> void display(T value)        { System.out.println("Generic: " + value); }
static void display(Integer value)      { System.out.println("Integer: " + value); }
```
Java will pick the more specific (`Integer`) overload when an `Integer` is passed, and fall back to the generic version otherwise.

---

## 9. Wildcards (`?`)

Used to represent an **unknown type**, mainly:
- As a **method argument type**
- When defining a class reference

### a) Unbounded Wildcard `<?>`
Use when you only need **read-only** access and don't care about the exact type.

```java
public void printList(List<?> list) {
    for (Object o : list) {
        System.out.println(o);
    }
}
```
- ❌ You **cannot add** elements (except `null`) because the compiler doesn't know the real type.
- ✅ You can only read as `Object`.

```java
List<?> list = new ArrayList<String>();
// list.add("hi");  ❌ compile error
list.add(null);      // ✅ only null allowed
```

### b) Upper Bounded Wildcard `<? extends Type>`
Restricts to `Type` and its **subclasses**. Good for **reading** (producer of data).

```java
public static double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) {
        total += n.doubleValue();
    }
    return total;
}

sum(Arrays.asList(1, 2, 3));         // works with List<Integer>
sum(Arrays.asList(1.5, 2.5));        // works with List<Double>
```
- You **cannot add** elements here either (except `null`) — because the compiler doesn't know if it's `List<Integer>`, `List<Double>`, etc.

### c) Lower Bounded Wildcard `<? super Type>`
Restricts to `Type` and its **superclasses**. Good for **writing** (consumer of data).

```java
public static void addNumbers(List<? super Integer> list) {
    list.add(10);   // ✅ allowed — Integer or its subtypes can safely go in
    list.add(20);
}

List<Number> numbers = new ArrayList<>();
addNumbers(numbers);   // works — Number is a superclass of Integer
```

### PECS Rule (important interview mnemonic)
> **P**roducer **E**xtends, **C**onsumer **S**uper

| Wildcard | Use case | Can Read? | Can Write? |
|---|---|---|---|
| `<?>` | Unknown/read-only | ✅ (as Object) | ❌ (except null) |
| `<? extends T>` | You only **read** from it (producer) | ✅ (as T) | ❌ (except null) |
| `<? super T>` | You only **write** to it (consumer) | ✅ (as Object) | ✅ (T and subtypes) |

---

## 10. Type Erasure (How Generics Work Internally)

Generics are a **compile-time only** feature. At compile time:
1. The compiler **checks type correctness** using the generic type info.
2. After checking, it **erases** the generic type information and replaces it:
    - Unbounded `T` → replaced with `Object`
    - Bounded `T extends Number` → replaced with `Number` (the bound)
3. Necessary **casts are inserted automatically** by the compiler in the bytecode.

### Example
**Source code:**
```java
class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
```

**After erasure (conceptually, what bytecode looks like):**
```java
class Box {
    private Object value;
    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}
```

With a bound:
```java
class NumberBox<T extends Number> { T value; }
// erased to:
class NumberBox { Number value; }
```

**Interview one-liner:** *"Generics exist only at compile time for type checking; the JVM never sees generic types at runtime — this is called Type Erasure. It's why you can't create generic arrays, can't use `instanceof` with a parameterized type, and can't overload methods that differ only by generic type."*

---

## 11. Generic Exceptions — Why They're Not Allowed

```java
class GenericException<T> extends Exception { }  // ❌ COMPILE ERROR
```
**Error:** *"generic class may not extend java.lang.Throwable"*

### Why?
- Exceptions are thrown/caught at **runtime** by the JVM.
- But due to **Type Erasure**, generic type info doesn't exist at runtime.
- The JVM needs the **exact class** to create/catch exception objects — it can't work with an erased/unknown type.
- So Java explicitly **disallows a class extending `Throwable` (directly or indirectly) from being generic.**

### Workaround 1 — Create type-specific exception classes
```java
class StringProcessingException extends Exception { }
class IntegerProcessingException extends Exception { }
```

### Workaround 2 — Non-generic exception class, but with a generic constructor
```java
class MyException extends Exception {
    <T> MyException(T data) {
        super(data.toString() + " -> " + data.getClass().getName());
    }
}

try {
    throw new MyException(123);       // works — constructor is generic, class isn't
} catch (MyException e) {
    System.out.println(e.getMessage());
}

try {
    throw new MyException("Error string");
} catch (MyException e) {
    System.out.println(e.getMessage());
}
```
This works because the **class itself is a plain, concrete `Exception` subclass** (so the JVM can handle it normally) — only the **constructor** takes a generic parameter, which is fine since constructors run and resolve types before erasure removes generic info from usage points.

---

## 12. Quick-Fire Interview Q&A

**Q1: When were Generics introduced in Java?**
→ Java 5.

**Q2: What two problems do Generics solve?**
→ Type safety (compile-time checking) and eliminating manual/explicit casting.

**Q3: What is Type Erasure?**
→ The process by which the compiler removes generic type information after compile-time type checking, replacing type parameters with `Object` (or their bound) in the bytecode.

**Q4: Why can't you create a generic exception class?**
→ Because exceptions are resolved by the JVM at runtime, but generic type information is erased at runtime (Type Erasure) — so the JVM cannot know the actual type to construct/catch.

**Q5: What's the difference between `<? extends T>` and `<? super T>`?**
→ `extends` = upper bound, used when reading/producing data (Producer Extends). `super` = lower bound, used when writing/consuming data (Consumer Super). Together: **PECS**.

**Q6: Can a class be non-generic but have a generic constructor or method?**
→ Yes. The `<T>` is placed just before the constructor/method's return type to scope it locally, independent of the class declaration.

**Q7: In bounded type parameters with multiple bounds, what's the rule on ordering?**
→ The class (if any) must come first, followed by interfaces, all joined with `&`. Example: `<T extends Number & Comparable<T>>`.

**Q8: Why does raw-type usage (`List list = new ArrayList()`) give a warning, not an error?**
→ For backward compatibility with pre-Java-5 code; it's discouraged but legal, hence a warning rather than a compile error.

**Q9: What does the diamond operator `<>` do?**
→ Lets you omit the repeated type on the right-hand side since Java 7 (type inference): `List<String> list = new ArrayList<>();`

**Q10: Can you use primitive types as generic type arguments?**
→ No. Generics work only with reference types (use wrapper classes: `Integer`, `Double`, etc., not `int`, `double`).

---

## 13. One-Page Summary Table

| Concept | Key Point |
|---|---|
| Generic class | `class Box<T> { }` |
| Generic interface | `interface Container<T> { }` |
| Bounded type | `<T extends Number>` |
| Multiple bounds | `<T extends Number & Printable>` (class first, then interfaces) |
| Generic constructor | `<T> ClassName(T val) { }` — class itself may be non-generic |
| Generic method | `static <T> void method(T val) { }` — `<T>` before return type |
| Unbounded wildcard | `List<?>` — read-only |
| Upper bounded wildcard | `List<? extends T>` — producer, read-only |
| Lower bounded wildcard | `List<? super T>` — consumer, can write |
| PECS | Producer **extends**, Consumer **super** |
| Type erasure | Generic info removed after compile-time check; replaced by `Object`/bound |
| Generic exceptions | Not allowed — class can't extend `Throwable` and be generic; use generic constructors or type-specific exception classes instead |