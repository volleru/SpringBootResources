# Java Coding Problems — Most Asked at TCS, Infosys & Wipro

> A curated set of the coding problems most frequently asked in service-company
> interviews (TCS NQT / Ninja / Digital, Infosys SP/DSE, Wipro Turbo/Elite).
> Each problem has the **problem statement**, a **clean Java solution**, the
> **complexity**, and the **key idea** the interviewer is testing.
>
> Tip for interviews: always state your approach, time/space complexity, and
> edge cases *before* you start coding.

---

## Table of Contents

1. [Strings](#1-strings)
2. [Numbers & Math](#2-numbers--math)
3. [Arrays](#3-arrays)
4. [Searching & Sorting](#4-searching--sorting)
5. [Recursion](#5-recursion)
6. [Patterns (Star / Number)](#6-patterns)
7. [Collections / HashMap](#7-collections--hashmap)
8. [Java 8 Streams (commonly asked now)](#8-java-8-streams)
9. [Interview Tips & Common Theory Q&A](#9-interview-tips)

---

## 1. Strings

### 1.1 Reverse a String (without using `reverse()`)

**Asked at:** TCS, Infosys, Wipro (very common)

```java
public class ReverseString {
    public static String reverse(String s) {
        char[] arr = s.toCharArray();
        int left = 0, right = arr.length - 1;
        while (left < right) {
            char temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
            left++;
            right--;
        }
        return new String(arr);
    }

    public static void main(String[] args) {
        System.out.println(reverse("hello")); // olleh
    }
}
```
- **Time:** O(n) · **Space:** O(n)
- **Key idea:** two-pointer swap. Interviewers often forbid `StringBuilder.reverse()`.

---

### 1.2 Check if a String is a Palindrome

```java
public static boolean isPalindrome(String s) {
    int i = 0, j = s.length() - 1;
    while (i < j) {
        if (s.charAt(i) != s.charAt(j)) {
            return false;
        }
        i++;
        j--;
    }
    return true;
}
// isPalindrome("madam") -> true
```
- **Time:** O(n) · **Space:** O(1)

---

### 1.3 Count Vowels and Consonants

```java
public static void countVowelsConsonants(String s) {
    s = s.toLowerCase();
    int vowels = 0, consonants = 0;
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c >= 'a' && c <= 'z') {
            if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
                vowels++;
            } else {
                consonants++;
            }
        }
    }
    System.out.println("Vowels: " + vowels + ", Consonants: " + consonants);
}
```
- **Time:** O(n)

---

### 1.4 Find Duplicate Characters in a String

```java
import java.util.*;

public static void findDuplicates(String s) {
    Map<Character, Integer> count = new HashMap<>();
    for (char c : s.toCharArray()) {
        count.put(c, count.getOrDefault(c, 0) + 1);
    }
    for (Map.Entry<Character, Integer> e : count.entrySet()) {
        if (e.getValue() > 1) {
            System.out.println(e.getKey() + " -> " + e.getValue());
        }
    }
}
// "programming" -> r->2, g->2, m->2
```
- **Time:** O(n) · **Space:** O(k) for distinct chars

---

### 1.5 First Non-Repeating Character

```java
import java.util.*;

public static Character firstNonRepeating(String s) {
    Map<Character, Integer> count = new LinkedHashMap<>(); // preserves order
    for (char c : s.toCharArray()) {
        count.put(c, count.getOrDefault(c, 0) + 1);
    }
    for (Map.Entry<Character, Integer> e : count.entrySet()) {
        if (e.getValue() == 1) {
            return e.getKey();
        }
    }
    return null;
}
// "swiss" -> 'w'
```
- **Key idea:** `LinkedHashMap` keeps insertion order so you return the *first* one.

---

### 1.6 Check if Two Strings are Anagrams

```java
import java.util.Arrays;

public static boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) {
        return false;
    }
    char[] x = a.toCharArray();
    char[] y = b.toCharArray();
    Arrays.sort(x);
    Arrays.sort(y);
    return Arrays.equals(x, y);
}
// "listen", "silent" -> true
```
- **Time:** O(n log n). Can be O(n) with a 26-size count array.

---

### 1.7 Reverse Words in a Sentence

```java
public static String reverseWords(String sentence) {
    String[] words = sentence.trim().split("\\s+");
    StringBuilder sb = new StringBuilder();
    for (int i = words.length - 1; i >= 0; i--) {
        sb.append(words[i]);
        if (i > 0) {
            sb.append(" ");
        }
    }
    return sb.toString();
}
// "Java is fun" -> "fun is Java"
```

---

### 1.8 Remove All White Spaces / Count Word Frequency

```java
// Count occurrences of each word
import java.util.*;

public static Map<String, Integer> wordFrequency(String text) {
    Map<String, Integer> freq = new HashMap<>();
    for (String word : text.toLowerCase().split("\\s+")) {
        freq.put(word, freq.getOrDefault(word, 0) + 1);
    }
    return freq;
}
```

---

## 2. Numbers & Math

### 2.1 Check if a Number is Prime

```java
public static boolean isPrime(int n) {
    if (n <= 1) {
        return false;
    }
    for (int i = 2; i <= Math.sqrt(n); i++) {
        if (n % i == 0) {
            return false;
        }
    }
    return true;
}
```
- **Time:** O(√n) — interviewers love when you optimize from O(n) to O(√n).

---

### 2.2 Fibonacci Series

```java
// Iterative (preferred)
public static void fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 0; i < n; i++) {
        System.out.print(a + " ");
        int next = a + b;
        a = b;
        b = next;
    }
}
// 0 1 1 2 3 5 8 13 ...
```
- **Time:** O(n) · **Space:** O(1)

---

### 2.3 Factorial

```java
public static long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}
```

---

### 2.4 Check Armstrong Number

> A 3-digit number equal to the sum of cubes of its digits, e.g. 153 = 1³+5³+3³.

```java
public static boolean isArmstrong(int num) {
    int original = num, sum = 0;
    int digits = String.valueOf(num).length();
    while (num > 0) {
        int d = num % 10;
        sum += Math.pow(d, digits);
        num /= 10;
    }
    return sum == original;
}
// 153 -> true, 9474 -> true
```

---

### 2.5 Reverse an Integer

```java
public static int reverseNumber(int n) {
    int rev = 0;
    while (n != 0) {
        rev = rev * 10 + n % 10;
        n /= 10;
    }
    return rev;
}
// 1234 -> 4321
```

---

### 2.6 Sum of Digits

```java
public static int sumOfDigits(int n) {
    int sum = 0;
    n = Math.abs(n);
    while (n > 0) {
        sum += n % 10;
        n /= 10;
    }
    return sum;
}
```

---

### 2.7 Check Palindrome Number

```java
public static boolean isPalindromeNumber(int n) {
    int original = n, rev = 0;
    while (n > 0) {
        rev = rev * 10 + n % 10;
        n /= 10;
    }
    return original == rev;
}
// 121 -> true
```

---

### 2.8 GCD and LCM

```java
public static int gcd(int a, int b) {
    while (b != 0) {
        int t = b;
        b = a % b;
        a = t;
    }
    return a;
}

public static int lcm(int a, int b) {
    return (a * b) / gcd(a, b);
}
```
- **Key idea:** Euclidean algorithm. `lcm = a*b / gcd`.

---

### 2.9 Count Number of Digits

```java
public static int countDigits(int n) {
    if (n == 0) {
        return 1;
    }
    int count = 0;
    n = Math.abs(n);
    while (n > 0) {
        count++;
        n /= 10;
    }
    return count;
}
```

---

## 3. Arrays

### 3.1 Find Largest and Smallest Element

```java
public static void minMax(int[] arr) {
    int min = arr[0], max = arr[0];
    for (int x : arr) {
        if (x < min) {
            min = x;
        }
        if (x > max) {
            max = x;
        }
    }
    System.out.println("Min: " + min + ", Max: " + max);
}
```
- **Time:** O(n)

---

### 3.2 Find Second Largest Element

```java
public static int secondLargest(int[] arr) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int x : arr) {
        if (x > first) {
            second = first;
            first = x;
        } else if (x > second && x != first) {
            second = x;
        }
    }
    return second;
}
```
- **Key idea:** single pass, track top two. Avoid sorting (O(n log n)).

---

### 3.3 Reverse an Array

```java
public static void reverse(int[] arr) {
    int i = 0, j = arr.length - 1;
    while (i < j) {
        int t = arr[i];
        arr[i] = arr[j];
        arr[j] = t;
        i++;
        j--;
    }
}
```

---

### 3.4 Remove Duplicates from an Array

```java
import java.util.*;

public static int[] removeDuplicates(int[] arr) {
    LinkedHashSet<Integer> set = new LinkedHashSet<>();
    for (int x : arr) {
        set.add(x);
    }
    int[] result = new int[set.size()];
    int i = 0;
    for (int x : set) {
        result[i++] = x;
    }
    return result;
}
```

---

### 3.5 Find Missing Number (1 to N)

```java
public static int missingNumber(int[] arr, int n) {
    int expected = n * (n + 1) / 2;
    int actual = 0;
    for (int x : arr) {
        actual += x;
    }
    return expected - actual;
}
// arr = {1,2,4,5}, n=5 -> 3
```
- **Key idea:** sum formula trick — O(n) time, O(1) space.

---

### 3.6 Find Pairs with a Given Sum

```java
import java.util.*;

public static void pairSum(int[] arr, int target) {
    Set<Integer> seen = new HashSet<>();
    for (int x : arr) {
        int complement = target - x;
        if (seen.contains(complement)) {
            System.out.println(complement + " + " + x + " = " + target);
        }
        seen.add(x);
    }
}
```
- **Time:** O(n) using HashSet (vs O(n²) brute force).

---

### 3.7 Move All Zeros to End

```java
public static void moveZeros(int[] arr) {
    int index = 0;
    for (int x : arr) {
        if (x != 0) {
            arr[index++] = x;
        }
    }
    while (index < arr.length) {
        arr[index++] = 0;
    }
}
// {0,1,0,3,12} -> {1,3,12,0,0}
```

---

### 3.8 Find Frequency / Maximum Occurring Element

```java
import java.util.*;

public static int mostFrequent(int[] arr) {
    Map<Integer, Integer> count = new HashMap<>();
    int maxCount = 0, result = arr[0];
    for (int x : arr) {
        int c = count.getOrDefault(x, 0) + 1;
        count.put(x, c);
        if (c > maxCount) {
            maxCount = c;
            result = x;
        }
    }
    return result;
}
```

---

### 3.9 Rotate Array by K Positions

```java
public static void rotate(int[] arr, int k) {
    int n = arr.length;
    k = k % n;
    reverse(arr, 0, n - 1);
    reverse(arr, 0, k - 1);
    reverse(arr, k, n - 1);
}

private static void reverse(int[] arr, int i, int j) {
    while (i < j) {
        int t = arr[i];
        arr[i] = arr[j];
        arr[j] = t;
        i++;
        j--;
    }
}
// {1,2,3,4,5}, k=2 -> {4,5,1,2,3}
```
- **Key idea:** reversal algorithm, O(n) time, O(1) space.

---

## 4. Searching & Sorting

### 4.1 Binary Search

```java
public static int binarySearch(int[] arr, int target) {
    int low = 0, high = arr.length - 1;
    while (low <= high) {
        int mid = low + (high - low) / 2; // avoids overflow
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return -1;
}
```
- **Time:** O(log n). Array must be sorted. Note `low + (high-low)/2` to avoid integer overflow.

---

### 4.2 Bubble Sort

```java
public static void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                int t = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = t;
                swapped = true;
            }
        }
        if (!swapped) {
            break; // already sorted
        }
    }
}
```
- **Time:** O(n²), best O(n) with the `swapped` flag.

---

### 4.3 Selection Sort

```java
public static void selectionSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) {
                minIdx = j;
            }
        }
        int t = arr[minIdx];
        arr[minIdx] = arr[i];
        arr[i] = t;
    }
}
```

---

### 4.4 Insertion Sort

```java
public static void insertionSort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int key = arr[i];
        int j = i - 1;
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}
```

---

### 4.5 Quick Sort (asked in Wipro/Infosys advanced rounds)

```java
public static void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}

private static int partition(int[] arr, int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    for (int j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            int t = arr[i];
            arr[i] = arr[j];
            arr[j] = t;
        }
    }
    int t = arr[i + 1];
    arr[i + 1] = arr[high];
    arr[high] = t;
    return i + 1;
}
```
- **Time:** avg O(n log n), worst O(n²).

---

## 5. Recursion

### 5.1 Factorial (Recursive)

```java
public static long factorial(int n) {
    if (n <= 1) {
        return 1;
    }
    return n * factorial(n - 1);
}
```

### 5.2 Fibonacci (Recursive)

```java
public static int fib(int n) {
    if (n <= 1) {
        return n;
    }
    return fib(n - 1) + fib(n - 2);
}
```
- **Note:** O(2ⁿ) — mention you'd memoize for efficiency.

### 5.3 Sum of N Natural Numbers

```java
public static int sum(int n) {
    if (n == 0) {
        return 0;
    }
    return n + sum(n - 1);
}
```

### 5.4 Tower of Hanoi

```java
public static void hanoi(int n, char from, char aux, char to) {
    if (n == 1) {
        System.out.println("Move disk 1 from " + from + " to " + to);
        return;
    }
    hanoi(n - 1, from, to, aux);
    System.out.println("Move disk " + n + " from " + from + " to " + to);
    hanoi(n - 1, aux, from, to);
}
```

---

## 6. Patterns

> Pattern programs are extremely common in TCS/Wipro freshers' rounds.

### 6.1 Right-Angled Triangle of Stars

```java
public static void starTriangle(int n) {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= i; j++) {
            System.out.print("* ");
        }
        System.out.println();
    }
}
/*
*
* *
* * *
* * * *
*/
```

### 6.2 Pyramid Pattern

```java
public static void pyramid(int n) {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n - i; j++) {
            System.out.print(" ");
        }
        for (int k = 1; k <= 2 * i - 1; k++) {
            System.out.print("*");
        }
        System.out.println();
    }
}
/*
   *
  ***
 *****
*******
*/
```

### 6.3 Number Pyramid / Floyd's Triangle

```java
public static void floyds(int n) {
    int num = 1;
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= i; j++) {
            System.out.print(num++ + " ");
        }
        System.out.println();
    }
}
/*
1
2 3
4 5 6
7 8 9 10
*/
```

### 6.4 Pascal's Triangle

```java
public static void pascal(int n) {
    for (int i = 0; i < n; i++) {
        int num = 1;
        for (int j = 0; j <= i; j++) {
            System.out.print(num + " ");
            num = num * (i - j) / (j + 1);
        }
        System.out.println();
    }
}
```

---

## 7. Collections / HashMap

### 7.1 Sort a HashMap by Value

```java
import java.util.*;

public static void sortByValue(Map<String, Integer> map) {
    List<Map.Entry<String, Integer>> list = new ArrayList<>(map.entrySet());
    list.sort(Map.Entry.comparingByValue()); // ascending
    for (Map.Entry<String, Integer> e : list) {
        System.out.println(e.getKey() + " = " + e.getValue());
    }
}
```

### 7.2 Find Duplicate Elements in a List

```java
import java.util.*;

public static Set<Integer> findDuplicates(List<Integer> list) {
    Set<Integer> seen = new HashSet<>();
    Set<Integer> dups = new HashSet<>();
    for (int x : list) {
        if (!seen.add(x)) {
            dups.add(x);
        }
    }
    return dups;
}
```
- **Key idea:** `Set.add()` returns `false` if the element already exists.

### 7.3 Remove Duplicates from a List Preserving Order

```java
import java.util.*;

public static List<Integer> dedupe(List<Integer> list) {
    return new ArrayList<>(new LinkedHashSet<>(list));
}
```

### 7.4 Difference: HashMap vs Hashtable vs ConcurrentHashMap

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Thread-safe | No | Yes (synchronized) | Yes (segment/bucket locks) |
| Null key/values | 1 null key, many null values | None allowed | None allowed |
| Performance | Fast | Slow (full lock) | Fast (fine-grained lock) |
| Introduced | 1.2 | 1.0 (legacy) | 1.5 |

---

## 8. Java 8 Streams

> Increasingly asked even at service companies. Know these idioms.

```java
import java.util.*;
import java.util.stream.*;

List<Integer> nums = Arrays.asList(5, 2, 8, 1, 9, 3);

// 1. Filter even numbers
List<Integer> evens = nums.stream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());

// 2. Sum of all elements
int sum = nums.stream().mapToInt(Integer::intValue).sum();

// 3. Find max
Optional<Integer> max = nums.stream().max(Integer::compareTo);

// 4. Sort descending
List<Integer> sorted = nums.stream()
        .sorted(Comparator.reverseOrder())
        .collect(Collectors.toList());

// 5. Count word frequency
Map<String, Long> freq = Stream.of("a", "b", "a", "c", "b", "a")
        .collect(Collectors.groupingBy(w -> w, Collectors.counting()));

// 6. Find first non-repeating char using streams
String s = "swiss";
Character firstNonRep = s.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(c -> c, LinkedHashMap::new, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey)
        .findFirst()
        .orElse(null);

// 7. Second highest number
int secondMax = nums.stream()
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst()
        .orElse(-1);
```

---

## 9. Interview Tips

### Frequently Asked Theory (Quick Answers)

**Q: Difference between `==` and `.equals()`?**
`==` compares references (memory address); `.equals()` compares content/value. For `String`, always use `.equals()`.

**Q: What is the difference between `String`, `StringBuilder`, `StringBuffer`?**
- `String` — immutable.
- `StringBuilder` — mutable, **not** thread-safe, faster.
- `StringBuffer` — mutable, thread-safe (synchronized), slower.

**Q: Difference between `ArrayList` and `LinkedList`?**
- `ArrayList` — backed by dynamic array, fast random access O(1), slow insert/delete in middle O(n).
- `LinkedList` — doubly linked list, fast insert/delete O(1), slow random access O(n).

**Q: Difference between `final`, `finally`, `finalize`?**
- `final` — keyword to make variable constant / method non-overridable / class non-inheritable.
- `finally` — block that always executes after try/catch.
- `finalize()` — method called by GC before object is destroyed (deprecated).

**Q: What is the difference between overloading and overriding?**
- **Overloading** — same method name, different parameters, compile-time (static) polymorphism.
- **Overriding** — subclass redefines parent method, same signature, runtime (dynamic) polymorphism.

**Q: What is the difference between an abstract class and an interface?**
- Abstract class — can have constructors, state, concrete + abstract methods; single inheritance.
- Interface — pure contract (default/static methods since Java 8); multiple inheritance of type.

**Q: What are the four pillars of OOP?**
Encapsulation, Inheritance, Polymorphism, Abstraction.

### Coding Round Strategy
1. **Clarify** the input/output and constraints.
2. **State** brute-force first, then optimize.
3. **Discuss** time & space complexity.
4. **Handle edge cases:** empty input, nulls, single element, negatives, overflow.
5. **Dry-run** your code with a small example before saying "done".

### Most Repeated Across All Three Companies
- Reverse string / number
- Palindrome check
- Prime / Fibonacci / Factorial / Armstrong
- Second largest in array
- Remove duplicates
- Count character/word frequency
- Anagram check
- Pattern printing
- Swap two numbers without a third variable

```java
// Swap without third variable
int a = 5, b = 10;
a = a + b; // 15
b = a - b; // 5
a = a - b; // 10
```

---

*Practice each by hand on paper — service-company rounds often ask you to write code on a whiteboard or in a plain editor without auto-complete.*
