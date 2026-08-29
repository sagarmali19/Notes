# Java Map / Collection Framework — Complete Interview Notes
### For 3+ years experience Java Backend Developer

---

## Table of Contents
1. Map Interface & Collection Framework Overview
2. HashMap — Internal Structure & Data Storage (Deep Dive)
3. How HashMap Retrieves Data (get() internals)
4. Hash Collisions & Resolution
5. HashMap Resizing (Rehashing)
6. Why override hashCode() and equals() for custom objects
7. LinkedHashMap + LRU Cache Implementation
8. WeakHashMap
9. IdentityHashMap
10. SortedMap
11. NavigableMap
12. TreeMap (Red-Black Tree internals)
13. Hashtable
14. ConcurrentHashMap (Java 7 vs Java 8 internals)
15. Immutable Map
16. EnumMap
17. ConcurrentSkipListMap
18. Bonus Topics (fail-fast vs fail-safe, ConcurrentModificationException, Map.Entry, Java 8 Map methods, comparison tables, common interview Q&A, coding problems)

---

## 1. Map Interface & Collection Framework Overview

`Map<K, V>` is **not** part of the `Collection` interface hierarchy (no `add()`, it stores key-value pairs), but it **is** part of the Java Collections Framework.

```
Map<K,V>  (interface)
 ├── AbstractMap<K,V>
 │     ├── HashMap<K,V>
 │     │     └── LinkedHashMap<K,V>
 │     ├── IdentityHashMap<K,V>
 │     ├── WeakHashMap<K,V>
 │     ├── EnumMap<K,V>
 │     └── TreeMap<K,V> (implements NavigableMap → SortedMap)
 ├── SortedMap<K,V> (interface, extended by NavigableMap)
 │     └── NavigableMap<K,V> (interface)
 │           ├── TreeMap<K,V>
 │           └── ConcurrentSkipListMap<K,V>
 ├── ConcurrentMap<K,V> (interface)
 │     ├── ConcurrentHashMap<K,V>
 │     └── ConcurrentSkipListMap<K,V>
 └── Hashtable<K,V> (legacy, implements Map directly, predates Collections Framework)
       └── Properties
```

**Key Points:**
- `Map` breaks the `Collection` contract intentionally — you can get `keySet()`, `values()`, `entrySet()` as views (backed by the map, changes reflect both ways).
- All Map implementations allow you to iterate via `entrySet()` — **preferred over `keySet()` + `get()`** because entrySet() avoids a second lookup per key.

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

// BEST practice for iteration (single lookup)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// AVOID (extra get() call per key = O(1) extra but wasteful)
for (String key : map.keySet()) {
    System.out.println(key + " = " + map.get(key));
}
```

### Core Map Implementations at a glance

| Implementation | Ordering | Null keys | Null values | Thread-safe | Backing structure |
|---|---|---|---|---|---|
| HashMap | No order | 1 allowed | Multiple allowed | No | Array + LinkedList/RB-Tree (buckets) |
| LinkedHashMap | Insertion or access order | 1 allowed | Multiple allowed | No | HashMap + Doubly Linked List |
| TreeMap | Sorted (natural/comparator) | Not allowed (NPE) | Multiple allowed | No | Red-Black Tree |
| Hashtable | No order | Not allowed | Not allowed | Yes (synchronized) | Array + LinkedList |
| ConcurrentHashMap | No order | Not allowed | Not allowed | Yes (fine-grained) | Array + LinkedList/RB-Tree + CAS |
| WeakHashMap | No order | 1 allowed | Multiple allowed | No | Array + LinkedList (weak refs to keys) |
| IdentityHashMap | No order | 1 allowed | Multiple allowed | No | Open-addressing array |
| EnumMap | Natural order of enum | Not allowed | Multiple allowed | No | Array indexed by enum ordinal |
| ConcurrentSkipListMap | Sorted | Not allowed | Not allowed | Yes (lock-free) | Skip List |

---

## 2. HashMap — Internal Structure & Data Storage (Deep Dive)

### 2.1 The core data structure

Internally, `HashMap` maintains an **array of `Node<K,V>` objects** called the **table** (bucket array):

```java
// Simplified from java.util.HashMap source
transient Node<K,V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;   // forms a singly linked list within a bucket

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash; this.key = key; this.value = value; this.next = next;
    }
}
```

- Default initial capacity = **16** (must always be a power of 2).
- Default load factor = **0.75**.
- Threshold = capacity × load factor (resize trigger point) = 16 × 0.75 = 12.

Since Java 8, if a bucket's linked list grows beyond **8 nodes** AND table capacity ≥ 64, that bucket is converted into a **Red-Black Tree** (`TreeNode`) for O(log n) worst-case lookup instead of O(n). This is called **treeification**.

```java
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;   // untreeify when shrinks back below this on split
static final int MIN_TREEIFY_CAPACITY = 64; // won't treeify if table smaller than this; resizes instead
```

### 2.2 How `put(key, value)` stores data — step by step

```java
map.put("apple", 10);
```

Internally (simplified logic from `HashMap.putVal()`):

**Step 1 — Compute hash of the key.**
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
This is called **hash spreading / perturbation**. Why XOR with `h >>> 16`?
- `hashCode()` can return any 32-bit int, but the bucket index only uses the **lower bits** (see step 2).
- If two keys have hashCodes that differ only in high bits (common with poor hashCode implementations, e.g. `Object.hashCode()` based on memory address), those high bits would never influence bucket selection without spreading — causing more collisions.
- XOR-ing the high 16 bits into the low 16 bits mixes entropy from the whole 32-bit hash into the bits that matter, reducing collisions cheaply (cheaper than a full re-hash function).

**Step 2 — Compute bucket index.**
```java
index = (n - 1) & hash;   // n = table.length (power of 2)
```
Because `n` is always a power of 2, `(n-1)` is a bit-mask of all 1s (e.g., n=16 → n-1=15=0b1111). `hash & (n-1)` is equivalent to `hash % n` but much faster (bitwise AND vs modulo division). **This is exactly why capacity must always be a power of 2.**

**Step 3 — Insert into the bucket.**
- If `table[index]` is empty → place new `Node` there directly.
- If not empty (collision) →
    - Traverse the linked list (or tree) at that bucket.
    - If a node with an **equal hash AND equal key** (`key.equals()`) is found → **replace** its value (put overwrites), return old value.
    - Otherwise, append the new node at the **end** of the list (Java 8+; Java 7 used head-insertion).
    - If the bucket is already a tree (`TreeNode`), insert via red-black tree insertion logic (`putTreeVal`).
    - If the linked list length after insertion ≥ `TREEIFY_THRESHOLD (8)` and table capacity ≥ 64 → convert bucket to a tree (`treeifyBin`).

**Step 4 — Check resize condition.**
- `size++`. If `size > threshold` → call `resize()` (doubles capacity, see Section 5).

### Visual representation

```
table[] (capacity 16)
index 0  -> null
index 1  -> Node("apple", 10) -> Node("grape", 5)   // collision: both hashed to bucket 1
index 2  -> null
...
index 15 -> Node("mango", 3)
```

### 2.3 Full annotated code walkthrough

```java
import java.util.HashMap;

public class HashMapInternalsDemo {
    public static void main(String[] args) {
        HashMap<String, Integer> map = new HashMap<>();

        // Internally: table = null until first put() -> lazily initialized to size 16
        map.put("apple", 10);   // hash("apple") computed, index = hash & 15, Node placed
        map.put("grape", 20);
        map.put("mango", 30);

        // put() returns the OLD value if key existed, else null
        Integer old = map.put("apple", 100); // apple's value updated 10 -> 100
        System.out.println("Old value: " + old); // 10

        System.out.println(map.get("apple")); // 100
    }
}
```

### 2.4 Why HashMap is NOT thread-safe (structural insight)

- Multiple threads calling `put()` simultaneously can corrupt the linked list structure (e.g., in Java 7, concurrent resize could create a **cyclic linked list**, causing `get()` to infinite-loop — a well-known production bug).
- Java 8 fixed the cycle bug but HashMap is still **not thread-safe**: lost updates, inconsistent size, or `ConcurrentModificationException` during iteration can occur under concurrent modification.
- Use `ConcurrentHashMap` or `Collections.synchronizedMap(new HashMap<>())` for thread safety.

---

## 3. How HashMap Retrieves Data — `get()` internals

```java
map.get("apple");
```

Step by step (`HashMap.getNode()`):

1. Compute `hash("apple")` using the same spreading function as `put`.
2. Compute bucket index: `(n-1) & hash`.
3. Go to `table[index]`:
    - If `table[index] == null` → return `null` (key not present).
    - Else, check the **first node**: if `node.hash == hash` AND (`node.key == key` OR `key.equals(node.key)`) → return `node.value`.
    - Else, if the bucket is a **tree** → do red-black tree search: O(log n).
    - Else, traverse the **linked list**, using the same hash+equals check for each node: O(n) worst case per bucket (n = list length), but average O(1) if hashCode() is well distributed.

```java
// Simplified getNode() logic
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

**Why both `hash` comparison AND `equals()` check?**
- Comparing `int hash` first is a cheap, fast filter (avoids calling potentially expensive `equals()` unnecessarily).
- Two different keys CAN have the same hash (collision), so `equals()` is the final authority to confirm actual key match.

**Time Complexity Summary:**

| Operation | Average Case | Worst Case (Java 7, many collisions) | Worst Case (Java 8+, treeified) |
|---|---|---|---|
| get/put/remove | O(1) | O(n) | O(log n) |

---

## 4. Hash Collisions & Resolution

**What is a collision?** Two different keys produce the same bucket index (`(n-1) & hash`), either because their `hashCode()` values are equal, or because different hash codes map to the same index after masking.

### 4.1 Collision Resolution Strategy used by HashMap: **Separate Chaining**

Each bucket holds a linked list (or tree) of all entries that hash to that index — as shown above. This is opposed to **open addressing** (used by `IdentityHashMap`), where on collision you probe for the *next* free slot in the array itself.

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public int hashCode() {
        return 1; // BAD hashCode on purpose - forces all Points into same bucket
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
}

public class CollisionDemo {
    public static void main(String[] args) {
        Map<Point, String> map = new HashMap<>();
        map.put(new Point(1, 1), "A");
        map.put(new Point(2, 2), "B");
        map.put(new Point(3, 3), "C");
        // All 3 go into the SAME bucket (hashCode always returns 1)
        // They form a linked list: Node(1,1) -> Node(2,2) -> Node(3,3)
        // get() must now traverse the list, comparing hash + equals for each - O(n)!
        System.out.println(map.get(new Point(2, 2))); // "B" - found via equals() after traversal
    }
}
```

### 4.2 Why collisions degrade performance & how Java 8 mitigates it

- Worst case: a poor `hashCode()` (like above) turns HashMap into a glorified `LinkedList` — O(n) per operation.
- **Java 8 improvement:** if a single bucket's chain length ≥ 8 (and table capacity ≥ 64), it's converted to a **Red-Black Tree**, bounding worst-case lookup to **O(log n)**.
- This is also a defense against **algorithmic complexity attacks** (an attacker crafting many keys with identical hashCodes to intentionally degrade a server's HashMap-based structure, e.g. HTTP parameter parsing — a real historical DoS vector in several web frameworks).

### 4.3 Other collision resolution strategies (general CS knowledge, good to mention)

| Strategy | Description | Used by |
|---|---|---|
| Separate Chaining | Bucket holds list/tree of colliding entries | `HashMap`, `Hashtable`, `ConcurrentHashMap` |
| Open Addressing (Linear Probing) | On collision, probe next slot(s) in array | `IdentityHashMap` |
| Open Addressing (Double Hashing) | Use a second hash function to compute probe sequence | Some DB indexes |

---

## 5. HashMap Resizing (Rehashing)

### 5.1 When does resize happen?

After every successful `put()`, `size` is incremented and compared:
```java
if (++size > threshold)
    resize();
```
where `threshold = capacity * loadFactor` (default 16 × 0.75 = 12).

So inserting the **13th** entry into a default HashMap triggers a resize.

### 5.2 What resize() does

1. **Doubles the capacity**: `newCapacity = oldCapacity << 1` (16 → 32 → 64 ...). Capacity always stays a power of 2.
2. **Doubles the threshold** too: `newThreshold = newCapacity * loadFactor`.
3. **Re-distributes every existing node** from the old table into the new table.

### 5.3 The clever Java 8 rehashing trick (avoids recomputing hashCode)

In Java 7, resize required recalculating `(n-1) & hash` for every single node individually — expensive.

In Java 8+, since capacity always doubles (a single new bit becomes part of the mask), each old bucket's entries can only go to **one of exactly two positions** in the new table:
- Same index (`j`), or
- `j + oldCapacity`

This is determined by checking **one extra bit** of the original hash (`hash & oldCapacity`):
```java
// Simplified resize() logic for a bucket's linked list
if ((e.hash & oldCap) == 0) {
    // stays at same index j in new table -> "lo" list
} else {
    // moves to index j + oldCap -> "hi" list
}
```

**Why this works:** Since `newCap = oldCap * 2`, the new mask (`newCap - 1`) has exactly one more significant bit than the old mask (`oldCap - 1`). Whether an entry moves depends solely on that new bit. This avoids recomputing the hash and avoids full-blown rehashing — just a bit check and a list split. Both `lo` and `hi` sublists preserve **relative order** (Java 8+ guarantee, unlike Java 7 which reversed order and had a resize-during-concurrency bug causing infinite loops).

### 5.4 Example walk-through

```java
HashMap<Integer, String> map = new HashMap<>(); // capacity 16, threshold 12

for (int i = 1; i <= 13; i++) {
    map.put(i, "val" + i);
    // After the 13th put(), size(13) > threshold(12) -> resize() triggered
    // capacity 16 -> 32, threshold 12 -> 24
}
```

### 5.5 Performance implication & tuning tip

- Resizing is O(n) (must rehash-place every entry) — expensive if it happens repeatedly during heavy insertion.
- **Interview tip:** If you know the expected number of entries upfront, **pre-size the HashMap** to avoid multiple resizes:
```java
// If you expect ~1000 entries, avoid resize churn:
int expectedSize = 1000;
Map<String, String> map = new HashMap<>((int) (expectedSize / 0.75) + 1);
```

### 5.6 Load Factor trade-off

| Load Factor | Effect |
|---|---|
| Lower (e.g. 0.5) | More memory used (larger table, sparser), fewer collisions, faster lookups |
| Higher (e.g. 0.9) | Less memory, more collisions, slower lookups, resizes less often |
| Default 0.75 | Balanced trade-off between time and space cost (per Java doc / empirical tuning) |

---

## 6. Why Override `hashCode()` and `equals()` for Custom Objects

### 6.1 The contract (very frequently asked)

From `Object` class Javadoc, the **hashCode-equals contract**:
1. If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** be true.
2. If `a.hashCode() == b.hashCode()`, `a.equals(b)` is **not required** to be true (hash collision is allowed).
3. `hashCode()` must consistently return the same value across multiple invocations (as long as no fields used in `equals()` change).
4. `equals()` must be reflexive, symmetric, transitive, consistent, and `x.equals(null)` must return `false`.

### 6.2 Why it matters for HashMap specifically

If you **don't override `hashCode()` and `equals()`**, Java uses `Object`'s default implementation:
- Default `hashCode()` is derived from the object's **memory address** (identity hash).
- Default `equals()` checks **reference equality** (`==`).

This means two "logically equal" custom objects (same field values) will:
- Have **different hashCodes** → land in **different buckets**.
- Fail `equals()` → even if they collide, HashMap won't treat them as duplicates.

**Result: You can never retrieve a value using a "different but equal" key object**, and duplicate logical keys will silently coexist.

### 6.3 Demonstration of the bug

```java
class Employee {
    int id;
    String name;
    Employee(int id, String name) { this.id = id; this.name = name; }
    // NO hashCode() / equals() override
}

public class BadKeyDemo {
    public static void main(String[] args) {
        Map<Employee, String> map = new HashMap<>();
        map.put(new Employee(101, "Raj"), "Engineering");

        // Logically the "same" employee, but a DIFFERENT object reference
        Employee lookupKey = new Employee(101, "Raj");
        System.out.println(map.get(lookupKey)); // prints: null  <-- BUG!

        map.put(lookupKey, "Sales"); // Adds a SECOND entry instead of updating!
        System.out.println(map.size()); // 2 (should logically be 1)
    }
}
```

### 6.4 Correct implementation

```java
import java.util.Objects;

class Employee {
    int id;
    String name;

    Employee(int id, String name) { this.id = id; this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                     // reflexive shortcut
        if (o == null || getClass() != o.getClass()) return false;
        Employee employee = (Employee) o;
        return id == employee.id && Objects.equals(name, employee.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name); // combines field hashes consistently
    }
}

public class GoodKeyDemo {
    public static void main(String[] args) {
        Map<Employee, String> map = new HashMap<>();
        map.put(new Employee(101, "Raj"), "Engineering");

        Employee lookupKey = new Employee(101, "Raj");
        System.out.println(map.get(lookupKey)); // "Engineering" - correctly found!

        map.put(lookupKey, "Sales"); // correctly UPDATES the same entry
        System.out.println(map.size()); // 1
    }
}
```

### 6.5 `Objects.hash()` internal logic (good to know)

```java
public static int hash(Object... values) {
    return Arrays.hashCode(values);
}
// Arrays.hashCode does: result = 1; for each element: result = 31*result + (elem==null?0:elem.hashCode());
```
**Why 31?** It's an odd prime — odd so multiplication doesn't lose the lowest bit info via overflow shifting, and prime so it distributes bits well. Also, `31 * i == (i << 5) - i`, which JIT compilers can auto-optimize (bit-shift instead of multiply) — a nice historical performance justification.

### 6.6 Rules of thumb for custom key objects
- **Prefer immutable objects as keys** (e.g., `String`, `Integer`, or custom classes with `final` fields). If a mutable field used in `hashCode()`/`equals()` changes *after* insertion, the entry becomes "lost" — its bucket location no longer matches its current hash, and `get()` will fail to find it.
- Always override both together, never just one (violates the contract).
- Use `Objects.equals()` and `Objects.hash()` (from `java.util.Objects`) to avoid NPEs and boilerplate.
- IDEs (IntelliJ/Eclipse) and Lombok's `@EqualsAndHashCode` can auto-generate these safely.

```java
// Demonstrating the mutable-key danger
class MutableKey {
    int value;
    MutableKey(int value) { this.value = value; }
    @Override public int hashCode() { return value; }
    @Override public boolean equals(Object o) {
        return o instanceof MutableKey mk && mk.value == value;
    }
}

public class MutableKeyBugDemo {
    public static void main(String[] args) {
        Map<MutableKey, String> map = new HashMap<>();
        MutableKey key = new MutableKey(1);
        map.put(key, "data");

        key.value = 2; // mutate the key AFTER insertion - hashCode changes!

        System.out.println(map.get(key));          // null! (looked up bucket for hash=2, but stored in bucket for hash=1)
        System.out.println(map.get(new MutableKey(1))); // also null (bucket 1 has a Node whose current hashCode is now 2 - inconsistent!)
        // The entry is effectively "lost" in the map, still consuming memory
    }
}
```

---

## 7. LinkedHashMap + LRU Cache Implementation

### 7.1 What is LinkedHashMap?

`LinkedHashMap` extends `HashMap` and additionally maintains a **doubly linked list** running through all entries, giving predictable iteration order:
- **Insertion order** (default), or
- **Access order** (if constructed with `accessOrder = true`) — every `get()`/`put()` moves the entry to the end of the list ("most recently used" at the tail).

```java
class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after; // extra pointers for the linked list, in addition to HashMap's `next`
}
```

So each entry lives in **two structures simultaneously**: the HashMap bucket array (for O(1) lookup) and the doubly linked list (for ordered iteration).

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LinkedHashMapDemo {
    public static void main(String[] args) {
        // insertion-order (default)
        Map<String, Integer> insertionOrderMap = new LinkedHashMap<>();
        insertionOrderMap.put("c", 3);
        insertionOrderMap.put("a", 1);
        insertionOrderMap.put("b", 2);
        System.out.println(insertionOrderMap); // {c=3, a=1, b=2} - insertion order preserved

        // access-order (true) - used for LRU
        Map<String, Integer> accessOrderMap = new LinkedHashMap<>(16, 0.75f, true);
        accessOrderMap.put("c", 3);
        accessOrderMap.put("a", 1);
        accessOrderMap.put("b", 2);
        accessOrderMap.get("c"); // accessing "c" moves it to the END (most recently used)
        System.out.println(accessOrderMap); // {a=1, b=2, c=3}
    }
}
```

### 7.2 Building an LRU Cache using LinkedHashMap (VERY common interview question)

`LinkedHashMap` provides a hook method specifically designed for this:
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false; // override to return true when you want the oldest entry evicted
}
```

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // initialCapacity, loadFactor, accessOrder=true (crucial!)
        super(16, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Called automatically after every put(); if true, HashMap removes 'eldest' for us
        return size() > capacity;
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        cache.get(1);          // access 1 -> moves 1 to most-recently-used end
        cache.put(4, "D");     // capacity exceeded -> evicts least recently used (which is 2, since 1 was just accessed)

        System.out.println(cache.keySet()); // [3, 1, 4]  -- "2" was evicted
    }
}
```

**How it works internally:**
1. `accessOrder=true` means every `get()` call internally calls `afterNodeAccess(node)`, unlinking the node and re-linking it at the **tail** of the doubly linked list.
2. Every `put()` calls `afterNodeInsertion(evict)`, which checks `removeEldestEntry(head)`. The **head** of the doubly linked list is always the least-recently-used entry.
3. If `removeEldestEntry` returns `true`, the head node is removed — O(1) since it's a direct linked-list pointer, no traversal needed.

**This gives O(1) get and O(1) put for a full LRU cache** — that's the whole trick: HashMap gives O(1) lookup, and the linked list gives O(1) access-order tracking + O(1) eviction of the oldest.

### 7.3 Manual LRU Cache (without extending LinkedHashMap — sometimes asked to implement from scratch)

```java
import java.util.HashMap;
import java.util.Map;

public class ManualLRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // dummy head (MRU side)
    private final Node<K, V> tail = new Node<>(null, null); // dummy tail (LRU side)

    static class Node<K, V> {
        K key; V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    public ManualLRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        remove(node);
        insertAtFront(node);
        return node.value;
    }

    public void put(K key, V value) {
        if (map.containsKey(key)) {
            remove(map.get(key));
        }
        if (map.size() == capacity) {
            Node<K, V> lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
        Node<K, V> node = new Node<>(key, value);
        insertAtFront(node);
        map.put(key, node);
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insertAtFront(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```
This is the classic **HashMap + Doubly Linked List** approach — asked frequently as a whiteboard/LeetCode-style question (LeetCode #146). Know both the `LinkedHashMap`-based shortcut AND this manual version.

---

## 8. WeakHashMap

`WeakHashMap` stores keys using **`WeakReference`**. If a key object has no other strong references anywhere in the application, it becomes eligible for **garbage collection**, and the GC will automatically remove the corresponding entry from the map (during the next GC cycle).

### 8.1 Why / when to use it

Ideal for **memory-sensitive caches** where you want entries to be automatically cleaned up once the key is no longer used elsewhere (avoids memory leaks from long-lived caches holding references indefinitely).

```java
import java.util.Map;
import java.util.WeakHashMap;

public class WeakHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        Map<Object, String> weakMap = new WeakHashMap<>();

        Object key1 = new Object();
        weakMap.put(key1, "Value for key1");

        System.out.println("Before GC: " + weakMap.size()); // 1

        key1 = null; // remove the only strong reference to key1
        System.gc();  // suggest GC (not guaranteed but very likely for this trivial case)
        Thread.sleep(500);

        System.out.println("After GC: " + weakMap.size()); // 0 (entry auto-removed)
    }
}
```

### 8.2 Internal mechanism
- Each `Entry` internally extends `WeakReference<Object>` wrapping the key.
- When GC clears a weak reference, that reference is placed on a `ReferenceQueue`.
- `WeakHashMap` checks this queue at the start of most operations (`get`, `put`, `size`, etc.) via `expungeStaleEntries()`, physically removing the dead entries from the table.

**Note:** Only **keys** are weakly referenced; **values** are held with normal strong references (so if a value indirectly references its own key, that keeps the key alive — a common gotcha).

**Real-world use case:** listener registries, class-metadata caches (e.g. ClassLoader-based caches), canonicalizing mappings where you don't want to prevent normal GC.

---

## 9. IdentityHashMap

`IdentityHashMap` uses **reference equality (`==`)** instead of `equals()`, and **`System.identityHashCode()`** instead of `hashCode()`, for comparing keys (and optionally values in `remove`/`containsValue`).

```java
import java.util.IdentityHashMap;
import java.util.Map;

public class IdentityHashMapDemo {
    public static void main(String[] args) {
        String key1 = new String("A"); // deliberately new object, not interned
        String key2 = new String("A"); // logically equal, but different reference

        Map<String, Integer> normalMap = new java.util.HashMap<>();
        normalMap.put(key1, 1);
        normalMap.put(key2, 2); // equals() is true -> overwrites key1's entry
        System.out.println(normalMap.size()); // 1

        Map<String, Integer> identityMap = new IdentityHashMap<>();
        identityMap.put(key1, 1);
        identityMap.put(key2, 2); // key1 != key2 (different references) -> treated as DIFFERENT keys
        System.out.println(identityMap.size()); // 2
    }
}
```

### 9.1 Internal structure difference
- Uses **open addressing** (linear probing), NOT separate chaining — stores keys and values in a **single flat `Object[]` array**, alternating: `[key0, value0, key1, value1, ...]`.
- On collision, it linearly probes to the next slot instead of chaining a list.
- This makes it more memory-compact for this specific use case but means it does NOT implement the general `Map` contract properly per Javadoc (it violates the `equals`/`hashCode` contract intentionally, so it's documented as a special-purpose tool, not a general-purpose Map).

**Real-world use case:** Topology-preserving object graph traversals (e.g., serialization frameworks, deep-copy/clone utilities) where you must track "have I already visited this exact object instance" regardless of whether two objects are logically `.equals()`. Also used internally in some IDE/JVM tooling and `java.io.ObjectOutputStream`'s internal handle table.

---

## 10. SortedMap

`SortedMap<K,V>` is an interface extending `Map<K,V>` that maintains keys in **ascending order**, either:
- **Natural ordering** (keys implement `Comparable`), or
- A custom **`Comparator`** provided at construction time.

```java
public interface SortedMap<K,V> extends Map<K,V> {
    Comparator<? super K> comparator();
    SortedMap<K,V> subMap(K fromKey, K toKey);
    SortedMap<K,V> headMap(K toKey);
    SortedMap<K,V> tailMap(K fromKey);
    K firstKey();
    K lastKey();
}
```

```java
import java.util.SortedMap;
import java.util.TreeMap;

public class SortedMapDemo {
    public static void main(String[] args) {
        SortedMap<Integer, String> map = new TreeMap<>();
        map.put(5, "five");
        map.put(1, "one");
        map.put(3, "three");

        System.out.println(map);               // {1=one, 3=three, 5=five} - sorted!
        System.out.println(map.firstKey());     // 1
        System.out.println(map.lastKey());      // 5
        System.out.println(map.headMap(3));      // {1=one} - keys strictly less than 3
        System.out.println(map.tailMap(3));      // {3=three, 5=five} - keys >= 3
        System.out.println(map.subMap(1, 5));    // {1=one, 3=three} - [1,5)
    }
}
```
`TreeMap` is the primary implementation. `ConcurrentSkipListMap` is the thread-safe implementation.

---

## 11. NavigableMap

`NavigableMap<K,V>` extends `SortedMap<K,V>`, adding **navigation methods** to find closest matches — extremely useful and commonly tested:

```java
public interface NavigableMap<K,V> extends SortedMap<K,V> {
    Map.Entry<K,V> lowerEntry(K key);    // strictly less than key
    Map.Entry<K,V> floorEntry(K key);    // less than or equal to key
    Map.Entry<K,V> ceilingEntry(K key);  // greater than or equal to key
    Map.Entry<K,V> higherEntry(K key);   // strictly greater than key
    Map.Entry<K,V> firstEntry();
    Map.Entry<K,V> lastEntry();
    Map.Entry<K,V> pollFirstEntry();     // retrieves AND removes first entry
    Map.Entry<K,V> pollLastEntry();
    NavigableMap<K,V> descendingMap();   // reverse-order view
    NavigableSet<K> navigableKeySet();
}
```

```java
import java.util.NavigableMap;
import java.util.TreeMap;

public class NavigableMapDemo {
    public static void main(String[] args) {
        NavigableMap<Integer, String> map = new TreeMap<>();
        map.put(10, "ten");
        map.put(20, "twenty");
        map.put(30, "thirty");

        System.out.println(map.lowerKey(20));   // 10  (strictly less than 20)
        System.out.println(map.floorKey(20));    // 20  (<= 20)
        System.out.println(map.ceilingKey(20));  // 20  (>= 20)
        System.out.println(map.higherKey(20));   // 30  (strictly greater than 20)

        System.out.println(map.floorKey(15));    // 10  (largest key <= 15)
        System.out.println(map.ceilingKey(15));  // 20  (smallest key >= 15)

        System.out.println(map.descendingMap()); // {30=thirty, 20=twenty, 10=ten}

        Map.Entry<Integer, String> polled = map.pollFirstEntry(); // removes & returns 10=ten
        System.out.println(polled + " | remaining: " + map);
    }
}
```

**Real-world use case:** range queries, "find nearest timestamp/version ≤ X" problems (e.g., time-series lookups, versioned config lookups, interval scheduling) — a classic use in system design and DSA interviews.

---

## 12. TreeMap (Red-Black Tree Internals)

### 12.1 Structure
`TreeMap` implements `NavigableMap` using a **self-balancing binary search tree — specifically a Red-Black Tree**.

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key; V value;
    Entry<K,V> left, right, parent;
    boolean color = BLACK;
}
```

### 12.2 Why Red-Black Tree (not a plain BST)?

A plain BST can degrade to O(n) (e.g., inserting sorted data creates a linked-list-shaped tree). A Red-Black Tree guarantees **O(log n)** for insert/delete/search by enforcing balancing invariants via node "colors" and rotations:
1. Every node is Red or Black.
2. Root is always Black.
3. No two adjacent Red nodes (Red node's children must be Black).
4. Every path from a node to its descendant `null` leaves has the same number of Black nodes ("black-height").
5. New nodes are inserted as Red, then re-balanced (rotations + recoloring) to restore invariants.

These invariants bound the tree height to `~2*log2(n)`, guaranteeing worst-case **O(log n)** for `put`, `get`, `remove` — unlike `HashMap`'s average O(1) but with **ordering** as a bonus.

### 12.3 Code example

```java
import java.util.Comparator;
import java.util.TreeMap;

public class TreeMapDemo {
    public static void main(String[] args) {
        // Natural ordering
        TreeMap<String, Integer> map = new TreeMap<>();
        map.put("banana", 2);
        map.put("apple", 1);
        map.put("cherry", 3);
        System.out.println(map); // {apple=1, banana=2, cherry=3}

        // Custom comparator - reverse order
        TreeMap<String, Integer> reverseMap = new TreeMap<>(Comparator.reverseOrder());
        reverseMap.putAll(map);
        System.out.println(reverseMap); // {cherry=3, banana=2, apple=1}

        // Custom objects require Comparable or an explicit Comparator
        TreeMap<Employee2, String> empMap = new TreeMap<>(Comparator.comparingInt(e -> e.id));
        empMap.put(new Employee2(3, "C"), "role3");
        empMap.put(new Employee2(1, "A"), "role1");
        System.out.println(empMap.firstKey().id); // 1
    }
}

class Employee2 {
    int id; String name;
    Employee2(int id, String name) { this.id = id; this.name = name; }
}
```

**Important:** `TreeMap` does **NOT allow `null` keys** (throws `NullPointerException`) because it must compare keys to place them — comparing against `null` is undefined. `null` values ARE allowed.

**TreeMap vs HashMap — when to use which (classic interview question):**

| Criteria | HashMap | TreeMap |
|---|---|---|
| Ordering | None | Sorted by key |
| Time complexity | O(1) avg | O(log n) always |
| Null keys | 1 allowed | Not allowed |
| Use case | Fast lookups, no order needed | Range queries, sorted iteration, "nearest key" lookups |

---

## 13. Hashtable

### 13.1 Overview
`Hashtable` is a **legacy** class (pre-dates Collections Framework, JDK 1.0), retrofitted to implement `Map` in Java 1.2. It is **synchronized** — every method (`get`, `put`, etc.) is `synchronized` at the **object/instance level** (a single global lock for the whole map).

```java
public synchronized V put(K key, V value) { ... }
public synchronized V get(Object key) { ... }
```

### 13.2 Key differences from HashMap

| Feature | HashMap | Hashtable |
|---|---|---|
| Thread safety | No | Yes (coarse-grained, whole-object lock) |
| Null keys/values | 1 null key, multiple null values | **No nulls allowed at all** (throws NPE) |
| Performance | Faster (no locking overhead) | Slower (lock contention under concurrency) |
| Iterator | fail-fast | Enumeration is NOT fail-fast (legacy); Iterator (via keySet etc.) IS fail-fast |
| Introduced | Java 1.2 (Collections Framework) | Java 1.0 (legacy) |

```java
import java.util.Hashtable;

public class HashtableDemo {
    public static void main(String[] args) {
        Hashtable<String, String> table = new Hashtable<>();
        table.put("key1", "value1");
        try {
            table.put(null, "value2"); // throws NullPointerException
        } catch (NullPointerException e) {
            System.out.println("Hashtable does not allow null keys!");
        }
    }
}
```

**Why avoid Hashtable in modern code:** the single global lock means only **one thread** can access the map at a time (even for reads), causing heavy contention. `ConcurrentHashMap` gives much better concurrent throughput via fine-grained locking. **Hashtable is considered obsolete — use `ConcurrentHashMap` for thread-safety, or plain `HashMap` for single-threaded use.**

---

## 14. ConcurrentHashMap (Java 7 vs Java 8 Internals) — HIGH PRIORITY TOPIC

This is one of the **most commonly asked deep-dive topics** for 3+ YOE Java backend interviews. Know both versions.

### 14.1 Java 7 Implementation — Segment-based locking

Java 7's `ConcurrentHashMap` divides the map into a fixed number of **Segments** (default 16), each of which is essentially its **own independent Hashtable-like structure with its own lock**.

```java
// Simplified Java 7 structure
Segment<K,V>[] segments;   // default 16 segments

static class Segment<K,V> extends ReentrantLock {
    HashEntry<K,V>[] table; // each segment has its own bucket array
}
```

- A `put()` only locks the **specific segment** the key's hash falls into, NOT the whole map.
- This allows up to **16 threads to write concurrently** (one per segment) without blocking each other, as long as they hit different segments.
- Reads (`get()`) generally don't need locks at all (uses `volatile` reads), for good read concurrency.
- Concurrency level (number of segments) was fixed at creation time and could be tuned via constructor: `new ConcurrentHashMap<>(initialCapacity, loadFactor, concurrencyLevel)`.

### 14.2 Java 8+ Implementation — Node-level locking + CAS (major redesign)

Java 8 **removed the Segment concept entirely**. It now uses the **same `Node<K,V>[] table` structure as HashMap**, but achieves thread-safety via:

1. **CAS (Compare-And-Swap)** operations for lock-free updates when inserting into an **empty bucket**.
2. **`synchronized` block on the first node of a bucket** (not the whole table) only when there's an actual collision to resolve within that specific bucket.
3. Reads are (mostly) **lock-free**, using `volatile` fields (`Node.val`, `Node.next`) to ensure visibility across threads without blocking.

```java
// Simplified putVal() logic (Java 8 ConcurrentHashMap)
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i;
        if (tab == null) tab = initTable();  // lazy init, CAS-guarded
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // Bucket is empty - try CAS to insert directly, NO LOCK NEEDED
            if (casTabAt(tab, i, null, new Node<>(hash, key, value, null)))
                break; // success, no locking required
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f); // help ongoing resize
        else {
            // Collision on this bucket - lock only THIS bucket's head node
            synchronized (f) {
                // traverse linked list or tree, insert/update
            }
        }
    }
    addCount(1L, binCount); // updates size using CAS-based counter (LongAdder-like)
}
```

**Why this is faster than segment-locking:** Lock granularity went from "1 lock per ~1/16th of the map" to effectively **"1 lock per bucket"** (there can be thousands of buckets), massively reducing contention. Also, empty-bucket insertion needs **zero locking at all** (pure CAS).

### 14.3 Treeification in ConcurrentHashMap

Just like `HashMap`, if a bucket's chain exceeds 8 nodes (and table ≥ 64), it treeifies into a red-black tree — same performance protection against hash-flooding attacks, done safely under the bucket's lock.

### 14.4 Size tracking — why not just `AtomicInteger`?

Under heavy concurrent writes, a single shared counter (even atomic) becomes a contention hotspot (all threads CAS-racing on ONE variable). Java 8 `ConcurrentHashMap` uses a `LongAdder`-inspired striped counter (`CounterCell[]` array) — different threads increment different cells (based on thread-probe hashing), and `size()`/`mappingCount()` sums them up only when actually queried. This trades a tiny bit of read-latency for drastically reduced write contention.

### 14.5 Key behavioral differences from HashMap

| Feature | HashMap | ConcurrentHashMap |
|---|---|---|
| Thread-safe | No | Yes |
| Null keys/values | 1 null key allowed | **No nulls at all** (throws NPE) — because in a concurrent context, `map.get(key) == null` is ambiguous (does it mean absent, or the value IS null?) — this ambiguity is dangerous without external synchronization, so nulls are disallowed entirely. |
| Locking | N/A | Java 7: segment locks. Java 8+: CAS + per-bucket synchronized |
| Iterator | fail-fast (throws CME) | **Weakly consistent** (fail-safe) — never throws CME; may or may not reflect updates made during iteration |
| Performance under concurrency | N/A (not safe) | High throughput, especially reads |

### 14.6 Code example demonstrating safe concurrent use + atomic compound operations

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ConcurrentHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> counterMap = new ConcurrentHashMap<>();
        ExecutorService executor = Executors.newFixedThreadPool(10);

        // Simulate 10 threads incrementing counters for the same set of keys concurrently
        for (int i = 0; i < 1000; i++) {
            String key = "key" + (i % 5);
            executor.submit(() ->
                // merge() is ATOMIC - safe compound read-modify-write, no explicit locking needed
                counterMap.merge(key, 1, Integer::sum)
            );
        }
        executor.shutdown();
        executor.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
        System.out.println(counterMap); // each of the 5 keys should sum to 200, total 1000

        // Other useful atomic methods:
        counterMap.putIfAbsent("newKey", 0);       // atomic "insert if not exists"
        counterMap.computeIfAbsent("anotherKey", k -> 100); // atomic lazy init
        counterMap.compute("key0", (k, v) -> v == null ? 1 : v + 1); // atomic update
    }
}
```

**Interview gotcha:** `putIfAbsent`, `compute`, `computeIfAbsent`, `merge` are all **atomic** on `ConcurrentHashMap` — this is the main reason to prefer these over manual `if (!map.containsKey(k)) map.put(k, v)` patterns, which have a race condition (check-then-act) even on a "thread-safe" map.

```java
// RACE CONDITION - even though map is thread-safe, this compound operation is NOT atomic:
if (!concurrentMap.containsKey("x")) {
    concurrentMap.put("x", 1); // another thread could insert "x" between these two lines!
}

// CORRECT - atomic:
concurrentMap.putIfAbsent("x", 1);
```

---

## 15. Immutable Map

An immutable map cannot be structurally modified after creation (no `put`, `remove`, `clear` — these throw `UnsupportedOperationException`).

### 15.1 Ways to create immutable maps

**(a) `Collections.unmodifiableMap()`** — wraps an existing map in a read-only VIEW (the underlying map can still be modified through the original reference — it's a "shallow" immutability, a common gotcha).

```java
import java.util.*;

public class UnmodifiableMapDemo {
    public static void main(String[] args) {
        Map<String, Integer> mutable = new HashMap<>();
        mutable.put("a", 1);

        Map<String, Integer> readOnlyView = Collections.unmodifiableMap(mutable);

        try {
            readOnlyView.put("b", 2); // throws UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("Cannot modify via the view");
        }

        mutable.put("c", 3); // but the ORIGINAL map can still be changed!
        System.out.println(readOnlyView); // {a=1, c=3} <- view reflects the change! NOT truly immutable.
    }
}
```

**(b) `Map.of()` / `Map.ofEntries()` (Java 9+)** — creates a **truly immutable** map (not a view of anything mutable).

```java
Map<String, Integer> trulyImmutable = Map.of("a", 1, "b", 2, "c", 3); // up to 10 pairs supported via Map.of()

Map<String, Integer> largerMap = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
    // ... unlimited entries via ofEntries()
);
// trulyImmutable.put("d", 4); -> throws UnsupportedOperationException
// Map.of() also does NOT allow null keys or values (throws NPE immediately at creation)
// Map.of() does NOT allow duplicate keys (throws IllegalArgumentException at creation)
```

**(c) `Collections.emptyMap()` / `Collections.singletonMap()`** — specialized immutable maps for 0 or 1 entries (memory-efficient special-cased implementations).

```java
Map<String, Integer> empty = Collections.emptyMap();
Map<String, Integer> single = Collections.singletonMap("onlyKey", 42);
```

**(d) Guava's `ImmutableMap`** (third-party, but frequently mentioned in interviews for large codebases):
```java
// com.google.common.collect.ImmutableMap
ImmutableMap<String, Integer> guavaMap = ImmutableMap.of("a", 1, "b", 2);
ImmutableMap<String, Integer> builderMap = ImmutableMap.<String, Integer>builder()
        .put("a", 1)
        .put("b", 2)
        .build();
```
Guava's version pre-dates `Map.of()` (Java 9), offers a fluent `Builder` API for arbitrary sizes, and additionally guarantees **iteration order = insertion order** (which `Map.of()` explicitly does NOT guarantee — its iteration order can even vary between JVM runs, by design, to prevent code from accidentally depending on it).

### 15.2 Why use immutable maps
- **Thread safety for free** — no synchronization needed for read-only shared config/lookup data across threads.
- Prevents accidental/unintended mutation bugs (defensive by design — fail fast at the point of misuse rather than silent corruption).
- Enables safe use as constants (`public static final Map<...> CONFIG = Map.of(...)`).

---

## 16. EnumMap

`EnumMap<K extends Enum<K>, V>` is a specialized `Map` implementation exclusively for **enum keys**, backed internally by a simple **array indexed directly by the enum's `ordinal()`** — not a hash table at all.

```java
public class EnumMap<K extends Enum<K>, V> {
    private final Class<K> keyType;
    private transient Object[] vals; // array sized to keyType.getEnumConstants().length
}
```

### 16.1 Code example

```java
import java.util.EnumMap;
import java.util.Map;

enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

public class EnumMapDemo {
    public static void main(String[] args) {
        EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
        schedule.put(Day.WED, "Team meeting");
        schedule.put(Day.MON, "Sprint planning");
        schedule.put(Day.FRI, "Retro");

        // Iteration order is ALWAYS the natural enum declaration order (by ordinal), regardless of insertion order:
        System.out.println(schedule); // {MON=Sprint planning, WED=Team meeting, FRI=Retro}
    }
}
```

### 16.2 Why use EnumMap over HashMap<EnumType, V>

| Aspect | EnumMap | HashMap<Enum, V> |
|---|---|---|
| Internal structure | Simple array, indexed by ordinal() | Hash table with buckets |
| Time complexity | O(1) true constant, no hashing/collision overhead at all | O(1) average, but with hashing overhead |
| Memory | Very compact (array sized exactly to enum constant count) | Higher overhead (Node objects, load factor slack) |
| Iteration order | Natural enum order (declaration order) | Unspecified |
| Null keys | Not allowed | 1 allowed |

**Interview tip:** "Always prefer `EnumMap` over `HashMap` when your key type is an enum" — it's faster, more memory-efficient, and gives deterministic ordering, with basically zero downside. This is a common **best-practice** question (also mentioned in Effective Java, Item 37).

---

## 17. ConcurrentSkipListMap

`ConcurrentSkipListMap<K,V>` is a **thread-safe, sorted** map — essentially the concurrent equivalent of `TreeMap`, implementing `ConcurrentNavigableMap` (which extends both `ConcurrentMap` and `NavigableMap`).

### 17.1 Internal structure — Skip List

Instead of a Red-Black Tree (which is hard to rebalance under lock-free concurrent conditions), it uses a **Skip List**: a probabilistic data structure with multiple "levels" of linked lists, where higher levels skip over more elements — like an express lane.

```
Level 3:  HEAD ------------------------> 30 --------------------------> NIL
Level 2:  HEAD ------------> 15 -------> 30 --------------------------> NIL
Level 1:  HEAD ------> 10 -> 15 -------> 30 --------> 40 -------------> NIL
Level 0:  HEAD -> 5 -> 10 -> 15 -> 20 -> 30 -> 35 --> 40 -> 45 -------> NIL
```

- Each node is randomly assigned a "height" (number of levels it participates in) when inserted, typically via coin-flip logic, giving `O(log n)` expected search/insert/delete time.
- Unlike a balanced tree, skip lists **don't need complex rotations to rebalance** — insertion is just linking new nodes at randomly chosen levels, which allows highly efficient **lock-free (CAS-based)** concurrent implementations.

### 17.2 Code example

```java
import java.util.concurrent.ConcurrentSkipListMap;

public class ConcurrentSkipListMapDemo {
    public static void main(String[] args) {
        ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
        map.put(5, "five");
        map.put(1, "one");
        map.put(3, "three");

        System.out.println(map);              // {1=one, 3=three, 5=five} - always sorted
        System.out.println(map.firstKey());    // 1
        System.out.println(map.ceilingKey(2)); // 3
        System.out.println(map.floorKey(4));   // 3

        // Safe for concurrent access WITHOUT explicit synchronization, unlike TreeMap
    }
}
```

### 17.3 ConcurrentSkipListMap vs TreeMap vs ConcurrentHashMap

| Feature | TreeMap | ConcurrentHashMap | ConcurrentSkipListMap |
|---|---|---|---|
| Sorted | Yes | No | Yes |
| Thread-safe | No | Yes | Yes |
| Time complexity | O(log n) | O(1) avg | O(log n) expected |
| Null keys | Not allowed | Not allowed | Not allowed |
| Concurrency mechanism | N/A | CAS + synchronized (bucket-level) | Lock-free CAS (skip list) |

**When to use:** Need a **sorted, thread-safe map** with concurrent access (e.g., a real-time leaderboard, time-ordered event index, or range-query cache shared across threads) — `TreeMap` isn't thread-safe, and `ConcurrentHashMap` isn't sorted, so `ConcurrentSkipListMap` fills that specific gap.

---

## 18. Bonus Topics (Frequently Bundled Into Map Interview Questions)

### 18.1 Fail-Fast vs Fail-Safe Iterators

**Fail-fast** (`HashMap`, `ArrayList`, `TreeMap`, `Hashtable`'s Iterator, etc.):
- Internally tracks a `modCount` (modification counter). The `Iterator` captures `expectedModCount` at creation.
- Every `next()` call checks `modCount == expectedModCount`; if they differ (structural modification happened outside the iterator, e.g. another thread called `put`/`remove`, or you called `map.remove()` directly instead of `iterator.remove()`), it throws `ConcurrentModificationException` (CME).
- **Not a guarantee**, just a "best-effort" bug-detection mechanism — the JavaDoc explicitly states it should not be relied on for correctness, only for catching bugs.

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1); map.put("b", 2);

for (String key : map.keySet()) {
    if (key.equals("a")) {
        map.remove(key); // throws ConcurrentModificationException on next iterator.next()
    }
}

// CORRECT way to remove during iteration:
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<String, Integer> entry = it.next();
    if (entry.getKey().equals("a")) {
        it.remove(); // safe - uses Iterator's own remove(), keeps modCount in sync
    }
}

// ALSO CORRECT (Java 8+):
map.entrySet().removeIf(entry -> entry.getKey().equals("a"));
```

**Fail-safe / weakly consistent** (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentSkipListMap`):
- Never throws CME.
- Iterator reflects the state of the map **at some point during iteration**, may or may not include concurrent modifications made after iterator creation — this is called **"weakly consistent"** (guaranteed not to throw, but not guaranteed to be a perfect snapshot either).

### 18.2 `Map.Entry` interface

```java
public interface Entry<K,V> {
    K getKey();
    V getValue();
    V setValue(V value); // mutates the underlying map entry directly!
}
```
```java
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    if (entry.getKey().equals("a")) {
        entry.setValue(entry.getValue() * 10); // legal - updates map in place, no CME
    }
}
```

### 18.3 Java 8 Functional Map Methods (VERY commonly asked, know these cold)

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);

// getOrDefault - avoid null checks
int val = map.getOrDefault("z", 0); // 0, since "z" absent

// putIfAbsent - insert only if key missing (or mapped to null)
map.putIfAbsent("a", 100); // no-op, "a" already present
map.putIfAbsent("b", 2);   // inserted

// computeIfAbsent - lazy initialization pattern (extremely common, e.g. multi-map building)
Map<String, List<Integer>> multiMap = new HashMap<>();
multiMap.computeIfAbsent("key1", k -> new ArrayList<>()).add(1);
multiMap.computeIfAbsent("key1", k -> new ArrayList<>()).add(2);
System.out.println(multiMap); // {key1=[1, 2]}

// computeIfPresent - update only if key currently exists
map.computeIfPresent("a", (k, v) -> v + 1);

// compute - unconditional read-modify-write; can also remove by returning null
map.compute("a", (k, v) -> (v == null) ? 1 : v + 1);
map.compute("a", (k, v) -> null); // removes the entry!

// merge - combine old and new value, or insert if absent; auto-removes if remapping returns null
map.merge("counter", 1, Integer::sum); // classic word-count pattern

// forEach
map.forEach((k, v) -> System.out.println(k + "->" + v));

// replaceAll
map.replaceAll((k, v) -> v * 2);
```

**Classic word-frequency counter using `merge()`** (asked as a mini coding exercise very often):
```java
String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
Map<String, Integer> freq = new HashMap<>();
for (String w : words) {
    freq.merge(w, 1, Integer::sum);
}
System.out.println(freq); // {banana=2, cherry=1, apple=3}
```

### 18.4 `Comparable` vs `Comparator` (relevant for TreeMap/sorting keys)

```java
// Comparable - "natural ordering", defined INSIDE the class
class Person implements Comparable<Person> {
    int age;
    public int compareTo(Person other) { return Integer.compare(this.age, other.age); }
}

// Comparator - external, allows MULTIPLE different orderings for the same class
Comparator<Person> byAgeDesc = (p1, p2) -> Integer.compare(p2.age, p1.age);
Comparator<Person> byAgeDescLambda = Comparator.comparingInt((Person p) -> p.age).reversed();

TreeMap<Person, String> map = new TreeMap<>(byAgeDesc); // custom order without modifying Person class
```

### 18.5 Capacity, Load Factor, Threshold — Quick Reference

```java
new HashMap<>();                 // capacity=16, loadFactor=0.75, threshold=12
new HashMap<>(64);               // capacity=64 (rounded UP to next power of 2 if not already), loadFactor=0.75
new HashMap<>(64, 0.9f);         // capacity=64, loadFactor=0.9, threshold=57
```
**Gotcha:** If you pass `new HashMap<>(20)`, the actual capacity becomes **32** (next power of 2 ≥ 20) — the constructor's `tableSizeFor()` method rounds up. This is a common trivia/trick question.

```java
// Simplified tableSizeFor logic
static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

### 18.6 HashSet is built on HashMap (great "gotcha" trivia)

```java
// From java.util.HashSet source (simplified)
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object(); // dummy value shared by all entries

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
}
```
So everything you know about `HashMap` internals (hashing, buckets, resizing, treeification) applies directly to `HashSet` — it's literally a `HashMap` where you only ever look at the keys.

### 18.7 Common Interview Q&A Rapid-Fire

**Q: What's the time complexity of HashMap operations?**
A: O(1) average case. Worst case O(n) in Java 7 (long collision chains), reduced to O(log n) in Java 8+ due to treeification of buckets ≥ 8 entries (when table capacity ≥ 64).

**Q: Can a HashMap have a null key? What about ConcurrentHashMap?**
A: HashMap allows exactly ONE null key (stored at bucket index 0, since `hash(null) = 0`) and multiple null values. ConcurrentHashMap, Hashtable, and TreeMap do NOT allow null keys (throw NPE) — for ConcurrentHashMap specifically, this is to avoid the ambiguity between "key absent" and "key mapped to null" in a concurrent context where you can't atomically check-then-read.

**Q: Why is HashMap's default capacity 16 and not, say, 10?**
A: Must be a power of 2 so `(n-1) & hash` can substitute for the more expensive `hash % n` modulo operation, and so the Java 8 resize-splitting trick (checking one extra bit) works cleanly.

**Q: What happens if two objects have the same hashCode but are not equal?**
A: They land in the same bucket (collision) but are stored as separate nodes in that bucket's list/tree — `equals()` distinguishes them during lookup. This is expected and valid behavior, not a bug.

**Q: What happens if you use a mutable object as a HashMap key and then mutate it?**
A: The entry can become permanently "lost" — its current hash no longer matches the bucket it's actually stored in, so `get()`/`remove()` using an equivalent key will fail to locate it. Best practice: use immutable keys.

**Q: Difference between `HashMap` and `ConcurrentHashMap` regarding iteration during modification?**
A: HashMap's iterator is fail-fast (throws CME). ConcurrentHashMap's iterator is weakly consistent (never throws CME, may or may not reflect concurrent changes).

**Q: How do you make a HashMap thread-safe without using ConcurrentHashMap?**
A: `Collections.synchronizedMap(new HashMap<>())` — wraps the map with synchronized methods (coarse locking, similar drawback to Hashtable). Also must manually synchronize when iterating:
```java
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());
synchronized (syncMap) { // REQUIRED for safe iteration, even though individual ops are synchronized
    for (String key : syncMap.keySet()) { ... }
}
```

**Q: Why does `ConcurrentHashMap` not allow null keys/values while `HashMap` does?**
A: In a single-threaded HashMap, `map.get(k) == null` is unambiguous with a follow-up `map.containsKey(k)` check performed safely. In a concurrent map, between the `get()` and `containsKey()` calls, another thread could modify the map — making that check-then-act pattern racy/unsafe. Disallowing null entirely removes the ambiguity by design (per Doug Lea, the author of `java.util.concurrent`).

**Q: What is the difference between `size()` and `mappingCount()` in ConcurrentHashMap?**
A: `size()` returns an `int` (legacy Map interface method, capped at Integer.MAX_VALUE, and is a "best effort" estimate under concurrent modification). `mappingCount()` (added in Java 8) returns a `long`, better suited for maps with more than `Integer.MAX_VALUE` estimated entries, and is the currently recommended API.

**Q: How would you implement your own simple HashMap? (sometimes asked as a live coding exercise)**
```java
public class SimpleHashMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private Entry<K, V>[] table;
    private int size = 0;

    static class Entry<K, V> {
        K key; V value; Entry<K, V> next;
        Entry(K key, V value) { this.key = key; this.value = value; }
    }

    @SuppressWarnings("unchecked")
    public SimpleHashMap() {
        table = new Entry[DEFAULT_CAPACITY];
    }

    private int indexFor(K key) {
        int hash = (key == null) ? 0 : key.hashCode();
        hash ^= (hash >>> 16); // spread bits, same trick as real HashMap
        return (table.length - 1) & hash;
    }

    public void put(K key, V value) {
        int idx = indexFor(key);
        Entry<K, V> current = table[idx];
        while (current != null) {
            if (java.util.Objects.equals(current.key, key)) {
                current.value = value; // update existing
                return;
            }
            current = current.next;
        }
        Entry<K, V> newEntry = new Entry<>(key, value);
        newEntry.next = table[idx]; // head insertion
        table[idx] = newEntry;
        size++;
        if (size > table.length * 0.75) resize();
    }

    public V get(K key) {
        int idx = indexFor(key);
        Entry<K, V> current = table[idx];
        while (current != null) {
            if (java.util.Objects.equals(current.key, key)) return current.value;
            current = current.next;
        }
        return null;
    }

    @SuppressWarnings("unchecked")
    private void resize() {
        Entry<K, V>[] oldTable = table;
        table = new Entry[oldTable.length * 2];
        size = 0;
        for (Entry<K, V> head : oldTable) {
            Entry<K, V> current = head;
            while (current != null) {
                put(current.key, current.value); // re-insert into new table
                current = current.next;
            }
        }
    }
}
```

### 18.8 Complete Time/Space Complexity Comparison Table

| Map Type | get/put avg | get/put worst | Sorted | Thread-safe | Null keys | Null values |
|---|---|---|---|---|---|---|
| HashMap | O(1) | O(log n)* | No | No | 1 | Many |
| LinkedHashMap | O(1) | O(log n)* | Insertion/Access | No | 1 | Many |
| TreeMap | O(log n) | O(log n) | Yes | No | No | Many |
| Hashtable | O(1) | O(n) | No | Yes (coarse) | No | No |
| ConcurrentHashMap | O(1) | O(log n)* | No | Yes (fine-grained) | No | No |
| WeakHashMap | O(1) | O(n) | No | No | 1 | Many |
| IdentityHashMap | O(1) | O(n) | No | No | 1 | Many |
| EnumMap | O(1) true | O(1) true | Enum ordinal order | No | No | Many |
| ConcurrentSkipListMap | O(log n) | O(log n) | Yes | Yes (lock-free) | No | No |

\* O(log n) worst-case only applies post Java-8 treeification (chain ≥ 8 AND capacity ≥ 64); otherwise O(n).

### 18.9 Choosing the Right Map — Decision Guide

```
Need thread-safety?
├── No
│   ├── Need sorting?
│   │   ├── Yes → TreeMap (or EnumMap if keys are enums)
│   │   └── No
│   │       ├── Need insertion/access order or LRU cache? → LinkedHashMap
│   │       ├── Need GC-friendly auto-cleanup (weak refs)? → WeakHashMap
│   │       ├── Need reference-equality keys (rare, special cases)? → IdentityHashMap
│   │       └── Default choice → HashMap
│   └── Immutable/read-only shared config? → Map.of() / Collections.unmodifiableMap()
└── Yes
    ├── Need sorting? → ConcurrentSkipListMap
    └── No → ConcurrentHashMap (never Hashtable in new code)
```

---

## Quick Revision Checklist (night before interview)

- [ ] Can explain `hash()` spreading function and WHY `h ^ (h >>> 16)` is used
- [ ] Can explain `(n-1) & hash` and why capacity must be power of 2
- [ ] Can explain separate chaining vs open addressing, and which classes use which
- [ ] Can explain treeification threshold (8) and untreeify threshold (6), and min capacity (64)
- [ ] Can explain the Java 8 resize trick (`hash & oldCap` splits into lo/hi lists) without recomputing hash
- [ ] Can write correct `hashCode()`/`equals()` for a custom class and explain the contract
- [ ] Can implement LRU cache both via `LinkedHashMap` (removeEldestEntry) and manually (HashMap + DLL)
- [ ] Can explain WeakHashMap's use of WeakReference + ReferenceQueue
- [ ] Can explain IdentityHashMap's open addressing + reference equality
- [ ] Know NavigableMap methods: floor/ceiling/lower/higher, and their difference
- [ ] Can explain TreeMap's Red-Black tree balancing guarantee (O(log n))
- [ ] Can explain Hashtable's coarse locking vs ConcurrentHashMap's fine-grained CAS+synchronized approach (Java 7 segments vs Java 8 bucket-level)
- [ ] Know why ConcurrentHashMap disallows null (ambiguity in concurrent check-then-act)
- [ ] Know atomic compound ops: putIfAbsent, compute, computeIfAbsent, merge — and WHY they matter for concurrency
- [ ] Know the difference between Map.of() (truly immutable) and Collections.unmodifiableMap() (mutable view)
- [ ] Know EnumMap is array-based (ordinal-indexed), not hash-based — faster & more compact than HashMap<Enum,V>
- [ ] Can explain skip list structure and why it enables lock-free concurrency (vs RB-tree rotations)
- [ ] Know fail-fast (modCount, CME) vs fail-safe (weakly consistent) iterators, and which classes use which
- [ ] Remember: HashSet is literally backed by a HashMap internally

---

*End of notes. Good luck with your interviews!*