# Java Multithreading — Interview-Ready Notes

**How to use this document:** Read a section, run the code examples yourself (don't just read them — type them out and execute), then attempt every practice question at the end of that section *without* looking at the example. Come back to questions you couldn't solve after a day. By the end, you should be able to explain every boxed concept out loud and defend your code choices in an interview.

---

## Table of Contents
1. Fundamentals — Program, Process, Thread, Multitasking, Multithreading
2. Creating Threads (Thread class, Runnable, Lambda)
3. Thread Lifecycle & States
4. Thread Methods (sleep, join, interrupt, yield, daemon, priority)
5. Synchronization & Race Conditions
6. Locks — Intrinsic vs Explicit, `Lock` interface
7. ReentrantLock (deep dive)
8. Lock Fairness
9. ReadWriteLock / ReentrantReadWriteLock
10. Deadlock
11. Inter-thread Communication (wait/notify/notifyAll)
12. Thread Pools & the Executor Framework
13. Callable, Future, and `invokeAll`/`invokeAny`
14. ScheduledExecutorService
15. CountDownLatch
16. CyclicBarrier
17. CompletableFuture
18. Volatile & Atomic Classes
19. Classic Concurrency Problems (Producer-Consumer, etc.)
20. Rapid-Fire Interview Q&A Bank
21. Final Coding Challenge Set (mixed, mock-interview style)

---

## 1. Fundamentals

| Term | Meaning |
|---|---|
| **CPU** | The brain of the computer. A **core** is the unit inside a CPU that actually executes instructions. |
| **Program** | A set of instructions written in a coding language, sitting on disk (passive). |
| **Process** | An **instance of a running program**. When you run a program, the OS creates a process to manage its execution (own memory space, own resources). |
| **Thread** | The **smallest unit of execution within a process**. A process can have multiple threads; threads within the same process **share memory/resources** but run independently. |
| **Multitasking** | Running multiple *processes* seemingly at once. On a single core this is done via rapid **time-slicing/context switching** (illusion of parallelism). On multi-core, tasks are genuinely distributed across cores. |
| **Multithreading** | The ability to execute **multiple threads within a single process** concurrently. |

**Concrete example (from the notes):**
- **Program** → Google Chrome (the installed app)
- **Process** → the running Chrome instance when you open it
- **Multitasking** → VLC playing music, Chrome downloading a file, and MS Word open, all "at once"
- **Multithreading** (inside one Chrome process) → one thread plays a YouTube video, another thread downloads a file, another renders the home page — all sharing the same process memory

| | Single Core | Multi Core |
|---|---|---|
| Mechanism | Time-slicing + context switching → *illusion* of parallelism | True parallelism — tasks distributed across different cores |

> **Interview one-liner:** "A process is an isolated instance of a running program with its own memory; a thread is a lightweight sub-unit of a process that shares that memory with sibling threads. Multithreading gives you concurrency (and true parallelism on multi-core) without the overhead of spinning up a whole new process."

### Practice Questions — Section 1
1. Explain the difference between a process and a thread to a 5-year-old, then to a senior engineer.
2. Why is creating a new thread cheaper than creating a new process?
3. Give a real-world example (not from these notes) of multitasking vs multithreading.
4. Is multithreading always faster than single-threaded code? Give a scenario where adding threads makes a program *slower*.
5. On a single-core machine, does multithreading actually improve throughput for CPU-bound work? What about I/O-bound work? Explain why.

---

## 2. Creating Threads in Java

JVM starts with a single **main thread**. You can inspect it:
```java
public class MainThreadDemo {
    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName()); // "main"
    }
}
```

There are two classic ways to create a new thread, plus a modern lambda shortcut.

### 2.1 Extend the `Thread` class
```java
class World extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println("World thread: " + i);
        }
    }
}

public class Main {
    public static void main(String[] args) {
        World obj = new World();
        obj.start();          // starts a NEW thread; JVM will call run() on it
        for (int i = 0; i < 5; i++) {
            System.out.println("Main thread: " + i);
        }
    }
}
```
> **Note the interleaving** — output order between "Main thread" and "World thread" lines is **not guaranteed**. That non-determinism is the whole point of this topic.

### 2.2 Implement the `Runnable` interface (preferred)
```java
class World implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " is running");
    }
}

public class Main {
    public static void main(String[] args) {
        World world = new World();          // 1. instance of the task
        Thread t1 = new Thread(world);      // 2. thread object wrapping the task
        t1.start();                         // 3. start() → run() executes on new thread
    }
}
```

### 2.3 Lambda expression (Runnable is a functional interface)
```java
public class Main {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> System.out.println("Running via lambda"));
        t1.start();
    }
}
```

### When to extend Thread vs implement Runnable?
- If your class **already extends** some other class (Java has no multiple inheritance) → you **must** implement `Runnable`.
- If it doesn't need to extend anything else → either works, but **prefer `Runnable`**: it separates "the task" from "the thread that runs it," lets multiple threads share the same task object, and lets you submit the task to an `ExecutorService` (Thread objects can't be reused once run).

> **Critical distinction interviewers love:** calling `run()` directly does **NOT** start a new thread — it just calls a normal method on the current thread. Only `start()` spins up a new call stack / OS thread.
```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.run();    // prints "main"  -> ran on the CURRENT thread, no concurrency!
t.start();  // prints "Thread-0" (or similar) -> actually runs concurrently
```

### Practice Questions — Section 2
1. Write a program that creates 3 threads, each printing its own name and numbers 1–5, using **all three** approaches (extend Thread, implement Runnable, lambda) — one thread per approach.
2. What happens if you call `start()` twice on the same `Thread` object? (Try it — read the exception name and explain why the JVM enforces this.)
3. Write a `Runnable` task that computes the square of a number passed via constructor, then run the *same* Runnable instance from 3 different `Thread` objects.
4. Explain, with code, why calling `t.run()` instead of `t.start()` defeats the purpose of multithreading.
5. Create an anonymous inner class implementation of `Runnable` (without lambda) that prints "Hello from anonymous thread".

---

## 3. Thread Lifecycle & States

A thread moves through these states (`Thread.State` enum):

| State | Meaning |
|---|---|
| **NEW** | Thread object created but `start()` not yet called. |
| **RUNNABLE** | `start()` called; thread is either running or waiting for CPU time. (Java merges "ready" and "running" into one state — there is no separate "Running" state in the enum; it's called Runnable even while actually executing.) |
| **BLOCKED** | Waiting to acquire a lock held by another thread (e.g., waiting to enter a `synchronized` block). |
| **WAITING** | Waiting indefinitely for another thread to signal it (e.g., called `wait()` with no timeout, or `join()` with no timeout). |
| **TIMED_WAITING** | Waiting for a specified time — e.g., `Thread.sleep(1000)`, `wait(timeout)`, `join(timeout)`. |
| **TERMINATED** | Thread has finished executing (`run()` returned, or an uncaught exception killed it). |

```java
public class LifecycleDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                System.out.println(e);
            }
        });

        System.out.println(t1.getState());  // NEW
        t1.start();
        System.out.println(t1.getState());  // RUNNABLE (or TIMED_WAITING if it already hit sleep)
        Thread.sleep(500);
        System.out.println(t1.getState());  // TIMED_WAITING
        t1.join();
        System.out.println(t1.getState());  // TERMINATED
    }
}
```

**`join()` explained:** `t1.join()` makes the **calling thread** (often `main`) wait until `t1` finishes execution. After `join()` returns, `t1` is guaranteed to be in `TERMINATED` state.

```java
class World extends Thread {
    public void run() {
        for (int i = 0; i < 3; i++) System.out.println("World: " + i);
    }
}
public class Main {
    public static void main(String[] args) throws InterruptedException {
        World w = new World();
        w.start();
        w.join();  // main waits here until w finishes
        System.out.println("This prints only after World thread is done");
    }
}
```

### Practice Questions — Section 3
1. Write code that prints the `Thread.State` of a worker thread at 5 different points in its lifecycle (NEW, RUNNABLE, TIMED_WAITING, WAITING via a lock, TERMINATED).
2. What state is a thread in while it's *blocked waiting to enter a synchronized block* held by another thread? Verify by writing two threads contending for one lock and printing state from a third "observer" thread.
3. Remove the `join()` call from the example above — what changes about the output order? Run it 5 times and note whether the order is deterministic.
4. Can a `TERMINATED` thread be `start()`-ed again? Try it and name the exception.
5. Write a program with 3 worker threads where `main` uses `join()` on each to wait for all of them before printing "All done".

---

## 4. Thread Methods (sleep, join, interrupt, yield, daemon, priority)

| Method | What it does |
|---|---|
| `start()` | Creates a new call stack and JVM eventually calls `run()` on it. |
| `run()` | Contains the logic to be executed by the thread. |
| `sleep(ms)` | **Static** method — puts the *currently executing* thread into `TIMED_WAITING` for the given duration. Does not release any locks it holds. |
| `join()` / `join(ms)` | Calling thread waits for the target thread to finish (or for the timeout). |
| `setPriority(int)` | Hints the JVM to prefer scheduling this thread. Range: `MIN_PRIORITY=1`, `NORM_PRIORITY=5` (default), `MAX_PRIORITY=10`. **Not a guarantee**, purely a hint to the scheduler. |
| `interrupt()` | Asks a thread to stop what it's doing (e.g., wake it up if sleeping/waiting) and sets its "interrupted" flag. Doesn't forcibly kill it — cooperative cancellation. |
| `yield()` | Hints to the scheduler "I'm willing to pause and let another thread of the same priority run." Not a guarantee either. |
| `setDaemon(true)` | Marks a thread as a **daemon** (background) thread. JVM exits once all **user (non-daemon)** threads finish — it does **not** wait for daemon threads. |
| `Thread.currentThread().getName()` | Returns the name of the currently executing thread. |

### 4.1 `sleep()` + `InterruptedException`
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    System.out.println(e);
}
```
If another thread calls `interrupt()` on a sleeping thread, the sleep is broken early and an `InterruptedException` is thrown (the sleeping thread moves out of `TIMED_WAITING`).

### 4.2 `interrupt()` in action
```java
public class InterruptDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(5000);
                System.out.println("Finished sleeping normally");
            } catch (InterruptedException e) {
                System.out.println("Sleep was interrupted!");
                // best practice: restore the interrupted status if you swallow it here
                Thread.currentThread().interrupt();
            }
        });
        t1.start();
        Thread.sleep(1000);   // let t1 go to sleep first
        t1.interrupt();       // wake it up early
    }
}
```
> **Rule of thumb the notes emphasize:** always handle interruption in the `catch` block — typically by calling `Thread.currentThread().interrupt()` again so callers up the stack still see that this thread was interrupted, or by cleanly exiting the task.

### 4.3 `setPriority()` and naming threads
```java
Thread t1 = new Thread(() -> System.out.println("high priority task"));
t1.setPriority(Thread.MAX_PRIORITY);   // 10 — just a hint
Thread t2 = new Thread(() -> {}, "WorkerThread-1"); // pass name via constructor
System.out.println(t2.getName());  // WorkerThread-1
```

### 4.4 `yield()`
```java
Runnable task = () -> {
    for (int i = 0; i < 3; i++) {
        System.out.println(Thread.currentThread().getName() + " : " + i);
        Thread.yield(); // hint scheduler to let another equal-priority thread run
    }
};
new Thread(task, "T1").start();
new Thread(task, "T2").start();
```

### 4.5 Daemon threads
```java
Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("Background heartbeat...");
        try { Thread.sleep(500); } catch (InterruptedException e) { break; }
    }
});
daemon.setDaemon(true);  // MUST be set BEFORE start()
daemon.start();

System.out.println("Main thread finishing.");
// JVM exits once main (user thread) finishes, killing the daemon
// even though its while(true) loop never naturally ends.
```
> Classic real example: JVM's own Garbage Collector thread is a daemon thread.

### Practice Questions — Section 4
1. Write a program with two threads that alternate printing numbers 1–10 using `yield()`, and observe (it won't be perfectly alternating — explain why).
2. Demonstrate `setDaemon(true)` by writing an infinite-loop background thread that never prints "done" because the JVM exits before it can.
3. Write a program where thread `t1` sleeps for 5 seconds, but `main` interrupts it after 1 second. Catch the `InterruptedException` and print a custom message.
4. What is the default priority of a newly created thread if you never call `setPriority()`? Verify with `getPriority()`.
5. Try calling `setDaemon(true)` **after** calling `start()`. What exception do you get, and why does that ordering matter?
6. Explain the difference between `sleep()` and `wait()` in one sentence each (preview for Section 11) — what does each do to a held lock?

---

## 5. Synchronization & Race Conditions

**Race condition:** when two or more threads access a **shared resource** concurrently and the final result depends on unpredictable timing/interleaving of their execution — producing inconsistent or wrong results.

### The classic broken counter
```java
class Counter {
    private int count = 0;
    public void increment() {
        count++;   // NOT ATOMIC! It's really: read count, add 1, write count back
    }
    public int getCount() { return count; }
}

public class RaceConditionDemo {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Runnable task = () -> {
            for (int i = 0; i < 100000; i++) counter.increment();
        };
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join();  t2.join();
        System.out.println("Final count: " + counter.getCount());
        // Expected 200000, but you will almost always get LESS — a race condition!
    }
}
```
Why? `count++` compiles to 3 steps (load, increment, store). If both threads load the same value before either stores, one increment is lost.

### The fix: `synchronized`
```java
class Counter {
    private int count = 0;

    // Option A: synchronized method — locks on "this"
    public synchronized void increment() {
        count++;
    }

    // Option B: synchronized block — locks on an explicit object, more granular
    private final Object lock = new Object();
    public void incrementV2() {
        synchronized (lock) {
            count++;
        }
    }

    public int getCount() { return count; }
}
```
Now only **one thread at a time** can execute the critical section → **mutual exclusion** is guaranteed, and the race condition disappears (run the demo above with `synchronized` and you'll reliably get 200000).

> **Interview definition to memorize:** *"`synchronized` ensures mutual exclusion — only one thread can hold the monitor lock of a given object at a time, so the critical section it guards is executed atomically with respect to other threads synchronizing on the same object."*

### Practice Questions — Section 5
1. Reproduce the broken counter above, run it 5 times, and record the actual output each time. Then fix it with `synchronized` and confirm you always get the expected value.
2. What's the difference between synchronizing on `this` (method-level `synchronized`) vs synchronizing on a private lock object? Why might the private-lock approach be safer in library code?
3. Write a `BankAccount` class with `deposit()` and `withdraw()` methods that must be thread-safe. Spawn 10 threads each depositing 100 units concurrently; verify the final balance is exactly 1000× the per-thread total.
4. Can two different threads run two **different synchronized methods of the same object** at the same time if they synchronize on the same monitor? Why or why not?
5. Does `synchronized` protect a *field read* the same way it protects a write? Investigate/explain what happens if only the writer method is synchronized but the reader method isn't (tie this back to Section 18 — visibility vs atomicity).
6. Convert the `Counter` class to use a `synchronized` **static** method instead of instance method — what does the lock now apply to, and why does that matter if you create multiple `Counter` objects?

---

## 6. Locks — Intrinsic vs Explicit

**Analogy from the notes:** Imagine a shared notebook and 3 students (threads S1, S2, S3). Whoever wants to write in the notebook must first acquire a lock on it. `synchronized` gives you this automatically ("intrinsic" lock, built into every Java object); `java.util.concurrent.locks` gives you the same idea but with far more control ("explicit" locks).

| | Intrinsic Lock (`synchronized`) | Explicit Lock (`java.util.concurrent.locks.Lock`) |
|---|---|---|
| Where it comes from | Built into every object, used via the `synchronized` keyword | `ReentrantLock`, `ReentrantReadWriteLock` etc. from `java.util.concurrent.locks` |
| Acquire/release | Automatic (block entry/exit, including on exceptions) | Manual — you must call `lock()` to acquire and `unlock()` (in a `finally` block!) to release |
| Waiting behavior | Other thread waits **indefinitely**, uninterruptibly | Can `tryLock()` (fail fast), `tryLock(timeout, unit)` (wait up to a limit), or `lockInterruptibly()` (can be interrupted while waiting) |
| Fairness | No control | Can request a **fair** lock (FIFO ordering) |
| Read/Write split | No | Yes, via `ReentrantReadWriteLock` |

### 6.1 Explicit Lock API
```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();

    public void increment() {
        lock.lock();                 // acquire
        try {
            count++;
        } finally {
            lock.unlock();            // ALWAYS release in finally, even if an exception occurs
        }
    }
}
```

| Method | Behavior |
|---|---|
| `lock()` | Acquires the lock; blocks (in `BLOCKED`-like waiting) indefinitely until available. |
| `tryLock()` | Tries to acquire immediately; returns `true`/`false` without waiting. |
| `tryLock(time, unit)` | Waits up to the given time; returns `true` if acquired in time, else `false`. |
| `unlock()` | Releases the lock — **must always be called in a `finally` block.** |
| `lockInterruptibly()` | Like `lock()`, but the **waiting** thread can be interrupted (throws `InterruptedException`) — `synchronized`/plain `lock()` cannot be interrupted while waiting. |

```java
// tryLock example — don't block forever, just skip if busy
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Couldn't get the lock, doing something else instead");
}

// tryLock with timeout
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try { /* critical section */ } finally { lock.unlock(); }
} else {
    System.out.println("Gave up waiting after 3 seconds");
}

// lockInterruptibly — always put the interrupt call in the catch block
try {
    lock.lockInterruptibly();
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Practice Questions — Section 6
1. Rewrite the `BankAccount` class from Section 5 using `ReentrantLock` instead of `synchronized`.
2. Write a program where 3 threads try `tryLock()` on the same resource; only the one that gets `true` proceeds, the others print "busy, skipping" and move on immediately.
3. Demonstrate `tryLock(time, unit)`: have thread A hold the lock for 5 seconds while thread B tries `tryLock(2, TimeUnit.SECONDS)` — show that B gives up and prints a timeout message.
4. What's the #1 rule about calling `unlock()`? Write a buggy example that forgets to call `unlock()` in a `finally` block and explain what could go wrong if an exception is thrown inside the critical section.
5. Demonstrate `lockInterruptibly()`: have a thread waiting on the lock get interrupted by another thread, and confirm it throws `InterruptedException` instead of waiting forever.
6. In your own words: why would you ever choose explicit `Lock` over plain `synchronized`? List 3 concrete capabilities you get.

---

## 7. ReentrantLock (Deep Dive)

**Why "reentrant"?** It allows the **same thread** that already holds the lock to acquire it again (e.g., recursive calls, or one synchronized method calling another) **without deadlocking itself**. Java maintains an internal **hold count** to track this.

```java
import java.util.concurrent.locks.ReentrantLock;

class LockExample {
    private final Lock lock = new ReentrantLock();

    public void outerMethod() {
        lock.lock();                 // hold count = 1
        try {
            System.out.println("In outerMethod");
            innerMethod();            // same thread re-acquires the SAME lock
        } finally {
            lock.unlock();            // hold count = 0 (fully released) only after this final unlock
        }
    }

    public void innerMethod() {
        lock.lock();                  // hold count = 2 (reentrant acquire, no deadlock)
        try {
            System.out.println("In innerMethod");
        } finally {
            lock.unlock();            // hold count = 1
        }
    }

    public static void main(String[] args) {
        LockExample obj = new LockExample();
        obj.outerMethod();
    }
}
```
**Trace of the hold count:**
1. `outerMethod()` calls `lock.lock()` → hold count = 1
2. It calls `innerMethod()`, which calls `lock.lock()` again → hold count = 2 (allowed because it's the *same thread*)
3. `innerMethod()`'s `finally` calls `unlock()` → hold count = 1 (NOT fully released yet)
4. `outerMethod()`'s `finally` calls `unlock()` → hold count = 0 (**fully released** — now another thread can acquire it)

> **`IllegalMonitorStateException`** is thrown if you call `unlock()` more times than you called `lock()` on that thread (i.e., trying to release a lock you don't hold).

### Practice Questions — Section 7
1. Write a recursive method (e.g., recursive factorial) guarded by a `ReentrantLock`, and prove via `getHoldCount()` that the hold count increases with each recursive call and decreases back to 0 at the end.
2. Try calling `lock.unlock()` one extra time (without a matching `lock()`), and confirm you get `IllegalMonitorStateException`. Explain in your own words why the JVM throws this.
3. Would a plain `synchronized` block have the same reentrant behavior for a method calling another synchronized method on the same object? Test it and explain why `synchronized` is *also* reentrant by default (this is a common trick question).
4. Write code using `ReentrantLock.isLocked()` and `getHoldCount()` to print lock state before/after each acquire and release in a nested-call scenario.
5. Explain a real scenario (e.g., a class with a public synchronized API method that internally calls another public synchronized API method) where non-reentrant locking would deadlock a single thread against itself.

---

## 8. Lock Fairness

**Fairness** = the order in which waiting threads are granted the lock. By default, `ReentrantLock` is **unfair** (a thread that just released the lock, or a new arrival, may "barge" ahead of threads that have been waiting longer) — this is usually *faster* in practice but can lead to **starvation** for unlucky threads.

```java
// Fair lock — guarantees FIFO order among waiting threads
private final Lock fairLock = new ReentrantLock(true);   // true = fairness enabled

// Unfair (default) lock — better throughput, no ordering guarantee
private final Lock unfairLock = new ReentrantLock();       // or explicitly new ReentrantLock(false)
```

- `synchronized` gives you **no fairness control at all** — it doesn't even distinguish between read and write access.
- With an unfair lock, it's *possible* (though usually rare) that a particular thread never gets scheduled the resource — **starvation**.

### Practice Questions — Section 8
1. Write a program with 5 threads competing for a fair `ReentrantLock(true)` vs an unfair `ReentrantLock()` — print the order in which each thread acquires the lock across many iterations, and compare fairness in practice.
2. Define "starvation" in your own words and explain how it differs from "deadlock" (preview of Section 10).
3. Why might a fair lock be *slower* overall even though it "sounds nicer"? (Hint: think about context-switch overhead of strict ordering.)
4. In what real-world system would you actually want a fair lock despite the throughput cost? Give an example (e.g., a rate-limited resource where request order = fairness to users).

---

## 9. ReadWriteLock / ReentrantReadWriteLock

Interface: `ReadWriteLock`. Standard implementation: `ReentrantReadWriteLock`.

It splits one lock into **two coordinated locks**:
- **Read lock** — shared / non-exclusive. **Multiple threads can hold the read lock simultaneously** (as long as no thread holds the write lock).
- **Write lock** — exclusive. Only one thread can hold it, and **no readers** can hold the read lock at the same time.

They're not independent — they come from the *same* `ReentrantReadWriteLock` instance and coordinate with each other: a writer never overlaps with readers or another writer, while readers never block each other.

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class SharedData {
    private int data = 0;
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void write(int value) {
        rwLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " writing " + value);
            data = value;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public int read() {
        rwLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " reading " + data);
            return data;
        } finally {
            rwLock.readLock().unlock();
        }
    }
}
```

### When to use `ReentrantLock` vs `ReentrantReadWriteLock`

| | `ReentrantLock` | `ReentrantReadWriteLock` |
|---|---|---|
| Best for | Balanced read/write, or write-heavy workloads | Read-heavy workloads |
| Downside | Unnecessary blocking on reads (readers block each other even though reading doesn't mutate state) | Slight overhead managing two locks; possible **writer starvation** if reads never stop coming in |

### Practice Questions — Section 9
1. Build a simple thread-safe cache (`Map<String,String>`) guarded by `ReentrantReadWriteLock`. Spawn 5 reader threads and 1 writer thread; prove multiple readers can read concurrently (log timestamps) while a writer gets exclusive access.
2. Modify the same cache to use a plain `ReentrantLock` instead. Compare — can two "readers" run concurrently now? Why not?
3. Simulate **writer starvation**: keep launching reader threads continuously and show a waiting writer thread never (or rarely) gets a chance. How would you mitigate this? (Hint: fairness parameter.)
4. What exception would you expect if a thread holding only the read lock tries to acquire the write lock directly (without releasing the read lock first)? Try it and explain (this is called *lock upgrading*, and plain `ReentrantReadWriteLock` does **not** support it directly).
5. Explain in one sentence why `ReentrantReadWriteLock` is *not* two independent locks even though it exposes `.readLock()` and `.writeLock()` separately.

---

## 10. Deadlock

**Definition:** Two or more threads are blocked forever, each waiting for a resource held by the other — a **circular dependency** on a set of locks.

### Classic deadlock example
```java
class Deadlock {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void method1() {
        synchronized (lockA) {
            System.out.println("Thread 1: Holding lock A...");
            try { Thread.sleep(50); } catch (InterruptedException e) {}
            System.out.println("Thread 1: Waiting for lock B...");
            synchronized (lockB) {
                System.out.println("Thread 1: Holding lock A and B");
            }
        }
    }

    public void method2() {
        synchronized (lockB) {
            System.out.println("Thread 2: Holding lock B...");
            try { Thread.sleep(50); } catch (InterruptedException e) {}
            System.out.println("Thread 2: Waiting for lock A...");
            synchronized (lockA) {
                System.out.println("Thread 2: Holding lock A and B");
            }
        }
    }
}

public class DeadlockDemo {
    public static void main(String[] args) {
        Deadlock d = new Deadlock();
        new Thread(d::method1).start();
        new Thread(d::method2).start();
        // Thread 1 holds A, waits for B.
        // Thread 2 holds B, waits for A.
        // Neither ever proceeds -> DEADLOCK, program hangs forever.
    }
}
```

### The fix: consistent lock ordering
```java
public void method2Fixed() {
    synchronized (lockA) {   // acquire in the SAME order as method1 (A then B)
        synchronized (lockB) {
            System.out.println("Thread 2: safely acquired both locks");
        }
    }
}
```
If **every thread always acquires locks in the same global order**, circular waiting becomes impossible.

How `Thread` accepts a method reference like `obj::method`

`Thread`'s constructor takes a `Runnable`:
```java
public Thread(Runnable target)
```
`Runnable` is a **functional interface** (exactly one abstract method, `void run()`), so Java lets you satisfy it with a **method reference** or **lambda** instead of writing a full class that implements it.

```java
new Thread(deadLock1::method1).start();
```
is exactly equivalent to:
```java
new Thread(() -> deadLock1.method1()).start();
```
which is exactly equivalent to:
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        deadLock1.method1();   // just calls the existing method on the existing object
    }
}).start();
```

**Important:** `DeadLock1` (or whatever class `deadLock1` is) is **not** implementing `Runnable`. The compiler generates a tiny anonymous `Runnable` behind the scenes whose `run()` body simply calls `deadLock1.method1()`. This works purely because of **target typing / signature matching**:
1. `Thread`'s constructor expects a `Runnable`.
2. `Runnable` needs a method matching `void someName()` — no args, no return value.
3. `method1()` has the signature `public void method1()` — a perfect match — so the compiler wraps the reference as a `Runnable` automatically.

#### Proof: any zero-arg void method works, from any class
This isn't special to thread-related classes — it works for literally any class with a matching method, because there's no inheritance relationship involved at all, just a signature match:
```java
class Printer {
    public void printHello() {
        System.out.println("Hello!");
    }
}

Printer p = new Printer();
new Thread(p::printHello).start();  // works fine — Printer never mentions Runnable
```

> **Takeaway:** `obj::method` (or a lambda) is just a shorthand way of creating a `Runnable` implementation whose `run()` calls that method. The object (`obj`) and its class are completely unrelated to threading — only the method reference itself needs to match `Runnable`'s `void run()` shape.
### Other deadlock-avoidance techniques
- Use `tryLock(timeout)` instead of blocking `lock()`/`synchronized` — back off and retry if you can't get all needed locks in time.
- Minimize the number of locks held at once / keep critical sections small.
- Use higher-level concurrency utilities (e.g., `java.util.concurrent` collections, `CompletableFuture`) instead of hand-rolled locking wherever possible.

### Practice Questions — Section 10
1. Reproduce the deadlock example above exactly — run it and confirm it hangs (you'll need to kill the process).
2. Fix it using consistent lock ordering, and prove it no longer deadlocks by running it 20 times in a loop.
3. Fix the *original* (unordered) version instead using `tryLock(timeout)` on both locks — if a thread can't get both locks within the timeout, it should release what it has and retry.
4. Name the 4 classical necessary conditions for deadlock (mutual exclusion, hold-and-wait, no preemption, circular wait) and explain how each fix above breaks one of them.
5. Write a 3-lock, 3-thread circular deadlock (A→B, B→C, C→A) and then fix it.
6. How would you *detect* a deadlock in a running production JVM without restarting it? (Hint: `jstack`, thread dump analysis.)

---

## 11. Inter-thread Communication (`wait` / `notify` / `notifyAll`)

Without proper communication between threads, you get **wasted CPU cycles** (busy-waiting / polling in a loop) or **deadlocks**. Java provides `wait()`, `notify()`, and `notifyAll()` — defined on `Object` (not `Thread`!) — to let a thread sleep until another thread signals it.

| Method | Behavior |
|---|---|
| `wait()` | Tells the **current thread** to release the lock on this object and go into the `WAITING` state until another thread calls `notify()`/`notifyAll()` on the same object. |
| `notify()` | Wakes up **a single** waiting thread (JVM picks which one — no guarantee which). |
| `notifyAll()` | Wakes up **all** threads waiting on this object's monitor; they then compete to re-acquire the lock. |

> **Critical rule:** `wait()`, `notify()`, `notifyAll()` can **only be called from within a `synchronized` block/method** on the object you're calling them on — otherwise you get `IllegalMonitorStateException`. This is because they operate on the object's monitor, which you must already own.

### Producer-Consumer using wait/notify
```java
import java.util.LinkedList;
import java.util.Queue;

class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 5;

    public synchronized void produce(int value) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // buffer full -> release lock, sleep until consumer signals
        }
        queue.add(value);
        System.out.println("Produced: " + value);
        notifyAll();  // wake up any waiting consumer
    }

    public synchronized void consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // buffer empty -> release lock, sleep until producer signals
        }
        int value = queue.poll();
        System.out.println("Consumed: " + value);
        notifyAll();  // wake up any waiting producer
    }
}

public class ProducerConsumerDemo {
    public static void main(String[] args) {
        SharedBuffer buffer = new SharedBuffer();

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) buffer.produce(i);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) buffer.consume();
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        producer.start();
        consumer.start();
    }
}
```
> **Why a `while` loop and not `if`?** Because of **spurious wakeups** (a thread can wake from `wait()` without an actual `notify()`, per the JVM spec) and because with `notifyAll()`, multiple threads race to recheck the condition — only a `while` loop re-validates the condition every time the thread wakes up.

wait/notify/notifyAll operate on an object's monitor (the intrinsic lock), and any object — not just a Thread — can be synchronized on and have multiple threads waiting on it. Since the coordination is about the shared object's lock state, not the calling thread's own state, these methods belong on Object, and that's also why they can only be called while holding that object's monitor (inside a synchronized block on it).

### Practice Questions — Section 11
1. Implement the producer-consumer example above from scratch without looking, then modify it to use a bounded buffer of size 3.
2. Replace `notifyAll()` with `notify()` in the producer-consumer code with 2 producers and 2 consumers — can you construct a scenario (even rare) where using `notify()` instead of `notifyAll()` causes a thread to wait forever? Explain why `notifyAll()` is usually safer despite being less efficient.
3. Remove the `while` loop and replace it with `if` — try to trigger a bug (it may need many runs) and explain the failure mode you'd theoretically expect (spurious wakeup / stale condition check).
4. Call `wait()` **outside** a `synchronized` block and confirm you get `IllegalMonitorStateException`.
5. Implement a simple **"ping-pong"** program: two threads alternately print "ping" and "pong" exactly 10 times each, using `wait()`/`notify()` to force strict alternation.
6. Explain (without code) how you'd implement the same producer-consumer pattern using a `BlockingQueue` instead of manual `wait`/`notify` — and why that's usually the better real-world choice.

---

## 12. Thread Pools & the Executor Framework

**Problem with manual thread creation:** creating a new OS thread per task is expensive, and you lose control over how many threads run concurrently — too many threads can crash the app or thrash the CPU with context switching.

**Thread Pool** = a collection of pre-initialized, reusable threads ready to execute tasks, giving you:
1. **Resource management** — cap the number of live threads.
2. **Better response time** — no cost of creating a thread per request; a pooled thread is often already alive.
3. **Control over thread count** — tune the pool size to your hardware/workload.

Java's **Executor Framework** (`java.util.concurrent`) manages thread pools for you so you rarely need to create raw `Thread` objects yourself in real code.

```
Executor  (root interface — just execute(Runnable))
   └── ExecutorService   (adds lifecycle mgmt, submit(), shutdown(), Future<T>)
          └── ScheduledExecutorService  (adds delayed / periodic execution)
```

### 12.1 Creating and using an ExecutorService
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorDemo {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(5); // pool of 5 threads

        for (int i = 1; i <= 10; i++) {
            int taskId = i; // effectively final for the lambda
            executor.submit(() -> {
                System.out.println("Task " + taskId + " running on " + Thread.currentThread().getName());
            });
        }

        executor.shutdown();  // stop accepting new tasks; lets already-submitted tasks finish
        // NOTE: main thread does NOT wait for the pool to finish just by calling shutdown()!
    }
}
```
> **Common gotcha:** calling `shutdown()` does **not** block the calling thread. If you need to wait for all submitted tasks to actually complete before moving on, use:
```java
executor.shutdown();
boolean finishedInTime = executor.awaitTermination(5, TimeUnit.SECONDS);
```

### 12.2 Common Executors factory methods
| Factory method | Pool behavior |
|---|---|
| `Executors.newFixedThreadPool(n)` | Fixed number of `n` threads; extra tasks queue up. |
| `Executors.newSingleThreadExecutor()` | Exactly 1 thread — tasks run sequentially, in order. |
| `Executors.newCachedThreadPool()` | Pool size grows/shrinks automatically as needed; **danger:** no cap, so an unbounded number of threads can be created under heavy load — you lose control of "how many threads are getting generated." |
| `Executors.newScheduledThreadPool(n)` | Like fixed pool, but supports delayed/periodic tasks (see Section 14). |

### 12.3 Checking completion / shutdown status
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Long> future = executor.submit(() -> factorial(20));

System.out.println(future.isDone());     // false, probably — task may still be running
long result = future.get();              // BLOCKS until result is ready
System.out.println(future.isDone());     // true now

executor.shutdown();
System.out.println(executor.isTerminated());  // false immediately after shutdown() (tasks may still be running)
executor.awaitTermination(5, TimeUnit.SECONDS);
System.out.println(executor.isTerminated());  // true once all tasks are actually done (or timeout occurred)
```

### Practice Questions — Section 12
1. Create a `newFixedThreadPool(3)` and submit 10 tasks, each printing its task number and thread name, then sleeping 500ms. Observe how tasks queue up across only 3 threads.
2. Explain, with an experiment, why `executor.shutdown()` alone does not guarantee all tasks are finished by the time the next line of code runs. Fix it using `awaitTermination`.
3. Compare `newFixedThreadPool(5)` vs `newCachedThreadPool()` by submitting 1000 very short tasks — log how many actual threads get created in each case (`Thread.currentThread().getName()` distinct count).
4. What happens if you `submit()` a task to an executor **after** calling `shutdown()`? Try it and name the exception.
5. Implement a `newSingleThreadExecutor()`-based task queue and prove tasks always execute in submission order (unlike a multi-thread pool).
6. What's the difference between `shutdown()` and `shutdownNow()`? Try `shutdownNow()` on a pool with long-running tasks and observe/explain the returned list and thread interruption behavior.

---

## 13. Callable, Future, `invokeAll`/`invokeAny`

`Runnable.run()` returns nothing (`void`) and can't throw checked exceptions. `Callable<V>` is the "returns a value" version:

```java
import java.util.concurrent.Callable;

Callable<Integer> task = () -> {
    // can return a value AND throw checked exceptions
    return 10 * 10;
};
```

### `Future<T>` — a handle to a result that will exist later
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(() -> 42);   // submit(Callable) returns a Future

System.out.println(future.get());          // blocks until the result is ready, then returns it
System.out.println(future.isDone());       // true/false — was the task completed?
System.out.println(future.isCancelled());  // true/false — was it cancelled before completing?
future.cancel(true);                       // attempt to cancel (interrupt if running)

executor.shutdown();
```

| Future method | Meaning |
|---|---|
| `get()` | Blocks the calling thread until the task finishes, then returns its result (or throws if it threw). |
| `get(timeout, unit)` | Same, but throws `TimeoutException` if it takes too long. |
| `isDone()` | Non-blocking check — has the task completed (normally, exceptionally, or via cancellation)? |
| `isCancelled()` | Was the task cancelled before it completed normally? |
| `cancel(boolean mayInterrupt)` | Attempts to cancel execution. |

### `invokeAll` vs `invokeAny`
```java
List<Callable<Integer>> tasks = List.of(
    () -> { Thread.sleep(100); return 1; },
    () -> { Thread.sleep(50);  return 2; },
    () -> { Thread.sleep(200); return 3; }
);

ExecutorService executor = Executors.newFixedThreadPool(3);

// invokeAll: main thread BLOCKS until ALL tasks complete, returns a List<Future<Integer>>
List<Future<Integer>> results = executor.invokeAll(tasks);
for (Future<Integer> f : results) System.out.println(f.get());

// invokeAny: runs all tasks, returns the result of whichever ONE finishes first
// successfully (cancels the rest) -- returns a plain Integer, not a Future
Integer firstResult = executor.invokeAny(tasks);
System.out.println("First completed: " + firstResult);

executor.shutdown();
```

### Practice Questions — Section 13
1. Write a `Callable<Long>` that computes the factorial of a given number, submit it to an executor, and print the result via `future.get()`.
2. Demonstrate `future.get(timeout, unit)` throwing `TimeoutException` for a task that intentionally sleeps longer than the timeout.
3. Submit a task, then call `future.cancel(true)` immediately — check `isCancelled()` and see what happens if you then call `future.get()` (name the exception).
4. Write 5 Callables simulating different "server calls" with random sleep durations; use `invokeAny` to get the fastest one's result, and print which one "won."
5. Use `invokeAll` on the same 5 Callables and print all 5 results in original submission order — explain why `invokeAll` waits for the *slowest* task while `invokeAny` doesn't.
6. What checked exception can a `Callable` throw that a `Runnable` cannot? Show what you must do differently when calling `Runnable`'s logic that might throw vs a `Callable`'s (try/catch vs propagate via `Future.get()` → `ExecutionException`).

---

## 14. ScheduledExecutorService

For tasks that need to run **once after a delay**, or **repeatedly**.

```java
import java.util.concurrent.*;

ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// 1. schedule() — run ONE-SHOT task after a given delay
scheduler.schedule(() -> System.out.println("Ran once after delay"), 3, TimeUnit.SECONDS);

// 2. scheduleAtFixedRate() — runs periodically; each execution START is scheduled at
//    fixed intervals REGARDLESS of how long the previous execution took
//    (successive executions' start times are independent of prior completion)
scheduler.scheduleAtFixedRate(() ->
    System.out.println("Fixed rate tick at " + System.currentTimeMillis()),
    0, 2, TimeUnit.SECONDS);   // initialDelay=0, period=2s

// 3. scheduleWithFixedDelay() — the delay is measured from the END of one execution
//    to the START of the next; guarantees a gap between runs even if a task overruns
scheduler.scheduleWithFixedDelay(() ->
    System.out.println("Fixed delay tick at " + System.currentTimeMillis()),
    0, 2, TimeUnit.SECONDS);
```

| Method | Timing semantics |
|---|---|
| `schedule(task, delay, unit)` | One-shot; runs once after the delay passes. |
| `scheduleAtFixedRate(task, initialDelay, period, unit)` | Runs periodically; each execution's start is independent of when the previous execution finished (can "pile up" if tasks run longer than the period). |
| `scheduleWithFixedDelay(task, initialDelay, delay, unit)` | Runs periodically; waits for the specified delay **after** the previous run finishes before starting the next. |

### Practice Questions — Section 14
1. Write a one-shot `schedule()` task that prints "Reminder!" after 5 seconds, and confirm `main` doesn't block waiting for it.
2. Compare `scheduleAtFixedRate` vs `scheduleWithFixedDelay` with a task that deliberately sleeps *longer* than the period (e.g., task takes 3s but period is 1s) — log timestamps for both and explain the different behavior you observe.
3. Build a simple "heartbeat" logger that prints a timestamp every 2 seconds forever using `scheduleAtFixedRate`, and show how to cleanly stop it (cancel the returned `ScheduledFuture`).
4. What does `scheduler.schedule(...)` return, and how would you use that return value to cancel a pending one-shot task before it runs?
5. Explain a real-world use case better suited to `scheduleAtFixedRate` (e.g., polling a metrics endpoint) vs one better suited to `scheduleWithFixedDelay` (e.g., retrying a slow, variable-length cleanup job).

---

## 15. CountDownLatch

**Purpose:** let one or more threads **wait until a set of operations performed by other threads finishes**.

- Starts with an integer count.
- Waiting thread(s) call `await()` — they block.
- Worker threads call `countDown()` after finishing their piece of work, decrementing the count.
- Once count reaches **zero**, all waiting threads are released.
- **The count can never be reset or reused** once it hits zero (it's a one-shot gate).

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int numWorkers = 3;
        CountDownLatch latch = new CountDownLatch(numWorkers);

        for (int i = 1; i <= numWorkers; i++) {
            int workerId = i;
            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " starting work...");
                    Thread.sleep(1000L * workerId);
                    System.out.println("Worker " + workerId + " done!");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();   // signal this worker's piece is done
                }
            }).start();
        }

        System.out.println("Main thread waiting for all workers...");
        latch.await();   // blocks until count reaches 0
        System.out.println("All workers finished! Main thread proceeding.");
    }
}
```

### Practice Questions — Section 15
1. Implement the example above from scratch, then modify it so `main` uses `latch.await(timeout, unit)` to give up waiting after 2 seconds even if workers aren't done — print a "timed out" message in that case.
2. Simulate an application startup sequence: 3 services (DB connection, cache warm-up, config load) must all finish before the "server" thread prints "Ready to accept requests." Use `CountDownLatch(3)`.
3. What happens if you call `countDown()` more times than the initial count? (Try it — does it throw, or just floor at zero?)
4. Can a `CountDownLatch` be reused for a second round of work after reaching zero? Try calling `await()` again after the count hit 0 and explain the (lack of) blocking behavior.
5. Explain the difference between `CountDownLatch` and `join()` — could you replace this example with 3 `join()` calls instead? What's the key structural difference (fixed known threads vs a dynamic "arrival count")?

---

## 16. CyclicBarrier

**Purpose:** similar to `CountDownLatch`, but it's **reusable** ("cyclic") and it's about threads waiting for **each other** (peer-to-peer) rather than a controller waiting for workers.

- All participating threads call `await()` on the barrier.
- Once the last thread arrives (count reaches the barrier limit), **all threads are released together** and can proceed.
- Optionally, a "barrier action" (`Runnable`) runs once, right when the last thread arrives, before releasing everyone.
- Unlike `CountDownLatch`, the barrier automatically **resets** and can be reused for the next round.

**Analogy from the notes:** You're waiting at the movie entrance for your friends. Only once the *last* friend arrives does the whole group walk in together.
- friends = workers/threads/parties
- movie entrance = the barrier

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        int numThreads = 3;
        CyclicBarrier barrier = new CyclicBarrier(numThreads,
            () -> System.out.println("All threads reached the barrier! Proceeding together...")
        );

        for (int i = 1; i <= numThreads; i++) {
            int id = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + id + " doing work...");
                    Thread.sleep(500L * id);
                    System.out.println("Thread " + id + " waiting at barrier");
                    barrier.await();  // blocks until all 3 threads call await()
                    System.out.println("Thread " + id + " passed the barrier!");
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

**Use case highlighted in the notes:** matrix multiplication — split work into chunks across threads, each thread computes its chunk, then all must synchronize at a barrier before moving to the next phase (e.g., combining partial results).

### `CountDownLatch` vs `CyclicBarrier` — quick comparison

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| Reusable? | No — one-shot | Yes — automatically resets |
| Who waits? | One or more "controller" threads wait for workers | All participating threads wait for **each other** |
| Triggered by | `countDown()` (can be called by non-participating threads too) | `await()` itself (only participants call it) |
| Optional action on completion | No built-in hook | Yes — barrier action `Runnable` runs once when last thread arrives |

### Practice Questions — Section 16
1. Implement the example above, then run it for **2 rounds** (call `barrier.await()` again in a loop inside each thread) to prove the barrier resets and can be reused.
2. Implement a simplified parallel matrix multiplication: split rows across 4 worker threads, each computes its rows, then all synchronize at a `CyclicBarrier` before a 5th step prints "Matrix multiplication complete."
3. What is `BrokenBarrierException` and when is it thrown? (Hint: what happens if one of the participating threads is interrupted while waiting at the barrier — what happens to the *other* waiting threads?)
4. Rewrite the movie-entrance analogy as working code: 4 "friend" threads arrive at random times, and only after the last one arrives does a barrier action print "Let's go watch the movie!"
5. Could you replace a `CyclicBarrier` with a `CountDownLatch` if you only need "wait once, never reuse"? What would you lose in that translation (e.g., the mutual "waiting for each other" symmetry, or the automatic barrier-action hook)?

---

## 17. CompletableFuture

Introduced to handle **asynchronous computation** more expressively than plain `Future` (which only supports blocking `.get()` — no chaining, no combining, no callbacks).

```java
import java.util.concurrent.CompletableFuture;

// Run a task asynchronously (by default, on the common ForkJoinPool)
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    // some potentially slow computation
    return 10 * 10;
});

// Block and get the result (like Future.get())
System.out.println(future.get());       // 100 — checked exceptions to handle

// Chain a transformation on the result once it's ready, WITHOUT blocking to attach the chain
CompletableFuture<String> chained = future.thenApply(result -> "Result was: " + result);
System.out.println(chained.get());
```

| Method | Purpose |
|---|---|
| `CompletableFuture.supplyAsync(Supplier<T>)` | Runs a task asynchronously and returns a value. |
| `CompletableFuture.runAsync(Runnable)` | Runs a task asynchronously with **no** return value. |
| `.get()` | Blocks and retrieves the result (like `Future`). |
| `.thenApply(Function)` | Applies a transformation to the result once available — returns a **new** `CompletableFuture`. |
| `.thenAccept(Consumer)` | Consumes the result (side effect), returns `CompletableFuture<Void>`. |
| `.thenCompose(Function -> CompletableFuture)` | Chains another **async** operation that itself returns a `CompletableFuture` (flattening, avoiding nested futures). |
| `.thenCombine(other, BiFunction)` | Combines results of two independent `CompletableFuture`s. |
| `.exceptionally(Function<Throwable,T>)` | Provides a fallback value if the async computation threw. |
| `.isDone()` / `.cancel()` | Same idea as plain `Future`. |

```java
CompletableFuture<Integer> cf = CompletableFuture
    .supplyAsync(() -> 10)
    .thenApply(x -> x * 2)          // 20
    .thenApply(x -> x + 5);         // 25

System.out.println(cf.get());       // 25

// Handling errors
CompletableFuture<Integer> safe = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Boom!");
    return 1;
}).exceptionally(ex -> {
    System.out.println("Recovered from: " + ex.getMessage());
    return -1;
});
System.out.println(safe.get()); // -1
```

**Threading detail worth remembering:** by default, `CompletableFuture` tasks run on daemon threads from `ForkJoinPool.commonPool()`. You can control which threads are used by supplying your own `Executor`:
```java
ExecutorService myPool = Executors.newFixedThreadPool(4);
CompletableFuture.supplyAsync(() -> compute(), myPool);
```
> The `CompletableFuture` task itself doesn't dictate daemon-vs-user thread — that's a property of whichever executor actually runs it.

### Practice Questions — Section 17
1. Chain 3 `.thenApply()` calls that each transform a number (e.g., double it, add 10, then convert to String), and print the final result.
2. Use `.exceptionally()` to recover from a deliberately-thrown exception inside `supplyAsync`, providing a fallback value.
3. Use `.thenCombine()` to combine the results of two independent `CompletableFuture`s (e.g., one fetches a "user," the other fetches their "orders") into a single combined summary string.
4. Compare `.thenApply()` vs `.thenCompose()`: write a case where using `.thenApply()` with a function that itself returns a `CompletableFuture<T>` results in a nested `CompletableFuture<CompletableFuture<T>>`, then fix it using `.thenCompose()`.
5. Supply a custom `ExecutorService` to `supplyAsync()` and print the thread name inside the task to prove it's running on your pool, not the common `ForkJoinPool`.
6. Use `CompletableFuture.allOf(...)` to wait for 3 independent async tasks to all complete before printing "All done" (look this method up — it's the natural next step after `thenCombine` for more than 2 futures).

---

## 18. Volatile & Atomic Classes

### The problem: thread-local caching of variables
Each thread's CPU core can keep a **local (cached) copy** of a shared variable for performance. This means one thread's write might not be immediately visible to another thread reading the "same" variable — a **visibility** problem (separate from the atomicity problem `synchronized` solves).

### `volatile` — fixes visibility
```java
class SharedFlag {
    private volatile boolean running = true;   // forces reads/writes to go to MAIN MEMORY

    public void stop() {
        running = false;   // this write is immediately visible to all other threads
    }

    public void run() {
        while (running) {
            // do work — without volatile, this loop might NEVER see `running` become false
            // because the thread could keep reading its own cached copy forever
        }
        System.out.println("Stopped!");
    }
}
```
> **What `volatile` guarantees:** every read of a volatile variable sees the most recent write from *any* thread (visibility), and it prevents certain instruction reordering (a happens-before relationship). **What it does NOT guarantee:** atomicity of compound operations like `count++` (that's still read-modify-write in 3 steps, even if `count` is volatile!).

```java
// THIS IS STILL BROKEN even with volatile, because ++ is not atomic:
private volatile int count = 0;
public void increment() { count++; }  // race condition still possible!
```

### `Atomic` classes — fixes atomicity (without locks!)
For simple compound operations (increment, compare-and-set, etc.), `java.util.concurrent.atomic` gives you lock-free, thread-safe classes built on low-level CPU **compare-and-swap (CAS)** instructions — much faster than `synchronized`/`Lock` for simple counters under contention.

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicBoolean;

class Counter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();   // atomic, thread-safe, no synchronized needed
    }

    public int getCount() {
        return count.get();
    }
}

public class AtomicDemo {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Runnable task = () -> { for (int i = 0; i < 100000; i++) counter.increment(); };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join();  t2.join();

        System.out.println(counter.getCount()); // reliably 200000
    }
}
```

Common `Atomic*` classes: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<T>`.

Useful methods: `get()`, `set(v)`, `incrementAndGet()`, `getAndIncrement()`, `addAndGet(n)`, `compareAndSet(expected, newValue)`.

### `volatile` vs `synchronized` vs `Atomic*` — when to use which

| | Guarantees | Best for |
|---|---|---|
| `volatile` | Visibility only, no atomicity | A simple flag read/written by multiple threads (e.g., a "stop" signal) |
| `synchronized` / `Lock` | Both visibility AND atomicity, for arbitrarily complex critical sections | Multi-step operations, invariants across multiple fields |
| `Atomic*` | Both, but only for single-variable, simple compound operations (CAS-based, lock-free) | High-contention simple counters/flags where you want max throughput |

### Practice Questions — Section 18
1. Write a `while(running)` loop in one thread and flip `running = false` from another thread — first *without* `volatile` (it may or may not terminate, JVM/JIT dependent) then *with* `volatile` (should reliably terminate). Explain what you observe.
2. Prove that a `volatile int` is **still** subject to race conditions for `count++` by running the concurrent-increment demo with a `volatile int` (no Atomic, no synchronized) and confirming the final count is less than expected.
3. Rewrite the same broken counter using `AtomicInteger` and confirm you reliably get the correct total.
4. Use `AtomicInteger.compareAndSet(expected, newValue)` to implement a simple optimistic "update only if unchanged" pattern, and explain what CAS means in your own words.
5. When would you prefer `synchronized` over `AtomicInteger` even for something that looks like "just a counter"? (Hint: think about needing to update **two related variables together** atomically.)
6. Use `AtomicReference<T>` to implement a simple lock-free stack push operation using `compareAndSet` in a loop (a classic CAS-retry pattern).

---

## 19. Classic Concurrency Problems (Practice These End-to-End)

These are extremely common **whiteboard/live-coding** interview questions. Try to solve each one **without notes first**, using whichever tool from this document fits best, then check your solution against the hints.

1. **Producer-Consumer** — implement with (a) `wait`/`notify`, and again with (b) `BlockingQueue` (`ArrayBlockingQueue`/`LinkedBlockingQueue`). Compare code complexity.
2. **Print numbers in order using multiple threads** — 3 threads print "1 Fizz", "2 Buzz", etc. in strict round-robin order (classic "FizzBuzz multithreaded" or "print odd/even alternately using 2 threads").
3. **Dining Philosophers** — N philosophers, N forks, avoid deadlock (hint: consistent lock ordering, or a resource-limiting semaphore allowing only N-1 philosophers to try eating at once).
4. **Bounded Blocking Queue from scratch** — implement your own thread-safe bounded queue using only `wait`/`notify` (no `java.util.concurrent` classes) supporting `put(item)` and `take()`.
5. **Thread-safe Singleton** — implement double-checked locking with `volatile` for a lazy singleton, and explain why *both* the `volatile` keyword and the double-check (`if (instance == null)` twice) are necessary.
6. **Rate Limiter** — implement a simple thread-safe token-bucket rate limiter usable by multiple concurrent threads.
7. **Even/Odd Printer** — two threads: one prints only even numbers, the other only odd numbers, alternating 1 to 20, using either `wait/notify` or a `Semaphore` pair.
8. **Parallel sum of a large array** — split an array into chunks, use an `ExecutorService`/`ForkJoinPool` to sum chunks in parallel, then combine (classic fork-join / divide-and-conquer exercise). Compare timing vs a single-threaded sum for a very large array.
9. **Timeout-safe cache** — build a cache where a background thread periodically expires stale entries while reader/writer threads use it concurrently (`ReadWriteLock` + `ScheduledExecutorService`).
10. **Semaphore-based connection pool** — implement a simple fixed-size resource pool (e.g., simulated DB connections) using `java.util.concurrent.Semaphore` to cap concurrent access.

> **Bonus concept not in your original notes but commonly asked alongside these:** `Semaphore` — a counter-based permit system (`acquire()`/`release()`) that limits how many threads can access a resource concurrently. Worth a quick read-up since #7 and #10 above use it.

---

## 20. Rapid-Fire Interview Q&A Bank

Try answering each in **1–3 sentences out loud**, then check yourself against the sections above.

1. What's the difference between a process and a thread?
2. What's the difference between `start()` and `run()`?
3. Why is `Runnable` generally preferred over extending `Thread`?
4. What are the 6 thread states in Java, and what triggers each transition?
5. What does `Thread.sleep()` do to a lock the thread is holding? (Nothing — it does NOT release it, unlike `wait()`.)
6. What does `wait()` do to a lock the thread is holding? (Releases it.)
7. What is a race condition? Give an example.
8. What does `synchronized` actually guarantee?
9. What's the difference between synchronizing on `this` vs a private lock object?
10. What is a `ReentrantLock`, and why is it called "reentrant"?
11. What's the difference between `lock()`, `tryLock()`, and `lockInterruptibly()`?
12. What's the #1 rule for calling `unlock()`?
13. What is lock fairness, and what's the tradeoff of using a fair lock?
14. When would you use `ReentrantReadWriteLock` over a plain `ReentrantLock`?
15. What is a deadlock, and name two ways to prevent one.
16. What are the 4 necessary conditions for deadlock?
17. Explain `wait()` vs `notify()` vs `notifyAll()`. Why must they be called inside a `synchronized` block?
18. Why do we check the wait condition in a `while` loop, not an `if`?
19. What is a thread pool, and why use one instead of raw threads?
20. What's the difference between `Runnable` and `Callable`?
21. What does `Future.get()` do, and what's the risk of calling it with no timeout?
22. What's the difference between `invokeAll` and `invokeAny`?
23. What's the difference between `scheduleAtFixedRate` and `scheduleWithFixedDelay`?
24. What's the difference between `CountDownLatch` and `CyclicBarrier`?
25. Why is `CountDownLatch` a "one-shot" tool while `CyclicBarrier` is reusable?
26. What problem does `CompletableFuture` solve that plain `Future` doesn't?
27. What's the difference between `.thenApply()` and `.thenCompose()`?
28. What does `volatile` guarantee, and what does it NOT guarantee?
29. Why is `count++` still unsafe even if `count` is `volatile`?
30. How do `Atomic*` classes achieve thread safety without locks? (CAS.)
31. When would `synchronized`/`Lock` still be necessary even though `Atomic*` classes exist?
32. What is a daemon thread, and how does it differ from a user thread in terms of JVM shutdown?
33. What is thread starvation, and how does it differ from deadlock?
34. Why can't two threads execute two *different* synchronized methods of the same object simultaneously (if both synchronize on the same monitor)?
35. What is `IllegalMonitorStateException` and what 2 different situations can cause it (unlock without lock; wait/notify without holding the monitor)?

---

## 21. Final Coding Challenge Set (Mixed, Mock-Interview Style)

Attempt these under time pressure (20–30 min each) as if in a live interview — explain your approach out loud before coding.

1. **Thread-safe Bank Transfer:** Implement `transfer(Account from, Account to, double amount)` that is safe to call concurrently from many threads transferring between arbitrary account pairs, **without deadlocking** (classic "lock two objects in a consistent order" problem — order locks by, e.g., account ID).
2. **Multi-threaded Word Count:** Given a huge list of files, use an `ExecutorService` to count word frequencies in parallel and merge results into one thread-safe `Map`.
3. **Build a simple `ExecutorService`-backed download manager:** submit N "download" tasks (simulated with `Thread.sleep`), track progress with a shared `AtomicInteger` completed-counter, and print a live "X/N complete" as tasks finish.
4. **Implement a Debounced Event Handler:** using `ScheduledExecutorService`, coalesce rapid-fire "events" so only the last one within a 300ms window actually triggers an action (cancel/reschedule pending tasks).
5. **Build your own simple `CountDownLatch`** from scratch using only `synchronized`/`wait`/`notifyAll` (don't use `java.util.concurrent.CountDownLatch`) — a great way to prove you understand what's happening under the hood.
6. **Build your own simple bounded `BlockingQueue`** from scratch (put/take with wait/notify) — another "implement it yourself" classic.
7. **Detect and fix a deadlock:** given a buggy 2-thread, 2-lock snippet (write one deliberately, like Section 10), take a thread dump (`jstack` or `Thread.getAllStackTraces()`) and identify the deadlock programmatically or from the dump output.
8. **Async pipeline with CompletableFuture:** fetch a "user ID" async → then fetch "user profile" async (`thenCompose`) → then fetch "recommendations" in parallel with "recent orders" (`thenCombine`) → assemble a final response object, with `.exceptionally()` fallback at each stage.

---

### Closing self-check
Before your interview, you should be able to:
- [ ] Draw the thread lifecycle diagram from memory
- [ ] Write a producer-consumer solution from scratch, both with `wait/notify` and `BlockingQueue`
- [ ] Explain deadlock's 4 conditions and fix a live deadlock via lock ordering
- [ ] Explain why `volatile` ≠ atomic, and when you need `Atomic*` vs `synchronized`
- [ ] Compare `CountDownLatch` vs `CyclicBarrier` vs `Semaphore` fluently
- [ ] Justify `ExecutorService` over raw `Thread` creation, and know all 4 common factory methods and their tradeoffs
- [ ] Chain and combine `CompletableFuture`s without falling into nested-future traps