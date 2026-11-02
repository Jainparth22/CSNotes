# Placement Study Notes — Part 1: DSA

*Built from your Master Placement & Interview Checklist. Code is in Python 3 (say the word and I will convert to C++ or Java).*

## 0. How to use these notes

Every topic follows the same card so that revision is fast:

- **Recognize**: the words in a problem that should make this pattern pop into your head.
- **Idea**: the core intuition, with a diagram where it helps.
- **Template**: code you can rebuild from memory.
- **Complexity**: time and space.
- **Loopholes**: the bugs, edge cases and traps interviewers (and online assessments) love.
- **Practice**: classic problems to cement it.

**Revision mode before an interview or OA:** read only *Recognize*, *Loopholes* and the template for the topic.

**Series plan:** Part 1 DSA (this doc) · Part 2 OS, DBMS, Networks · Part 3 SQL and Git · Part 4 OOP, Design Patterns, LLD · Part 5 System Design · Part 6 Backend, AI/ML, Projects, HR.

**Contents of Part 1:** 1 Complexity · 2 Arrays/Strings · 3 Hashing · 4 Two Pointers · 5 Sliding Window · 6 Prefix Sums and Difference Arrays · 7 Binary Search (and on answer) · 8 Fast/Slow Pointers · 9 Linked Lists · 10 Stack and Monotonic Stack · 11 Queue and Deque · 12 Heap/Top-K · 13 Intervals · 14 Sorting · 15 Matrix · 16 Trees/BST · 17 Graphs (BFS, DFS, Topo, DSU, Dijkstra, MST, Bellman-Ford, Floyd-Warshall, SCC) · 18 Recursion, Backtracking, Divide and Conquer · 19 Greedy · 20 Dynamic Programming · 21 Trie · 22 Bit Manipulation · 23 KMP and Z · 24 Kadane · 25 Pattern-recognition cheat sheet.

---

## 1. Time and space complexity analysis

**Recognize the target complexity from constraints** (this is the single most useful OA skill):

| Constraint on n | Acceptable complexity |
| --- | --- |
| n ≤ 10 | O(n!) permutations |
| n ≤ 20 to 25 | O(2^n) subsets, bitmask |
| n ≤ 500 | O(n^3) (Floyd-Warshall, interval DP) |
| n ≤ 5,000 | O(n^2) (2D DP) |
| n ≤ 10^5 to 10^6 | O(n log n) or O(n) |
| n ≤ 10^8 | O(n), tiny constant |
| n up to 10^18 | O(log n) or O(1) (math, binary search) |

Rule of thumb: about 10^8 simple operations per second in C++/Java, about 10^7 in Python.

**Rules**

1. Drop constants and lower-order terms: 3n^2 + 5n → O(n^2).
2. Sequential blocks add; nested loops multiply.
3. Recursion: write the recurrence, then solve it.
4. Space = extra memory **including the recursion stack** (DFS on a skewed tree is O(n) space).
5. **Amortized** cost: average over a sequence. Dynamic-array append is O(1) amortized (resize doubles capacity); monotonic stack is O(n) total even though it has a while-loop inside a for-loop; DSU with path compression is near O(1) amortized.

**Common recurrences**

| Recurrence | Result | Example |
| --- | --- | --- |
| T(n) = T(n/2) + O(1) | O(log n) | binary search |
| T(n) = T(n-1) + O(1) | O(n) | linear recursion |
| T(n) = 2T(n/2) + O(n) | O(n log n) | merge sort |
| T(n) = 2T(n/2) + O(1) | O(n) | tree traversal |
| T(n) = T(n-1) + T(n-2) | O(2^n) (about 1.618^n) | naive Fibonacci |
| T(n) = 2T(n-1) | O(2^n) | subsets |

**Master theorem:** T(n) = aT(n/b) + O(n^d). Compare d with log\_b(a): if d < log\_b a then O(n^(log\_b a)); if d = log\_b a then O(n^d log n); if d > log\_b a then O(n^d).

**Loopholes**

- `s += ch` in a loop is O(n^2) in many languages (strings immutable). Use a list and `''.join`.
- Slicing `a[i:j]` copies, costing O(j-i). Slicing inside a loop silently makes O(n^2).
- `x in list` is O(n); `x in set` is O(1) average, O(n) worst case.
- `list.pop(0)` and `list.insert(0, x)` are O(n). Use `collections.deque`.
- Sorting inside a loop multiplies the cost by n log n.
- Python's default recursion limit is 1000. For deep recursion use `sys.setrecursionlimit` or convert to iteration.
- In C++/Java, `mid = (lo + hi) / 2` can overflow; use `lo + (hi - lo) / 2`. Sums of up to 10^5 values of 10^9 need `long long`/`long`.
- State both time **and** space. Say what n is when there are multiple inputs (O(V + E), O(m \* n)).

---

## 2. Arrays and strings

**Idea:** contiguous memory gives O(1) index access, O(n) insert/delete in the middle. Most problems are solved by (a) one clever pass, (b) in-place swaps, or (c) a helper structure (map, prefix sums).

**Recognize:** "in place", "O(1) extra space", "rotate", "reverse", "partition", "anagram", "palindrome", "subarray/substring".

**Template 1: rotate array right by k using three reversals**

```python
def rotate(a, k):
    n = len(a)
    k %= n                      # loophole: k can exceed n
    def rev(l, r):
        while l < r:
            a[l], a[r] = a[r], a[l]
            l += 1; r -= 1
    rev(0, n - 1); rev(0, k - 1); rev(k, n - 1)
```

Why it works: reversing the whole array moves the last k items to the front in reverse order; reversing each part restores their order.

**Template 2: Dutch National Flag (sort 0/1/2 in one pass)**

```python
def sort_colors(a):
    lo, mid, hi = 0, 0, len(a) - 1
    while mid <= hi:
        if a[mid] == 0:
            a[lo], a[mid] = a[mid], a[lo]; lo += 1; mid += 1
        elif a[mid] == 1:
            mid += 1
        else:
            a[mid], a[hi] = a[hi], a[mid]; hi -= 1   # do NOT advance mid here
```

Invariant: `[0, lo)` are 0s, `[lo, mid)` are 1s, `(hi, end]` are 2s, `[mid, hi]` unknown.

**Template 3: valid anagram / frequency compare**

```python
from collections import Counter
def is_anagram(s, t):
    return Counter(s) == Counter(t)          # O(n) time, O(alphabet) space
```

**Template 4: palindrome check ignoring non-alphanumerics (two pointers)**

```python
def is_pal(s):
    l, r = 0, len(s) - 1
    while l < r:
        while l < r and not s[l].isalnum(): l += 1
        while l < r and not s[r].isalnum(): r -= 1
        if s[l].lower() != s[r].lower(): return False
        l += 1; r -= 1
    return True
```

**Complexity:** most are O(n) time, O(1) space.

**Loopholes**

- Off-by-one on loop bounds; always test n = 0, 1, 2.
- Modifying a list while iterating over it; iterate a copy or by index.
- `k % n` when n = 0 raises a division error; guard empty input.
- Case sensitivity, spaces, Unicode, negative numbers, duplicates.
- Asked for O(1) space but you built a copy: re-read the requirement.

**Practice:** Rotate Array, Sort Colors, Move Zeroes, Valid Palindrome, Reverse Words in a String, Product of Array Except Self, Next Permutation, Spiral Matrix.

---

## 3. Hashing: HashMap and HashSet

**Idea:** a hash function maps a key to a bucket index. Lookup, insert, delete are O(1) **average**.

```
key -> hash(key) % capacity -> bucket
[0] -> (k1,v1) -> (k5,v5)    <- collision handled by chaining
[1] -> empty
[2] -> (k2,v2)
```

- **Collision handling:** chaining (linked list or tree per bucket) or open addressing (linear probing, quadratic probing, double hashing).
- **Load factor** = entries / buckets. When it crosses a threshold (Java 0.75) the table doubles and everything is rehashed (O(n) occasionally, O(1) amortized).
- **Java HashMap:** buckets with 8 or more entries become red-black trees (worst case O(log n)).
- **HashSet** is a HashMap with dummy values.

**Recognize:** "find a pair", "count frequency", "first non-repeating", "duplicate", "group by", "seen before", "subarray sum equals".

**Template 1: Two Sum**

```python
def two_sum(a, target):
    seen = {}
    for i, x in enumerate(a):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i                  # insert AFTER checking so an element does not pair with itself
```

**Template 2: Group Anagrams**

```python
from collections import defaultdict
def group_anagrams(words):
    g = defaultdict(list)
    for w in words:
        g[tuple(sorted(w))].append(w)    # or a 26-length count tuple for O(L) per word
    return list(g.values())
```

**Template 3: Longest Consecutive Sequence in O(n)**

```python
def longest_consecutive(a):
    s, best = set(a), 0
    for x in s:
        if x - 1 not in s:               # only start counting at the beginning of a run
            y = x
            while y + 1 in s: y += 1
            best = max(best, y - x + 1)
    return best
```

**Template 4: Subarray Sum Equals K (prefix sum + map)**

```python
def subarray_sum(a, k):
    cnt, pre, ans = {0: 1}, 0, 0         # {0:1} handles subarrays starting at index 0
    for x in a:
        pre += x
        ans += cnt.get(pre - k, 0)
        cnt[pre] = cnt.get(pre, 0) + 1
    return ans
```

**Loopholes**

- Keys must be hashable and immutable: a Python list cannot be a key (use a tuple); in Java, mutating an object after using it as a key loses it.
- Worst case is O(n) with adversarial collisions (relevant for C++ `unordered_map` on Codeforces-style tests; use a custom hash or `map`).
- Iteration order is not guaranteed for HashSet/unordered\_map; Python 3.7+ dicts keep insertion order but do not rely on it for sets.
- Floating-point keys are unreliable.
- Insert-then-check bugs: Two Sum with duplicates, pairing an element with itself.
- When memory is tight and keys are small integers or lowercase letters, use an array of size 26/128 instead.

**Practice:** Two Sum, Group Anagrams, Longest Consecutive Sequence, Subarray Sum Equals K, Top K Frequent Elements, Contains Duplicate II, Isomorphic Strings, LRU Cache (hash map + doubly linked list, see LLD).

---

## 4. Two pointers

**Recognize:** sorted array or string, "pair/triplet with sum", "in-place", "remove duplicates", "palindrome", "merge two sorted", "container/area".

**Three flavours**

```
1) Opposite ends:      [ l -> ........ <- r ]        pairs, palindromes, container
2) Same direction:     [ writer ... reader -> ]      remove duplicates, move zeroes, partition
3) Two arrays:         a: i ->     b: j ->           merge, intersection
```

**Template 1: Two Sum on a sorted array**

```python
def two_sum_sorted(a, t):
    l, r = 0, len(a) - 1
    while l < r:
        s = a[l] + a[r]
        if s == t: return [l, r]
        if s < t: l += 1      # need a bigger sum, only moving l can help
        else: r -= 1
```

**Template 2: 3Sum with duplicate skipping**

```python
def three_sum(a):
    a.sort(); res = []
    for i in range(len(a) - 2):
        if i and a[i] == a[i - 1]: continue            # skip duplicate anchors
        l, r = i + 1, len(a) - 1
        while l < r:
            s = a[i] + a[l] + a[r]
            if s < 0: l += 1
            elif s > 0: r -= 1
            else:
                res.append([a[i], a[l], a[r]])
                l += 1; r -= 1
                while l < r and a[l] == a[l - 1]: l += 1   # skip duplicate left values
    return res
```

**Template 3: Container With Most Water**

```python
def max_area(h):
    l, r, best = 0, len(h) - 1, 0
    while l < r:
        best = max(best, min(h[l], h[r]) * (r - l))
        if h[l] < h[r]: l += 1       # the shorter wall limits area; moving the taller one can only shrink width
        else: r -= 1
    return best
```

**Template 4: Remove duplicates from sorted array in place (reader/writer)**

```python
def remove_dup(a):
    w = 0
    for r in range(len(a)):
        if w == 0 or a[r] != a[w - 1]:
            a[w] = a[r]; w += 1
    return w
```

**Complexity:** O(n) after an optional O(n log n) sort; O(1) space.

**Loopholes**

- Sorting destroys original indices. If the problem asks for indices, use a hash map instead.
- Forgetting to skip duplicates produces repeated triplets; skipping wrongly can drop valid answers (skip only *after* recording one).
- `l < r` versus `l <= r` matters: pairs need distinct elements.
- The correctness argument is the interview answer: say why discarding one pointer can never lose the optimum.

**Practice:** Two Sum II, 3Sum, 3Sum Closest, Container With Most Water, Trapping Rain Water, Remove Duplicates, Move Zeroes, Merge Sorted Array.

---

## 5. Sliding window

**Recognize:** "longest/shortest substring or subarray satisfying a condition", "contiguous", "window of size k", "at most k distinct".

```
array:  [ 2  1  5  1  3  2 ]      window [l .. r]
         l        r  -> expand r each step; shrink l while the window is invalid
```

**Fixed window (size k)**

```python
def max_sum_k(a, k):
    cur = sum(a[:k]); best = cur
    for i in range(k, len(a)):
        cur += a[i] - a[i - k]        # add new, drop old
        best = max(best, cur)
    return best
```

**Variable window template**

```python
def longest_unique(s):
    last, l, best = {}, 0, 0
    for r, ch in enumerate(s):
        if ch in last and last[ch] >= l:     # duplicate inside the current window
            l = last[ch] + 1
        last[ch] = r
        best = max(best, r - l + 1)
    return best
```

**Minimum Window Substring (need/have counters)**

```python
from collections import Counter
def min_window(s, t):
    need, missing = Counter(t), len(t)
    l, best = 0, (float('inf'), 0, 0)
    for r, ch in enumerate(s):
        if need[ch] > 0: missing -= 1
        need[ch] -= 1
        while missing == 0:                       # window valid: try to shrink
            if r - l + 1 < best[0]: best = (r - l + 1, l, r)
            need[s[l]] += 1
            if need[s[l]] > 0: missing += 1
            l += 1
    return '' if best[0] == float('inf') else s[best[1]:best[2] + 1]
```

**Trick: "exactly K" = "at most K" minus "at most K-1"** (e.g. Subarrays with K Different Integers).

**Sliding Window Maximum** needs a monotonic deque (section 11).

**Complexity:** O(n) because each index enters and leaves the window at most once.

**Loopholes**

- Sliding window requires **monotonicity**: expanding never fixes a violation and shrinking never breaks validity. With **negative numbers** this fails; use prefix sums + hash map (or monotonic deque) instead.
- Update the answer at the right moment: for "longest valid" update after restoring validity; for "shortest valid" update inside the shrink loop.
- `last[ch] >= l` check in the unique-substring version: without it a stale index drags `l` backwards.
- Window size off-by-one: length is `r - l + 1`.

**Practice:** Longest Substring Without Repeating Characters, Minimum Window Substring, Longest Repeating Character Replacement, Permutation in String, Fruit Into Baskets, Max Consecutive Ones III, Subarrays with K Different Integers.

---

## 6. Prefix sums, suffix techniques and difference arrays

**Prefix sum:** `P[0] = 0`, `P[i+1] = P[i] + a[i]`. Then `sum(l..r) = P[r+1] - P[l]` in O(1).

```python
def build_prefix(a):
    P = [0] * (len(a) + 1)
    for i, x in enumerate(a): P[i + 1] = P[i] + x
    return P
```

**2D prefix sum:** `S[i][j]` is the sum of the rectangle from (0,0) to (i-1,j-1). Query rectangle (r1,c1)-(r2,c2) = `S[r2+1][c2+1] - S[r1][c2+1] - S[r2+1][c1] + S[r1][c1]` (inclusion-exclusion).

**Prefix/suffix product (Product of Array Except Self, no division)**

```python
def product_except_self(a):
    n = len(a); res = [1] * n
    left = 1
    for i in range(n): res[i] = left; left *= a[i]
    right = 1
    for i in range(n - 1, -1, -1): res[i] *= right; right *= a[i]
    return res
```

**Difference array: many range updates in O(1) each**

```python
def apply_updates(n, updates):          # updates: (l, r, val) means a[l..r] += val
    d = [0] * (n + 1)
    for l, r, v in updates:
        d[l] += v; d[r + 1] -= v
    for i in range(1, n): d[i] += d[i - 1]   # prefix sum of d reconstructs the array
    return d[:n]
```

Use for: Corporate Flight Bookings, Car Pooling, Range Addition, meeting-room counts.

**Subarray sum divisible by K:** store counts of `prefix % k`; two equal remainders mean the subarray between them is divisible.

**Loopholes**

- Prefix array has size n + 1 (index shift is the usual bug).
- Initialize the hash map with `{0: 1}`.
- In C++/Java `%` of a negative number is negative; use `((x % k) + k) % k`.
- Large sums overflow 32-bit ints.
- Difference-array bound `r + 1` can equal n; allocate n + 1.

**Practice:** Range Sum Query, Subarray Sum Equals K, Continuous Subarray Sum, Product of Array Except Self, Range Addition, Car Pooling, Matrix Block Sum.

---

## 7. Binary search and binary search on answer

**Idea:** repeatedly halve a search space where a yes/no predicate is monotone.

```
F F F F T T T T      find the first T
        ^ lo/hi converge here
```

**Template: first index where predicate is true (works for lower\_bound, upper\_bound, everything)**

```python
def first_true(lo, hi, pred):       # search in [lo, hi), returns hi if none
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if pred(mid): hi = mid
        else: lo = mid + 1
    return lo
```

- `lower_bound(x)` = first index with `a[i] >= x`; `upper_bound(x)` = first index with `a[i] > x`.
- Count of x = `upper_bound - lower_bound`.

**Search in rotated sorted array**

```python
def search_rotated(a, t):
    l, r = 0, len(a) - 1
    while l <= r:
        m = (l + r) // 2
        if a[m] == t: return m
        if a[l] <= a[m]:                       # left half sorted
            if a[l] <= t < a[m]: r = m - 1
            else: l = m + 1
        else:                                   # right half sorted
            if a[m] < t <= a[r]: l = m + 1
            else: r = m - 1
    return -1
```

**Binary search on answer (an interview favourite)**

Recognize: "minimize the maximum", "maximize the minimum", "smallest X such that it is possible", "capacity", "speed", "days".

Recipe: (1) define `feasible(x)` that is monotone, (2) set bounds `[lo, hi]` from the problem, (3) binary search for the first feasible x.

```python
# Koko Eating Bananas: min speed k so all piles finish in h hours
def min_speed(piles, h):
    def ok(k): return sum((p + k - 1) // k for p in piles) <= h
    return first_true(1, max(piles) + 1, ok)

# Capacity To Ship Packages Within D Days
def ship(weights, D):
    def ok(cap):
        days, cur = 1, 0
        for w in weights:
            if cur + w > cap: days += 1; cur = 0
            cur += w
        return days <= D
    return first_true(max(weights), sum(weights) + 1, ok)
```

Same pattern: Split Array Largest Sum, Aggressive Cows, Allocate Books, Median of two sorted arrays (partition search), integer square root.

**Complexity:** O(log(range) \* cost of feasible).

**Loopholes**

- `lo = mid` with `mid = (lo + hi) // 2` loops forever; use `lo = mid + 1` or round mid up.
- Pick the lower bound correctly (for ship capacity it is `max(weights)`, not 1).
- Mid overflow in C++/Java.
- Duplicates in rotated array degrade to O(n) in the worst case.
- Floating-point answers: loop a fixed 50 to 100 iterations instead of `while hi - lo > eps`.
- If `feasible` is not monotone the technique does not apply: verify before coding.

**Practice:** Binary Search, Search Insert Position, First and Last Position, Search in Rotated Sorted Array, Find Peak Element, Koko Eating Bananas, Split Array Largest Sum, Median of Two Sorted Arrays.

---

## 8. Fast and slow pointers (Floyd's tortoise and hare)

**Recognize:** "cycle", "middle of linked list", "kth from end", "happy number", "duplicate number with O(1) space".

```
head -> 1 -> 2 -> 3 -> 4 -> 5
                   ^         |
                   +---------+      cycle
slow moves 1 step, fast moves 2 steps; if they ever meet there is a cycle
```

**Detect cycle and find its start**

```python
def cycle_start(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next; fast = fast.next.next
        if slow is fast:
            p = head
            while p is not slow:           # restart one pointer from head, move both 1 step
                p = p.next; slow = slow.next
            return p
    return None
```

**Why the start is found:** let a = distance head to cycle start, b = distance cycle start to meeting point, c = cycle length. Fast travelled twice slow: 2(a + b) = a + b + kc, so a = kc - b. Walking a steps from the head and a steps from the meeting point (which is the same as kc - b steps around the cycle) both land on the cycle start.

**Middle of list:** when fast reaches the end, slow is at the middle (second middle for even length with the loop above).

**Find the Duplicate Number (array as a linked list)**: treat `i -> a[i]`; the duplicate is the cycle entry.

**Complexity:** O(n) time, O(1) space.

**Loopholes**

- Always guard `fast and fast.next` before `fast.next.next`.
- Even vs odd length decides which middle you return; write down which one the problem wants.
- The two pointers must start at the same node for the start-finding proof.

**Practice:** Linked List Cycle I/II, Middle of Linked List, Happy Number, Find the Duplicate Number, Palindrome Linked List.

---

## 9. Linked lists and linked-list reversal

**Idea:** nodes connected by pointers. O(1) insert/delete when you hold the node, O(n) access by index.

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val, self.next = val, next
```

**Recognize:** "in-place", "reverse", "merge", "reorder", "remove nth from end", "palindrome list".

**Reverse a list (the most important template)**

```
prev=None  cur=1 -> 2 -> 3 -> None
step 1:    None <- 1    2 -> 3        prev=1, cur=2
step 2:    None <- 1 <- 2    3        prev=2, cur=3
step 3:    None <- 1 <- 2 <- 3        prev=3, cur=None  -> return prev
```

```python
def reverse(head):
    prev, cur = None, head
    while cur:
        nxt = cur.next      # save
        cur.next = prev     # flip
        prev, cur = cur, nxt
    return prev
```

**Recursive version:** `reverse(head.next)` returns the new head; then `head.next.next = head; head.next = None`. O(n) stack space.

**Dummy node trick:** put a fake node before head so that deleting or inserting at the head needs no special case.

```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    fast = slow = dummy
    for _ in range(n): fast = fast.next       # gap of n between the pointers
    while fast.next:
        fast = fast.next; slow = slow.next
    slow.next = slow.next.next
    return dummy.next
```

**Merge two sorted lists**

```python
def merge(a, b):
    dummy = tail = ListNode()
    while a and b:
        if a.val <= b.val: tail.next, a = a, a.next
        else: tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next
```

**Reverse nodes in k-group, reorder list, palindrome list:** find the middle with fast/slow, reverse the second half, compare or interleave, optionally restore.

**Complexity:** reversal and merge are O(n) time, O(1) space.

**Loopholes**

- Save `cur.next` **before** overwriting it, or you lose the rest of the list.
- Null checks: empty list, single node, `n` equal to the list length (head removal, solved by the dummy node).
- Cycles cause infinite loops in your own tests; never print a list blindly.
- Returning `prev` (new head) rather than the old `head`.
- Doubly linked lists: update **both** `next` and `prev`, in the right order.
- In C++ remember to free removed nodes; in Java it is garbage collected.

**Practice:** Reverse Linked List I/II, Merge Two Sorted Lists, Merge K Sorted Lists (heap), Remove Nth Node From End, Reorder List, Palindrome Linked List, Add Two Numbers, Copy List with Random Pointer, LRU Cache.

---

## 10. Stack and monotonic stack

**Idea:** LIFO. Push, pop, peek are O(1).

**Recognize (plain stack):** "matching brackets", "evaluate expression", "undo", "nested structure", "simplify path", "decode string".

```python
def valid_parentheses(s):
    pair, st = {')': '(', ']': '[', '}': '{'}, []
    for ch in s:
        if ch in pair:
            if not st or st.pop() != pair[ch]: return False
        else:
            st.append(ch)
    return not st          # leftover openers mean invalid
```

**Monotonic stack:** keep the stack sorted (increasing or decreasing). Each element is pushed once and popped once, so the whole algorithm is O(n).

**Recognize:** "next greater/smaller element", "previous smaller", "daily temperatures", "largest rectangle in histogram", "stock span", "trapping rain water".

```
Next greater element, scanning left to right with a decreasing stack of indices:
 a = [2, 1, 5, 3]
 i=0 push 0          stack: [0]
 i=1 1<2 push 1      stack: [0,1]
 i=2 5>1 pop1->ans[1]=5; 5>2 pop0->ans[0]=5; push 2
 i=3 3<5 push 3      stack: [2,3]  (rest have no greater -> -1)
```

```python
def next_greater(a):
    ans, st = [-1] * len(a), []             # st holds indices with decreasing values
    for i, x in enumerate(a):
        while st and a[st[-1]] < x:
            ans[st.pop()] = x
        st.append(i)
    return ans
```

**Largest Rectangle in Histogram**

```python
def largest_rect(h):
    st, best = [], 0                         # increasing stack of indices
    h = h + [0]                              # sentinel flushes the stack
    for i, x in enumerate(h):
        while st and h[st[-1]] >= x:
            height = h[st.pop()]
            left = st[-1] if st else -1
            best = max(best, height * (i - left - 1))
        st.append(i)
    return best
```

Width when popping bar `j` is from `left + 1` to `i - 1`, because the stack below `j` is the previous strictly smaller bar and `i` is the next smaller-or-equal bar.

**Min Stack (O(1) getMin):** store `(value, current_min)` pairs.

**Loopholes**

- Pop from an empty stack raises an error; check `if st`.
- Strict vs non-strict comparison (`<` vs `<=`) changes how duplicates are handled; reason about it explicitly.
- Store **indices**, not values, when you need distances or widths.
- Circular arrays (Next Greater Element II): iterate `2 * n` and use `i % n`.
- Expression evaluation: precedence, unary minus, spaces, integer division truncating toward zero.

**Practice:** Valid Parentheses, Min Stack, Daily Temperatures, Next Greater Element I/II, Largest Rectangle in Histogram, Maximal Rectangle, Trapping Rain Water, Decode String, Basic Calculator, Remove K Digits.

---

## 11. Queue and deque

**Queue (FIFO):** BFS, level order, task scheduling, producer-consumer. In Python use `collections.deque`; never `list.pop(0)`.

**Deque (double-ended):** O(1) at both ends. **Monotonic deque** keeps candidates in order for sliding-window extremes.

```
Sliding Window Maximum, k = 3, a = [1, 3, -1, -3, 5]
deque holds indices, values decreasing front to back
 front = current window maximum
```

```python
from collections import deque
def max_sliding_window(a, k):
    dq, res = deque(), []
    for i, x in enumerate(a):
        while dq and a[dq[-1]] <= x: dq.pop()       # smaller values can never be a max again
        dq.append(i)
        if dq[0] <= i - k: dq.popleft()             # front index left the window
        if i >= k - 1: res.append(a[dq[0]])
    return res
```

**Other patterns:** implement a queue using two stacks (amortized O(1)), circular queue with a fixed array and `(head + size) % cap`, rotting oranges and 0-1 BFS (deque with weights 0/1: push 0-edges to the front).

**Complexity:** each index is pushed and popped once, so O(n).

**Loopholes**

- Distinguish queue empty vs full in a circular buffer (keep a `size` counter).
- Remove expired indices from the front **and** dominated values from the back.
- BFS: mark nodes visited **when you enqueue**, not when you dequeue, or the queue grows exponentially.

**Practice:** Sliding Window Maximum, Implement Queue using Stacks, Design Circular Queue, Shortest Subarray with Sum at least K, Rotting Oranges, 01 Matrix.

---

## 12. Heap / priority queue and Top-K

**Idea:** a complete binary tree stored in an array where each parent is at most its children (min-heap). `push` and `pop` are O(log n), `peek` is O(1), building a heap from n items is O(n).

```
index:    0  1  2  3  4  5
value:   [1, 3, 2, 7, 4, 5]        parent(i) = (i-1)//2,  children = 2i+1, 2i+2
          1
        /   \
       3     2
      / \   /
     7   4 5
```

Python's `heapq` is a **min-heap**. For a max-heap push negatives.

**Recognize:** "top K", "kth largest/smallest", "merge K sorted", "median of a stream", "schedule by priority", "closest points".

**Top-K largest with a size-K min-heap: O(n log k)**

```python
import heapq
def top_k(a, k):
    h = []
    for x in a:
        heapq.heappush(h, x)
        if len(h) > k: heapq.heappop(h)      # evict the smallest, keeping the k largest
    return h                                  # h[0] is the kth largest
```

**Merge K sorted lists**

```python
def merge_k(lists):
    h = [(l.val, i, l) for i, l in enumerate(lists) if l]   # index i breaks ties; ListNode is not comparable
    heapq.heapify(h)
    dummy = tail = ListNode()
    while h:
        _, i, node = heapq.heappop(h)
        tail.next = node; tail = node
        if node.next: heapq.heappush(h, (node.next.val, i, node.next))
    return dummy.next
```

**Median from a data stream (two heaps)**

```python
class MedianFinder:
    def __init__(self): self.lo, self.hi = [], []     # lo = max-heap (negated), hi = min-heap
    def add(self, x):
        heapq.heappush(self.lo, -x)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))
    def median(self):
        if len(self.lo) > len(self.hi): return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

**Quickselect** finds the kth element in O(n) average, O(n^2) worst case (random pivot mitigates).

**Complexity table**

| Approach | Time | Space |
| --- | --- | --- |
| Sort | O(n log n) | O(1) to O(n) |
| Min-heap of size k | O(n log k) | O(k) |
| Quickselect | O(n) average | O(1) |
| Heap build (heapify) | O(n) | O(1) |

**Loopholes**

- Heap is **not** sorted; only the root is guaranteed.
- Min-heap for K *largest*, max-heap for K *smallest* (counter-intuitive, a frequent mistake).
- Python tuple comparison falls through to the next element on ties; add a unique tie-breaker if later elements are not comparable.
- Removing an arbitrary element is O(n); use lazy deletion (mark and skip when popped).
- Java `PriorityQueue` is a min-heap by default; supply a comparator and avoid `a - b` overflow, use `Integer.compare`.

**Practice:** Kth Largest Element, Top K Frequent Elements, Merge K Sorted Lists, Find Median from Data Stream, K Closest Points, Task Scheduler, Reorganize String, Meeting Rooms II.

---

## 13. Intervals and merge intervals

**Recognize:** "overlapping", "meeting rooms", "merge", "insert interval", "minimum arrows", "free time".

**Core idea:** **sort by start**, then sweep, comparing the current start with the previous end.

```
sorted:  [1,3] [2,6] [8,10] [9,12]
          |---|
            |-----|           overlap since 2 <= 3 -> merge to [1,6]
                     |--| |--|  [8,10] and [9,12] overlap -> [8,12]
```

```python
def merge_intervals(iv):
    iv.sort(key=lambda x: x[0])
    res = [iv[0]]
    for s, e in iv[1:]:
        if s <= res[-1][1]: res[-1][1] = max(res[-1][1], e)    # max: the new one may be nested
        else: res.append([s, e])
    return res
```

**Meeting Rooms II (minimum rooms)**: sort starts and ends separately and sweep, or use a min-heap of end times.

```python
def min_rooms(iv):
    iv.sort()
    h = []
    for s, e in iv:
        if h and h[0] <= s: heapq.heapreplace(h, e)     # reuse the earliest-free room
        else: heapq.heappush(h, e)
    return len(h)
```

**Non-overlapping intervals / activity selection (greedy):** sort by **end**, keep the interval if its start is at least the last kept end.

**Complexity:** O(n log n) from the sort.

**Loopholes**

- Decide whether touching intervals (`[1,2]` and `[2,3]`) overlap; the problem statement tells you (`<=` vs `<`).
- Use `max` of ends when merging, not simply the new end.
- Sort by start for merging, by end for maximum non-overlapping selection.
- Empty input; single interval; intervals given as tuples cannot be mutated in place.

**Practice:** Merge Intervals, Insert Interval, Non-overlapping Intervals, Meeting Rooms I/II, Minimum Number of Arrows to Burst Balloons, Interval List Intersections, Employee Free Time.

---

## 14. Sorting

| Algorithm | Best | Average | Worst | Space | Stable | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Bubble | O(n) | O(n^2) | O(n^2) | O(1) | Yes | early-exit flag gives O(n) best |
| Selection | O(n^2) | O(n^2) | O(n^2) | O(1) | No | minimal swaps |
| Insertion | O(n) | O(n^2) | O(n^2) | O(1) | Yes | great for small or nearly sorted data |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | linked lists, external sorting |
| Quick | O(n log n) | O(n log n) | O(n^2) | O(log n) | No | fastest in practice; random pivot |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | No | in place, poor cache behaviour |
| Counting | O(n + k) | O(n + k) | O(n + k) | O(k) | Yes | integers in a small range |
| Radix | O(d(n + k)) |  |  | O(n + k) | Yes | digit by digit |

**Stable** means equal keys keep their original relative order. It matters when sorting by multiple keys.

**Merge sort**

```python
def merge_sort(a):
    if len(a) <= 1: return a
    m = len(a) // 2
    L, R = merge_sort(a[:m]), merge_sort(a[m:])
    i = j = 0; out = []
    while i < len(L) and j < len(R):
        if L[i] <= R[j]: out.append(L[i]); i += 1      # <= keeps it stable
        else: out.append(R[j]); j += 1
    return out + L[i:] + R[j:]
```

Merge sort doubles as the tool for **counting inversions** (count `len(L) - i` whenever you take from R).

**Quick sort (Lomuto partition) and quickselect**

```python
import random
def quick_sort(a, lo, hi):
    if lo >= hi: return
    p = random.randint(lo, hi); a[p], a[hi] = a[hi], a[p]   # random pivot avoids the O(n^2) sorted-input trap
    pivot, i = a[hi], lo
    for j in range(lo, hi):
        if a[j] < pivot: a[i], a[j] = a[j], a[i]; i += 1
    a[i], a[hi] = a[hi], a[i]
    quick_sort(a, lo, i - 1); quick_sort(a, i + 1, hi)
```

**Language sorts:** Python's Timsort and Java's object sort are stable, O(n log n) (hybrid merge and insertion). Java's primitive `Arrays.sort` is dual-pivot quicksort. C++ `std::sort` is introsort (quick + heap + insertion), **not** stable; use `stable_sort` if needed.

**Pick by situation:** nearly sorted → insertion; limited memory → heap or quick; need stability → merge; small integer range → counting; linked list → merge; huge data on disk → external merge sort.

**Loopholes**

- Quick sort worst case on sorted input with a first/last pivot; use a random or median-of-three pivot.
- Recursion depth on quick sort: recurse into the smaller half first for O(log n) stack.
- Custom comparators must be consistent (transitive); a broken comparator crashes Java's TimSort.
- Sorting strings by custom rule (Largest Number): compare `a + b` with `b + a`.
- Sorting is often the hidden first step that turns O(n^2) into O(n log n): two pointers, intervals, greedy.

**Practice:** Sort an Array, Sort Colors, Merge Sorted Array, Largest Number, Count of Smaller Numbers After Self, Reverse Pairs, Kth Largest Element.

---

## 15. Matrix problems

**Recognize:** grid, 2D array, "rotate", "spiral", "search in matrix", "islands", "flood fill", "shortest path in a grid".

**Direction arrays (memorize):**

```python
DIRS4 = [(1, 0), (-1, 0), (0, 1), (0, -1)]
DIRS8 = DIRS4 + [(1, 1), (1, -1), (-1, 1), (-1, -1)]
```

**Rotate image 90 degrees clockwise in place = transpose, then reverse each row**

```python
def rotate(m):
    n = len(m)
    for i in range(n):
        for j in range(i + 1, n): m[i][j], m[j][i] = m[j][i], m[i][j]
    for row in m: row.reverse()
```

**Spiral traversal:** maintain four boundaries `top, bottom, left, right` and shrink them after each side. Check `top <= bottom` and `left <= right` before the reverse passes to avoid duplicates on single rows/columns.

**Search a 2D matrix**

- Rows and columns each sorted, and the first of a row is greater than the last of the previous: treat as a flattened array, binary search with `row = mid // cols, col = mid % cols` (O(log mn)).
- Rows sorted and columns sorted only: start at the top-right, move left if too big, down if too small (O(m + n)).

**Number of Islands (grid DFS/BFS)**

```python
def num_islands(g):
    R, C, cnt = len(g), len(g[0]), 0
    def dfs(r, c):
        if not (0 <= r < R and 0 <= c < C) or g[r][c] != '1': return
        g[r][c] = '0'                       # mark visited by sinking the land
        for dr, dc in DIRS4: dfs(r + dr, c + dc)
    for r in range(R):
        for c in range(C):
            if g[r][c] == '1': cnt += 1; dfs(r, c)
    return cnt
```

**Set Matrix Zeroes in O(1) space:** use the first row and column as markers, with two flags for whether they themselves must be zeroed.

**Loopholes**

- `m[i][j]` is **row, column**; mixing up `(x, y)` and `(row, col)` is the classic bug.
- Build a matrix with `[[0] * C for _ in range(R)]`, **not** `[[0] * C] * R` (all rows alias the same list).
- Empty matrix or empty first row (`len(g[0])` crashes).
- Deep recursive DFS on a 10^6 grid overflows the stack; use BFS or an explicit stack.
- Mutating the input grid might not be allowed; use a visited set instead.
- BFS for shortest path on an unweighted grid; DFS does **not** give shortest paths.

**Practice:** Rotate Image, Spiral Matrix, Set Matrix Zeroes, Search a 2D Matrix I/II, Number of Islands, Flood Fill, Word Search, Rotting Oranges, Shortest Path in Binary Matrix, Pacific Atlantic Water Flow.

---

## 16. Trees, BST, tree DFS and tree BFS

**Definitions:** a tree is a connected acyclic graph. A binary tree has at most two children. A **BST** keeps `left < node < right` for **every** node (not just the immediate children).

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val, self.left, self.right = val, left, right
```

**Types:** full (0 or 2 children), complete (filled level by level, left to right; heaps), perfect (all levels full: 2^h - 1 nodes), balanced (height O(log n); AVL, red-black), degenerate (a linked list).

**DFS traversals**

```
        1
       / \
      2   3         preorder  (root, L, R): 1 2 4 5 3
     / \            inorder   (L, root, R): 4 2 5 1 3   <- sorted order for a BST
    4   5           postorder (L, R, root): 4 5 2 3 1
```

```python
def inorder(root):
    res = []
    def go(n):
        if not n: return
        go(n.left); res.append(n.val); go(n.right)
    go(root); return res

def inorder_iterative(root):
    res, st, cur = [], [], root
    while cur or st:
        while cur: st.append(cur); cur = cur.left
        cur = st.pop(); res.append(cur.val); cur = cur.right
    return res
```

**The "return a value from a subtree" pattern (most tree DFS problems)**

```python
def height(n):                         # max depth
    return 0 if not n else 1 + max(height(n.left), height(n.right))

def diameter(root):                    # longest path in edges
    best = 0
    def depth(n):
        nonlocal best
        if not n: return 0
        l, r = depth(n.left), depth(n.right)
        best = max(best, l + r)        # path bending at n
        return 1 + max(l, r)           # path going up from n
    depth(root); return best

def lca(root, p, q):                   # lowest common ancestor (general tree)
    if not root or root is p or root is q: return root
    L, R = lca(root.left, p, q), lca(root.right, p, q)
    return root if L and R else (L or R)
```

For a BST the LCA is found by comparing with the root: both smaller go left, both larger go right, otherwise the root is the LCA.

**BFS / level order**

```python
from collections import deque
def level_order(root):
    if not root: return []
    res, q = [], deque([root])
    while q:
        level = []
        for _ in range(len(q)):          # fix the level size before popping
            n = q.popleft(); level.append(n.val)
            if n.left: q.append(n.left)
            if n.right: q.append(n.right)
        res.append(level)
    return res
```

Use BFS for: level order, zigzag, right-side view, minimum depth, width of tree.

**Validate BST (pass bounds, not just parent comparisons)**

```python
def is_bst(n, lo=float('-inf'), hi=float('inf')):
    if not n: return True
    if not (lo < n.val < hi): return False
    return is_bst(n.left, lo, n.val) and is_bst(n.right, n.val, hi)
```

**BST operations:** search, insert, delete in O(h). Delete cases: leaf (remove), one child (replace by the child), two children (replace with the inorder successor, the minimum of the right subtree, then delete that node). Balanced trees keep h = O(log n); a degenerate BST makes h = n.

**Build a tree from traversals:** preorder + inorder uniquely determine a binary tree: the first preorder element is the root; find it in inorder to split left/right sizes. (Preorder + postorder only determine a *full* binary tree.)

**Serialize and deserialize:** preorder with null markers like `1,2,#,#,3,#,#`.

**Complexity:** traversals O(n) time, O(h) space (O(n) skewed, O(log n) balanced). BFS uses O(width).

**Loopholes**

- Validating a BST by checking only `left.val < node.val < right.val` is **wrong**; a deep descendant can violate the ancestor. Use bounds or inorder strictly increasing.
- Duplicates in BST: the problem must define the rule (left or right or counted).
- Integer bounds: use `-inf/inf`, not `INT_MIN`, since node values can equal `INT_MIN`.
- Height of an empty tree is 0 (counted in nodes) or -1 (counted in edges); be consistent.
- Recursion depth of a skewed tree may overflow; use iterative or Morris traversal (O(1) space).
- Distinguish "path" definitions: root-to-leaf, any-to-any, must contain at least one node. Max path sum must allow negative values to be dropped (`max(0, child)`).

**Practice:** Max Depth, Invert Tree, Same Tree, Symmetric Tree, Level Order, Validate BST, Kth Smallest in BST, LCA (BST and binary tree), Diameter, Path Sum I/II/III, Binary Tree Maximum Path Sum, Construct from Preorder and Inorder, Serialize/Deserialize, Right Side View.

---

## 17. Graphs

**Vocabulary:** vertex/node, edge, directed/undirected, weighted, degree (in/out), path, cycle, connected component, DAG (directed acyclic graph), bipartite (2-colourable, no odd cycle), sparse (E near V) vs dense (E near V^2).

**Representations**

| Representation | Space | Edge check | Iterate neighbours | Use when |
| --- | --- | --- | --- | --- |
| Adjacency list | O(V + E) | O(deg) | O(deg) | almost always (sparse) |
| Adjacency matrix | O(V^2) | O(1) | O(V) | dense graphs, Floyd-Warshall |
| Edge list | O(E) | O(E) | O(E) | Kruskal, Bellman-Ford |

```python
from collections import defaultdict, deque
g = defaultdict(list)
for u, v in edges:
    g[u].append(v)
    g[v].append(u)          # remove this line for a directed graph
```

**Recognize graph problems in disguise:** grid (cells are nodes), word ladder (words are nodes), course prerequisites, accounts merge, "connected", "shortest number of steps", dependencies, network delay.

### 17.1 Graph BFS and DFS

```
   1 --- 2 --- 5
   |     |
   3 --- 4        BFS from 1: 1 | 2 3 | 4 5      (level by level, queue)
                  DFS from 1: 1 2 4 3 ... 5      (go deep first, stack/recursion)
```

```python
def bfs(g, src):
    dist, q = {src: 0}, deque([src])
    while q:
        u = q.popleft()
        for v in g[u]:
            if v not in dist:             # mark when ENQUEUING
                dist[v] = dist[u] + 1
                q.append(v)
    return dist                            # shortest hops in an UNWEIGHTED graph

def dfs(g, u, seen):
    seen.add(u)
    for v in g[u]:
        if v not in seen: dfs(g, v, seen)
```

**Complexity:** O(V + E) time, O(V) space.

**Cycle detection**

- Undirected: DFS and see a visited neighbour that is **not the parent** (or DSU).
- Directed: DFS with three colours (0 unvisited, 1 in the current recursion stack, 2 done). Reaching a node with colour 1 means a cycle (a back edge).

**Bipartite check:** BFS/DFS two-colouring; if an edge joins equal colours it is not bipartite.

**Clone graph, connected components, word ladder (BFS on strings), 0-1 BFS, multi-source BFS (push all sources first, e.g. Rotting Oranges).**

**Loopholes**

- Graph may be **disconnected**: loop over all nodes and start a traversal from each unvisited one.
- Marking visited on dequeue instead of enqueue produces duplicates and TLE.
- Undirected cycle detection: ignoring the parent edge is required, but with **multi-edges** you must compare edge ids, not just the parent vertex.
- Self-loops, parallel edges, 0-indexed vs 1-indexed nodes.
- BFS gives shortest paths only when all edges have equal weight.
- Recursive DFS can hit the recursion limit on 10^5 nodes.

**Practice:** Number of Islands, Clone Graph, Course Schedule I/II, Word Ladder, Is Graph Bipartite, Pacific Atlantic Water Flow, Number of Provinces, Surrounded Regions.

### 17.2 Topological sort (DAG only)

**Idea:** a linear ordering where every edge u -> v has u before v. Exists **iff** the graph is a DAG.

**Recognize:** "prerequisites", "build order", "task dependencies", "alien dictionary".

**Kahn's algorithm (BFS on in-degrees)**

```python
def topo_sort(n, edges):
    g = [[] for _ in range(n)]; indeg = [0] * n
    for u, v in edges: g[u].append(v); indeg[v] += 1
    q = deque(i for i in range(n) if indeg[i] == 0)
    order = []
    while q:
        u = q.popleft(); order.append(u)
        for v in g[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return order if len(order) == n else []     # fewer than n nodes means a cycle
```

**DFS version:** post-order, then reverse. Detect cycles with the 3-colour method.

**Complexity:** O(V + E).

**Loopholes:** the order is **not unique**; use a min-heap instead of a queue if the problem wants lexicographically smallest. Always compare `len(order)` with `n` to detect cycles. In Alien Dictionary, a longer word before its own prefix (`abc` before `ab`) is invalid.

**Practice:** Course Schedule I/II, Alien Dictionary, Parallel Courses, Find Eventual Safe States, Minimum Height Trees.

### 17.3 Union-Find / Disjoint Set Union (DSU)

**Idea:** maintain a partition of elements into sets with two operations: `find(x)` returns the set representative, `union(a, b)` merges two sets.

```
parent: 0 1 2 3 4        union(0,1) union(1,2):   0 <- 1, 0 <- 2   (one tree rooted at 0)
```

**Optimizations (always mention both):**

1. **Path compression:** during `find`, point every visited node directly to the root.
2. **Union by rank / size:** attach the smaller tree under the larger.

Together: amortized O(alpha(n)) per operation, where alpha is the inverse Ackermann function (below 5 for any practical n).

```python
class DSU:
    def __init__(self, n):
        self.p = list(range(n)); self.size = [1] * n; self.count = n
    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]        # path halving (iterative compression)
            x = self.p[x]
        return x
    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb: return False                # already connected: this edge would form a cycle
        if self.size[ra] < self.size[rb]: ra, rb = rb, ra
        self.p[rb] = ra; self.size[ra] += self.size[rb]; self.count -= 1
        return True
```

**Use for:** connected components counting, cycle detection in an undirected graph, Kruskal, Redundant Connection, Accounts Merge, Number of Islands II (online), Satisfiability of Equality Equations.

**Loopholes**

- Compare **roots**, never raw parents.
- Without union by size and compression, a chain makes `find` O(n).
- DSU cannot **split** sets (no deletion); for offline deletions process queries in reverse.
- DSU works for undirected connectivity only; directed reachability needs SCC.
- Remember to map non-integer ids (strings, emails) to indices first.

**Practice:** Number of Provinces, Redundant Connection, Accounts Merge, Graph Valid Tree, Most Stones Removed, Making a Large Island.

### 17.4 Dijkstra's algorithm (single-source shortest path, non-negative weights)

**Idea:** greedily finalize the unvisited node with the smallest known distance, then relax its edges. A min-heap provides the next node.

```python
import heapq
def dijkstra(g, src, n):                          # g[u] = [(v, w), ...]
    dist = [float('inf')] * n; dist[src] = 0
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue                  # stale entry (lazy deletion)
        for v, w in g[u]:
            if d + w < dist[v]:
                dist[v] = d + w
                heapq.heappush(pq, (dist[v], v))
    return dist
```

To rebuild the path, store `parent[v] = u` on each relaxation.

**Complexity:** O((V + E) log V) with a binary heap, O(V^2) with an array (better for dense graphs).

**Why it works:** when a node is popped with the smallest tentative distance, no other route can be shorter because all remaining edges are non-negative. This argument **breaks with negative edges**.

**Loopholes**

- **Negative weights:** Dijkstra gives wrong answers; use Bellman-Ford.
- Skip stale heap entries (`d > dist[u]`) or the complexity degrades.
- Use `inf` carefully so that `inf + w` does not overflow in C++/Java.
- For unweighted graphs use BFS; for weights 0/1 use 0-1 BFS.
- Variants: grid with costs (Path With Minimum Effort: minimize the max edge, still Dijkstra with `max`), K stops (Cheapest Flights Within K Stops: use Bellman-Ford with K+1 rounds or Dijkstra with state `(node, stops)`).

**Practice:** Network Delay Time, Path With Minimum Effort, Cheapest Flights Within K Stops, Swim in Rising Water, Shortest Path in Weighted Graph.

### 17.5 Minimum spanning tree: Kruskal's and Prim's

**MST:** a subset of edges connecting all V vertices with minimum total weight and no cycles (V - 1 edges). Applies to connected, undirected, weighted graphs.

**Kruskal (sort edges + DSU):**

```python
def kruskal(n, edges):                 # edges: (w, u, v)
    dsu, total, used = DSU(n), 0, 0
    for w, u, v in sorted(edges):
        if dsu.union(u, v):
            total += w; used += 1
    return total if used == n - 1 else -1       # -1 if the graph is disconnected
```

Complexity: O(E log E).

**Prim (grow one tree with a min-heap):**

```python
def prim(g, n):                        # g[u] = [(v, w)]
    seen, pq, total = [False] * n, [(0, 0)], 0
    while pq:
        w, u = heapq.heappop(pq)
        if seen[u]: continue
        seen[u] = True; total += w
        for v, wt in g[u]:
            if not seen[v]: heapq.heappush(pq, (wt, v))
    return total
```

Complexity: O(E log V).

**Cut and cycle properties:** the lightest edge crossing any cut is in some MST; the heaviest edge on any cycle is not in the MST. If all weights are distinct the MST is unique.

**Kruskal vs Prim:** Kruskal is simple with an edge list and sparse graphs; Prim suits dense graphs and adjacency structures.

**Loopholes:** MST gives a minimum **total**, not shortest paths between nodes (do not confuse it with Dijkstra). Disconnected graphs yield a spanning forest. Negative weights are fine for MST.

**Practice:** Min Cost to Connect All Points, Connecting Cities With Minimum Cost, Critical and Pseudo-Critical Edges.

### 17.6 Bellman-Ford and Floyd-Warshall

**Bellman-Ford (single source, handles negative edges, detects negative cycles)**

```python
def bellman_ford(n, edges, src):                 # edges: (u, v, w)
    dist = [float('inf')] * n; dist[src] = 0
    for _ in range(n - 1):                       # a shortest path has at most n-1 edges
        changed = False
        for u, v, w in edges:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w; changed = True
        if not changed: break
    for u, v, w in edges:                        # an n-th improvement means a negative cycle
        if dist[u] + w < dist[v]: return None
    return dist
```

Complexity: O(V \* E). For "at most K edges" problems, run exactly K rounds on a **copy** of `dist` per round.

**Floyd-Warshall (all-pairs shortest paths)**

```python
def floyd(d):                      # d[i][j] initial weights, inf if no edge, d[i][i] = 0
    n = len(d)
    for k in range(n):             # k MUST be the outermost loop
        for i in range(n):
            for j in range(n):
                if d[i][k] + d[k][j] < d[i][j]:
                    d[i][j] = d[i][k] + d[k][j]
```

Complexity: O(V^3) time, O(V^2) space. Suitable for V up to a few hundred. A negative cycle exists if some `d[i][i] < 0`.

**Which shortest-path algorithm?**

| Situation | Use |
| --- | --- |
| Unweighted | BFS |
| Weights 0 or 1 | 0-1 BFS |
| Non-negative weights | Dijkstra |
| Negative weights, single source | Bellman-Ford |
| All pairs, small V | Floyd-Warshall |
| DAG | Topological order + relax |

**Loopholes:** `k` must be the outer loop in Floyd-Warshall (a classic bug). With negative cycles shortest paths are undefined. Avoid `inf + (-w)` overflow issues; skip relaxing from `inf`.

**Practice:** Cheapest Flights Within K Stops, Find the City With the Smallest Number of Neighbors, Network Delay Time.

### 17.7 Strongly connected components (SCC)

**Definition:** in a directed graph, a maximal set of vertices where every vertex can reach every other. Contracting each SCC gives a DAG.

**Kosaraju's algorithm (two DFS passes):**

1. DFS on the graph, push vertices on a stack in **finishing order**.
2. Build the **transpose** graph (reverse all edges).
3. Pop vertices from the stack; each unvisited one starts a DFS on the transpose that collects exactly one SCC.

```python
def kosaraju(n, g):
    seen, order = [False] * n, []
    def dfs1(u):
        seen[u] = True
        for v in g[u]:
            if not seen[v]: dfs1(v)
        order.append(u)
    for i in range(n):
        if not seen[i]: dfs1(i)
    rg = [[] for _ in range(n)]
    for u in range(n):
        for v in g[u]: rg[v].append(u)
    seen, comps = [False] * n, []
    def dfs2(u, comp):
        seen[u] = True; comp.append(u)
        for v in rg[u]:
            if not seen[v]: dfs2(v, comp)
    for u in reversed(order):
        if not seen[u]:
            comp = []; dfs2(u, comp); comps.append(comp)
    return comps
```

**Tarjan's algorithm** does it in one DFS using discovery time `disc[u]`, low-link `low[u]` and a stack; a node with `low[u] == disc[u]` is the root of an SCC. Also gives bridges (`low[v] > disc[u]`) and articulation points.

**Complexity:** O(V + E).

**Loopholes:** the second pass must use the **transpose** graph and the **reverse finishing order**. SCCs are meaningful for directed graphs only (for undirected graphs use connected components).

**Practice:** Critical Connections in a Network (Tarjan bridges), Course Schedule (cycle = SCC larger than 1), 2-SAT.

---

## 18. Recursion, backtracking and divide and conquer

### 18.1 Recursion

Every recursive solution needs (1) a **base case**, (2) a **smaller subproblem**, (3) trust that the recursive call is correct. Each call occupies a stack frame, so depth equals extra space.

```
fact(3) -> 3 * fact(2) -> 2 * fact(1) -> 1 * fact(0) -> returns 1 (base case)
```

**Tail recursion:** the recursive call is the last operation; some languages optimize it, Python and Java do not.

**Loopholes:** missing or unreachable base case gives infinite recursion and stack overflow; overlapping subproblems (naive Fibonacci) mean you need memoization; mutable arguments shared across calls.

### 18.2 Backtracking

**Idea:** build a solution incrementally; at each step **choose**, **explore**, **un-choose** (undo). Prune branches that cannot succeed.

```
Subsets of [1,2,3]: decision tree
             []
        /          \
      [1]           []
     /   \         /  \
  [1,2]  [1]     [2]   []     ... each element: include or exclude  -> 2^n leaves
```

**Universal template**

```python
def backtrack(path, choices):
    if is_solution(path):
        res.append(path[:])            # COPY the path
        return
    for c in choices:
        if not valid(c): continue      # prune
        path.append(c)                 # choose
        backtrack(path, next_choices)  # explore
        path.pop()                     # un-choose
```

**Subsets, permutations, combinations**

```python
def subsets(a):
    res = []
    def go(i, path):
        res.append(path[:])
        for j in range(i, len(a)):
            path.append(a[j]); go(j + 1, path); path.pop()
    go(0, []); return res

def permutations(a):
    res, used = [], [False] * len(a)
    def go(path):
        if len(path) == len(a): res.append(path[:]); return
        for i in range(len(a)):
            if used[i]: continue
            used[i] = True; path.append(a[i]); go(path); path.pop(); used[i] = False
    go([]); return res

def combination_sum(c, target):         # numbers reusable
    res = []
    def go(i, path, rem):
        if rem == 0: res.append(path[:]); return
        for j in range(i, len(c)):
            if c[j] > rem: continue
            path.append(c[j]); go(j, path, rem - c[j]); path.pop()   # j (not j+1) allows reuse
    go(0, [], target); return res
```

**N-Queens:** place one queen per row; track used columns and the two diagonals (`r - c` and `r + c`) in sets, giving O(1) conflict checks.

**Handling duplicates:** sort the input and skip `if j > i and a[j] == a[j-1]` inside the loop (same recursion level only).

**Complexity:** subsets O(2^n \* n), permutations O(n! \* n), N-Queens roughly O(n!) with pruning.

**Loopholes**

- Append `path[:]`, not `path`, or every stored answer ends up empty/identical.
- Forgetting to un-choose (including un-marking `used`).
- Duplicate handling: skip rule applies only to siblings at the same depth.
- Pruning order: validate before recursing, not after.
- If constraints are n up to 20 or so, backtracking or bitmask is expected; if n is 10^5 it is never backtracking.

**Practice:** Subsets I/II, Permutations I/II, Combination Sum I/II, Letter Combinations, Generate Parentheses, Palindrome Partitioning, N-Queens, Sudoku Solver, Word Search.

### 18.3 Divide and conquer

**Idea:** split into independent subproblems, solve recursively, **combine**. Differs from DP because subproblems do not overlap.

| Problem | Divide | Combine | Time |
| --- | --- | --- | --- |
| Merge sort | halves | merge | O(n log n) |
| Quick sort | partition by pivot | nothing | O(n log n) average |
| Binary search | one half | nothing | O(log n) |
| Max subarray | halves | best of left, right, crossing | O(n log n) |
| Fast exponentiation | exponent / 2 | square | O(log n) |
| Closest pair of points | by x | check strip | O(n log n) |
| Count inversions | halves | count while merging | O(n log n) |

```python
def power(x, n):                 # x^n in O(log n)
    if n < 0: return 1 / power(x, -n)
    if n == 0: return 1
    half = power(x, n // 2)
    return half * half * (x if n % 2 else 1)
```

**Loopholes:** compute the half power **once** (calling it twice makes it O(n)); negative exponents; the recurrence tells the complexity (use the Master theorem); exponent overflow, use modular arithmetic (`pow(x, n, mod)` in Python).

---

## 19. Greedy

**Idea:** at each step take the locally best choice, and **prove** it leads to a global optimum. No backtracking, no table.

**Recognize:** "minimum number of", "maximum number of" with sorting, intervals, scheduling, coins (canonical systems), jump problems, "can you reach".

**How to justify (interview answer):**

1. **Greedy-choice property:** some optimal solution contains the greedy first choice.
2. **Exchange argument:** swap any optimal solution's first choice for the greedy one without making it worse.
3. **Optimal substructure:** the remainder is the same problem, smaller.

**Templates**

```python
# Jump Game: can we reach the last index?
def can_jump(a):
    reach = 0
    for i, x in enumerate(a):
        if i > reach: return False
        reach = max(reach, i + x)
    return True

# Jump Game II: minimum jumps (BFS-by-layers greedy)
def min_jumps(a):
    jumps = cur_end = far = 0
    for i in range(len(a) - 1):
        far = max(far, i + a[i])
        if i == cur_end: jumps += 1; cur_end = far
    return jumps

# Gas Station: start index that completes the circuit
def can_complete(gas, cost):
    if sum(gas) < sum(cost): return -1
    tank = start = 0
    for i in range(len(gas)):
        tank += gas[i] - cost[i]
        if tank < 0: start = i + 1; tank = 0   # no start in [start, i] works
    return start

# Activity selection / Non-overlapping intervals: sort by END time
def max_activities(iv):
    iv.sort(key=lambda x: x[1]); last, cnt = float('-inf'), 0
    for s, e in iv:
        if s >= last: cnt += 1; last = e
    return cnt
```

**Classic greedy problems:** Huffman coding (min-heap of frequencies), Fractional Knapsack (sort by value/weight), Task Scheduler, Candy, Assign Cookies, Partition Labels, Meeting Rooms, Minimum Platforms, Job Sequencing with deadlines.

**Greedy vs DP:** if a local choice is provably safe, greedy; if choices interact and you cannot prove it, DP. Fractional knapsack is greedy; **0/1 knapsack is not** (counterexample: capacity 50; items (10kg, 60), (20kg, 100), (30kg, 120): greedy by ratio takes 10+20 = 160, optimal is 20+30 = 220).

**Complexity:** usually O(n log n) for the sort, O(n) otherwise.

**Loopholes**

- Choosing the wrong sort key (interval scheduling: by end, not by start or length).
- Coin change with arbitrary denominations is **not** greedy (coins {1, 3, 4}, amount 6: greedy gives 4+1+1 = 3 coins, optimal 3+3 = 2). Use DP.
- Always try a small counterexample before trusting a greedy idea.
- Ties: break them deterministically.

**Practice:** Jump Game I/II, Gas Station, Candy, Task Scheduler, Partition Labels, Non-overlapping Intervals, Lemonade Change, Minimum Number of Platforms, Huffman Encoding.

---

## 20. Dynamic programming

**Idea:** solve overlapping subproblems **once** and store results. Two requirements: **optimal substructure** and **overlapping subproblems**.

**Five-step recipe (say this out loud in interviews)**

1. **State:** what does `dp[i]` (or `dp[i][j]`) mean, in one sentence?
2. **Transition:** how does a state depend on smaller states?
3. **Base case(s).**
4. **Order** of computation (what must be filled first).
5. **Answer:** which cell(s) hold the result.

**Top-down (memoization) vs bottom-up (tabulation)**

```python
from functools import lru_cache

@lru_cache(None)
def fib(n):                       # top-down: recursion + cache, O(n) time
    return n if n < 2 else fib(n - 1) + fib(n - 2)

def fib_bu(n):                    # bottom-up with space optimization, O(1) space
    a, b = 0, 1
    for _ in range(n): a, b = b, a + b
    return a
```

Top-down is easier to write and only computes needed states; bottom-up avoids recursion limits and enables space optimization (rolling arrays).

```
Decision map:
  Count ways / min cost / max value / feasibility, and brute force is exponential with repeated states -> DP
  one sequence (index i)          -> 1D DP        (climbing stairs, house robber, coin change)
  two sequences or grid (i, j)    -> 2D DP        (LCS, edit distance, unique paths)
  items + capacity                -> knapsack
  subarray / substring i..j       -> interval DP
  subset of up to ~20 items       -> bitmask DP
```

### 20.1 One-dimensional DP

```python
# Climbing Stairs: ways to reach step n using 1 or 2 steps
def climb(n):
    a, b = 1, 1
    for _ in range(n - 1): a, b = b, a + b
    return b

# House Robber: max sum without taking adjacent elements
def rob(a):
    take, skip = 0, 0
    for x in a: take, skip = skip + x, max(take, skip)
    return max(take, skip)

# Coin Change (min coins; unbounded)
def coin_change(coins, amt):
    INF = amt + 1; dp = [0] + [INF] * amt
    for a in range(1, amt + 1):
        for c in coins:
            if c <= a: dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amt] if dp[amt] != INF else -1

# Word Break
def word_break(s, words):
    W, dp = set(words), [True] + [False] * len(s)
    for i in range(1, len(s) + 1):
        dp[i] = any(dp[j] and s[j:i] in W for j in range(i))
    return dp[len(s)]
```

Other 1D problems: Decode Ways, Maximum Product Subarray (track both max and min), Perfect Squares, Jump Game, Palindromic Substrings (expand around centre).

### 20.2 Two-dimensional DP

```python
# Unique Paths in an m x n grid (move right/down)
def unique_paths(m, n):
    dp = [1] * n
    for _ in range(1, m):
        for j in range(1, n): dp[j] += dp[j - 1]       # rolling 1D array
    return dp[-1]

# Minimum Path Sum
def min_path(g):
    R, C = len(g), len(g[0])
    for i in range(R):
        for j in range(C):
            if i == j == 0: continue
            up = g[i - 1][j] if i else float('inf')
            left = g[i][j - 1] if j else float('inf')
            g[i][j] += min(up, left)
    return g[-1][-1]

# Edit Distance (Levenshtein)
def edit_distance(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1): dp[i][0] = i          # delete all i chars
    for j in range(n + 1): dp[0][j] = j          # insert all j chars
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]: dp[i][j] = dp[i - 1][j - 1]
            else: dp[i][j] = 1 + min(dp[i - 1][j],      # delete
                                     dp[i][j - 1],      # insert
                                     dp[i - 1][j - 1])  # replace
    return dp[m][n]
```

### 20.3 Advanced DP

**LCS (Longest Common Subsequence)**

```
dp[i][j] = LCS length of a[:i] and b[:j]
   a[i-1] == b[j-1]  ->  dp[i-1][j-1] + 1
   otherwise         ->  max(dp[i-1][j], dp[i][j-1])
```

```python
def lcs(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            dp[i][j] = dp[i-1][j-1] + 1 if a[i-1] == b[j-1] else max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

Related: Longest Palindromic Subsequence = LCS of `s` and `reverse(s)`; Shortest Common Supersequence = `m + n - LCS`; Minimum deletions to make strings equal = `m + n - 2 * LCS`. **Subsequence** is non-contiguous; **substring** is contiguous (for Longest Common Substring reset to 0 on mismatch and track the max).

**LIS (Longest Increasing Subsequence)**

```python
# O(n^2) DP
def lis_n2(a):
    dp = [1] * len(a)
    for i in range(len(a)):
        for j in range(i):
            if a[j] < a[i]: dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)

# O(n log n): tails[k] = smallest tail of any increasing subsequence of length k+1
from bisect import bisect_left
def lis(a):
    tails = []
    for x in a:
        i = bisect_left(tails, x)          # bisect_left gives STRICTLY increasing
        if i == len(tails): tails.append(x)
        else: tails[i] = x
    return len(tails)
```

`tails` is **not** the actual subsequence, only its length is correct. For non-decreasing, use `bisect_right`.

**Knapsack family**

```python
# 0/1 knapsack: each item at most once. Iterate capacity DOWNWARD with a 1D array.
def knapsack01(wt, val, W):
    dp = [0] * (W + 1)
    for w, v in zip(wt, val):
        for c in range(W, w - 1, -1):          # downward so an item is not reused
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[W]

# Unbounded knapsack / coin change ways: iterate capacity UPWARD (reuse allowed)
def coin_ways(coins, amt):
    dp = [1] + [0] * amt
    for c in coins:                            # coins in the OUTER loop => combinations
        for a in range(c, amt + 1): dp[a] += dp[a - c]
    return dp[amt]
```

Swapping the loops (amount outer, coins inner) counts **permutations** instead of combinations. Variants: Subset Sum, Partition Equal Subset Sum, Target Sum, Last Stone Weight II, Ones and Zeroes.

**Interval DP:** `dp[i][j]` for the substring/subarray `i..j`; iterate by increasing **length**, try every split point `k` in between.

```python
# Matrix chain / Burst Balloons pattern
def burst_balloons(nums):
    a = [1] + nums + [1]; n = len(a)
    dp = [[0] * n for _ in range(n)]
    for length in range(2, n):                 # window size
        for i in range(n - length):
            j = i + length
            for k in range(i + 1, j):          # k = LAST balloon burst in (i, j)
                dp[i][j] = max(dp[i][j], dp[i][k] + a[i] * a[k] * a[j] + dp[k][j])
    return dp[0][n - 1]
```

Others: Matrix Chain Multiplication, Palindrome Partitioning II, Minimum Cost to Merge Stones, Longest Palindromic Substring. Complexity usually O(n^3).

**Bitmask DP:** encode the set of chosen items as an integer mask (bit i = item i used). Use when n is about 20 or fewer. State count is 2^n.

```python
# Travelling Salesman (shortest tour visiting every city once): O(2^n * n^2)
def tsp(dist):
    n = len(dist); INF = float('inf')
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0                                  # start at city 0, mask {0}
    for mask in range(1 << n):
        for u in range(n):
            if dp[mask][u] == INF or not (mask >> u) & 1: continue
            for v in range(n):
                if (mask >> v) & 1: continue      # already visited
                nm = mask | (1 << v)
                dp[nm][v] = min(dp[nm][v], dp[mask][u] + dist[u][v])
    return min(dp[(1 << n) - 1][u] + dist[u][0] for u in range(n))
```

Also: Partition to K Equal Sum Subsets, Shortest Path Visiting All Nodes, Assignment Problem.

**Other patterns to know:** state-machine DP (Best Time to Buy and Sell Stock with cooldown or at most k transactions: states hold/not hold/cooldown), DP on trees (House Robber III), digit DP, DP with binary search (LIS), grid DP with obstacles, longest palindromic substring.

**Complexity rule:** time = (number of states) x (work per transition).

**Loopholes**

- Define the state in words first; most bugs come from an unclear definition.
- Base-case rows/columns and `dp[0]`: off-by-one between 0-indexed strings and 1-indexed DP tables (`a[i-1]`).
- 0/1 knapsack 1D: loop capacity **downward**; unbounded: **upward**.
- Initialize with `inf` for min problems and `0` or `-inf` for max/count problems; mixing causes wrong answers.
- Memoization with mutable arguments (lists) is unhashable; pass tuples or indices.
- Python recursion depth with memoization at n = 10^4 to 10^5: switch to bottom-up.
- Counting problems often need modulo `10^9 + 7`; apply it after **every** addition/multiplication.
- Space optimization: only keep the previous row (or two variables) when the transition needs just that.
- Do not use DP when greedy suffices, and do not use greedy when a counterexample exists.

**Practice (grouped):** 1D: Climbing Stairs, House Robber I/II, Coin Change I/II, Word Break, Decode Ways, Maximum Product Subarray. 2D: Unique Paths, Min Path Sum, Edit Distance, LCS, Longest Palindromic Substring, Regex Matching. Knapsack: Partition Equal Subset Sum, Target Sum. LIS: LIS, Russian Doll Envelopes. Interval: Burst Balloons, Matrix Chain. Bitmask: TSP, Partition to K Equal Sum Subsets. Stocks: all six Best Time to Buy and Sell Stock variants.

---

## 21. Trie (prefix tree)

**Idea:** a tree where each edge is a character and each root-to-node path spells a prefix. Lookups cost O(L) in the word length, independent of how many words are stored.

```
Insert: cat, car, dog
        root
       /    \
      c      d
      |      |
      a      o
     / \     |
    t*  r*   g*          * = end of word
```

**Recognize:** "prefix", "autocomplete", "word search II", "dictionary", "longest common prefix", "maximum XOR of two numbers" (binary trie).

```python
class TrieNode:
    __slots__ = ('ch', 'end')
    def __init__(self): self.ch, self.end = {}, False

class Trie:
    def __init__(self): self.root = TrieNode()
    def insert(self, w):
        n = self.root
        for c in w:
            n = n.ch.setdefault(c, TrieNode())
        n.end = True
    def _walk(self, s):
        n = self.root
        for c in s:
            if c not in n.ch: return None
            n = n.ch[c]
        return n
    def search(self, w):
        n = self._walk(w); return bool(n and n.end)       # full word must be marked
    def starts_with(self, p):
        return self._walk(p) is not None
```

**Word Search II:** build a trie of all words, DFS the grid following trie edges; this prunes branches that are not a prefix of any word (far faster than running Word Search once per word). Delete found words from the trie to avoid duplicates.

**Complexity:** insert/search O(L); space O(total characters x alphabet) in the worst case (use a dict per node, or a 26-array for lowercase letters).

**Trie vs hash set:** a trie supports prefix queries, sorted iteration and shared prefixes; a hash set only gives exact membership but with lower memory and constant factors.

**Loopholes**

- `search` must check the **end-of-word** flag; `starts_with` must not.
- Memory is heavy with 26-wide arrays; use dicts for sparse alphabets.
- Handle deletion carefully (prune nodes with no children and no end flag), or just unmark `end`.
- Case, empty string, and duplicate insertions.

**Practice:** Implement Trie, Design Add and Search Words (with `.`), Word Search II, Replace Words, Longest Word in Dictionary, Maximum XOR of Two Numbers in an Array, Top K Frequent Words, Autocomplete System.

---

## 22. Bit manipulation

**Basics**

| Op | Meaning | Example |
| --- | --- | --- |
| `a & b` | AND | `5 & 3 = 1` |
| `a \| b` | OR | `5 \| 3 = 7` |
| `a ^ b` | XOR (same bits give 0) | `5 ^ 3 = 6` |
| `~a` | NOT | `~5 = -6` (two's complement) |
| `a << k` | multiply by 2^k | `1 << 3 = 8` |
| `a >> k` | divide by 2^k (floor) | `20 >> 2 = 5` |

**XOR properties:** `a ^ a = 0`, `a ^ 0 = a`, commutative and associative. So XOR of all numbers cancels every pair, leaving the unpaired one.

**Cheat sheet of tricks**

```python
def check_bit(x, i): return (x >> i) & 1
def set_bit(x, i):   return x | (1 << i)
def clear_bit(x, i): return x & ~(1 << i)
def toggle_bit(x, i):return x ^ (1 << i)
def is_pow2(x):      return x > 0 and (x & (x - 1)) == 0     # x & (x-1) clears the lowest set bit
def lowest_set(x):   return x & -x                            # isolates the lowest set bit
def count_bits(x):                                           # Brian Kernighan: O(number of set bits)
    c = 0
    while x: x &= x - 1; c += 1
    return c
```

**Classic problems**

```python
def single_number(a):                 # every element twice except one
    r = 0
    for x in a: r ^= x
    return r

def missing_number(a):                # 0..n with one missing
    r = len(a)
    for i, x in enumerate(a): r ^= i ^ x
    return r

def subsets_bitmask(a):               # enumerate all 2^n subsets
    n = len(a)
    return [[a[i] for i in range(n) if mask >> i & 1] for mask in range(1 << n)]

def swap(a, b):                       # XOR swap (works if a and b are different variables)
    a ^= b; b ^= a; a ^= b
    return a, b
```

Other favourites: Single Number II/III (count bits modulo 3 / split by lowest differing bit), Reverse Bits, Hamming Distance, Counting Bits (`dp[i] = dp[i >> 1] + (i & 1)`), Sum of Two Integers without `+` (carry = `(a & b) << 1`, sum = `a ^ b`), Gray code, Maximum XOR (binary trie).

**Loopholes**

- Operator precedence: in C/C++/Java `==` binds tighter than `&`, so write `(x & 1) == 0`.
- Python ints are unbounded: `~x` and negative shifts behave differently from 32-bit languages; mask with `& 0xFFFFFFFF` when simulating 32-bit.
- In C++/Java, `1 << 31` overflows an `int`; use `1LL << k` / `1L << k`.
- Right shift of negative numbers: arithmetic (`>>`) keeps the sign; Java's `>>>` is the logical shift.
- Signed overflow in C/C++ is undefined behaviour.

**Practice:** Single Number I/II/III, Number of 1 Bits, Power of Two, Counting Bits, Missing Number, Reverse Bits, Sum of Two Integers, Subsets (bitmask), Maximum XOR.

---

## 23. String algorithms: KMP and Z-function

**Problem:** find all occurrences of pattern `p` (length m) in text `t` (length n). Naive matching is O(n \* m) because on a mismatch it restarts from scratch. KMP and Z achieve **O(n + m)** by reusing what has already matched.

### KMP (Knuth-Morris-Pratt)

**Idea:** precompute the **LPS** (longest proper prefix which is also a suffix) array for the pattern. On mismatch, fall back using LPS instead of restarting.

```
p   = a b a b c
lps = 0 0 1 2 0        lps[3] = 2 because 'ab' is both a prefix and a suffix of 'abab'
```

```python
def build_lps(p):
    lps, k = [0] * len(p), 0
    for i in range(1, len(p)):
        while k and p[i] != p[k]: k = lps[k - 1]     # fall back
        if p[i] == p[k]: k += 1
        lps[i] = k
    return lps

def kmp_search(t, p):
    if not p: return []
    lps, k, res = build_lps(p), 0, []
    for i, ch in enumerate(t):
        while k and ch != p[k]: k = lps[k - 1]
        if ch == p[k]: k += 1
        if k == len(p):
            res.append(i - k + 1)                   # match starts here
            k = lps[k - 1]                          # continue to find overlapping matches
    return res
```

**Why O(n + m):** `k` increases at most once per character and every fallback decreases it, so total fallbacks cannot exceed total increases.

### Z-function

`z[i]` = length of the longest substring starting at `i` that matches a **prefix** of the string. Search by building `p + '#' + t` and reporting positions with `z[i] == len(p)`.

```python
def z_function(s):
    n = len(s); z = [0] * n; l = r = 0
    for i in range(1, n):
        if i < r: z[i] = min(r - i, z[i - l])        # reuse the previously matched window [l, r)
        while i + z[i] < n and s[z[i]] == s[i + z[i]]: z[i] += 1
        if i + z[i] > r: l, r = i, i + z[i]
    return z

def z_search(t, p):
    s = p + '#' + t                                   # '#' must not occur in p or t
    z = z_function(s)
    return [i - len(p) - 1 for i in range(len(s)) if z[i] == len(p)]
```

**Related tools:** Rabin-Karp (rolling hash, O(n + m) average, beware collisions: verify matches and use a large modulus or double hash), Manacher (longest palindromic substring in O(n)), suffix array/automaton (advanced).

**Where KMP/Z ideas appear:** Repeated Substring Pattern (`(s + s)[1:-1]` contains `s`, or `n % (n - lps[-1]) == 0`), Shortest Palindrome (LPS of `s + '#' + reverse(s)`), Find the Index of the First Occurrence, Longest Happy Prefix (`lps[-1]`).

**Loopholes**

- Use a separator that cannot appear in the strings (Z-search).
- Empty pattern; pattern longer than text.
- After a full match, continue with `lps[k - 1]`, not 0, to catch overlapping matches.
- Rolling hash: handle negative modulo and collisions; Python's built-in `str.find` is already optimized, but interviews want the algorithm.

**Practice:** Find the Index of the First Occurrence, Repeated Substring Pattern, Shortest Palindrome, Longest Happy Prefix, Sum of Scores of Built Strings (Z).

---

## 24. Kadane's algorithm (maximum subarray)

**Idea:** `best_ending_here = max(x, best_ending_here + x)`. A running sum that turns negative can only hurt, so restart at the current element.

```python
def max_subarray(a):
    cur = best = a[0]
    for x in a[1:]:
        cur = max(x, cur + x)        # extend or restart
        best = max(best, cur)
    return best

def max_subarray_with_indices(a):
    cur = best = a[0]; s = bs = be = 0
    for i in range(1, len(a)):
        if a[i] > cur + a[i]: cur, s = a[i], i
        else: cur += a[i]
        if cur > best: best, bs, be = cur, s, i
    return best, bs, be
```

```
a = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
cur:  -2  1  -2  4   3  5  6   1  5     best = 6 (subarray [4, -1, 2, 1])
```

**Variants**

- **Maximum circular subarray:** `max(kadane_max, total_sum - kadane_min)`; if all numbers are negative the answer is just `kadane_max`.
- **Maximum product subarray:** track both current max and min product because a negative can flip them.
- **Max sum rectangle in a 2D matrix:** fix two rows or columns, collapse into a 1D array, run Kadane (O(n^3)).
- **Minimum subarray sum:** negate the array.

**Complexity:** O(n) time, O(1) space.

**Loopholes**

- Initializing `best = 0` fails when all elements are negative (the answer is the largest single element). Start with `a[0]`.
- The empty subarray: some problems allow it (answer is at least 0), most do not.
- Prefix-sum alternative: `max(P[j] - min(P[i] for i < j))`.
- Kadane does not directly give a length-bounded or exactly-K result; that needs a deque or other techniques.

**Practice:** Maximum Subarray, Maximum Sum Circular Subarray, Maximum Product Subarray, Best Time to Buy and Sell Stock (Kadane on price differences), Max Sum of Rectangle No Larger Than K.

---

## 25. Pattern-recognition cheat sheet

Use this table first on an unseen problem.

| Clue in the problem | Try |
| --- | --- |
| Sorted array, find pair/triplet | Two pointers, binary search |
| Contiguous subarray/substring, longest/shortest, all positives | Sliding window |
| Contiguous subarray with negatives, sum equals K | Prefix sum + hash map |
| Many range updates | Difference array |
| Range sum queries | Prefix sum (static), Fenwick/segment tree (updates) |
| "Minimum X such that feasible", monotone answer | Binary search on answer |
| Linked list cycle, middle, kth from end | Fast/slow pointers, dummy node |
| Next greater/smaller, histogram | Monotonic stack |
| Window max/min | Monotonic deque |
| Top K, kth, streaming median, merge K | Heap |
| Overlapping ranges, scheduling | Sort + sweep, heap |
| Frequency, existence, complements | Hash map / set |
| Tree depth, paths, LCA, subtree info | DFS returning values |
| Level order, shortest steps, unweighted graph | BFS |
| Dependencies, ordering | Topological sort |
| Connectivity, grouping, cycles (undirected) | DSU / DFS |
| Weighted shortest path, non-negative | Dijkstra |
| Negative weights / at most K edges | Bellman-Ford |
| Connect all with min cost | Kruskal / Prim |
| All combinations/permutations/subsets, n up to 20 | Backtracking, bitmask |
| Count ways, min/max with overlapping subproblems | DP |
| Optimal choice provable locally, interval scheduling | Greedy |
| Prefix queries, dictionary, word grid | Trie |
| Single/missing number, powers of two, subsets of small sets | Bit manipulation |
| Pattern search in text, repeated/palindromic prefixes | KMP / Z / rolling hash |
| Maximum subarray sum | Kadane |

**Interview protocol for any DSA question (use every time)**

1. **Clarify:** input size, ranges, duplicates, negatives, sorted or not, empty input, return format, in-place or not.
2. **Examples:** walk through a small case plus an edge case.
3. **Brute force** first with its complexity (shows you understand the problem).
4. **Optimize:** name the pattern and the bottleneck you are removing.
5. **Pseudocode** and invariants; confirm with the interviewer.
6. **Code** cleanly with meaningful names.
7. **Dry-run** one sample plus edge cases (empty, single element, all equal, max size).
8. **State time and space complexity** and possible improvements.

**Online-assessment checklist:** read constraints and pick the complexity target; write the brute force to validate if time permits; watch integer overflow, 0/1-indexing, and fast I/O (`sys.stdin.readline`); test with the smallest and largest inputs; submit partial credit early; leave time at the end for edge cases.

**Spaced-revision plan for the 250+ problem target:** log every problem with pattern, trick, and mistake; re-solve failures after 1 day, 1 week and 1 month; every week do two mixed timed sets of 3 problems in 90 minutes.

---

*End of Part 1. Next in the series: Part 2 covers Operating Systems, DBMS and Computer Networks.*

---

# Appendix A: C++ Companion (C++17)

Same section numbers as above. Read the theory, loopholes and complexity in the main sections; use this appendix for the C++ code. Every snippet assumes this header:

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
using vi = vector<int>;
using pii = pair<int,int>;
const int INF = 1e9;
const ll LINF = 1e18;
const int MOD = 1e9 + 7;
```

## A.0 STL cheat sheet and C++ specific loopholes

| Need | Container / function | Cost |
| --- | --- | --- |
| Dynamic array | `vector<T>`: `push_back`, `pop_back`, `back()`, `size()` | O(1) amortized push |
| Hash map / set | `unordered_map`, `unordered_set` | O(1) average, O(n) worst |
| Ordered map / set | `map`, `set` (red-black tree), `lower_bound`, `upper_bound` | O(log n) |
| Duplicates allowed | `multiset`, `multimap` | O(log n) |
| Max-heap | `priority_queue<int>` | O(log n) push/pop |
| Min-heap | `priority_queue<int, vector<int>, greater<int>>` | O(log n) |
| Stack / queue | `stack<T>`, `queue<T>` | O(1) |
| Double-ended queue | `deque<T>`: `push_front`, `pop_front`, `push_back`, `pop_back` | O(1) |
| Sort | `sort(a.begin(), a.end())`, custom comparator lambda | O(n log n) |
| Binary search | `lower_bound(a.begin(), a.end(), x)` first element >= x; `upper_bound` first > x; `binary_search` | O(log n) |
| Misc | `accumulate`, `reverse`, `rotate`, `iota`, `fill`, `max_element`, `next_permutation`, `unique` |  |
| Bits | `__builtin_popcount(x)`, `__builtin_clz(x)`, `__builtin_ctz(x)` (`ll` versions end in `ll`) | O(1) |

**Loopholes specific to C++**

- `map::operator[]` **inserts** a default value for a missing key; use `count()` or `find()` to test existence without inserting.
- `size()` returns `size_t` (unsigned). `a.size() - 1` on an empty vector wraps to a huge number. Cast: `int n = a.size();`.
- Signed overflow is undefined behaviour. Use `ll` for sums and products; `INT_MAX + 1` is a bug.
- Mid calculation: `lo + (hi - lo) / 2`.
- Erasing from a vector or map while iterating invalidates iterators. Use `it = v.erase(it)`, or the erase-remove idiom: `v.erase(remove(v.begin(), v.end(), x), v.end())`.
- `vector<bool>` is bit-packed and does not return real references; prefer `vector<char>` when you need a reference.
- Comparators must be a **strict weak ordering**: use `<`, never `<=`, or `sort` can crash.
- `unordered_map` can be attacked with colliding keys on judges; use a custom hash (splitmix64) or `map` when hacks are possible.
- Passing a vector by value copies it; use `vector<int>&` or `const vector<int>&`.
- `priority_queue` of pairs compares `first`, then `second`. For a min-heap by distance store `{dist, node}` with `greater<>`.
- Use `'\n'` instead of `endl` and add `ios::sync_with_stdio(false); cin.tie(nullptr);` for fast I/O.
- Uninitialized local variables and arrays contain garbage; initialize them (`vector<int> a(n, 0)`).
- Stack depth for recursion is about 10^5 to 10^6 frames depending on the platform; prefer iterative approaches for 10^6 depth.
- Dangling references/pointers after a `vector` reallocation: do not keep `&v[0]` across `push_back`.

## A.1 Complexity helper

```cpp
// Typical contest/OA main skeleton
int main() {
    ios::sync_with_stdio(false); cin.tie(nullptr);
    int t; cin >> t;
    while (t--) { /* solve */ }
}
```

Fast power with modulus, used constantly (divide and conquer):

```cpp
ll power(ll x, ll n, ll mod = MOD) {
    ll r = 1; x %= mod;
    while (n > 0) {
        if (n & 1) r = r * x % mod;
        x = x * x % mod;
        n >>= 1;
    }
    return r;
}
```

## A.2 Arrays and strings

```cpp
void rotateRight(vector<int>& a, int k) {
    int n = a.size(); if (n == 0) return;
    k %= n;
    reverse(a.begin(), a.end());
    reverse(a.begin(), a.begin() + k);
    reverse(a.begin() + k, a.end());
}

void sortColors(vector<int>& a) {            // Dutch national flag
    int lo = 0, mid = 0, hi = (int)a.size() - 1;
    while (mid <= hi) {
        if (a[mid] == 0) swap(a[lo++], a[mid++]);
        else if (a[mid] == 1) mid++;
        else swap(a[mid], a[hi--]);          // do not advance mid
    }
}

bool isAnagram(const string& s, const string& t) {
    if (s.size() != t.size()) return false;
    int cnt[26] = {};
    for (char c : s) cnt[c - 'a']++;
    for (char c : t) if (--cnt[c - 'a'] < 0) return false;
    return true;
}

bool isPalindrome(const string& s) {
    int l = 0, r = (int)s.size() - 1;
    while (l < r) {
        while (l < r && !isalnum(s[l])) l++;
        while (l < r && !isalnum(s[r])) r--;
        if (tolower(s[l]) != tolower(s[r])) return false;
        l++; r--;
    }
    return true;
}
```

## A.3 Hashing

```cpp
vector<int> twoSum(vector<int>& a, int target) {
    unordered_map<int,int> seen;
    for (int i = 0; i < (int)a.size(); i++) {
        auto it = seen.find(target - a[i]);
        if (it != seen.end()) return {it->second, i};
        seen[a[i]] = i;                      // insert after the check
    }
    return {};
}

vector<vector<string>> groupAnagrams(vector<string>& words) {
    unordered_map<string, vector<string>> g;
    for (auto& w : words) { string k = w; sort(k.begin(), k.end()); g[k].push_back(w); }
    vector<vector<string>> res;
    for (auto& [k, v] : g) res.push_back(v);
    return res;
}

int longestConsecutive(vector<int>& a) {
    unordered_set<int> s(a.begin(), a.end());
    int best = 0;
    for (int x : s) {
        if (s.count(x - 1)) continue;        // only start at the beginning of a run
        int y = x;
        while (s.count(y + 1)) y++;
        best = max(best, y - x + 1);
    }
    return best;
}

int subarraySum(vector<int>& a, int k) {     // prefix sum + map
    unordered_map<ll,int> cnt; cnt[0] = 1;
    ll pre = 0; int ans = 0;
    for (int x : a) {
        pre += x;
        auto it = cnt.find(pre - k);
        if (it != cnt.end()) ans += it->second;
        cnt[pre]++;
    }
    return ans;
}
```

Custom hash to defeat anti-hash tests:

```cpp
struct CustomHash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t R = chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + R);
    }
};
// unordered_map<int,int,CustomHash> m;
```

## A.4 Two pointers

```cpp
vector<int> twoSumSorted(vector<int>& a, int t) {
    int l = 0, r = (int)a.size() - 1;
    while (l < r) {
        int s = a[l] + a[r];
        if (s == t) return {l, r};
        s < t ? l++ : r--;
    }
    return {};
}

vector<vector<int>> threeSum(vector<int> a) {
    sort(a.begin(), a.end());
    vector<vector<int>> res; int n = a.size();
    for (int i = 0; i < n - 2; i++) {
        if (i && a[i] == a[i-1]) continue;
        int l = i + 1, r = n - 1;
        while (l < r) {
            ll s = (ll)a[i] + a[l] + a[r];
            if (s < 0) l++;
            else if (s > 0) r--;
            else {
                res.push_back({a[i], a[l], a[r]});
                l++; r--;
                while (l < r && a[l] == a[l-1]) l++;
            }
        }
    }
    return res;
}

int maxArea(vector<int>& h) {
    int l = 0, r = (int)h.size() - 1, best = 0;
    while (l < r) {
        best = max(best, min(h[l], h[r]) * (r - l));
        h[l] < h[r] ? l++ : r--;
    }
    return best;
}

int removeDuplicates(vector<int>& a) {
    int w = 0;
    for (int r = 0; r < (int)a.size(); r++)
        if (w == 0 || a[r] != a[w-1]) a[w++] = a[r];
    return w;
}
```

## A.5 Sliding window

```cpp
ll maxSumK(vector<int>& a, int k) {
    ll cur = 0, best;
    for (int i = 0; i < k; i++) cur += a[i];
    best = cur;
    for (int i = k; i < (int)a.size(); i++) { cur += a[i] - a[i-k]; best = max(best, cur); }
    return best;
}

int longestUnique(const string& s) {
    vector<int> last(256, -1);
    int l = 0, best = 0;
    for (int r = 0; r < (int)s.size(); r++) {
        unsigned char c = s[r];
        if (last[c] >= l) l = last[c] + 1;
        last[c] = r;
        best = max(best, r - l + 1);
    }
    return best;
}

string minWindow(const string& s, const string& t) {
    vector<int> need(128, 0);
    for (char c : t) need[c]++;
    int missing = t.size(), l = 0, bestLen = INT_MAX, bestL = 0;
    for (int r = 0; r < (int)s.size(); r++) {
        if (need[s[r]]-- > 0) missing--;
        while (missing == 0) {
            if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestL = l; }
            if (++need[s[l]] > 0) missing++;
            l++;
        }
    }
    return bestLen == INT_MAX ? "" : s.substr(bestL, bestLen);
}

// exactly K = atMost(K) - atMost(K-1)
int atMostKDistinct(vector<int>& a, int k) {
    unordered_map<int,int> cnt; int l = 0, res = 0;
    for (int r = 0; r < (int)a.size(); r++) {
        if (cnt[a[r]]++ == 0) k--;
        while (k < 0) if (--cnt[a[l++]] == 0) k++;
        res += r - l + 1;
    }
    return res;
}
```

## A.6 Prefix sums and difference arrays

```cpp
vector<ll> buildPrefix(const vector<int>& a) {
    vector<ll> P(a.size() + 1, 0);
    for (int i = 0; i < (int)a.size(); i++) P[i+1] = P[i] + a[i];
    return P;                                // sum(l..r) = P[r+1] - P[l]
}

vector<int> productExceptSelf(vector<int>& a) {
    int n = a.size(); vector<int> res(n, 1);
    int left = 1;  for (int i = 0; i < n; i++)   { res[i] = left;  left *= a[i]; }
    int right = 1; for (int i = n - 1; i >= 0; i--) { res[i] *= right; right *= a[i]; }
    return res;
}

vector<int> applyUpdates(int n, vector<array<int,3>>& ups) {   // {l, r, val}
    vector<int> d(n + 1, 0);
    for (auto& [l, r, v] : ups) { d[l] += v; d[r + 1] -= v; }
    for (int i = 1; i < n; i++) d[i] += d[i-1];
    d.resize(n); return d;
}

int subarraysDivByK(vector<int>& a, int k) {
    vector<int> cnt(k, 0); cnt[0] = 1;
    int pre = 0, ans = 0;
    for (int x : a) {
        pre = ((pre + x) % k + k) % k;       // normalize negative remainders
        ans += cnt[pre]++;
    }
    return ans;
}
```

## A.7 Binary search and binary search on answer

```cpp
// First x in [lo, hi) where pred(x) is true; returns hi if none
template <class F>
ll firstTrue(ll lo, ll hi, F pred) {
    while (lo < hi) {
        ll mid = lo + (hi - lo) / 2;
        if (pred(mid)) hi = mid; else lo = mid + 1;
    }
    return lo;
}

// STL: lower_bound / upper_bound
int countOf(const vector<int>& a, int x) {
    return upper_bound(a.begin(), a.end(), x) - lower_bound(a.begin(), a.end(), x);
}

int searchRotated(vector<int>& a, int t) {
    int l = 0, r = (int)a.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (a[m] == t) return m;
        if (a[l] <= a[m]) {                          // left half sorted
            if (a[l] <= t && t < a[m]) r = m - 1; else l = m + 1;
        } else {
            if (a[m] < t && t <= a[r]) l = m + 1; else r = m - 1;
        }
    }
    return -1;
}

int minEatingSpeed(vector<int>& piles, int h) {
    auto ok = [&](ll k) { ll hrs = 0; for (int p : piles) hrs += (p + k - 1) / k; return hrs <= h; };
    return firstTrue(1, *max_element(piles.begin(), piles.end()) + 1, ok);
}

int shipWithinDays(vector<int>& w, int D) {
    auto ok = [&](ll cap) {
        int days = 1; ll cur = 0;
        for (int x : w) { if (cur + x > cap) { days++; cur = 0; } cur += x; }
        return days <= D;
    };
    return firstTrue(*max_element(w.begin(), w.end()), accumulate(w.begin(), w.end(), 0LL) + 1, ok);
}
```

Floating-point binary search: run a fixed 100 iterations, not `while (hi - lo > eps)`.

## A.8 Fast and slow pointers

```cpp
struct ListNode {
    int val; ListNode* next;
    ListNode(int v = 0, ListNode* n = nullptr) : val(v), next(n) {}
};

ListNode* cycleStart(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next; fast = fast->next->next;
        if (slow == fast) {
            ListNode* p = head;
            while (p != slow) { p = p->next; slow = slow->next; }
            return p;
        }
    }
    return nullptr;
}

ListNode* middleNode(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    return slow;
}

int findDuplicate(vector<int>& a) {                  // array as linked list
    int slow = a[0], fast = a[0];
    do { slow = a[slow]; fast = a[a[fast]]; } while (slow != fast);
    slow = a[0];
    while (slow != fast) { slow = a[slow]; fast = a[fast]; }
    return slow;
}
```

## A.9 Linked lists

```cpp
ListNode* reverseList(ListNode* head) {
    ListNode *prev = nullptr, *cur = head;
    while (cur) { ListNode* nxt = cur->next; cur->next = prev; prev = cur; cur = nxt; }
    return prev;
}

ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0, head);                      // stack-allocated dummy
    ListNode *fast = &dummy, *slow = &dummy;
    for (int i = 0; i < n; i++) fast = fast->next;
    while (fast->next) { fast = fast->next; slow = slow->next; }
    ListNode* del = slow->next;
    slow->next = del->next;
    delete del;                                   // C++: free the removed node
    return dummy.next;
}

ListNode* mergeTwoLists(ListNode* a, ListNode* b) {
    ListNode dummy; ListNode* tail = &dummy;
    while (a && b) {
        if (a->val <= b->val) { tail->next = a; a = a->next; }
        else { tail->next = b; b = b->next; }
        tail = tail->next;
    }
    tail->next = a ? a : b;
    return dummy.next;
}
```

## A.10 Stack and monotonic stack

```cpp
bool validParentheses(const string& s) {
    stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') st.push(c);
        else {
            if (st.empty()) return false;
            char o = st.top(); st.pop();
            if ((c == ')' && o != '(') || (c == ']' && o != '[') || (c == '}' && o != '{')) return false;
        }
    }
    return st.empty();
}

vector<int> nextGreater(const vector<int>& a) {
    int n = a.size(); vector<int> ans(n, -1); stack<int> st;      // indices, decreasing values
    for (int i = 0; i < n; i++) {
        while (!st.empty() && a[st.top()] < a[i]) { ans[st.top()] = a[i]; st.pop(); }
        st.push(i);
    }
    return ans;
}

int largestRectangle(vector<int> h) {
    h.push_back(0);                                // sentinel
    stack<int> st; int best = 0;
    for (int i = 0; i < (int)h.size(); i++) {
        while (!st.empty() && h[st.top()] >= h[i]) {
            int height = h[st.top()]; st.pop();
            int left = st.empty() ? -1 : st.top();
            best = max(best, height * (i - left - 1));
        }
        st.push(i);
    }
    return best;
}

class MinStack {
    stack<pair<int,int>> st;                        // {value, min so far}
public:
    void push(int x) { st.push({x, st.empty() ? x : min(x, st.top().second)}); }
    void pop() { st.pop(); }
    int top() { return st.top().first; }
    int getMin() { return st.top().second; }
};
```

## A.11 Queue and deque

```cpp
vector<int> maxSlidingWindow(vector<int>& a, int k) {
    deque<int> dq; vector<int> res;
    for (int i = 0; i < (int)a.size(); i++) {
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        dq.push_back(i);
        if (dq.front() <= i - k) dq.pop_front();
        if (i >= k - 1) res.push_back(a[dq.front()]);
    }
    return res;
}

class MyCircularQueue {
    vector<int> d; int head = 0, sz = 0, cap;
public:
    MyCircularQueue(int k) : d(k), cap(k) {}
    bool enQueue(int x) { if (sz == cap) return false; d[(head + sz++) % cap] = x; return true; }
    bool deQueue() { if (!sz) return false; head = (head + 1) % cap; sz--; return true; }
    int Front() { return sz ? d[head] : -1; }
    int Rear() { return sz ? d[(head + sz - 1) % cap] : -1; }
};
```

## A.12 Heap and Top-K

```cpp
vector<int> topKLargest(const vector<int>& a, int k) {
    priority_queue<int, vector<int>, greater<int>> h;     // min-heap of size k
    for (int x : a) { h.push(x); if ((int)h.size() > k) h.pop(); }
    vector<int> res;
    while (!h.empty()) { res.push_back(h.top()); h.pop(); }
    return res;                                          // ascending; res[0] is the kth largest
}

ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto* l : lists) if (l) pq.push(l);
    ListNode dummy; ListNode* tail = &dummy;
    while (!pq.empty()) {
        ListNode* n = pq.top(); pq.pop();
        tail->next = n; tail = n;
        if (n->next) pq.push(n->next);
    }
    return dummy.next;
}

class MedianFinder {
    priority_queue<int> lo;                                          // max-heap (lower half)
    priority_queue<int, vector<int>, greater<int>> hi;               // min-heap (upper half)
public:
    void addNum(int x) {
        lo.push(x); hi.push(lo.top()); lo.pop();
        if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }
    }
    double findMedian() {
        return lo.size() > hi.size() ? lo.top() : (lo.top() + (double)hi.top()) / 2.0;
    }
};

int kthLargest(vector<int> a, int k) {                    // quickselect via STL
    nth_element(a.begin(), a.begin() + (k - 1), a.end(), greater<int>());
    return a[k - 1];                                      // O(n) average
}
```

## A.13 Intervals

```cpp
vector<vector<int>> mergeIntervals(vector<vector<int>> iv) {
    if (iv.empty()) return {};
    sort(iv.begin(), iv.end());                       // sorts by start, then end
    vector<vector<int>> res = {iv[0]};
    for (int i = 1; i < (int)iv.size(); i++) {
        if (iv[i][0] <= res.back()[1]) res.back()[1] = max(res.back()[1], iv[i][1]);
        else res.push_back(iv[i]);
    }
    return res;
}

int minMeetingRooms(vector<vector<int>> iv) {
    sort(iv.begin(), iv.end());
    priority_queue<int, vector<int>, greater<int>> ends;          // earliest end time on top
    for (auto& v : iv) {
        if (!ends.empty() && ends.top() <= v[0]) ends.pop();      // reuse the room
        ends.push(v[1]);
    }
    return ends.size();
}

int eraseOverlapIntervals(vector<vector<int>> iv) {               // greedy: sort by END
    sort(iv.begin(), iv.end(), [](auto& a, auto& b) { return a[1] < b[1]; });
    int keep = 0, last = INT_MIN;
    for (auto& v : iv) if (v[0] >= last) { keep++; last = v[1]; }
    return iv.size() - keep;
}
```

## A.14 Sorting

```cpp
void mergeSort(vector<int>& a, int l, int r) {                    // sorts [l, r)
    if (r - l <= 1) return;
    int m = l + (r - l) / 2;
    mergeSort(a, l, m); mergeSort(a, m, r);
    vector<int> tmp; tmp.reserve(r - l);
    int i = l, j = m;
    while (i < m && j < r) tmp.push_back(a[i] <= a[j] ? a[i++] : a[j++]);   // <= keeps stability
    while (i < m) tmp.push_back(a[i++]);
    while (j < r) tmp.push_back(a[j++]);
    copy(tmp.begin(), tmp.end(), a.begin() + l);
}

mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());
void quickSort(vector<int>& a, int lo, int hi) {                  // sorts [lo, hi]
    if (lo >= hi) return;
    int p = lo + rng() % (hi - lo + 1);                           // random pivot
    swap(a[p], a[hi]);
    int pivot = a[hi], i = lo;
    for (int j = lo; j < hi; j++) if (a[j] < pivot) swap(a[i++], a[j]);
    swap(a[i], a[hi]);
    quickSort(a, lo, i - 1); quickSort(a, i + 1, hi);
}

// Count inversions using merge sort
ll countInv(vector<int>& a, int l, int r) {
    if (r - l <= 1) return 0;
    int m = l + (r - l) / 2;
    ll cnt = countInv(a, l, m) + countInv(a, m, r);
    vector<int> tmp; int i = l, j = m;
    while (i < m && j < r) {
        if (a[i] <= a[j]) tmp.push_back(a[i++]);
        else { tmp.push_back(a[j++]); cnt += m - i; }            // all remaining left items are greater
    }
    while (i < m) tmp.push_back(a[i++]);
    while (j < r) tmp.push_back(a[j++]);
    copy(tmp.begin(), tmp.end(), a.begin() + l);
    return cnt;
}

// Largest Number: custom comparator
string largestNumber(vector<int>& nums) {
    vector<string> s; for (int x : nums) s.push_back(to_string(x));
    sort(s.begin(), s.end(), [](const string& a, const string& b) { return a + b > b + a; });
    if (s[0] == "0") return "0";
    string res; for (auto& x : s) res += x;
    return res;
}

// Counting sort for values in [0, K]
void countingSort(vector<int>& a, int K) {
    vector<int> cnt(K + 1, 0);
    for (int x : a) cnt[x]++;
    int idx = 0;
    for (int v = 0; v <= K; v++) while (cnt[v]--) a[idx++] = v;
}
```

## A.15 Matrix

```cpp
const int dr[4] = {1, -1, 0, 0}, dc[4] = {0, 0, 1, -1};

void rotateImage(vector<vector<int>>& m) {
    int n = m.size();
    for (int i = 0; i < n; i++) for (int j = i + 1; j < n; j++) swap(m[i][j], m[j][i]);
    for (auto& row : m) reverse(row.begin(), row.end());
}

vector<int> spiralOrder(vector<vector<int>>& m) {
    vector<int> res; if (m.empty()) return res;
    int top = 0, bot = m.size() - 1, left = 0, right = m[0].size() - 1;
    while (top <= bot && left <= right) {
        for (int j = left; j <= right; j++) res.push_back(m[top][j]);
        top++;
        for (int i = top; i <= bot; i++) res.push_back(m[i][right]);
        right--;
        if (top <= bot) { for (int j = right; j >= left; j--) res.push_back(m[bot][j]); bot--; }
        if (left <= right) { for (int i = bot; i >= top; i--) res.push_back(m[i][left]); left++; }
    }
    return res;
}

int numIslands(vector<vector<char>>& g) {
    int R = g.size(), C = g[0].size(), cnt = 0;
    for (int i = 0; i < R; i++) for (int j = 0; j < C; j++) {
        if (g[i][j] != '1') continue;
        cnt++;
        queue<pii> q; q.push({i, j}); g[i][j] = '0';              // BFS avoids deep recursion
        while (!q.empty()) {
            auto [r, c] = q.front(); q.pop();
            for (int d = 0; d < 4; d++) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr >= 0 && nr < R && nc >= 0 && nc < C && g[nr][nc] == '1') {
                    g[nr][nc] = '0'; q.push({nr, nc});
                }
            }
        }
    }
    return cnt;
}
```

## A.16 Trees

```cpp
struct TreeNode {
    int val; TreeNode *left, *right;
    TreeNode(int v = 0) : val(v), left(nullptr), right(nullptr) {}
};

vector<int> inorderIterative(TreeNode* root) {
    vector<int> res; stack<TreeNode*> st; TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        res.push_back(cur->val);
        cur = cur->right;
    }
    return res;
}

int maxDepth(TreeNode* n) { return !n ? 0 : 1 + max(maxDepth(n->left), maxDepth(n->right)); }

int diameterOfBinaryTree(TreeNode* root) {
    int best = 0;
    function<int(TreeNode*)> depth = [&](TreeNode* n) -> int {
        if (!n) return 0;
        int l = depth(n->left), r = depth(n->right);
        best = max(best, l + r);
        return 1 + max(l, r);
    };
    depth(root); return best;
}

TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    TreeNode* L = lowestCommonAncestor(root->left, p, q);
    TreeNode* R = lowestCommonAncestor(root->right, p, q);
    return (L && R) ? root : (L ? L : R);
}

vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res; if (!root) return res;
    queue<TreeNode*> q; q.push(root);
    while (!q.empty()) {
        int sz = q.size(); vector<int> level;
        while (sz--) {
            TreeNode* n = q.front(); q.pop();
            level.push_back(n->val);
            if (n->left) q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(level);
    }
    return res;
}

bool isBST(TreeNode* n, ll lo = LLONG_MIN, ll hi = LLONG_MAX) {   // ll bounds: node values can be INT_MIN
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return isBST(n->left, lo, n->val) && isBST(n->right, n->val, hi);
}

TreeNode* insertBST(TreeNode* root, int v) {
    if (!root) return new TreeNode(v);
    if (v < root->val) root->left = insertBST(root->left, v);
    else root->right = insertBST(root->right, v);
    return root;
}

TreeNode* deleteBST(TreeNode* root, int key) {
    if (!root) return nullptr;
    if (key < root->val) root->left = deleteBST(root->left, key);
    else if (key > root->val) root->right = deleteBST(root->right, key);
    else {
        if (!root->left || !root->right) {                       // 0 or 1 child
            TreeNode* child = root->left ? root->left : root->right;
            delete root; return child;
        }
        TreeNode* s = root->right;                                // inorder successor
        while (s->left) s = s->left;
        root->val = s->val;
        root->right = deleteBST(root->right, s->val);
    }
    return root;
}

TreeNode* buildTree(vector<int>& pre, vector<int>& in) {         // preorder + inorder
    unordered_map<int,int> pos;
    for (int i = 0; i < (int)in.size(); i++) pos[in[i]] = i;
    int pi = 0;
    function<TreeNode*(int,int)> go = [&](int l, int r) -> TreeNode* {
        if (l > r) return nullptr;
        TreeNode* root = new TreeNode(pre[pi++]);
        int m = pos[root->val];
        root->left = go(l, m - 1);
        root->right = go(m + 1, r);
        return root;
    };
    return go(0, (int)in.size() - 1);
}
```

## A.17 Graphs

```cpp
// BFS: shortest hops in an unweighted graph
vector<int> bfs(const vector<vector<int>>& g, int src) {
    vector<int> dist(g.size(), -1); queue<int> q;
    dist[src] = 0; q.push(src);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : g[u]) if (dist[v] == -1) { dist[v] = dist[u] + 1; q.push(v); }   // mark on enqueue
    }
    return dist;
}

// DFS (recursive) and directed cycle detection with 3 colours
bool hasCycleDirected(const vector<vector<int>>& g) {
    int n = g.size(); vector<int> color(n, 0);
    function<bool(int)> dfs = [&](int u) {
        color[u] = 1;
        for (int v : g[u]) {
            if (color[v] == 1) return true;                       // back edge
            if (color[v] == 0 && dfs(v)) return true;
        }
        color[u] = 2; return false;
    };
    for (int i = 0; i < n; i++) if (color[i] == 0 && dfs(i)) return true;
    return false;
}

// Bipartite check
bool isBipartite(const vector<vector<int>>& g) {
    int n = g.size(); vector<int> col(n, -1);
    for (int s = 0; s < n; s++) {
        if (col[s] != -1) continue;
        queue<int> q; q.push(s); col[s] = 0;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v : g[u]) {
                if (col[v] == -1) { col[v] = col[u] ^ 1; q.push(v); }
                else if (col[v] == col[u]) return false;
            }
        }
    }
    return true;
}

// Topological sort (Kahn). Returns empty if there is a cycle.
vector<int> topoSort(int n, const vector<pii>& edges) {
    vector<vector<int>> g(n); vector<int> indeg(n, 0);
    for (auto [u, v] : edges) { g[u].push_back(v); indeg[v]++; }
    queue<int> q;
    for (int i = 0; i < n; i++) if (!indeg[i]) q.push(i);
    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop(); order.push_back(u);
        for (int v : g[u]) if (--indeg[v] == 0) q.push(v);
    }
    if ((int)order.size() != n) order.clear();
    return order;
}

// DSU with path compression and union by size
struct DSU {
    vector<int> p, sz; int comps;
    DSU(int n) : p(n), sz(n, 1), comps(n) { iota(p.begin(), p.end(), 0); }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (sz[a] < sz[b]) swap(a, b);
        p[b] = a; sz[a] += sz[b]; comps--;
        return true;
    }
};

// Dijkstra: g[u] = {(v, w)}
vector<ll> dijkstra(const vector<vector<pair<int,int>>>& g, int src) {
    int n = g.size(); vector<ll> dist(n, LINF);
    priority_queue<pair<ll,int>, vector<pair<ll,int>>, greater<>> pq;
    dist[src] = 0; pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;                                // stale entry
        for (auto [v, w] : g[u])
            if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
    }
    return dist;
}

// Kruskal MST: edges = {w, u, v}
ll kruskal(int n, vector<array<int,3>> edges) {
    sort(edges.begin(), edges.end());
    DSU d(n); ll total = 0; int used = 0;
    for (auto [w, u, v] : edges) if (d.unite(u, v)) { total += w; used++; }
    return used == n - 1 ? total : -1;
}

// Prim MST
ll prim(const vector<vector<pair<int,int>>>& g) {
    int n = g.size(); vector<char> seen(n, 0); ll total = 0;
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq; pq.push({0, 0});
    while (!pq.empty()) {
        auto [w, u] = pq.top(); pq.pop();
        if (seen[u]) continue;
        seen[u] = 1; total += w;
        for (auto [v, wt] : g[u]) if (!seen[v]) pq.push({wt, v});
    }
    return total;
}

// Bellman-Ford: edges = {u, v, w}. Returns false if a negative cycle is reachable.
bool bellmanFord(int n, const vector<array<int,3>>& edges, int src, vector<ll>& dist) {
    dist.assign(n, LINF); dist[src] = 0;
    for (int i = 0; i < n - 1; i++) {
        bool changed = false;
        for (auto [u, v, w] : edges)
            if (dist[u] != LINF && dist[u] + w < dist[v]) { dist[v] = dist[u] + w; changed = true; }
        if (!changed) break;
    }
    for (auto [u, v, w] : edges) if (dist[u] != LINF && dist[u] + w < dist[v]) return false;
    return true;
}

// Floyd-Warshall. d[i][j] = LINF/4 if no edge, d[i][i] = 0
void floydWarshall(vector<vector<ll>>& d) {
    int n = d.size();
    for (int k = 0; k < n; k++)                                   // k is the OUTER loop
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                d[i][j] = min(d[i][j], d[i][k] + d[k][j]);
}

// SCC: Kosaraju
vector<vector<int>> kosaraju(const vector<vector<int>>& g) {
    int n = g.size(); vector<char> seen(n, 0); vector<int> order;
    function<void(int)> dfs1 = [&](int u) {
        seen[u] = 1;
        for (int v : g[u]) if (!seen[v]) dfs1(v);
        order.push_back(u);
    };
    for (int i = 0; i < n; i++) if (!seen[i]) dfs1(i);
    vector<vector<int>> rg(n);
    for (int u = 0; u < n; u++) for (int v : g[u]) rg[v].push_back(u);
    fill(seen.begin(), seen.end(), 0);
    vector<vector<int>> comps;
    function<void(int, vector<int>&)> dfs2 = [&](int u, vector<int>& comp) {
        seen[u] = 1; comp.push_back(u);
        for (int v : rg[u]) if (!seen[v]) dfs2(v, comp);
    };
    for (int i = n - 1; i >= 0; i--) {
        int u = order[i];
        if (!seen[u]) { comps.emplace_back(); dfs2(u, comps.back()); }
    }
    return comps;
}

// SCC + bridges via Tarjan (disc/low). Bridges below:
void findBridges(const vector<vector<int>>& g, vector<pii>& bridges) {
    int n = g.size(), timer = 0;
    vector<int> disc(n, -1), low(n, 0);
    function<void(int,int)> dfs = [&](int u, int parent) {
        disc[u] = low[u] = timer++;
        for (int v : g[u]) {
            if (v == parent) continue;                            // multi-edges need edge ids instead
            if (disc[v] == -1) {
                dfs(v, u);
                low[u] = min(low[u], low[v]);
                if (low[v] > disc[u]) bridges.push_back({u, v});
            } else low[u] = min(low[u], disc[v]);
        }
    };
    for (int i = 0; i < n; i++) if (disc[i] == -1) dfs(i, -1);
}
```

In C++ use `INF/4` or `LINF/4` as the "no edge" value in Floyd-Warshall so that `inf + inf` does not overflow.

## A.18 Recursion, backtracking, divide and conquer

```cpp
vector<vector<int>> subsets(vector<int>& a) {
    vector<vector<int>> res; vector<int> path;
    function<void(int)> go = [&](int i) {
        res.push_back(path);
        for (int j = i; j < (int)a.size(); j++) {
            path.push_back(a[j]); go(j + 1); path.pop_back();
        }
    };
    go(0); return res;
}

vector<vector<int>> permutations(vector<int>& a) {
    vector<vector<int>> res; vector<int> path; vector<char> used(a.size(), 0);
    function<void()> go = [&]() {
        if (path.size() == a.size()) { res.push_back(path); return; }
        for (int i = 0; i < (int)a.size(); i++) {
            if (used[i]) continue;
            used[i] = 1; path.push_back(a[i]); go(); path.pop_back(); used[i] = 0;
        }
    };
    go(); return res;
}

// Subsets with duplicates: sort first, then skip equal siblings
vector<vector<int>> subsetsWithDup(vector<int> a) {
    sort(a.begin(), a.end());
    vector<vector<int>> res; vector<int> path;
    function<void(int)> go = [&](int i) {
        res.push_back(path);
        for (int j = i; j < (int)a.size(); j++) {
            if (j > i && a[j] == a[j-1]) continue;
            path.push_back(a[j]); go(j + 1); path.pop_back();
        }
    };
    go(0); return res;
}

vector<vector<int>> combinationSum(vector<int>& c, int target) {
    vector<vector<int>> res; vector<int> path;
    function<void(int,int)> go = [&](int i, int rem) {
        if (rem == 0) { res.push_back(path); return; }
        for (int j = i; j < (int)c.size(); j++) {
            if (c[j] > rem) continue;
            path.push_back(c[j]); go(j, rem - c[j]); path.pop_back();   // j: reuse allowed
        }
    };
    go(0, target); return res;
}

int totalNQueens(int n) {
    int cnt = 0; vector<char> col(n, 0), d1(2 * n, 0), d2(2 * n, 0);
    function<void(int)> go = [&](int r) {
        if (r == n) { cnt++; return; }
        for (int c = 0; c < n; c++) {
            if (col[c] || d1[r + c] || d2[r - c + n]) continue;
            col[c] = d1[r + c] = d2[r - c + n] = 1;
            go(r + 1);
            col[c] = d1[r + c] = d2[r - c + n] = 0;
        }
    };
    go(0); return cnt;
}

// Divide and conquer: fast exponent (see A.1 for the modular version)
double myPow(double x, long long n) {
    if (n < 0) return 1.0 / myPow(x, -n);                         // long long avoids INT_MIN overflow
    if (n == 0) return 1.0;
    double half = myPow(x, n / 2);
    return n % 2 ? half * half * x : half * half;
}
```

## A.19 Greedy

```cpp
bool canJump(vector<int>& a) {
    int reach = 0;
    for (int i = 0; i < (int)a.size(); i++) {
        if (i > reach) return false;
        reach = max(reach, i + a[i]);
    }
    return true;
}

int minJumps(vector<int>& a) {
    int jumps = 0, curEnd = 0, far = 0;
    for (int i = 0; i < (int)a.size() - 1; i++) {
        far = max(far, i + a[i]);
        if (i == curEnd) { jumps++; curEnd = far; }
    }
    return jumps;
}

int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < (int)gas.size(); i++) {
        int d = gas[i] - cost[i];
        total += d; tank += d;
        if (tank < 0) { start = i + 1; tank = 0; }
    }
    return total >= 0 ? start : -1;
}

int maxActivities(vector<pii> iv) {                               // {start, end}, sort by end
    sort(iv.begin(), iv.end(), [](auto& a, auto& b) { return a.second < b.second; });
    int cnt = 0, last = INT_MIN;
    for (auto [s, e] : iv) if (s >= last) { cnt++; last = e; }
    return cnt;
}

double fractionalKnapsack(vector<pair<int,int>> items, int W) {   // {value, weight}
    sort(items.begin(), items.end(), [](auto& a, auto& b) { return (double)a.first / a.second > (double)b.first / b.second; });
    double res = 0;
    for (auto [v, w] : items) {
        if (W >= w) { res += v; W -= w; }
        else { res += (double)v * W / w; break; }
    }
    return res;
}
```

## A.20 Dynamic programming

```cpp
// 1D
int climbStairs(int n) { int a = 1, b = 1; for (int i = 1; i < n; i++) { int c = a + b; a = b; b = c; } return b; }

int rob(vector<int>& a) {
    int take = 0, skip = 0;
    for (int x : a) { int nt = skip + x; skip = max(take, skip); take = nt; }
    return max(take, skip);
}

int coinChange(vector<int>& coins, int amt) {
    vector<int> dp(amt + 1, amt + 1); dp[0] = 0;
    for (int a = 1; a <= amt; a++)
        for (int c : coins) if (c <= a) dp[a] = min(dp[a], dp[a - c] + 1);
    return dp[amt] > amt ? -1 : dp[amt];
}

ll coinWays(vector<int>& coins, int amt) {                         // combinations: coins OUTER loop
    vector<ll> dp(amt + 1, 0); dp[0] = 1;
    for (int c : coins) for (int a = c; a <= amt; a++) dp[a] += dp[a - c];
    return dp[amt];
}

bool wordBreak(const string& s, vector<string>& words) {
    unordered_set<string> W(words.begin(), words.end());
    int n = s.size(); vector<char> dp(n + 1, 0); dp[0] = 1;
    for (int i = 1; i <= n; i++)
        for (int j = 0; j < i && !dp[i]; j++)
            if (dp[j] && W.count(s.substr(j, i - j))) dp[i] = 1;
    return dp[n];
}

// 2D
int uniquePaths(int m, int n) {
    vector<int> dp(n, 1);
    for (int i = 1; i < m; i++) for (int j = 1; j < n; j++) dp[j] += dp[j - 1];
    return dp[n - 1];
}

int editDistance(const string& a, const string& b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++) for (int j = 1; j <= n; j++)
        dp[i][j] = a[i-1] == b[j-1] ? dp[i-1][j-1]
                                    : 1 + min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]});
    return dp[m][n];
}

int lcs(const string& a, const string& b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; i++) for (int j = 1; j <= n; j++)
        dp[i][j] = a[i-1] == b[j-1] ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}

// LIS in O(n log n); lower_bound = strictly increasing, upper_bound = non-decreasing
int lis(const vector<int>& a) {
    vector<int> tails;
    for (int x : a) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x); else *it = x;
    }
    return tails.size();
}

// 0/1 knapsack: capacity loop goes DOWNWARD
int knapsack01(vector<int>& wt, vector<int>& val, int W) {
    vector<int> dp(W + 1, 0);
    for (int i = 0; i < (int)wt.size(); i++)
        for (int c = W; c >= wt[i]; c--) dp[c] = max(dp[c], dp[c - wt[i]] + val[i]);
    return dp[W];
}

// Subset sum / partition equal subset
bool canPartition(vector<int>& a) {
    int sum = accumulate(a.begin(), a.end(), 0);
    if (sum % 2) return false;
    vector<char> dp(sum / 2 + 1, 0); dp[0] = 1;
    for (int x : a) for (int s = sum / 2; s >= x; s--) dp[s] = dp[s] || dp[s - x];
    return dp[sum / 2];
}

// Interval DP: Burst Balloons
int maxCoins(vector<int>& nums) {
    vector<int> a = {1}; a.insert(a.end(), nums.begin(), nums.end()); a.push_back(1);
    int n = a.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len < n; len++)
        for (int i = 0; i + len < n; i++) {
            int j = i + len;
            for (int k = i + 1; k < j; k++)
                dp[i][j] = max(dp[i][j], dp[i][k] + a[i] * a[k] * a[j] + dp[k][j]);
        }
    return dp[0][n - 1];
}

// Bitmask DP: TSP, O(2^n * n^2)
int tsp(const vector<vector<int>>& dist) {
    int n = dist.size(), FULL = 1 << n;
    vector<vector<int>> dp(FULL, vector<int>(n, INF)); dp[1][0] = 0;
    for (int mask = 1; mask < FULL; mask++)
        for (int u = 0; u < n; u++) {
            if (dp[mask][u] == INF || !(mask >> u & 1)) continue;
            for (int v = 0; v < n; v++) {
                if (mask >> v & 1) continue;
                int nm = mask | (1 << v);
                dp[nm][v] = min(dp[nm][v], dp[mask][u] + dist[u][v]);
            }
        }
    int best = INF;
    for (int u = 0; u < n; u++) if (dp[FULL - 1][u] != INF) best = min(best, dp[FULL - 1][u] + dist[u][0]);
    return best;
}

// Top-down memoization pattern in C++
ll memo[105][105];                       // memset(memo, -1, sizeof memo) before use
ll solve(int i, int j) {
    if (i < 0 || j < 0) return 0;
    ll& res = memo[i][j];
    if (res != -1) return res;
    return res = /* transition */ 0;
}
```

Modular counting reminder: after every addition/multiplication do `x %= MOD`; subtraction needs `(a - b + MOD) % MOD`.

## A.21 Trie

```cpp
struct TrieNode {
    TrieNode* ch[26] = {};
    bool end = false;
};

class Trie {
    TrieNode* root = new TrieNode();
    TrieNode* walk(const string& s) {
        TrieNode* n = root;
        for (char c : s) { if (!n->ch[c - 'a']) return nullptr; n = n->ch[c - 'a']; }
        return n;
    }
public:
    void insert(const string& w) {
        TrieNode* n = root;
        for (char c : w) {
            if (!n->ch[c - 'a']) n->ch[c - 'a'] = new TrieNode();
            n = n->ch[c - 'a'];
        }
        n->end = true;
    }
    bool search(const string& w) { auto n = walk(w); return n && n->end; }
    bool startsWith(const string& p) { return walk(p) != nullptr; }
    // Remember: free nodes in a destructor in production code to avoid leaks.
};

// Binary trie for Maximum XOR of two numbers
struct BinTrie {
    vector<array<int,2>> t{{-1, -1}};
    void insert(int x) {
        int u = 0;
        for (int b = 30; b >= 0; b--) {
            int bit = x >> b & 1;
            if (t[u][bit] == -1) { t[u][bit] = t.size(); t.push_back({-1, -1}); }
            u = t[u][bit];
        }
    }
    int maxXor(int x) {
        int u = 0, res = 0;
        for (int b = 30; b >= 0; b--) {
            int bit = x >> b & 1;
            if (t[u][bit ^ 1] != -1) { res |= 1 << b; u = t[u][bit ^ 1]; }
            else u = t[u][bit];
        }
        return res;
    }
};
```

## A.22 Bit manipulation

```cpp
bool checkBit(int x, int i) { return (x >> i) & 1; }
int setBit(int x, int i) { return x | (1 << i); }
int clearBit(int x, int i) { return x & ~(1 << i); }
int toggleBit(int x, int i) { return x ^ (1 << i); }
bool isPow2(ll x) { return x > 0 && (x & (x - 1)) == 0; }
int lowestSetBit(int x) { return x & -x; }
int countBits(unsigned x) { int c = 0; while (x) { x &= x - 1; c++; } return c; }   // or __builtin_popcount(x)

int singleNumber(vector<int>& a) { int r = 0; for (int x : a) r ^= x; return r; }

int missingNumber(vector<int>& a) {
    int r = a.size();
    for (int i = 0; i < (int)a.size(); i++) r ^= i ^ a[i];
    return r;
}

vector<int> countBitsTo(int n) {                                  // dp[i] = dp[i >> 1] + (i & 1)
    vector<int> dp(n + 1, 0);
    for (int i = 1; i <= n; i++) dp[i] = dp[i >> 1] + (i & 1);
    return dp;
}

uint32_t reverseBits(uint32_t n) {
    uint32_t r = 0;
    for (int i = 0; i < 32; i++) { r = (r << 1) | (n & 1); n >>= 1; }
    return r;
}

int getSum(int a, int b) {                                        // sum without '+'
    while (b) {
        unsigned carry = (unsigned)(a & b) << 1;                  // unsigned avoids signed-overflow UB
        a ^= b; b = carry;
    }
    return a;
}

vector<vector<int>> subsetsBitmask(vector<int>& a) {
    int n = a.size(); vector<vector<int>> res;
    for (int mask = 0; mask < (1 << n); mask++) {
        vector<int> cur;
        for (int i = 0; i < n; i++) if (mask >> i & 1) cur.push_back(a[i]);
        res.push_back(cur);
    }
    return res;
}

// Iterate all submasks of a mask (used in bitmask DP, O(3^n) total)
void submasks(int mask) {
    for (int s = mask; s > 0; s = (s - 1) & mask) { /* use s */ }
}
```

Precedence reminder: `x & 1 == 0` parses as `x & (1 == 0)`; always write `(x & 1) == 0`.

## A.23 KMP and Z-function

```cpp
vector<int> buildLPS(const string& p) {
    vector<int> lps(p.size(), 0);
    for (int i = 1, k = 0; i < (int)p.size(); i++) {
        while (k && p[i] != p[k]) k = lps[k - 1];
        if (p[i] == p[k]) k++;
        lps[i] = k;
    }
    return lps;
}

vector<int> kmpSearch(const string& t, const string& p) {
    vector<int> res; if (p.empty()) return res;
    vector<int> lps = buildLPS(p);
    for (int i = 0, k = 0; i < (int)t.size(); i++) {
        while (k && t[i] != p[k]) k = lps[k - 1];
        if (t[i] == p[k]) k++;
        if (k == (int)p.size()) { res.push_back(i - k + 1); k = lps[k - 1]; }
    }
    return res;
}

vector<int> zFunction(const string& s) {
    int n = s.size(); vector<int> z(n, 0);
    for (int i = 1, l = 0, r = 0; i < n; i++) {
        if (i < r) z[i] = min(r - i, z[i - l]);
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;
        if (i + z[i] > r) { l = i; r = i + z[i]; }
    }
    return z;
}

vector<int> zSearch(const string& t, const string& p) {
    string s = p + '#' + t;                                        // separator must not appear in p or t
    vector<int> z = zFunction(s), res;
    for (int i = p.size() + 1; i < (int)s.size(); i++)
        if (z[i] == (int)p.size()) res.push_back(i - p.size() - 1);
    return res;
}

// Rolling hash (Rabin-Karp) with double modulus to reduce collisions
struct RollingHash {
    static const ll M1 = 1000000007, M2 = 998244353, B = 131;
    vector<ll> h1, h2, p1, p2;
    RollingHash(const string& s) : h1(s.size() + 1), h2(s.size() + 1), p1(s.size() + 1, 1), p2(s.size() + 1, 1) {
        for (int i = 0; i < (int)s.size(); i++) {
            h1[i+1] = (h1[i] * B + s[i]) % M1; h2[i+1] = (h2[i] * B + s[i]) % M2;
            p1[i+1] = p1[i] * B % M1;          p2[i+1] = p2[i] * B % M2;
        }
    }
    pair<ll,ll> get(int l, int r) {                                // hash of s[l, r)
        ll a = ((h1[r] - h1[l] * p1[r - l]) % M1 + M1) % M1;
        ll b = ((h2[r] - h2[l] * p2[r - l]) % M2 + M2) % M2;
        return {a, b};
    }
};
```

## A.24 Kadane

```cpp
ll maxSubarray(const vector<int>& a) {
    ll cur = a[0], best = a[0];
    for (int i = 1; i < (int)a.size(); i++) {
        cur = max((ll)a[i], cur + a[i]);
        best = max(best, cur);
    }
    return best;
}

tuple<ll,int,int> maxSubarrayIdx(const vector<int>& a) {
    ll cur = a[0], best = a[0]; int s = 0, bs = 0, be = 0;
    for (int i = 1; i < (int)a.size(); i++) {
        if (a[i] > cur + a[i]) { cur = a[i]; s = i; } else cur += a[i];
        if (cur > best) { best = cur; bs = s; be = i; }
    }
    return {best, bs, be};
}

ll maxCircularSubarray(const vector<int>& a) {
    ll total = 0, curMax = 0, curMin = 0, best = a[0], worst = a[0];
    for (int x : a) {
        total += x;
        curMax = max((ll)x, curMax + x); best = max(best, curMax);
        curMin = min((ll)x, curMin + x); worst = min(worst, curMin);
    }
    return best < 0 ? best : max(best, total - worst);            // all-negative array: answer is best
}

int maxProductSubarray(vector<int>& a) {
    int mx = a[0], mn = a[0], res = a[0];
    for (int i = 1; i < (int)a.size(); i++) {
        if (a[i] < 0) swap(mx, mn);                               // a negative flips max and min
        mx = max(a[i], mx * a[i]);
        mn = min(a[i], mn * a[i]);
        res = max(res, mx);
    }
    return res;
}
```

## A.25 C++ interview-time checklist

- Write `int n = a.size();` once; avoid `size_t` arithmetic traps.
- Prefer `vector` over raw arrays; pass containers by reference.
- Use `ll` for any sum, product or prefix that could exceed 2 \* 10^9; apply `MOD` after each operation.
- Use `auto& [x, y]` structured bindings for pairs and `array<int,3>` for small tuples.
- Use `emplace_back`/`push_back({a, b})`; use `reserve` when the final size is known.
- Check `.empty()` before `.top()`, `.front()`, `.back()`, `.pop()`; calling them on an empty container is undefined behaviour.
- Sort with a lambda comparator using strict `<`; remember `stable_sort` when equal elements must keep order.
- For recursion, use `function<...>` lambdas inside a function (small overhead) or a free function with references; convert to BFS/iterative when depth is above about 10^5.
- Compile and test mentally with: empty input, one element, all equal, maximum values (overflow), negative numbers.

---

*End of Appendix A. Part 2 (OS, DBMS, Networks) will include C++ examples wherever code is needed.*
