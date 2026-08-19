# Java String — Interview Ready Notes (3+ YOE)

---

## 1. String Basics & Immutability

`String` in Java is **immutable** — once created, its internal character data can never change. Every "modification" method (`concat`, `substring`, `replace`, etc.) returns a **new** String object.

```java
String s = "hello";
s.concat(" world");
System.out.println(s); // "hello" — unchanged!

s = s.concat(" world"); // new object assigned
System.out.println(s); // "hello world"
```

### Why is String immutable? (Very common question)

1. **String Pool / Caching** — Immutability allows the JVM to safely reuse String literals via the String Constant Pool without worrying about one reference changing the value for all others.
2. **Thread Safety** — Immutable objects are inherently thread-safe; no synchronization needed when sharing across threads.
3. **Security** — Strings are used for class names in `Class.forName()`, file paths, network connections, DB URLs. If String were mutable, someone could change the value after a security check but before use (TOCTOU attack).
4. **HashCode Caching** — Since the value can't change, `String` caches its hashCode after first computation, making it very efficient as a `HashMap` key.
5. **Class Loading** — `Class.forName(className)` relies on String immutability so the loader isn't tricked mid-operation.

### Internal implementation (Java 9+ change — good to mention)

- **Before Java 9**: `String` stored data as `char[]` (2 bytes/char, UTF-16).
- **Java 9+ (JEP 254 — Compact Strings)**: `String` stores data as `byte[]` + a coder flag (LATIN1 or UTF16). If all characters fit in Latin-1 (1 byte), it uses 1 byte/char — saving significant memory since most strings are ASCII.

```java
private final byte[] value;   // Java 9+
private final byte coder;     // LATIN1 (0) or UTF16 (1)
```

---

## 2. String Pool (String Intern Pool)

The String pool is a special memory region inside the **Heap** (moved from PermGen to Heap in Java 7) that stores unique String literals.

```java
String s1 = "hello";          // goes into String pool
String s2 = "hello";          // reuses same pool reference
String s3 = new String("hello"); // creates NEW object on heap (NOT pool)
String s4 = s3.intern();      // forces JVM to return the pooled reference

System.out.println(s1 == s2); // true  — same pool reference
System.out.println(s1 == s3); // false — s3 is a separate heap object
System.out.println(s1 == s4); // true  — intern() returned pool reference
```

**Diagram (mental model):**
```
Heap:
 ├── String Pool (subset of heap since Java 7)
 │     "hello"  <── s1, s2, s4 point here
 └── Regular heap object
       "hello"  <── s3 (new String()) points here
```

---

## 3. `==` vs `.equals()` vs `.equalsIgnoreCase()`

| Operator/Method | Compares | Notes |
|---|---|---|
| `==` | Reference (memory address) | Use only for pool-literal comparisons or null checks |
| `.equals()` | Content (case-sensitive) | Overridden in String class to compare char-by-char |
| `.equalsIgnoreCase()` | Content (case-insensitive) | |
| `.compareTo()` | Lexicographic order | Returns int (diff of char codes at first mismatch), used in sorting |

```java
String a = "test";
String b = new String("test");

a == b;              // false
a.equals(b);          // true
a.compareTo(b);        // 0
```

**Interview trap:** Never use `==` to compare Strings in business logic — always use `.equals()`. `==` "accidentally" works with literals due to pooling, which is a classic trap in coding interviews.

---

## 4. String vs StringBuilder vs StringBuffer

| Feature | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (immutable) | No | Yes (synchronized methods) |
| Performance | Slow for concatenation in loops | Fast | Slower than StringBuilder (sync overhead) |
| Introduced | JDK 1.0 | JDK 1.5 | JDK 1.0 |
| Use case | Fixed/rarely-changed text | Single-threaded heavy string building | Multi-threaded string building |

```java
// Bad — creates N new String objects (O(n^2) time complexity)
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;   // each += creates a new String
}

// Good — mutable buffer, amortized O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result2 = sb.toString();
```

**Why is `+=` in a loop bad?** Under the hood, javac actually compiles simple `+` concatenation using `StringBuilder` (or `invokedynamic` + `StringConcatFactory` since Java 9), BUT in a loop, a **new StringBuilder is created on every iteration**, so the optimization doesn't help — you still get O(n²) behavior.

```java
// What the compiler roughly generates for: String s = a + b;
String s = new StringBuilder().append(a).append(b).toString();
```

**StringBuilder key methods:**
```java
StringBuilder sb = new StringBuilder("hello");
sb.append(" world");     // hello world
sb.insert(5, ",");        // hello, world
sb.reverse();              // dlrow ,olleh
sb.deleteCharAt(0);        // remove index 0
sb.replace(0, 2, "XY");    // replace range
sb.setLength(0);           // clear the buffer (common reset trick)
sb.charAt(0);
sb.length();
sb.capacity();              // internal buffer size (default 16 + string)
```

---

## 5. hashCode() and Immutability Synergy

```java
@Override
public int hashCode() {
    int h = hash; // cached
    if (h == 0 && value.length > 0) {
        for (char c : value) {
            h = 31 * h + c;
        }
        hash = h; // cache it — safe because String is immutable
    }
    return h;
}
```

This is why `String` is a great `HashMap`/`HashSet` key — hashCode is computed once and cached forever.

**Why 31?** It's an odd prime. Odd because multiplying by an even number and overflowing loses information (equivalent to a bit shift with data loss). Prime helps distribute hash values more uniformly, reducing collisions. Also, `31 * i == (i << 5) - i`, which the JVM can optimize.

---

## 6. Common String Methods Cheat Sheet

```java
String s = "  Hello World  ";

s.trim();                  // "Hello World" — removes leading/trailing whitespace (ASCII <= ' ')
s.strip();                 // Java 11+, Unicode-aware whitespace trim
s.length();                // 15
s.toUpperCase();            // "  HELLO WORLD  "
s.toLowerCase();
s.charAt(2);                 // 'H'
s.indexOf("World");          // position of first match, -1 if absent
s.substring(2, 7);           // "Hello" (start inclusive, end exclusive)
s.split("\\s+");             // regex split, e.g. ["Hello", "World"]
s.replace('l', 'L');         // char replace
s.replaceAll("l+", "L");     // regex replace
s.contains("World");          // boolean
s.startsWith("He");
s.endsWith("ld  ");
s.isEmpty();                  // length == 0
s.isBlank();                  // Java 11+, true if only whitespace
String.join("-", "a", "b");   // "a-b"
String.valueOf(123);          // "123"
"abc".repeat(3);               // Java 11+, "abcabcabc"
"  a  ".chars();                // IntStream of char codes (Java 8+)
```

---

## 7. Immutable Records & Text Blocks (Java 15+/17+ — recent additions worth knowing)

```java
// Text Block (Java 15+) — great for JSON/SQL in code
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;
```

---

## 8. Common Coding Interview Problems (High Frequency)

### a) Reverse a String
```java
public static String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}
// Manual, without built-in reverse (common follow-up):
public static String reverseManual(String s) {
    char[] chars = s.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left++] = chars[right];
        chars[right--] = temp;
    }
    return new String(chars);
}
```

### b) Check Palindrome
```java
public static boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left++) != s.charAt(right--)) return false;
    }
    return true;
}
```

### c) Check Anagram
```java
public static boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] freq = new int[26];
    for (char c : a.toCharArray()) freq[c - 'a']++;
    for (char c : b.toCharArray()) freq[c - 'a']--;
    for (int f : freq) if (f != 0) return false;
    return true;
}
```

### d) First Non-Repeating Character
```java
public static char firstNonRepeating(String s) {
    Map<Character, Integer> countMap = new LinkedHashMap<>();
    for (char c : s.toCharArray()) {
        countMap.merge(c, 1, Integer::sum);
    }
    for (Map.Entry<Character, Integer> e : countMap.entrySet()) {
        if (e.getValue() == 1) return e.getKey();
    }
    return '-';
}
```

### e) Count Occurrences of Each Character
```java
public static Map<Character, Long> charFrequency(String s) {
    return s.chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(c -> c, Collectors.counting()));
}
```

### f) Longest Substring Without Repeating Characters (sliding window — very popular)
```java
public static int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int maxLen = 0, start = 0;
    for (int end = 0; end < s.length(); end++) {
        char c = s.charAt(end);
        if (lastIndex.containsKey(c) && lastIndex.get(c) >= start) {
            start = lastIndex.get(c) + 1;
        }
        lastIndex.put(c, end);
        maxLen = Math.max(maxLen, end - start + 1);
    }
    return maxLen;
}
```

### g) String Compression (Run-Length Encoding)
```java
public static String compress(String s) {
    StringBuilder sb = new StringBuilder();
    int count = 1;
    for (int i = 1; i <= s.length(); i++) {
        if (i < s.length() && s.charAt(i) == s.charAt(i - 1)) {
            count++;
        } else {
            sb.append(s.charAt(i - 1)).append(count);
            count = 1;
        }
    }
    return sb.length() < s.length() ? sb.toString() : s;
}
```

---

## 9. Performance & Memory Considerations (Backend-specific, senior-level talking points)

- **Avoid `+` concatenation in loops** — use `StringBuilder`.
- **Use `String.format` or text blocks sparingly in hot paths** — formatting has overhead; prefer StringBuilder for performance-critical code.
- **`substring()` (Java 7+)**: No longer shares the backing char array with the original string (this was changed to fix memory leak issues where a small substring kept a huge original array alive). Pre-Java 7, `substring()` could cause memory leaks.
- **Use `intern()` carefully**: Interning large numbers of unique dynamic strings can bloat the String pool and cause `OutOfMemoryError` in older JVMs (pre-Java 8, when pool was in PermGen with fixed size). Since Java 7+/8, pool is in heap, but still not free — avoid interning unless you have genuine duplicate-heavy data (e.g., deduplicating large datasets).
- **StringBuilder pre-sizing**: `new StringBuilder(expectedCapacity)` avoids repeated internal array resizing (default capacity is 16, grows by `(oldCapacity * 2) + 2`).
- **Switch on Strings**: Internally uses `hashCode()` + `equals()` (compiled to a hash-based lookup, not literal `==`), so it's O(1) average, not a chain of `if-else`.

```java
switch (input) {
    case "GET": handleGet(); break;
    case "POST": handlePost(); break;
    default: handleDefault();
}
```

---

## 10. Tricky Interview Questions & Answers

**Q: How many objects are created here?**
```java
String s = new String("hello");
```
**A:** Up to 2 — one in the String pool (if `"hello"` literal doesn't already exist there) and one on the heap via `new`. If `"hello"` is already in the pool (common case), only 1 new object is created (the heap object); the pool reference is reused, not recreated.

**Q: Is String thread-safe?**
**A:** Yes, because it's immutable — no state can change after construction, so concurrent reads from multiple threads are inherently safe.

**Q: Why does String override `equals()` and `hashCode()` but StringBuilder doesn't?**
**A:** `StringBuilder` is mutable — if it overrode `equals()`/`hashCode()` based on content, using it as a HashMap key would break the hash contract when its content changes after insertion. So it deliberately uses `Object`'s identity-based equals/hashCode.

**Q: What's the time complexity of String concatenation `a + b`?**
**A:** O(n + m) for a single operation (creates new array of combined size), but O(n²) if done repeatedly in a loop with `n` iterations, because each iteration allocates and copies the whole growing string.

**Q: Difference between `String.format()` and concatenation?**
**A:** `String.format()` is more readable for complex formatting (padding, locale, number formatting) but has more overhead (parses format string via regex-like state machine at runtime) — avoid in tight loops.

**Q: Can you use String as a key in HashMap? Why is it a good choice?**
**A:** Yes — immutability guarantees the hashCode won't change after insertion (which would break the HashMap's bucket placement), and hashCode is cached, making lookups fast.

---

## 11. Quick-Fire Revision Table

| Concept | One-liner |
|---|---|
| Immutability | Value can't change after creation; new object created for "modifications" |
| String Pool | Cache of literals in heap (Java 7+) to save memory via reuse |
| `intern()` | Manually places/retrieves a String from the pool |
| `==` vs `.equals()` | Reference vs content comparison |
| StringBuilder | Mutable, fast, not thread-safe |
| StringBuffer | Mutable, thread-safe (synchronized), slower |
| hashCode caching | Computed once, safe due to immutability |
| Compact Strings (Java 9+) | byte[] storage instead of char[], saves memory for Latin-1 strings |
| substring() fix (Java 7+) | No longer shares backing array — prevents memory leaks |

---

### Suggested Follow-Up Topics to Prep Next
Collections (HashMap internals), Multithreading/Concurrency, JVM memory model, Streams API, Exception handling, Spring Boot core annotations.