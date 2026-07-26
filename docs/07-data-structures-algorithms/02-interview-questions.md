# Interview Questions — DSA Problems for Backend

20+ canonical backend-interview problems. Each entry lists the pattern, an approach sketch with complexity, and a tiny I/O example. Try the problem first; use this to check your thinking and to look up the pattern when you're stuck.

Convention: `n` is input size; assume valid, non-null inputs unless stated otherwise.

## Easy

### Q1: Two Sum — return indices of two numbers that add to target

**Approach:** Hash-map, one pass. For each index `i`, check whether `target - nums[i]` is in a `seen` map; if so return both indices, else store `nums[i] -> i`. Each value is inspected once and lookup is O(1) average. Time O(n), space O(n).

**Example:** `nums = [2,7,11,15]`, `target = 9` -> `[0,1]`.

### Q2: Valid Parentheses — are `()[]{}` balanced

**Approach:** Stack. On an open bracket push; on a close bracket require it to match the top and pop. At the end the stack must be empty. Time O(n), space O(n).

**Example:** `s = "([{}])"` -> `true`; `s = "([)]"` -> `false`.

### Q3: Valid Anagram — are two strings character-frequency equal

**Approach:** Frequency map (or a fixed 26-int array). Count `s`, decrement with `t`, all zero at the end. Time O(n), space O(1) (bounded alphabet).

**Example:** `s = "anagram"`, `t = "nagaram"` -> `true`.

### Q4: Binary Search — locate target in a sorted array

**Approach:** Standard `[lo, hi]` binary search with `mid = lo + (hi-lo)/2`. Time O(log n), space O(1). Don't write `mid = (lo+hi)/2` — overflow.

**Example:** `nums = [-1,0,3,5,9,12]`, `target = 9` -> index `4`.

### Q5: Reverse Linked List — reverse a singly linked list

**Approach:** Iterate, keeping `prev`, `curr`, `next`. At each step point `curr.Next = prev`, advance. Time O(n), space O(1). Recursive version uses O(n) stack.

**Example:** `1->2->3->4->5` -> `5->4->3->2->1`.

### Q6: Best Time to Buy and Sell Stock — max profit from one buy/sell

**Approach:** One pass, track the minimum price seen so far and the best profit (= `price - min`). Time O(n), space O(1). Equivalent to a running prefix minimum.

**Example:** `prices = [7,1,5,3,6,4]` -> `5` (buy at 1, sell at 6).

### Q7: Climbing Stairs — number of distinct ways to climb n stairs taking 1 or 2 steps

**Approach:** 1D DP. `dp[i] = dp[i-1] + dp[i-2]`, `dp[0]=dp[1]=1`. Only two previous values needed, so constant space. Time O(n), space O(1).

**Example:** `n = 3` -> `3` (1+1+1, 1+2, 2+1).

## Medium

### Q8: Longest Substring Without Repeating Characters — longest substring with all distinct chars

**Approach:** Sliding window. Maintain `left` and a map of last occurrence of each char; expand `right`, and when `s[right]` was seen at `>= left`, jump `left` past it; track the max window. Each char entered/leaves the window once. Time O(n), space O(min(n, alphabet)).

**Example:** `s = "abcabcbb"` -> `3` (`"abc"`).

### Q9: Merge Intervals — merge overlapping intervals

**Approach:** Sort by start. Iterate; if the current start `<= last merged end`, extend the merged end to `max(end, current.end)`; else push the current as a new interval. Time O(n log n) for the sort, O(n) for the pass. Space O(n) for the output. Backend-flavored: merging maintenance windows, deduping overlapping alerts.

**Example:** `[[1,3],[2,6],[8,10],[15,18]]` -> `[[1,6],[8,10],[15,18]]`.

### Q10: LRU Cache — design an O(1) get/put cache evicting least-recently-used — backend relevant

**Approach:** Hash map keyed by `key -> doubly linked list node` plus a doubly linked list ordered by recency (head = most recent). `get`: O(1) map lookup, move node to head. `put`: insert at head; if over capacity, evict tail and remove it from the map. The list gives O(1) splice for recency updates; the map gives O(1) access by key. Time O(1) per op. Space O(capacity). This is one of the most backend-relevant design problems: it's literally an in-memory cache.

**Example:** `LRUCache(2); put(1,1); put(2,2); get(1)=1; put(3,3)` evicts key 2; `get(2)=-1`.

```go
type entry struct {
    key, val   int
    prev, next *entry
}
type LRUCache struct {
    cap        int
    m          map[int]*entry
    head, tail *entry
}
func NewLRU(cap int) *LRUCache {
    h, t := &entry{}, &entry{}
    h.next, t.prev = t, h
    return &LRUCache{cap, map[int]*entry{}, h, t}
}
func (c *LRUCache) unlink(e *entry) {
    e.prev.next, e.next.prev = e.next, e.prev
}
func (c *LRUCache) pushFront(e *entry) {
    e.prev, e.next = c.head, c.head.next
    c.head.next.prev, c.head.next = e, e
}
func (c *LRUCache) Get(key int) int {
    if e, ok := c.m[key]; ok {
        c.unlink(e); c.pushFront(e)
        return e.val
    }
    return -1
}
func (c *LRUCache) Put(key, val int) {
    if e, ok := c.m[key]; ok {
        c.unlink(e); e.val = val; c.pushFront(e); return
    }
    e := &entry{key: key, val: val}
    c.m[key] = e; c.pushFront(e)
    if len(c.m) > c.cap {
        lru := c.tail.prev
        c.unlink(lru); delete(c.m, lru.key)
    }
}
```

### Q11: Kth Largest Element in an Array

**Approach:** Maintain a min-heap of size K; iterate nums; push then if len > K pop; the top is the Kth largest. Time O(n log k), space O(k). Quickselect (randomized partitioning) is O(n) average, O(n^2) worst — fine, but the heap version is simpler to get bug-free under pressure.

**Example:** `nums = [3,2,1,5,6,4], k = 2` -> `5`.

### Q12: Top K Frequent Elements — k most frequent values

**Approach:** Frequency map (`value -> count`) then min-heap of size k keyed by count. Time O(n log k), space O(n). Alternative "bucket sort" with buckets indexed by frequency is O(n) time and O(n) space.

**Example:** `nums = [1,1,1,2,2,3], k = 2` -> `[1,2]`.

### Q13: Course Schedule — can all courses be finished given prerequisites — backend relevant

**Approach:** Topological sort (Kahn's). Build adjacency list + in-degree array; enqueue zero-in-degree nodes; pop, decrement neighbors, enqueue new zeros; if processed count == n, no cycle. Time O(V+E), space O(V+E). Maps directly to dependencies between services/migrations/CI steps.

**Example:** `n = 2, prerequisites = [[1,0]]` -> `true`; `[[1,0],[0,1]]` -> `false` (cycle).

### Q14: Number of Islands — count connected land cells in a grid

**Approach:** DFS/BFS. For each unvisited land cell increment count and flood-fill (mark visited) all reachable land. Each cell visited once. Time O(rows·cols), space O(rows·cols) worst case (call stack for DFS, queue for BFS).

**Example:** grid with `['111','010','011']` -> `3`.

### Q15: Rotting Oranges — minimum minutes until all fresh oranges rot (or -1)

**Approach:** Multi-source BFS. Enqueue all initially-rotten cells; BFS layer by layer, infecting adjacent fresh cells; track the last layer reached; if any fresh cell remains after, return -1. Time O(rows·cols), space O(rows·cols).

**Example:** `[[2,1,1],[1,1,0],[0,1,1]]` -> `4`.

### Q16: Binary Tree Level Order Traversal — return node values grouped by depth

**Approach:** BFS with a queue; for each level record its size, process that many nodes, collect children. Time O(n), space O(n).

**Example:** tree `[3,9,20,null,null,15,7]` -> `[[3],[9,20],[15,7]]`.

### Q17: Maximum Subarray — Kadane, contiguous subarray with largest sum

**Approach:** 1D DP / Kadane. `best = max(best, running)` where `running = max(x, running+x)`. Time O(n), space O(1).

**Example:** `nums = [-2,1,-3,4,-1,2,1,-5,4]` -> `6` (`[4,-1,2,1]`).

### Q18: Detect Cycle in a Linked List — is there a cycle?

**Approach:** Floyd's tortoise & hare. Slow moves 1, fast 2; if they meet -> cycle; if fast hits nil -> no cycle. Time O(n), space O(1). The hash-map "visited pointer" alternative uses O(n) space.

**Example:** `head = [3,2,0,-4]` with cycle at index 1 -> `true`.

### Q19: Coin Change — fewest coins to make a target amount (unbounded knapsack)

**Approach:** 1D DP. `dp[0] = 0`; `dp[a] = 1 + min over coins (dp[a - coin])` when `a >= coin`. If any `dp[a - coin]` is unreachable, ignore it. Time O(amount · coins), space O(amount). Initialize with a sentinel larger than any reachable answer (e.g., `amount+1`).

**Example:** `coins = [1,2,5], amount = 11` -> `3` (`5+5+1`).

### Q20: Longest Palindromic Substring — longest substring that's a palindrome

**Approach:** Expand-around-center: iterate each index `i`, expand outward for odd-length and even-length palindromes centered there. Track the longest seen. Time O(n^2), space O(1). DP is O(n^2) using `isPal[i][j]` and O(n^2) space; Manacher's is O(n) but rarely needed.

**Example:** `s = "babad"` -> `"bab"` (or `"aba"`).

### Q21: Design Twitter — post tweets and get a user's news feed (most recent 10 from self + followees) — backend relevant

**Approach:** Each user has a chronologically-ordered list of tweets (`(timestamp, tweetId)`); `getNewsFeed` does a K-way merge of the most recent tweets of the user and followed users, picking the k (10) most recent. Use a max-heap of size `num_followees+1` seeded with each list's head; pop the largest, push the next tweet of that user. Time O(k · log F) for the feed where F is the number of followees; space O(total_tweets). Maps cleanly to fan-out/merge patterns: combine recent items from multiple sources by timestamp, the heart of many backend aggregators.

**Example:** `postTweet(1,5); follow(1,2); postTweet(2,7); getNewsFeed(1)` -> `[7,5]`.

### Q22: Serialize and Deserialize Binary Tree — encode/decode a binary tree to a string

**Approach:** Pre-order (or level-order) traversal, writing null markers for missing children. On deserialize, parse the values in order, recursing and honoring null markers. Time O(n), space O(n) for the string and recursion stack. Backend-flavored: unit-testing tree-shaped data, storing JSON document trees, copying structures over a wire.

**Example:** tree `[1,2,3,null,null,4,5]` -> `"1,2,#,#,3,4,#,#,5,#,#"` (schema is up to you).

### Q23: Daily Temperatures — for each day, how many days until a warmer one

**Approach:** Monotonic stack. Iterate left to right keeping a stack of indices whose answer is unknown, strictly decreasing by temperature. When the current temperature beats the stack top, pop it — the current index is its "next warmer day" — and record the distance. Each index is pushed and popped at most once. Time O(n), space O(n). This is the canonical "next greater element" shape; the same stack solves *Stock Span* and *Largest Rectangle in Histogram*.

**Example:** `temperatures = [73,74,75,71,69,72,76,73]` -> `[1,1,4,2,1,1,0,0]`.

### Q24: Meeting Rooms II — minimum number of rooms for all meetings

**Approach:** Intervals + min-heap sweep. Sort intervals by start; walk them keeping a min-heap of end times of in-progress meetings. Before pushing the current meeting's end, pop every end time `<=` current start (those rooms freed up). The peak heap size is the answer. Time O(n log n), space O(n). Backend-flavored: peak concurrency estimation — "how many workers/connections do these overlapping jobs need at once" is exactly this problem.

**Example:** `intervals = [[0,30],[5,10],[15,20]]` -> `2`.

## Hard

### Q25: Serialize and Deserialize a Trie of Dictionary Words — import/export a routing/dictionary trie

**Approach:** Represent the trie as nested words; serialize with BFS recording (char, isEnd) per node and parent index. On deserialize, rebuild using the same edge ordering. Time O(total chars), space O(total chars). Backend-flavored: a trie is the natural representation for routing tables, autocomplete, and prefix indexes.

**Example:** dictionary `["app","apple","bat"]` -> some self-describing encoding, e.g. `root->a(p(p(le?))?)->b(at?)`.

### Q26: Palindrome Pairs — for which pair `(i,j)` is `words[i] + words[j]` a palindrome

**Approach:** Trie of reversed words + a hash-map lookup. For each word w, find candidates whose reverse complements w's suffix/prefix such that the leftover is itself a palindrome. Time O(sum of word lengths · alphabet) with a trie, or O(n · k^2) with simpler hash lookups where `k` is max word length. Space O(sum of word lengths).

**Example:** `words = ["abcd","dcba","lls","s","sssll"]` -> `[[0,1],[1,0],[3,2],[2,4]]`.

### Q27: Median of Two Sorted Arrays — median of combined sorted arrays in O(log(min(n,m)))

**Approach:** Binary-search the partition index in the shorter array, with the longer array's partition determined by the count split; ensure the max of the left halves <= min of the right halves, otherwise adjust. Time O(log(min(n,m))), space O(1). This is the canonical "edges of two merged streams are deterministic" hard problem — relevant whenever you maintain two ranked streams and want the global median without merging.

**Example:** `A = [1,3], B = [2]` -> `2.0`.

---

## Quick-study order

If you have limited time, practice in this order to maximize signal:

1. Two Sum, Valid Parentheses, Binary Search, Best Time to Buy/Sell Stock — warm-up, baseline correctness.
2. LRU Cache, Course Schedule, Merge Intervals, Design Twitter — backend-flavored; do these twice.
3. Longest Substring Without Repeating Characters, Number of Islands, Rotting Oranges, Maximum Subarray, Daily Temperatures, Meeting Rooms II — high-frequency patterns (sliding window, BFS, Kadane, monotonic stack, intervals).
4. Kth Largest Element, Top K Frequent Elements — heap pattern.
5. Coin Change, Climbing Stairs — DP basics. LIS/LCS only if you expect DP-heavy loops.
6. Serialize/Deserialize Binary Tree — pointer/recursion correctness under pressure.

For each, write code (not pseudo) in your interview language, state time **and** space complexity out loud, and have two test cases ready (the normal case and one edge case).