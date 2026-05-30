# 30 LeetCode-Style Problems for Wipro Interview Prep (Java)

Curated mix of **easy + medium** problems commonly asked at Wipro and similar Indian IT companies for senior Java roles. Covers arrays, strings, linked lists, trees, hashing, two-pointers, sliding window, stack, heap, design (LRU), and core DP patterns.

Each problem includes problem statement, approach, complexity, and a clean Java solution.

---

## 📦 Section 1 — Arrays & Hashing

### 1. Two Sum (Easy)
Given `nums` and `target`, return indices of two numbers that sum to target.

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        if (seen.containsKey(target - nums[i])) return new int[]{seen.get(target - nums[i]), i};
        seen.put(nums[i], i);
    }
    return new int[]{};
}
```
**O(n) time, O(n) space.**

---

### 2. 3Sum (Medium)
Find all unique triplets in array that sum to 0.

**Approach:** Sort → fix one element → two-pointer on the rest. Skip duplicates.

```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;
        int l = i + 1, r = nums.length - 1;
        while (l < r) {
            int sum = nums[i] + nums[l] + nums[r];
            if (sum == 0) {
                res.add(List.of(nums[i], nums[l], nums[r]));
                while (l < r && nums[l] == nums[l+1]) l++;
                while (l < r && nums[r] == nums[r-1]) r--;
                l++; r--;
            } else if (sum < 0) l++;
            else r--;
        }
    }
    return res;
}
```
**O(n²) time, O(1) extra space.**

---

### 3. Contains Duplicate (Easy)
Return true if any element appears twice.

```java
public boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) if (!seen.add(n)) return true;
    return false;
}
```
**O(n) time, O(n) space.**

---

### 4. Product of Array Except Self (Medium)
Return array where `output[i]` = product of all elements except `nums[i]`. **No division.**

**Approach:** Prefix product from left, then suffix product from right in one pass each.

```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    res[0] = 1;
    for (int i = 1; i < n; i++) res[i] = res[i-1] * nums[i-1];
    int right = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= right;
        right *= nums[i];
    }
    return res;
}
```
**O(n) time, O(1) extra space.**

---

### 5. Maximum Subarray — Kadane's Algorithm (Medium)
Find the contiguous subarray with the largest sum.

```java
public int maxSubArray(int[] nums) {
    int curr = nums[0], max = nums[0];
    for (int i = 1; i < nums.length; i++) {
        curr = Math.max(nums[i], curr + nums[i]);
        max = Math.max(max, curr);
    }
    return max;
}
```
**O(n) time, O(1) space.**

---

### 6. Merge Intervals (Medium)
Given a collection of intervals, merge overlapping ones.

```java
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> merged = new ArrayList<>();
    for (int[] curr : intervals) {
        if (merged.isEmpty() || merged.get(merged.size()-1)[1] < curr[0]) {
            merged.add(curr);
        } else {
            merged.get(merged.size()-1)[1] = Math.max(merged.get(merged.size()-1)[1], curr[1]);
        }
    }
    return merged.toArray(new int[0][]);
}
```
**O(n log n) time.**

---

### 7. Best Time to Buy and Sell Stock (Easy)
One buy + one sell — max profit.

```java
public int maxProfit(int[] prices) {
    int min = Integer.MAX_VALUE, profit = 0;
    for (int p : prices) {
        min = Math.min(min, p);
        profit = Math.max(profit, p - min);
    }
    return profit;
}
```
**O(n) time, O(1) space.**

---

## 🔤 Section 2 — Strings

### 8. Valid Anagram (Easy)
Check if `t` is an anagram of `s`.

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (int i = 0; i < s.length(); i++) {
        count[s.charAt(i) - 'a']++;
        count[t.charAt(i) - 'a']--;
    }
    for (int c : count) if (c != 0) return false;
    return true;
}
```

---

### 9. Valid Palindrome (Easy)
Two pointers, skip non-alphanumeric.

```java
public boolean isPalindrome(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) {
        while (l < r && !Character.isLetterOrDigit(s.charAt(l))) l++;
        while (l < r && !Character.isLetterOrDigit(s.charAt(r))) r--;
        if (Character.toLowerCase(s.charAt(l)) != Character.toLowerCase(s.charAt(r))) return false;
        l++; r--;
    }
    return true;
}
```

---

### 10. Longest Substring Without Repeating Characters (Medium)
**Pattern: Sliding window.**

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> idx = new HashMap<>();
    int max = 0, start = 0;
    for (int i = 0; i < s.length(); i++) {
        if (idx.containsKey(s.charAt(i))) {
            start = Math.max(start, idx.get(s.charAt(i)) + 1);
        }
        idx.put(s.charAt(i), i);
        max = Math.max(max, i - start + 1);
    }
    return max;
}
```
**O(n) time, O(min(n, charset)) space.**

---

### 11. Group Anagrams (Medium)
Group strings that are anagrams of each other.

```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] c = s.toCharArray();
        Arrays.sort(c);
        map.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(map.values());
}
```
**O(n × k log k)** where k = max string length.

---

### 12. Longest Palindromic Substring (Medium)
**Expand around center** — every char (and every gap between chars) is a potential center.

```java
public String longestPalindrome(String s) {
    int start = 0, end = 0;
    for (int i = 0; i < s.length(); i++) {
        int l1 = expand(s, i, i);     // odd length
        int l2 = expand(s, i, i + 1); // even length
        int len = Math.max(l1, l2);
        if (len > end - start) {
            start = i - (len - 1) / 2;
            end = i + len / 2;
        }
    }
    return s.substring(start, end + 1);
}
private int expand(String s, int l, int r) {
    while (l >= 0 && r < s.length() && s.charAt(l) == s.charAt(r)) { l--; r++; }
    return r - l - 1;
}
```
**O(n²) time, O(1) space.**

---

## 🔗 Section 3 — Linked Lists

### 13. Reverse a Linked List (Easy)

```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

---

### 14. Merge Two Sorted Lists (Easy)

```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { tail.next = l1; l1 = l1.next; }
        else                  { tail.next = l2; l2 = l2.next; }
        tail = tail.next;
    }
    tail.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

---

### 15. Linked List Cycle (Easy) — Floyd's Algorithm

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

---

### 16. Remove Nth Node From End of List (Medium)
**Two-pointer technique** — fast pointer N steps ahead.

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i <= n; i++) fast = fast.next;
    while (fast != null) { fast = fast.next; slow = slow.next; }
    slow.next = slow.next.next;
    return dummy.next;
}
```

---

### 17. Add Two Numbers (Medium)
Two linked lists store digits in reverse — add and return as linked list.

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), curr = dummy;
    int carry = 0;
    while (l1 != null || l2 != null || carry != 0) {
        int sum = carry;
        if (l1 != null) { sum += l1.val; l1 = l1.next; }
        if (l2 != null) { sum += l2.val; l2 = l2.next; }
        carry = sum / 10;
        curr.next = new ListNode(sum % 10);
        curr = curr.next;
    }
    return dummy.next;
}
```

---

## 🌳 Section 4 — Trees

### 18. Maximum Depth of Binary Tree (Easy)

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

---

### 19. Invert Binary Tree (Easy)

```java
public TreeNode invertTree(TreeNode root) {
    if (root == null) return null;
    TreeNode left = invertTree(root.left);
    TreeNode right = invertTree(root.right);
    root.left = right;
    root.right = left;
    return root;
}
```

---

### 20. Validate Binary Search Tree (Medium)
Use min/max bounds — every node must lie strictly within.

```java
public boolean isValidBST(TreeNode root) {
    return valid(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
private boolean valid(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return valid(node.left, min, node.val) && valid(node.right, node.val, max);
}
```

---

### 21. Binary Tree Level Order Traversal (Medium)
Classic **BFS** with a queue.

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            level.add(node.val);
            if (node.left != null) q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
        res.add(level);
    }
    return res;
}
```

---

### 22. Lowest Common Ancestor of a Binary Tree (Medium)

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

---

## 📚 Section 5 — Stack & Queue

### 23. Valid Parentheses (Easy)

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pair = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (pair.containsValue(c)) stack.push(c);
        else if (stack.isEmpty() || stack.pop() != pair.get(c)) return false;
    }
    return stack.isEmpty();
}
```

---

### 24. Min Stack (Medium)
Stack supporting `push`, `pop`, `top`, `getMin` — all O(1).

**Trick:** Keep a parallel stack of running mins.

```java
class MinStack {
    private Deque<Integer> stack = new ArrayDeque<>();
    private Deque<Integer> mins  = new ArrayDeque<>();

    public void push(int val) {
        stack.push(val);
        mins.push(mins.isEmpty() ? val : Math.min(val, mins.peek()));
    }
    public void pop() { stack.pop(); mins.pop(); }
    public int top() { return stack.peek(); }
    public int getMin() { return mins.peek(); }
}
```

---

### 25. Evaluate Reverse Polish Notation (Medium)

```java
public int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (String t : tokens) {
        switch (t) {
            case "+", "-", "*", "/" -> {
                int b = stack.pop(), a = stack.pop();
                stack.push(switch (t) {
                    case "+" -> a + b;
                    case "-" -> a - b;
                    case "*" -> a * b;
                    default  -> a / b;
                });
            }
            default -> stack.push(Integer.parseInt(t));
        }
    }
    return stack.pop();
}
```

---

## 🎯 Section 6 — Design

### 26. LRU Cache (Medium) ⭐ **Very common at Wipro**
`get(key)` and `put(key, value)` in **O(1)**.

**Approach:** HashMap + Doubly Linked List. Map → node; list keeps usage order. Head = most recent, Tail = least recent.

```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head, tail;  // sentinels

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        moveToHead(node);
        return node.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.val = value;
            moveToHead(node);
            return;
        }
        if (map.size() == capacity) {
            Node lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
        Node node = new Node(key, value);
        addToHead(node);
        map.put(key, node);
    }

    private void addToHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    private void moveToHead(Node node) { remove(node); addToHead(node); }

    private static class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }
}
```
**Follow-up:** Could also use `LinkedHashMap` with `accessOrder=true` and override `removeEldestEntry()` — interviewers often want the manual DLL approach.

---

## 🔢 Section 7 — Binary Search

### 27. Binary Search (Easy)

```java
public int search(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l <= r) {
        int mid = l + (r - l) / 2;  // avoid overflow
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) l = mid + 1;
        else r = mid - 1;
    }
    return -1;
}
```

---

### 28. Search in Rotated Sorted Array (Medium)

```java
public int search(int[] nums, int target) {
    int l = 0, r = nums.length - 1;
    while (l <= r) {
        int mid = l + (r - l) / 2;
        if (nums[mid] == target) return mid;
        if (nums[l] <= nums[mid]) {  // left half sorted
            if (nums[l] <= target && target < nums[mid]) r = mid - 1;
            else l = mid + 1;
        } else {                     // right half sorted
            if (nums[mid] < target && target <= nums[r]) l = mid + 1;
            else r = mid - 1;
        }
    }
    return -1;
}
```
**O(log n) time.**

---

## 🪜 Section 8 — Dynamic Programming Basics

### 29. Climbing Stairs (Easy)
`f(n) = f(n-1) + f(n-2)` — Fibonacci.

```java
public int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

---

### 30. Coin Change (Medium)
Minimum number of coins to make amount. Return -1 if impossible.

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int a = 1; a <= amount; a++) {
        for (int c : coins) {
            if (c <= a) dp[a] = Math.min(dp[a], dp[a - c] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```
**O(amount × coins) time, O(amount) space.**

---

## 🧠 Pattern Cheat Sheet

| Pattern | Signals | Examples in this list |
|---|---|---|
| **HashMap lookup** | "have I seen this before?" | Two Sum, Contains Duplicate, Group Anagrams |
| **Two pointers** | sorted array, palindromes | 3Sum, Valid Palindrome |
| **Sliding window** | longest/shortest subarray with property | Longest Substring Without Repeating |
| **Fast & slow pointer** | linked list cycle / middle / nth from end | Linked List Cycle, Remove Nth |
| **Stack** | matching, undo/redo, RPN, min in O(1) | Valid Parens, Min Stack, RPN |
| **BFS** | level-by-level traversal, shortest path in unweighted | Level Order Traversal |
| **DFS (recursion)** | tree / graph traversal, validation | Max Depth, Validate BST, LCA |
| **Binary search** | sorted array, O(log n) | Binary Search, Rotated Search |
| **DP (1D)** | "count ways" or "min/max to reach" | Climbing Stairs, Coin Change |
| **Hash + DLL** | O(1) get/put with order | LRU Cache |
| **Prefix/Suffix product** | "all except self" without division | Product Except Self |
| **Expand around center** | longest palindromic substring | Longest Palindrome |

---

## 💡 Interview Tips for Wipro (Java-specific)

1. **Always state complexity** before writing code: "I'll do this in O(n) using a HashMap."
2. **Edge cases first** — empty array, single element, null inputs, integer overflow.
3. **Use `Deque` not `Stack`** — `Stack` is legacy in Java (synchronized, slow). Use `ArrayDeque`.
4. **Prefer `HashMap` over `Hashtable`** — same reason.
5. **Watch for integer overflow** — `l + (r - l) / 2` in binary search, `long` cast for sum problems.
6. **`StringBuilder` for string building** — never use `+=` in a loop.
7. **For LRU specifically** — interviewer may ask "Why DLL not singly linked?" → Answer: O(1) removal of a known node.
8. **Talk through your approach before coding** — Wipro interviewers value communication.
9. **Mention Java 21 features when relevant** — records, switch expressions, sealed classes show modern Java knowledge.
10. **Test your solution** — walk through with the example input after writing.
