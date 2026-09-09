# Java Arrays — Interview Notes (3+ YOE Backend Developer)

## 1. Fundamentals

**What is an array?**
A fixed-size, contiguous block of memory holding elements of the *same type*, accessed via a zero-based index.

Key facts interviewers probe for:
- Arrays in Java are **objects** (stored on heap), even for primitives. `int[] arr` is a reference variable.
- Size is fixed at creation time — no resizing. `ArrayList` exists for dynamic sizing.
- Default values on creation: `0`/`0.0` for numeric types, `false` for boolean, `null` for object references.
- Arrays store `length` as a **public final field** (not a method) — `arr.length`, not `arr.length()`. (Strings use `.length()`, collections use `.size()` — a classic gotcha.)
- Array access is **O(1)** because of contiguous memory + index arithmetic: `address = base + index * elementSize`.

```java
int[] a = new int[5];              // declaration + allocation, all zeros
int[] b = {1, 2, 3, 4, 5};         // declaration + initialization
int[] c = new int[]{1, 2, 3};      // explicit form (needed when passing as argument)
String[] names = new String[3];    // all elements null
```

---

## 2. Memory Model & Object Nature

- Single-dimensional array of primitives → contiguous block of actual values.
- Array of objects → contiguous block of **references**; actual objects live elsewhere on the heap.
- **Multi-dimensional arrays in Java are "arrays of arrays"**, not true matrices like C. This means rows can have different lengths (jagged arrays).

```java
int[][] jagged = new int[3][];
jagged[0] = new int[]{1, 2};
jagged[1] = new int[]{1, 2, 3, 4};
jagged[2] = new int[]{5};
```

**Array covariance (common trap question):**
```java
Object[] objArr = new String[3];   // legal - arrays are covariant
objArr[0] = 10;                    // compiles fine, but throws ArrayStoreException at RUNTIME
```
Explanation: Java allows a subtype array to be assigned to a supertype array reference, but the JVM checks the actual runtime type on every store, so putting the wrong type in throws `ArrayStoreException`. This is a favorite "what happens here" question.

---

## 3. Arrays vs ArrayList (very frequently asked)

| Aspect | Array | ArrayList |
|---|---|---|
| Size | Fixed | Dynamic (grows by 1.5x when full) |
| Primitives | Yes, directly | No, needs autoboxing (`Integer`, not `int`) |
| Performance | Faster, no boxing overhead | Slight overhead due to boxing/object wrapper |
| Type safety | Compile-time array-store checks | Uses generics |
| Multi-dimensional | Native support | Needs `List<List<Integer>>` |
| Methods | Only via `java.util.Arrays` utility class | Rich API: add, remove, contains, etc. |

**When to choose which in an interview answer:** fixed-size numeric data / performance-critical code (matrix, buffers) → array. Unknown size, frequent insert/delete → `ArrayList`.

---

## 4. The `java.util.Arrays` Utility Class

```java
int[] arr = {5, 3, 8, 1, 9};

Arrays.sort(arr);                          // in-place, dual-pivot quicksort for primitives - O(n log n)
Arrays.sort(arr, 1, 4);                    // sort sub-range [from, to)
int idx = Arrays.binarySearch(arr, 8);     // O(log n), array MUST be sorted first
int[] copy = Arrays.copyOf(arr, 10);       // new array, padded with 0s
int[] range = Arrays.copyOfRange(arr, 1, 3);
boolean eq = Arrays.equals(arr, copy);     // element-wise comparison
Arrays.fill(arr, 0);                       // fill entire array with value
String s = Arrays.toString(arr);           // "[0, 0, 0, ...]" - for printing
List<Integer> list = Arrays.asList(1,2,3); // FIXED-SIZE list backed by array - .add()/.remove() throw UnsupportedOperationException
```

**Interview trap:** `Arrays.asList()` returns a fixed-size list backed by the array — modifying its structure (add/remove) throws `UnsupportedOperationException`. But `list.set(0, x)` works and even mutates the original array.

**Sorting objects with a Comparator:**
```java
Integer[] nums = {5, 2, 8, 1};
Arrays.sort(nums, Collections.reverseOrder());     // descending
Arrays.sort(nums, (a, b) -> b - a);                // lambda comparator

// Sorting an array of objects by a field
Arrays.sort(employees, Comparator.comparing(Employee::getSalary));
Arrays.sort(employees, Comparator.comparing(Employee::getSalary).reversed()
                                  .thenComparing(Employee::getName));
```
Note: `Arrays.sort()` on an **object array** uses **TimSort** (stable, O(n log n) worst case), while primitive arrays use **dual-pivot quicksort** (not stable, but no stability concern since primitives have no identity beyond value). This distinction ("why is object sort stable but primitive sort isn't") is a genuine 3+ YOE-level question.

---

## 5. `System.arraycopy` vs `Arrays.copyOf` vs `clone()`

```java
// System.arraycopy - fastest, native method, copies into an EXISTING destination array
System.arraycopy(src, 0, dest, 0, src.length);

// Arrays.copyOf - creates a NEW array internally (uses System.arraycopy under the hood)
int[] copy = Arrays.copyOf(src, src.length);

// clone() - shallow copy; for 2D arrays only the outer array is deep-copied, inner arrays are shared references
int[][] matrix = {{1,2},{3,4}};
int[][] shallow = matrix.clone();
shallow[0][0] = 99;   // also changes matrix[0][0] !! because shallow[0] and matrix[0] point to the SAME inner array
```
Deep copy for 2D arrays requires manually cloning each row, or `Arrays.stream(matrix).map(int[]::clone).toArray(int[][]::new)`.

---

## 6. Passing Arrays to Methods

Java is **always pass-by-value**, but for arrays the "value" being passed is the reference (pointer to the heap object).

```java
void modify(int[] arr) {
    arr[0] = 100;        // mutates caller's array - visible outside
    arr = new int[5];    // reassigns local reference only - NOT visible outside
}
```
This is a very common "predict the output" interview question.

---

## 7. Time Complexity Cheat Sheet

| Operation | Array | ArrayList |
|---|---|---|
| Access by index | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) |
| Search (sorted, binary search) | O(log n) | O(log n) via Collections.binarySearch |
| Insert at end | O(1) amortized N/A (fixed size) | O(1) amortized |
| Insert at middle | O(n) (shift elements) | O(n) |
| Delete | O(n) (shift elements) | O(n) |
| Sort | O(n log n) | O(n log n) |

---

## 8. Classic Coding Problems (highest ROI for interviews)

### 8.1 Reverse an array in-place (two-pointer)
```java
void reverse(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left++;
        right--;
    }
}
```
Time: O(n), Space: O(1).

### 8.2 Find the missing number (1 to n)
```java
int findMissing(int[] arr, int n) {
    int expectedSum = n * (n + 1) / 2;
    int actualSum = 0;
    for (int x : arr) actualSum += x;
    return expectedSum - actualSum;
}
```
Alternative (avoids overflow on large n): XOR all numbers 1..n with all array elements — pairs cancel out, missing number remains.

### 8.3 Find duplicate(s) — using a HashSet
```java
int findDuplicate(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    for (int x : arr) {
        if (!seen.add(x)) return x;   // add() returns false if already present
    }
    return -1;
}
```
Time O(n), Space O(n). Follow-up: "can you do it in O(1) space?" → **Floyd's cycle detection** (treat array as a linked list via index-as-pointer, works when values are in range [1, n]).

### 8.4 Two Sum (classic — always asked)
```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>(); // value -> index
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i);
    }
    throw new IllegalArgumentException("No solution");
}
```
Time O(n), Space O(n) — beats the naive O(n²) brute force.

### 8.5 Kadane's Algorithm — Maximum Subarray Sum
```java
int maxSubArray(int[] nums) {
    int maxSoFar = nums[0], maxEndingHere = nums[0];
    for (int i = 1; i < nums.length; i++) {
        maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
        maxSoFar = Math.max(maxSoFar, maxEndingHere);
    }
    return maxSoFar;
}
```
O(n) time, O(1) space. Very commonly asked to test DP intuition.

### 8.6 Rotate array by k positions (in-place, O(1) space)
```java
void rotate(int[] nums, int k) {
    k %= nums.length;
    reverse(nums, 0, nums.length - 1);
    reverse(nums, 0, k - 1);
    reverse(nums, k, nums.length - 1);
}

void reverse(int[] arr, int left, int right) {
    while (left < right) {
        int t = arr[left]; arr[left] = arr[right]; arr[right] = t;
        left++; right--;
    }
}
```
Trick: three reversals achieve rotation without extra array. Time O(n), Space O(1).

### 8.7 Merge two sorted arrays (in-place, e.g. LeetCode 88 style)
```java
void merge(int[] nums1, int m, int[] nums2, int n) {
    int i = m - 1, j = n - 1, k = m + n - 1;
    while (j >= 0) {
        if (i >= 0 && nums1[i] > nums2[j]) {
            nums1[k--] = nums1[i--];
        } else {
            nums1[k--] = nums2[j--];
        }
    }
}
```
Merging from the **back** avoids overwriting unread elements in `nums1`. O(m+n) time, O(1) extra space.

### 8.8 Sliding Window — Maximum sum subarray of size k
```java
int maxSumSubarray(int[] arr, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i];
    int maxSum = windowSum;
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k];   // slide: add new, remove old
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```
Reduces O(n·k) brute force to O(n).

### 8.9 Prefix Sum — Range sum queries
```java
class PrefixSum {
    int[] prefix;
    PrefixSum(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++)
            prefix[i + 1] = prefix[i] + nums[i];
    }
    int rangeSum(int left, int right) {   // inclusive
        return prefix[right + 1] - prefix[left];
    }
}
```
O(n) preprocessing, O(1) per query — good answer when asked to optimize repeated range-sum queries.

### 8.10 Binary search (know it cold, including the overflow bug)
```java
int binarySearch(int[] arr, int target) {
    int low = 0, high = arr.length - 1;
    while (low <= high) {
        int mid = low + (high - low) / 2;   // NOT (low+high)/2 -> avoids integer overflow
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) low = mid + 1;
        else high = mid - 1;
    }
    return -1;
}
```

---

## 9. Frequently Asked Conceptual Q&A

**Q: Why does `arr.length` not have parentheses while `String.length()` does?**
A: `length` on an array is a public final field set at creation; `length()` on `String` is a method because `String` is backed by an internal char/byte array whose size can be computed/cached but the API was designed as a class method for encapsulation consistency with other collection-like methods.

**Q: Can you store `null` in a primitive array?**
A: No — primitive arrays (`int[]`, `boolean[]`, etc.) can't hold `null`, only object arrays (`Integer[]`, `String[]`) can.

**Q: How is a 2D array stored in memory?**
A: As an array of references, each pointing to a separate 1D array (row) which can even have different lengths — not a single contiguous 2D block like in C/C++.

**Q: Is `Arrays.sort()` stable?**
A: For objects — yes (TimSort). For primitives — the sort is not "stable" in the meaningful sense since primitives have no separate identity from their value.

**Q: Difference between shallow copy and deep copy for arrays of objects?**
A: `clone()`/`Arrays.copyOf()` on an object array copies only the references — the underlying objects are shared. Mutating a shared object's internal state affects both arrays. True deep copy requires copying each object explicitly (or via copy constructors/serialization).

**Q: How would you make an array thread-safe?**
A: Arrays themselves have no built-in synchronization. Options: wrap access with explicit locks, use `java.util.concurrent.atomic.AtomicIntegerArray` / `AtomicLongArray` / `AtomicReferenceArray` for lock-free thread-safe element updates, or copy-on-write patterns for read-heavy scenarios.

**Q: What's the time complexity to insert an element at the beginning of an array?**
A: O(n) — since arrays are fixed size and contiguous, you must create a new array (or shift) and move every existing element over by one.

---

## 10. Quick Tips for the Interview

1. Always clarify: **is the array sorted?** — determines if binary search / two-pointer is applicable.
2. State time & space complexity explicitly after every solution — interviewers expect it unprompted at 3+ YOE.
3. For "optimize this" follow-ups, the usual progression is: brute force O(n²) → HashMap O(n) time/O(n) space → two-pointer/sliding-window O(n) time/O(1) space.
4. Mention **edge cases** out loud: empty array, single element, all duplicates, negative numbers, integer overflow.
5. If discussing a matrix problem, clarify row-major vs column-major traversal and whether it's a jagged array.
6. Know when interviewers are testing **language depth** (arraycopy vs clone, covariance, pass-by-value) vs **algorithmic depth** (two-pointer, sliding window, prefix sum) — answer accordingly.