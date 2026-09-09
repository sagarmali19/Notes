# Object-Oriented Programming (OOP) in Core Java — Interview-Ready Notes

---

## 1. What is OOP? Why does it exist?

**OOP (Object-Oriented Programming)** is a programming paradigm based on the concept of **"objects"** — real-world entities that bundle **data (fields)** and **behavior (methods)** together.

Before OOP, procedural languages (like C) separated data and functions, which made large codebases hard to maintain, extend, and reuse. OOP solves this by modeling code around real-world entities.

### The 4 Pillars of OOP
| Pillar | One-line meaning |
|---|---|
| **Encapsulation** | Hiding internal data, exposing controlled access |
| **Inheritance** | Reusing/extending behavior of another class |
| **Polymorphism** | One interface, many implementations |
| **Abstraction** | Hiding implementation details, showing only essential features |

> 💡 **Interview Tip:** Interviewers almost always ask "Explain OOP concepts with example" as an opener. Memorize a one-liner + a 3-line code snippet for each pillar — that's exactly what this document gives you below.

---

## 2. Classes and Objects

A **class** is a blueprint/template. An **object** is a real instance created from that blueprint, occupying memory (on the heap).

```java
class Car {
    // fields (state)
    String brand;
    int speed;

    // method (behavior)
    void accelerate() {
        speed += 10;
        System.out.println(brand + " is now at " + speed + " km/h");
    }
}

public class Main {
    public static void main(String[] args) {
        Car car1 = new Car();      // object creation
        car1.brand = "Toyota";
        car1.speed = 0;
        car1.accelerate();         // Toyota is now at 10 km/h

        Car car2 = new Car();      // a completely separate object
        car2.brand = "Honda";
        car2.accelerate();         // Honda is now at 10 km/h
    }
}
```

**Key points:**
- `class` = compile-time concept (no memory allocated until an object is created).
- `object` = runtime entity created using the `new` keyword.
- Each object has its **own copy** of instance variables but **shares** the same method code.

---

## 3. Encapsulation

**Definition:** Wrapping data (variables) and code (methods) together as a single unit, and restricting direct access to the data using `private` fields + public `getters/setters`.

**Why?** Protects internal state from invalid/unauthorized modification, and allows validation logic to be centralized.

```java
class BankAccount {
    private double balance;   // hidden from outside world

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount > 0) {          // validation logic lives here
            balance += amount;
        } else {
            System.out.println("Invalid deposit amount");
        }
    }

    public void withdraw(double amount) {
        if (amount > balance) {
            System.out.println("Insufficient funds");
        } else {
            balance -= amount;
        }
    }
}

public class Main {
    public static void main(String[] args) {
        BankAccount acc = new BankAccount();
        acc.deposit(1000);
        // acc.balance = -5000;  // ❌ Not possible — compile error, balance is private
        acc.withdraw(200);
        System.out.println(acc.getBalance()); // 800.0
    }
}
```

> 💡 **Interview one-liner:** "Encapsulation = data hiding + controlled access via getters/setters. It achieves loose coupling and protects data integrity."

**POJO / JavaBean** is a real-world example of encapsulation — private fields + public getters/setters + no-arg constructor.

---

## 4. Inheritance

**Definition:** Mechanism where one class (**subclass/child**) acquires the properties and behavior of another class (**superclass/parent**) using the `extends` keyword.

**Why?** Code reusability and establishing an "IS-A" relationship.

```java
class Animal {
    String name;

    void eat() {
        System.out.println(name + " is eating");
    }
}

class Dog extends Animal {     // Dog IS-A Animal
    void bark() {
        System.out.println(name + " is barking");
    }
}

public class Main {
    public static void main(String[] args) {
        Dog d = new Dog();
        d.name = "Tommy";
        d.eat();   // inherited from Animal -> Tommy is eating
        d.bark();  // defined in Dog        -> Tommy is barking
    }
}
```

### Types of Inheritance in Java

| Type | Supported directly? | Example |
|---|---|---|
| Single | ✅ | `class B extends A` |
| Multilevel | ✅ | `class C extends B extends A` |
| Hierarchical | ✅ | `class B extends A`, `class C extends A` |
| Multiple (via classes) | ❌ NOT supported | Would cause the "Diamond Problem" |
| Multiple (via interfaces) | ✅ | `class D implements X, Y` |

### Why doesn't Java support multiple inheritance with classes?
To avoid the **Diamond Problem** — ambiguity when two parent classes have a method with the same signature and the compiler can't decide which one to inherit.

```java
// Hypothetical (NOT valid Java) — shown only to explain WHY it's disallowed
class A { void show() { System.out.println("A"); } }
class B { void show() { System.out.println("B"); } }
// class C extends A, B { }   // ❌ Compiler wouldn't know which show() to use
```

Java solves this ambiguity for interfaces using **default methods** (Java 8+) — if a class implements two interfaces with the same default method, it is **forced to override** it explicitly.

### The `super` keyword

Used to refer to the immediate parent class — to access parent fields, methods, or constructor.

```java
class Animal {
    String name = "Animal";
    void sound() { System.out.println("Some generic sound"); }
}

class Cat extends Animal {
    String name = "Cat";

    void sound() {
        super.sound();                         // calls parent's method
        System.out.println("Meow");
    }

    void printNames() {
        System.out.println(name);              // Cat  (this class's field)
        System.out.println(super.name);        // Animal (parent's field)
    }
}
```

---

## 5. Polymorphism

**Definition:** "Many forms" — the same method/operator behaves differently based on context. Two types:

### 5.1 Compile-Time Polymorphism (Method Overloading)
Same method name, **different parameter list** (number/type/order), resolved at **compile time**. Also called **static binding**.

```java
class Calculator {
    int add(int a, int b) {
        return a + b;
    }

    double add(double a, double b) {     // different parameter type
        return a + b;
    }

    int add(int a, int b, int c) {       // different number of params
        return a + b + c;
    }
}

public class Main {
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        System.out.println(calc.add(2, 3));         // 5      -> int version
        System.out.println(calc.add(2.5, 3.5));      // 6.0    -> double version
        System.out.println(calc.add(1, 2, 3));       // 6      -> 3-arg version
    }
}
```

> ⚠️ **Important:** You **cannot** overload a method just by changing the **return type**. Parameters must differ.

### 5.2 Run-Time Polymorphism (Method Overriding)
Subclass provides a **specific implementation** of a method already defined in its parent class. Resolved at **runtime** based on the actual object type. Also called **dynamic binding** or **dynamic method dispatch**.

```java
class Shape {
    double area() {
        return 0;
    }
}

class Circle extends Shape {
    double radius;
    Circle(double radius) { this.radius = radius; }

    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}

class Rectangle extends Shape {
    double length, width;
    Rectangle(double l, double w) { length = l; width = w; }

    @Override
    double area() {
        return length * width;
    }
}

public class Main {
    public static void main(String[] args) {
        Shape s1 = new Circle(5);        // Parent reference, Child object
        Shape s2 = new Rectangle(4, 6);

        System.out.println(s1.area());   // 78.53... -> Circle's area() called
        System.out.println(s2.area());   // 24.0     -> Rectangle's area() called
    }
}
```

**Rules for Method Overriding:**
1. Method signature (name + parameters) must be **exactly same**.
2. Return type must be same or a **covariant type** (subtype).
3. Access modifier **cannot be more restrictive** (can widen, e.g., `protected` → `public`, but not the reverse).
4. Cannot override `static`, `final`, or `private` methods.
5. Use `@Override` annotation — not mandatory, but a best practice (catches errors at compile time).

### Overloading vs Overriding — Classic Interview Table

| Feature | Overloading | Overriding |
|---|---|---|
| Binding | Compile-time (static) | Runtime (dynamic) |
| Class relation | Same class | Parent-child (inheritance) |
| Parameters | Must differ | Must be exactly same |
| Return type | Can differ | Must be same/covariant |
| `private`/`static`/`final` methods | Can be overloaded | Cannot be overridden |

---

## 6. Abstraction

**Definition:** Hiding *how* something is done and showing only *what* is done. Achieved via **abstract classes** and **interfaces** in Java.

### 6.1 Abstract Class

```java
abstract class Vehicle {
    abstract void start();      // abstract method — no body, must be implemented by subclass

    void fuel() {                // concrete method — can have implementation
        System.out.println("Refueling vehicle");
    }
}

class Bike extends Vehicle {
    @Override
    void start() {
        System.out.println("Bike started with a kick");
    }
}

public class Main {
    public static void main(String[] args) {
        // Vehicle v = new Vehicle();  // ❌ Cannot instantiate abstract class
        Vehicle v = new Bike();        // ✅ Allowed via reference
        v.start();                     // Bike started with a kick
        v.fuel();                      // Refueling vehicle
    }
}
```

**Rules:**
- Declared using the `abstract` keyword.
- Can have both abstract (no body) and concrete (with body) methods.
- Cannot be instantiated directly.
- Can have constructors, static methods, final methods.
- A class with even **one** abstract method **must** be declared abstract.

### 6.2 Interface

```java
interface Payable {
    double TAX_RATE = 0.18;              // implicitly public static final
    void pay(double amount);             // implicitly public abstract

    default void printReceipt() {        // default method (Java 8+)
        System.out.println("Receipt printed");
    }

    static void info() {                 // static method (Java 8+)
        System.out.println("Payable interface v1.0");
    }
}

class CreditCardPayment implements Payable {
    @Override
    public void pay(double amount) {
        System.out.println("Paid " + amount + " via Credit Card");
    }
}

public class Main {
    public static void main(String[] args) {
        Payable p = new CreditCardPayment();
        p.pay(500);              // Paid 500.0 via Credit Card
        p.printReceipt();        // Receipt printed (default method)
        Payable.info();          // Payable interface v1.0 (static method)
    }
}
```

**Interface Evolution across Java versions (frequently asked!):**
- **Before Java 8:** Only `public static final` variables and `public abstract` methods.
- **Java 8:** Added `default` and `static` methods (to allow interface evolution without breaking existing implementations).
- **Java 9:** Added `private` methods (helper methods for default methods, not exposed outside).

### Abstract Class vs Interface

| Feature | Abstract Class | Interface |
|---|---|---|
| Methods | Abstract + concrete | Abstract, default, static, private (Java 9+) |
| Variables | Any type (instance vars allowed) | Only `public static final` (constants) |
| Multiple inheritance | Not supported (single class inheritance) | A class can implement multiple interfaces |
| Constructor | Can have one | Cannot have one |
| Access modifiers | Any (`private`, `protected`, `public`) | Methods are `public` by default |
| When to use | When classes share **common state/code** | When unrelated classes need to share **a contract/capability** |

> 💡 **Interview one-liner:** "Use an abstract class for an IS-A relationship with shared code; use an interface for a CAN-DO capability (like `Comparable`, `Runnable`)."

---

## 7. Constructors

A **constructor** initializes an object when it's created. Same name as class, no return type.

```java
class Student {
    String name;
    int age;

    // 1. Default constructor (no-arg)
    Student() {
        name = "Unknown";
        age = 0;
    }

    // 2. Parameterized constructor
    Student(String name, int age) {
        this.name = name;   // 'this' resolves ambiguity between field and parameter
        this.age = age;
    }

    // 3. Copy constructor (Java has no built-in one, we write our own)
    Student(Student s) {
        this.name = s.name;
        this.age = s.age;
    }
}

public class Main {
    public static void main(String[] args) {
        Student s1 = new Student();                 // Unknown, 0
        Student s2 = new Student("Riya", 21);        // Riya, 21
        Student s3 = new Student(s2);                // copy of s2 -> Riya, 21
    }
}
```

### Constructor Chaining — `this()` and `super()`

```java
class Person {
    String name;
    Person() {
        this("Default Name");     // calls the parameterized constructor below
        System.out.println("Person() called");
    }
    Person(String name) {
        this.name = name;
        System.out.println("Person(String) called");
    }
}

class Employee extends Person {
    Employee() {
        super("Employee Default"); // must be the FIRST statement; calls Person(String)
        System.out.println("Employee() called");
    }
}

public class Main {
    public static void main(String[] args) {
        new Employee();
        // Output:
        // Person(String) called
        // Employee() called
    }
}
```

**Key Rules:**
- `this()` and `super()` **must be the first statement** in a constructor.
- You cannot use both `this()` and `super()` in the same constructor.
- If you don't explicitly call `super()`, Java **automatically inserts** a call to the no-arg `super()` at the start.
- Constructors are **not inherited**, but a subclass can call a superclass's constructor via `super()`.

---

## 8. `static` Keyword

`static` members belong to the **class**, not to individual objects — shared across all instances.

```java
class Counter {
    static int count = 0;    // static/class variable — shared by all objects

    Counter() {
        count++;             // increments the shared copy
    }

    static void showCount() {          // static method
        System.out.println("Count: " + count);
        // 'this' cannot be used here — no object context
    }

    static {                            // static block — runs ONCE when class loads
        System.out.println("Class loaded into memory");
    }
}

public class Main {
    public static void main(String[] args) {
        new Counter();
        new Counter();
        new Counter();
        Counter.showCount();     // Count: 3 (called using class name, not object)
    }
}
```

**Static vs Instance:**

| | Static | Instance |
|---|---|---|
| Belongs to | Class | Object |
| Memory | Allocated once (Method/Metaspace area) | Allocated per object (Heap) |
| Access | `ClassName.member` | `object.member` |
| Can access instance members? | ❌ No (without an object reference) | ✅ Yes |

---

## 9. `this` Keyword

Refers to the **current object instance**. Common uses:

```java
class Box {
    int side;

    Box(int side) {
        this.side = side;        // 1. Distinguish field from parameter
    }

    Box() {
        this(10);                 // 2. Constructor chaining
    }

    Box grow() {
        this.side += 5;
        return this;              // 3. Return current object (enables method chaining)
    }
}

public class Main {
    public static void main(String[] args) {
        Box b = new Box();
        b.grow().grow();          // method chaining because grow() returns 'this'
        System.out.println(b.side); // 20
    }
}
```

---

## 10. Access Modifiers

Control the **visibility/scope** of classes, methods, and variables.

| Modifier | Same Class | Same Package | Subclass (different package) | Different Package |
|---|---|---|---|---|
| `private` | ✅ | ❌ | ❌ | ❌ |
| default (no modifier) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
class Demo {
    private int a = 1;      // only within Demo
    int b = 2;               // package-private (default)
    protected int c = 3;     // package + subclasses
    public int d = 4;        // everywhere
}
```

> 💡 **Interview tip:** `private` is most commonly used for fields (encapsulation), `public` for methods meant to be an API/contract, `protected` for members meant to be overridden by subclasses.

---

## 11. `final` Keyword

`final` can be applied to variables, methods, and classes — each meaning something different.

```java
final class Constants {              // 1. final class -> cannot be extended/inherited
    // class MyConst extends Constants {}   // ❌ Compile error
}

class Vehicle {
    final int wheels = 4;            // 2. final variable -> value cannot be reassigned
    // wheels = 6;  // ❌ Compile error

    final void run() {               // 3. final method -> cannot be overridden
        System.out.println("Running");
    }
}

// class Car extends Vehicle {
//     void run() { }   // ❌ Compile error - cannot override final method
// }
```

**Interview tip:** `final` reference variables (for objects) mean the **reference** can't change, but the object's internal state **can still be mutated** (unless the object itself is immutable, like `String`).

```java
final StringBuilder sb = new StringBuilder("Hi");
sb.append(" there");    // ✅ Allowed — modifying internal state
// sb = new StringBuilder("New");  // ❌ Not allowed — reassigning reference
```

---

## 12. `Object` Class Methods — `equals()`, `hashCode()`, `toString()`

Every class in Java implicitly extends `Object`. Three commonly overridden methods:

```java
class Employee {
    int id;
    String name;

    Employee(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "'}";
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Employee)) return false;
        Employee e = (Employee) obj;
        return id == e.id && name.equals(e.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}

import java.util.Objects;
public class Main {
    public static void main(String[] args) {
        Employee e1 = new Employee(1, "Amit");
        Employee e2 = new Employee(1, "Amit");

        System.out.println(e1);              // uses toString() -> Employee{id=1, name='Amit'}
        System.out.println(e1 == e2);         // false -> different references
        System.out.println(e1.equals(e2));    // true  -> logically equal (custom equals)
    }
}
```

**The `equals()`/`hashCode()` Contract (VERY frequently asked):**
1. If two objects are **equal** (via `equals()`), they **must** have the same `hashCode()`.
2. The reverse is **not** guaranteed — equal hash codes don't mean the objects are equal (hash collisions).
3. If you override `equals()`, you **must** override `hashCode()` too — otherwise objects behave inconsistently in `HashMap`/`HashSet`.

`==` vs `equals()`:
- `==` compares **references** (memory addresses) for objects, and **values** for primitives.
- `equals()` compares **logical/content equality** (if overridden).

---

## 13. Association, Aggregation, and Composition (Object Relationships)

These describe **HAS-A** relationships (as opposed to inheritance's IS-A relationship).

### Association
A general relationship where objects know about each other, but have **independent lifecycles**.
```java
class Teacher { String name; }
class Student {
    Teacher teacher;   // Student is "associated with" a Teacher
}
```

### Aggregation (Weak HAS-A)
A "whole-part" relationship where the **part can exist independently** of the whole.
```java
class Department {
    String name;
}
class Professor {
    Department dept;    // Professor "has-a" Department, but Department can exist without a specific Professor
    Professor(Department dept) { this.dept = dept; }
}
```

### Composition (Strong HAS-A)
A "whole-part" relationship where the **part CANNOT exist without the whole** — the child's lifecycle depends on the parent.
```java
class Engine {
    void start() { System.out.println("Engine starting..."); }
}
class Car {
    private final Engine engine = new Engine();  // Engine is created INSIDE Car; cannot exist independently

    void drive() {
        engine.start();
        System.out.println("Car is moving");
    }
}
```

> 💡 **Interview one-liner:** "Aggregation = weak ownership (Department outlives Professor). Composition = strong ownership (Engine dies with the Car). Both are types of Association."

---

## 14. Inner Classes (Nested Classes)

| Type | Description |
|---|---|
| **Static nested class** | Declared `static` inside a class; doesn't need outer instance |
| **Inner class (non-static)** | Needs an instance of the outer class to exist |
| **Local class** | Defined inside a method body |
| **Anonymous class** | No name; used for one-time implementation, often of an interface |

```java
class Outer {
    int outerField = 10;

    static class StaticNested {         // doesn't need Outer instance
        void show() { System.out.println("Static nested class"); }
    }

    class Inner {                        // needs Outer instance
        void show() { System.out.println("Outer field: " + outerField); }
    }
}

public class Main {
    public static void main(String[] args) {
        Outer.StaticNested sn = new Outer.StaticNested();
        sn.show();

        Outer outer = new Outer();
        Outer.Inner inner = outer.new Inner();   // note the syntax
        inner.show();

        // Anonymous class example
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println("Running anonymously");
            }
        };
        r.run();
    }
}
```

---

## 15. Object Cloning

Creating an exact copy of an object using the `Cloneable` interface and `clone()` method.

```java
class Address implements Cloneable {
    String city;
    Address(String city) { this.city = city; }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // Shallow copy (default clone() behavior)
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    // Deep copy (manual)
    protected Person deepClone() {
        return new Person(this.name, new Address(this.address.city));
    }
}
```

- **Shallow copy:** Copies field values as-is; if a field is an object reference, both original and clone point to the **same** referenced object.
- **Deep copy:** Recursively copies referenced objects too, so original and clone are **fully independent**.

---

## 16. Common Interview Q&A (Rapid Fire)

**Q1: Why is Java not 100% object-oriented?**
Because it uses primitive types (`int`, `char`, `boolean`, etc.) which are not objects.

**Q2: Can we override a static method?**
No — static methods are resolved via **method hiding**, not overriding. Binding is based on reference type, not object type (compile-time).
```java
class A { static void show() { System.out.println("A"); } }
class B extends A { static void show() { System.out.println("B"); } }
// A obj = new B(); obj.show(); -> prints "A" (based on reference type, not runtime object)
```

**Q3: Can a constructor be `private`?**
Yes — commonly used in the **Singleton design pattern** to prevent external instantiation.
```java
class Singleton {
    private static Singleton instance;
    private Singleton() {}   // private constructor

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**Q4: Difference between Abstraction and Encapsulation?**
Abstraction hides **implementation complexity** (design-level, "what to do"). Encapsulation hides **internal data** (implementation-level, "how it's protected"). Abstraction is achieved via abstract classes/interfaces; encapsulation via access modifiers.

**Q5: What is the diamond problem, and how does Java avoid it?**
Occurs with multiple inheritance when two parents have the same method, creating ambiguity. Java avoids it for classes by disallowing multiple class inheritance; for interfaces, Java 8+ forces the implementing class to explicitly override conflicting default methods.

**Q6: Is Java "pass by value" or "pass by reference"?**
Always **pass by value**. For objects, the *value of the reference* (memory address) is passed — so you can modify the object's internal state, but reassigning the parameter inside the method won't affect the original reference.
```java
void changeName(Employee e) {
    e.name = "Changed";     // ✅ affects original object
    e = new Employee(2, "New");  // ❌ does not affect the caller's reference
}
```

**Q7: What is coupling and cohesion?**
- **Coupling:** Degree of dependency between classes/modules — aim for **low coupling**.
- **Cohesion:** Degree to which elements of a single module belong together — aim for **high cohesion**.

**Q8: Can an interface extend another interface? Can it extend multiple?**
Yes to both — `interface C extends A, B {}` is valid (unlike classes, interfaces support multiple inheritance since there's no state/implementation conflict for abstract methods).

**Q9: What is the difference between IS-A and HAS-A relationships?**
IS-A = Inheritance (`Dog extends Animal`). HAS-A = Composition/Aggregation (`Car has an Engine`). Favor **"composition over inheritance"** — it's more flexible and avoids tight coupling from deep inheritance hierarchies.

**Q10: What is dynamic method dispatch?**
The mechanism by which a call to an overridden method is resolved at **runtime** based on the actual object type, not the reference type — this is how runtime polymorphism works in Java (via JVM's virtual method table).

---

## 17. Quick Revision Cheat-Sheet

- **Encapsulation** → private fields + getters/setters → data hiding
- **Inheritance** → `extends` / `implements` → code reuse, IS-A
- **Polymorphism** → Overloading (compile-time) + Overriding (runtime)
- **Abstraction** → abstract class (partial) / interface (full contract)
- **Constructors** → same name as class, no return type, chained via `this()`/`super()`
- **static** → belongs to class, shared across objects
- **final** → variable (constant), method (no override), class (no inheritance)
- **this vs super** → current object vs immediate parent
- **equals/hashCode/toString** → override together, used heavily in collections
- **Composition > Inheritance** → prefer HAS-A for flexibility

---

*End of notes. Practice explaining each pillar out loud with a code example — that's exactly how these get asked in interviews.*