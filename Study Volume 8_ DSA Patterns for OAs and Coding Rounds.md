# Study Volume 8: DSA Patterns for OAs and Coding Rounds

Eighth volume of the in-depth study set. Volumes 1 to 7 covered the theory rounds. This one is for the **online assessment and the coding interview**, where the skill is recognising which pattern a problem is hiding and writing it correctly under a clock. The earlier **OA notes** are the quick-reference sheet. This volume is the study layer: for every pattern you get the **signal that tells you to use it, the invariant that makes it correct, the template, a traced example, the variants, and the traps**.

**How this page was checked (please read, it is the reason you can trust the code):** every function on this page was **written once, run against a brute-force or known-answer reference on thousands of random inputs, and then pasted into this document by a script**, so the code you read is exactly the code that was tested. The traces are also produced by running code, and each trace asserts that it ends with the same answer as the tested function. Totals are in the next section. Passing random tests is strong evidence, not a proof, and a few templates assume stated preconditions (for example distinct values), which are written beside each one.

**Verification totals from the run that produced this page:** 45 test groups, **80,415 random cases** (50,300 for arrays through intervals, 30,115 for trees through KMP), every one agreeing with a brute-force or known-answer reference.

## Contents of this volume

- Part 1: How to attack a problem, constraints to complexity, recognition table, Python specifics
- Part 2: Arrays and hashing
- Part 3: Two pointers
- Part 4: Sliding window
- Part 5: Prefix sums and difference arrays
- Part 6: Binary search, including binary search on the answer
- Part 7: Linked lists
- Part 8: Stack and monotonic stack
- Part 9: Heaps and top-K
- Part 10: Intervals
- Part 11: Trees
- Part 12: Graphs: BFS, DFS, topological sort, union-find, shortest paths, minimum spanning tree
- Part 13: Backtracking
- Part 14: Dynamic programming
- Part 15: Greedy
- Part 16: Trie, bit manipulation, KMP
- Part 17: Practice plan, the must-solve list, and the mistake log

## Part 1: How to attack a problem

### 1.1 The six-step routine (about 5 minutes before any code)

1. **Restate** the problem in your own words and check it against the examples. Note the exact input and output types.
2. **Read the constraints.** They tell you the target complexity (table below).
3. **Brute force first**, out loud. State its complexity and why it is too slow. In an OA, **submit it if the clock is short**, because partial credit counts and it doubles as your checker.
4. **Find the bottleneck** and match it to a pattern with the recognition table.
5. **Write the invariant** in one sentence ('everything left of `l` is already valid', 'the stack is decreasing'). If you cannot say it, you do not yet understand the code you are about to write.
6. **Test by hand** on a small example plus three edge cases (empty, single element, all equal or all negative) **before** submitting.

### 1.2 Constraints tell you the complexity

| n up to | Target | Typical techniques |
| --- | --- | --- |
| 10 | O(n!) | permutations, backtracking |
| 20 | O(2^n x n) | subsets, bitmask DP |
| 500 | O(n^3) | Floyd-Warshall, interval DP |
| 5,000 | O(n^2) | 2D DP, pairwise checks |
| 10^5 to 10^6 | O(n log n) or O(n) | sort, heap, binary search, two pointers, hashing, sliding window |
| 10^9 to 10^18 | O(log n) or O(1) | binary search on the answer, maths |

A Python program does roughly 10^7 simple operations per second, so **10^8 operations is too slow** and 10^6 to 10^7 is comfortable. If the limit is O(n^2) and n is 10^5, you are missing a pattern.

### 1.3 Signal to pattern

| You see... | Reach for... |
| --- | --- |
| Sorted array, or 'find a pair or triple' | Two pointers, binary search, hash map |
| Contiguous subarray or substring with a condition | Sliding window (non-negative or character problems), prefix sums with a hash map (negatives allowed) |
| 'Minimise the maximum' or 'maximise the minimum' | Binary search on the answer |
| K largest, K most frequent, merge K sorted, a running median | Heap |
| Next greater or smaller element, spans, histograms | Monotonic stack |
| Overlapping ranges, scheduling, rooms | Sort by start (or end) then sweep |
| Count the ways, minimum cost, can you reach, longest or shortest sequence | Dynamic programming |
| Generate all subsets, permutations, combinations, placements | Backtracking |
| Prerequisites, dependencies, build order | Topological sort |
| Are X and Y connected, merge groups dynamically | Union-find |
| Fewest steps in an unweighted graph or grid | BFS |
| Cheapest path with weights | Dijkstra |
| Prefix lookups, autocomplete, word search | Trie |
| Find the one element that is different | XOR |
| Choose locally best and never regret | Greedy (prove it, or test against brute force) |

### 1.4 Python habits that save time in an OA

- Read input fast: `import sys; input = sys.stdin.readline` (and `sys.stdin.read().split()` for big inputs).
- Recursion limit is about 1,000 by default. For depth up to 10^5 use `sys.setrecursionlimit(10**6)`, or convert to an iterative stack (the traversal templates below are iterative on purpose).
- `heapq` is a **min-heap**. Negate values for a max-heap, and push tuples like `(priority, tiebreaker, item)` so equal priorities never compare unorderable objects.
- Never `list.pop(0)` in a loop (O(n)). Use `collections.deque`.
- Build strings with `''.join(parts)`. Use `Counter`, `defaultdict`, `bisect`, `itertools` and `functools.lru_cache`.
- Integers never overflow in Python. In C++ or Java use `long long` or `long` for sums and products.
- Copy before you mutate when a template returns a new structure. Several functions here take a copy (`nums[:]`) on purpose.
- **A mutable default argument or a shared list in `[[0] * n] * m`** (all rows are the same list) are classic silent bugs. Use `[[0] * n for _ in range(m)]`.

### 1.5 The edge-case checklist (run it before every submit)

- Empty input, one element, two elements.
- All elements equal, already sorted, reverse sorted.
- Zeros and negatives, especially in sum and product problems.
- Duplicates, and whether the answer must be unique.
- The maximum n with the worst structure (a chain-shaped tree, a star graph, all identical characters).
- Off-by-one at window and binary-search boundaries, and inclusive versus exclusive ends.
- Integer division and modulo with negatives (Python floors, C++ truncates).

## Part 2: Arrays and hashing

**Recognise it:** 'have I seen this before', counting, grouping, pairing, or an O(n^2) solution whose inner loop is really a **lookup**. **The idea:** trade memory for time, remember what you have seen in a dict or set so each lookup is O(1).

### Two sum, and the 'complement' idea

Store each value's index. For every new element ask whether `target - x` was already seen. One pass, O(n) time and space. (If the array were sorted you could use two pointers with O(1) space.)

```python
def two_sum(nums, target):
    seen = {}                                   # value -> index of earlier element
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
    return []
```

### Group anagrams: a canonical form as the key

Words that are anagrams have the same sorted letters, so that tuple is a perfect dictionary key. For long words a count tuple (26 numbers) avoids the O(k log k) sort.

```python
def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups[tuple(sorted(w))].append(w)     # canonical form: the sorted letters
    return list(groups.values())
```

### Longest consecutive sequence: only start at the beginning of a run

Put everything in a set, and only begin counting from `x` when `x - 1` is **not** in the set. Each element is then visited a constant number of times, so the nested loop is really O(n) total. Sorting would be O(n log n).

```python
def longest_consecutive(nums):
    s, best = set(nums), 0
    for x in s:
        if x - 1 not in s:                      # only start counting at the beginning of a run
            y = x
            while y + 1 in s: y += 1
            best = max(best, y - x + 1)
    return best
```

### Subarray sum equals K: prefix sums plus a hash map

**Invariant:** `count[p]` is the number of earlier prefixes with sum `p`. A subarray ending at the current position sums to K exactly when some earlier prefix equals `current - K`. This works **with negative numbers**, which is where a sliding window fails.

```python
def subarray_sum_k(nums, k):
    count, s, ans = {0: 1}, 0, 0                # count[p] = how many prefix sums equal p so far
    for x in nums:
        s += x
        ans += count.get(s - k, 0)              # a prior prefix of (s - k) means a subarray summing to k ends here
        count[s] = count.get(s, 0) + 1
    return ans
```

Trace on `nums = [1, 2, 3, -3, 3]`, `k = 3` (the answer is 5):

```text
nums = [1, 2, 3, -3, 3], k = 3
x  | prefix sum | look up s-k | matches | answer so far | counts of prefix sums   
---+------------+-------------+---------+---------------+-------------------------
1  | 1          | -2          | 0       | 0             | {0: 1, 1: 1}            
2  | 3          | 0           | 1       | 1             | {0: 1, 1: 1, 3: 1}      
3  | 6          | 3           | 1       | 2             | {0: 1, 1: 1, 3: 1, 6: 1}
-3 | 3          | 0           | 1       | 3             | {0: 1, 1: 1, 3: 2, 6: 1}
3  | 6          | 3           | 2       | 5             | {0: 1, 1: 1, 3: 2, 6: 2}
```

### Product of array except self: prefix times suffix, no division

Fill each slot with the product of everything to its left, then sweep from the right multiplying by the product of everything to its right. O(n) time and O(1) extra space (besides the output), and it works when the array contains zeros, where division would fail.

```python
def product_except_self(nums):
    n = len(nums); out = [1] * n
    left = 1
    for i in range(n):                          # out[i] = product of everything to the left
        out[i] = left; left *= nums[i]
    right = 1
    for i in range(n - 1, -1, -1):              # multiply by the product of everything to the right
        out[i] *= right; right *= nums[i]
    return out
```

### Rotate an array, in place

Reverse the whole array, then reverse the first `k` and the last `n - k` parts. Remember `k %= n`.

```python
def rotate_right(a, k):
    n = len(a); k %= n if n else 1
    a.reverse(); a[:k] = reversed(a[:k]); a[k:] = reversed(a[k:])
    return a
```

### First missing positive: the array is its own hash table (cyclic sort)

The answer lies in `1..n+1`. Put each value `v` in range at index `v - 1` by swapping, then the first index holding the wrong value is the answer. O(n) time and O(1) space (here a copy is made for safety).

```python
def first_missing_positive(nums):
    a, n = nums[:], len(nums)
    for i in range(n):                          # cyclic sort: put value v at index v-1
        while 1 <= a[i] <= n and a[a[i] - 1] != a[i]:
            j = a[i] - 1
            a[i], a[j] = a[j], a[i]
    for i in range(n):
        if a[i] != i + 1: return i + 1
    return n + 1
```

**Classics to solve:** two sum, group anagrams, contains duplicate, valid anagram, top K frequent elements, longest consecutive sequence, subarray sum equals K, product of array except self, 4Sum II, longest substring problems (next part).

**Traps:** using a list as a dict key (unhashable, use a tuple), forgetting that a dict preserves insertion order only for display not for logic, and assuming the elements are non-negative when they are not.

## Part 3: Two pointers

**Recognise it:** the input is **sorted** (or you can sort it), you need a **pair or triple** with a property, an in-place rearrangement, or a palindrome-like check from both ends. **The idea:** two indices move toward each other (or in the same direction) and each step **provably discards candidates** that cannot be the answer, turning O(n^2) into O(n).

### 3Sum: fix one, then two pointers on the rest

Sort. For each `i`, search for a pair summing to `-a[i]` with `l` and `r` moving inward. **Skip duplicates** at the outer and inner level so the same triple is not reported twice. O(n^2).

```python
def three_sum(nums):
    a, res = sorted(nums), []
    for i in range(len(a) - 2):
        if i and a[i] == a[i - 1]: continue
        l, r = i + 1, len(a) - 1
        while l < r:
            s = a[i] + a[l] + a[r]
            if s < 0: l += 1
            elif s > 0: r -= 1
            else:
                res.append([a[i], a[l], a[r]]); l += 1
                while l < r and a[l] == a[l - 1]: l += 1
    return res
```

### Container with most water: always move the shorter wall

**Why it is correct:** the area is limited by the shorter wall. Moving the taller wall inward can only shrink the width and cannot raise the limit, so the shorter wall can never be part of a better pair and can be discarded.

```python
def max_area(h):
    l, r, best = 0, len(h) - 1, 0
    while l < r:
        best = max(best, min(h[l], h[r]) * (r - l))
        if h[l] < h[r]: l += 1                  # the shorter wall limits the area, so move it
        else: r -= 1
    return best
```

### Trapping rain water: process the side with the smaller wall

The water above a bar is `min(maxLeft, maxRight) - height`. If `h[l] < h[r]` then the right side already guarantees a wall at least as tall as `h[l]`, so the water at `l` is decided by `lmax` alone, and `l` can advance. Symmetrically for `r`. O(n) time, O(1) space.

```python
def trap(h):
    l, r, lmax, rmax, water = 0, len(h) - 1, 0, 0, 0
    while l < r:
        if h[l] < h[r]:
            lmax = max(lmax, h[l]); water += lmax - h[l]; l += 1
        else:
            rmax = max(rmax, h[r]); water += rmax - h[r]; r -= 1
    return water
```

```text
heights: [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
side | index | height | max so far | water added | total
-----+-------+--------+------------+-------------+------
L    | 0     | 0      | 0          | 0           | 0    
R    | 11    | 1      | 1          | 0           | 0    
L    | 1     | 1      | 1          | 0           | 0    
L    | 2     | 0      | 1          | 1           | 1    
R    | 10    | 2      | 2          | 0           | 1    
R    | 9     | 1      | 2          | 1           | 2    
R    | 8     | 2      | 2          | 0           | 2    
L    | 3     | 2      | 2          | 0           | 2    
L    | 4     | 1      | 2          | 1           | 3    
L    | 5     | 0      | 2          | 2           | 5    
L    | 6     | 1      | 2          | 1           | 6    
total water: 6
```

### Sort colours (Dutch national flag): three regions, one pass

Keep `lo` (next slot for a 0), `hi` (next slot for a 2) and `i` (the scanner). **Invariant:** everything before `lo` is 0, everything between `lo` and `i` is 1, everything after `hi` is 2. When you swap a value from `hi`, **do not advance `i`**, because the swapped-in value is unexamined.

```python
def sort_colors(a):                             # Dutch national flag: one pass, three regions
    lo, i, hi = 0, 0, len(a) - 1
    while i <= hi:
        if a[i] == 0: a[lo], a[i] = a[i], a[lo]; lo += 1; i += 1
        elif a[i] == 2: a[hi], a[i] = a[i], a[hi]; hi -= 1
        else: i += 1
    return a
```

**Classics:** two sum II (sorted), 3Sum, 3Sum closest, container with most water, trapping rain water, remove duplicates from sorted array, move zeroes, valid palindrome, sort colours, merge sorted arrays.

**Traps:** forgetting to skip duplicates, moving both pointers when only one should move, and using two pointers on an **unsorted** array without sorting first.

## Part 4: Sliding window

**Recognise it:** a **contiguous** subarray or substring with a condition (no repeats, at most K distinct, sum at least S, all characters of T covered), and values that are **non-negative** (or the condition is monotonic as the window grows). **The idea:** expand the right end every step, **shrink the left end while the window is invalid**, and update the best answer once it is valid. Each index enters and leaves the window at most once, so O(n).

**When it fails:** if the array can contain **negative numbers** and the condition is a sum, adding an element can make the sum smaller, so shrinking is no longer monotonic. Use prefix sums with a hash map instead (Part 2).

### Longest substring without repeating characters

Remember where each character was last seen. On a repeat inside the window, jump `l` directly past the previous occurrence.

```python
def longest_unique_substring(s):
    last, l, best = {}, 0, 0
    for r, ch in enumerate(s):
        if ch in last and last[ch] >= l: l = last[ch] + 1
        last[ch] = r
        best = max(best, r - l + 1)
    return best
```

```text
input: abcabcbb
r | char | l | window | best | event                 
--+------+---+--------+------+-----------------------
0 | a    | 0 | a      | 1    |                       
1 | b    | 0 | ab     | 2    |                       
2 | c    | 0 | abc    | 3    |                       
3 | a    | 1 | bca    | 3    | a repeats, move l to 1
4 | b    | 2 | cab    | 3    | b repeats, move l to 2
5 | c    | 3 | abc    | 3    | c repeats, move l to 3
6 | b    | 5 | cb     | 3    | b repeats, move l to 5
7 | b    | 7 | b      | 3    | b repeats, move l to 7
```

### Minimum window substring

Track how many required characters are still `missing`. Expand until the window covers everything, then shrink from the left while it still covers, recording the smallest. The `need` counter goes negative for surplus characters, which tells you which left characters can be dropped safely.

```python
def min_window(s, t):
    need, missing = Counter(t), len(t)
    l, best = 0, (float('inf'), 0, 0)
    for r, ch in enumerate(s):
        if need[ch] > 0: missing -= 1
        need[ch] -= 1
        if missing == 0:
            while need[s[l]] < 0: need[s[l]] += 1; l += 1        # shrink while the window is still valid
            if r - l + 1 < best[0]: best = (r - l + 1, l, r + 1)
            need[s[l]] += 1; missing += 1; l += 1                # drop the leftmost to look for the next window
    return s[best[1]:best[2]] if best[0] != float('inf') else ''
```

### Longest repeating character replacement

A window is valid when `length - (count of its most frequent character) <= k`. You only need `maxf` to **never decrease** (a stale larger `maxf` cannot produce a wrong larger answer because the window only grows when a new true maximum appears), which is why a lazy `maxf` is enough.

```python
def character_replacement(s, k):
    count, l, maxf, best = Counter(), 0, 0, 0
    for r, ch in enumerate(s):
        count[ch] += 1; maxf = max(maxf, count[ch])
        while (r - l + 1) - maxf > k:           # more than k characters would need replacing
            count[s[l]] -= 1; l += 1
        best = max(best, r - l + 1)
    return best
```

### Sliding window maximum: a monotonic deque

Keep a deque of indices whose values are **decreasing**. Pop from the back anything smaller than the new element (it can never be a maximum while the new one is in the window), and pop from the front when the index leaves the window. The front is always the window's maximum.

```python
def max_sliding_window(a, k):
    dq, out = deque(), []                       # indices, values decreasing from front to back
    for i, x in enumerate(a):
        while dq and a[dq[-1]] <= x: dq.pop()
        dq.append(i)
        if dq[0] <= i - k: dq.popleft()
        if i >= k - 1: out.append(a[dq[0]])
    return out
```

**Classics:** longest substring without repeating characters, minimum window substring, longest repeating character replacement, permutation in string, fruit into baskets, max consecutive ones III, sliding window maximum, subarrays with at most K distinct values (and 'exactly K' as at most K minus at most K-1).

**Traps:** an off-by-one in `r - l + 1`, forgetting to remove the outgoing element's contribution, and using a window on negative numbers.

## Part 5: Prefix sums and difference arrays

**Recognise it:** many **range-sum queries** on a static array, many **range updates** applied before one read, or 'subarray with sum divisible by K' style conditions. **The idea:** precompute cumulative sums so any range sum is two lookups, `sum(a[l..r]) = pre[r+1] - pre[l]`. The same idea in 2D uses inclusion-exclusion, and the **difference array** is its mirror image for range **updates**.

### 2D prefix sums: any rectangle in O(1)

`P[r][c]` holds the sum of the rectangle from the top-left to `(r-1, c-1)`. A query adds and subtracts four corners so the overlapping area is counted exactly once. Build is O(R x C), each query O(1).

```python
class NumMatrix:
    def __init__(self, M):
        R, C = len(M), len(M[0])
        self.P = [[0] * (C + 1) for _ in range(R + 1)]
        for r in range(R):
            for c in range(C):
                self.P[r + 1][c + 1] = M[r][c] + self.P[r][c + 1] + self.P[r + 1][c] - self.P[r][c]
    def region(self, r1, c1, r2, c2):
        P = self.P
        return P[r2 + 1][c2 + 1] - P[r1][c2 + 1] - P[r2 + 1][c1] + P[r1][c1]
```

### Difference array: many range additions, one pass to read

To add `v` to `a[l..r]`, record `d[l] += v` and `d[r+1] -= v`. A running sum over `d` then reconstructs every position. K updates cost O(K) plus one O(n) pass, instead of O(K x n).

```python
def apply_range_adds(n, updates):               # updates are (l, r, v): add v to a[l..r]
    d = [0] * (n + 1)
    for l, r, v in updates:
        d[l] += v; d[r + 1] -= v
    out, run = [], 0
    for i in range(n):
        run += d[i]; out.append(run)
    return out
```

**Other prefix tricks:** prefix and suffix products (Part 2), prefix sums modulo K with a hash map for 'sum divisible by K', prefix XOR, prefix counts of a character for substring queries, and 'balance' prefix sums (+1 for one thing, -1 for another) for 'equal number of X and Y'.

**Traps:** an off-by-one between `pre[r+1] - pre[l]` and `pre[r] - pre[l]`, forgetting the extra slot `d[n]` in a difference array, and integer overflow in C++ or Java (use 64-bit sums).

## Part 6: Binary search

**Recognise it:** a **sorted** or **monotonic** structure, a question that asks for the **first** or **last** position satisfying something, or 'minimise the maximum' and 'maximise the minimum'. **The idea:** keep a range `[lo, hi)` that is **guaranteed to contain the answer** and halve it each step. O(log n).

**One template, used every time.** This version finds the **first index where the condition becomes true**, using a half-open range, and never loops forever because `lo = mid + 1` always makes progress.

```python
def lower_bound(a, x):                          # first index with a[i] >= x
    lo, hi = 0, len(a)
    while lo < hi:
        mid = (lo + hi) // 2
        if a[mid] < x: lo = mid + 1
        else: hi = mid
    return lo
```

```text
a = [1, 3, 5, 7, 9, 11], x = 7
lo | hi | mid | a[mid] | decision              
---+----+-----+--------+-----------------------
0  | 6  | 3   | 7      | a[mid] >= x: hi = mid 
0  | 3  | 1   | 3      | a[mid] < x: lo = mid+1
2  | 3  | 2   | 5      | a[mid] < x: lo = mid+1
answer: index 3
```

**Why `hi = mid` and not `mid - 1`:** `mid` itself might be the answer, so it must stay inside the range. **Why `(lo + hi) // 2` is safe in Python:** integers do not overflow. In C++ or Java write `lo + (hi - lo) / 2`.

### Search in a rotated sorted array (distinct values)

At every step **one of the two halves is sorted**. Decide which half is sorted, check whether the target lies inside it, and discard the other half.

```python
def search_rotated(a, target):                  # distinct values
    lo, hi = 0, len(a) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if a[mid] == target: return mid
        if a[lo] <= a[mid]:                     # the left half is sorted
            if a[lo] <= target < a[mid]: hi = mid - 1
            else: lo = mid + 1
        else:                                   # the right half is sorted
            if a[mid] < target <= a[hi]: lo = mid + 1
            else: hi = mid - 1
    return -1
```

```python
def find_min_rotated(a):
    lo, hi = 0, len(a) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if a[mid] > a[hi]: lo = mid + 1         # the minimum is to the right of mid
        else: hi = mid
    return a[lo]
```

### Find a peak element

Compare `a[mid]` with its right neighbour and **climb toward the larger side**. A peak must exist in that direction, because the sequence cannot rise forever (the boundary is treated as negative infinity).

```python
def find_peak(a):
    lo, hi = 0, len(a) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if a[mid] < a[mid + 1]: lo = mid + 1    # climb toward the larger neighbour
        else: hi = mid
    return lo
```

### Binary search on the answer (the pattern that wins OAs)

**Recognise it:** 'what is the **smallest** speed, capacity, time, or limit such that something becomes possible?' If feasibility is **monotonic** (if X works then any larger X also works), you can binary search the **answer itself** instead of searching an array. The recipe:

1. Define the search range `[lo, hi]` (the answer is guaranteed to lie inside it).
2. Write `ok(x)`, a **greedy O(n) check** that decides whether `x` is feasible.
3. Run the template: if `ok(mid)` then the answer is `mid` or smaller (`hi = mid`), else larger (`lo = mid + 1`).

Total cost O(n log range).

```python
def min_feasible(lo, hi, ok):                   # smallest x in [lo, hi] with ok(x) True, ok is monotonic
    while lo < hi:
        mid = (lo + hi) // 2
        if ok(mid): hi = mid
        else: lo = mid + 1
    return lo
```

**Koko eating bananas.** Find the minimum eating speed so that all piles are finished within `h` hours. Hours needed at speed `k` is `sum(ceil(pile / k))`, which only decreases as `k` grows, so it is monotonic.

```python
def koko_min_speed(piles, hours):
    ok = lambda k: sum(-(-p // k) for p in piles) <= hours      # ceil(p / k) hours per pile
    return min_feasible(1, max(piles), ok)
```

```text
piles [3, 6, 7, 11], must finish within 8 hours
lo | hi | speed k | hours needed | feasible?
---+----+---------+--------------+----------
1  | 11 | 6       | 6            | ok       
1  | 6  | 3       | 10           | too slow 
4  | 6  | 5       | 8            | ok       
4  | 5  | 4       | 8            | ok       
minimum speed: 4
```

**Split array largest sum.** Split into at most `m` contiguous parts so that the **largest part sum is as small as possible**. The cap must be at least `max(nums)` and at most `sum(nums)`. The check is greedy: fill each part until adding one more item would exceed the cap.

```python
def split_array_largest_sum(nums, m):
    def ok(cap):                                # can we split into at most m parts each with sum <= cap ?
        parts, cur = 1, 0
        for x in nums:
            if cur + x > cap: parts += 1; cur = x
            else: cur += x
        return parts <= m
    return min_feasible(max(nums), sum(nums), ok)
```

**Classics:** binary search, search insert position, first and last position of a target, search in rotated array (with and without duplicates), find minimum in rotated array, find peak, search a 2D matrix, square root, Koko eating bananas, ship packages within D days, split array largest sum, capacity to ship, aggressive cows, minimum days to make bouquets, smallest divisor given a threshold.

**Known-answer checks for several templates in this part and the earlier ones (all produced by running the code):**

```text
three_sum([-1,0,1,2,-1,-4]) -> [[-1, -1, 2], [-1, 0, 1]]
max_area([1,8,6,2,5,4,8,3,7]) -> 49
min_window("ADOBECODEBANC","ABC") -> BANC
max_sliding_window([1,3,-1,-3,5,3,6,7],3) -> [3, 3, 5, 5, 6, 7]
product_except_self([1,2,3,4]) -> [24, 12, 8, 6]
first_missing_positive([3,4,-1,1]) -> 2
search_rotated([4,5,6,7,0,1,2], 0) -> 4
split_array_largest_sum([7,2,5,10,8], 2) -> 18
longest_consecutive([100,4,200,1,3,2]) -> 4
daily_temperatures([73,74,75,71,69,72,76,73]) -> [1, 1, 4, 2, 1, 1, 0, 0]
```

**Traps:** an infinite loop from `lo = mid` with floor division, using `<=` in the loop with the half-open template (mixing the two conventions is the top source of off-by-one bugs, so pick one and never mix), a non-monotonic `ok` function (the search then silently returns garbage), and a range that does not contain the answer.

## Part 7: Linked lists

**Recognise it:** pointer manipulation, in-place reversal, cycles, merging, 'k-th from the end'. **The idea:** the bugs are almost all about **losing a reference**. Two habits prevent most of them: **save `next` before you overwrite it**, and use a **dummy head** whenever the head might change.

The node class and two helpers used to build and read lists in the tests:

```python
class ListNode:
    def __init__(self, val=0, nxt=None): self.val, self.next = val, nxt
```

```python
def make_list(vals):
    head = ListNode(); cur = head
    for v in vals: cur.next = ListNode(v); cur = cur.next
    return head.next
```

```python
def to_list(head):
    out = []
    while head: out.append(head.val); head = head.next
    return out
```

### Reverse a list

Walk once, pointing each node backwards. The order of the three assignments matters.

```python
def reverse_list(head):
    prev = None
    while head:
        nxt = head.next                         # save, because we are about to overwrite head.next
        head.next = prev
        prev, head = head, nxt
    return prev
```

### Find the cycle entrance: Floyd's tortoise and hare

`slow` moves one step and `fast` two. If there is a cycle they must meet inside it. **Then reset one pointer to the head and move both one step at a time: they meet at the cycle entrance.** (Reason: if the head is `a` steps from the entrance and the meeting point is `b` steps past the entrance, then `a` equals the remaining cycle length minus `b`, modulo the cycle length.)

```python
def cycle_start(head):                          # Floyd's tortoise and hare
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast:
            slow = head                         # distance from head to the entrance equals meeting point to the entrance
            while slow is not fast: slow, fast = slow.next, fast.next
            return slow
    return None
```

```text
list 0->1->2->3->4->5->(back to 2)
slow | fast
-----+-----
1    | 2   
2    | 4   
3    | 2   
4    | 4   
they meet; reset one pointer to head and walk both 1 step at a time -> they meet at node 2, the cycle entrance
```

### Merge two sorted lists with a dummy head

The dummy node removes the special case for the first element. Attach whichever node is smaller, and at the end attach the leftover tail.

```python
def merge_two(a, b):
    dummy = cur = ListNode()
    while a and b:
        if a.val <= b.val: cur.next, a = a, a.next
        else: cur.next, b = b, b.next
        cur = cur.next
    cur.next = a or b
    return dummy.next
```

### Remove the Nth node from the end: a gap of N between two pointers

Advance `fast` N steps first, then move both until `fast` reaches the last node. `slow` then stands just **before** the node to delete. The dummy head makes deleting the first node work.

```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head); fast = slow = dummy
    for _ in range(n): fast = fast.next         # open a gap of n nodes
    while fast.next: fast, slow = fast.next, slow.next
    slow.next = slow.next.next
    return dummy.next
```

### Middle of the list

```python
def middle(head):
    slow = fast = head
    while fast and fast.next: slow, fast = slow.next, fast.next.next
    return slow
```

**Classics:** reverse linked list (iterative and recursive), merge two sorted lists, linked list cycle I and II, middle of the list, remove Nth from end, palindrome linked list (find the middle, reverse the second half, compare), reorder list, reverse nodes in k-group, copy list with random pointer (hash map from old node to new node), add two numbers, intersection of two lists, LRU cache (hash map plus doubly linked list, in the earlier notes).

**Traps:** forgetting to return the new head after reversing, not handling the empty list and the single node, creating an accidental cycle when you forget to set a tail's `next` to `None`, and comparing nodes with `==` when you mean identity (`is`).

## Part 8: Stack and monotonic stack

**Recognise it:** matching pairs (brackets, nested structures), undoing, expression evaluation, and the **'next greater or smaller element'** family (spans, temperatures, histograms). **The idea of the monotonic stack:** keep a stack of indices whose values are **sorted** (decreasing for 'next greater'). When a new element breaks the order, it is the answer for everything it pops. Every index is pushed once and popped once, so **O(n) overall** even though there is a loop inside a loop.

### Valid parentheses

Push openers, and on a closer the top of the stack must be the matching opener. At the end the stack must be empty.

```python
def valid_parentheses(s):
    pairs, st = {')': '(', ']': '[', '}': '{'}, []
    for ch in s:
        if ch in pairs:
            if not st or st.pop() != pairs[ch]: return False
        else: st.append(ch)
    return not st
```

### Next greater element

```python
def next_greater(a):
    res, st = [-1] * len(a), []                 # st holds indices still waiting for a greater value
    for i, x in enumerate(a):
        while st and a[st[-1]] < x: res[st.pop()] = x
        st.append(i)
    return res
```

```text
input: [2, 1, 2, 4, 3]  (stack shows values, kept decreasing)
i | x | popped (answer = x) | stack  | result so far       
--+---+---------------------+--------+---------------------
0 | 2 | -                   | [2]    | [-1, -1, -1, -1, -1]
1 | 1 | -                   | [2, 1] | [-1, -1, -1, -1, -1]
2 | 2 | a[1]=1              | [2, 2] | [-1, 2, -1, -1, -1] 
3 | 4 | a[2]=2, a[0]=2      | [4]    | [4, 2, 4, -1, -1]   
4 | 3 | -                   | [4, 3] | [4, 2, 4, -1, -1]   
```

### Daily temperatures: the distance to the next warmer day

Same stack, but record the **index difference** when you pop.

```python
def daily_temperatures(t):
    ans, st = [0] * len(t), []
    for i, x in enumerate(t):
        while st and t[st[-1]] < x:
            j = st.pop(); ans[j] = i - j
        st.append(i)
    return ans
```

### Largest rectangle in a histogram

For each bar the best rectangle that uses it as the **shortest** bar extends left and right until a shorter bar. A stack of **increasing** heights finds both limits: when a bar is popped, the new top of the stack is the nearest shorter bar on the left, and the current index is the nearest shorter bar on the right. A sentinel `0` at the end flushes the stack.

```python
def largest_rectangle(h):
    st, best = [], 0                            # indices with increasing heights
    for i, x in enumerate(h + [0]):             # a sentinel zero flushes the stack at the end
        while st and h[st[-1]] >= x:
            height = h[st.pop()]
            left = st[-1] if st else -1
            best = max(best, height * (i - left - 1))
        st.append(i)
    return best
```

```text
heights [2, 1, 5, 6, 2, 3] (a sentinel 0 is appended)
i | x | rectangles closed                                          | stack (heights) | best
--+---+------------------------------------------------------------+-----------------+-----
0 | 2 | -                                                          | [2]             | 0   
1 | 1 | h=2 width=1 area=2                                         | [1]             | 2   
2 | 5 | -                                                          | [1, 5]          | 2   
3 | 6 | -                                                          | [1, 5, 6]       | 2   
4 | 2 | h=6 width=1 area=6; h=5 width=2 area=10                    | [1, 2]          | 10  
5 | 3 | -                                                          | [1, 2, 3]       | 10  
6 | 0 | h=3 width=1 area=3; h=2 width=4 area=8; h=1 width=6 area=6 | [0]             | 10  
```

**Classics:** valid parentheses, min stack, evaluate reverse Polish notation, daily temperatures, next greater element I and II (circular: loop twice), stock span, largest rectangle in histogram, maximal rectangle (apply the histogram per row), trapping rain water (stack version), remove K digits, sum of subarray minimums, decode string, simplify path.

**Traps:** storing values instead of indices when you need distances or widths, forgetting the sentinel that flushes remaining elements, using `<` versus `<=` (this decides how **equal** elements are treated, and the correct choice depends on the problem), and popping from an empty stack.

## Part 9: Heaps and top-K

**Recognise it:** 'K largest', 'K smallest', 'K most frequent', 'merge K sorted', 'schedule by priority', 'median of a stream'. **The idea:** a heap gives the smallest (or largest) element in O(1) and insert or remove in O(log n). **To find the K largest, keep a min-heap of size K:** the root is the K-th largest so far and the smallest of the current best K, so anything smaller than the root can be ignored. O(n log K), better than sorting when K is small.

### K-th largest element

```python
def kth_largest(nums, k):
    heap = []                                   # min-heap of the k largest seen so far
    for x in nums:
        if len(heap) < k: heapq.heappush(heap, x)
        elif x > heap[0]: heapq.heapreplace(heap, x)
    return heap[0]
```

### Top K frequent elements

```python
def top_k_frequent(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]
```

(`most_common` uses a heap internally for small K. The bucket-sort variant, with buckets indexed by frequency, is O(n).)

### Median of a data stream: two heaps

Keep the **smaller half in a max-heap** and the **larger half in a min-heap**, balanced so their sizes differ by at most one. The median is the top of the larger heap, or the average of the two tops. Each insert is O(log n) and the median is O(1).

```python
class MedianFinder:
    def __init__(self): self.lo, self.hi = [], []         # lo: max-heap (negated), hi: min-heap
    def add(self, x):
        heapq.heappush(self.lo, -x)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))   # move the largest of lo to hi
        if len(self.hi) > len(self.lo): heapq.heappush(self.lo, -heapq.heappop(self.hi))
    def median(self):
        return -self.lo[0] if len(self.lo) > len(self.hi) else (-self.lo[0] + self.hi[0]) / 2
```

```text
two heaps: lo holds the smaller half (max-heap), hi the larger half (min-heap)
add | lo (smaller half) | hi (larger half) | median
----+-------------------+------------------+-------
5   | [5]               | []               | 5     
2   | [2]               | [5]              | 3.5   
8   | [2, 5]            | [8]              | 5     
1   | [1, 2]            | [5, 8]           | 3.5   
9   | [1, 2, 5]         | [8, 9]           | 5     
```

### Merge K sorted lists

Keep one pointer into each list and a min-heap of the current heads. Pop the smallest, push the next element from the same list. O(N log K) for N total elements.

```python
def merge_k_sorted(lists):
    heap = [(l[0], i, 0) for i, l in enumerate(lists) if l]
    heapq.heapify(heap); out = []
    while heap:
        v, i, j = heapq.heappop(heap); out.append(v)
        if j + 1 < len(lists[i]): heapq.heappush(heap, (lists[i][j + 1], i, j + 1))
    return out
```

**Classics:** K-th largest element, top K frequent elements, K closest points to the origin, merge K sorted lists, find median from data stream, task scheduler, reorganise string, connect ropes, meeting rooms II, sliding window median, ugly numbers, Dijkstra (Part 12).

**Traps:** `heapq` is a min-heap (negate for a max-heap), pushing items that cannot be compared (add a tiebreaker index), and calling `sorted` inside a loop when a heap would do.

## Part 10: Intervals

**Recognise it:** ranges with a start and an end, 'overlapping', 'merge', 'insert', 'how many rooms'. **The idea:** **sort by start time** and sweep, so only the **last merged interval** can overlap the next one. Remember to decide whether **touching** intervals (`[1,2]` and `[2,3]`) count as overlapping, because it changes the comparison between `<` and `<=`.

### Merge intervals

```python
def merge_intervals(iv):
    out = []
    for s, e in sorted(iv):
        if out and s <= out[-1][1]: out[-1][1] = max(out[-1][1], e)
        else: out.append([s, e])
    return out
```

### Insert an interval into a sorted, non-overlapping list

Three phases: copy everything that ends **before** the new interval starts, **absorb** everything that overlaps by widening the new interval, then copy the rest.

```python
def insert_interval(iv, new):
    out, i, n = [], 0, len(iv)
    while i < n and iv[i][1] < new[0]: out.append(iv[i]); i += 1          # strictly before
    while i < n and iv[i][0] <= new[1]:                                   # overlapping: absorb
        new = [min(new[0], iv[i][0]), max(new[1], iv[i][1])]; i += 1
    out.append(new)
    out.extend(iv[i:])                                                    # strictly after
    return out
```

### Minimum meeting rooms: sort starts and ends separately

A room is needed whenever a meeting starts before the **earliest still-running** meeting ends. Sorting the starts and the ends independently lets a single sweep count the concurrent meetings.

```python
def min_meeting_rooms(iv):
    starts, ends = sorted(s for s, e in iv), sorted(e for s, e in iv)
    rooms = best = j = 0
    for s in starts:
        while ends[j] <= s: j += 1; rooms -= 1                            # a meeting ended before this one starts
        rooms += 1; best = max(best, rooms)
    return best
```

```text
merge [[1,3],[8,10],[2,6],[15,18],[17,20]] -> [[1, 6], [8, 10], [15, 20]]
insert [4,8] into [[1,2],[3,5],[6,7],[8,10],[12,16]] -> [[1, 2], [3, 10], [12, 16]]
meeting rooms for [(0,30),(5,10),(15,20)] -> 2
```

**Classics:** merge intervals, insert interval, non-overlapping intervals (greedy: sort by **end** time and keep what fits, Part 15), meeting rooms I and II, interval list intersections (two pointers), employee free time, car pooling and flight bookings (difference array), the skyline problem (sweep line with a heap).

**Traps:** sorting by the wrong key, mutating the input list while iterating, mishandling touching intervals, and forgetting that the last interval still has to be appended.

## Part 11: Trees

**Recognise it:** hierarchical data, 'depth', 'path', 'ancestor', 'level by level', 'is it a valid BST'. **The idea:** nearly every tree problem is **'recurse on the children, then combine'**. The only design decision is **what flows down** (arguments such as bounds or a running sum) and **what flows up** (return values such as heights). Name them before you write code.

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None): self.val, self.left, self.right = val, left, right
```

`build_tree` builds a tree from a level-order list with `None` for missing children, which is the format most platforms use:

```python
def build_tree(vals):                           # level-order list, None for a missing child
    if not vals or vals[0] is None: return None
    root = TreeNode(vals[0]); q, i = deque([root]), 1
    while q and i < len(vals):
        node = q.popleft()
        if i < len(vals) and vals[i] is not None: node.left = TreeNode(vals[i]); q.append(node.left)
        i += 1
        if i < len(vals) and vals[i] is not None: node.right = TreeNode(vals[i]); q.append(node.right)
        i += 1
    return root
```

### Traversals and what each is good for

- **Inorder** (left, node, right): on a **BST it yields sorted order**, the key BST fact. **Preorder** (node first): copying a tree, serialising. **Postorder** (children first): deleting, computing heights. **Level order** (BFS): anything 'by level'.
- The iterative inorder below avoids Python's recursion limit on a chain-shaped tree of depth 10^5.

```python
def inorder(root):                              # iterative: no recursion limit problems
    out, st, cur = [], [], root
    while cur or st:
        while cur: st.append(cur); cur = cur.left
        cur = st.pop(); out.append(cur.val); cur = cur.right
    return out
```

### Level order: the 'process one level' trick

At the top of the loop the queue holds **exactly one level**, so `for _ in range(len(q))` processes it. Variants: right-side view (last of each level), zigzag, average per level, minimum depth, connect next pointers.

```python
def level_order(root):
    if not root: return []
    q, out = deque([root]), []
    while q:
        out.append([n.val for n in q])           # everything currently in the queue is one level
        for _ in range(len(q)):
            n = q.popleft()
            if n.left: q.append(n.left)
            if n.right: q.append(n.right)
    return out
```

### Depth and diameter: what flows up

`max_depth` returns a height upward. The **diameter** needs two numbers at each node: the path that **bends here** (`left + right`, recorded in a variable) and the path that **continues upward** (`1 + max(left, right)`, returned). The same shape solves maximum path sum, longest univalue path, and many others.

```python
def max_depth(root):
    return 0 if not root else 1 + max(max_depth(root.left), max_depth(root.right))
```

```python
def diameter(root):                              # longest path measured in edges
    best = 0
    def height(n):
        nonlocal best
        if not n: return 0
        l, r = height(n.left), height(n.right)
        best = max(best, l + r)                  # a path that bends at n
        return 1 + max(l, r)                     # a path that continues upward
    height(root)
    return best
```

### Validate a BST: what flows down

Checking only 'left is smaller, right is larger' against the **parent** is the classic mistake. A node deep in the left subtree must be smaller than **every ancestor** it sits to the left of. Pass the allowed `(lo, hi)` range down.

```python
def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root: return True
    if not (lo < root.val < hi): return False
    return is_valid_bst(root.left, lo, root.val) and is_valid_bst(root.right, root.val, hi)
```

### Lowest common ancestor

If the current node is `p` or `q`, it is the answer for its subtree. Otherwise recurse both sides: if **both** return something, the current node is the split point and the LCA, otherwise return whichever side found something. (In a BST you can walk down comparing values instead.)

```python
def lca(root, p, q):                             # p and q are nodes of the tree
    if not root or root is p or root is q: return root
    l, r = lca(root.left, p, q), lca(root.right, p, q)
    return root if l and r else (l or r)
```

### Serialise and deserialise a tree

Level-order with a marker for `None`. This is the same idea as the heap-array layout of a complete tree.

```python
def serialize(root):
    out, q = [], deque([root])
    while q:
        n = q.popleft()
        if n is None: out.append('#'); continue
        out.append(str(n.val)); q.append(n.left); q.append(n.right)
    return ','.join(out)
```

```python
def deserialize(s):
    vals = s.split(',')
    if vals[0] == '#': return None
    root = TreeNode(int(vals[0])); q, i = deque([root]), 1
    while q:
        n = q.popleft()
        for side in ('left', 'right'):
            if i < len(vals) and vals[i] != '#':
                child = TreeNode(int(vals[i])); setattr(n, side, child); q.append(child)
            i += 1
    return root
```

**Classics:** maximum depth, invert tree, same tree, symmetric tree, diameter, balanced tree, level order and its variants, path sum I, II and III, validate BST, kth smallest in BST (inorder), LCA of a BST and of a binary tree, serialise and deserialise, construct a tree from preorder and inorder, binary tree maximum path sum, flatten to a linked list, count good nodes.

**Traps:** forgetting the `None` base case, validating a BST against only the parent, confusing **height in nodes with height in edges**, mutating shared state (use `nonlocal` or return values), recursion depth on skewed trees, and assuming a **binary tree** is a **BST**.

## Part 12: Graphs

**Recognise it:** relationships, grids, 'connected', 'shortest', 'dependencies', 'cycle'. **The idea:** a grid is a graph (cells are nodes, the four neighbours are edges). Choose the traversal by the question:

| Question | Tool |
| --- | --- |
| Fewest steps, unweighted | **BFS** |
| Reachability, components, cycle detection, paths | **DFS** (or BFS) |
| Ordering with dependencies | **Topological sort** |
| Merge groups, 'are these connected' as edges arrive | **Union-find** |
| Cheapest path, non-negative weights | **Dijkstra** |
| Negative weights, or 'at most K edges' | **Bellman-Ford** |
| All pairs, small graph (about 400 nodes) | **Floyd-Warshall** |
| Cheapest way to connect everything | **Minimum spanning tree** (Kruskal or Prim) |

**Representation:** an **adjacency list** (`dict` or list of lists) for sparse graphs, which is almost always the right choice. **Mark nodes visited when you push them, not when you pop them**, otherwise a node can enter the queue many times.

### Number of islands: BFS on a grid

```python
def num_islands(grid):
    R, C, seen, count = len(grid), len(grid[0]), set(), 0
    for r in range(R):
        for c in range(C):
            if grid[r][c] == 1 and (r, c) not in seen:
                count += 1; seen.add((r, c)); q = deque([(r, c)])
                while q:
                    x, y = q.popleft()
                    for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                        nx, ny = x + dx, y + dy
                        if 0 <= nx < R and 0 <= ny < C and grid[nx][ny] == 1 and (nx, ny) not in seen:
                            seen.add((nx, ny)); q.append((nx, ny))          # mark when PUSHED, not popped
    return count
```

### Shortest path in a grid: BFS gives fewest steps

```python
def shortest_path_grid(grid):                    # 0 = open, 1 = wall, 4-directional, from top-left to bottom-right
    R, C = len(grid), len(grid[0])
    if grid[0][0] or grid[R - 1][C - 1]: return -1
    dist, q = {(0, 0): 1}, deque([(0, 0)])
    while q:
        x, y = q.popleft()
        if (x, y) == (R - 1, C - 1): return dist[(x, y)]
        for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nx, ny = x + dx, y + dy
            if 0 <= nx < R and 0 <= ny < C and not grid[nx][ny] and (nx, ny) not in dist:
                dist[(nx, ny)] = dist[(x, y)] + 1; q.append((nx, ny))
    return -1
```

### Topological sort (Kahn's algorithm)

Repeatedly take a node with **indegree 0**, remove it, and decrement its neighbours' indegrees. If you cannot take all `n` nodes, **there is a cycle** (so this also answers 'can all courses be finished'). The alternative is DFS postorder reversed.

```python
def topo_sort(n, edges):                         # Kahn's algorithm; returns [] when there is a cycle
    adj, indeg = [[] for _ in range(n)], [0] * n
    for u, v in edges: adj[u].append(v); indeg[v] += 1
    q, order = deque(i for i in range(n) if indeg[i] == 0), []
    while q:
        u = q.popleft(); order.append(u)
        for v in adj[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return order if len(order) == n else []
```

```text
edges [(5, 2), (5, 0), (4, 0), (4, 1), (2, 3), (3, 1)]
take | its out-edges | newly free (indegree 0) | queue now
-----+---------------+-------------------------+----------
4    | [0, 1]        | []                      | [5]      
5    | [2, 0]        | [2, 0]                  | [2, 0]   
2    | [3]           | [3]                     | [0, 3]   
0    | []            | []                      | [3]      
3    | [1]           | [1]                     | [1]      
1    | []            | []                      | []       
order: [4, 5, 2, 0, 3, 1]
```

### Union-find (disjoint set union)

`find` with **path compression** and `union` by size give nearly constant time per operation. It answers 'are a and b connected' and counts components as edges arrive. A `union` that returns `False` means the edge **closes a cycle**, which is exactly how you find a redundant edge and how Kruskal avoids cycles.

```python
class DSU:
    def __init__(self, n): self.p, self.sz, self.comps = list(range(n)), [1] * n, n
    def find(self, x):
        while self.p[x] != x: self.p[x] = self.p[self.p[x]]; x = self.p[x]
        return x
    def union(self, a, b):
        a, b = self.find(a), self.find(b)
        if a == b: return False
        if self.sz[a] < self.sz[b]: a, b = b, a
        self.p[b] = a; self.sz[a] += self.sz[b]; self.comps -= 1
        return True
```

### Dijkstra's algorithm: shortest paths with non-negative weights

Always expand the **closest unfinalised node** (the heap's minimum). The `if d > dist[u]: continue` line skips **stale** heap entries (a node can be pushed several times with improving distances). O((V + E) log V). **It is wrong with negative edges.**

```python
def dijkstra(n, edges, src):                     # edges: (u, v, w) directed, w >= 0
    adj = defaultdict(list)
    for u, v, w in edges: adj[u].append((v, w))
    dist = [float('inf')] * n; dist[src] = 0; pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue                 # stale queue entry
        for v, w in adj[u]:
            if d + w < dist[v]: dist[v] = d + w; heapq.heappush(pq, (dist[v], v))
    return dist
```

```text
edges (u, v, w): [(0, 1, 4), (0, 2, 1), (2, 1, 2), (1, 3, 1), (2, 3, 5)]
pop (dist,node) | relaxations          | dist array    
----------------+----------------------+---------------
(0,0)           | dist[1]=4, dist[2]=1 | [0, 4, 1, inf]
(1,2)           | dist[1]=3, dist[3]=6 | [0, 3, 1, 6]  
(3,1)           | dist[3]=4            | [0, 3, 1, 4]  
(4,1)           | stale, skip          | [0, 3, 1, 4]  
(4,3)           | no improvement       | [0, 3, 1, 4]  
(6,3)           | stale, skip          | [0, 3, 1, 4]  
```

### Bipartite check

Colour with two colours using BFS. A conflict along an edge means an **odd cycle**, so the graph is not bipartite. Remember to start a BFS from **every unvisited node** (the graph may be disconnected).

```python
def is_bipartite(n, edges):
    adj = defaultdict(list)
    for u, v in edges: adj[u].append(v); adj[v].append(u)
    color = {}
    for s in range(n):
        if s in color: continue
        color[s] = 0; q = deque([s])
        while q:
            u = q.popleft()
            for v in adj[u]:
                if v not in color: color[v] = 1 - color[u]; q.append(v)
                elif color[v] == color[u]: return False
    return True
```

### Minimum spanning tree: Kruskal

Sort edges by weight and add an edge if it connects two different components (union-find). A spanning tree has exactly `n - 1` edges, so fewer means the graph is disconnected. Prim's algorithm grows one tree with a heap and suits dense graphs.

```python
def kruskal(n, edges):                           # (u, v, w); returns the MST weight or None if disconnected
    dsu, total, used = DSU(n), 0, 0
    for u, v, w in sorted(edges, key=lambda e: e[2]):
        if dsu.union(u, v): total += w; used += 1
    return total if used == n - 1 else None
```

**Classics:** number of islands, max area of island, flood fill, rotting oranges (multi-source BFS: push every rotten orange first), clone graph, course schedule I and II, number of connected components, redundant connection, graph valid tree, word ladder, network delay time, cheapest flights within K stops, path with minimum effort, pacific atlantic water flow, surrounded regions, alien dictionary, min cost to connect all points.

**Traps:** marking visited too late, forgetting disconnected components, running Dijkstra with negative weights, treating an undirected edge as directed (add both directions), mixing up **node count versus edge count** in the tree check, and recursion depth in DFS on large graphs (use an explicit stack).

## Part 13: Backtracking

**Recognise it:** 'generate **all** subsets, permutations, combinations, partitions, placements', and constraint puzzles (N-Queens, Sudoku, word search). **The idea:** build the answer one decision at a time in a path: **choose, explore, un-choose**. The recursion tree **is** the search space, and **pruning** (stopping a branch as soon as it cannot succeed) is what makes it fast enough. Complexity is exponential by nature (2^n subsets, n! permutations), so n is small.

### Subsets: include or skip each element

```python
def subsets(nums):
    res, path = [], []
    def go(i):
        if i == len(nums): res.append(path[:]); return
        path.append(nums[i]); go(i + 1)          # choose
        path.pop(); go(i + 1)                    # skip (un-choose)
    go(0); return res
```

### Permutations with duplicates: sort, then skip repeats at the same depth

`if i and nums[i] == nums[i-1] and not used[i-1]` skips a duplicate only when its identical predecessor has not been used at this level, which prevents generating the same permutation twice.

```python
def permutations_unique(nums):                   # handles duplicates by sorting and skipping repeats at the same depth
    nums, res, used, path = sorted(nums), [], [False] * len(nums), []
    def go():
        if len(path) == len(nums): res.append(path[:]); return
        for i in range(len(nums)):
            if used[i] or (i and nums[i] == nums[i - 1] and not used[i - 1]): continue
            used[i] = True; path.append(nums[i]); go(); path.pop(); used[i] = False
    go(); return res
```

### Combination sum: pass the start index, and `i` (not `i + 1`) to allow reuse

The **start index** forces a non-decreasing order, so `[2, 3]` and `[3, 2]` are not both generated. Because the candidates are sorted, `break` (not `continue`) prunes everything after the first candidate that is too large.

```python
def combination_sum(cands, target):              # unlimited reuse, distinct positive candidates
    cands, res, path = sorted(cands), [], []
    def go(start, remain):
        if remain == 0: res.append(path[:]); return
        for i in range(start, len(cands)):
            if cands[i] > remain: break          # sorted, so every later candidate is too big as well
            path.append(cands[i]); go(i, remain - cands[i]); path.pop()   # pass i, not i+1: reuse allowed
    go(0, target); return res
```

### N-Queens: constraint sets make each check O(1)

Track used columns and the two diagonals (`r - c` and `r + c` are constant along a diagonal).

```python
def n_queens(n):
    cols, d1, d2, count = set(), set(), set(), 0
    def go(r):
        nonlocal count
        if r == n: count += 1; return
        for c in range(n):
            if c in cols or (r - c) in d1 or (r + c) in d2: continue
            cols.add(c); d1.add(r - c); d2.add(r + c)
            go(r + 1)
            cols.remove(c); d1.remove(r - c); d2.remove(r + c)
    go(0); return count
```

### Generate parentheses: constrain while building

Only open if you have openings left, and only close if there are more opens than closes. Invalid strings are **never generated**, which is stronger than generating everything and filtering.

```python
def generate_parentheses(n):
    res = []
    def go(s, opens, closes):
        if len(s) == 2 * n: res.append(s); return
        if opens < n: go(s + '(', opens + 1, closes)
        if closes < opens: go(s + ')', opens, closes + 1)   # never close more than were opened
    go('', 0, 0); return res
```

```text
subsets([1,2,3]) -> [[1, 2, 3], [1, 2], [1, 3], [1], [2, 3], [2], [3], []]
permutations_unique([1,1,2]) -> [[1, 1, 2], [1, 2, 1], [2, 1, 1]]
combination_sum([2,3,6,7], 7) -> [[2, 2, 3], [7]]
n_queens(1..8) -> [1, 0, 0, 2, 10, 4, 40, 92]
generate_parentheses(3) -> ['((()))', '(()())', '(())()', '()(())', '()()()']
```

**Classics:** subsets I and II, permutations I and II, combination sum I, II and III, letter combinations of a phone number, generate parentheses, palindrome partitioning, N-Queens, Sudoku solver, word search, restore IP addresses.

**Traps:** appending the **same list object** instead of a copy (`res.append(path[:])`, not `res.append(path)`), forgetting to undo the choice, generating duplicates when the input has repeats, and missing the pruning that turns an impossible runtime into a feasible one. If subproblems **repeat**, the same recursion with a **memo** is dynamic programming.

## Part 14: Dynamic programming

**Recognise it:** 'count the number of ways', 'minimum or maximum cost', 'can it be done', 'longest or shortest sequence', and a brute force that **recomputes the same subproblems**. **The idea:** if the problem has **optimal substructure** (the best answer is built from best answers of smaller pieces) and **overlapping subproblems**, store each subproblem's answer once.

### The five-step method (say it aloud in the interview)

1. **State:** what does `dp[i]` (or `dp[i][j]`) **mean**, in one precise sentence? ('minimum coins to make amount `i`', 'length of the LCS of the first `i` and first `j` characters').
2. **Transition:** how is the state computed from smaller states?
3. **Base cases:** the smallest states, answered directly.
4. **Order:** compute so that dependencies are ready (increasing `i`, or by increasing length).
5. **Answer and space:** which cell is the answer, and can a rolling row or two variables replace the table?

**Top-down (memoised recursion) versus bottom-up (table):** top-down is easier to write from the brute force and only computes needed states, bottom-up avoids recursion limits and enables space optimisation. Start top-down to find the recurrence, convert if needed. In Python, `@functools.lru_cache(None)` turns a recursive brute force into memoised DP in one line (watch the recursion depth).

**Families:** 1D linear (climbing stairs, house robber, decode ways), unbounded and bounded knapsack (coin change, subset sum, partition), grids (paths, minimum path sum), two strings (LCS, edit distance), subsequences (LIS), intervals and partitions, trees and bitmasks.

### 1D: climbing stairs and house robber

`dp[i] = dp[i-1] + dp[i-2]` for stairs, constant space with two variables. For house robber, at each house you either **rob it** (previous house skipped) or **skip it**.

```python
def climb_stairs(n):
    a, b = 1, 1
    for _ in range(n - 1): a, b = b, a + b
    return b if n >= 1 else 1
```

```python
def house_robber(a):
    take, skip = 0, 0
    for x in a: take, skip = skip + x, max(take, skip)     # take: robbed the last house; skip: did not
    return max(take, skip)
```

### Coin change (minimum coins): unbounded knapsack

`dp[a] = min over coins c of dp[a - c] + 1`. Initialise with 'impossible' (`amount + 1`), and report `-1` if it stays impossible. **Greedy fails** for arbitrary denominations (coins 1, 3, 4 and amount 6: greedy takes 4+1+1, the best is 3+3).

```python
def coin_change(coins, amount):
    INF = amount + 1; dp = [0] + [INF] * amount
    for a in range(1, amount + 1):
        for c in coins:
            if c <= a: dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] < INF else -1
```

```text
coins [1, 2, 5], amount 11
amount :  0  1  2  3  4  5  6  7  8  9 10 11
min    :  0  1  1  2  2  1  2  2  3  3  2  3
answer dp[11] = 3  (5 + 5 + 1)
```

### Coin change (number of ways): the loop order matters

To count **combinations** (order does not matter), put the **coins in the outer loop**. Swapping the loops counts **permutations** (sequences) instead.

```python
def coin_change_ways(coins, amount):             # number of combinations: coins in the OUTER loop
    dp = [1] + [0] * amount
    for c in coins:
        for a in range(c, amount + 1): dp[a] += dp[a - c]
    return dp[amount]
```

### Longest increasing subsequence in O(n log n)

`tails[k]` is the **smallest possible tail** of an increasing subsequence of length `k + 1`. For each number, binary search where it fits: replace the first tail that is `>= x` (strict increase), or append. The **length** of `tails` is the answer, but `tails` itself is **not** the subsequence. For non-decreasing, use `bisect_right`.

```python
def lis_nlogn(a):
    import bisect
    tails = []
    for x in a:
        i = bisect.bisect_left(tails, x)         # strictly increasing subsequence
        if i == len(tails): tails.append(x)
        else: tails[i] = x
    return len(tails)
```

### Longest common subsequence: the two-string table

`dp[i][j]` is the LCS of the first `i` characters of `a` and the first `j` of `b`. Match: take the diagonal plus one. Mismatch: the better of dropping a character from either string.

```python
def lcs(a, b):
    dp = [[0] * (len(b) + 1) for _ in range(len(a) + 1)]
    for i in range(1, len(a) + 1):
        for j in range(1, len(b) + 1):
            dp[i][j] = dp[i - 1][j - 1] + 1 if a[i - 1] == b[j - 1] else max(dp[i - 1][j], dp[i][j - 1])
    return dp[-1][-1]
```

```text
      -  B  D  C  A  B
-   0  0  0  0  0  0
A   0  0  0  0  1  1
B   0  1  1  1  1  2
C   0  1  1  2  2  2
B   0  1  1  2  2  3
D   0  1  2  2  2  3
LCS length of ABCBD and BDCAB = 3
```

### Edit distance: three operations

`dp[i][j]` is the minimum edits to turn the first `i` characters of `a` into the first `j` of `b`. On a mismatch take `1 + min(delete, insert, replace)`, which are the cells above, to the left, and diagonal.

```python
def edit_distance(a, b):
    prev = list(range(len(b) + 1))
    for i in range(1, len(a) + 1):
        cur = [i] + [0] * len(b)
        for j in range(1, len(b) + 1):
            cur[j] = prev[j - 1] if a[i - 1] == b[j - 1] else 1 + min(prev[j], cur[j - 1], prev[j - 1])   # delete, insert, replace
        prev = cur
    return prev[-1]
```

```text
      -  r  o  s
-   0  1  2  3
h   1  1  2  3
o   2  2  1  2
r   3  2  2  2
s   4  3  3  2
e   5  4  4  3
edit distance horse -> ros = 3
```

### 0/1 knapsack: iterate capacity **backwards**

Each item may be used **once**. Going backwards over capacity guarantees that `dp[c - w]` still refers to the state **before** this item was added. Going forwards would allow reusing the item (that is the **unbounded** variant, used for coin change).

```python
def knapsack_01(weights, values, cap):
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(cap, w - 1, -1):          # BACKWARDS so each item is used at most once
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]
```

### Partition equal subset sum, and word break

Subset sum is a knapsack with a boolean state ('is this sum reachable'). Word break is 'can the prefix of length `i` be segmented', checking every split point.

```python
def can_partition(nums):
    total = sum(nums)
    if total % 2: return False
    reach = {0}
    for x in nums: reach |= {s + x for s in reach}
    return total // 2 in reach
```

```python
def word_break(s, words):
    ws, dp = set(words), [True] + [False] * len(s)
    for i in range(1, len(s) + 1):
        dp[i] = any(dp[j] and s[j:i] in ws for j in range(i))
    return dp[-1]
```

### Grid paths with obstacles

```python
def unique_paths_blocked(grid):                  # 1 = obstacle
    R, C = len(grid), len(grid[0]); dp = [[0] * C for _ in range(R)]
    for r in range(R):
        for c in range(C):
            if grid[r][c]: continue
            if r == c == 0: dp[r][c] = 1
            else: dp[r][c] = (dp[r - 1][c] if r else 0) + (dp[r][c - 1] if c else 0)
    return dp[-1][-1]
```

### Maximum subarray (Kadane) and maximum product subarray

Kadane: the best subarray **ending here** is either the element alone or the element extending the best one ending before it. For **products**, track both the maximum **and minimum** ending here, because multiplying by a negative swaps them.

```python
def max_subarray(a):                             # Kadane
    cur = best = a[0]
    for x in a[1:]:
        cur = max(x, cur + x); best = max(best, cur)
    return best
```

```text
input: [-2, 1, -3, 4, -1, 2, 1, -5, 4]
x  | best sum ending here | best overall
---+----------------------+-------------
-2 | -2                   | -2          
1  | 1                    | 1           
-3 | -2                   | 1           
4  | 4                    | 4           
-1 | 3                    | 4           
2  | 5                    | 5           
1  | 6                    | 6           
-5 | 1                    | 6           
4  | 5                    | 6           
answer: 6  (subarray [4, -1, 2, 1])
```

```python
def max_product_subarray(a):
    hi = lo = best = a[0]
    for x in a[1:]:
        cands = (x, hi * x, lo * x)              # a negative number swaps the roles of max and min
        hi, lo = max(cands), min(cands); best = max(best, hi)
    return best
```

### Palindromes, decoding, stock

Longest palindromic substring: expand around each of the 2n-1 centres (O(n^2), O(1) space). Decode ways: `dp[i]` adds `dp[i-1]` if the single digit is valid and `dp[i-2]` if the pair is in 10 to 26. Best time to buy and sell once: track the lowest price so far.

```python
def longest_palindrome(s):                       # expand around every centre
    best = ''
    for c in range(len(s)):
        for l, r in ((c, c), (c, c + 1)):
            while l >= 0 and r < len(s) and s[l] == s[r]: l -= 1; r += 1
            if r - l - 1 > len(best): best = s[l + 1:r]
    return best
```

```python
def decode_ways(s):
    if not s or s[0] == '0': return 0
    a, b = 1, 1                                  # ways for the prefix two back, one back
    for i in range(1, len(s)):
        cur = 0
        if s[i] != '0': cur += b
        if 10 <= int(s[i - 1:i + 1]) <= 26: cur += a
        a, b = b, cur
    return b
```

```python
def best_time_to_buy_sell(prices):
    lo, best = float('inf'), 0
    for p in prices: lo = min(lo, p); best = max(best, p - lo)
    return best
```

**Classics (a minimum set):** climbing stairs, house robber I and II, coin change I and II, decode ways, word break, longest increasing subsequence, longest common subsequence, edit distance, unique paths I and II, minimum path sum, 0/1 knapsack, partition equal subset sum, target sum, longest palindromic substring, maximum subarray, maximum product subarray, best time to buy and sell stock (I to IV and with cooldown), burst balloons (interval DP), matrix chain multiplication, and one bitmask DP (travelling salesman or assign tasks).

**Traps:** a vague state definition, wrong loop order (coin ways, knapsack direction), forgetting base cases, using `[[0] * n] * m`, an off-by-one between 0-based strings and 1-based table indices, not handling 'impossible' states, and **assuming greedy works when it does not**. If it asks for **all** solutions rather than a count or an optimum, DP counts them but cannot list them cheaply, use backtracking.

## Part 15: Greedy

**Recognise it:** an optimisation where a **locally best choice is never regretted**: scheduling, covering, jumping, refuelling. **The idea:** pick the best-looking option now. The danger is that greedy often *looks* right and is wrong, so **justify it** with an **exchange argument** (if an optimal solution made a different choice, you can swap in the greedy choice without making it worse), or **fuzz-test it against a brute force**, which is exactly how these templates were checked.

### Jump game: track the farthest reachable index

```python
def can_jump(a):
    reach = 0
    for i, x in enumerate(a):
        if i > reach: return False
        reach = max(reach, i + x)
    return True
```

### Activity selection: sort by **end** time

To fit the maximum number of non-overlapping intervals, always take the compatible interval that **finishes earliest**, because it leaves the most room for the rest. Sorting by **start** time (or by length) is wrong and fails on simple counterexamples.

```python
def max_non_overlapping(iv):                     # activity selection: sort by END time, take whatever fits
    count, end = 0, float('-inf')
    for s, e in sorted(iv, key=lambda x: x[1]):
        if s >= end: count += 1; end = e
    return count
```

### Gas station: if you run out, no start up to here works

If the tank goes negative at station `i`, **none** of the starts from the current start up to `i` can work (each reaches `i` with at least as little fuel as the current one), so restart at `i + 1`. If the total gas is less than the total cost, it is impossible.

```python
def gas_station(gas, cost):
    if sum(gas) < sum(cost): return -1
    start = tank = 0
    for i in range(len(gas)):
        tank += gas[i] - cost[i]
        if tank < 0: start, tank = i + 1, 0      # cannot reach i+1 from any start up to i
    return start
```

**Classics:** jump game I and II, activity selection and non-overlapping intervals, gas station, assign cookies, candy, task scheduler, partition labels, minimum number of arrows to burst balloons, merge triplets, lemonade change, fractional knapsack, Huffman coding, minimum platforms.

**Contrast to remember:** **fractional** knapsack is greedy (take the best value per weight), **0/1** knapsack is DP. Coin change with arbitrary coins is DP, with canonical coin systems (1, 5, 10, 25) greedy works.

## Part 16: Trie, bit manipulation, KMP

### Trie (prefix tree)

**Recognise it:** insert and search words, **prefix** queries, autocomplete, word search II, longest common prefix, maximum XOR of two numbers (a binary trie). Each node maps a character to a child, and a marker flags the end of a word. Insert, search and prefix check are all O(length of the word), independent of how many words are stored. **Trade-off against a hash set:** a trie costs more memory but supports prefix queries, which a set cannot.

```python
class Trie:
    def __init__(self): self.root = {}
    def insert(self, w):
        n = self.root
        for ch in w: n = n.setdefault(ch, {})
        n['$'] = True
    def _walk(self, w):
        n = self.root
        for ch in w:
            if ch not in n: return None
            n = n[ch]
        return n
    def search(self, w): n = self._walk(w); return bool(n and '$' in n)
    def starts_with(self, p): return self._walk(p) is not None
```

### Bit manipulation

**Facts to know cold:** `x & (x - 1)` clears the lowest set bit, `x & -x` isolates it, `x >> 1` halves, `1 << k` is 2^k, `a ^ a = 0` and `a ^ 0 = a` (XOR is its own inverse). A power of two has exactly one set bit. **In Python integers are unbounded**, so for negative numbers under a 32-bit assumption mask with `& 0xFFFFFFFF`.

```python
def single_number(a):
    x = 0
    for v in a: x ^= v                           # equal pairs cancel out
    return x
```

```python
def count_bits_table(n):
    dp = [0] * (n + 1)
    for i in range(1, n + 1): dp[i] = dp[i >> 1] + (i & 1)
    return dp
```

```python
def is_power_of_two(x): return x > 0 and x & (x - 1) == 0
```

**Classics:** single number I, II and III, number of 1 bits, counting bits, power of two, missing number (XOR of indices and values), reverse bits, sum of two integers without `+`, subsets via a bitmask (`for mask in range(1 << n)`), maximum XOR pair.

### KMP: string search without re-scanning

The **prefix function** `pi[i]` is the length of the longest proper prefix of `pat[:i+1]` that is also its suffix. On a mismatch, instead of restarting, fall back to `pi[k-1]`, so the text pointer **never moves backwards**. O(n + m).

```python
def prefix_function(s):
    pi = [0] * len(s)
    for i in range(1, len(s)):
        k = pi[i - 1]
        while k and s[i] != s[k]: k = pi[k - 1]
        if s[i] == s[k]: k += 1
        pi[i] = k
    return pi
```

```python
def kmp_search(text, pat):                       # all start indices of pat in text
    if not pat: return []
    pi, k, out = prefix_function(pat), 0, []
    for i, ch in enumerate(text):
        while k and ch != pat[k]: k = pi[k - 1]
        if ch == pat[k]: k += 1
        if k == len(pat): out.append(i - k + 1); k = pi[k - 1]
    return out
```

```text
pattern  : a  b  a  b  a  c  a
prefix fn: 0  0  1  2  3  0  1
search "abababaca" for "ababaca" -> starts at [2]
```

**Related:** the Z-function (similar idea), Rabin-Karp (rolling hash, watch collisions), 'repeated substring pattern' (`s in (s + s)[1:-1]`), shortest palindrome. Python's `str.find` is already fast, so these matter for interviews more than for OAs.

## Part 17: Practice plan, the must-solve list, and the mistake log

### 17.1 How to practise so it sticks

- **Study by pattern, not by random problem.** Do 8 to 12 problems of one pattern in a row (easy, then medium), so recognition becomes automatic.
- **25 to 30 minutes of honest attempt, then hints.** If stuck, read only the **pattern name**, try again for 10 minutes, then read a solution and **re-implement it from a blank file the next day**, and once more a week later.
- **Say the brute force, the bottleneck, the optimisation, the invariant and the complexity out loud** for every problem. That is exactly what an interviewer scores.
- **Fuzz your own solutions.** Write a 5-line brute force and compare on random small inputs, as this volume did. It catches off-by-one and edge-case bugs that hand-picked examples never will, and it is the single best habit for OAs where you cannot see the hidden tests.
- **Timed OAs:** 10 mocks of 90 minutes, three problems, no hints, solved in order of confidence. After each, log what failed.

### 17.2 The mistake log (keep it, review it before every OA)

| Date | Problem | Pattern | What went wrong | Category | Fix to remember |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  | misread, wrong pattern, off-by-one, edge case, time, bug |  |

The categories repeat. If 'off-by-one' keeps appearing, adopt one binary-search and one window template and stop improvising. If 'edge case' keeps appearing, run the checklist in 1.5 before submitting.

### 17.3 The must-solve list (about 120 problems, grouped by pattern)

| Pattern | Problems |
| --- | --- |
| Arrays and hashing | Two sum, group anagrams, top K frequent, longest consecutive sequence, subarray sum equals K, product of array except self, first missing positive, 4Sum II |
| Two pointers | 3Sum, container with most water, trapping rain water, sort colours, remove duplicates from sorted array, valid palindrome |
| Sliding window | Longest substring without repeating, minimum window substring, longest repeating character replacement, permutation in string, sliding window maximum, max consecutive ones III |
| Prefix sums | Range sum query 2D, subarray sum divisible by K, car pooling, continuous subarray sum, contiguous array |
| Binary search | First and last position, search in rotated array, find minimum in rotated array, find peak, Koko eating bananas, ship packages, split array largest sum, median of two sorted arrays |
| Linked list | Reverse list, merge two lists, cycle II, remove Nth from end, reorder list, reverse in k-group, copy list with random pointer, add two numbers |
| Stack | Valid parentheses, min stack, daily temperatures, next greater element II, largest rectangle in histogram, decode string, evaluate RPN |
| Heap | Kth largest, top K frequent, merge K lists, find median from stream, task scheduler, K closest points |
| Intervals | Merge intervals, insert interval, non-overlapping intervals, meeting rooms II, employee free time |
| Trees | Max depth, invert tree, diameter, level order, validate BST, LCA, kth smallest in BST, serialise and deserialise, binary tree maximum path sum, construct from preorder and inorder |
| Graphs | Number of islands, rotting oranges, clone graph, course schedule I and II, redundant connection, word ladder, network delay time, cheapest flights within K stops, pacific atlantic, min cost to connect all points |
| Backtracking | Subsets I and II, permutations I and II, combination sum, generate parentheses, N-Queens, word search, palindrome partitioning |
| Dynamic programming | Climbing stairs, house robber I and II, coin change I and II, longest increasing subsequence, LCS, edit distance, unique paths, 0/1 knapsack, partition equal subset sum, word break, decode ways, maximum product subarray, stock series, burst balloons |
| Greedy | Jump game I and II, gas station, non-overlapping intervals, partition labels, candy |
| Trie, bits, strings | Implement trie, word search II, single number, counting bits, KMP-style search |

### 17.4 Interview questions about the coding round itself

- **How do you choose between BFS and DFS?** BFS for the fewest steps in an unweighted graph and level-by-level work, DFS for reachability, components, cycle detection and exhaustive search.
- **When does a sliding window fail?** When values can be negative and the condition is a sum (not monotonic). Use prefix sums with a hash map.
- **How do you recognise binary search on the answer?** The answer is a number in a known range and feasibility is monotonic. Write a greedy `ok(x)` and search.
- **Why is a monotonic stack O(n) despite the nested loop?** Each index is pushed once and popped at most once.
- **How do you tell greedy from DP?** Try to prove the greedy choice with an exchange argument or find a counterexample. If a locally best choice can be regretted, use DP.
- **How do you debug a failing hidden test?** Write a brute force, generate random small inputs, compare, and print the first mismatch. Then reduce it to the smallest failing case.
- **How do you test a solution you cannot run against hidden cases?** The edge-case checklist plus a mental run of the smallest examples, and, if you have time, a quick brute-force comparison.
- **What do you do when you are stuck for ten minutes in an OA?** Submit the brute force for partial credit, move to the next problem, and come back with a fresh view.

### Revise in 30 seconds: DSA patterns

- **Routine:** restate, constraints give the complexity, brute force first, find the bottleneck, match the pattern, write the invariant, test small and edge cases.
- **Hashing:** store what you have seen. Prefix sums plus a dict handle negatives. **Two pointers:** sorted input, discard candidates provably. **Sliding window:** non-negative or monotonic conditions, expand then shrink.
- **Binary search:** half-open `[lo, hi)`, `hi = mid` and `lo = mid + 1`, never mix conventions. **On the answer:** monotonic `ok(x)` with a greedy check. **Monotonic stack:** each index pushed and popped once.
- **Heap:** K largest uses a min-heap of size K. Median: two heaps. **Intervals:** sort by start, decide touching. **Trees:** decide what flows down and what flows up.
- **Graphs:** BFS for fewest steps, topological sort for dependencies (leftover nodes mean a cycle), union-find for connectivity, Dijkstra for non-negative weights.
- **Backtracking:** choose, explore, un-choose, copy the path, prune. **DP:** state, transition, base, order, space. Coin ways: coins outer. Knapsack 0/1: capacity backwards. **Greedy:** prove it or fuzz it.
- Fuzz your own solutions against a brute force.

## What Volume 8 covers, and what comes next

Volume 8 covers the DSA section of your checklist in depth: 17 pattern families, each with its recognition signal, invariant, a tested template, a traced example, classic problems and traps, the routine and practice plan, and the must-solve list. Every function was fuzz-tested against a reference before it was pasted here.

Remaining in the series: **Volume 9: programming and systems** (C++, Linux, compilers, computer architecture), then a short **final revision map** that links every 'Revise in 30 seconds' box into one page per round (OA, technical round, interview) and a last-week schedule.