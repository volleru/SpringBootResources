# Top 15 Easy LeetCode-Style Interview Questions & Answers (Java)

Classic warm-up problems frequently asked in screening rounds. Each includes problem, approach, complexity, and clean Java solution.

---

## 1. Two Sum

**Problem:** Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`.

**Example:** `nums = [2,7,11,15], target = 9` → `[0,1]`

**Approach:** Use a HashMap to store `value → index`. For each number, check if `target - num` already exists.

**Complexity:** O(n) time, O(n) space.

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i);
    }
    return new int[]{};
}
```

---

## 2. Valid Parentheses

**Problem:** Given a string with `()[]{}`, determine if input is valid (every open bracket has a matching close in correct order).

**Example:** `"()[]{}"` → `true`, `"(]"` → `false`

**Approach:** Stack — push opens, on close pop and check match.

**Complexity:** O(n) time, O(n) space.

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c);
        } else if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
            return false;
        }
    }
    return stack.isEmpty();
}
```

---

## 3. Reverse a Linked List

**Problem:** Given the head of a singly linked list, reverse it.

**Approach:** Iterate, flipping each node's `next` pointer using three pointers (prev, curr, next).

**Complexity:** O(n) time, O(1) space.

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

## 4. Merge Two Sorted Lists

**Problem:** Merge two sorted linked lists into one sorted list.

**Approach:** Dummy node, walk both lists, attach smaller node.

**Complexity:** O(n + m) time, O(1) space.

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

## 5. Best Time to Buy and Sell Stock

**Problem:** Given daily stock prices, find max profit from one buy + one sell (buy must precede sell).

**Example:** `[7,1,5,3,6,4]` → `5` (buy at 1, sell at 6)

**Approach:** Track running minimum price; at each day compute potential profit.

**Complexity:** O(n) time, O(1) space.

```java
public int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE, maxProfit = 0;
    for (int p : prices) {
        minPrice = Math.min(minPrice, p);
        maxProfit = Math.max(maxProfit, p - minPrice);
    }
    return maxProfit;
}
```

---

## 6. Valid Palindrome

**Problem:** Check if a string is a palindrome considering only alphanumeric chars, ignoring case.

**Example:** `"A man, a plan, a canal: Panama"` → `true`

**Approach:** Two pointers from both ends, skip non-alphanumeric.

**Complexity:** O(n) time, O(1) space.

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

## 7. Maximum Subarray (Kadane's Algorithm)

**Problem:** Find the contiguous subarray with the largest sum.

**Example:** `[-2,1,-3,4,-1,2,1,-5,4]` → `6` (subarray `[4,-1,2,1]`)

**Approach:** At each index, decide: extend current subarray or start fresh.

**Complexity:** O(n) time, O(1) space.

```java
public int maxSubArray(int[] nums) {
    int currSum = nums[0], maxSum = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currSum = Math.max(nums[i], currSum + nums[i]);
        maxSum = Math.max(maxSum, currSum);
    }
    return maxSum;
}
```

---

## 8. Contains Duplicate

**Problem:** Return `true` if any value appears at least twice in the array.

**Approach:** HashSet — return true on first repeat.

**Complexity:** O(n) time, O(n) space.

```java
public boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) {
        if (!seen.add(n)) return true;
    }
    return false;
}
```

---

## 9. Valid Anagram

**Problem:** Given two strings, return true if `t` is an anagram of `s`.

**Example:** `s = "anagram", t = "nagaram"` → `true`

**Approach:** Count chars (array of 26 for lowercase letters).

**Complexity:** O(n) time, O(1) space.

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

## 10. Climbing Stairs (Fibonacci)

**Problem:** You can climb 1 or 2 steps at a time. How many distinct ways to reach step `n`?

**Example:** `n = 3` → `3` (1+1+1, 1+2, 2+1)

**Approach:** `f(n) = f(n-1) + f(n-2)` — Fibonacci pattern; iterate bottom-up.

**Complexity:** O(n) time, O(1) space.

```java
public int climbStairs(int n) {
    if (n <= 2) return n;
    int prev1 = 1, prev2 = 2;
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev1 = prev2;
        prev2 = curr;
    }
    return prev2;
}
```

---

## 11. Single Number

**Problem:** Every element appears twice except one. Find that one. Linear time, no extra memory.

**Example:** `[4,1,2,1,2]` → `4`

**Approach:** XOR all elements — duplicates cancel out (`a ^ a = 0`, `a ^ 0 = a`).

**Complexity:** O(n) time, O(1) space.

```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;
    return result;
}
```

---

## 12. Invert Binary Tree

**Problem:** Mirror a binary tree (swap left and right at every node).

**Approach:** Recursive — swap children, recurse.

**Complexity:** O(n) time, O(h) space (recursion depth).

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

## 13. Maximum Depth of Binary Tree

**Problem:** Return the depth (number of nodes along longest path from root to leaf).

**Approach:** Recursive DFS — `1 + max(leftDepth, rightDepth)`.

**Complexity:** O(n) time, O(h) space.

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

---

## 14. Linked List Cycle

**Problem:** Determine if a linked list has a cycle.

**Approach:** Floyd's Tortoise & Hare — slow pointer (+1) and fast pointer (+2). If they meet, there's a cycle.

**Complexity:** O(n) time, O(1) space.

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

## 15. FizzBuzz

**Problem:** For `i` from 1 to n: print "Fizz" if divisible by 3, "Buzz" if by 5, "FizzBuzz" if by both, otherwise the number.

**Approach:** Iterate; check mod 15 first, then 3, then 5.

**Complexity:** O(n) time, O(n) output space.

```java
public List<String> fizzBuzz(int n) {
    List<String> result = new ArrayList<>(n);
    for (int i = 1; i <= n; i++) {
        if (i % 15 == 0)      result.add("FizzBuzz");
        else if (i % 3 == 0)  result.add("Fizz");
        else if (i % 5 == 0)  result.add("Buzz");
        else                  result.add(String.valueOf(i));
    }
    return result;
}
```

---

## Quick Reference — Patterns to Recognize

| Pattern | When to use | Example problems |
|---|---|---|
| **HashMap lookup** | Need O(1) "have I seen this?" | Two Sum, Contains Duplicate |
| **Two pointers** | Sorted array, palindrome, pair sums | Valid Palindrome |
| **Sliding window** | Subarray / substring with constraint | Max Subarray (variant) |
| **Stack** | Matching, nearest greater/smaller | Valid Parentheses |
| **Fast & slow pointer** | Cycle detection, middle of list | Linked List Cycle |
| **DFS recursion** | Tree problems | Max Depth, Invert Tree |
| **DP (bottom-up)** | "Count ways", "min/max to reach" | Climbing Stairs |
| **XOR trick** | Find unique element among duplicates | Single Number |
| **Kadane's** | Max contiguous sum | Maximum Subarray |
