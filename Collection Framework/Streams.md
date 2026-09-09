# Java Streams — Complete Interview-Ready Notes (Hands-on Edition)

---

## Table of Contents
1. Stream Fundamentals (mental model)
2. Ways to Create Streams
3. Intermediate Operations
4. Terminal Operations
5. Collectors (deep dive)
6. Parallel Streams
7. Iterator vs Stream vs Spliterator
8. Optional (closely tied to Streams)
9. Primitive Streams (IntStream/LongStream/DoubleStream)
10. Common Interview Questions & Gotchas
11. Practice Checklist

---

## 1. Stream Fundamentals

**What is a Stream?** A sequence of elements supporting functional-style operations. It is **not a data structure** — it doesn't store data, it pipes data from a source through a pipeline of operations.

Key properties (favorite interview question: *"What are the characteristics of a Stream?"*):
- **Not a data structure** — no storage, just a view/pipeline over a source (Collection, array, I/O channel, generator).
- **Functional in nature** — operations produce a result, they don't modify the source (unless you deliberately do so inside a lambda, which is discouraged).
- **Laziness-seeking** — intermediate operations are **lazy**; nothing executes until a terminal operation is invoked.
- **Consumable / one-time use** — once a terminal operation runs, the stream is closed. Reusing it throws `IllegalStateException: stream has already been operated upon or closed`.
- **Can be infinite** — via `Stream.iterate` / `Stream.generate`, combined with `limit()`.
- **Can be parallelized** easily via `parallelStream()` / `.parallel()`.

```java
import java.util.*;
import java.util.stream.*;

public class StreamReuseDemo {
    public static void main(String[] args) {
        Stream<Integer> stream = Stream.of(1, 2, 3);
        stream.forEach(System.out::println); // terminal op #1 - consumes the stream

        try {
            stream.forEach(System.out::println); // BOOM
        } catch (IllegalStateException e) {
            System.out.println("Caught: " + e.getMessage());
        }
    }
}
```

**Pipeline anatomy:**
```
Source  ->  Intermediate Op(s) [lazy, return Stream]  ->  Terminal Op [eager, triggers execution, returns non-Stream]
```

```java
import java.util.*;
import java.util.stream.*;

public class LazinessDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");

        Stream<String> s = names.stream()
            .filter(n -> {
                System.out.println("filtering " + n);
                return n.length() > 3;
            })
            .map(n -> {
                System.out.println("mapping " + n);
                return n.toUpperCase();
            });

        System.out.println("Pipeline built. Nothing printed yet above this line except this sentence executing.");
        System.out.println("Now calling terminal op:");
        List<String> result = s.collect(Collectors.toList());
        System.out.println(result);
        // Notice: filter+map run PER ELEMENT (vertical execution), not filter-all-then-map-all.
    }
}
```
**Key takeaway:** Streams process element-by-element through the *whole* pipeline (vertical), not stage-by-stage across all elements (horizontal). This is why `filter().map().filter()` can short-circuit early with `findFirst()`/`limit()` without processing the entire source.

---

## 2. Ways to Create Streams

### 2.1 From a Collection
```java
import java.util.*;
import java.util.stream.*;

public class CreateFromCollection {
    public static void main(String[] args) {
        List<String> list = List.of("a", "b", "c");
        Stream<String> s1 = list.stream();          // sequential
        Stream<String> s2 = list.parallelStream();   // parallel

        Set<Integer> set = Set.of(1, 2, 3);
        Stream<Integer> s3 = set.stream();

        Map<String, Integer> map = Map.of("x", 1, "y", 2);
        Stream<Map.Entry<String, Integer>> s4 = map.entrySet().stream();
        Stream<String> keys = map.keySet().stream();
        Stream<Integer> values = map.values().stream();

        s1.forEach(System.out::println);
        s4.forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));
    }
}
```

### 2.2 `Stream.of()`
```java
import java.util.stream.*;

public class StreamOfDemo {
    public static void main(String[] args) {
        Stream<Integer> s1 = Stream.of(1, 2, 3, 4);
        s1.forEach(System.out::println);

        Stream<String> single = Stream.of("onlyOne");
        single.forEach(System.out::println);

        // Careful: Stream.of() with a single array argument treats it as varargs -> stream of elements
        Integer[] arr = {10, 20, 30};
        Stream<Integer> s2 = Stream.of(arr);
        System.out.println(s2.count()); // 3
    }
}
```

### 2.3 From Arrays — `Arrays.stream()`
```java
import java.util.Arrays;
import java.util.stream.*;

public class ArraysStreamDemo {
    public static void main(String[] args) {
        int[] intArr = {1, 2, 3, 4, 5};
        IntStream is = Arrays.stream(intArr); // primitive stream, no boxing
        System.out.println(is.sum());

        String[] strArr = {"x", "y", "z"};
        Stream<String> ss = Arrays.stream(strArr);
        ss.forEach(System.out::println);

        // partial range: from index 1 (inclusive) to 4 (exclusive)
        int[] partial = {10, 20, 30, 40, 50};
        Arrays.stream(partial, 1, 4).forEach(System.out::println); // 20 30 40
    }
}
```

### 2.4 `Stream.builder()`
```java
import java.util.stream.*;

public class StreamBuilderDemo {
    public static void main(String[] args) {
        Stream<String> stream = Stream.<String>builder()
            .add("one")
            .add("two")
            .add("three")
            .build();
        stream.forEach(System.out::println);
    }
}
```

### 2.5 `Stream.empty()`
```java
import java.util.stream.*;

public class StreamEmptyDemo {
    public static void main(String[] args) {
        Stream<String> empty = Stream.empty();
        System.out.println(empty.count()); // 0
        // Useful as a safe default return instead of returning null from a method
    }
}
```

### 2.6 Infinite Streams — `Stream.iterate()` and `Stream.generate()`
```java
import java.util.stream.*;

public class InfiniteStreamsDemo {
    public static void main(String[] args) {
        // iterate: seed + UnaryOperator, must limit() or it runs forever
        Stream.iterate(1, n -> n * 2)
            .limit(10)
            .forEach(System.out::println); // 1 2 4 8 16 32 64 128 256 512

        // iterate with predicate (Java 9+) - like a for loop with condition, self-limiting
        Stream.iterate(1, n -> n < 100, n -> n * 2)
            .forEach(System.out::println); // 1 2 4 8 16 32 64

        // generate: Supplier, purely random/computed values, order not guaranteed to be meaningful
        Stream.generate(Math::random)
            .limit(5)
            .forEach(System.out::println);

        // generate a constant stream
        Stream.generate(() -> "X").limit(3).forEach(System.out::println);
    }
}
```

### 2.7 `IntStream.range()` / `IntStream.rangeClosed()`
```java
import java.util.stream.*;

public class RangeDemo {
    public static void main(String[] args) {
        IntStream.range(1, 5).forEach(System.out::println);        // 1 2 3 4  (end exclusive)
        IntStream.rangeClosed(1, 5).forEach(System.out::println);  // 1 2 3 4 5 (end inclusive)

        // common interview trick: sum of first 100 natural numbers
        int sum = IntStream.rangeClosed(1, 100).sum();
        System.out.println(sum); // 5050
    }
}
```

### 2.8 From a String — chars / lines
```java
import java.util.stream.*;

public class StringStreamDemo {
    public static void main(String[] args) {
        "hello".chars() // IntStream of char codes
            .mapToObj(c -> String.valueOf((char) c))
            .forEach(System.out::println);

        String multiline = "line1\nline2\nline3";
        multiline.lines().forEach(System.out::println); // Java 11+
    }
}
```

### 2.9 From a File — `Files.lines()`
```java
import java.nio.file.*;
import java.io.IOException;
import java.util.stream.*;

public class FileLinesDemo {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("data.txt"); // create this file to test
        try (Stream<String> lines = Files.lines(path)) {
            long count = lines.filter(l -> !l.isBlank()).count();
            System.out.println("Non-blank lines: " + count);
        }
        // Files.lines() must be closed (try-with-resources) - it holds a file handle open
    }
}
```

### 2.10 `Collections.nCopies` + `stream()`, and `Random.ints()`
```java
import java.util.*;
import java.util.stream.*;

public class MiscCreationDemo {
    public static void main(String[] args) {
        Collections.nCopies(3, "repeat").stream().forEach(System.out::println);

        new Random(42).ints(5, 0, 100).forEach(System.out::println); // 5 random ints in [0,100)

        // StreamSupport from an Iterable/Spliterator (rare, but interview-worthy)
        Iterable<String> iterable = List.of("p", "q", "r");
        Stream<String> s = StreamSupport.stream(iterable.spliterator(), false);
        s.forEach(System.out::println);
    }
}
```

**Practice task:** Write a program that creates a stream of the first 20 Fibonacci numbers using `Stream.iterate()` with an `int[]` or a record pair as state.

```java
// Solution sketch
import java.util.stream.*;

public class FibonacciDemo {
    public static void main(String[] args) {
        Stream.iterate(new int[]{0, 1}, f -> new int[]{f[1], f[0] + f[1]})
            .limit(20)
            .map(f -> f[0])
            .forEach(System.out::println);
    }
}
```

---

## 3. Intermediate Operations
*(Lazy — return a `Stream`, can be chained, don't execute until a terminal op is called.)*

### 3.1 `filter(Predicate)`
```java
import java.util.*;
import java.util.stream.*;

public class FilterDemo {
    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
            new Employee("Alice", "Eng", 95000),
            new Employee("Bob", "Sales", 60000),
            new Employee("Charlie", "Eng", 110000),
            new Employee("Dave", "HR", 55000)
        );

        emps.stream()
            .filter(e -> e.dept().equals("Eng"))
            .filter(e -> e.salary() > 100000) // multiple filters chain like AND
            .forEach(System.out::println);
    }
}
```

### 3.2 `map(Function)` — one-to-one transformation
```java
import java.util.*;
import java.util.stream.*;

public class MapDemo {
    public static void main(String[] args) {
        List<String> names = List.of("alice", "bob", "charlie");

        List<Integer> lengths = names.stream()
            .map(String::length)
            .collect(Collectors.toList());
        System.out.println(lengths); // [5, 3, 7]

        List<String> upper = names.stream()
            .map(String::toUpperCase)
            .toList(); // Java 16+ shortcut for collect(toList()), returns immutable list
        System.out.println(upper);
    }
}
```

### 3.3 `flatMap(Function)` — one-to-many, flattening nested structures
```java
import java.util.*;
import java.util.stream.*;

public class FlatMapDemo {
    public static void main(String[] args) {
        List<List<Integer>> nested = List.of(
            List.of(1, 2, 3),
            List.of(4, 5),
            List.of(6, 7, 8, 9)
        );

        List<Integer> flat = nested.stream()
            .flatMap(List::stream)
            .collect(Collectors.toList());
        System.out.println(flat); // [1,2,3,4,5,6,7,8,9]

        // classic interview question: flatten a List<String> of sentences into a List<String> of unique words
        List<String> sentences = List.of("hello world", "world of java", "java streams rock");
        List<String> uniqueWords = sentences.stream()
            .flatMap(s -> Arrays.stream(s.split(" ")))
            .distinct()
            .sorted()
            .toList();
        System.out.println(uniqueWords);

        // flatMap vs map: map would give Stream<Stream<Integer>> or Stream<String[]> - not flattened
        List<List<Integer>> stillNested = nested.stream()
            .map(List::stream) // WRONG if you wanted flattening - gives Stream<Stream<Integer>>
            .map(Stream::toList)
            .toList();
        System.out.println(stillNested); // [[1,2,3],[4,5],[6,7,8,9]]
    }
}
```

### 3.4 `sorted()` and `sorted(Comparator)`
```java
import java.util.*;
import java.util.stream.*;
import java.util.Comparator;

public class SortedDemo {
    record Person(String name, int age) {}

    public static void main(String[] args) {
        List<Integer> nums = List.of(5, 3, 8, 1, 9);
        nums.stream().sorted().forEach(System.out::println); // natural order

        nums.stream().sorted(Comparator.reverseOrder()).forEach(System.out::println);

        List<Person> people = List.of(
            new Person("Bob", 30),
            new Person("Alice", 25),
            new Person("Charlie", 25)
        );

        // multi-level sort: by age asc, then name asc
        people.stream()
            .sorted(Comparator.comparingInt(Person::age).thenComparing(Person::name))
            .forEach(System.out::println);

        // descending
        people.stream()
            .sorted(Comparator.comparing(Person::name).reversed())
            .forEach(System.out::println);

        // NOTE: sorted() on unsorted natural-order-incompatible types without Comparator throws ClassCastException
    }
}
```

### 3.5 `distinct()` — uses `equals()`/`hashCode()`
```java
import java.util.*;
import java.util.stream.*;

public class DistinctDemo {
    record Point(int x, int y) {} // record auto-generates equals/hashCode

    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 2, 3, 3, 3, 4);
        nums.stream().distinct().forEach(System.out::println); // 1 2 3 4

        List<Point> points = List.of(new Point(1,1), new Point(1,1), new Point(2,2));
        points.stream().distinct().forEach(System.out::println); // relies on record's equals/hashCode -> 2 elements

        // Gotcha: for a plain class without equals/hashCode override, distinct() uses reference equality
    }
}
```

### 3.6 `limit(n)` and `skip(n)` — pagination pattern
```java
import java.util.stream.*;

public class LimitSkipDemo {
    public static void main(String[] args) {
        IntStream.rangeClosed(1, 20)
            .skip(5)   // skip first 5
            .limit(5)  // take next 5
            .forEach(System.out::println); // 6 7 8 9 10

        // classic pagination helper
        int pageNumber = 3, pageSize = 4; // 1-indexed page
        IntStream.rangeClosed(1, 20)
            .skip((long) (pageNumber - 1) * pageSize)
            .limit(pageSize)
            .forEach(System.out::println); // 9 10 11 12
    }
}
```

### 3.7 `peek()` — side-effect / debugging (NOT for mutating business logic)
```java
import java.util.*;
import java.util.stream.*;

public class PeekDemo {
    public static void main(String[] args) {
        List<String> result = Stream.of("a", "bb", "ccc")
            .peek(s -> System.out.println("original: " + s))
            .map(String::toUpperCase)
            .peek(s -> System.out.println("mapped: " + s))
            .collect(Collectors.toList());

        System.out.println(result);
        // WARNING: peek() may be skipped entirely if the JIT/stream determines the result
        // isn't needed for elements beyond what's consumed (e.g., with findFirst()). Never
        // rely on peek() for logic with side effects that affect correctness.
    }
}
```

### 3.8 `mapToInt` / `mapToLong` / `mapToDouble` / `boxed()`
```java
import java.util.*;
import java.util.stream.*;

public class MapToPrimitiveDemo {
    public static void main(String[] args) {
        List<String> words = List.of("apple", "banana", "kiwi");

        int totalChars = words.stream()
            .mapToInt(String::length) // Stream<String> -> IntStream (avoids boxing)
            .sum();
        System.out.println(totalChars);

        // boxed(): IntStream -> Stream<Integer>, needed when you want to use Collectors etc.
        List<Integer> lengths = words.stream()
            .mapToInt(String::length)
            .boxed()
            .collect(Collectors.toList());
        System.out.println(lengths);
    }
}
```

### 3.9 `takeWhile()` / `dropWhile()` (Java 9+)
```java
import java.util.*;
import java.util.stream.*;

public class TakeDropWhileDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 3, 4, 5, 1, 2);

        // takeWhile: takes elements until predicate first fails, THEN STOPS (even if later elements match)
        nums.stream().takeWhile(n -> n < 4).forEach(System.out::println); // 1 2 3

        // dropWhile: drops elements until predicate first fails, then takes ALL remaining
        nums.stream().dropWhile(n -> n < 4).forEach(System.out::println); // 4 5 1 2

        // Contrast with filter (checks EVERY element, doesn't stop early):
        nums.stream().filter(n -> n < 4).forEach(System.out::println); // 1 2 3 1 2
    }
}
```

**Practice tasks for Section 3:**
1. Given `List<String> words`, return a list of unique word lengths, sorted descending.
2. Given `List<List<String>> teams`, flatten to a single sorted list of all unique player names.
3. Given an `IntStream` of 1–50, skip multiples of 3, then take the first 10 remaining.

---

## 4. Terminal Operations
*(Eager — trigger pipeline execution, return a non-Stream result, close the stream.)*

### 4.1 `forEach()` / `forEachOrdered()`
```java
import java.util.*;
import java.util.stream.*;

public class ForEachDemo {
    public static void main(String[] args) {
        List.of(1, 2, 3).stream().forEach(System.out::println);

        // In PARALLEL streams, forEach does NOT guarantee order!
        List.of(1, 2, 3, 4, 5).parallelStream().forEach(System.out::println); // order may vary each run

        // forEachOrdered forces encounter order even in parallel (at a performance cost)
        List.of(1, 2, 3, 4, 5).parallelStream().forEachOrdered(System.out::println); // always 1 2 3 4 5
    }
}
```

### 4.2 `collect()` — covered deeply in Section 5

### 4.3 `reduce()` — folding a stream into a single value
```java
import java.util.*;
import java.util.stream.*;

public class ReduceDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 3, 4, 5);

        // 1-arg: returns Optional<T> (empty stream -> empty Optional)
        Optional<Integer> sum1 = nums.stream().reduce((a, b) -> a + b);
        System.out.println(sum1.get()); // 15

        // 2-arg: identity + accumulator, returns T directly (identity is the default for empty stream)
        int sum2 = nums.stream().reduce(0, (a, b) -> a + b);
        System.out.println(sum2); // 15

        int product = nums.stream().reduce(1, (a, b) -> a * b);
        System.out.println(product); // 120

        // 3-arg: identity + accumulator + combiner (needed for PARALLEL streams with type change)
        int totalLength = Stream.of("a", "bb", "ccc")
            .reduce(0,
                (partialResult, str) -> partialResult + str.length(), // accumulator: (int, String) -> int
                (r1, r2) -> r1 + r2); // combiner: merges partial results from different threads
        System.out.println(totalLength); // 6

        // max/min via reduce (though Comparator-based max()/min() terminal ops are cleaner - see 4.4)
        Optional<Integer> max = nums.stream().reduce(Integer::max);
        System.out.println(max.get()); // 5
    }
}
```

**Interview gotcha:** Why does `reduce` need a *combiner* for parallel streams?
Because in parallel execution, the source is split into chunks, each chunk is reduced independently (using identity + accumulator), and then the combiner merges those partial results together. In sequential streams, the combiner is never actually invoked.

### 4.4 `count()`, `min()`, `max()`
```java
import java.util.*;
import java.util.stream.*;
import java.util.Comparator;

public class CountMinMaxDemo {
    record Product(String name, double price) {}

    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop", 80000),
            new Product("Mouse", 500),
            new Product("Keyboard", 1500)
        );

        long count = products.stream().filter(p -> p.price() > 1000).count();
        System.out.println(count); // 2

        Optional<Product> cheapest = products.stream().min(Comparator.comparingDouble(Product::price));
        Optional<Product> costliest = products.stream().max(Comparator.comparingDouble(Product::price));
        System.out.println(cheapest.get());
        System.out.println(costliest.get());
    }
}
```

### 4.5 `anyMatch()`, `allMatch()`, `noneMatch()` — short-circuiting
```java
import java.util.*;
import java.util.stream.*;

public class MatchDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(2, 4, 6, 8, 10);

        System.out.println(nums.stream().allMatch(n -> n % 2 == 0));  // true
        System.out.println(nums.stream().anyMatch(n -> n > 8));       // true
        System.out.println(nums.stream().noneMatch(n -> n < 0));      // true

        // Gotcha: allMatch() on an EMPTY stream returns true (vacuous truth)
        System.out.println(Stream.<Integer>empty().allMatch(n -> n > 100)); // true
        // noneMatch() on an empty stream also returns true
        System.out.println(Stream.<Integer>empty().noneMatch(n -> n > 100)); // true
        // anyMatch() on an empty stream returns false
        System.out.println(Stream.<Integer>empty().anyMatch(n -> n > 100)); // false

        // These SHORT-CIRCUIT: stops as soon as the answer is determined
        List.of(1, 2, 3, 4, 5).stream()
            .peek(n -> System.out.println("checking " + n))
            .anyMatch(n -> n == 3); // stops after checking 3, doesn't check 4,5
    }
}
```

### 4.6 `findFirst()` vs `findAny()`
```java
import java.util.*;
import java.util.stream.*;

public class FindDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 3, 4, 5);

        Optional<Integer> first = nums.stream().filter(n -> n > 2).findFirst();
        System.out.println(first.get()); // 3 - deterministic, respects encounter order

        // findAny(): in a SEQUENTIAL stream behaves like findFirst; in a PARALLEL stream
        // it returns as soon as ANY thread finds a match - faster but non-deterministic order
        Optional<Integer> any = nums.parallelStream().filter(n -> n > 2).findAny();
        System.out.println(any.get()); // could be 3, 4, or 5
    }
}
```

### 4.7 `toArray()`
```java
import java.util.*;
import java.util.stream.*;

public class ToArrayDemo {
    public static void main(String[] args) {
        Object[] arr1 = Stream.of("a", "b", "c").toArray();

        String[] arr2 = Stream.of("a", "b", "c").toArray(String[]::new); // typed array via generator
        System.out.println(Arrays.toString(arr2));

        int[] intArr = IntStream.of(1, 2, 3).toArray(); // primitive stream toArray is simpler
        System.out.println(Arrays.toString(intArr));
    }
}
```

### 4.8 `Stream.close()` and `onClose()` — rarely asked, but good to know
```java
import java.util.stream.*;

public class StreamCloseDemo {
    public static void main(String[] args) {
        Stream<String> s = Stream.of("a", "b")
            .onClose(() -> System.out.println("Stream closed!"));
        s.forEach(System.out::println);
        s.close(); // manually trigger onClose (usually done via try-with-resources for I/O streams)
    }
}
```

**Practice tasks for Section 4:**
1. Given `List<Order>` with an `amount` field, compute total revenue using `reduce` AND using `mapToDouble().sum()`. Compare readability.
2. Given `List<String>` of emails, check if all contain `@` using `allMatch`.
3. Find the first employee older than 40 using `findFirst`; explain why `findAny` would be a bad choice here even though it's "faster."

---

## 5. Collectors — Deep Dive

`Collectors` is a utility class of `Collector` factory methods used with `.collect()`. This is the single most tested area in Streams interviews.

### 5.1 `toList()`, `toUnmodifiableList()`, `toSet()`, `toCollection()`
```java
import java.util.*;
import java.util.stream.*;

public class BasicCollectorsDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6);

        List<Integer> list = nums.stream().collect(Collectors.toList()); // mutable, no guarantee on impl
        List<Integer> immutableList = nums.stream().collect(Collectors.toUnmodifiableList());
        List<Integer> viaShortcut = nums.stream().toList(); // Java 16+, ALWAYS immutable

        Set<Integer> set = nums.stream().collect(Collectors.toSet()); // duplicates removed, unordered

        // toCollection: control the exact target collection type
        TreeSet<Integer> treeSet = nums.stream().collect(Collectors.toCollection(TreeSet::new));
        System.out.println(treeSet); // sorted, deduped: [1, 2, 3, 4, 5, 6, 9]

        LinkedList<Integer> linkedList = nums.stream().collect(Collectors.toCollection(LinkedList::new));

        try {
            immutableList.add(99); // throws UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("Can't modify immutable list");
        }
    }
}
```

### 5.2 `joining()` — for `Stream<String>` only
```java
import java.util.*;
import java.util.stream.*;

public class JoiningDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie");

        String plain = names.stream().collect(Collectors.joining());
        System.out.println(plain); // AliceBobCharlie

        String csv = names.stream().collect(Collectors.joining(", "));
        System.out.println(csv); // Alice, Bob, Charlie

        String withBrackets = names.stream().collect(Collectors.joining(", ", "[", "]"));
        System.out.println(withBrackets); // [Alice, Bob, Charlie]
    }
}
```

### 5.3 `toMap()` — with duplicate-key merge function
```java
import java.util.*;
import java.util.stream.*;

public class ToMapDemo {
    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
            new Employee("Alice", "Eng", 95000),
            new Employee("Bob", "Sales", 60000),
            new Employee("Charlie", "Eng", 110000)
        );

        // basic toMap: key mapper, value mapper
        Map<String, Double> nameToSalary = emps.stream()
            .collect(Collectors.toMap(Employee::name, Employee::salary));
        System.out.println(nameToSalary);

        // DUPLICATE KEY problem: two employees with same dept -> IllegalStateException without merge function
        try {
            Map<String, Double> deptToSalary = emps.stream()
                .collect(Collectors.toMap(Employee::dept, Employee::salary));
        } catch (IllegalStateException e) {
            System.out.println("Duplicate key error: " + e.getMessage());
        }

        // FIX: provide a merge function (BinaryOperator<V>) - here we keep the max salary per dept
        Map<String, Double> deptToMaxSalary = emps.stream()
            .collect(Collectors.toMap(Employee::dept, Employee::salary, Double::max));
        System.out.println(deptToMaxSalary); // {Eng=110000.0, Sales=60000.0}

        // 4-arg version: also control the Map implementation (e.g., TreeMap for sorted keys)
        Map<String, Double> sortedDeptMap = emps.stream()
            .collect(Collectors.toMap(Employee::dept, Employee::salary, Double::max, TreeMap::new));
        System.out.println(sortedDeptMap);
    }
}
```

### 5.4 `groupingBy()` — the most important one for interviews
```java
import java.util.*;
import java.util.stream.*;

public class GroupingByDemo {
    record Employee(String name, String dept, double salary, int age) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
            new Employee("Alice", "Eng", 95000, 28),
            new Employee("Bob", "Sales", 60000, 35),
            new Employee("Charlie", "Eng", 110000, 41),
            new Employee("Dave", "HR", 55000, 29),
            new Employee("Eve", "Sales", 62000, 26),
            new Employee("Frank", "Eng", 87000, 33)
        );

        // 1-arg: group into Map<K, List<T>>
        Map<String, List<Employee>> byDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept));
        byDept.forEach((dept, list) -> System.out.println(dept + " -> " + list.size()));

        // 2-arg: groupingBy + downstream collector
        Map<String, Long> countByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
        System.out.println(countByDept); // {Eng=3, HR=1, Sales=2}

        Map<String, Double> avgSalaryByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));
        System.out.println(avgSalaryByDept);

        Map<String, Double> totalSalaryByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.summingDouble(Employee::salary)));
        System.out.println(totalSalaryByDept);

        Map<String, List<String>> namesByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.mapping(Employee::name, Collectors.toList())));
        System.out.println(namesByDept);

        // 3-arg: groupingBy + Map factory + downstream (control map type, e.g. TreeMap for sorted keys)
        Map<String, Long> sortedCountByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.counting()));
        System.out.println(sortedCountByDept); // keys sorted alphabetically

        // NESTED groupingBy - group by dept, then by age bracket
        Map<String, Map<String, List<Employee>>> byDeptThenAgeBracket = emps.stream()
            .collect(Collectors.groupingBy(
                Employee::dept,
                Collectors.groupingBy(e -> e.age() < 30 ? "Young" : "Senior")
            ));
        byDeptThenAgeBracket.forEach((dept, ageMap) -> {
            System.out.println(dept + ":");
            ageMap.forEach((bracket, list) -> System.out.println("  " + bracket + " -> " +
                list.stream().map(Employee::name).toList()));
        });

        // groupingBy + maxBy: find highest-paid employee per dept
        Map<String, Optional<Employee>> highestPaidPerDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept,
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));
        highestPaidPerDept.forEach((dept, empOpt) -> System.out.println(dept + " -> " + empOpt.get().name()));

        // groupingBy + summarizingDouble: get min/max/avg/sum/count in ONE pass
        Map<String, DoubleSummaryStatistics> statsByDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.summarizingDouble(Employee::salary)));
        statsByDept.forEach((dept, stats) ->
            System.out.printf("%s: min=%.0f max=%.0f avg=%.0f count=%d%n",
                dept, stats.getMin(), stats.getMax(), stats.getAverage(), stats.getCount()));
    }
}
```

### 5.5 `partitioningBy()` — special case of grouping, always produces exactly 2 buckets (true/false)
```java
import java.util.*;
import java.util.stream.*;

public class PartitioningByDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        Map<Boolean, List<Integer>> evenOdd = nums.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0));
        System.out.println(evenOdd.get(true));  // evens
        System.out.println(evenOdd.get(false)); // odds

        // partitioningBy + downstream collector
        Map<Boolean, Long> evenOddCounts = nums.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0, Collectors.counting()));
        System.out.println(evenOddCounts); // {false=5, true=5}

        // Difference from groupingBy(booleanFn): partitioningBy ALWAYS has both true and false keys
        // present in the resulting map, even if one bucket is empty. groupingBy would omit missing keys.
        Map<Boolean, List<Integer>> allNegative = List.of(-1, -2, -3).stream()
            .collect(Collectors.partitioningBy(n -> n > 0));
        System.out.println(allNegative); // {false=[-1, -2, -3], true=[]}  <-- true key still present!
    }
}
```

### 5.6 `counting()`, `summingInt/Long/Double()`, `averagingInt/Long/Double()`, `summarizing*()`, `minBy`/`maxBy`
```java
import java.util.*;
import java.util.stream.*;

public class NumericCollectorsDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(4, 8, 15, 16, 23, 42);

        long count = nums.stream().collect(Collectors.counting());
        int sum = nums.stream().collect(Collectors.summingInt(Integer::intValue));
        double avg = nums.stream().collect(Collectors.averagingInt(Integer::intValue));
        Optional<Integer> min = nums.stream().collect(Collectors.minBy(Comparator.naturalOrder()));
        Optional<Integer> max = nums.stream().collect(Collectors.maxBy(Comparator.naturalOrder()));

        IntSummaryStatistics stats = nums.stream().collect(Collectors.summarizingInt(Integer::intValue));
        System.out.println(stats); // gives count, sum, min, avg, max all at once
        System.out.println(stats.getMax());
        System.out.println(stats.getAverage());
    }
}
```

### 5.7 `Collectors.mapping()` and `Collectors.filtering()` — downstream adapters
```java
import java.util.*;
import java.util.stream.*;

public class MappingFilteringDemo {
    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
            new Employee("Alice", "Eng", 95000),
            new Employee("Bob", "Sales", 60000),
            new Employee("Charlie", "Eng", 110000),
            new Employee("Dave", "Eng", 45000)
        );

        // mapping(): transform elements WITHIN a downstream collector
        Map<String, List<String>> deptToNames = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.mapping(Employee::name, Collectors.toList())));
        System.out.println(deptToNames);

        // filtering(): filter WITHIN a group (unlike stream().filter() BEFORE groupingBy,
        // this keeps the group key present even if the filtered list becomes empty)
        Map<String, List<Employee>> highEarnersPerDept = emps.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.filtering(e -> e.salary() > 100000, Collectors.toList())));
        System.out.println(highEarnersPerDept);
        // {Eng=[Charlie...], Sales=[]}  <- Sales key still present with empty list

        // Contrast: filtering BEFORE groupingBy loses the "Sales" key entirely if no Sales employee qualifies
        Map<String, List<Employee>> filteredBeforeGrouping = emps.stream()
            .filter(e -> e.salary() > 100000)
            .collect(Collectors.groupingBy(Employee::dept));
        System.out.println(filteredBeforeGrouping); // {Eng=[Charlie...]}  <- no Sales key at all
    }
}
```

### 5.8 `Collectors.reducing()`
```java
import java.util.*;
import java.util.stream.*;

public class ReducingCollectorDemo {
    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
            new Employee("Alice", "Eng", 95000),
            new Employee("Bob", "Sales", 60000),
            new Employee("Charlie", "Eng", 110000)
        );

        // 1-arg reducing: BinaryOperator, returns Optional<T>
        Optional<Employee> highestPaid = emps.stream()
            .collect(Collectors.reducing((e1, e2) -> e1.salary() > e2.salary() ? e1 : e2));
        System.out.println(highestPaid.get().name());

        // 2-arg reducing: identity + BinaryOperator, returns T directly
        double totalSalary = emps.stream()
            .map(Employee::salary)
            .collect(Collectors.reducing(0.0, Double::sum));
        System.out.println(totalSalary);

        // 3-arg reducing: mapper + identity + BinaryOperator (map-then-reduce in one collector)
        double totalSalary2 = emps.stream()
            .collect(Collectors.reducing(0.0, Employee::salary, Double::sum));
        System.out.println(totalSalary2);
    }
}
```

### 5.9 `Collectors.teeing()` (Java 12+) — run TWO collectors and merge results
```java
import java.util.*;
import java.util.stream.*;

public class TeeingDemo {
    public static void main(String[] args) {
        List<Integer> nums = List.of(4, 8, 15, 16, 23, 42);

        // compute average WITHOUT summarizingInt, using teeing to combine sum + count
        double average = nums.stream()
            .collect(Collectors.teeing(
                Collectors.summingInt(Integer::intValue),
                Collectors.counting(),
                (sum, count) -> (double) sum / count
            ));
        System.out.println(average);

        // real-world example: get min AND max in one pass, packaged into a record
        record MinMax(int min, int max) {}
        MinMax result = nums.stream()
            .collect(Collectors.teeing(
                Collectors.minBy(Comparator.naturalOrder()),
                Collectors.maxBy(Comparator.naturalOrder()),
                (min, max) -> new MinMax(min.get(), max.get())
            ));
        System.out.println(result);
    }
}
```

### 5.10 Custom Collector via `Collector.of()`
```java
import java.util.*;
import java.util.stream.*;

public class CustomCollectorDemo {
    public static void main(String[] args) {
        List<String> words = List.of("apple", "banana", "cherry", "date");

        // Custom collector that builds a comma-separated string using StringBuilder (like joining, but manual)
        Collector<String, StringBuilder, String> customJoiner = Collector.of(
            StringBuilder::new,                              // supplier: create the mutable container
            (sb, s) -> { if (sb.length() > 0) sb.append(", "); sb.append(s); }, // accumulator
            (sb1, sb2) -> { if (sb1.length() > 0 && sb2.length() > 0) sb1.append(", "); return sb1.append(sb2); }, // combiner (parallel merge)
            StringBuilder::toString                          // finisher: container -> final result
        );

        String result = words.stream().collect(customJoiner);
        System.out.println(result); // apple, banana, cherry, date
    }
}
```

**Practice tasks for Section 5:**
1. Group a `List<Order>` by `customerId`, downstream to total order amount (Map<String, Double>).
2. Partition a `List<Student>` into pass/fail by marks >= 40, and within each partition sort by name.
3. Build `Map<String, Long>` of word frequency counts from a paragraph.
4. Use `teeing()` to compute both the longest and shortest string in a `List<String>` in a single pass.

---

## 6. Parallel Streams

### 6.1 Creating & when to use
```java
import java.util.*;
import java.util.stream.*;

public class ParallelBasicsDemo {
    public static void main(String[] args) {
        List<Integer> nums = IntStream.rangeClosed(1, 1_000_000).boxed().toList();

        // Option 1: parallelStream() directly from Collection
        long count1 = nums.parallelStream().filter(n -> n % 2 == 0).count();

        // Option 2: convert an existing sequential stream
        long count2 = nums.stream().parallel().filter(n -> n % 2 == 0).count();

        System.out.println(count1 + " " + count2);

        // Check if a stream is parallel
        System.out.println(nums.stream().isParallel());          // false
        System.out.println(nums.stream().parallel().isParallel()); // true

        // sequential() converts back
        System.out.println(nums.parallelStream().sequential().isParallel()); // false
    }
}
```

**How parallel streams work internally:**
- Uses the **common `ForkJoinPool`** (`ForkJoinPool.commonPool()`), shared JVM-wide, sized to `Runtime.getRuntime().availableProcessors() - 1` by default.
- Uses **fork/join (divide and conquer)**: source is recursively split via `Spliterator`, each chunk processed on a pool thread, results combined.
- **NOT always faster.** Overhead of splitting + thread coordination + result merging can outweigh benefits for small datasets or cheap operations.

### 6.2 When parallel streams help vs hurt
```java
import java.util.*;
import java.util.stream.*;

public class ParallelPerfDemo {
    public static void main(String[] args) {
        int n = 50_000_000;

        long start = System.nanoTime();
        long seqSum = LongStream.rangeClosed(1, n).sum();
        System.out.println("Sequential: " + (System.nanoTime() - start) / 1_000_000 + "ms");

        start = System.nanoTime();
        long parSum = LongStream.rangeClosed(1, n).parallel().sum();
        System.out.println("Parallel: " + (System.nanoTime() - start) / 1_000_000 + "ms");

        System.out.println(seqSum == parSum);
    }
}
```

**Rules of thumb (frequently asked in interviews):**
| Use Parallel When | Avoid Parallel When |
|---|---|
| Large dataset (typically 10,000+ elements) | Small datasets |
| CPU-bound, computationally expensive per-element work | I/O-bound work (parallel streams don't help — threads block waiting on I/O and starve the shared pool) |
| Source splits efficiently: `ArrayList`, arrays, `IntStream.range` | Source splits poorly: `LinkedList`, `Stream.iterate()` (sequential dependency), I/O-based streams |
| Operations are stateless & non-interfering | Operations have shared mutable state, ordering dependency (unless using `forEachOrdered`) |
| You've actually benchmarked and confirmed a gain | "It sounds faster" — always measure |

### 6.3 Common pitfalls: shared mutable state
```java
import java.util.*;
import java.util.concurrent.atomic.*;
import java.util.stream.*;

public class ParallelPitfallDemo {
    public static void main(String[] args) {
        List<Integer> nums = IntStream.rangeClosed(1, 10000).boxed().toList();

        // WRONG: using a plain ArrayList / non-thread-safe mutable state in parallel forEach -> race condition
        List<Integer> unsafeList = new ArrayList<>();
        nums.parallelStream().forEach(unsafeList::add); // DATA RACE - may throw or lose elements
        System.out.println("Unsafe list size (buggy, unreliable): " + unsafeList.size());

        // WRONG: mutable counter via plain int/Integer[] - also a race condition
        int[] unsafeCounter = {0};
        nums.parallelStream().forEach(n -> unsafeCounter[0]++); // lost updates
        System.out.println("Unsafe counter (buggy): " + unsafeCounter[0] + " (expected 10000)");

        // RIGHT: use collect() which handles thread-safe accumulation via combiners
        List<Integer> safeList = nums.parallelStream().collect(Collectors.toList());
        System.out.println("Safe list size: " + safeList.size());

        // RIGHT: use AtomicInteger/AtomicLong for simple counters, or better, use count()/reduce()
        AtomicInteger safeCounter = new AtomicInteger(0);
        nums.parallelStream().forEach(n -> safeCounter.incrementAndGet());
        System.out.println("Safe counter (AtomicInteger): " + safeCounter.get());

        // BEST: avoid manual state entirely, use built-in terminal ops
        long correctCount = nums.parallelStream().count();
        System.out.println("Best (count()): " + correctCount);
    }
}
```

### 6.4 `reduce()` in parallel — why identity/combiner matter more here
```java
import java.util.stream.*;

public class ParallelReduceDemo {
    public static void main(String[] args) {
        // In parallel, the accumulator runs on chunks independently, combiner merges chunk results.
        // Identity MUST be a true identity value for the operation (e.g., 0 for sum, 1 for product,
        // "" for concatenation) or results will be WRONG in parallel mode.
        int sum = IntStream.rangeClosed(1, 1000).parallel()
            .reduce(0, Integer::sum);
        System.out.println(sum); // 500500 - correct because 0 is a true identity for +

        // order-sensitive operations (like subtraction) give WRONG/non-deterministic results in parallel
        int wrongDiff = IntStream.rangeClosed(1, 5).parallel()
            .reduce(0, (a, b) -> a - b); // NOT associative! Don't do this in parallel.
        System.out.println("Unreliable result: " + wrongDiff);
    }
}
```

### 6.5 `ordered vs unordered`
```java
import java.util.stream.*;

public class UnorderedDemo {
    public static void main(String[] args) {
        // unordered() hints the pipeline that encounter order doesn't matter,
        // which can let parallel operations like limit()/distinct() skip expensive order-preserving work
        long count = IntStream.rangeClosed(1, 1000)
            .parallel()
            .unordered()
            .distinct()
            .count();
        System.out.println(count);
    }
}
```

**Practice tasks for Section 6:**
1. Benchmark sequential vs parallel `stream().map()` on a CPU-heavy operation (e.g., checking primality) for 1M numbers.
2. Fix a buggy parallel aggregation that uses a shared `HashMap` — rewrite using `Collectors.groupingBy` (which is parallel-safe).
3. Explain (out loud, as if in an interview) why `Stream.iterate()` parallelizes poorly.

---

## 7. Iterator vs Stream vs Spliterator

### 7.1 Classic `Iterator` — imperative, external iteration
```java
import java.util.*;

public class IteratorDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));

        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            String val = it.next();
            System.out.println(val);
            if (val.equals("b")) {
                it.remove(); // SAFE removal during iteration (unlike a for-each loop, which throws ConcurrentModificationException)
            }
        }
        System.out.println(list); // [a, c, d]

        // ListIterator: bidirectional, allows set() too
        ListIterator<String> lit = list.listIterator();
        while (lit.hasNext()) {
            String val = lit.next();
            lit.set(val.toUpperCase()); // in-place replace
        }
        System.out.println(list); // [A, C, D]

        // Backward iteration
        while (lit.hasPrevious()) {
            System.out.println(lit.previous());
        }
    }
}
```

### 7.2 `ConcurrentModificationException` demo (why Iterator.remove() exists)
```java
import java.util.*;

public class CMEDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));

        try {
            for (Integer n : list) {
                if (n == 3) {
                    list.remove(n); // structural modification during for-each -> CME on next iteration
                }
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("Caught CME: modifying a collection while iterating with for-each/Iterator (without Iterator.remove) is unsafe");
        }

        // Safe alternatives:
        list.removeIf(n -> n == 3); // Collection.removeIf - safe, and Stream-flavored
        System.out.println(list);
    }
}
```

### 7.3 Custom `Iterable`/`Iterator` implementation (sometimes asked to check OOP + Streams understanding)
```java
import java.util.*;

public class RangeIterable implements Iterable<Integer> {
    private final int start, end;
    public RangeIterable(int start, int end) { this.start = start; this.end = end; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int current = start;
            @Override public boolean hasNext() { return current < end; }
            @Override public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return current++;
            }
        };
    }

    public static void main(String[] args) {
        for (int i : new RangeIterable(1, 6)) {
            System.out.println(i);
        }
        // Because we implemented Iterable, we can ALSO get a Stream from it via StreamSupport:
        Iterable<Integer> range = new RangeIterable(1, 6);
        StreamSupport.stream(range.spliterator(), false)
            .map(i -> i * i)
            .forEach(System.out::println);
    }
}
```
*(needs `import java.util.stream.StreamSupport;`)*

### 7.4 Iterator vs Stream — interview comparison table
| Aspect | Iterator | Stream |
|---|---|---|
| Style | External / imperative (`hasNext()`, `next()`) | Internal / declarative (pass a function, library controls iteration) |
| Reusability | Single traversal | Single traversal (both consumed once) |
| Mutation | Can remove elements via `Iterator.remove()` | Cannot mutate source; produces new results |
| Parallelism | Manual, error-prone | Built-in via `parallelStream()` |
| Laziness | N/A (eager step-by-step) | Intermediate ops are lazy |
| Use case | Fine-grained control, safe removal during traversal | Data transformation pipelines, aggregation |

### 7.5 `Spliterator` — what powers Stream iteration internally
```java
import java.util.*;

public class SpliteratorDemo {
    public static void main(String[] args) {
        List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8);
        Spliterator<Integer> spliterator = list.spliterator();

        // tryAdvance: process one element, like Iterator.next() but returns boolean
        spliterator.tryAdvance(System.out::println); // prints 1

        // trySplit: splits the remaining elements into two Spliterators - basis of parallel streams
        Spliterator<Integer> other = spliterator.trySplit();
        System.out.println("Split 1 remaining estimate: " + spliterator.estimateSize());
        System.out.println("Split 2 remaining estimate: " + other.estimateSize());

        other.forEachRemaining(System.out::println);
        spliterator.forEachRemaining(System.out::println);

        // characteristics(): ORDERED, SIZED, DISTINCT, SORTED, IMMUTABLE, CONCURRENT, etc.
        // These hints let Stream operations optimize (e.g., skip a sort if SORTED is already set).
        System.out.println(Spliterator.ORDERED & list.spliterator().characteristics());
    }
}
```

**Practice tasks for Section 7:**
1. Implement a custom `Iterable<T>` for a simple linked-list class, then consume it both via a for-each loop and via `StreamSupport.stream()`.
2. Demonstrate `ConcurrentModificationException` and fix it three different ways: `Iterator.remove()`, `removeIf()`, and collecting into a new list.

---

## 8. Optional (tightly coupled with Streams)

```java
import java.util.*;
import java.util.stream.*;

public class OptionalDemo {
    record User(String name, String email) {}

    public static void main(String[] args) {
        List<User> users = List.of(new User("Alice", "alice@x.com"), new User("Bob", null));

        // Creating Optionals
        Optional<String> present = Optional.of("value");          // throws NPE if null passed
        Optional<String> empty = Optional.empty();
        Optional<String> nullable = Optional.ofNullable(null);    // safe for possibly-null values

        // Common combinators
        Optional<User> found = users.stream().filter(u -> u.name().equals("Alice")).findFirst();

        found.ifPresent(u -> System.out.println("Found: " + u.name()));
        found.ifPresentOrElse(
            u -> System.out.println("Found: " + u.name()),
            () -> System.out.println("Not found")
        );

        String name = found.map(User::name).orElse("Unknown");
        System.out.println(name);

        // orElseGet: lazy - only invoked if empty (prefer over orElse() for expensive defaults)
        String name2 = Optional.<User>empty().map(User::name).orElseGet(() -> "computed default");
        System.out.println(name2);

        // orElseThrow
        try {
            Optional.<User>empty().orElseThrow(() -> new NoSuchElementException("no user"));
        } catch (NoSuchElementException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        // flatMap on Optional - chaining nested Optionals (e.g., avoiding Optional<Optional<T>>)
        Optional<User> maybeUser = users.stream().filter(u -> u.name().equals("Bob")).findFirst();
        Optional<String> email = maybeUser.flatMap(u -> Optional.ofNullable(u.email()));
        System.out.println(email.orElse("no email"));

        // Optional -> Stream (Java 9+): useful for flattening a List<Optional<T>>
        List<Optional<String>> emails = users.stream()
            .map(u -> Optional.ofNullable(u.email()))
            .toList();
        List<String> validEmails = emails.stream()
            .flatMap(Optional::stream)
            .toList();
        System.out.println(validEmails); // [alice@x.com]

        // ANTI-PATTERN: calling .get() without checking isPresent()/isEmpty() first
        // ANTI-PATTERN: using Optional as a field type or method parameter (only use as RETURN type)
        // ANTI-PATTERN: Optional.of(null) -> throws NPE immediately, use ofNullable instead
    }
}
```

---

## 9. Primitive Streams — `IntStream`, `LongStream`, `DoubleStream`

**Why they exist:** avoid autoboxing overhead (`Integer` object per `int`) for numeric-heavy pipelines, and provide numeric-specific terminal ops (`sum()`, `average()`, `summaryStatistics()`).

```java
import java.util.*;
import java.util.stream.*;

public class PrimitiveStreamsDemo {
    public static void main(String[] args) {
        IntStream is = IntStream.rangeClosed(1, 10);
        System.out.println(is.sum()); // 55

        OptionalDouble avg = IntStream.rangeClosed(1, 10).average();
        System.out.println(avg.getAsDouble()); // 5.5

        IntSummaryStatistics stats = IntStream.rangeClosed(1, 10).summaryStatistics();
        System.out.println(stats.getMax() + " " + stats.getMin() + " " + stats.getSum());

        // Converting between primitive and object streams
        Stream<Integer> boxed = IntStream.rangeClosed(1, 5).boxed();
        IntStream unboxed = boxed.mapToInt(Integer::intValue);

        // mapToObj: primitive -> object stream
        List<String> strs = IntStream.rangeClosed(1, 5)
            .mapToObj(i -> "Item" + i)
            .toList();
        System.out.println(strs);

        // asLongStream / asDoubleStream: widening conversions
        LongStream longs = IntStream.rangeClosed(1, 5).asLongStream();
        DoubleStream doubles = IntStream.rangeClosed(1, 5).asDoubleStream();

        // No IntStream.collect(Collectors...) directly - Collectors work on Stream<T>, not primitive streams
        // But IntStream has its own collect(supplier, accumulator, combiner):
        List<Integer> collected = IntStream.rangeClosed(1, 5)
            .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
        System.out.println(collected);
    }
}
```

---

## 10. Common Interview Questions & Gotchas (Rapid Fire)

**Q1: Difference between `map()` and `flatMap()`?**
`map()` is 1-to-1 (`T -> R`); `flatMap()` is 1-to-many where each element maps to a Stream, and those streams are flattened into a single stream (`T -> Stream<R>`, flattened).

**Q2: Can you reuse a stream?**
No — once a terminal operation runs, calling any operation on the same stream throws `IllegalStateException`. Create a new stream from the source instead.

**Q3: Difference between intermediate and terminal operations?**
Intermediate: lazy, returns `Stream`, can be chained (e.g., `filter`, `map`, `sorted`). Terminal: eager, triggers execution, returns non-Stream (`void`, value, collection), closes the stream (e.g., `collect`, `forEach`, `reduce`).

**Q4: Why are Streams lazy?**
Enables optimizations: short-circuiting (`findFirst`, `limit`, `anyMatch`), avoiding unnecessary work, and fusing operations into a single pass over the data instead of materializing intermediate collections at every stage.

**Q5: `Collectors.toList()` vs `Stream.toList()` (Java 16+)?**
`Stream.toList()` is a shorthand that always returns an **immutable** list. `Collectors.toList()` makes no guarantee about mutability (implementation detail — historically `ArrayList`, mutable in practice, but you shouldn't rely on that).

**Q6: How does `distinct()` work internally?**
Uses `equals()`/`hashCode()` — internally backed by a `HashSet`-like structure to track seen elements while preserving encounter order for ordered streams.

**Q7: What happens if you call `sorted()` on an infinite stream?**
It never terminates (or throws `OutOfMemoryError`) because `sorted()` is a **stateful** operation that must buffer the entire stream before it can emit anything. You must `limit()` before `sorted()` on infinite streams, or avoid sorting infinite streams altogether.

```java
// Demonstrates stateful vs stateless intermediate ops
// Stateless (per-element, no memory of prior elements): filter, map, peek
// Stateful (must see multiple/all elements): distinct, sorted, limit, skip
```

**Q8: Why does `peek()` sometimes not execute?**
JIT/stream optimizations can elide `peek()` calls if the pipeline determines the result isn't needed (e.g., `count()` on a `SIZED` source may skip traversal entirely in some JDK versions). Never use `peek()` for anything with side effects that matter to correctness — use `map()` or `forEach()` instead.

**Q9: `Comparator.comparing()` vs `thenComparing()`?**
```java
import java.util.*;
record Emp(String name, int age, double salary) {}
// Sort by age, tie-break by salary descending, then by name
Comparator<Emp> cmp = Comparator.comparingInt(Emp::age)
    .thenComparing(Comparator.comparingDouble(Emp::salary).reversed())
    .thenComparing(Emp::name);
```

**Q10: How do you convert a `Stream` back to an array, `List`, `Set`, `Map`?**
`toArray()`, `.toList()`/`collect(Collectors.toList())`, `collect(Collectors.toSet())`, `collect(Collectors.toMap(...))`.

**Q11: What's the difference between `Collectors.groupingBy` and `Collectors.partitioningBy`?**
`groupingBy` can produce any number of groups keyed by an arbitrary classifier function. `partitioningBy` always produces exactly two groups keyed by `Boolean` (true/false), even if one is empty.

**Q12: Is a stream pipeline guaranteed to execute in the order operations are written?**
Intermediate operations execute *per element*, vertically, in the order written, but the *overall* traversal order across elements depends on encounter order (which parallel streams may not preserve unless you use `forEachOrdered`).

**Q13: How would you process a huge file without loading it all into memory, using Streams?**
```java
import java.nio.file.*;
import java.io.IOException;
try (var lines = Files.lines(Path.of("huge.txt"))) {
    long errorCount = lines.filter(l -> l.contains("ERROR")).count();
    System.out.println(errorCount);
} catch (IOException e) { e.printStackTrace(); }
// Files.lines() streams lazily line-by-line rather than loading the whole file into a List first.
```

**Q14: Explain short-circuiting operations. Name them.**
Operations that can terminate processing before examining every element: `anyMatch`, `allMatch`, `noneMatch`, `findFirst`, `findAny`, `limit`. Combined with lazy evaluation, they let you avoid processing an entire (even infinite) stream.

**Q15: Why shouldn't you use `Collectors.toMap()` blindly on data that might have duplicate keys?**
It throws `IllegalStateException` at runtime on the *second* duplicate key encountered, unless you supply a merge function (3rd arg) — a very common production bug when data model assumptions change.

**Q16: What's the difference between `Stream.concat()` behavior and simply merging two lists?**
```java
import java.util.stream.*;
Stream<Integer> a = Stream.of(1, 2, 3);
Stream<Integer> b = Stream.of(4, 5, 6);
Stream<Integer> combined = Stream.concat(a, b);
combined.forEach(System.out::println);
// Gotcha: deeply nested Stream.concat() calls can cause StackOverflowError for many streams -
// prefer flatMap over a collection of streams for combining many sources.
```

**Q17: Explain the difference between `Collectors.summingInt` and just using `mapToInt(...).sum()`.**
Functionally similar for a flat stream, but `Collectors.summingInt` is meant as a **downstream collector** (e.g., inside `groupingBy`), whereas `mapToInt().sum()` works directly on a stream and can't be nested as a downstream collector argument.

**Q18: Can a Stream pipeline throw checked exceptions inside a lambda directly?**
No — functional interfaces like `Function`, `Predicate`, etc. don't declare checked exceptions. You must wrap checked exceptions (try/catch inside the lambda, rethrow unchecked) or write a custom functional interface that permits `throws`.
```java
import java.util.*;
import java.util.stream.*;
import java.nio.file.*;

List<String> paths = List.of("a.txt", "b.txt");
List<String> contents = paths.stream()
    .map(p -> {
        try {
            return Files.readString(Path.of(p));
        } catch (java.io.IOException e) {
            throw new RuntimeException(e); // wrap checked -> unchecked
        }
    })
    .toList();
```

**Q19: What does `IntStream.range(0, list.size())` pattern let you do that a plain `.stream()` doesn't?**
Access the index alongside the element (Streams don't have a built-in indexed-forEach):
```java
import java.util.*;
import java.util.stream.*;
List<String> names = List.of("a", "b", "c");
IntStream.range(0, names.size())
    .forEach(i -> System.out.println(i + ": " + names.get(i)));
```

**Q20: How is `Stream` different from `RxJava`/reactive streams?**
Java Streams are **pull-based, synchronous, single-threaded-per-element (or fork/join for parallel), one-time-use, no built-in backpressure or async operators**. Reactive streams (RxJava, Project Reactor) are **push-based, asynchronous, support backpressure, hot/cold sources, and are reusable/subscribable multiple times**.

---

## 11. Practice Checklist

Work through these without looking at the answers first — that's the whole point.

- [ ] Create a stream 6 different ways (Collection, `Stream.of`, `Arrays.stream`, `Stream.iterate`, `Stream.generate`, `IntStream.range`).
- [ ] Chain `filter → map → sorted → limit → collect` on a `List<Employee>`-like object.
- [ ] Use `flatMap` to flatten a nested list and also to split+flatten sentences into words.
- [ ] Use `reduce` with 1-arg, 2-arg, and 3-arg overloads; explain when the combiner is invoked.
- [ ] Build 5 different `groupingBy` pipelines: simple, with `counting()`, with `mapping()`, with nested `groupingBy`, with a custom `TreeMap` supplier.
- [ ] Use `partitioningBy` and explain why the result always has both `true`/`false` keys.
- [ ] Write a custom `Collector` using `Collector.of()`.
- [ ] Convert a sequential pipeline to parallel, benchmark both, and explain the result.
- [ ] Reproduce a parallel-stream race condition bug, then fix it using `Collectors.groupingBy` or `AtomicInteger`.
- [ ] Implement a custom `Iterable`/`Iterator`, then stream it via `StreamSupport`.
- [ ] Demonstrate and fix a `ConcurrentModificationException` three different ways.
- [ ] Use `Optional` chaining (`map`, `flatMap`, `filter`, `orElseGet`, `orElseThrow`) to avoid null checks in a small program.
- [ ] Use `teeing()` to compute two aggregates in a single pass.
- [ ] Explain out loud (mock interview style) all 20 questions in Section 10 without notes.

---

### Final tip for the interview
Interviewers love asking you to **trace output line-by-line** for a tricky pipeline involving `peek`, `limit`, laziness, or parallel non-determinism. Practice reading pipelines top-to-bottom, one element at a time through the *entire* pipeline (vertical execution), not stage-by-stage — that mental model answers 80% of "what will this print" questions.