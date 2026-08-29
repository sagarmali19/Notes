# Java Collections Framework — Interview Notes (Part 1: List family)

---

## 1. Collection Framework — Introduction

The **Collection Framework** is a unified architecture (interfaces + implementations + algorithms) to store and manipulate groups of objects.

**Root hierarchy** (see diagram above):

```
Iterable
   └── Collection
          ├── List   (ordered, allows duplicates)
          ├── Set    (no duplicates)
          └── Queue  (FIFO/priority processing)

Map (separate hierarchy — key/value pairs, does NOT extend Collection)
```

**Why Collection Framework exists (interview one-liner):**
Before Java 1.2, we had Array, Vector, Hashtable — no common interface, so no interoperability. Collections Framework gives:
- Unified API (`Collection`, `List`, `Set`, `Map` interfaces)
- Reduced coding effort (built-in algorithms in `Collections` class — sort, search, shuffle)
- Increased performance (well-tuned, high-performance data structure implementations)
- Interoperability between unrelated APIs

---

## 2. List Interface — Why use it?

`List<E>` is an **ordered collection** (also called a *sequence*). Key properties:
- Maintains **insertion order**
- Allows **duplicate elements**
- Elements accessed by **index** (like arrays), but **grows dynamically** unlike arrays
- Allows `null` elements (except a few implementations)

**Why List over a plain array?**

| Array | List |
|---|---|
| Fixed size | Grows/shrinks dynamically |
| Can hold primitives | Holds only objects (autoboxing for primitives) |
| No built-in utility methods | Rich API: `add()`, `remove()`, `sort()`, `search()`, `iterate()` |
| No type safety at collection level pre-generics | Generics give compile-time type safety |

**When to use List:** When you need ordered data, allow duplicates, need frequent index-based access or need to maintain insertion sequence — e.g., a list of orders, transaction history, search results.

---

## 3. ArrayList — Internal Working

`ArrayList<E>` implements `List`, backed internally by a **resizable Object array**.

```java
transient Object[] elementData; // the backing array
private int size;               // number of actual elements
```

### Key internal mechanics

1. **Default capacity**: When you do `new ArrayList<>()`, internally it does NOT allocate size-10 array immediately (since Java 7+). It points to a shared empty array `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`. Array of size 10 is allocated **lazily** on first `add()`.

2. **Growth / resizing**: When array is full and you call `add()`:
    - `newCapacity = oldCapacity + (oldCapacity >> 1)` → **1.5x growth** (NOT doubling, that's a common interview mistake!)
    - Internally uses `Arrays.copyOf()` to create a new array and copies old elements — an **O(n) operation**.

3. **Time complexity:**

| Operation | Complexity | Why |
|---|---|---|
| `get(index)` | O(1) | Direct array index access |
| `add(element)` (at end) | O(1) amortized | Occasionally O(n) during resize |
| `add(index, element)` | O(n) | Shifts elements right using `System.arraycopy` |
| `remove(index)` | O(n) | Shifts elements left to fill gap |
| `contains()` / `indexOf()` | O(n) | Linear search |

### Code Example

```java
import java.util.*;

public class ArrayListDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(); // capacity 0 initially, becomes 10 on first add

        list.add("Java");
        list.add("Spring");
        list.add("Hibernate");
        list.add(1, "SpringBoot"); // insert at index 1 -> O(n) shift

        System.out.println(list); // [Java, SpringBoot, Spring, Hibernate]

        list.remove("Spring");     // O(n) - search + shift
        list.remove(0);            // O(n) - shift

        System.out.println(list.get(0)); // O(1) access

        // Fail-fast iterator demo
        List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4));
        try {
            for (int n : nums) {
                if (n == 2) nums.remove(Integer.valueOf(2)); // structural modification during iteration
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("CME thrown - ArrayList iterator is fail-fast");
        }
    }
}
```

**Important interview points on ArrayList:**
- `ArrayList` is **not synchronized** (not thread-safe). Use `Collections.synchronizedList()` or `CopyOnWriteArrayList` for thread safety.
- Iterator is **fail-fast** — throws `ConcurrentModificationException` if the list is structurally modified (add/remove) while iterating, other than through the iterator's own `remove()`.
- Use `list.set(index, value)` — O(1) — to modify, not to `add`.

---

## 4. Comparable vs Comparator

Both are used to define **sorting logic** for custom objects, but differ significantly.

| | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Sorting logic location | Inside the class itself (natural ordering) | Outside the class — a separate class/lambda |
| Number of sort sequences | Only one (single natural order) | Multiple — as many Comparators as you want |
| How to invoke | `Collections.sort(list)` | `Collections.sort(list, comparator)` |
| Modifies original class? | Yes, class must implement it | No, class remains untouched |

**When to use Comparable vs Comparator**

Comparable — use when the class has one natural/default sorting order that makes sense everywhere it's used (e.g., `Employee` sorted by `id`, `String` sorted alphabetically). You implement it inside the class itself, so every place that sorts this object gets that order by default via `Collections.sort(list)`.

Comparator — use when you need multiple/alternate sort orders, or you can't modify the class (e.g., it's from a third-party library), or the sort logic is situational (sort by salary here, by name there). You write it externally and pass it in: `list.sort(comparator)`.

> **Rule of thumb:** If there's one "obvious" way to order the object → `Comparable`. If sorting varies by use case or you don't own the class → `Comparator`.

### Comparable Example

```java
import java.util.*;

class Employee implements Comparable<Employee> {
    int id;
    String name;
    double salary;

    Employee(int id, String name, double salary) {
        this.id = id; this.name = name; this.salary = salary;
    }

    // Natural ordering -> by id
    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);
    }

    @Override
    public String toString() {
        return name + "(" + id + ", " + salary + ")";
    }
}

public class ComparableDemo {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee(3, "Amit", 55000),
            new Employee(1, "Bhavna", 72000),
            new Employee(2, "Chirag", 61000)
        ));

        Collections.sort(employees); // uses compareTo() -> sorted by id
        System.out.println(employees); // [Bhavna(1,..), Chirag(2,..), Amit(3,..)]
    }
}
```

### Comparator Example (multiple sort orders without touching Employee class)

```java
import java.util.*;

public class ComparatorDemo {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee(3, "Amit", 55000),
            new Employee(1, "Bhavna", 72000),
            new Employee(2, "Chirag", 61000)
        ));

        // Sort by salary using anonymous class
        Comparator<Employee> bySalary = new Comparator<Employee>() {
            @Override
            public int compare(Employee e1, Employee e2) {
                return Double.compare(e1.salary, e2.salary);
            }
        };
        employees.sort(bySalary);
        System.out.println(employees); // sorted by salary asc

        // Sort by name using lambda (Java 8+)
        employees.sort((e1, e2) -> e1.name.compareTo(e2.name));
        System.out.println(employees);

        // Using Comparator.comparing + thenComparing + reversed (most used in real projects)
        employees.sort(
            Comparator.comparing((Employee e) -> e.salary)
                      .reversed()
                      .thenComparing(e -> e.name)
        );
        System.out.println(employees); // highest salary first, name as tiebreaker
    }
}
```

**Interview tip:** `Comparator.comparing()`, `.thenComparing()`, `.reversed()`, and method references (`Comparator.comparing(Employee::getSalary)`) are what you should demo — interviewers love seeing you know the modern fluent API, not just anonymous classes.

---

## 5. LinkedList — Internal Working

`LinkedList<E>` implements `List` **and** `Deque`. Internally it's a **doubly linked list** — each node holds `item`, `next`, `prev` references.

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```

There's no backing array. `first` and `last` node references are maintained.

### Time complexity

| Operation | Complexity | Why |
|---|---|---|
| `addFirst()` / `addLast()` | O(1) | Just relink head/tail pointers |
| `add(index, e)` (middle) | O(n) | Must traverse to reach the index first |
| `get(index)` | O(n) | No random access — must traverse from head or tail (whichever closer) |
| `remove(Node)` once located | O(1) | Just relink — but locating the node itself is O(n) |

**ArrayList vs LinkedList — the classic interview question:**

| | ArrayList | LinkedList |
|---|---|---|
| Backing structure | Dynamic array | Doubly linked list |
| Random access `get(i)` | O(1) | O(n) |
| Insert/delete at ends | O(1) amortized (end) | O(1) |
| Insert/delete in middle | O(n) (shifting) | O(n) (traversal) but no shifting once located |
| Memory overhead | Lower (just array) | Higher (each node stores 2 extra references) |
| Implements Deque? | No | Yes — can be used as Stack/Queue too |

**Rule of thumb to answer confidently:** If you need frequent random access → ArrayList. If you need frequent insertion/deletion at head/tail (queue-like usage) → LinkedList.

### Code Example

```java
import java.util.*;

public class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList<String> ll = new LinkedList<>();
        ll.add("B");
        ll.addFirst("A");   // O(1)
        ll.addLast("C");    // O(1)
        System.out.println(ll); // [A, B, C]

        // Used as Deque (double-ended queue)
        ll.push("Front");   // addFirst
        ll.pop();           // removeFirst - LIFO behaviour, stack-like

        // Used as Queue
        Queue<String> queue = new LinkedList<>();
        queue.offer("task1");
        queue.offer("task2");
        System.out.println(queue.poll()); // task1 - FIFO
    }
}
```

---

## 6. Vector — Internal Working

`Vector<E>` is a **legacy class** (present since JDK 1.0, later retrofitted to implement `List`). Internally, also a **resizable array**, same shape as `ArrayList`.

**Key differences from ArrayList:**

| | ArrayList | Vector |
|---|---|---|
| Thread-safe? | No | **Yes** — every method is `synchronized` |
| Growth factor | 1.5x | **2x (doubles)** — `newCapacity = oldCapacity * 2` |
| Performance | Faster (no sync overhead) | Slower due to per-method locking |
| Legacy? | Modern (Java 1.2+) | Legacy (Java 1.0) |
| Iterator | fail-fast | fail-fast (also has legacy `Enumeration`, which is NOT fail-fast) |

```java
import java.util.*;

public class VectorDemo {
    public static void main(String[] args) {
        Vector<Integer> v = new Vector<>(); // default capacity 10, doubles on growth
        v.add(1);
        v.add(2);

        // Thread safety demonstrated - but coarse-grained (whole method locked)
        synchronized (v) {
            for (int i : v) {
                System.out.println(i);
            }
        }
    }
}
```

**Interview point:** Vector is largely obsolete in modern code. It's asked mainly for conceptual comparison against ArrayList and to test if you know `Collections.synchronizedList(new ArrayList<>())` is the preferred modern alternative when synchronization is genuinely needed, and `CopyOnWriteArrayList` is preferred for read-heavy concurrent scenarios.

---

## 7. Stack — Internal Working

`Stack<E>` **extends Vector** (yes — a design flaw acknowledged by Java itself; `Stack` inherits all of Vector's array-based, synchronized behaviour). It represents **LIFO (Last In First Out)**.

```java
public class Stack<E> extends Vector<E> { ... }
```

Because it extends Vector:
- Every push/pop is synchronized → thread-safe but slower
- It's backed by the same resizable, doubling array

### Key methods

| Method | Behaviour |
|---|---|
| `push(e)` | Adds to top — `addElement()` internally |
| `pop()` | Removes & returns top element; throws `EmptyStackException` if empty |
| `peek()` | Returns top without removing |
| `empty()` | Checks if stack is empty |
| `search(obj)` | Returns 1-based position from top, -1 if not found |

```java
import java.util.*;

public class StackDemo {
    public static void main(String[] args) {
        Stack<Integer> stack = new Stack<>();
        stack.push(10);
        stack.push(20);
        stack.push(30);

        System.out.println(stack.peek());  // 30
        System.out.println(stack.pop());   // 30 removed
        System.out.println(stack);         // [10, 20]
        System.out.println(stack.search(10)); // 2 (1-based from top)
    }
}
```

**Interview point:** Because of the design flaw (extending Vector instead of composing it), **modern code prefers `ArrayDeque` as a stack** instead:

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(10);
stack.push(20);
System.out.println(stack.pop()); // 20 - LIFO, faster, not synchronized, no legacy baggage
```

---

## 8. CopyOnWriteArrayList — Internal Working

`CopyOnWriteArrayList<E>` is a **thread-safe variant of ArrayList** from `java.util.concurrent`, designed for scenarios where **reads vastly outnumber writes** (e.g., listener lists, caches read by many threads, rarely updated config lists).

### How it works internally

```java
private transient volatile Object[] array;
```

- The backing array reference is `volatile` — guarantees visibility of the latest array across threads without synchronization on reads.
- **Every mutative operation** (`add`, `set`, `remove`) does the following:
    1. Acquires a `ReentrantLock` (only one writer at a time)
    2. **Creates a brand-new copy of the entire underlying array** (`Arrays.copyOf`)
    3. Performs the modification on the new array
    4. **Atomically swaps** the `array` reference to point to the new array
    5. Releases the lock

- **Reads (`get`, iteration) require NO locking at all** — they simply read whatever `array` reference is currently visible to them. This is why it's extremely fast for reads.

- **Iterator is a snapshot iterator** — it iterates over the array reference captured **at the time the iterator was created**. It is NOT fail-fast; it will **never throw `ConcurrentModificationException`**, and it will **not reflect additions/removals** made after the iterator was created (weakly consistent).

### Trade-offs

| Pros | Cons |
|---|---|
| Extremely fast, lock-free reads | Every write copies the ENTIRE array → O(n) memory + time per write |
| No `ConcurrentModificationException` ever | Not suitable for write-heavy lists (memory churn, GC pressure) |
| Safe iteration without external locking | Iterator shows a stale snapshot, not live updates |

### Code Example

```java
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.*;

public class COWListDemo {
    public static void main(String[] args) throws InterruptedException {
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        cowList.add("A");
        cowList.add("B");
        cowList.add("C");

        // Demonstrating snapshot iterator behaviour
        Iterator<String> it = cowList.iterator();
        cowList.add("D"); // modifies list AFTER iterator was created

        System.out.println("Iterating snapshot:");
        while (it.hasNext()) {
            System.out.println(it.next()); // prints A, B, C only - NOT D
        }
        System.out.println("Actual list now: " + cowList); // [A, B, C, D]

        // No ConcurrentModificationException even though we modified during "iteration"

        // Real-world use case: multiple threads reading, occasional writer
        CopyOnWriteArrayList<Runnable> listeners = new CopyOnWriteArrayList<>();
        listeners.add(() -> System.out.println("Listener 1 fired"));
        listeners.add(() -> System.out.println("Listener 2 fired"));

        // Many threads can safely iterate & invoke listeners concurrently
        for (Runnable r : listeners) {
            r.run();
        }
    }
}
```

**Interview one-liner comparison:**

| | `Collections.synchronizedList(new ArrayList<>())` | `CopyOnWriteArrayList` |
|---|---|---|
| Locking strategy | Locks on every read AND write | Locks only on write; reads are lock-free |
| Good for | Balanced read/write | Read-heavy, write-rare |
| Iterator | Fail-fast (needs manual `synchronized` block during iteration) | Fail-safe snapshot iterator |
| Write cost | O(1)/O(n) same as ArrayList | O(n) always — full array copy |

---

## Quick Summary Table — List Implementations

| Class | Backing structure | Thread-safe | Growth | Best for |
|---|---|---|---|---|
| `ArrayList` | Dynamic array | No | 1.5x | Fast random access, general purpose |
| `LinkedList` | Doubly linked list | No | N/A (node-based) | Frequent insert/delete at ends, deque/queue use |
| `Vector` | Dynamic array | Yes (synchronized) | 2x | Legacy, rarely used now |
| `Stack` | Dynamic array (extends Vector) | Yes (synchronized) | 2x | Legacy LIFO — prefer `ArrayDeque` instead |
| `CopyOnWriteArrayList` | Dynamic array, copied on write | Yes (lock-free reads) | Full copy per write | Read-heavy concurrent scenarios |

---