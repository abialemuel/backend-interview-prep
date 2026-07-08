# Common DSA Patterns for Interviews

Most interview problems are not novel. They are a recognizable pattern plus a small twist. Pattern recognition — knowing "when I see X, reach for Y" — is the highest-leverage skill you can build. This file walks through the patterns that come up most often in backend interviews, with how to recognize each, the core idea, the time/space complexity, and one canonical problem.

## Big-O, Big-Theta, Big-Omega

- **Big-O** is an **asymptotic upper bound** — the worst-case growth rate of runtime (or memory) as input grows. `f(n) = O(g(n))` means `f` is eventually bounded above by a constant multiple of `g`. In interviews, "what's the time complexity?" means "what's the Big-O?"
- **Big-Omega** is the symmetric lower bound: `f(n) = Ω(g(n))` means `f` is at least `g` asymptotically. Rarely asked.
- **Big-Theta** is a tight bound: `f(n) = Θ(g(n))` means it is both `O(g(n))` and `Ω(g(n))` — `f` grows exactly like `g` up to constants.

Interview practice: state Big-O (the upper bound) cleanly. If your worst case is `O(n)` and best case is `O(1)`, say "worst case O(n), best case O(1)"; don't bother with Theta unless the interviewer asks. Tight bounds matter when you can prove a lower bound (e.g., comparison sort is `Ω(n log n)`), which is rarely required.

A common subtlety: **amortized** vs worst-case. A Go slice append is `O(1)` amortized but `O(n)` worst case on the realloc step. For a hash map, most operations are `O(1)` amortized average and `O(n)` worst case. Say "amortized O(1)" to be precise; many interviewers consider that the correct answer for hash tables.

## Time vs space trade-off

The most common interview move is trading memory for time. Classic example: **Two Sum**. Brute force is `O(n^2)` time, `O(1)` space. Storing seen values in a hash map makes it `O(n)` time, `O(n)` space. You nearly always pay space to win time. Be explicit: when you propose the hash map solution, say "I'm spending O(n) space to bring time from O(n^2) down to O(n)."

The reverse — trading time for space — happens when memory is the constraint: e.g. computing a checksum vs storing a copy, recomputing values vs memoizing, or streaming a sort vs loading all data (external merge sort). For backend roles, also discuss the **constant factors and I/O cost**: an O(n) algorithm that does sequential disk reads beats an O(log n) algorithm that does random I/O on spinning disk.

---

## The patterns

### 1. Two pointers / fast-slow pointers

**Recognize it by:** "sorted/in-place", "remove duplicates", "in-place array manipulation", "find a pair/triplet sum", "detect a cycle", "find middle / kth from end" of a linked list. Two indices (or two pointers in a list) moving through the data.

**Core idea:** Two pointers move through the structure (often in one direction) such that each step makes a decisive comparison. Variants:
- Opposite ends converging (sorted two-sum, container with most water).
- Same direction, one fast one slow (remove duplicates, dedupe in place).
- Fast & slow (Floyd's cycle detection).

**Complexity:** Usually O(n) time, O(1) space.

**Canonical problem:** *Valid Palindrome* (or *Two Sum II - Input Array Is Sorted*). For fast-slow variant: *Linked List Cycle*.

```go
func isPalindrome(s []byte) bool {
    l, r := 0, len(s)-1
    for l < r {
        for l < r && !isAlnum(s[l]) { l++ }
        for l < r && !isAlnum(s[r]) { r-- }
        if toLower(s[l]) != toLower(s[r]) {
            return false
        }
        l++; r--
    }
    return true
}
```

### 2. Sliding window

**Recognize it by:** "subarray / substring of size K", "longest/shortest with property P", "maximum sum of a subarray of size K", "contains all of" / "at most K distinct". Input is an array or string and you optimize over a contiguous window.

**Core idea:** Maintain a window `[left, right]` and slide it across, adding at `right` and shrinking from `left` to keep an invariant (e.g. at most K distinct chars). The window is "expanded" on every step and "contracted" only when the invariant breaks, so each element is added and removed at most once.

**Complexity:** O(n) time, O(k) space for the window contents (or O(1) for fixed counts).

**Canonical problem:** *Longest Substring Without Repeating Characters*; also *Minimum Window Substring*.

```go
func lengthOfLongestSubstring(s string) int {
    last := map[byte]int{}
    left, best := 0, 0
    for right := 0; right < len(s); right++ {
        if i, ok := last[s[right]]; ok && i >= left {
            left = i + 1
        }
        last[s[right]] = right
        if right-left+1 > best {
            best = right - left + 1
        }
    }
    return best
}
```

### 3. Binary search (and binary search on answer)

**Recognize it by:** "**sorted**" array/matrix, "find first position where condition holds", "log n", "search in rotated sorted array", "split array / minimize the largest sum", "koko eating bananas", "minimum time to achieve X".

**Core idea:** Repeatedly halve the search space by comparing the middle to a target. The general form maintains `[lo, hi]` and an `ans` candidate, narrowing based on `predicate(mid)`:

1. Maintain `lo, hi` bounding the answer space.
2. `mid = lo + (hi-lo)/2`.
3. If `predicate(mid)` is true, record `ans = mid` and move `hi` down; else move `lo` up.
4. Return `ans`.

**Binary search on the answer:** when you can phrase the problem as "is it possible to achieve value X?" being monotone in X, search over X. Classic for *split array largest sum*, *koko eating bananas*, *capacity to ship packages*. Backend flavor: "what's the min cluster size to keep p99 under X" falls in the same pattern when feasibility is monotone.

**Complexity:** O(log n) for plain search; O(log(range) · n) for search-on-answer (the feasibility check dominates).

**Canonical problem:** *Binary Search*; search-on-answer: *Split Array Largest Sum*.

```go
func search(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        switch {
        case nums[mid] == target:
            return mid
        case nums[mid] < target:
            lo = mid + 1
        default:
            hi = mid - 1
        }
    }
    return -1
}
```

### 4. BFS / DFS (trees and graphs)

**Recognize it by:** "level order", "shortest path in unweighted graph", "all paths", "number of islands / connected components", "flood fill", "clone a graph", "binary tree traversal".

**Core idea:** Two ways to traverse. BFS uses a queue, explores level by level, naturally finds shortest path in unweighted graphs. DFS uses the call stack (or explicit stack), explores one branch fully. Both cost O(V+E) time and O(V) space (or O(height) for tree DFS).

**Complexity:** O(V+E) time, O(V) auxiliary space. (BFS queue or DFS stack/recursion.)

**Canonical problems:** *Binary Tree Level Order Traversal* (BFS), *Number of Islands* (DFS/BFS over a grid), *Rotting Oranges* (multi-source BFS with a distance counter).

Multi-source BFS pattern: push all initial "sources" into the queue at once, BFS outward; the time-step when an element is popped equals its distance from the nearest source. This is the trick for *Rotting Oranges* and *Walls and Gates*.

### 5. Backtracking

**Recognize it by:** "all combinations / permutations / subsets", "generate" + small n, "solve the puzzle" (n-queens, sudoku, word search), "any valid sequence of choices". When you need to enumerate a search tree of discrete choices.

**Core idea:** Recursive DFS over a decision tree. At each node, try every legal choice, recurse, then **undo** the choice (backtrack). The key skill is identifying (a) the choices at each step, (b) the constraints to prune branches early, and (c) the base case for recording a result.

**Complexity:** Exponential in n in the worst case, but pruned properly it stays tractable for small n (typical n <= 20). O(branching^depth) time, O(depth) recursion space.

**Canonical problem:** *Subsets* / *Permutations* / *Letter Combinations of a Phone Number*. Constraint-heavy: *N-Queens*, *Word Search*.

```go
func permute(nums []int) [][]int {
    var res [][]int
    var bt func(int)
    bt = func(start int) {
        if start == len(nums) {
            cp := append([]int(nil), nums...)
            res = append(res, cp)
            return
        }
        for i := start; i < len(nums); i++ {
            nums[start], nums[i] = nums[i], nums[start]
            bt(start + 1)
            nums[start], nums[i] = nums[i], nums[start]
        }
    }
    bt(0)
    return res
}
```

### 6. Topological sort — backend relevant

**Recognize it by:** "dependencies", "prerequisites", "build order", "package compilation order", "task scheduling with ordering", "course schedule", "alien dictionary ordering". Any time "X must happen before Y" and you want a feasible order, or to detect a cycle that makes none exist.

**Core idea:** Use Kahn's algorithm — count in-degree of every node, enqueue the zero-in-degree nodes, pop one, decrement neighbors' in-degrees, enqueue new zero-in-degree ones. The pop order is the topological order; if not all nodes are processed, there's a cycle.

**Complexity:** O(V+E) time and space.

**Backend relevance:** Dependency resolution (packages, build steps, migrations, service startup ordering, CI job ordering, k8s init containers / prestarts) is literally topological sort. Interviewers love "Course Schedule" because it maps directly to "can these services / migrations start in a valid order?".

**Canonical problem:** *Course Schedule* (and *Course Schedule II* for the actual ordering); also *Alien Dictionary*.

```go
func canFinish(n int, edges [][]int) bool {
    g := make([][]int, n)
    indeg := make([]int, n)
    for _, e := range edges {
        g[e[1]] = append(g[e[1]], e[0])
        indeg[e[0]]++
    }
    q := []int{}
    for i := 0; i < n; i++ {
        if indeg[i] == 0 { q = append(q, i) }
    }
    done := 0
    for len(q) > 0 {
        v := q[0]; q = q[1:]
        done++
        for _, u := range g[v] {
            indeg[u]--
            if indeg[u] == 0 { q = append(q, u) }
        }
    }
    return done == n
}
```

### 7. Hash map for O(n) lookups / frequency counting

**Recognize it by:** "two pass", "have we seen", "count occurrences", "anagram", "first non-repeating", "find the duplicate", "group by". Whenever you find yourself writing "for each element, scan the rest," ask "can I memoize what I've seen?"

**Core idea:** Trade O(n) space for O(1) lookup. Two passes: first builds the map (counts / positions / seen), second uses it. Sometimes one pass is enough (look up the complement as you go).

**Complexity:** O(n) time, O(n) space average.

**Canonical problem:** *Two Sum*, *Valid Anagram*, *Group Anagrams*, *Top K Frequent Elements* (combined with a heap).

```php
function twoSum(array $nums, int $target): array {
    $seen = [];
    foreach ($nums as $i => $x) {
        $need = $target - $x;
        if (isset($seen[$need])) {
            return [$seen[$need], $i];
        }
        $seen[$x] = $i;
    }
    return [];
}
```

### 8. Heap / priority queue

**Recognize it by:** "kth largest / smallest", "top K", "merge K sorted lists", "streaming median", "smallest range covering K lists", "schedule tasks with cooldown". Any problem about maintaining the K extreme elements under a stream, or repeatedly extracting the min/max.

**Core idea:** Use a binary heap (min or max) for O(log n) insertion and O(1) peek of the min/max. For "top K" you keep a *min-heap of size K* so the smallest of the K-best is at the top and gets evicted when something larger arrives. For streaming median, keep two heaps: a max-heap for the lower half and a min-heap for the upper half, balanced.

**Complexity:** O(n log k) time and O(k) space for top-K. O(log n) per op on a full heap.

**Canonical problem:** *Kth Largest Element in an Array*, *Top K Frequent Elements*, *Find Median from Data Stream*, *Merge K Sorted Lists*, *Design Twitter*.

```go
import "container/heap"

type MaxHeap []int
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h MaxHeap) Len() int           { return len(h) }
func (h *MaxHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() any {
    x := (*h)[len(*h)-1]
    *h = (*h)[:len(*h)-1]
    return x
}

func findKthLargest(nums []int, k int) int {
    h := &MaxHeap{}
    for _, x := range nums { heap.Push(h, x) }
    for i := 0; i < k-1; i++ { heap.Pop(h) }
    return heap.Pop(h).(int)
}
```

### 9. Union-Find / Disjoint Set Union

**Recognize it by:** "connected components", "are A and B in the same set", "number of connected components", "MST", "redundant connection", "accounts merge", "when does the graph become connected". When the operation is repeatedly unioning sets and querying "are these in the same set."

**Core idea:** Maintain disjoint sets with two arrays: `parent[]` and `rank[]`. `find(x)` walks to the root (with **path compression**, flattening); `union(a,b)` attaches the smaller-rank root under the larger one (**union by rank**). With both optimizations every op is amortized `O(α(n))`, effectively `O(1)`.

**Complexity:** O(α(n)) per operation, where α is inverse Ackermann. O(n) space.

**Canonical problem:** *Number of Connected Components in an Undirected Graph*, *Redundant Connection*, *Accounts Merge*, *Kruskal's MST*.

```go
type DSU struct{ parent, rank []int }
func NewDSU(n int) *DSU {
    p, r := make([]int, n), make([]int, n)
    for i := range p { p[i] = i }
    return &DSU{p, r}
}
func (d *DSU) find(x int) int {
    for d.parent[x] != x {
        d.parent[x] = d.parent[d.parent[x]]
        x = d.parent[x]
    }
    return x
}
func (d *DSU) union(a, b int) bool {
    ra, rb := d.find(a), d.find(b)
    if ra == rb { return false }
    switch {
    case d.rank[ra] < d.rank[rb]: d.parent[ra] = rb
    case d.rank[ra] > d.rank[rb]: d.parent[rb] = ra
    default: d.parent[rb] = ra; d.rank[ra]++
    }
    return true
}
```

### 10. Dynamic programming

**Recognize it by:** "number of ways", "minimum/maximum cost to reach", "is it possible given choices", "longest subsequence/substring with property", overlapping subproblems with a small state. The clue is **the same subproblem is solved many times** — that's the engine DP exploits.

**Core idea:** Define `dp(state)` = optimal value of the subproblem, write a recurrence relating `dp(state)` to smaller subproblems, and either memoize top-down or fill a table bottom-up. The hard part is identifying the state and the recurrence. Common shapes:
- **1D DP:** Fibonacci-like. `dp[i]` from `dp[i-1], dp[i-2]` (*Climbing Stairs*, *House Robber*).
- **2D DP:** grids or two sequences. `dp[i][j]` (*Unique Paths*, *Longest Common Subsequence*, *Edit Distance*).
- **Knapsack (0/1 and unbounded):** `dp[i][capacity]` decision take-or-skip per item (*Coin Change*, *Partition Equal Subset Sum*).
- **LIS (Longest Increasing Subsequence):** `dp[i]` = length of LIS ending at i (O(n^2)) or patience sort (O(n log n)).
- **LCS (Longest Common Subsequence):** `dp[i][j]` comparing two strings, classic edit-distance transfer.

Recognizing overlap is the trick: if a naive recursive solution recomputes the same substates (e.g., `fib(n-1)` and `fib(n-2)` both compute `fib(n-3)`), memoize or invert to bottom-up.

**Complexity:** O(states · transition-cost) time, O(states) space (often compressible to one row).

**Canonical problems:** *Climbing Stairs* (1D), *Coin Change* (unbounded knapsack), *Longest Increasing Subsequence*, *Longest Palindromic Substring* (interval or reverse-string LCS), *Longest Common Subsequence*.

Keep DP brief in an interview: state definition, recurrence, base, fill order. That's enough for the interviewer to follow.

### 11. Prefix sum / difference arrays

**Recognize it by:** "sum of elements in range [l,r]", "frequent range queries, no updates" (prefix), "range add 'increment by k over [l, r]', then read final array" (difference array), "matrix block sum".

**Core idea:** Build `prefix[i] = sum of arr[0..i-1]`; then `sum(arr[l..r]) = prefix[r+1] - prefix[l]`, O(1) per query after O(n) preprocessing. Difference array is the inverse — for batch range adds you bump `diff[l] += k; diff[r+1] -= k`, then prefix-sum the diff to recover the final array. Both turn O(n) per query/update into O(1).

**Complexity:** O(n) preprocessing + O(1) per query/update. O(n) space.

**Canonical problem:** *Range Sum Query - Immutable*, *Corporate Flight Bookings* (difference array), *Number of Ways to Split Array*.

### 12. Monotonic stack / queue

**Recognize it by:** "next greater / smaller element", "largest rectangle in histogram", "daily temperatures", "stock span", "sliding window maximum" (monotonic deque), "max in every window of size K".

**Core idea:** Maintain a stack (or deque) that is monotonic in some order. When a new element is processed, pop from the top of the stack while doing so preserves the monotonic property — the popped elements find "their answer" at this moment. This turns a "for each i, scan right for next greater" O(n^2) task into O(n).

**Complexity:** O(n) time (each element pushed/popped at most once). O(n) space.

**Canonical problem:** *Daily Temperatures* / *Next Greater Element* (stack); *Sliding Window Maximum* (deque); *Largest Rectangle in Histogram* (stack).

### 13. Bit manipulation basics

**Recognize it by:** "without using +", "single number / one missing", "power of two", "count set bits", "swap without temp", "subset of bits", "XOR of all". When the problem reduces to "treat the number as a vector of bits."

**Core ideas to know:
- `x ^ x == 0`; XOR is commutative/associative; XOR-ing all values finds the singleton in `O(n)`, `O(1)` space (*Single Number*).
- `x & (x-1)` flips the lowest set bit off — basis for counting bits and power-of-two check (`x > 0 && x & (x-1) == 0`).
- `x ^ y` swaps without temp.
- Right shift `>>` divides by 2; left shift `<<` multiplies by 2.
- For subsets of a set of size n, iterate a mask from 0 to `2^n - 1`; bit i set means "include element i."

**Complexity:** O(1) for basic ops on fixed-width ints; O(n · wordsize) when computing over n values.

**Canonical problem:** *Single Number*, *Number of 1 Bits*, *Power of Two*, *Subsets* (with bitmask enumeration as an alternative to backtracking).

---

## Final tuning for backend interviews

- State complexity at the start: time **and** space; mention average vs worst if it matters.
- Prefer solutions that map to a real-world backend artifact (a map for an index, a heap for a scheduler, a topological sort for a build graph). This makes your answer easier to defend in follow-ups.
- Edge cases worth always checking: empty input, single element, all-same input, the input already sorted, the input reversed, off-by-one window bounds.
- If your first solution is the obvious O(n^2), say so explicitly, then immediately say "I can do better with X for O(n)" — interviewers are listening for that pivot.