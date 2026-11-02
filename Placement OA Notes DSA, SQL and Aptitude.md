# Placement OA Notes: DSA, SQL and Aptitude

Online assessments reward pattern recognition and clean execution under a clock. These notes follow your master checklist (Sections 1 and 5), with Python templates you can retype from memory. Part A is strategy and the first half of the DSA patterns, Part B is trees, graphs, DP and the rest, Part C is SQL and aptitude.

## Part A: Running an OA

### Time plan (typical 90 min, 3 coding questions)

- First 5 minutes: read every problem. Rank them by how sure you are, not by the order given.
- Easy one: 15 min. Medium: 30 min. Hard: 30 to 35 min. Keep 10 min for re-checking edge cases.
- Always submit a brute force that passes some tests before attempting the optimal one. Partial credit counts, and a working brute force is also your checker for the clever version.
- If you are stuck for 10 minutes with no new idea, switch problems and come back.

### Constraints tell you the complexity

| n | Target complexity | Usual technique |
| --- | --- | --- |
| up to 10 | O(n!) | permutations, backtracking |
| up to 20 | O(2^n \* n) | bitmask, subsets |
| up to 500 | O(n^3) | Floyd-Warshall, interval DP |
| up to 5,000 | O(n^2) | 2D DP, pair checks |
| up to 10^5 or 10^6 | O(n log n) or O(n) | sort, heap, binary search, two pointers, hashing |
| up to 10^9 or 10^18 | O(log n) or O(sqrt n) | binary search on answer, math, fast power |

### Signal to pattern

- Sorted array, or 'find pair or triplet': two pointers or binary search.
- Contiguous subarray or substring with a condition: sliding window, or prefix sums plus hash map when negatives are allowed.
- 'Minimise the maximum' or 'maximise the minimum': binary search on answer.
- 'K largest', 'K most frequent', 'merge K': heap.
- Next greater or smaller element, span, histogram: monotonic stack.
- Overlapping ranges, meeting rooms, scheduling: sort by start (or end) then sweep.
- Count the ways, min cost, can you reach: DP.
- Generate all combinations, permutations, subsets: backtracking.
- Dependencies, prerequisites, ordering: topological sort.
- Connected groups, 'are X and Y connected', dynamic merging: DSU.
- Shortest path with weights: Dijkstra. Unweighted: BFS.
- Prefix matching, autocomplete, word dictionary: trie.

### Python habits that save time

- Fast input: `import sys; input = sys.stdin.readline`.
- Deep recursion: `sys.setrecursionlimit(10**6)`, or convert to an iterative stack. Recursion on 10^5 depth can still crash.
- `heapq` is a min-heap only. For a max-heap push negatives.
- Never `list.pop(0)` in a loop. Use `collections.deque`.
- Build strings with `''.join(parts)`, not repeated `+=`.
- Use `bisect_left` and `bisect_right` instead of hand-rolled searches when you can.
- `Counter`, `defaultdict`, `deque`, `heapq`, `bisect`, `itertools` and `functools.lru_cache` cover most needs.
- If the OA is in C++ or Java, watch integer overflow: use `long long` or `long` for sums and products.

### Edge-case checklist (run through it before every submit)

- Empty input, a single element, two elements.
- All elements equal, already sorted, reverse sorted.
- Negative numbers and zeros, especially in product and sum problems.
- Duplicates, and whether the problem wants unique results.
- Maximum n with worst-case structure (a chain-shaped tree, a star graph, all identical characters).
- Off-by-one on window and binary-search boundaries.

## Part A continued: DSA patterns, first half

### Time and space complexity

- Count the dominant loop nest. Drop constants and lower-order terms.
- Recursion: write the recurrence. T(n) = 2T(n/2) + n gives n log n. T(n) = T(n-1) + 1 gives n. Two branches with no memoisation gives 2^n.
- Amortised: a dynamic array append is O(1) amortised. A monotonic stack is O(n) overall because each element is pushed and popped once.
- Space counts the recursion stack too. DFS on a skewed tree uses O(n) stack.
- Python specifics: `x in list` is O(n), `x in set` is O(1) average, slicing copies, `sorted` is O(n log n), `heapq.heapify` is O(n).

### Arrays and strings

**Spot it:** anything on a linear structure. Often hides another pattern.

**Moves:** in-place overwrite with a write pointer, reverse via swap, count with an array of 26 or 128, sort then scan, use the array itself as a hash (cyclic sort for values in 1..n).

**Classics:** move zeroes, remove duplicates from sorted array, rotate array (reverse three times), product of array except self, first missing positive, longest common prefix, valid palindrome, group anagrams, string compression, Boyer-Moore majority vote.

**Traps:** mutating while iterating, forgetting that Python strings are immutable, and off-by-one on rotation (`k %= n`).

```python
def rotate(a, k):
    n = len(a); k %= n
    a.reverse(); a[:k] = reversed(a[:k]); a[k:] = reversed(a[k:])
```

### Hashing: HashMap and HashSet

**Spot it:** 'have I seen this before', counting, pairing, grouping, or anything that is O(n^2) because of a lookup.

**Moves:** store value to index for two-sum, store frequency, store canonical form as a key (sorted word, tuple of counts), store first-seen index.

**Classics:** two sum, group anagrams, longest consecutive sequence (start only at numbers with no predecessor), subarray sum equals k, contains duplicate within distance k, isomorphic strings, word pattern, LRU cache.

**Traps:** lists are unhashable (use tuples), and hash maps give no order guarantee beyond insertion order in Python 3.7+.

### Two pointers

**Spot it:** sorted input, pair or triplet sums, in-place partitioning, palindrome checks, merging two sorted sequences.

```python
def three_sum(a):
    a.sort(); res = []
    for i in range(len(a) - 2):
        if i and a[i] == a[i-1]: continue
        l, r = i + 1, len(a) - 1
        while l < r:
            s = a[i] + a[l] + a[r]
            if s < 0: l += 1
            elif s > 0: r -= 1
            else:
                res.append([a[i], a[l], a[r]]); l += 1
                while l < r and a[l] == a[l-1]: l += 1
    return res
```

**Classics:** two sum II, container with most water, 3Sum, trapping rain water, sort colours (Dutch flag), merge sorted arrays, remove element.

**Traps:** skipping duplicates correctly, and moving the right pointer when the left one is the smaller bar in container-type problems.

### Sliding window

**Spot it:** contiguous subarray or substring with a constraint (at most K distinct, sum at least S, no repeats). Fixed-size windows are the easy variant.

**Template:** expand `r` every step, shrink `l` while the window is invalid, update the answer after the window becomes valid.

```python
def longest_unique(s):
    last, l, best = {}, 0, 0
    for r, ch in enumerate(s):
        if ch in last and last[ch] >= l:
            l = last[ch] + 1
        last[ch] = r
        best = max(best, r - l + 1)
    return best
```

**Classics:** longest substring without repeating characters, minimum window substring, max sum subarray of size K, longest repeating character replacement, permutation in string, fruit into baskets, sliding window maximum (with a deque, see below).

**Trick:** 'exactly K' equals 'at most K' minus 'at most K-1'.

**Trap:** sliding window fails when negatives are allowed and you want a sum condition. Use prefix sums with a hash map instead.

### Prefix sums, prefix/suffix techniques, difference arrays

```python
from itertools import accumulate
pre = [0] + list(accumulate(a))        # sum(a[l:r]) = pre[r] - pre[l]

def subarray_sum_k(a, k):
    seen, s, ans = {0: 1}, 0, 0
    for x in a:
        s += x
        ans += seen.get(s - k, 0)
        seen[s] = seen.get(s, 0) + 1
    return ans
```

- Prefix and suffix products give 'product except self' without division.
- Difference array handles many range updates: `d[l] += v; d[r+1] -= v`, then one prefix sum rebuilds the array. Use for 'add v to range \[l, r\]' queries, flight bookings, car pooling.
- 2D prefix sums answer rectangle sum queries in O(1): `P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]`.
- Subarray sum divisible by K: store prefix sum mod K in the hash map.

### Binary search

Pick one template and use it every time. This one finds the first index where the condition is true.

```python
def lower_bound(a, x):          # first index with a[i] >= x
    lo, hi = 0, len(a)
    while lo < hi:
        mid = (lo + hi) // 2
        if a[mid] < x: lo = mid + 1
        else: hi = mid
    return lo
```

**Classics:** search insert position, first and last position of a value, search in rotated sorted array (decide which half is sorted), find peak element, minimum in rotated array, search a 2D matrix, square root.

**Traps:** infinite loop from `lo = mid` with floor division, and overflow of `(lo + hi) / 2` in C++ or Java (`lo + (hi - lo) / 2`).

### Binary search on answer

**Spot it:** 'minimise the largest', 'maximise the smallest', 'least time or capacity such that'. The answer space is monotonic: if X works, anything larger works.

```python
def min_feasible(lo, hi, ok):   # smallest x in [lo, hi] where ok(x) is True
    while lo < hi:
        mid = (lo + hi) // 2
        if ok(mid): hi = mid
        else: lo = mid + 1
    return lo
```

**Recipe:** define the range (lo is the largest single item, hi is the sum), write `ok(x)` as a greedy O(n) check, then run the template. Total O(n log range).

**Classics:** Koko eating bananas, ship packages within D days, split array largest sum, aggressive cows, allocate books, minimum days to make bouquets, smallest divisor given a threshold.

### Fast and slow pointers

- Cycle detection: slow moves 1, fast moves 2. They meet iff there is a cycle.
- Cycle start: after they meet, reset one pointer to head and advance both by 1. They meet at the cycle entrance.
- Middle of a list: when fast reaches the end, slow is at the middle. Also used for 'happy number' and 'find the duplicate number'.

### Linked lists and reversal

```python
def reverse(head):
    prev = None
    while head:
        nxt = head.next
        head.next = prev
        prev, head = head, nxt
    return prev
```

- Use a dummy head node whenever the head might change (remove nth from end, merge two lists, remove elements).
- Remove Nth from end: two pointers with an N-step gap.
- Palindrome list: find middle, reverse the second half, compare.
- Reverse in groups of K: reverse each block, reconnect carefully, draw it on paper first.
- Merge K sorted lists: min-heap of (value, index, node).
- Copy list with random pointer: hash map from old node to new node, or interleave copies.

**Trap:** losing the `next` reference before you overwrite it. Save `nxt` first.

### Stack and monotonic stack

- Plain stack: balanced brackets, evaluate postfix, decode string, simplify path, min stack (store the running minimum with each element).
- Monotonic stack keeps indices with values in sorted order, and each element is popped once, so O(n).

```python
def next_greater(a):
    res, st = [-1] * len(a), []
    for i, x in enumerate(a):
        while st and a[st[-1]] < x:
            res[st.pop()] = x
        st.append(i)
    return res
```

**Classics:** daily temperatures, next greater element I and II (iterate twice for circular), stock span, largest rectangle in histogram (pop while the current bar is shorter, width comes from the new stack top), trapping rain water, remove K digits, sum of subarray minimums.

### Queue and deque

- BFS uses a queue. Level-by-level BFS processes `len(q)` nodes per round.
- Sliding window maximum: deque holding indices with decreasing values.

```python
from collections import deque
def max_window(a, k):
    dq, out = deque(), []
    for i, x in enumerate(a):
        while dq and a[dq[-1]] <= x: dq.pop()
        dq.append(i)
        if dq[0] <= i - k: dq.popleft()
        if i >= k - 1: out.append(a[dq[0]])
    return out
```

- Also: implement queue using two stacks, circular queue, rotting oranges, 0-1 BFS (deque with weights 0 and 1).

### Heap and Top-K

- Python: `heapq.heappush`, `heappop`, `heapify`, `nlargest`. Max-heap by negating values.
- Top K largest: keep a min-heap of size K, so the root is the Kth largest. O(n log k).
- Kth largest overall can also use quickselect in O(n) average.
- Two heaps for a running median: a max-heap for the lower half, a min-heap for the upper half, rebalance so sizes differ by at most 1.

```python
import heapq
from collections import Counter
def top_k_frequent(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]
```

**Classics:** Kth largest element, top K frequent, K closest points, merge K sorted lists, task scheduler, reorganise string, find median from data stream, meeting rooms II (heap of end times), connect ropes (Huffman style).

### Intervals

```python
def merge(iv):
    iv.sort()
    out = [iv[0]]
    for s, e in iv[1:]:
        if s <= out[-1][1]: out[-1][1] = max(out[-1][1], e)
        else: out.append([s, e])
    return out
```

- Insert interval: three phases (before, overlapping, after).
- Non-overlapping intervals (minimum removals): sort by end time, greedily keep the earliest-ending.
- Meeting rooms II: sort starts and ends separately and sweep, or use a min-heap of end times.
- Interval intersections: two pointers, advance the one that ends first.
- Sweep line: turn each interval into +1 at start and -1 at end, sort the events, track the running total (max overlap, skyline).

**Trap:** whether touching intervals like \[1,2\] and \[2,3\] count as overlapping. Read the statement.

### Sorting

- Know merge sort (stable, O(n log n), O(n) space), quick sort (O(n log n) average, O(n^2) worst, in place, unstable), heap sort (O(n log n), in place), counting sort (O(n + k)).
- Python's `sort` is Timsort, stable, and accepts a key: `a.sort(key=lambda x: (-x[1], x[0]))`.
- Quickselect for the Kth element in O(n) average.
- Merge sort modifications count inversions and 'reverse pairs'.
- Custom comparator: largest number is solved with `functools.cmp_to_key` on `a+b` versus `b+a`.
- Sorting is also a preprocessing step that unlocks greedy and two-pointer solutions.

### Matrix problems

```python
DIRS = [(1, 0), (-1, 0), (0, 1), (0, -1)]
def neighbours(r, c, R, C):
    for dr, dc in DIRS:
        nr, nc = r + dr, c + dc
        if 0 <= nr < R and 0 <= nc < C:
            yield nr, nc
```

- Rotate image 90 degrees clockwise: transpose, then reverse each row.
- Spiral order: shrink four boundaries (top, bottom, left, right) and stop when they cross.
- Search a row-and-column sorted matrix: start at the top-right, move left if too big, down if too small. O(R + C).
- Set matrix zeroes: use the first row and column as markers for O(1) space.
- Grid flood fill and island counting are BFS or DFS on a matrix (covered in graphs).
- Treat a flat index as `r * C + c` when you need a single key for DSU or a visited set.

## Part B: DSA patterns, second half

### Trees and BST

```python
class Node:
    def __init__(self, val=0, left=None, right=None):
        self.val, self.left, self.right = val, left, right
```

- Traversals: inorder (left, node, right), preorder, postorder, level order. Inorder of a BST is sorted, which is the key BST fact.
- Validate a BST by passing down `(low, high)` bounds, not by comparing only with the parent.
- BST operations are O(h): O(log n) when balanced, O(n) when skewed. Delete a node with two children by replacing it with its inorder successor.
- Lowest common ancestor in a BST: walk down, split where one value is smaller and the other larger. In a general tree: recurse, return the node if it equals p or q, and the answer is the node where both sides return non-null.
- Know: height, diameter (max of left height + right height over all nodes), balanced check, symmetric tree, same tree, serialize and deserialize, construct tree from preorder and inorder.

### Tree DFS

The usual shape is: base case, recurse on children, combine. Decide whether information flows down (parameters) or up (return value).

```python
def max_path_sum(root):
    best = float('-inf')
    def gain(n):
        nonlocal best
        if not n: return 0
        l, r = max(gain(n.left), 0), max(gain(n.right), 0)
        best = max(best, n.val + l + r)      # path bending at n
        return n.val + max(l, r)             # path extending upward
    gain(root)
    return best
```

**Classics:** path sum I and II, max depth, diameter, binary tree maximum path sum, count good nodes, flatten to linked list, kth smallest in BST (inorder), subtree of another tree.

### Tree BFS and level order

```python
from collections import deque
def level_order(root):
    if not root: return []
    q, out = deque([root]), []
    while q:
        out.append([n.val for n in q])
        for _ in range(len(q)):
            n = q.popleft()
            if n.left: q.append(n.left)
            if n.right: q.append(n.right)
    return out
```

The loop `for _ in range(len(q))` is the 'process one level' trick. Variants: right-side view (last of each level), zigzag, average per level, minimum depth, connect next pointers, vertical order (BFS with column index).

### Graphs: representation, BFS and DFS

```python
from collections import defaultdict, deque
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v); graph[v].append(u)    # drop the second append if directed

def bfs(src):
    dist, q = {src: 0}, deque([src])
    while q:
        u = q.popleft()
        for v in graph[u]:
            if v not in dist:
                dist[v] = dist[u] + 1
                q.append(v)
    return dist

def dfs(u, seen):
    seen.add(u)
    for v in graph[u]:
        if v not in seen: dfs(v, seen)
```

- BFS gives shortest path in an unweighted graph. DFS suits connectivity, cycle detection, path existence, components and backtracking on graphs.
- Grid problems are graphs: number of islands, rotting oranges (multi-source BFS: push every rotten cell first), word ladder (BFS over words), shortest path in a binary matrix, surrounded regions, pacific atlantic water flow (reverse DFS from the borders).
- Clone graph: hash map from old node to its copy.
- Cycle detection: undirected uses the parent check or DSU. Directed uses three colours (white, grey, black) in DFS. A back edge to a grey node means a cycle.
- Bipartite check: two-colour with BFS, a conflict means not bipartite.
- Mark visited when you push, not when you pop, otherwise nodes enter the queue many times.

### Topological sort

**Spot it:** prerequisites, build order, 'can all tasks finish', alien dictionary. Only valid on a DAG.

```python
def topo(n, edges):                 # Kahn's algorithm
    adj, indeg = [[] for _ in range(n)], [0] * n
    for u, v in edges:
        adj[u].append(v); indeg[v] += 1
    q = deque(i for i in range(n) if indeg[i] == 0)
    order = []
    while q:
        u = q.popleft(); order.append(u)
        for v in adj[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return order if len(order) == n else []     # shorter means a cycle
```

The DFS version pushes a node onto a list after all its children finish, then reverses the list. Both are O(V + E). Course schedule I and II, alien dictionary and parallel courses are the standard set.

### Union Find (DSU)

```python
class DSU:
    def __init__(self, n):
        self.p = list(range(n)); self.sz = [1] * n; self.comps = n
    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]       # path compression (halving)
            x = self.p[x]
        return x
    def union(self, a, b):
        a, b = self.find(a), self.find(b)
        if a == b: return False
        if self.sz[a] < self.sz[b]: a, b = b, a
        self.p[b] = a; self.sz[a] += self.sz[b]; self.comps -= 1
        return True
```

Path compression plus union by size gives near O(1) amortised, formally O(alpha(n)). **Classics:** number of provinces, redundant connection (the edge whose union returns False), accounts merge, number of islands II, Kruskal's MST, graph valid tree, satisfiability of equality equations.

### Dijkstra, MST, Bellman-Ford, Floyd-Warshall, SCC

```python
import heapq
def dijkstra(adj, src, n):
    dist = [float('inf')] * n; dist[src] = 0
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue                 # stale entry
        for v, w in adj[u]:
            if d + w < dist[v]:
                dist[v] = d + w
                heapq.heappush(pq, (dist[v], v))
    return dist
```

- **Dijkstra:** non-negative weights only. O((V + E) log V). Network delay time, cheapest flights within K stops (use Bellman-Ford style or state (node, stops)), path with minimum effort, swim in rising water.
- **Bellman-Ford:** relax all edges V-1 times. O(VE). Handles negative edges. A relaxation in the V-th round signals a negative cycle.
- **Floyd-Warshall:** all pairs in O(V^3). Loop order is `k` outermost: `d[i][j] = min(d[i][j], d[i][k] + d[k][j])`. Fine for V up to about 400.
- **MST (Kruskal):** sort edges, add with DSU if endpoints are in different sets. O(E log E). **Prim:** grow from a node with a min-heap of crossing edges. Min cost to connect points and connecting cities with minimum cost are typical.
- **SCC:** Kosaraju (DFS for finish order, DFS on the reversed graph in that order) or Tarjan (low-link values). Condense SCCs into a DAG for problems about cycles and reachability. Rarely asked in an OA, but expect it as a theory question.

### Backtracking and recursion

Template: choose, explore, un-choose.

```python
def subsets(nums):
    res, path = [], []
    def go(i):
        if i == len(nums):
            res.append(path[:]); return
        path.append(nums[i]); go(i + 1)       # take
        path.pop(); go(i + 1)                 # skip
    go(0)
    return res
```

- Permutations: a `used` array or swapping. Combinations and combination sum: pass a start index so you never reuse earlier elements.
- Duplicates in the input: sort first and skip `if i > start and a[i] == a[i-1]` at the same recursion level.
- Prune early. N-Queens (track columns and both diagonals in sets), Sudoku, word search, palindrome partitioning, generate parentheses (count opens and closes).
- Recursion basics: always have a base case that is reachable, know the call-stack depth, and memoise when subproblems repeat. That is exactly where backtracking turns into DP.

### Divide and conquer

Split, solve halves, combine. Merge sort, quicksort, binary search, count inversions, maximum subarray (divide version), closest pair of points, fast exponentiation.

```python
def power(x, n):
    if n == 0: return 1
    half = power(x, n // 2)
    return half * half * (x if n % 2 else 1)
```

Master theorem shortcut for T(n) = aT(n/b) + f(n): compare f(n) with n^(log\_b a). Equal gives an extra log factor, bigger f dominates, smaller f means the leaves dominate.

### Greedy

A locally best choice that stays safe globally. Always ask why it is safe (an exchange argument), and try a small counter-example before you commit.

**Classics:** activity selection (sort by end time), jump game I and II, gas station, candy, assign cookies, task scheduler, minimum platforms, fractional knapsack, Huffman coding, lemonade change, partition labels, minimum number of arrows to burst balloons.

**Warning:** coin change with arbitrary denominations is not greedy, it is DP. 0/1 knapsack is DP, fractional knapsack is greedy.

### Dynamic programming: 1D and 2D

**Method:** (1) define the state in words, (2) write the transition, (3) fix the base cases, (4) choose the order, (5) shrink the space if the transition only looks back a row or two. Start with a recursive solution plus memo, then convert to a table.

```python
from functools import lru_cache
def climb(n):                            # 1D, constant space
    a, b = 1, 1
    for _ in range(n - 1): a, b = b, a + b
    return b

def coin_change(coins, amount):          # unbounded, min coins
    INF = amount + 1
    dp = [0] + [INF] * amount
    for a in range(1, amount + 1):
        for c in coins:
            if c <= a: dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] < INF else -1
```

- **1D classics:** climbing stairs, house robber I and II, coin change I and II (count ways: loop coins outside, amount inside), decode ways, word break, maximum product subarray, jump game.
- **2D classics on grids:** unique paths, minimum path sum, maximal square, dungeon game.
- **2D on two strings:** edit distance, LCS, distinct subsequences, regex and wildcard matching.
- **Space reduction:** if `dp[i][j]` needs only row `i-1`, keep two rows or one row (iterate backwards for 0/1 knapsack).

### Advanced DP: LIS, LCS, knapsack, interval, bitmask

```python
from bisect import bisect_left
def lis(a):                               # O(n log n)
    tails = []
    for x in a:
        i = bisect_left(tails, x)
        if i == len(tails): tails.append(x)
        else: tails[i] = x
    return len(tails)

def lcs(a, b):
    dp = [[0] * (len(b) + 1) for _ in range(len(a) + 1)]
    for i in range(1, len(a) + 1):
        for j in range(1, len(b) + 1):
            dp[i][j] = dp[i-1][j-1] + 1 if a[i-1] == b[j-1] else max(dp[i-1][j], dp[i][j-1])
    return dp[-1][-1]

def knap01(wt, val, cap):
    dp = [0] * (cap + 1)
    for w, v in zip(wt, val):
        for c in range(cap, w - 1, -1):    # backwards, so each item is used once
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]
```

- LIS: the O(n^2) version is `dp[i] = 1 + max(dp[j])` for `j < i` with `a[j] < a[i]`. Strictly increasing uses `bisect_left`, non-decreasing uses `bisect_right`.
- Knapsack family: subset sum, partition equal subset sum, target sum, last stone weight II, coin change. Recognise 'pick items to hit a capacity' and it is probably this.
- Interval DP: `dp[i][j]` over substrings or ranges, iterate by length. Matrix chain multiplication, burst balloons, palindromic substrings, minimum cost to cut a stick. O(n^3).
- Bitmask DP: `dp[mask][last]` for up to about 20 items. Travelling salesman, assign tasks to workers, partition into K equal subsets.
- Other: DP on trees (return two states per node, rob or skip), DP on stocks (states hold, sold, cooldown), digit DP (rare).

### Trie

```python
class Trie:
    def __init__(self):
        self.root = {}
    def insert(self, w):
        n = self.root
        for ch in w: n = n.setdefault(ch, {})
        n['$'] = True
    def search(self, w, prefix=False):
        n = self.root
        for ch in w:
            if ch not in n: return False
            n = n[ch]
        return prefix or '$' in n
```

Insert and search are O(length of word). Uses: autocomplete, word search II (DFS on the grid guided by the trie), longest common prefix, add and search word with `.` wildcard, maximum XOR of two numbers (binary trie). Trade-off against a hash set: tries cost more memory but support prefix queries.

### Bit manipulation

- `x & 1` odd check, `x >> 1` divide by 2, `1 << k` power of two, `x & (x - 1)` clears the lowest set bit, `x & -x` isolates it.
- Power of two: `x > 0 and x & (x - 1) == 0`. Count set bits: loop with `x &= x - 1`, or `bin(x).count('1')`.
- XOR facts: `a ^ a = 0`, `a ^ 0 = a`, commutative. So single number is XOR of everything. Missing number is XOR of indices and values. Swap without a temp: `a ^= b; b ^= a; a ^= b`.
- Subsets of a set with n items: loop `mask` from 0 to `2**n - 1` and test bit `i` with `mask >> i & 1`.
- Single number III, counting bits (`dp[i] = dp[i >> 1] + (i & 1)`), reverse bits, sum of two integers without `+`.
- In Python integers are unbounded. Negative numbers need a mask like `& 0xFFFFFFFF` when the problem assumes 32-bit.

### String algorithms: KMP and Z-function

```python
def prefix_function(s):                   # KMP failure table
    pi = [0] * len(s)
    for i in range(1, len(s)):
        k = pi[i - 1]
        while k and s[i] != s[k]: k = pi[k - 1]
        if s[i] == s[k]: k += 1
        pi[i] = k
    return pi
```

- `pi[i]` is the length of the longest proper prefix of `s[:i+1]` that is also its suffix. Pattern search: run it on `pattern + '#' + text`, a value equal to `len(pattern)` marks a match. O(n + m).
- Z-function: `z[i]` is the length of the longest common prefix of `s` and `s[i:]`. Same use cases, often easier to reason about.
- Related: Rabin-Karp rolling hash (watch collisions, use a large modulus), shortest palindrome, repeated substring pattern (`s in (s + s)[1:-1]`).

### Kadane's algorithm

```python
def max_subarray(a):
    cur = best = a[0]
    for x in a[1:]:
        cur = max(x, cur + x)
        best = max(best, cur)
    return best
```

Extensions: return the indices (reset the start when `cur` restarts at `x`), circular array (max of Kadane and total minus the minimum subarray, guard the all-negative case), maximum product subarray (track both max and min), best time to buy and sell stock (Kadane on daily differences).

## Part C: SQL for OAs

Sample tables used below: `employees(id, name, dept_id, salary, manager_id, hired_on)`, `departments(id, name)`, `orders(id, user_id, amount, created_at)`, `logins(user_id, login_date)`.

### Basic clauses

- Logical execution order: FROM, JOIN, WHERE, GROUP BY, HAVING, SELECT, DISTINCT, ORDER BY, LIMIT. This is why you cannot use a SELECT alias in WHERE but can in ORDER BY.
- `WHERE` filters rows before grouping, `HAVING` filters groups after aggregation.
- `LIMIT n OFFSET m` (MySQL, PostgreSQL), `TOP n` or `FETCH FIRST` (SQL Server, Oracle).
- `NULL` is never equal to anything, including itself. Use `IS NULL`, `COALESCE(x, 0)`. `COUNT(*)` counts rows, `COUNT(col)` skips NULLs.

```sql
SELECT dept_id, COUNT(*) AS headcount, AVG(salary) AS avg_sal
FROM employees
WHERE salary > 30000
GROUP BY dept_id
HAVING COUNT(*) >= 5
ORDER BY avg_sal DESC
LIMIT 3;
```

### DML and DDL

- `INSERT INTO t (a, b) VALUES (1, 2)`, `UPDATE t SET a = 1 WHERE ...`, `DELETE FROM t WHERE ...`. Forgetting WHERE on UPDATE or DELETE touches every row.
- `CREATE TABLE` with constraints (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `CHECK`, `DEFAULT`), `ALTER TABLE ... ADD COLUMN`, `DROP TABLE`.
- **DELETE vs TRUNCATE vs DROP:** DELETE removes rows (DML, can use WHERE, logged per row, can roll back in most engines, fires triggers). TRUNCATE removes all rows fast (DDL, resets identity, no WHERE, rollback depends on the engine). DROP removes the table itself, structure and data.

### Joins

- `INNER JOIN`: matching rows only. `LEFT JOIN`: all left rows, NULLs on the right when nothing matches. `RIGHT JOIN`: mirror image. `FULL JOIN`: everything from both sides. `CROSS JOIN`: every pairing.
- Filtering the right table of a LEFT JOIN in WHERE turns it into an inner join. Put that condition in `ON` instead.
- Anti-join (rows with no match): `LEFT JOIN ... WHERE right.id IS NULL`, or `NOT EXISTS`.
- Self join, employees with their managers:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

- Multi-column join: `ON a.x = b.x AND a.y = b.y`. NULL keys never match with `=`, so use `IS NOT DISTINCT FROM` (PostgreSQL) or handle NULLs explicitly.

### Subqueries, CTEs, set operations

```sql
-- employees earning more than their department's average (correlated)
SELECT name FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);

-- same with a CTE
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS a FROM employees GROUP BY dept_id
)
SELECT e.name FROM employees e JOIN dept_avg d USING (dept_id) WHERE e.salary > d.a;
```

- A correlated subquery references the outer query and runs conceptually once per outer row. Often replaceable by a join or window function.
- CTEs make queries readable, can be chained, and recursive CTEs walk hierarchies (org charts, category trees).
- `UNION` removes duplicates (sorts or hashes, slower), `UNION ALL` keeps them (faster). Also `INTERSECT` and `EXCEPT`.
- `EXISTS` stops at the first match and handles NULLs safely. `NOT IN` returns nothing if the subquery contains a NULL, so prefer `NOT EXISTS`.

### Aggregates and CASE

```sql
SELECT dept_id,
       SUM(CASE WHEN salary >= 50000 THEN 1 ELSE 0 END) AS high_paid,
       COUNT(*) AS total
FROM employees GROUP BY dept_id;
```

Conditional aggregation (`SUM(CASE ...)`) is how you pivot and compute several filtered counts in one pass.

### Window functions

```sql
SELECT name, dept_id, salary,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
  RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drnk,
  LAG(salary)  OVER (PARTITION BY dept_id ORDER BY hired_on)   AS prev_sal,
  SUM(salary)  OVER (PARTITION BY dept_id ORDER BY hired_on)   AS running_total
FROM employees;
```

- With ties 100, 100, 90: `ROW_NUMBER` gives 1, 2, 3. `RANK` gives 1, 1, 3. `DENSE_RANK` gives 1, 1, 2.
- `LAG` looks at the previous row, `LEAD` at the next one. Useful for day-over-day change.
- Aggregates with `OVER` keep every row. Add a frame like `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` for moving averages.
- You cannot filter on a window function in WHERE. Wrap it in a subquery or CTE.

### The standard interview queries

```sql
-- Nth highest salary (N = 2)
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;
-- or robust against ties and short tables:
SELECT MAX(salary) FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS r FROM employees
) t WHERE r = 2;

-- Top 3 earners per department
SELECT * FROM (
  SELECT *, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS r FROM employees
) t WHERE r <= 3;

-- Duplicate rows
SELECT name, COUNT(*) FROM employees GROUP BY name HAVING COUNT(*) > 1;
-- Delete duplicates, keep the lowest id
DELETE FROM employees WHERE id NOT IN (
  SELECT MIN(id) FROM employees GROUP BY name
);

-- Consecutive login days (gaps and islands)
SELECT user_id, MIN(login_date) AS start_d, MAX(login_date) AS end_d, COUNT(*) AS streak
FROM (
  SELECT user_id, login_date,
         login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day' AS grp
  FROM (SELECT DISTINCT user_id, login_date FROM logins) d
) t
GROUP BY user_id, grp;

-- Day-1 or next-day retention
SELECT COUNT(DISTINCT b.user_id) * 1.0 / COUNT(DISTINCT a.user_id)
FROM logins a LEFT JOIN logins b
  ON a.user_id = b.user_id AND b.login_date = a.login_date + 1;

-- Employees earning more than their manager
SELECT e.name FROM employees e JOIN employees m ON e.manager_id = m.id WHERE e.salary > m.salary;

-- Customers who never ordered
SELECT u.id FROM users u WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

**Pagination:** offset pagination (`LIMIT 20 OFFSET 4000`) scans and discards 4000 rows and drifts when rows are inserted. Keyset pagination (`WHERE id > :last_id ORDER BY id LIMIT 20`) uses an index and stays fast on deep pages. Mention this trade-off when asked.

### Design and performance, OA-level

- Keys: primary key (unique, not null), foreign key (referential integrity, with `ON DELETE CASCADE` or `RESTRICT`), unique, composite keys.
- Normalisation in one line each: 1NF atomic values, 2NF no partial dependency on part of a composite key, 3NF no transitive dependency, BCNF every determinant is a candidate key. Denormalise on purpose for read-heavy paths.
- Views are saved queries (no storage unless materialised). Stored procedures are saved routines with logic and parameters.
- Transactions: `BEGIN; ... COMMIT;` or `ROLLBACK;`. Deeper theory is in the technical-round notes.
- Indexes speed reads and slow writes. Clustered index defines physical row order (one per table, the primary key in InnoDB), non-clustered is a separate structure pointing to rows. A composite index `(a, b)` helps filters on `a` or on `a, b`, not on `b` alone (leftmost-prefix rule).
- Wrapping an indexed column in a function (`WHERE YEAR(created_at) = 2026`) defeats the index. Use a range instead.
- `EXPLAIN` (or `EXPLAIN ANALYZE` in PostgreSQL) shows the plan. Look for sequential scans on big tables, nested loops over large inputs, and sorts that spill.
- Practise target from your checklist: 50 queries. A reasonable split is 15 joins and subqueries, 15 window-function problems, 10 aggregation and CASE, 10 gaps-and-islands, retention and date problems.

## Part C continued: Aptitude

Many service and product company OAs include a 30 to 45 minute aptitude block. Speed matters more than depth, so memorise the formulas and skip anything that takes over 90 seconds.

### Quantitative

- **Percentages:** increase by a% then decrease by a% gives net change of -a^2/100 percent. Successive changes of x% and y%: x + y + xy/100.
- **Profit and loss:** profit % = (SP - CP) / CP \* 100. Marked price and discount: SP = MP \* (1 - d). Two successive discounts a and b: a + b - ab/100.
- **Ratio and proportion, mixtures, alligation:** mix quantities at prices p1 and p2 to get a mean price m: ratio is (p2 - m) : (m - p1).
- **Averages:** new average after adding a value, weighted average, replacing one item.
- **Time, speed, distance:** km/h to m/s multiply by 5/18. Average speed for equal distances at a and b is 2ab/(a + b). Relative speed adds for opposite directions, subtracts for the same. Train crossing a pole is length / speed, crossing a platform is (length + platform) / speed. Boats: downstream = u + v, upstream = u - v.
- **Time and work:** rate = 1/time. Combined rate is the sum of rates. For pipes, a leak is a negative rate. Man-days: M1 D1 / W1 = M2 D2 / W2.
- **Simple and compound interest:** SI = PRT/100. CI amount = P(1 + r/100)^n. Difference between CI and SI for two years is P(r/100)^2.
- **Number system:** divisibility rules, unit digit cycles (powers of 2, 3, 7, 8 cycle every 4), remainders, LCM and HCF (product of two numbers equals LCM times HCF), number of factors from prime factorisation.
- **Permutations and combinations:** nPr = n!/(n-r)!, nCr = n!/(r!(n-r)!), circular arrangements (n-1)!, repeated letters divide by factorials. Handle 'at least one' by complement.
- **Probability:** favourable over total, P(A or B) = P(A) + P(B) - P(A and B), independent events multiply, conditional probability P(A|B) = P(A and B) / P(B).
- **Geometry and mensuration:** areas and volumes of standard shapes, Pythagorean triples (3-4-5, 5-12-13, 8-15-17).
- **Data interpretation:** read the axis and units first, estimate instead of computing exactly.

### Logical reasoning

- **Series and patterns:** check differences, ratios, alternating series, squares and cubes, primes, and Fibonacci-like sums.
- **Coding-decoding:** positional shifts, reversed alphabets (A + Z = 27 in position terms), letter-number mapping.
- **Blood relations:** draw a family tree with gender marks. Resolve one step at a time.
- **Direction sense:** draw it on paper, and track net displacement for the final distance.
- **Seating arrangement:** circular or linear, fill in definite clues first, then case-split.
- **Syllogisms:** Venn diagrams, and 'some' can be false in a 'possible' conclusion question, so test the minimal diagram.
- **Puzzles and Sudoku-style grids:** a table of attributes by rows and eliminate.

### Verbal (if present)

Reading comprehension (skim the questions before the passage), sentence correction (subject-verb agreement, tense, parallelism), fill in the blanks, synonyms and antonyms, para jumbles (find the opening sentence and the pronoun links).

## Practice plan from your checklist

- **250+ DSA problems, grouped by pattern.** Roughly 20 each for arrays and hashing, two pointers and sliding window, binary search, stack and heap, intervals, trees, graphs, DP and backtracking. Fewer on tries, bits and string algorithms.
- **Solve, then re-solve.** Attempt for 25 to 30 minutes without hints. If stuck, read only the pattern name, try again, then read the solution and re-code it from scratch the next day. Redo the problem a week later.
- **For each problem, say out loud** the brute force, why it is slow, the optimisation, correctness, and complexity. This is the exact skill the technical round tests.
- **10 timed OAs.** Use a real mock timer, 90 minutes, three problems, no hints. After each one, log what failed: misread, wrong pattern, bug, or time.
- **Keep a mistake log** with the pattern, the cause, and the fix. Review it before each OA.
- **One language.** Python is fine for most OAs. If a company requires C++ or Java, practise the STL or collections basics so syntax never costs you time.
