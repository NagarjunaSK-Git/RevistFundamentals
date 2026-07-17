# Top 75 DSA Questions for SDET Interviews (Java)

**Source note:** This list is compiled from cross-referencing frequency lists on GeeksforGeeks ("Top 100 DSA Interview Questions"), InterviewBit's SDET and DSA interview guides, Preplaced's "50 Most Asked DSA Questions", and recurring problems mentioned in SDET-specific LinkedIn interview-experience posts. The problems below repeatedly show up across these sources for SDET and general SDE interviews at companies like Microsoft, Amazon, and other product-based firms that hire SDETs.

**How to read each entry:**
- **Problem** – plain statement + a worked example
- **Brute Force** – the "obvious" way, explained for beginners
- **Better Solution** – one clean, interview-ready solution with explanation
- **Complexity** is called out for both so you understand *why* the better one is better

All code is plain Java so you can paste it directly into a `.java` file or an online IDE.

---

## Category 1: Arrays & Strings

### Q1. Two Sum
**Problem:** Given an array of integers and a target, return the indices of the two numbers that add up to the target.
**Example:** `nums = [2, 7, 11, 15]`, `target = 9` → output `[0, 1]` (because `2 + 7 = 9`).
**Frequency:** One of the single most repeated questions on every list (GfG, InterviewBit, Preplaced all list it #1).

**Brute Force** — check every pair with two loops:
```java
public int[] twoSumBrute(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] + nums[j] == target) {
                return new int[]{i, j};
            }
        }
    }
    return new int[]{-1, -1}; // not found
}
```
*Why brute force works but is slow:* you compare every element with every other element. For `n` numbers that's roughly `n*n` comparisons. Time: **O(n²)**, Space: **O(1)**.

**Better Solution** — HashMap (one pass):
```java
public int[] twoSumOptimal(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>(); // value -> index
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
```
**Explanation:** Instead of re-scanning the array for a partner, remember every number you've already seen in a HashMap as you go. Before inserting the current number, check if its "partner" (`target - nums[i]`) already exists in the map. HashMap lookups are O(1) on average, so the whole scan is **O(n)** time, **O(n)** space — a huge win over brute force for large arrays.

---

### Q2. Reverse an Array/String In-Place
**Problem:** Reverse an array (or char array) without using extra array space.
**Example:** `[1, 2, 3, 4, 5]` → `[5, 4, 3, 2, 1]`.

**Brute Force** — copy into a new array in reverse order:
```java
public int[] reverseBrute(int[] arr) {
    int[] result = new int[arr.length];
    for (int i = 0; i < arr.length; i++) {
        result[arr.length - 1 - i] = arr[i];
    }
    return result;
}
```
Time: O(n), but Space: **O(n)** extra — defeats the "in-place" requirement often asked in the interview.

**Better Solution** — two-pointer swap, no extra array:
```java
public void reverseOptimal(int[] arr) {
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
**Explanation:** Keep two pointers, one at the start and one at the end. Swap the elements they point to, then move them toward the middle. When they meet, the array is fully reversed. Time: **O(n)**, Space: **O(1)** — this "two-pointer" pattern shows up constantly in array/string problems, so it's worth memorizing.

---

### Q3. Find the Duplicate Number
**Problem:** Given an array of `n+1` integers where every value is between `1` and `n`, find the one duplicated number.
**Example:** `[1, 3, 4, 2, 2]` → `2`.

**Brute Force** — nested loop comparing every pair:
```java
public int findDuplicateBrute(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] == nums[j]) return nums[i];
        }
    }
    return -1;
}
```
Time: **O(n²)**, Space: O(1).

**Better Solution** — HashSet:
```java
public int findDuplicateOptimal(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int num : nums) {
        if (!seen.add(num)) { // add() returns false if it was already present
            return num;
        }
    }
    return -1;
}
```
**Explanation:** `Set.add()` returns `false` if the element already existed. So you try to add every number; the moment an add fails, that number is your duplicate. Time: **O(n)**, Space: **O(n)**. (Interviewers may follow up asking for O(1) space using Floyd's cycle detection on indices — worth knowing exists, but the HashSet answer is the standard "good enough" solution.)

---

### Q4. Maximum Subarray (Kadane's Algorithm)
**Problem:** Find the contiguous subarray with the largest sum.
**Example:** `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` → max sum is `6` (subarray `[4, -1, 2, 1]`).
**Frequency:** Confirmed frequently asked at Microsoft/Amazon per Preplaced's list.

**Brute Force** — try every possible subarray:
```java
public int maxSubArrayBrute(int[] nums) {
    int maxSum = Integer.MIN_VALUE;
    for (int i = 0; i < nums.length; i++) {
        int currentSum = 0;
        for (int j = i; j < nums.length; j++) {
            currentSum += nums[j];
            maxSum = Math.max(maxSum, currentSum);
        }
    }
    return maxSum;
}
```
Time: **O(n²)**, Space: O(1).

**Better Solution** — Kadane's Algorithm (single pass):
```java
public int maxSubArrayOptimal(int[] nums) {
    int maxSoFar = nums[0];
    int maxEndingHere = nums[0];
    for (int i = 1; i < nums.length; i++) {
        maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
        maxSoFar = Math.max(maxSoFar, maxEndingHere);
    }
    return maxSoFar;
}
```
**Explanation:** At each index, decide: "is it better to extend the previous subarray by adding the current number, or start a fresh subarray here?" That decision is `Math.max(nums[i], maxEndingHere + nums[i])`. Track the best value seen so far separately. One pass, no nested loop. Time: **O(n)**, Space: **O(1)**.

---

### Q5. Merge Overlapping Intervals
**Problem:** Given a list of intervals, merge all overlapping ones.
**Example:** `[[1,3],[2,6],[8,10],[15,18]]` → `[[1,6],[8,10],[15,18]]`.

**Brute Force** — repeatedly scan and merge any overlapping pair until no more merges happen:
```java
public List<int[]> mergeBrute(int[][] intervals) {
    List<int[]> result = new ArrayList<>(Arrays.asList(intervals));
    boolean mergedSomething = true;
    while (mergedSomething) {
        mergedSomething = false;
        for (int i = 0; i < result.size(); i++) {
            for (int j = i + 1; j < result.size(); j++) {
                int[] a = result.get(i), b = result.get(j);
                if (a[0] <= b[1] && b[0] <= a[1]) { // overlap check
                    a[0] = Math.min(a[0], b[0]);
                    a[1] = Math.max(a[1], b[1]);
                    result.remove(j);
                    mergedSomething = true;
                    break;
                }
            }
            if (mergedSomething) break;
        }
    }
    return result;
}
```
Time: worst case **O(n³)** because of repeated rescans — messy and slow.

**Better Solution** — sort by start time, then merge in one pass:
```java
public int[][] mergeOptimal(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // sort by start time
    List<int[]> merged = new ArrayList<>();
    for (int[] interval : intervals) {
        if (merged.isEmpty() || merged.get(merged.size() - 1)[1] < interval[0]) {
            merged.add(interval); // no overlap, add as new interval
        } else {
            merged.get(merged.size() - 1)[1] =
                Math.max(merged.get(merged.size() - 1)[1], interval[1]); // extend last interval
        }
    }
    return merged.toArray(new int[merged.size()][]);
}
```
**Explanation:** Once intervals are sorted by start time, overlapping intervals are always adjacent to each other, so you only need one linear pass, extending the last merged interval whenever the next one overlaps it. Time: **O(n log n)** (dominated by the sort), Space: **O(n)**.

---

### Q6. Move Zeroes to End
**Problem:** Move all zeroes in an array to the end while keeping the relative order of non-zero elements.
**Example:** `[0, 1, 0, 3, 12]` → `[1, 3, 12, 0, 0]`.

**Brute Force** — build a new array of non-zeros, then pad with zeros:
```java
public int[] moveZeroesBrute(int[] nums) {
    int[] result = new int[nums.length];
    int idx = 0;
    for (int num : nums) {
        if (num != 0) result[idx++] = num;
    }
    // remaining slots default to 0 already
    return result;
}
```
Time: O(n), Space: **O(n)** extra array.

**Better Solution** — in-place two-pointer:
```java
public void moveZeroesOptimal(int[] nums) {
    int insertPos = 0;
    for (int num : nums) {
        if (num != 0) {
            nums[insertPos++] = num;
        }
    }
    while (insertPos < nums.length) {
        nums[insertPos++] = 0;
    }
}
```
**Explanation:** `insertPos` tracks where the next non-zero value should go. Walk through the array once, copying every non-zero value forward; then fill whatever is left with zeros. No extra array needed. Time: **O(n)**, Space: **O(1)**.

---

### Q7. Longest Substring Without Repeating Characters
**Problem:** Find the length of the longest substring without repeating characters.
**Example:** `"abcabcbb"` → `3` (the substring `"abc"`).

**Brute Force** — check every substring:
```java
public int longestSubstringBrute(String s) {
    int maxLen = 0;
    for (int i = 0; i < s.length(); i++) {
        Set<Character> seen = new HashSet<>();
        for (int j = i; j < s.length(); j++) {
            if (seen.contains(s.charAt(j))) break;
            seen.add(s.charAt(j));
            maxLen = Math.max(maxLen, j - i + 1);
        }
    }
    return maxLen;
}
```
Time: **O(n²)**, Space: O(n).

**Better Solution** — sliding window with a HashMap of last-seen index:
```java
public int longestSubstringOptimal(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int maxLen = 0, windowStart = 0;
    for (int windowEnd = 0; windowEnd < s.length(); windowEnd++) {
        char c = s.charAt(windowEnd);
        if (lastIndex.containsKey(c) && lastIndex.get(c) >= windowStart) {
            windowStart = lastIndex.get(c) + 1; // shrink window past the repeat
        }
        lastIndex.put(c, windowEnd);
        maxLen = Math.max(maxLen, windowEnd - windowStart + 1);
    }
    return maxLen;
}
```
**Explanation:** This is the classic "sliding window" pattern. `windowStart`/`windowEnd` define the current no-repeat substring. As you extend `windowEnd`, if the character was already seen *inside* the current window, jump `windowStart` right after that earlier occurrence instead of resetting to zero. Time: **O(n)**, Space: **O(min(n, charset size))**.

---

### Q8. Valid Anagram
**Problem:** Determine whether two strings are anagrams of each other (same letters, same frequency, different order allowed).
**Example:** `"anagram"`, `"nagaram"` → `true`.

**Brute Force** — sort both strings and compare:
```java
public boolean isAnagramBrute(String s, String t) {
    if (s.length() != t.length()) return false;
    char[] sArr = s.toCharArray();
    char[] tArr = t.toCharArray();
    Arrays.sort(sArr);
    Arrays.sort(tArr);
    return Arrays.equals(sArr, tArr);
}
```
Time: **O(n log n)** because of sorting.

**Better Solution** — frequency count array:
```java
public boolean isAnagramOptimal(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] freq = new int[26]; // assumes lowercase a-z
    for (int i = 0; i < s.length(); i++) {
        freq[s.charAt(i) - 'a']++;
        freq[t.charAt(i) - 'a']--;
    }
    for (int count : freq) {
        if (count != 0) return false;
    }
    return true;
}
```
**Explanation:** Instead of sorting, count how many times each letter appears in `s` (increment) and in `t` (decrement) using the same array. If the two strings are true anagrams, every counter ends up exactly at zero. Time: **O(n)**, Space: **O(1)** (fixed 26-size array).

---

### Q9. Group Anagrams
**Problem:** Group a list of strings so that all anagrams end up in the same group.
**Example:** `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`.

**Brute Force** — compare every string with every group's representative:
```java
public List<List<String>> groupAnagramsBrute(String[] strs) {
    List<List<String>> groups = new ArrayList<>();
    for (String s : strs) {
        boolean placed = false;
        for (List<String> group : groups) {
            if (isAnagramOptimal(s, group.get(0))) { // reuse Q8 helper
                group.add(s);
                placed = true;
                break;
            }
        }
        if (!placed) {
            List<String> newGroup = new ArrayList<>();
            newGroup.add(s);
            groups.add(newGroup);
        }
    }
    return groups;
}
```
Time: **O(n² * k)** where k is average string length — comparing every string against every existing group is slow.

**Better Solution** — HashMap keyed by sorted string:
```java
public List<List<String>> groupAnagramsOptimal(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars); // anagrams share the same sorted key
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}
```
**Explanation:** All anagrams produce the exact same string once sorted (e.g., `"eat"` and `"tea"` both sort to `"aet"`). Use that sorted string as a HashMap key, and bucket every original word under it. Time: **O(n * k log k)** where k = average word length (dominated by sorting each word), Space: **O(n * k)**.

---

### Q10. Check if a String is a Palindrome
**Problem:** Determine whether a string reads the same forwards and backwards.
**Example:** `"racecar"` → `true`; `"hello"` → `false`.

**Brute Force** — build the reverse and compare:
```java
public boolean isPalindromeBrute(String s) {
    String reversed = new StringBuilder(s).reverse().toString();
    return s.equals(reversed);
}
```
Time: O(n), Space: **O(n)** extra for the reversed copy.

**Better Solution** — two-pointer, no extra string:
```java
public boolean isPalindromeOptimal(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}
```
**Explanation:** Compare characters from both ends moving inward; if any pair mismatches, it's not a palindrome. You can exit early on the first mismatch, and never allocate a second string. Time: **O(n)**, Space: **O(1)**.

---

### Q11. Rotate an Array by K Steps
**Problem:** Rotate an array to the right by `k` steps.
**Example:** `[1,2,3,4,5,6,7]`, `k=3` → `[5,6,7,1,2,3,4]`.

**Brute Force** — rotate one step at a time, k times:
```java
public void rotateBrute(int[] nums, int k) {
    int n = nums.length;
    k = k % n;
    for (int i = 0; i < k; i++) {
        int last = nums[n - 1];
        for (int j = n - 1; j > 0; j--) {
            nums[j] = nums[j - 1];
        }
        nums[0] = last;
    }
}
```
Time: **O(n*k)** — rotating one step at a time k times is wasteful.

**Better Solution** — reverse-based in-place rotation:
```java
public void rotateOptimal(int[] nums, int k) {
    int n = nums.length;
    k = k % n;
    reverse(nums, 0, n - 1);       // reverse whole array
    reverse(nums, 0, k - 1);       // reverse first k elements
    reverse(nums, k, n - 1);       // reverse remaining elements
}

private void reverse(int[] arr, int start, int end) {
    while (start < end) {
        int temp = arr[start];
        arr[start] = arr[end];
        arr[end] = temp;
        start++;
        end--;
    }
}
```
**Explanation:** A neat trick: reversing the whole array puts elements in the right "rotated" neighborhoods but backwards; reversing the first `k` and the remaining `n-k` elements separately fixes the internal order. Three linear passes, still Time: **O(n)**, Space: **O(1)**.

---

### Q12. Product of Array Except Self
**Problem:** Return an array where each element is the product of all other elements (without using division).
**Example:** `[1,2,3,4]` → `[24,12,8,6]`.

**Brute Force** — for every index, multiply everything except that index:
```java
public int[] productExceptSelfBrute(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    for (int i = 0; i < n; i++) {
        int product = 1;
        for (int j = 0; j < n; j++) {
            if (i != j) product *= nums[j];
        }
        result[i] = product;
    }
    return result;
}
```
Time: **O(n²)**.

**Better Solution** — prefix and suffix products:
```java
public int[] productExceptSelfOptimal(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1]; // prefix product up to i-1
    }
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= suffix;       // multiply in suffix product from the right
        suffix *= nums[i];
    }
    return result;
}
```
**Explanation:** `result[i]` needs "product of everything to the left" times "product of everything to the right." First pass fills in left-products; second pass (going backward) multiplies in the right-products on the fly using a running `suffix` variable. Time: **O(n)**, Space: **O(n)** for the output array only (no extra arrays needed).

---
## Category 2: Linked List

*Shared node definition used in this section:*
```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}
```

### Q13. Reverse a Linked List
**Problem:** Reverse a singly linked list.
**Example:** `1->2->3->4->5->null` → `5->4->3->2->1->null`.
**Frequency:** Confirmed as "one of the most frequently asked" per multiple 2026 interview-prep sources.

**Brute Force** — copy values into a list, then rebuild the linked list in reverse order:
```java
public ListNode reverseBrute(ListNode head) {
    List<Integer> values = new ArrayList<>();
    for (ListNode cur = head; cur != null; cur = cur.next) values.add(cur.val);
    Collections.reverse(values);
    ListNode dummy = new ListNode(0), tail = dummy;
    for (int v : values) {
        tail.next = new ListNode(v);
        tail = tail.next;
    }
    return dummy.next;
}
```
Time: O(n), Space: **O(n)** extra for the list and new nodes — unnecessary.

**Better Solution** — iterative pointer reversal, in-place:
```java
public ListNode reverseOptimal(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next; // save next before overwriting
        curr.next = prev;          // reverse the link
        prev = curr;
        curr = next;
    }
    return prev; // prev is now the new head
}
```
**Explanation:** Walk through the list once, and for each node, flip its `next` pointer to point backward instead of forward. Three pointers (`prev`, `curr`, `next`) are enough to do this without losing your place. Time: **O(n)**, Space: **O(1)**.

---

### Q14. Detect Cycle in a Linked List (Floyd's Algorithm)
**Problem:** Determine whether a linked list has a cycle.
**Example:** `1->2->3->4->2 (back to node 2)` → `true`.

**Brute Force** — HashSet of visited nodes:
```java
public boolean hasCycleBrute(ListNode head) {
    Set<ListNode> visited = new HashSet<>();
    for (ListNode cur = head; cur != null; cur = cur.next) {
        if (!visited.add(cur)) return true; // already seen this node
    }
    return false;
}
```
Time: O(n), Space: **O(n)** — needs to store every node visited.

**Better Solution** — Floyd's Tortoise and Hare (two pointers, different speeds):
```java
public boolean hasCycleOptimal(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true; // pointers met -> cycle
    }
    return false;
}
```
**Explanation:** `slow` moves 1 step at a time, `fast` moves 2 steps. If there's no cycle, `fast` reaches `null` and you exit. If there IS a cycle, `fast` will eventually "lap" `slow` and they'll land on the same node. Time: **O(n)**, Space: **O(1)** — this is the gold-standard answer interviewers want.

---

### Q15. Find Middle of Linked List
**Problem:** Find the middle node of a linked list in one pass.
**Example:** `1->2->3->4->5` → `3`.

**Brute Force** — count length, then walk to `length/2`:
```java
public ListNode middleBrute(ListNode head) {
    int len = 0;
    for (ListNode cur = head; cur != null; cur = cur.next) len++;
    ListNode cur = head;
    for (int i = 0; i < len / 2; i++) cur = cur.next;
    return cur;
}
```
Time: O(n) but requires **two passes** through the list.

**Better Solution** — slow/fast pointer, single pass:
```java
public ListNode middleOptimal(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```
**Explanation:** Same slow/fast idea as cycle detection. Since `fast` moves twice as quickly, by the time it reaches the end, `slow` is exactly at the midpoint. Time: **O(n)**, Space: **O(1)**, and only one traversal.

---

### Q16. Merge Two Sorted Linked Lists
**Problem:** Merge two sorted linked lists into one sorted list.
**Example:** `1->3->5` and `2->4->6` → `1->2->3->4->5->6`.

**Brute Force** — dump both lists' values into an array, sort, rebuild:
```java
public ListNode mergeBrute(ListNode l1, ListNode l2) {
    List<Integer> values = new ArrayList<>();
    for (ListNode cur = l1; cur != null; cur = cur.next) values.add(cur.val);
    for (ListNode cur = l2; cur != null; cur = cur.next) values.add(cur.val);
    Collections.sort(values);
    ListNode dummy = new ListNode(0), tail = dummy;
    for (int v : values) {
        tail.next = new ListNode(v);
        tail = tail.next;
    }
    return dummy.next;
}
```
Time: **O(n log n)** because of sorting — ignores the fact both lists are already sorted.

**Better Solution** — merge by comparing heads (like merge step of merge sort):
```java
public ListNode mergeOptimal(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            tail.next = l1;
            l1 = l1.next;
        } else {
            tail.next = l2;
            l2 = l2.next;
        }
        tail = tail.next;
    }
    tail.next = (l1 != null) ? l1 : l2; // attach whichever list has leftovers
    return dummy.next;
}
```
**Explanation:** Since both inputs are already sorted, just compare the current heads of each list, attach the smaller one to the result, and advance that list's pointer. A dummy head node avoids special-casing the very first attachment. Time: **O(n + m)**, Space: **O(1)** extra (reuses existing nodes).

---

### Q17. Remove Nth Node From End of List
**Problem:** Remove the nth node from the end of a linked list, in one pass.
**Example:** `1->2->3->4->5`, `n=2` → `1->2->3->5`.

**Brute Force** — find length, then walk to the node before the target and unlink it (two passes):
```java
public ListNode removeNthBrute(ListNode head, int n) {
    int len = 0;
    for (ListNode cur = head; cur != null; cur = cur.next) len++;
    if (len == n) return head.next; // removing the head itself
    ListNode cur = head;
    for (int i = 0; i < len - n - 1; i++) cur = cur.next;
    cur.next = cur.next.next;
    return head;
}
```
Works fine but needs two full passes over the list.

**Better Solution** — two pointers with a gap of `n`, one pass:
```java
public ListNode removeNthOptimal(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode slow = dummy, fast = dummy;
    for (int i = 0; i < n; i++) fast = fast.next; // advance fast n steps first
    while (fast.next != null) {
        slow = slow.next;
        fast = fast.next;
    }
    slow.next = slow.next.next; // unlink the target node
    return dummy.next;
}
```
**Explanation:** Move `fast` ahead by `n` nodes first. Then move `slow` and `fast` together — when `fast` reaches the last node, `slow` is sitting right before the node that needs removing, because the gap between them is always `n`. Time: **O(n)** single pass, Space: **O(1)**.

---

### Q18. Palindrome Linked List
**Problem:** Check whether a linked list reads the same forwards and backwards.
**Example:** `1->2->2->1` → `true`.

**Brute Force** — copy values into an array/list, then compare with its reverse:
```java
public boolean isPalindromeBrute(ListNode head) {
    List<Integer> values = new ArrayList<>();
    for (ListNode cur = head; cur != null; cur = cur.next) values.add(cur.val);
    int left = 0, right = values.size() - 1;
    while (left < right) {
        if (!values.get(left).equals(values.get(right))) return false;
        left++; right--;
    }
    return true;
}
```
Time: O(n), Space: **O(n)** for the copied list.

**Better Solution** — find middle, reverse second half, compare (reusing Q13/Q15 ideas):
```java
public boolean isPalindromeOptimal(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) { // find middle
        slow = slow.next;
        fast = fast.next.next;
    }
    ListNode secondHalf = reverseOptimal(slow); // reverse from middle onward
    ListNode firstHalf = head;
    while (secondHalf != null) {
        if (firstHalf.val != secondHalf.val) return false;
        firstHalf = firstHalf.next;
        secondHalf = secondHalf.next;
    }
    return true;
}
```
**Explanation:** Instead of copying data out, physically reverse the second half of the list in place (reusing the reversal technique from Q13), then walk both halves together comparing values. Time: **O(n)**, Space: **O(1)** — no auxiliary array needed.

---

### Q19. Intersection Point of Two Linked Lists
**Problem:** Given two linked lists that may merge into a shared tail, find the node where they intersect.
**Example:** `A: 1->2->\`, `B: 4->5->\`, both point into shared `6->7` → returns node `6`.

**Brute Force** — for every node in list A, scan all of list B for a match:
```java
public ListNode getIntersectionBrute(ListNode headA, ListNode headB) {
    for (ListNode a = headA; a != null; a = a.next) {
        for (ListNode b = headB; b != null; b = b.next) {
            if (a == b) return a;
        }
    }
    return null;
}
```
Time: **O(n*m)**.

**Better Solution** — two pointers that "wrap around" to the other list:
```java
public ListNode getIntersectionOptimal(ListNode headA, ListNode headB) {
    ListNode a = headA, b = headB;
    while (a != b) {
        a = (a == null) ? headB : a.next;
        b = (b == null) ? headA : b.next;
    }
    return a; // either the intersection node, or null if none
}
```
**Explanation:** When a pointer reaches the end of its own list, redirect it to the *head of the other list*. This equalizes the total distance both pointers travel, so they arrive at the intersection point at the same time (or both hit `null` together if there's no intersection). Time: **O(n + m)**, Space: **O(1)** — no hashing needed.

---

### Q20. Remove Duplicates from a Sorted Linked List
**Problem:** Given a sorted linked list, remove duplicate values so each value appears once.
**Example:** `1->1->2->3->3` → `1->2->3`.

**Brute Force** — for each node, scan forward removing any node with the same value:
```java
public ListNode removeDuplicatesBrute(ListNode head) {
    for (ListNode outer = head; outer != null; outer = outer.next) {
        ListNode runner = outer;
        while (runner.next != null) {
            if (runner.next.val == outer.val) {
                runner.next = runner.next.next;
            } else {
                runner = runner.next;
            }
        }
    }
    return head;
}
```
Time: **O(n²)** in general (though on an already-sorted list, duplicates are only adjacent, so this does more work than needed).

**Better Solution** — since the list is sorted, duplicates are always adjacent — single pass:
```java
public ListNode removeDuplicatesOptimal(ListNode head) {
    ListNode cur = head;
    while (cur != null && cur.next != null) {
        if (cur.val == cur.next.val) {
            cur.next = cur.next.next; // skip the duplicate
        } else {
            cur = cur.next;
        }
    }
    return head;
}
```
**Explanation:** Because the list is sorted, any duplicate of the current node must be its immediate neighbor — no need to scan ahead further. Just compare each node to the very next one and unlink matches. Time: **O(n)**, Space: **O(1)**.

---
## Category 3: Stack & Queue

### Q21. Valid Parentheses
**Problem:** Given a string of brackets `()[]{}`, determine if every bracket is properly closed and nested.
**Example:** `"{[()]}"` → `true`; `"{[(])}"` → `false`.
**Frequency:** Extremely common — checking balanced expressions is also directly relevant to SDETs who parse JSON/XML.

**Brute Force** — repeatedly find and remove any adjacent matching pair `()`, `[]`, `{}` until nothing changes:
```java
public boolean isValidBrute(String s) {
    boolean changed = true;
    while (changed) {
        changed = false;
        String before = s;
        s = s.replace("()", "").replace("[]", "").replace("{}", "");
        if (!s.equals(before)) changed = true;
    }
    return s.isEmpty();
}
```
Time: **O(n²)** — repeated string rebuilding is slow and wasteful.

**Better Solution** — Stack:
```java
public boolean isValidOptimal(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);
        } else { // it's a closing bracket
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) return false;
        }
    }
    return stack.isEmpty();
}
```
**Explanation:** Push every opening bracket onto a stack. When you see a closing bracket, it must match whatever is currently on top of the stack — if it doesn't (or the stack is empty), the string is invalid. At the end, the stack must be empty (nothing left unmatched). Time: **O(n)**, Space: **O(n)**.

---

### Q22. Implement a Queue Using Two Stacks
**Problem:** Implement FIFO queue behavior (`enqueue`/`dequeue`) using only stack operations (`push`/`pop`).
**Example:** enqueue 1, 2, 3 → dequeue returns 1, then 2, then 3 (first in, first out).

**Brute Force** — use a single stack and reverse it into a second stack on *every* dequeue call:
```java
class QueueUsingStacksBrute {
    Deque<Integer> stack = new ArrayDeque<>();
    void enqueue(int x) { stack.push(x); }
    int dequeue() {
        Deque<Integer> temp = new ArrayDeque<>();
        while (!stack.isEmpty()) temp.push(stack.pop());
        int result = temp.pop();
        while (!temp.isEmpty()) stack.push(temp.pop());
        return result;
    }
}
```
Every `dequeue()` costs **O(n)** because it fully reverses the stack and rebuilds it.

**Better Solution** — two stacks, only reverse when needed (amortized O(1)):
```java
class QueueUsingStacksOptimal {
    Deque<Integer> inStack = new ArrayDeque<>();
    Deque<Integer> outStack = new ArrayDeque<>();

    void enqueue(int x) { inStack.push(x); }

    int dequeue() {
        if (outStack.isEmpty()) {
            while (!inStack.isEmpty()) outStack.push(inStack.pop());
        }
        return outStack.pop();
    }
}
```
**Explanation:** Keep two stacks: one for incoming elements, one for outgoing. Only move elements from `inStack` to `outStack` when `outStack` runs dry — each element gets moved at most once, so the *total* work across many operations stays linear. This is called "amortized O(1)" — individual calls can occasionally cost more, but averaged over many calls it's cheap. Space: **O(n)**.

---

### Q23. Min Stack
**Problem:** Design a stack that supports `push`, `pop`, `top`, and retrieving the minimum element — all in O(1).
**Example:** push 3, push 5, push 2 → `getMin()` returns `2`.

**Brute Force** — plain stack, scan the whole stack for the minimum whenever asked:
```java
class MinStackBrute {
    Deque<Integer> stack = new ArrayDeque<>();
    void push(int x) { stack.push(x); }
    void pop() { stack.pop(); }
    int top() { return stack.peek(); }
    int getMin() {
        int min = Integer.MAX_VALUE;
        for (int val : stack) min = Math.min(min, val); // O(n) scan every time
        return min;
    }
}
```
`getMin()` is **O(n)** every single call — too slow if called frequently.

**Better Solution** — a second stack that tracks the running minimum:
```java
class MinStackOptimal {
    Deque<Integer> stack = new ArrayDeque<>();
    Deque<Integer> minStack = new ArrayDeque<>();

    void push(int x) {
        stack.push(x);
        int currentMin = minStack.isEmpty() ? x : Math.min(x, minStack.peek());
        minStack.push(currentMin);
    }
    void pop() { stack.pop(); minStack.pop(); }
    int top() { return stack.peek(); }
    int getMin() { return minStack.peek(); }
}
```
**Explanation:** Every time you push a value onto the main stack, also push the *smallest value seen so far* onto a parallel `minStack`. So `minStack.peek()` always instantly tells you the current minimum, and popping keeps both stacks in sync. Every operation is **O(1)** time, Space: **O(n)** for the extra stack.

---

### Q24. Next Greater Element
**Problem:** For each element in an array, find the next element to its right that is greater than it (or `-1` if none exists).
**Example:** `[4, 5, 2, 10, 8]` → `[5, 10, 10, -1, -1]`.

**Brute Force** — for each element, scan rightward until a bigger one is found:
```java
public int[] nextGreaterBrute(int[] nums) {
    int[] result = new int[nums.length];
    for (int i = 0; i < nums.length; i++) {
        result[i] = -1;
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[j] > nums[i]) {
                result[i] = nums[j];
                break;
            }
        }
    }
    return result;
}
```
Time: **O(n²)**.

**Better Solution** — monotonic stack:
```java
public int[] nextGreaterOptimal(int[] nums) {
    int[] result = new int[nums.length];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // stores indices
    for (int i = 0; i < nums.length; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i]; // found the "next greater" for that index
        }
        stack.push(i);
    }
    return result;
}
```
**Explanation:** Keep a stack of indices whose "next greater" hasn't been found yet. When the current number is bigger than the number at the top index of the stack, that current number IS the answer for that index — pop it and record it. Every index is pushed once and popped once, so total work is Time: **O(n)**, Space: **O(n)**.

---

### Q25. Evaluate Reverse Polish Notation
**Problem:** Evaluate an arithmetic expression given in postfix (Reverse Polish) notation.
**Example:** `["2","1","+","3","*"]` → `9` (because `(2 + 1) * 3 = 9`).

**Brute Force** — repeatedly scan the token list left to right, evaluating the first operator found and replacing it and its two operands with the result, then rescanning from the start:
```java
public int evalRPNBrute(List<String> tokens) {
    tokens = new ArrayList<>(tokens);
    while (tokens.size() > 1) {
        for (int i = 0; i < tokens.size(); i++) {
            if (isOperator(tokens.get(i))) {
                int b = Integer.parseInt(tokens.get(i - 1));
                int a = Integer.parseInt(tokens.get(i - 2));
                int res = apply(a, b, tokens.get(i));
                tokens.subList(i - 2, i + 1).clear();
                tokens.add(i - 2, String.valueOf(res));
                break; // restart the scan
            }
        }
    }
    return Integer.parseInt(tokens.get(0));
}
private boolean isOperator(String t) { return t.equals("+")||t.equals("-")||t.equals("*")||t.equals("/"); }
private int apply(int a, int b, String op) {
    switch (op) { case "+": return a+b; case "-": return a-b; case "*": return a*b; default: return a/b; }
}
```
Time: **O(n²)** due to repeated list restructuring and rescanning.

**Better Solution** — Stack, single left-to-right pass:
```java
public int evalRPNOptimal(List<String> tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (String token : tokens) {
        if (token.equals("+") || token.equals("-") || token.equals("*") || token.equals("/")) {
            int b = stack.pop();
            int a = stack.pop();
            switch (token) {
                case "+": stack.push(a + b); break;
                case "-": stack.push(a - b); break;
                case "*": stack.push(a * b); break;
                case "/": stack.push(a / b); break;
            }
        } else {
            stack.push(Integer.parseInt(token));
        }
    }
    return stack.pop();
}
```
**Explanation:** Postfix notation is *made* for a stack: push numbers as you see them; when you hit an operator, pop the two most recent numbers, apply the operator, and push the result back. By the end, exactly one number remains — the answer. Time: **O(n)**, Space: **O(n)**.

---

### Q26. Sliding Window Maximum
**Problem:** Given an array and a window size `k`, return the maximum value in every window as it slides across the array.
**Example:** `[1,3,-1,-3,5,3,6,7]`, `k=3` → `[3,3,5,5,6,7]`.

**Brute Force** — for every window, scan all `k` elements to find the max:
```java
public int[] maxSlidingWindowBrute(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    for (int i = 0; i + k <= n; i++) {
        int max = Integer.MIN_VALUE;
        for (int j = i; j < i + k; j++) max = Math.max(max, nums[j]);
        result[i] = max;
    }
    return result;
}
```
Time: **O(n*k)**.

**Better Solution** — Deque holding indices in decreasing order of value:
```java
public int[] maxSlidingWindowOptimal(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // stores indices, values decreasing
    for (int i = 0; i < n; i++) {
        while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
            deque.pollFirst(); // remove index that fell out of the window
        }
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
            deque.pollLast(); // remove smaller values, they can never be the max now
        }
        deque.offerLast(i);
        if (i >= k - 1) result[i - k + 1] = nums[deque.peekFirst()];
    }
    return result;
}
```
**Explanation:** The deque's front always holds the index of the current window's maximum. Before adding a new number, throw away any smaller numbers at the back of the deque — they can never win against a larger, more recent number while it's still in the window. Also drop the front if it has slid out of the window. Every index is added and removed at most once. Time: **O(n)**, Space: **O(k)**.

---
## Category 4: Trees

*Shared node definition used in this section:*
```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

### Q27. Inorder / Preorder / Postorder Traversal
**Problem:** Visit every node of a binary tree in a specific order. Inorder = left, root, right. Preorder = root, left, right. Postorder = left, right, root.
**Example tree:** `1` root, left child `2`, right child `3` → Inorder: `[2,1,3]`, Preorder: `[1,2,3]`, Postorder: `[2,3,1]`.

**Brute Force** — this is naturally a recursive problem; a "brute force" here is doing it *iteratively* with manual stack management before you learn recursion, which is more error-prone for a beginner:
```java
public List<Integer> inorderIterativeBrute(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode cur = root;
    while (cur != null || !stack.isEmpty()) {
        while (cur != null) { stack.push(cur); cur = cur.left; }
        cur = stack.pop();
        result.add(cur.val);
        cur = cur.right;
    }
    return result;
}
```
This *works* and is actually O(n) already, but it's harder to read/write correctly than the recursive version — many beginners get the stack logic wrong under interview pressure.

**Better Solution** — plain recursion (what interviewers actually expect first):
```java
public List<Integer> inorderOptimal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    inorderHelper(root, result);
    return result;
}
private void inorderHelper(TreeNode node, List<Integer> result) {
    if (node == null) return;
    inorderHelper(node.left, result);
    result.add(node.val);
    inorderHelper(node.right, result);
}
```
**Explanation:** A tree is a recursive structure — every subtree is itself a smaller tree. The base case is a `null` node (do nothing). For inorder: recurse left, visit the node, recurse right. Swap the order of those three lines to get preorder (visit, left, right) or postorder (left, right, visit). Time: **O(n)**, Space: **O(h)** for the recursion stack, where h = tree height.

---

### Q28. Level Order Traversal (BFS)
**Problem:** Return the values of a binary tree level by level, left to right.
**Example:** tree `1` with children `2,3` → `[[1],[2,3]]`.

**Brute Force** — recursively compute the height first, then do a separate pass per level (re-traversing the whole tree once per level):
```java
public List<List<Integer>> levelOrderBrute(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    int height = getHeight(root);
    for (int level = 0; level < height; level++) {
        List<Integer> currentLevel = new ArrayList<>();
        collectLevel(root, level, currentLevel);
        result.add(currentLevel);
    }
    return result;
}
private int getHeight(TreeNode node) {
    if (node == null) return 0;
    return 1 + Math.max(getHeight(node.left), getHeight(node.right));
}
private void collectLevel(TreeNode node, int level, List<Integer> out) {
    if (node == null) return;
    if (level == 0) { out.add(node.val); return; }
    collectLevel(node.left, level - 1, out);
    collectLevel(node.right, level - 1, out);
}
```
Time: **O(n * h)** because the tree is re-walked once per level.

**Better Solution** — Queue-based BFS, single pass:
```java
public List<List<Integer>> levelOrderOptimal(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> currentLevel = new ArrayList<>();
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            currentLevel.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(currentLevel);
    }
    return result;
}
```
**Explanation:** A Queue naturally processes things in the order they were added (FIFO) — perfect for level-by-level visiting. `levelSize` snapshots how many nodes are in the *current* level before you start adding next-level children, so you know exactly when one level ends and the next begins. Time: **O(n)**, Space: **O(n)**.

---

### Q29. Maximum Depth of Binary Tree
**Problem:** Find the height (max depth) of a binary tree.
**Example:** tree `1->2->3(leaf)` (a chain) → depth `3`.

**Brute Force** — level-order BFS counting levels (works, but is more code than needed for such a simple property):
```java
public int maxDepthBrute(TreeNode root) {
    if (root == null) return 0;
    int depth = 0;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        depth++;
    }
    return depth;
}
```
Time: O(n), Space: O(n) — correct but heavier machinery than necessary.

**Better Solution** — simple recursion:
```java
public int maxDepthOptimal(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepthOptimal(root.left), maxDepthOptimal(root.right));
}
```
**Explanation:** The depth of a tree is `1` (for the current node) plus whichever of its two subtrees is deeper. The recursion bottoms out at `null`, which has depth `0`. Time: **O(n)**, Space: **O(h)** for the call stack — much simpler code, same or better efficiency.

---

### Q30. Validate Binary Search Tree
**Problem:** Check whether a binary tree satisfies the BST property (left subtree < node < right subtree, for every node).
**Example:** `[5,1,4,null,null,3,6]` (node 4's left child is 3, which is less than root 5 but must be ≥ 4) → `false`.

**Brute Force** — for every node, recursively collect all values in its left and right subtrees, and manually check every one against the node's value:
```java
public boolean isValidBSTBrute(TreeNode root) {
    if (root == null) return true;
    List<Integer> leftVals = new ArrayList<>();
    List<Integer> rightVals = new ArrayList<>();
    collect(root.left, leftVals);
    collect(root.right, rightVals);
    for (int v : leftVals) if (v >= root.val) return false;
    for (int v : rightVals) if (v <= root.val) return false;
    return isValidBSTBrute(root.left) && isValidBSTBrute(root.right);
}
private void collect(TreeNode node, List<Integer> out) {
    if (node == null) return;
    out.add(node.val);
    collect(node.left, out);
    collect(node.right, out);
}
```
Time: **O(n²)** — re-collecting subtree values at every node is very wasteful.

**Better Solution** — recursion with a valid (min, max) range passed down:
```java
public boolean isValidBSTOptimal(TreeNode root) {
    return validate(root, null, null);
}
private boolean validate(TreeNode node, Integer min, Integer max) {
    if (node == null) return true;
    if ((min != null && node.val <= min) || (max != null && node.val >= max)) {
        return false;
    }
    return validate(node.left, min, node.val) && validate(node.right, node.val, max);
}
```
**Explanation:** Every node doesn't just need to be bigger than its immediate left child — it needs to be bigger than *everything* in its entire left subtree, and smaller than everything in its entire right subtree. Pass down a shrinking `(min, max)` valid range as you recurse: going left tightens the upper bound to the parent's value; going right tightens the lower bound. Time: **O(n)** — each node visited once, Space: **O(h)**.

---

### Q31. Lowest Common Ancestor (in a BST)
**Problem:** Find the lowest node that has both given nodes as descendants.
**Example:** BST with root `6`, nodes `2` and `8` → LCA is `6`.

**Brute Force** — find the full path from root to each node, then compare the paths to find where they last agree:
```java
public TreeNode lcaBrute(TreeNode root, TreeNode p, TreeNode q) {
    List<TreeNode> pathP = new ArrayList<>(), pathQ = new ArrayList<>();
    findPath(root, p.val, pathP);
    findPath(root, q.val, pathQ);
    TreeNode lca = null;
    for (int i = 0; i < Math.min(pathP.size(), pathQ.size()); i++) {
        if (pathP.get(i) == pathQ.get(i)) lca = pathP.get(i);
        else break;
    }
    return lca;
}
private boolean findPath(TreeNode node, int target, List<TreeNode> path) {
    if (node == null) return false;
    path.add(node);
    if (node.val == target) return true;
    if ((target < node.val && findPath(node.left, target, path)) ||
        (target > node.val && findPath(node.right, target, path))) return true;
    path.remove(path.size() - 1);
    return false;
}
```
Time: **O(h)** but does noticeably more work (two full path-builds plus a comparison) than necessary.

**Better Solution** — use the BST ordering property directly, single pass:
```java
public TreeNode lcaOptimal(TreeNode root, TreeNode p, TreeNode q) {
    TreeNode cur = root;
    while (cur != null) {
        if (p.val < cur.val && q.val < cur.val) {
            cur = cur.left;
        } else if (p.val > cur.val && q.val > cur.val) {
            cur = cur.right;
        } else {
            return cur; // p and q split here (or one of them IS cur) -> found LCA
        }
    }
    return null;
}
```
**Explanation:** In a BST, if both target values are smaller than the current node, the LCA must be in the left subtree; if both are bigger, it must be in the right subtree. The moment they're *not* both on the same side, you've found the split point — that's the LCA. Time: **O(h)** where h = tree height, Space: **O(1)**.

---

### Q32. Diameter of Binary Tree
**Problem:** Find the length of the longest path between any two nodes in a tree (path may or may not pass through the root).
**Example:** a tree where the longest path has 4 edges → diameter `4`.

**Brute Force** — for every node, compute the height of its left and right subtrees separately (recomputing height repeatedly):
```java
public int diameterBrute(TreeNode root) {
    if (root == null) return 0;
    int throughRoot = height(root.left) + height(root.right);
    int leftDia = diameterBrute(root.left);
    int rightDia = diameterBrute(root.right);
    return Math.max(throughRoot, Math.max(leftDia, rightDia));
}
private int height(TreeNode node) {
    if (node == null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
}
```
Time: **O(n²)** — `height()` is recalculated from scratch at every node.

**Better Solution** — compute height and track diameter in the same single pass:
```java
private int maxDiameter = 0;

public int diameterOptimal(TreeNode root) {
    maxDiameter = 0;
    heightAndUpdate(root);
    return maxDiameter;
}
private int heightAndUpdate(TreeNode node) {
    if (node == null) return 0;
    int leftHeight = heightAndUpdate(node.left);
    int rightHeight = heightAndUpdate(node.right);
    maxDiameter = Math.max(maxDiameter, leftHeight + rightHeight);
    return 1 + Math.max(leftHeight, rightHeight);
}
```
**Explanation:** Every recursive call already computes the height of a subtree — reuse that same call to *also* update a running "best diameter found so far" (`leftHeight + rightHeight` = longest path passing through this node). No need for a separate height function called repeatedly. Time: **O(n)**, Space: **O(h)**.

---

### Q33. Check if Two Trees are Symmetric (Mirror Images)
**Problem:** Determine if a binary tree is a mirror of itself around its center.
**Example:** `1` with children `2,2` where each `2` has children `3,4` and `4,3` respectively → `true`.

**Brute Force** — flatten both the left and right subtrees into lists via traversal, then check if one list is the exact reverse-with-null-markers of the other:
```java
public boolean isSymmetricBrute(TreeNode root) {
    if (root == null) return true;
    List<Integer> leftList = new ArrayList<>();
    List<Integer> rightList = new ArrayList<>();
    flatten(root.left, leftList, true);
    flatten(root.right, rightList, false);
    return leftList.equals(rightList);
}
private void flatten(TreeNode node, List<Integer> out, boolean leftFirst) {
    if (node == null) { out.add(null); return; }
    out.add(node.val);
    if (leftFirst) { flatten(node.left, out, true); flatten(node.right, out, true); }
    else { flatten(node.right, out, false); flatten(node.left, out, false); }
}
```
This works but builds two full extra lists just to compare them — extra memory for no real benefit.

**Better Solution** — recursively compare the two subtrees as mirrors directly:
```java
public boolean isSymmetricOptimal(TreeNode root) {
    if (root == null) return true;
    return isMirror(root.left, root.right);
}
private boolean isMirror(TreeNode t1, TreeNode t2) {
    if (t1 == null && t2 == null) return true;
    if (t1 == null || t2 == null) return false;
    return t1.val == t2.val
        && isMirror(t1.left, t2.right)
        && isMirror(t1.right, t2.left);
}
```
**Explanation:** Two trees are mirrors if their root values match, AND the left subtree of one is a mirror of the right subtree of the other (and vice versa) — check that recursively without ever materializing an intermediate list. Time: **O(n)**, Space: **O(h)**.

---

### Q34. Convert Sorted Array to a Balanced BST
**Problem:** Given a sorted array, build a height-balanced binary search tree from it.
**Example:** `[-10,-3,0,5,9]` → a balanced BST with `0` as root.

**Brute Force** — insert elements one at a time into an initially-empty BST using standard BST insertion (in array order, left to right):
```java
public TreeNode sortedArrayToBSTBrute(int[] nums) {
    TreeNode root = null;
    for (int num : nums) root = insert(root, num);
    return root;
}
private TreeNode insert(TreeNode node, int val) {
    if (node == null) return new TreeNode(val);
    if (val < node.val) node.left = insert(node.left, val);
    else node.right = insert(node.right, val);
    return node;
}
```
Since the array is sorted, inserting left-to-right produces a completely *unbalanced* tree (essentially a linked list!) — Time: **O(n²)** worst case, and it doesn't even satisfy the "balanced" requirement.

**Better Solution** — always pick the middle element as root, recursively:
```java
public TreeNode sortedArrayToBSTOptimal(int[] nums) {
    return build(nums, 0, nums.length - 1);
}
private TreeNode build(int[] nums, int lo, int hi) {
    if (lo > hi) return null;
    int mid = lo + (hi - lo) / 2;
    TreeNode node = new TreeNode(nums[mid]);
    node.left = build(nums, lo, mid - 1);
    node.right = build(nums, mid + 1, hi);
    return node;
}
```
**Explanation:** Picking the middle element as the root guarantees roughly equal numbers of elements go left and right, which is exactly what "balanced" means. Recurse on the left half and right half the same way. Time: **O(n)**, Space: **O(log n)** recursion depth.

---

### Q35. Path Sum (Root to Leaf)
**Problem:** Determine if the tree has a root-to-leaf path that adds up to a given target sum.
**Example:** tree `5->4->11->7` (leaf) with `targetSum=22`: `5+4+11+2=22` for another branch → `true`.

**Brute Force** — collect every complete root-to-leaf path into a list of sums, then check if the target is among them:
```java
public boolean hasPathSumBrute(TreeNode root, int targetSum) {
    List<Integer> allSums = new ArrayList<>();
    collectPathSums(root, 0, allSums);
    return allSums.contains(targetSum);
}
private void collectPathSums(TreeNode node, int currentSum, List<Integer> out) {
    if (node == null) return;
    currentSum += node.val;
    if (node.left == null && node.right == null) out.add(currentSum); // leaf reached
    collectPathSums(node.left, currentSum, out);
    collectPathSums(node.right, currentSum, out);
}
```
This visits every node correctly (O(n)) but wastes memory building a full list of *all* path sums, when you only need a yes/no answer and can stop as soon as one match is found.

**Better Solution** — recursion with early exit:
```java
public boolean hasPathSumOptimal(TreeNode root, int targetSum) {
    if (root == null) return false;
    if (root.left == null && root.right == null) { // leaf node
        return targetSum == root.val;
    }
    int remaining = targetSum - root.val;
    return hasPathSumOptimal(root.left, remaining) || hasPathSumOptimal(root.right, remaining);
}
```
**Explanation:** Subtract the current node's value from the target as you go down. At a leaf, check if exactly `0` remains (i.e., the leaf's value equals what's left of the target). Using `||` short-circuits — once one branch returns `true`, the other isn't even explored. Time: **O(n)** worst case but often less due to early exit, Space: **O(h)**.

---

### Q36. Serialize and Deserialize a Binary Tree
**Problem:** Convert a binary tree to a string, and be able to reconstruct the exact same tree from that string.
**Example:** tree `1->2,3` → serialized as something like `"1,2,null,null,3,null,null"`; deserializing it rebuilds the identical tree.

**Brute Force** — serialize using level-order (BFS) with explicit index bookkeeping to locate children, which gets fiddly with `null` placeholders and index math:
```java
public String serializeBrute(TreeNode root) {
    List<String> vals = new ArrayList<>();
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        TreeNode node = queue.poll();
        if (node == null) { vals.add("null"); continue; }
        vals.add(String.valueOf(node.val));
        queue.offer(node.left);
        queue.offer(node.right);
    }
    return String.join(",", vals);
}
// Deserializing this level-order format correctly requires careful index tracking
// to know which queued placeholder corresponds to which parent's child slot —
// this is doable but noticeably more error-prone to implement correctly than preorder.
```
Level-order serialization works, but reconstructing children-to-parent relationships from a flat BFS list is trickier to get right under interview time pressure.

**Better Solution** — preorder (root, left, right) with explicit "null" markers, using a Queue as the read pointer during deserialization:
```java
public String serializeOptimal(TreeNode root) {
    StringBuilder sb = new StringBuilder();
    buildString(root, sb);
    return sb.toString();
}
private void buildString(TreeNode node, StringBuilder sb) {
    if (node == null) { sb.append("null,"); return; }
    sb.append(node.val).append(",");
    buildString(node.left, sb);
    buildString(node.right, sb);
}

public TreeNode deserializeOptimal(String data) {
    Queue<String> tokens = new LinkedList<>(Arrays.asList(data.split(",")));
    return buildTree(tokens);
}
private TreeNode buildTree(Queue<String> tokens) {
    String val = tokens.poll();
    if (val.equals("null")) return null;
    TreeNode node = new TreeNode(Integer.parseInt(val));
    node.left = buildTree(tokens);
    node.right = buildTree(tokens);
    return node;
}
```
**Explanation:** Preorder naturally mirrors how you'd *rebuild* the tree: read a value, that's the root; recursively read what comes next as the left subtree, then the right subtree. Explicit `"null"` markers tell the deserializer exactly where each subtree ends, so no index math is needed — just keep polling tokens off the queue in the same order they were written. Time: **O(n)** for both operations, Space: **O(n)**.

---
## Category 5: Graphs

*Graphs are commonly represented as an adjacency list: `Map<Integer, List<Integer>>` or `List<List<Integer>>`.*

### Q37. BFS Traversal of a Graph
**Problem:** Visit every node reachable from a start node, level by level.
**Example:** graph `0-1, 0-2, 1-2, 2-3`, start `2` → visits in order `2, 0, 3, 1` (order depends on adjacency list order).

**Brute Force** — repeatedly scan the *entire* visited set and edge list to find the next unvisited neighbor, instead of using a proper queue:
```java
public List<Integer> bfsBrute(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> result = new ArrayList<>();
    Set<Integer> visited = new HashSet<>();
    List<Integer> frontier = new ArrayList<>();
    frontier.add(start);
    visited.add(start);
    while (!frontier.isEmpty()) {
        List<Integer> nextFrontier = new ArrayList<>();
        for (int node : frontier) {
            result.add(node);
            for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    nextFrontier.add(neighbor); // may add duplicates within same frontier
                }
            }
        }
        frontier = nextFrontier;
    }
    return result;
}
```
This mostly works but manually managing "frontiers" as lists is more error-prone (easy to introduce duplicate processing) than just using a Queue.

**Better Solution** — standard Queue-based BFS:
```java
public List<Integer> bfsOptimal(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> result = new ArrayList<>();
    Set<Integer> visited = new HashSet<>();
    Queue<Integer> queue = new LinkedList<>();
    queue.offer(start);
    visited.add(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
    return result;
}
```
**Explanation:** A Queue is the textbook tool for BFS: process the current node, add all its *unvisited* neighbors to the back of the queue, then move to the next node in the queue (always the "oldest" one added). The `visited` set prevents infinite loops on cyclic graphs. Time: **O(V + E)**, Space: **O(V)**.

---

### Q38. DFS Traversal of a Graph
**Problem:** Visit every node reachable from a start node, going as deep as possible before backtracking.
**Example:** same graph as Q37, start `2` → visits in depth-first order, e.g. `2, 0, 1, 3`.

**Brute Force** — manual stack management done clumsily, re-checking the visited set for the entire neighbor list every time instead of pushing individual neighbors cleanly:
```java
public List<Integer> dfsBrute(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> result = new ArrayList<>();
    Set<Integer> visited = new HashSet<>();
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited.contains(node)) continue; // duplicates can pile up in the stack
        visited.add(node);
        result.add(node);
        for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
            stack.push(neighbor); // pushes even already-visited neighbors
        }
    }
    return result;
}
```
Functionally correct, but pushes far more entries onto the stack than necessary (unvisited check happens too late).

**Better Solution** — clean recursive DFS:
```java
public List<Integer> dfsOptimal(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> result = new ArrayList<>();
    Set<Integer> visited = new HashSet<>();
    dfsHelper(graph, start, visited, result);
    return result;
}
private void dfsHelper(Map<Integer, List<Integer>> graph, int node, Set<Integer> visited, List<Integer> result) {
    if (visited.contains(node)) return;
    visited.add(node);
    result.add(node);
    for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
        dfsHelper(graph, neighbor, visited, result);
    }
}
```
**Explanation:** Recursion naturally models "go deep, then backtrack" — visit a node, mark it visited, then recursively dive into each unvisited neighbor before returning. Time: **O(V + E)**, Space: **O(V)** for the visited set plus recursion stack.

---

### Q39. Number of Islands
**Problem:** Given a 2D grid of `'1'` (land) and `'0'` (water), count the number of islands (connected groups of land, horizontally/vertically adjacent).
**Example:**
```
11000
11000
00100
00011
```
→ `3` islands.

**Brute Force** — for every land cell, do a full grid scan to check whether it belongs to an island already counted (tracking which cells have been "claimed" without proper flood fill), which is inefficient and awkward to get right:
```java
// A truly naive approach recomputes connectivity checks per cell, which
// is both slow and fiddly. In practice, for this problem, flood-fill IS
// the "textbook brute force" that most beginners reach for first —
// there isn't a simpler naive version worth writing badly on purpose.
// So here, "brute force" = flood fill using recursion (DFS), which also
// happens to be the standard efficient answer.
```

**Better Solution** — flood-fill (DFS) each unvisited land cell, sinking the whole island as you go:
```java
public int numIslandsOptimal(char[][] grid) {
    int count = 0;
    for (int r = 0; r < grid.length; r++) {
        for (int c = 0; c < grid[0].length; c++) {
            if (grid[r][c] == '1') {
                count++;
                sink(grid, r, c); // mark this whole island as visited
            }
        }
    }
    return count;
}
private void sink(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length || grid[r][c] != '1') {
        return;
    }
    grid[r][c] = '0'; // mark visited by "sinking" it
    sink(grid, r + 1, c);
    sink(grid, r - 1, c);
    sink(grid, r, c + 1);
    sink(grid, r, c - 1);
}
```
**Explanation:** Scan every cell; whenever you find unvisited land (`'1'`), that's a *new* island — increment the count, then use DFS to "sink" (mark visited) every connected land cell so it's never counted again. Time: **O(rows × cols)**, Space: **O(rows × cols)** worst case for the recursion stack.

---

### Q40. Detect Cycle in a Directed Graph
**Problem:** Determine if a directed graph contains a cycle.
**Example:** edges `0->1, 1->2, 2->0` → `true` (cycle exists).

**Brute Force** — for every node, try to find a path back to itself using a fresh BFS/DFS each time:
```java
public boolean hasCycleBrute(Map<Integer, List<Integer>> graph, int n) {
    for (int node = 0; node < n; node++) {
        if (canReachSelf(graph, node, node, new HashSet<>())) return true;
    }
    return false;
}
private boolean canReachSelf(Map<Integer, List<Integer>> graph, int target, int current, Set<Integer> visited) {
    for (int neighbor : graph.getOrDefault(current, new ArrayList<>())) {
        if (neighbor == target) return true;
        if (!visited.contains(neighbor)) {
            visited.add(neighbor);
            if (canReachSelf(graph, target, neighbor, visited)) return true;
        }
    }
    return false;
}
```
Time: **O(V * (V+E))** — running a full search from every single node is very redundant.

**Better Solution** — single DFS pass tracking nodes currently "in progress" (recursion stack):
```java
public boolean hasCycleOptimal(Map<Integer, List<Integer>> graph, int n) {
    int[] state = new int[n]; // 0 = unvisited, 1 = in progress, 2 = fully done
    for (int i = 0; i < n; i++) {
        if (state[i] == 0 && dfs(graph, i, state)) return true;
    }
    return false;
}
private boolean dfs(Map<Integer, List<Integer>> graph, int node, int[] state) {
    state[node] = 1; // mark as "in progress" (currently on the recursion stack)
    for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
        if (state[neighbor] == 1) return true;  // found a back-edge -> cycle!
        if (state[neighbor] == 0 && dfs(graph, neighbor, state)) return true;
    }
    state[node] = 2; // done exploring, safe
    return false;
}
```
**Explanation:** A cycle in a directed graph exists exactly when DFS finds an edge pointing back to a node that's still "in progress" (i.e., currently an ancestor in the current DFS path) — this is called a "back edge." Marking nodes as `1` (in progress) vs `2` (fully finished) instead of just a boolean `visited` is the key trick. Time: **O(V + E)**, single pass.

---

### Q41. Clone a Graph
**Problem:** Given a reference to a node in a connected undirected graph, return a deep copy (clone) of the entire graph.
**Example:** graph `1-2-3-1` (triangle) → return an identical but entirely new set of node objects with the same connections.

**Brute Force** — do a first pass to clone all nodes *without* connections, storing them in a list, then a second pass matching up connections by searching the list for the right cloned node each time:
```java
// Requires manually keeping parallel lists/indices in sync between originals
// and clones, then a separate O(V) search per edge to find "which clone is this" —
// clunky and easy to get wrong without a direct lookup.
```
This approach is workable but requires linear searches to match clones to originals — unnecessarily slow, Time: **O(V² )** roughly.

**Better Solution** — DFS/BFS with a HashMap from original node → cloned node:
```java
class GraphNode {
    int val;
    List<GraphNode> neighbors = new ArrayList<>();
    GraphNode(int val) { this.val = val; }
}

public GraphNode cloneGraphOptimal(GraphNode node) {
    return clone(node, new HashMap<>());
}
private GraphNode clone(GraphNode node, Map<GraphNode, GraphNode> visited) {
    if (node == null) return null;
    if (visited.containsKey(node)) return visited.get(node); // already cloned
    GraphNode copy = new GraphNode(node.val);
    visited.put(node, copy);
    for (GraphNode neighbor : node.neighbors) {
        copy.neighbors.add(clone(neighbor, visited));
    }
    return copy;
}
```
**Explanation:** The HashMap acts as your "have I already cloned this node?" lookup — the moment you clone a node, record it in the map immediately (before recursing into its neighbors) so that cyclic references don't cause infinite recursion. Every node is cloned exactly once. Time: **O(V + E)**, Space: **O(V)**.

---

### Q42. Topological Sort
**Problem:** Given a Directed Acyclic Graph (DAG), order the nodes so that every edge `u->v` has `u` appearing before `v`.
**Example:** edges `5->0, 4->0, 5->2, 2->3, 3->1` → one valid order: `4, 5, 0, 2, 3, 1`.

**Brute Force** — repeatedly scan all nodes looking for one with no remaining incoming edges, remove it, and repeat from scratch (re-scanning everything each round):
```java
public List<Integer> topSortBrute(Map<Integer, List<Integer>> graph, int n) {
    List<Integer> result = new ArrayList<>();
    Set<Integer> removed = new HashSet<>();
    while (result.size() < n) {
        int found = -1;
        for (int node = 0; node < n; node++) {
            if (removed.contains(node)) continue;
            boolean hasIncoming = false;
            for (List<Integer> edges : graph.values()) {
                if (edges.contains(node) && !removed.contains(edges.indexOf(node))) { hasIncoming = true; break; }
            }
            if (!hasIncoming) { found = node; break; }
        }
        result.add(found);
        removed.add(found);
    }
    return result;
}
```
Time: **O(V² * E)** roughly — recomputing "incoming edges" from scratch every round is very wasteful.

**Better Solution** — Kahn's Algorithm (BFS with in-degree counting):
```java
public List<Integer> topSortOptimal(Map<Integer, List<Integer>> graph, int n) {
    int[] inDegree = new int[n];
    for (List<Integer> neighbors : graph.values()) {
        for (int neighbor : neighbors) inDegree[neighbor]++;
    }
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++) {
        if (inDegree[i] == 0) queue.offer(i); // nodes with no dependencies first
    }
    List<Integer> result = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
            inDegree[neighbor]--;
            if (inDegree[neighbor] == 0) queue.offer(neighbor); // now free to process
        }
    }
    return result;
}
```
**Explanation:** Precompute how many incoming edges ("in-degree") each node has. Start with nodes that have zero incoming edges — they can safely go first. As you "remove" a processed node, decrement its neighbors' in-degree; the moment a neighbor's in-degree hits zero, it's ready to be processed too. Time: **O(V + E)**, Space: **O(V)**.

---

### Q43. Course Schedule (Can All Courses Be Finished?)
**Problem:** Given `numCourses` and a list of prerequisite pairs `[a, b]` (must take `b` before `a`), determine if it's possible to finish all courses.
**Example:** `numCourses=2`, `prerequisites=[[1,0]]` → `true` (take course 0, then course 1).

**Brute Force** — for each course, recursively try to "take" its prerequisites first, without tracking in-progress state, risking infinite recursion on cycles unless carefully guarded, and redoing work for shared prerequisites:
```java
public boolean canFinishBrute(int numCourses, int[][] prerequisites) {
    Map<Integer, List<Integer>> graph = new HashMap<>();
    for (int[] p : prerequisites) {
        graph.computeIfAbsent(p[0], k -> new ArrayList<>()).add(p[1]);
    }
    for (int course = 0; course < numCourses; course++) {
        if (!canTake(course, graph, new HashSet<>())) return false;
    }
    return true;
}
private boolean canTake(int course, Map<Integer, List<Integer>> graph, Set<Integer> visiting) {
    if (visiting.contains(course)) return false; // cycle detected on this path
    visiting.add(course);
    for (int prereq : graph.getOrDefault(course, new ArrayList<>())) {
        if (!canTake(prereq, graph, visiting)) return false;
    }
    visiting.remove(course); // note: without a "done" cache, shared prereqs get re-checked repeatedly
    return true;
}
```
This is essentially correct but lacks memoization of fully-verified nodes, so shared prerequisites get re-verified many times — Time: can blow up to exponential in dense graphs without a "done" set.

**Better Solution** — this is exactly "detect a cycle in a directed graph" (Q40) — if there's no cycle, all courses can be finished:
```java
public boolean canFinishOptimal(int numCourses, int[][] prerequisites) {
    Map<Integer, List<Integer>> graph = new HashMap<>();
    for (int[] p : prerequisites) {
        graph.computeIfAbsent(p[0], k -> new ArrayList<>()).add(p[1]);
    }
    int[] state = new int[numCourses]; // 0=unvisited, 1=in-progress, 2=done
    for (int course = 0; course < numCourses; course++) {
        if (state[course] == 0 && hasCycleFrom(course, graph, state)) return false;
    }
    return true;
}
private boolean hasCycleFrom(int node, Map<Integer, List<Integer>> graph, int[] state) {
    state[node] = 1;
    for (int neighbor : graph.getOrDefault(node, new ArrayList<>())) {
        if (state[neighbor] == 1) return true;
        if (state[neighbor] == 0 && hasCycleFrom(neighbor, graph, state)) return true;
    }
    state[node] = 2;
    return false;
}
```
**Explanation:** "Can all courses be finished?" is just "is this dependency graph free of cycles?" in disguise — a classic SDET-relevant pattern, since dependency cycles show up constantly in build systems and test-suite dependencies too. Using the 3-state marking (unvisited / in-progress / done) avoids re-verifying already-confirmed-safe nodes. Time: **O(V + E)**, Space: **O(V)**.

---
## Category 6: Recursion & Backtracking

### Q44. Factorial and Fibonacci (Recursive Basics)
**Problem:** Compute `n!` and the nth Fibonacci number recursively.
**Example:** `factorial(5) = 120`; `fibonacci(6) = 8` (sequence: 0,1,1,2,3,5,8).

**Brute Force** — plain (unoptimized) recursion, most beginner-relevant here since Fibonacci especially explodes without memoization:
```java
public int factorial(int n) {
    if (n <= 1) return 1;         // base case
    return n * factorial(n - 1);  // recursive case
}

public int fibonacciBrute(int n) {
    if (n <= 1) return n;
    return fibonacciBrute(n - 1) + fibonacciBrute(n - 2); // recomputes same values repeatedly
}
```
Factorial is fine at O(n). But `fibonacciBrute` recalculates the same subproblems over and over — Time: **O(2ⁿ)**, exponential and very slow for n > ~35.

**Better Solution** — memoized (top-down DP) Fibonacci:
```java
public int fibonacciOptimal(int n) {
    return fibHelper(n, new HashMap<>());
}
private int fibHelper(int n, Map<Integer, Integer> memo) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    int result = fibHelper(n - 1, memo) + fibHelper(n - 2, memo);
    memo.put(n, result);
    return result;
}
```
**Explanation:** Store each computed Fibonacci value in a map the first time it's calculated; if asked for the same value again, return the cached answer instantly instead of recomputing the whole recursive tree. This turns exponential time into Time: **O(n)**, Space: **O(n)**. This "remember what you've already solved" idea is the foundation of all Dynamic Programming (see Category 7).

---

### Q45. Permutations of a String/Array
**Problem:** Generate all possible orderings (permutations) of a set of elements.
**Example:** `"abc"` → `["abc","acb","bac","bca","cab","cba"]`.

**Brute Force** — actually, generating *all* permutations inherently requires visiting all `n!` arrangements — there's no way to "cheat" this with a faster algorithm. The beginner mistake is doing it iteratively with manual swapping logic that's hard to reason about and easy to get wrong:
```java
// A common beginner attempt: nested loops trying to manually track "used" indices
// with complex bookkeeping — this gets unreadable fast for n > 3 and is genuinely
// harder to write correctly than the backtracking version below. For this problem,
// clean backtracking (below) IS the standard/expected approach — there isn't a
// meaningfully different "worse but simpler" version worth presenting.
```

**Better Solution** — Backtracking:
```java
public List<String> permuteOptimal(String s) {
    List<String> result = new ArrayList<>();
    backtrack(s.toCharArray(), 0, result);
    return result;
}
private void backtrack(char[] chars, int start, List<String> result) {
    if (start == chars.length) {
        result.add(new String(chars));
        return;
    }
    for (int i = start; i < chars.length; i++) {
        swap(chars, start, i);           // choose
        backtrack(chars, start + 1, result); // explore
        swap(chars, start, i);           // un-choose (backtrack)
    }
}
private void swap(char[] chars, int i, int j) {
    char temp = chars[i]; chars[i] = chars[j]; chars[j] = temp;
}
```
**Explanation:** "Backtracking" means: try a choice, recursively explore everything that follows from it, then *undo* that choice before trying the next one. Here, at each position, try swapping in every remaining character, recurse to fill the rest of the positions, then swap back to restore the original order before trying the next candidate. Time: **O(n! * n)**, Space: **O(n)** recursion depth.

---

### Q46. Subsets of a Set (Power Set)
**Problem:** Generate all possible subsets of a set of distinct numbers (including the empty set and the full set).
**Example:** `[1,2,3]` → `[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]`.

**Brute Force** — bit manipulation trick (works, but is less intuitive for beginners since it requires understanding binary representation):
```java
public List<List<Integer>> subsetsBrute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    int n = nums.length;
    for (int mask = 0; mask < (1 << n); mask++) { // try every bitmask 0..2^n-1
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) subset.add(nums[i]); // bit i set -> include nums[i]
        }
        result.add(subset);
    }
    return result;
}
```
Works and is Time: **O(n * 2ⁿ)**, but the bit-mask logic is a stretch for someone new to DSA.

**Better Solution** — Backtracking (include / exclude each element):
```java
public List<List<Integer>> subsetsOptimal(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}
private void backtrack(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current)); // current subset is always valid, add it
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                        // choose
        backtrack(nums, i + 1, current, result);      // explore
        current.remove(current.size() - 1);           // un-choose
    }
}
```
**Explanation:** Every point in the recursion represents a valid subset — add it immediately. Then try extending it with each remaining element one at a time, recurse, and remove that element again before trying the next one. This naturally generates every combination without duplicates. Time: **O(n * 2ⁿ)**, Space: **O(n)** recursion depth.

---

### Q47. Generate Valid Parentheses Combinations
**Problem:** Given `n` pairs of parentheses, generate all combinations of well-formed parentheses strings.
**Example:** `n=3` → `["((()))","(()())","(())()","()(())","()()()"]`.

**Brute Force** — generate all `2^(2n)` possible strings of `(` and `)`, then filter to keep only the valid ones (reusing Q21's validator):
```java
public List<String> generateParenthesisBrute(int n) {
    List<String> result = new ArrayList<>();
    generateAll(new char[2 * n], 0, result);
    return result;
}
private void generateAll(char[] current, int pos, List<String> result) {
    if (pos == current.length) {
        String s = new String(current);
        if (isValidOptimal(s)) result.add(s); // reuse the validator from Q21
        return;
    }
    current[pos] = '(';
    generateAll(current, pos + 1, result);
    current[pos] = ')';
    generateAll(current, pos + 1, result);
}
```
Time: **O(2^(2n) * n)** — generates and checks a huge number of invalid strings unnecessarily.

**Better Solution** — Backtracking with validity constraints built in (only add a bracket if it keeps the string valid so far):
```java
public List<String> generateParenthesisOptimal(int n) {
    List<String> result = new ArrayList<>();
    backtrack(new StringBuilder(), 0, 0, n, result);
    return result;
}
private void backtrack(StringBuilder current, int open, int close, int n, List<String> result) {
    if (current.length() == 2 * n) {
        result.add(current.toString());
        return;
    }
    if (open < n) { // can still add an opening bracket
        current.append('(');
        backtrack(current, open + 1, close, n, result);
        current.deleteCharAt(current.length() - 1);
    }
    if (close < open) { // can only add a closing bracket if it won't make the string invalid
        current.append(')');
        backtrack(current, open, close + 1, n, result);
        current.deleteCharAt(current.length() - 1);
    }
}
```
**Explanation:** Never even attempt an invalid path — only add `(` if you haven't used all `n` yet, and only add `)` if doing so wouldn't exceed the number of `(` already placed. This "smart" backtracking prunes the search tree so you only ever build valid strings. Time: **O(4ⁿ / √n)** (the Catalan number bound — much better than the brute force's `2^(2n)`), Space: **O(n)**.

---

### Q48. N-Queens Problem
**Problem:** Place `n` queens on an `n×n` chessboard so that no two queens attack each other (same row, column, or diagonal).
**Example:** `n=4` → 2 valid solutions exist.

**Brute Force** — try every possible placement of `n` queens across all `n²` squares and check validity of the whole board at the end:
```java
// Trying every combination of n squares out of n^2 total squares is
// C(n^2, n) combinations — astronomically large even for small n (n=8
// already gives billions of combinations). This is why N-Queens is
// ALWAYS taught with backtracking; a truly brute force version isn't
// practically runnable even as a teaching example.
```

**Better Solution** — Backtracking, one queen per row, with pruning:
```java
public List<List<String>> solveNQueensOptimal(int n) {
    List<List<String>> result = new ArrayList<>();
    int[] queenCols = new int[n]; // queenCols[row] = column of the queen in that row
    backtrack(0, queenCols, n, result);
    return result;
}
private void backtrack(int row, int[] queenCols, int n, List<List<String>> result) {
    if (row == n) {
        result.add(buildBoard(queenCols, n));
        return;
    }
    for (int col = 0; col < n; col++) {
        if (isSafe(queenCols, row, col)) {
            queenCols[row] = col;             // place queen
            backtrack(row + 1, queenCols, n, result); // try next row
            // no explicit "undo" needed -- queenCols[row] gets overwritten next iteration
        }
    }
}
private boolean isSafe(int[] queenCols, int row, int col) {
    for (int r = 0; r < row; r++) {
        int c = queenCols[r];
        if (c == col || Math.abs(c - col) == Math.abs(r - row)) return false; // same column or diagonal
    }
    return true;
}
private List<String> buildBoard(int[] queenCols, int n) {
    List<String> board = new ArrayList<>();
    for (int col : queenCols) {
        StringBuilder row = new StringBuilder();
        for (int c = 0; c < n; c++) row.append(c == col ? 'Q' : '.');
        board.add(row.toString());
    }
    return board;
}
```
**Explanation:** Place one queen per row (this alone eliminates same-row conflicts automatically). Before placing a queen in a given column, check it doesn't share a column or diagonal with any queen already placed in earlier rows. If a row can't be filled safely, backtrack to the previous row and try a different column there. Time: roughly **O(n!)** worst case but heavily pruned in practice, Space: **O(n)**.

---

### Q49. Combination Sum
**Problem:** Given an array of candidate numbers and a target, find all unique combinations that sum to the target (numbers can be reused).
**Example:** `candidates=[2,3,6,7]`, `target=7` → `[[2,2,3],[7]]`.

**Brute Force** — generate every possible combination length by length (trying all subsets of every possible size, then filtering by sum), which redoes a lot of work checking sums independently of the recursive structure:
```java
// A truly "brute" approach here is to generate the full power set of
// combinations-with-repetition up to a max length, then filter by sum --
// impractically large since numbers can repeat indefinitely. In practice
// even the naive approach to this problem needs backtracking with a
// sum check as a stopping condition, shown below as "brute", vs. an
// optimized version that prunes and sorts first.
public List<List<Integer>> combinationSumBrute(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    backtrackBrute(candidates, target, 0, new ArrayList<>(), result);
    return result;
}
private void backtrackBrute(int[] candidates, int remaining, int start, List<Integer> current, List<List<Integer>> result) {
    if (remaining == 0) { result.add(new ArrayList<>(current)); return; }
    if (remaining < 0) return;
    for (int i = start; i < candidates.length; i++) { // no sorting/pruning -- explores every branch even hopeless ones
        current.add(candidates[i]);
        backtrackBrute(candidates, remaining - candidates[i], i, current, result);
        current.remove(current.size() - 1);
    }
}
```
Time: exponential, and explores many branches that can never succeed (e.g., trying a candidate way bigger than what's left).

**Better Solution** — sort candidates first, then prune branches early once the remaining target goes negative:
```java
public List<List<Integer>> combinationSumOptimal(int[] candidates, int target) {
    Arrays.sort(candidates); // sorting enables early pruning below
    List<List<Integer>> result = new ArrayList<>();
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}
private void backtrack(int[] candidates, int remaining, int start, List<Integer> current, List<List<Integer>> result) {
    if (remaining == 0) { result.add(new ArrayList<>(current)); return; }
    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > remaining) break; // sorted, so all later candidates are even bigger -- stop early
        current.add(candidates[i]);
        backtrack(candidates, remaining - candidates[i], i, current, result); // i, not i+1: allow reusing same number
        current.remove(current.size() - 1);
    }
}
```
**Explanation:** Sorting the candidates first means once a candidate is too big for the remaining target, every candidate after it (being even bigger) is guaranteed too big too — `break` immediately instead of checking them all. Passing `i` (not `i+1`) into the recursive call allows the same number to be reused, while `i+1` would prevent reuse. Time: much faster in practice due to pruning, though worst case is still exponential — Space: **O(target)** recursion depth.

---
## Category 7: Dynamic Programming

### Q50. Climbing Stairs
**Problem:** You can climb 1 or 2 steps at a time. How many distinct ways can you climb `n` stairs?
**Example:** `n=4` → `5` ways.

**Brute Force** — plain recursion, exploring both choices (1 step or 2 steps) at every stair, recomputing shared subproblems repeatedly:
```java
public int climbStairsBrute(int n) {
    if (n <= 2) return n;
    return climbStairsBrute(n - 1) + climbStairsBrute(n - 2); // massive overlap in subcalls
}
```
Time: **O(2ⁿ)** — exponential, same issue as naive Fibonacci (Q44), because this problem *is* Fibonacci in disguise.

**Better Solution** — bottom-up DP with two variables (no array needed):
```java
public int climbStairsOptimal(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2; // ways to reach stair 1 and stair 2
    for (int i = 3; i <= n; i++) {
        int current = prev1 + prev2;
        prev2 = prev1;
        prev1 = current;
    }
    return prev1;
}
```
**Explanation:** The number of ways to reach stair `n` is just the sum of ways to reach stair `n-1` and stair `n-2` (you arrive at `n` either from a 1-step or a 2-step). Build this up from the bottom instead of recursing from the top, keeping only the last two values in memory. Time: **O(n)**, Space: **O(1)**.

---

### Q51. Longest Common Subsequence (LCS)
**Problem:** Find the length of the longest subsequence common to two strings (subsequence = characters in order, not necessarily contiguous).
**Example:** `"abcde"`, `"ace"` → `3` (the subsequence `"ace"`).

**Brute Force** — recursively try every possible way of "keeping or skipping" characters from both strings, without caching results:
```java
public int lcsBrute(String s1, String s2, int i, int j) {
    if (i == s1.length() || j == s2.length()) return 0;
    if (s1.charAt(i) == s2.charAt(j)) {
        return 1 + lcsBrute(s1, s2, i + 1, j + 1);
    }
    return Math.max(lcsBrute(s1, s2, i + 1, j), lcsBrute(s1, s2, i, j + 1));
}
```
Time: **O(2^(m+n))** — the same `(i, j)` pair gets recomputed enormous numbers of times.

**Better Solution** — bottom-up DP table:
```java
public int lcsOptimal(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1]; // dp[i][j] = LCS length of s1[0..i) and s2[0..j)
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = 1 + dp[i - 1][j - 1];
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
```
**Explanation:** Build a table where `dp[i][j]` answers "what's the LCS of the first `i` characters of `s1` and the first `j` characters of `s2`?" If the characters at position `i-1` and `j-1` match, extend the LCS found without them by 1. If they don't match, take the best of either ignoring the current character of `s1` or of `s2`. Time: **O(m*n)**, Space: **O(m*n)**.

---

### Q52. 0/1 Knapsack
**Problem:** Given items with weights and values, and a knapsack capacity, maximize total value without exceeding capacity (each item used at most once).
**Example:** weights `[1,3,4,5]`, values `[1,4,5,7]`, capacity `7` → max value `9` (items with weight 3+4=7, value 4+5=9).

**Brute Force** — try every possible subset of items (include/exclude each one), check which fit within capacity:
```java
public int knapsackBrute(int[] weights, int[] values, int capacity, int i) {
    if (i == weights.length || capacity == 0) return 0;
    int excludeItem = knapsackBrute(weights, values, capacity, i + 1);
    int includeItem = 0;
    if (weights[i] <= capacity) {
        includeItem = values[i] + knapsackBrute(weights, values, capacity - weights[i], i + 1);
    }
    return Math.max(includeItem, excludeItem);
}
```
Time: **O(2ⁿ)** — every item independently doubles the number of branches explored.

**Better Solution** — bottom-up DP table:
```java
public int knapsackOptimal(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1]; // dp[i][w] = max value using first i items, capacity w
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i - 1][w]; // option 1: don't take item i-1
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(dp[i][w], values[i - 1] + dp[i - 1][w - weights[i - 1]]); // option 2: take it
            }
        }
    }
    return dp[n][capacity];
}
```
**Explanation:** `dp[i][w]` stores the best value achievable using only the first `i` items with capacity `w`. For each item, you either skip it (value stays the same as without it) or take it (add its value, and use up its weight from the remaining capacity) — take whichever option is better. Time: **O(n * capacity)**, Space: **O(n * capacity)** — "pseudo-polynomial," much better than exponential.

---

### Q53. Coin Change (Minimum Number of Coins)
**Problem:** Given coin denominations and a target amount, find the minimum number of coins needed to make that amount (or -1 if impossible).
**Example:** `coins=[1,2,5]`, `amount=11` → `3` (5+5+1).

**Brute Force** — try every coin at every step recursively:
```java
public int coinChangeBrute(int[] coins, int amount) {
    if (amount == 0) return 0;
    if (amount < 0) return Integer.MAX_VALUE;
    int minCoins = Integer.MAX_VALUE;
    for (int coin : coins) {
        int result = coinChangeBrute(coins, amount - coin);
        if (result != Integer.MAX_VALUE) minCoins = Math.min(minCoins, result + 1);
    }
    return minCoins;
}
```
Time: **O(coins^amount)** — exponential, recomputes the same remaining amounts repeatedly.

**Better Solution** — bottom-up DP array:
```java
public int coinChangeOptimal(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // "infinity" sentinel value
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```
**Explanation:** `dp[i]` = minimum coins needed to make amount `i`. Build it up from `dp[0] = 0` (zero coins for zero amount), and for every amount, try every coin: if using that coin, the remaining amount is `i - coin`, whose answer is already computed — just add 1 coin to it. Time: **O(amount * coins.length)**, Space: **O(amount)**.

---

### Q54. Longest Increasing Subsequence (LIS)
**Problem:** Find the length of the longest strictly increasing subsequence in an array.
**Example:** `[10,9,2,5,3,7,101,18]` → `4` (subsequence `[2,3,7,101]` or `[2,3,7,18]`).

**Brute Force** — try every subsequence recursively (include/exclude each element) and check increasing property:
```java
public int lisBrute(int[] nums, int i, int prevIndex) {
    if (i == nums.length) return 0;
    int exclude = lisBrute(nums, i + 1, prevIndex);
    int include = 0;
    if (prevIndex == -1 || nums[i] > nums[prevIndex]) {
        include = 1 + lisBrute(nums, i + 1, i);
    }
    return Math.max(include, exclude);
}
```
Time: **O(2ⁿ)**.

**Better Solution** — bottom-up DP where `dp[i]` = length of the LIS ending exactly at index `i`:
```java
public int lisOptimal(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1); // every element alone is an LIS of length 1
    int maxLen = 1;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    return maxLen;
}
```
**Explanation:** For each element, look at every earlier element that's smaller than it — if you can extend that earlier element's best increasing subsequence by adding the current element, do so. Track the best `dp[i]` across the whole array. Time: **O(n²)** (there's an O(n log n) binary-search version too, worth mentioning if the interviewer pushes for more optimal), Space: **O(n)**.

---

### Q55. Edit Distance (Levenshtein Distance)
**Problem:** Find the minimum number of operations (insert, delete, replace) to convert one string into another.
**Example:** `"horse"` → `"ros"` → `3` operations.

**Brute Force** — recursively try all three operations at every mismatched position:
```java
public int editDistanceBrute(String s1, String s2, int i, int j) {
    if (i == s1.length()) return s2.length() - j;
    if (j == s2.length()) return s1.length() - i;
    if (s1.charAt(i) == s2.charAt(j)) {
        return editDistanceBrute(s1, s2, i + 1, j + 1);
    }
    int insert = 1 + editDistanceBrute(s1, s2, i, j + 1);
    int delete = 1 + editDistanceBrute(s1, s2, i + 1, j);
    int replace = 1 + editDistanceBrute(s1, s2, i + 1, j + 1);
    return Math.min(insert, Math.min(delete, replace));
}
```
Time: **O(3^(m+n))** — exponential, huge overlap in recomputed `(i,j)` states.

**Better Solution** — bottom-up DP table:
```java
public int editDistanceOptimal(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i; // delete all of s1
    for (int j = 0; j <= n; j++) dp[0][j] = j; // insert all of s2
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1]; // characters match, no operation needed
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],      // replace
                                 Math.min(dp[i - 1][j],         // delete
                                          dp[i][j - 1]));       // insert
            }
        }
    }
    return dp[m][n];
}
```
**Explanation:** `dp[i][j]` = minimum operations to convert the first `i` characters of `s1` into the first `j` characters of `s2`. If the current characters match, no new operation is needed — reuse the answer for one character shorter on both. If they don't match, take the best of the three possible operations (each costs `1` plus whatever the smaller subproblem needed). Time: **O(m*n)**, Space: **O(m*n)**.

---

### Q56. House Robber
**Problem:** Given houses in a row with amounts of money, find the max money you can rob without robbing two adjacent houses.
**Example:** `[2,7,9,3,1]` → `12` (rob houses with 2, 9, 1 = 12).

**Brute Force** — recursively try robbing or skipping each house:
```java
public int robBrute(int[] nums, int i) {
    if (i >= nums.length) return 0;
    int robThis = nums[i] + robBrute(nums, i + 2);   // rob this house, skip next
    int skipThis = robBrute(nums, i + 1);            // skip this house
    return Math.max(robThis, skipThis);
}
```
Time: **O(2ⁿ)** without caching.

**Better Solution** — bottom-up DP with two variables:
```java
public int robOptimal(int[] nums) {
    int prev2 = 0, prev1 = 0; // best up to house i-2, and up to house i-1
    for (int num : nums) {
        int current = Math.max(prev1, prev2 + num);
        prev2 = prev1;
        prev1 = current;
    }
    return prev1;
}
```
**Explanation:** At each house, decide: rob it (its value plus the best result from two houses back, since the adjacent one must be skipped) or skip it (carry forward the best result from just one house back). Only the last two running totals need to be remembered. Time: **O(n)**, Space: **O(1)**.

---

### Q57. Fibonacci-Style Tiling / Word Break (Bonus DP Pattern Recognition)
**Problem:** Given a string and a dictionary of words, determine if the string can be segmented into a space-separated sequence of dictionary words.
**Example:** `s="leetcode"`, `dict=["leet","code"]` → `true`.

**Brute Force** — recursively try every possible "next word" split point:
```java
public boolean wordBreakBrute(String s, Set<String> dict, int start) {
    if (start == s.length()) return true;
    for (int end = start + 1; end <= s.length(); end++) {
        if (dict.contains(s.substring(start, end)) && wordBreakBrute(s, dict, end)) {
            return true;
        }
    }
    return false;
}
```
Time: **O(2ⁿ)** worst case — the same `start` index gets re-explored many times across different call paths.

**Better Solution** — bottom-up DP (boolean array):
```java
public boolean wordBreakOptimal(String s, Set<String> dict) {
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true; // empty string is always "breakable"
    for (int end = 1; end <= n; end++) {
        for (int start = 0; start < end; start++) {
            if (dp[start] && dict.contains(s.substring(start, end))) {
                dp[end] = true;
                break; // found one valid way, no need to check other start points for this "end"
            }
        }
    }
    return dp[n];
}
```
**Explanation:** `dp[i]` means "the substring `s[0..i)` can be fully broken into dictionary words." Build it up left to right: for every ending position, check every possible earlier breakpoint — if that earlier position was breakable AND the piece from there to here is a dictionary word, then this ending position is breakable too. Time: **O(n²)** (or O(n³) if substring extraction cost is counted), Space: **O(n)**.

---
## Category 8: Sorting & Searching

### Q58. Binary Search
**Problem:** Find the index of a target value in a sorted array.
**Example:** `[-1,0,3,5,9,12]`, `target=9` → index `4`.
**SDET relevance:** Binary search logic underlies many test-data-lookup and boundary-value-testing scenarios.

**Brute Force** — linear scan:
```java
public int linearSearchBrute(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] == target) return i;
    }
    return -1;
}
```
Time: **O(n)** — doesn't take advantage of the array being sorted.

**Better Solution** — Binary Search:
```java
public int binarySearchOptimal(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoids overflow vs (left+right)/2
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```
**Explanation:** Since the array is sorted, check the middle element — if it's the target, done; if the target is bigger, the answer must be in the right half (so discard the left half entirely); if smaller, discard the right half. Each step eliminates half of the remaining search space. Time: **O(log n)**, Space: **O(1)**.

---

### Q59. Bubble Sort
**Problem:** Sort an array using the bubble sort algorithm (foundational sorting knowledge often checked in SDET interviews).
**Example:** `[5,2,4,1,3]` → `[1,2,3,4,5]`.

**Brute Force** — (this genuinely IS bubble sort — it's already the "simple but slow" sort, so here brute force = the standard textbook version without the early-exit optimization):
```java
public void bubbleSortBrute(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}
```
Time: **O(n²)** always, even if the array is already sorted — it doesn't know to stop early.

**Better Solution** — Bubble sort with early-exit optimization:
```java
public void bubbleSortOptimal(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        if (!swapped) break; // no swaps this pass means the array is already sorted
    }
}
```
**Explanation:** If an entire pass through the array makes zero swaps, the array is already fully sorted — there's no point continuing. This makes best-case performance (already-sorted input) **O(n)** instead of always O(n²), while worst case remains **O(n²)**. (Note: for real production sorting, Java's built-in `Arrays.sort()` — which uses Dual-Pivot Quicksort or TimSort — should always be preferred; bubble sort is for interview fundamentals only.)

---

### Q60. Merge Sort
**Problem:** Sort an array using the merge sort algorithm.
**Example:** `[38,27,43,3,9,82,10]` → `[3,9,10,27,38,43,82]`.

**Brute Force** — this is naturally where interviewers want the "divide and conquer" solution directly; a genuinely worse alternative is Selection Sort, included here as the beginner-friendly O(n²) comparison point:
```java
public void selectionSortBrute(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) minIdx = j;
        }
        int temp = arr[minIdx]; arr[minIdx] = arr[i]; arr[i] = temp;
    }
}
```
Time: **O(n²)** — finds the minimum remaining element and swaps it into place, one at a time.

**Better Solution** — Merge Sort (divide and conquer):
```java
public void mergeSortOptimal(int[] arr, int left, int right) {
    if (left >= right) return; // base case: 1 element is already "sorted"
    int mid = left + (right - left) / 2;
    mergeSortOptimal(arr, left, mid);
    mergeSortOptimal(arr, mid + 1, right);
    merge(arr, left, mid, right);
}
private void merge(int[] arr, int left, int mid, int right) {
    int[] temp = new int[right - left + 1];
    int i = left, j = mid + 1, k = 0;
    while (i <= mid && j <= right) {
        temp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];
    }
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    System.arraycopy(temp, 0, arr, left, temp.length);
}
```
**Explanation:** Split the array in half recursively until pieces are single elements (trivially sorted), then merge sorted halves back together (same merging logic as Q16, merging two sorted lists). Time: **O(n log n)** — the guaranteed-efficient sort, always, regardless of input order, Space: **O(n)** for the temporary arrays used during merging.

---

### Q61. Quick Sort
**Problem:** Sort an array using the quicksort algorithm.
**Example:** `[10,7,8,9,1,5]` → `[1,5,7,8,9,10]`.

**Brute Force** — Insertion Sort as the beginner-friendly O(n²) comparison point (simple to write, intuitive, but slow):
```java
public void insertionSortBrute(int[] arr) {
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
Time: **O(n²)** worst case — builds up the sorted portion one element at a time, shifting as needed.

**Better Solution** — Quick Sort (partition-based divide and conquer):
```java
public void quickSortOptimal(int[] arr, int low, int high) {
    if (low >= high) return;
    int pivotIndex = partition(arr, low, high);
    quickSortOptimal(arr, low, pivotIndex - 1);
    quickSortOptimal(arr, pivotIndex + 1, high);
}
private int partition(int[] arr, int low, int high) {
    int pivot = arr[high]; // choose last element as pivot
    int i = low - 1; // boundary of elements smaller than pivot
    for (int j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
        }
    }
    int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp; // place pivot in correct spot
    return i + 1;
}
```
**Explanation:** Pick a pivot value; rearrange the array so everything smaller than the pivot is to its left and everything bigger is to its right (this is "partitioning"). Then recursively sort the left and right parts independently — the pivot itself never needs to move again. Time: **O(n log n)** average case (O(n²) worst case with a bad pivot choice, which is why production libraries randomize/median pivot selection), Space: **O(log n)** for the recursion stack.

---

### Q62. Search in a Rotated Sorted Array
**Problem:** Search for a target in an array that was sorted but then rotated at an unknown pivot point.
**Example:** `[4,5,6,7,0,1,2]`, `target=0` → index `4`.

**Brute Force** — plain linear scan, completely ignoring that the array has useful sorted structure:
```java
public int searchBrute(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] == target) return i;
    }
    return -1;
}
```
Time: **O(n)** — correct but throws away the sorted-rotation structure the problem is really testing.

**Better Solution** — Modified Binary Search:
```java
public int searchOptimal(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;
        if (nums[left] <= nums[mid]) { // left half is sorted normally
            if (nums[left] <= target && target < nums[mid]) right = mid - 1;
            else left = mid + 1;
        } else { // right half is sorted normally
            if (nums[mid] < target && target <= nums[right]) left = mid + 1;
            else right = mid - 1;
        }
    }
    return -1;
}
```
**Explanation:** Even after rotation, at least one half of any given window is still perfectly sorted. Figure out which half is sorted by comparing `nums[left]` and `nums[mid]`; then check whether the target could possibly live in that sorted half's range — if so, search there, otherwise search the other half. Time: **O(log n)**, Space: **O(1)**.

---

### Q63. Kth Largest Element in an Array
**Problem:** Find the kth largest element in an unsorted array.
**Example:** `[3,2,1,5,6,4]`, `k=2` → `5` (the 2nd largest).

**Brute Force** — sort the whole array, then index directly:
```java
public int findKthLargestBrute(int[] nums, int k) {
    Arrays.sort(nums);
    return nums[nums.length - k];
}
```
Time: **O(n log n)** — sorting the entire array when you only need one specific element is more work than necessary.

**Better Solution** — Min-Heap of size k:
```java
public int findKthLargestOptimal(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // remove the smallest, keeping only the k largest seen so far
        }
    }
    return minHeap.peek(); // the smallest of the k largest = the kth largest overall
}
```
**Explanation:** Maintain a min-heap that never grows past size `k`. Every time it would exceed `k`, kick out the smallest element in it — since you only ever keep the `k` largest values seen so far, the smallest one in that heap IS the kth largest overall. Time: **O(n log k)** — cheaper than sorting when `k` is much smaller than `n`, Space: **O(k)**.

---

### Q64. Find Peak Element
**Problem:** Find an index where the element is greater than its neighbors (a "peak") in an array.
**Example:** `[1,2,3,1]` → index `2` (value `3`).

**Brute Force** — linear scan comparing each element to both neighbors:
```java
public int findPeakBrute(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        boolean leftOk = (i == 0) || nums[i - 1] < nums[i];
        boolean rightOk = (i == nums.length - 1) || nums[i] > nums[i + 1];
        if (leftOk && rightOk) return i;
    }
    return -1;
}
```
Time: **O(n)** — correct, but doesn't exploit any structure, and a peak is *guaranteed* to exist, which hints a faster approach is possible.

**Better Solution** — Binary Search toward the "uphill" direction:
```java
public int findPeakOptimal(int[] nums) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] > nums[mid + 1]) {
            right = mid; // peak is at mid or to its left (downhill to the right means peak is here or before)
        } else {
            left = mid + 1; // still climbing uphill, peak is to the right
        }
    }
    return left;
}
```
**Explanation:** If the middle element is bigger than its right neighbor, you're on a "downhill" slope — a peak must exist at or before `mid`. If it's smaller, you're still climbing, so the peak must be further right. This lets you discard half the array each step, just like classic binary search. Time: **O(log n)**, Space: **O(1)**.

---

### Q65. Median of Two Sorted Arrays
**Problem:** Given two sorted arrays, find the median of the combined data (a well-known "hard" interview question, but worth knowing conceptually for SDET-adjacent statistics/QA-metrics work).
**Example:** `[1,3]`, `[2]` → median `2.0`.

**Brute Force** — merge both arrays into one sorted array, then find the middle:
```java
public double findMedianBrute(int[] nums1, int[] nums2) {
    int[] merged = new int[nums1.length + nums2.length];
    int i = 0, j = 0, k = 0;
    while (i < nums1.length && j < nums2.length) {
        merged[k++] = (nums1[i] <= nums2[j]) ? nums1[i++] : nums2[j++];
    }
    while (i < nums1.length) merged[k++] = nums1[i++];
    while (j < nums2.length) merged[k++] = nums2[j++];
    int n = merged.length;
    return (n % 2 == 1) ? merged[n / 2] : (merged[n / 2 - 1] + merged[n / 2]) / 2.0;
}
```
Time: **O(m+n)** — correct and reasonably efficient, but doesn't achieve the theoretically optimal bound the problem is famous for.

**Better Solution** — Binary Search on the smaller array to find the correct partition point (this is the well-known O(log(min(m,n))) approach; given its complexity, here is a clearly explained, slightly relaxed version using binary search over positions):
```java
public double findMedianOptimal(int[] nums1, int[] nums2) {
    if (nums1.length > nums2.length) return findMedianOptimal(nums2, nums1); // ensure nums1 is smaller
    int m = nums1.length, n = nums2.length;
    int low = 0, high = m;
    while (low <= high) {
        int cut1 = (low + high) / 2;
        int cut2 = (m + n + 1) / 2 - cut1;
        int left1 = (cut1 == 0) ? Integer.MIN_VALUE : nums1[cut1 - 1];
        int left2 = (cut2 == 0) ? Integer.MIN_VALUE : nums2[cut2 - 1];
        int right1 = (cut1 == m) ? Integer.MAX_VALUE : nums1[cut1];
        int right2 = (cut2 == n) ? Integer.MAX_VALUE : nums2[cut2];
        if (left1 <= right2 && left2 <= right1) { // valid partition found
            if ((m + n) % 2 == 0) {
                return (Math.max(left1, left2) + Math.min(right1, right2)) / 2.0;
            }
            return Math.max(left1, left2);
        } else if (left1 > right2) {
            high = cut1 - 1;
        } else {
            low = cut1 + 1;
        }
    }
    return -1; // input arrays weren't valid sorted arrays
}
```
**Explanation:** Instead of merging, binary search for a "cut point" in the smaller array such that everything to its left (combined with the corresponding cut in the other array) is smaller than everything to the right. Once that partition is valid, the median can be read off directly from the four boundary values. Time: **O(log(min(m, n)))**, Space: **O(1)**. *(For an SDET interview, being able to explain the O(m+n) merge approach clearly is usually sufficient — this optimal version is more commonly asked at SDE-heavy companies.)*

---
## Category 9: Hashing & Miscellaneous (Common SDET Favorites)

### Q66. Count Character Frequency in a String
**Problem:** Count how many times each character appears in a string.
**Example:** `"hello"` → `{h=1, e=1, l=2, o=1}`.

**Brute Force** — for each character, scan the entire string counting matches (recount every character from scratch):
```java
public Map<Character, Integer> charFrequencyBrute(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (freq.containsKey(c)) continue; // already counted this character
        int count = 0;
        for (int j = 0; j < s.length(); j++) {
            if (s.charAt(j) == c) count++;
        }
        freq.put(c, count);
    }
    return freq;
}
```
Time: **O(n²)** — rescans the whole string for every distinct character.

**Better Solution** — single-pass HashMap:
```java
public Map<Character, Integer> charFrequencyOptimal(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    return freq;
}
```
**Explanation:** Walk through the string exactly once, incrementing each character's count in the map as you encounter it. `getOrDefault(c, 0)` handles the "first time seeing this character" case cleanly. Time: **O(n)**, Space: **O(k)** where k = number of distinct characters.

---

### Q67. Find All Pairs With a Given Sum
**Problem:** Given an array and a target sum, find all unique pairs of numbers that add up to the target.
**Example:** `[1,5,7,-1,5]`, `target=6` → pairs `(1,5)` and `(7,-1)`.

**Brute Force** — check every pair with nested loops:
```java
public List<int[]> findPairsBrute(int[] nums, int target) {
    List<int[]> result = new ArrayList<>();
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] + nums[j] == target) {
                result.add(new int[]{nums[i], nums[j]});
            }
        }
    }
    return result;
}
```
Time: **O(n²)**.

**Better Solution** — HashSet (very similar pattern to Q1's Two Sum):
```java
public List<int[]> findPairsOptimal(int[] nums, int target) {
    List<int[]> result = new ArrayList<>();
    Set<Integer> seen = new HashSet<>();
    Set<Integer> usedAsPairFirst = new HashSet<>(); // avoid duplicate pairs
    for (int num : nums) {
        int complement = target - num;
        if (seen.contains(complement) && !usedAsPairFirst.contains(complement)) {
            result.add(new int[]{complement, num});
            usedAsPairFirst.add(num);
        }
        seen.add(num);
    }
    return result;
}
```
**Explanation:** Same idea as Two Sum (Q1): remember every number seen so far, and check if its complement (`target - num`) has already been seen. A second small set prevents the same pair being reported twice when duplicate values exist in the input. Time: **O(n)**, Space: **O(n)**.

---

### Q68. Check if Two Strings are Rotations of Each Other
**Problem:** Determine if one string is a rotation of another.
**Example:** `s1="waterbottle"`, `s2="erbottlewat"` → `true`.

**Brute Force** — generate every possible rotation of `s1` and compare each to `s2`:
```java
public boolean areRotationsBrute(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    for (int i = 0; i < s1.length(); i++) {
        String rotated = s1.substring(i) + s1.substring(0, i);
        if (rotated.equals(s2)) return true;
    }
    return false;
}
```
Time: **O(n²)** — builds up to `n` new strings, each comparison itself taking O(n).

**Better Solution** — clever one-liner using substring search:
```java
public boolean areRotationsOptimal(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    String doubled = s1 + s1; // concatenating s1 with itself contains every rotation of s1
    return doubled.contains(s2);
}
```
**Explanation:** If you concatenate a string with itself (`s1 + s1`), every possible rotation of `s1` appears somewhere inside that doubled string as a contiguous substring. So checking if `s2` is a rotation of `s1` reduces to a simple substring search. Time: **O(n)** on average (Java's `contains()` uses an efficient substring search), Space: **O(n)** for the doubled string.

---

### Q69. FizzBuzz
**Problem:** Print numbers 1 to n; for multiples of 3 print "Fizz", multiples of 5 print "Buzz", multiples of both print "FizzBuzz".
**Example:** `n=15` → `1,2,Fizz,4,Buzz,Fizz,7,8,Fizz,Buzz,11,Fizz,13,14,FizzBuzz`.
**SDET relevance:** A classic quick warm-up screening question — simple, but interviewers watch for clean conditional ordering.

**Brute Force** — checking divisibility by 3 and 5 separately, potentially printing twice by mistake (a very common beginner bug shown here so you can recognize and avoid it):
```java
public void fizzBuzzBuggy(int n) {
    for (int i = 1; i <= n; i++) {
        if (i % 3 == 0) System.out.println("Fizz");
        if (i % 5 == 0) System.out.println("Buzz"); // BUG: multiples of 15 print twice, not "FizzBuzz"
        if (i % 3 != 0 && i % 5 != 0) System.out.println(i);
    }
}
```
This has a real logic bug: multiples of 15 print "Fizz" then "Buzz" on two separate lines instead of a single "FizzBuzz" — a good reminder to always check the *most specific* condition first.

**Better Solution** — check the combined condition first:
```java
public void fizzBuzzOptimal(int n) {
    for (int i = 1; i <= n; i++) {
        if (i % 15 == 0) System.out.println("FizzBuzz"); // check most specific condition FIRST
        else if (i % 3 == 0) System.out.println("Fizz");
        else if (i % 5 == 0) System.out.println("Buzz");
        else System.out.println(i);
    }
}
```
**Explanation:** Since 15 is divisible by both 3 and 5, checking `i % 15 == 0` first (before the individual checks) correctly catches the "both" case before it's incorrectly handled by only one of the narrower checks. This "most specific condition first" pattern applies broadly whenever you have overlapping conditions. Time: **O(n)**, Space: **O(1)**.

---

### Q70. First Non-Repeating Character in a String
**Problem:** Find the first character in a string that doesn't repeat.
**Example:** `"swiss"` → `'w'` (s repeats, w doesn't, but w is the first one that never repeats).

**Brute Force** — for each character, scan the whole string to count its occurrences:
```java
public char firstNonRepeatingBrute(String s) {
    for (int i = 0; i < s.length(); i++) {
        boolean repeats = false;
        for (int j = 0; j < s.length(); j++) {
            if (i != j && s.charAt(i) == s.charAt(j)) { repeats = true; break; }
        }
        if (!repeats) return s.charAt(i);
    }
    return '\0'; // no non-repeating character found
}
```
Time: **O(n²)**.

**Better Solution** — two passes with a frequency HashMap:
```java
public char firstNonRepeatingOptimal(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    for (char c : s.toCharArray()) {
        if (freq.get(c) == 1) return c;
    }
    return '\0';
}
```
**Explanation:** First pass builds a complete frequency count of every character (same pattern as Q66). Second pass walks the string again in original order and returns the first character whose count is exactly `1`. Two linear passes are still much faster than one quadratic pass. Time: **O(n)**, Space: **O(k)**.

---

### Q71. Longest Common Prefix Among an Array of Strings
**Problem:** Find the longest string prefix common to all strings in an array.
**Example:** `["flower","flow","flight"]` → `"fl"`.

**Brute Force** — compare the first string against every other string character by character, restarting comparisons for every pair:
```java
public String longestCommonPrefixBrute(String[] strs) {
    if (strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        StringBuilder common = new StringBuilder();
        int minLen = Math.min(prefix.length(), strs[i].length());
        for (int j = 0; j < minLen; j++) {
            if (prefix.charAt(j) == strs[i].charAt(j)) common.append(prefix.charAt(j));
            else break;
        }
        prefix = common.toString();
    }
    return prefix;
}
```
This actually works reasonably well (Time: **O(n*m)** where m = shortest string length) — it's essentially the standard approach already, just written slightly more verbosely than necessary.

**Better Solution** — cleaner version using `String.startsWith()`, shrinking the candidate prefix directly:
```java
public String longestCommonPrefixOptimal(String[] strs) {
    if (strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (!strs[i].startsWith(prefix)) {
            prefix = prefix.substring(0, prefix.length() - 1); // shrink prefix by one character
        }
        if (prefix.isEmpty()) return "";
    }
    return prefix;
}
```
**Explanation:** Start assuming the whole first string is the common prefix, then for each subsequent string, keep chopping one character off the end of your candidate prefix until it actually IS a prefix of that string. By the end, whatever remains is guaranteed common to all strings checked so far. Time: **O(n*m)** worst case, but often faster in practice, Space: **O(1)** extra (ignoring the string itself).

---

### Q72. Find the Missing Number in an Array (1 to N)
**Problem:** Given an array containing `n` distinct numbers from `0` to `n` with exactly one missing, find the missing number.
**Example:** `[3,0,1]` → `2` (n=3, numbers 0-3, missing 2).

**Brute Force** — for every number from 0 to n, check if it exists in the array:
```java
public int missingNumberBrute(int[] nums) {
    int n = nums.length;
    for (int i = 0; i <= n; i++) {
        boolean found = false;
        for (int num : nums) {
            if (num == i) { found = true; break; }
        }
        if (!found) return i;
    }
    return -1;
}
```
Time: **O(n²)**.

**Better Solution** — Gauss's sum formula:
```java
public int missingNumberOptimal(int[] nums) {
    int n = nums.length;
    int expectedSum = n * (n + 1) / 2; // sum of 0..n
    int actualSum = 0;
    for (int num : nums) actualSum += num;
    return expectedSum - actualSum;
}
```
**Explanation:** The sum of all numbers from `0` to `n` has a known formula (`n*(n+1)/2`). Compute what the sum *should* be, subtract what the array's numbers *actually* sum to — the difference is exactly the missing number. Time: **O(n)**, Space: **O(1)**. (A HashSet approach also works in O(n) time/space if asked for an alternative, but the math trick uses no extra memory at all.)

---

### Q73. Check for Balanced Tags/Brackets in a Nested Structure (XML/JSON-style)
**Problem:** Given a string representing nested tags (similar to XML), verify that every opening tag has a matching, correctly nested closing tag. This is a direct real-world extension of Q21, highly relevant for SDETs validating API responses or config files.
**Example:** `"<a><b></b></a>"` → `true`; `"<a><b></a></b>"` → `false` (incorrectly nested).

**Brute Force** — repeatedly find and remove the first innermost matching tag pair (`<x></x>` with no other tags between them) until no more can be removed, similar to the Q21 brute-force approach:
```java
public boolean isBalancedTagsBrute(String s) {
    boolean changed = true;
    while (changed) {
        changed = false;
        int len = s.length();
        s = s.replaceAll("<(\\w+)></\\1>", ""); // remove innermost matched pairs
        if (s.length() != len) changed = true;
    }
    return s.isEmpty();
}
```
Time: **O(n²)** or worse — repeated regex scanning and string rebuilding on every iteration is expensive.

**Better Solution** — Stack of tag names (directly extending the Q21 pattern to named tags):
```java
public boolean isBalancedTagsOptimal(List<String> tokens) {
    // tokens is a pre-parsed list like ["<a>", "<b>", "</b>", "</a>"]
    Deque<String> stack = new ArrayDeque<>();
    for (String token : tokens) {
        if (!token.startsWith("</")) {
            String tagName = token.substring(1, token.length() - 1); // "<a>" -> "a"
            stack.push(tagName);
        } else {
            String tagName = token.substring(2, token.length() - 1); // "</a>" -> "a"
            if (stack.isEmpty() || !stack.pop().equals(tagName)) return false;
        }
    }
    return stack.isEmpty();
}
```
**Explanation:** Exactly the same idea as validating parentheses (Q21), just with named tags instead of bracket symbols: push opening tag names, and when a closing tag appears, it must match whatever is currently on top of the stack. Time: **O(n)**, Space: **O(n)**.

---

### Q74. Implement an LRU (Least Recently Used) Cache
**Problem:** Design a cache with `get(key)` and `put(key, value)` operations that evicts the least recently used item when it exceeds a fixed capacity — both operations must run in O(1).
**Example:** capacity `2`; put(1,1), put(2,2), get(1)→1, put(3,3) evicts key 2 (least recently used), get(2)→-1 (not found).
**Frequency:** Very frequently asked, especially at Microsoft/Amazon per multiple sources — also directly relevant to SDETs who build test caching/session layers.

**Brute Force** — a plain ArrayList/LinkedHashMap-free approach, storing entries in a list and linearly scanning to find/update/evict:
```java
class LRUCacheBrute {
    private final int capacity;
    private final List<int[]> entries = new ArrayList<>(); // each entry: [key, value]

    LRUCacheBrute(int capacity) { this.capacity = capacity; }

    int get(int key) {
        for (int i = 0; i < entries.size(); i++) {
            if (entries.get(i)[0] == key) {
                int[] entry = entries.remove(i);
                entries.add(entry); // move to "most recently used" end
                return entry[1];
            }
        }
        return -1;
    }
    void put(int key, int value) {
        for (int i = 0; i < entries.size(); i++) {
            if (entries.get(i)[0] == key) { entries.remove(i); break; }
        }
        if (entries.size() == capacity) entries.remove(0); // evict least recently used
        entries.add(new int[]{key, value});
    }
}
```
Every `get`/`put` requires a linear scan — Time: **O(n)** per operation, which fails the O(1) requirement.

**Better Solution** — `LinkedHashMap` with access-order mode (Java has this built in specifically for LRU-style caches):
```java
class LRUCacheOptimal extends LinkedHashMap<Integer, Integer> {
    private final int capacity;

    LRUCacheOptimal(int capacity) {
        super(capacity, 0.75f, true); // true = access-order (most recently accessed moves to the end)
        this.capacity = capacity;
    }

    int get(int key) {
        return super.getOrDefault(key, -1);
    }
    void put(int key, int value) {
        super.put(key, value);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity; // automatically evict the least recently used entry
    }
}
```
**Explanation:** Java's `LinkedHashMap` can be configured with `accessOrder=true`, which automatically reorders entries so the most recently accessed one moves to the "end" and the least recently used stays at the "front." Overriding `removeEldestEntry()` tells it to auto-evict once capacity is exceeded — all of this happens in O(1) internally via a HashMap plus doubly-linked list. Time: **O(1)** for both `get` and `put`, Space: **O(capacity)**. *(For a from-scratch interview answer without relying on `LinkedHashMap`, the standard approach is a HashMap combined with a manually implemented doubly-linked list — worth mentioning if the interviewer asks you to build it without the built-in class.)*

---

### Q75. Detect Duplicates in an Array Using a HashSet
**Problem:** Determine whether an array contains any duplicate values.
**Example:** `[1,2,3,1]` → `true`; `[1,2,3,4]` → `false`.
**SDET relevance:** This exact check is extremely common when validating test data sets, ensuring unique IDs, or checking for accidental duplicate test cases.

**Brute Force** — nested loop comparing every pair:
```java
public boolean containsDuplicateBrute(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] == nums[j]) return true;
        }
    }
    return false;
}
```
Time: **O(n²)**.

**Better Solution** — HashSet:
```java
public boolean containsDuplicateOptimal(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int num : nums) {
        if (!seen.add(num)) return true; // add() returns false if already present
    }
    return false;
}
```
**Explanation:** As in Q3, `Set.add()` conveniently returns `false` when the element is already in the set — the moment that happens, you've found a duplicate and can exit immediately. Time: **O(n)**, Space: **O(n)**. *(Alternative one-liner for a quick sanity check: `new HashSet<>(Arrays.asList(nums)).size() != nums.length` — but the loop version above is preferred in an interview since it early-exits and shows clearer reasoning.)*

---

## Quick-Reference Summary Table

| # | Problem | Category | Brute Force | Optimal | Key Pattern |
|---|---------|----------|:---:|:---:|---|
| 1 | Two Sum | Array | O(n²) | O(n) | HashMap |
| 2 | Reverse Array | Array | O(n) extra space | O(1) space | Two Pointer |
| 3 | Find Duplicate | Array | O(n²) | O(n) | HashSet |
| 4 | Max Subarray | Array | O(n²) | O(n) | Kadane's |
| 5 | Merge Intervals | Array | O(n³) | O(n log n) | Sort + Merge |
| 6 | Move Zeroes | Array | O(n) extra space | O(1) space | Two Pointer |
| 7 | Longest Substr w/o Repeat | String | O(n²) | O(n) | Sliding Window |
| 8 | Valid Anagram | String | O(n log n) | O(n) | Frequency Array |
| 9 | Group Anagrams | String | O(n²k) | O(nk log k) | HashMap + Sorted Key |
| 10 | Palindrome String | String | O(n) extra space | O(1) space | Two Pointer |
| 11 | Rotate Array | Array | O(n*k) | O(n) | Reverse Trick |
| 12 | Product Except Self | Array | O(n²) | O(n) | Prefix/Suffix |
| 13 | Reverse Linked List | Linked List | O(n) extra space | O(1) space | Pointer Reversal |
| 14 | Detect Cycle | Linked List | O(n) space | O(1) space | Floyd's Algorithm |
| 15 | Find Middle | Linked List | 2 passes | 1 pass | Slow/Fast Pointer |
| 16 | Merge Sorted Lists | Linked List | O(n log n) | O(n+m) | Two Pointer Merge |
| 17 | Remove Nth From End | Linked List | 2 passes | 1 pass | Gap Pointer |
| 18 | Palindrome Linked List | Linked List | O(n) space | O(1) space | Reverse Half |
| 19 | Intersection of Lists | Linked List | O(n*m) | O(n+m) | Pointer Swap |
| 20 | Remove Duplicates (sorted) | Linked List | O(n²) | O(n) | Adjacent Check |
| 21 | Valid Parentheses | Stack | O(n²) | O(n) | Stack |
| 22 | Queue Using Stacks | Stack | O(n) per op | Amortized O(1) | Two Stacks |
| 23 | Min Stack | Stack | O(n) getMin | O(1) getMin | Auxiliary Stack |
| 24 | Next Greater Element | Stack | O(n²) | O(n) | Monotonic Stack |
| 25 | Evaluate RPN | Stack | O(n²) | O(n) | Stack |
| 26 | Sliding Window Max | Stack/Queue | O(n*k) | O(n) | Monotonic Deque |
| 27 | Tree Traversals | Tree | Iterative w/ manual stack | Recursion | DFS |
| 28 | Level Order Traversal | Tree | O(n*h) | O(n) | BFS + Queue |
| 29 | Max Depth | Tree | O(n) w/ BFS | O(n) simpler | Recursion |
| 30 | Validate BST | Tree | O(n²) | O(n) | Min/Max Range |
| 31 | LCA (BST) | Tree | O(h) w/ paths | O(h) direct | BST Property |
| 32 | Diameter of Tree | Tree | O(n²) | O(n) | Height + Track Max |
| 33 | Symmetric Tree | Tree | O(n) extra space | O(h) space | Mirror Recursion |
| 34 | Sorted Array to BST | Tree | O(n²) unbalanced | O(n) balanced | Middle Element |
| 35 | Path Sum | Tree | O(n) all paths | O(n) early exit | Recursion |
| 36 | Serialize/Deserialize Tree | Tree | Level-order (fiddly) | Preorder + markers | Recursion + Queue |
| 37 | BFS Traversal | Graph | Manual frontier list | O(V+E) | Queue |
| 38 | DFS Traversal | Graph | Manual stack (messy) | O(V+E) | Recursion |
| 39 | Number of Islands | Graph | N/A (flood fill is standard) | O(rows*cols) | Flood Fill (DFS) |
| 40 | Detect Cycle (Directed) | Graph | O(V*(V+E)) | O(V+E) | 3-State DFS |
| 41 | Clone Graph | Graph | O(V²) | O(V+E) | HashMap + DFS |
| 42 | Topological Sort | Graph | O(V²*E) | O(V+E) | Kahn's Algorithm |
| 43 | Course Schedule | Graph | Exponential w/o cache | O(V+E) | Cycle Detection |
| 44 | Fibonacci | Recursion | O(2ⁿ) | O(n) | Memoization |
| 45 | Permutations | Backtracking | N/A (backtracking standard) | O(n!*n) | Backtracking |
| 46 | Subsets | Backtracking | O(n*2ⁿ) bitmask | O(n*2ⁿ) clearer | Include/Exclude |
| 47 | Generate Parentheses | Backtracking | O(4ⁿ) generate+filter | O(4ⁿ/√n) | Constrained Backtracking |
| 48 | N-Queens | Backtracking | Not runnable | O(n!) pruned | Backtracking |
| 49 | Combination Sum | Backtracking | Exponential unpruned | Faster w/ pruning | Sort + Prune |
| 50 | Climbing Stairs | DP | O(2ⁿ) | O(n) | Bottom-Up DP |
| 51 | Longest Common Subsequence | DP | O(2^(m+n)) | O(m*n) | DP Table |
| 52 | 0/1 Knapsack | DP | O(2ⁿ) | O(n*capacity) | DP Table |
| 53 | Coin Change | DP | O(coins^amount) | O(amount*coins) | DP Array |
| 54 | Longest Increasing Subseq | DP | O(2ⁿ) | O(n²) | DP Array |
| 55 | Edit Distance | DP | O(3^(m+n)) | O(m*n) | DP Table |
| 56 | House Robber | DP | O(2ⁿ) | O(n) | Bottom-Up DP |
| 57 | Word Break | DP | O(2ⁿ) | O(n²) | DP Array |
| 58 | Binary Search | Search | O(n) | O(log n) | Binary Search |
| 59 | Bubble Sort | Sort | O(n²) always | O(n²) worst, O(n) best | Early Exit |
| 60 | Merge Sort | Sort | O(n²) (Selection Sort) | O(n log n) | Divide & Conquer |
| 61 | Quick Sort | Sort | O(n²) (Insertion Sort) | O(n log n) avg | Partitioning |
| 62 | Search Rotated Array | Search | O(n) | O(log n) | Modified Binary Search |
| 63 | Kth Largest Element | Search | O(n log n) | O(n log k) | Min-Heap |
| 64 | Find Peak Element | Search | O(n) | O(log n) | Binary Search |
| 65 | Median of Two Sorted Arrays | Search | O(m+n) | O(log(min(m,n))) | Binary Search Partition |
| 66 | Char Frequency Count | Hashing | O(n²) | O(n) | HashMap |
| 67 | Pairs With Given Sum | Hashing | O(n²) | O(n) | HashSet |
| 68 | String Rotation Check | Hashing | O(n²) | O(n) | Concatenation Trick |
| 69 | FizzBuzz | Misc | Buggy version | O(n) | Condition Ordering |
| 70 | First Non-Repeating Char | Hashing | O(n²) | O(n) | HashMap, 2 passes |
| 71 | Longest Common Prefix | String | O(n*m) verbose | O(n*m) clean | Shrinking Prefix |
| 72 | Missing Number | Hashing | O(n²) | O(n) | Gauss's Formula |
| 73 | Balanced Tags (XML/JSON) | Hashing/Stack | O(n²) regex | O(n) | Stack |
| 74 | LRU Cache | Hashing/Design | O(n) per op | O(1) per op | LinkedHashMap |
| 75 | Contains Duplicate | Hashing | O(n²) | O(n) | HashSet |

---

## How to Use This List

1. **Week 1-2:** Focus on Categories 1-3 (Arrays/Strings, Linked List, Stack/Queue) — these come up in almost every SDET screen.
2. **Week 3:** Categories 4-5 (Trees, Graphs) — common at the onsite/loop stage.
3. **Week 4:** Categories 6-7 (Recursion/Backtracking, DP) — usually the hardest for beginners; don't skip the "brute force first" step, it builds the intuition DP relies on.
4. **Ongoing:** Categories 8-9 (Sorting/Searching, Hashing/Misc) — many of these (FizzBuzz, Balanced Tags, LRU Cache, Duplicate Detection) are especially SDET-flavored and worth prioritizing if your interview loop leans toward test-automation-adjacent coding rather than pure algorithms.

**A note on validation:** every solution above uses standard, well-established algorithms (two-pointer, sliding window, Floyd's cycle detection, Kadane's algorithm, backtracking, bottom-up DP, Kahn's algorithm, etc.) that are consistent with how they're taught across GeeksforGeeks, InterviewBit, and standard DSA textbooks. If you paste any snippet into an IDE and find an edge case that breaks it (e.g., empty array, single element, null input), that's a great next step — production-grade interview answers should always mention how they'd handle those edge cases too.
