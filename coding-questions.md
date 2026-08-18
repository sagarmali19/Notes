# Java Backend Developer — Hands-On Interview Practice List

For 3+ YOE. Solve each on your IDE without looking at solutions first, then compare with an optimal approach. Aim for 2-3 problems/day per topic block.

---

## 1. Arrays

- [ ] Two Sum (return indices of pair summing to target)
- [ ] Find the duplicate number in an array of n+1 integers (1..n range)
- [ ] Find the missing number in array of 1..n
- [ ] Rotate array by k positions (in-place)
- [ ] Kadane's Algorithm — maximum subarray sum
- [ ] Merge two sorted arrays into one sorted array
- [ ] Find a pair with a given sum in a sorted array (two-pointer)
- [ ] Move all zeroes to the end without changing order of other elements
- [ ] Find intersection / union of two arrays
- [ ] Find the second largest element without sorting
- [ ] Rearrange array so positive and negative numbers alternate
- [ ] Rotate an N×N matrix by 90 degrees in-place
- [ ] Trapping Rain Water (harder — good for product-based companies)

## 2. Strings

- [ ] Reverse a string without using built-in reverse
- [ ] Check if a string is a palindrome
- [ ] Check if two strings are anagrams
- [ ] Find the first non-repeating character in a string
- [ ] Count occurrences of each character (using array/HashMap)
- [ ] String compression: `"aabcccccaaa"` → `"a2b1c5a3"`
- [ ] Check if one string is a rotation of another
- [ ] Longest substring without repeating characters
- [ ] Reverse the words in a sentence (`"I am a dev"` → `"dev a am I"`)
- [ ] Valid Parentheses (stack-based — very common)
- [ ] Implement your own `atoi` (String to int without `parseInt`)
- [ ] Group anagrams from a list of strings

## 3. Exception Handling

- [ ] Create a custom **checked** exception (e.g. `InsufficientFundsException`) and use it in a mini banking transfer method
- [ ] Create a custom **unchecked** exception (e.g. `InvalidOrderStateException`)
- [ ] Implement `try-with-resources` using your own class that implements `AutoCloseable`
- [ ] Write a multi-catch block example (`catch (IOException | SQLException e)`)
- [ ] Demonstrate exception chaining — catch a `SQLException`, wrap it in a custom `ServiceException` with `cause` set
- [ ] Write a method showing execution order of `try/catch/finally` when `return` appears in all three
- [ ] Build a small validation utility that throws different custom exceptions for different failures (null check, range check, format check)
- [ ] Simulate a service layer that throws `ResourceNotFoundException` / `DuplicateResourceException`, and write a simple "handler" method that catches and maps them to different response codes (mimics `@ExceptionHandler` logic, without Spring)

## 4. OOP Concepts

- [ ] Design a `Shape` hierarchy (`Circle`, `Rectangle`, `Triangle`) demonstrating abstraction + polymorphism, compute area via a common method
- [ ] Demonstrate method overloading vs overriding with a runnable example, explain compile-time vs runtime binding
- [ ] Implement Singleton 4 ways: eager init, lazy init, double-checked locking, enum-based — explain trade-offs
- [ ] Implement Factory pattern (e.g. `VehicleFactory` returning `Car`/`Bike`)
- [ ] Implement Builder pattern (e.g. build an `HttpRequest` or `Pizza` object with optional params)
- [ ] Demonstrate composition vs inheritance with two versions of the same problem (`Car has-a Engine` vs `Car is-a Vehicle`)
- [ ] Create an interface with a `default` and a `static` method; create two interfaces with the same default method and resolve the diamond conflict in the implementing class
- [ ] Build a `BankAccount` class demonstrating proper encapsulation (private fields, validated setters)
- [ ] Override `equals()` and `hashCode()` for a custom `Employee` class — show what breaks in a `HashSet`/`HashMap` if you forget `hashCode()`
- [ ] Implement `Comparable` on `Employee` (natural ordering by id) and a separate `Comparator` (by salary, then name)
- [ ] Implement a simple Observer pattern (e.g. `StockPriceNotifier` with multiple subscribers)
- [ ] Explain + code abstract class vs interface — same behavior implemented both ways

## 5. Collections Framework

- [ ] Implement an LRU Cache using `LinkedHashMap` (override `removeEldestEntry`)
- [ ] Remove duplicates from a `List` while preserving order
- [ ] Find frequency of each element in a list using `HashMap`
- [ ] Sort a `HashMap` by its values
- [ ] Sort a `List<Employee>` by multiple fields using chained `Comparator`s
- [ ] Benchmark `ArrayList` vs `LinkedList` for add/get/remove at start/middle/end — explain results
- [ ] Reproduce a `ConcurrentModificationException` by modifying a list while iterating, then fix it using `Iterator.remove()` and again using `CopyOnWriteArrayList`
- [ ] Implement Producer-Consumer using `BlockingQueue`
- [ ] Group a `List<Employee>` by department manually with `HashMap<String, List<Employee>>` (no streams)
- [ ] Implement a `Stack` using `ArrayDeque`
- [ ] Use `TreeMap`/`TreeSet` with a custom `Comparator` to keep data sorted (e.g. leaderboard by score)
- [ ] Merge two `Map<String,Integer>` objects, summing values on key conflicts
- [ ] Explain and demonstrate `HashMap` vs `Hashtable` vs `ConcurrentHashMap` thread-safety differences with a small multi-thread test

## 6. Java 8 Features

- [ ] Rewrite a for-loop as a Stream pipeline (`filter` → `map` → `collect`)
- [ ] Find max/min/sum/average of a `List<Employee>` salary using Streams
- [ ] Group employees by department using `Collectors.groupingBy`, then count per group
- [ ] Use `Collectors.partitioningBy` (e.g. split students into pass/fail by marks)
- [ ] Sort `List<Employee>` using `Stream.sorted()` with `Comparator.comparing().thenComparing()`
- [ ] Refactor a method that returns `null` to return `Optional<T>` instead; chain `map`/`orElse`/`orElseThrow`
- [ ] Write your own functional interface (`@FunctionalInterface`) and use it with a lambda
- [ ] Use static, instance, and constructor method references in three different scenarios
- [ ] Use `flatMap` to flatten a `List<List<Integer>>` into a single list
- [ ] Build a word-frequency counter from a sentence/text using Streams + `Collectors.toMap`
- [ ] Use `Stream.reduce()` to compute a factorial or running total
- [ ] Compare `parallelStream()` vs `stream()` on a large list — time it, then discuss when parallel streams are unsafe (shared mutable state)
- [ ] Use `Collectors.toMap` with a merge function to handle duplicate keys
- [ ] Simulate pagination using `skip()` and `limit()` on a stream

## 7. Generics

- [ ] Write a generic class `Box<T>` with a generic `set`/`get`
- [ ] Write a generic method `<T extends Comparable<T>> T max(T a, T b)`
- [ ] Implement a generic `Pair<K, V>` class with `equals`/`hashCode`/`toString`
- [ ] Demonstrate a bounded type parameter (`<T extends Number>`) and why it's needed
- [ ] Demonstrate wildcards: write `copy(List<? extends T> src, List<? super T> dest)`
- [ ] Implement a generic `Stack<T>` backed by an array or `LinkedList`
- [ ] Write a generic `Repository<T, ID>` interface with `save`, `findById`, `deleteById` (mimics Spring Data JPA's `CrudRepository` — very relevant for backend roles)
- [ ] Demonstrate type erasure — show why `new T[10]` doesn't compile and how to work around it
- [ ] Write a generic utility method that behaves like `BiFunction<T, U, R>`

## 8. Multithreading

- [ ] Create threads 3 ways: extending `Thread`, implementing `Runnable`, implementing `Callable` — compare
- [ ] Implement Producer-Consumer using `wait()`/`notify()`
- [ ] Implement the same Producer-Consumer using `BlockingQueue` — compare code complexity
- [ ] Demonstrate `synchronized` method vs `synchronized` block on a shared counter
- [ ] Implement thread-safe Singleton using double-checked locking with `volatile`
- [ ] Use `ExecutorService` (`Executors.newFixedThreadPool`) to run multiple tasks, collect results via `Future`
- [ ] Use `CountDownLatch` to simulate "wait until 3 services finish starting up"
- [ ] Use `CyclicBarrier` to simulate threads waiting for each other at a checkpoint
- [ ] Use `ReentrantLock` (with `tryLock`) and compare against `synchronized`
- [ ] Deliberately create a deadlock with two locks acquired in different order, then fix it via consistent lock ordering
- [ ] Implement a thread-safe counter using `AtomicInteger`, compare performance/correctness vs a `synchronized` counter
- [ ] Use `CompletableFuture` to chain async calls: `thenApply`, `thenCombine`, `exceptionally`
- [ ] Demonstrate the `volatile` keyword fixing a visibility bug (a flag not being seen by another thread)
- [ ] Use `ReadWriteLock` for a simple cache with many readers and few writers

---

## Suggested order
1. Arrays + Strings (warm-up, build coding speed)
2. OOP + Exception Handling (core fundamentals interviewers probe deeply for 3+ YOE)
3. Collections (very commonly asked, pairs well with OOP's equals/hashCode)
4. Generics (short topic, do after Collections since Collections use generics heavily)
5. Java 8 (Streams/Optional — expect scenario-based questions, not just syntax)
6. Multithreading (hardest, do last, expect "explain what happens" follow-ups)

## Tips for practicing
- Don't just solve — say the time/space complexity out loud for Arrays/Strings problems.
- For OOP/Collections/Multithreading, be ready to explain **why**, not just produce working code — that's what gets asked in 3+ YOE interviews.
- For Java 8 and Generics, try writing the same solution both the "old way" (loops) and "new way" (streams) so you can explain trade-offs when asked.