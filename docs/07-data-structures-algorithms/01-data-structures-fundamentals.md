# Data Structures Fundamentals

Before patterns (two pointers, sliding window, DFS/BFS — covered in the next file), you need the structures themselves cold: what each one is, what it costs, and the small set of problems that show up on nearly every coding screen because they *are* the structure — you cannot bluff a linked-list reversal or a valid-parentheses check with a memorized pattern, you have to actually manipulate pointers or a stack correctly under time pressure.

This file goes structure by structure. Each section: the concept, the Big-O (full table in `README.md`), and 2-4 canonical problems that recur across companies — with a working solution and the complexity called out, the way you should state it out loud in an interview.

---

## Arrays and strings

**Concept:** A contiguous block of memory, fixed size (array) or growable (Go slice, dynamic array). Index access is O(1) because the address is `base + index*size`. The cost you pay for that is insertion/deletion in the middle: everything after the gap has to shift, so it's O(n). Strings are usually arrays of bytes/runes with the same trade-offs, plus the extra wrinkle that in Go a `string` is immutable — mutating one means building a new one (or working over `[]byte`/`[]rune`).

**Big-O:** access O(1), search O(n) unsorted / O(log n) sorted (binary search), insert/delete O(n) (O(1) at the end, amortized, for append).

### Must-know: Two Sum

The canonical time/space trade-off problem — brute force O(n²)/O(1) vs hash map O(n)/O(n).

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums)) // value -> index
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```

### Must-know: reverse an array in place, and rotate an array

Reversal is the O(1)-space primitive most rotation/palindrome problems are built from.

```go
func reverse(nums []int, lo, hi int) {
    for lo < hi {
        nums[lo], nums[hi] = nums[hi], nums[lo]
        lo++
        hi--
    }
}

// Rotate right by k: reverse whole array, then reverse each half —
// O(n) time, O(1) space, no extra array.
func rotate(nums []int, k int) {
    n := len(nums)
    k %= n
    reverse(nums, 0, n-1)
    reverse(nums, 0, k-1)
    reverse(nums, k, n-1)
}
```

### Must-know: valid anagram / group anagrams

Tests whether you reach for a frequency map instead of sorting-as-a-crutch (sorting each string is a valid O(n log n) answer, but a length-26 frequency array is O(n) and the answer that signals more fluency).

```go
func isAnagram(s, t string) bool {
    if len(s) != len(t) {
        return false
    }
    var counts [26]int
    for i := range s {
        counts[s[i]-'a']++
        counts[t[i]-'a']--
    }
    for _, c := range counts {
        if c != 0 {
            return false
        }
    }
    return true
}
```

---

## Linked lists

**Concept:** A chain of nodes, each holding a value and a pointer to the next (singly) or next+prev (doubly). No contiguous memory, so no O(1) index access — but insert/delete at a known node is O(1) because there's no shifting, just pointer rewiring. This is exactly why an LRU cache pairs a doubly linked list (O(1) move-to-front, O(1) evict-from-back) with a hash map (O(1) find-by-key) — neither structure alone gives you both.

**Big-O:** access/search O(n), insert/delete O(1) if you already have the node (O(n) to find it first).

### Must-know: reverse a linked list

The single most-asked linked-list question. Know both the iterative (O(1) space) and recursive (O(n) space, stack depth) versions — interviewers often ask for both as a follow-up.

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    for head != nil {
        next := head.Next
        head.Next = prev
        prev = head
        head = next
    }
    return prev
}

func reverseListRecursive(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseListRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```

### Must-know: detect a cycle (Floyd's algorithm)

Fast/slow pointers: if there's a cycle, the fast pointer (2 steps) laps the slow pointer (1 step) inside it; if there isn't, fast hits `nil` first. O(n) time, O(1) space — the entire point versus the O(n)-space "store visited nodes in a set" answer.

```go
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
```

The natural follow-up — "find where the cycle starts" — adds one more step: once slow and fast meet, reset one pointer to `head` and advance both one step at a time; they meet again exactly at the cycle's start. Worth having memorized, since it's asked as a direct follow-up more often than as a first question.

### Must-know: merge two sorted lists

The linked-list analogue of a merge-sort merge step; also the building block for "merge k sorted lists" (see the heap section below).

```go
func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    tail := dummy
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            tail.Next = l1
            l1 = l1.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
        }
        tail = tail.Next
    }
    if l1 != nil {
        tail.Next = l1
    } else {
        tail.Next = l2
    }
    return dummy.Next
}
```

### Must-know: remove the Nth node from the end

One-pass with a lead offset — the "why does the interviewer keep asking about a dummy head node" answer: it removes the special-case branch for deleting the actual head.

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{Next: head}
    slow, fast := dummy, dummy
    for i := 0; i < n; i++ {
        fast = fast.Next
    }
    for fast.Next != nil {
        slow = slow.Next
        fast = fast.Next
    }
    slow.Next = slow.Next.Next
    return dummy.Next
}
```

---

## Stacks and queues

**Concept:** A stack is LIFO — push/pop from one end, O(1) both. A queue is FIFO — enqueue at one end, dequeue at the other; O(1) with a doubly linked list or a ring buffer, but naively O(n) with a slice-as-queue because dequeuing from the front means shifting everything (`slice[1:]` doesn't help — it still holds the backing array). Backend relevance: a stack models function calls / undo / DFS; a queue models job processing order / BFS / rate limiting.

**Big-O:** push/pop/enqueue/dequeue O(1); access/search O(n).

### Must-know: valid parentheses

The canonical "is a stack the right structure" question — the LIFO property matches the nesting structure of brackets exactly.

```go
func isValid(s string) bool {
    stack := make([]byte, 0, len(s))
    pairs := map[byte]byte{')': '(', ']': '[', '}': '{'}
    for i := 0; i < len(s); i++ {
        c := s[i]
        if open, ok := pairs[c]; ok {
            if len(stack) == 0 || stack[len(stack)-1] != open {
                return false
            }
            stack = stack[:len(stack)-1]
        } else {
            stack = append(stack, c)
        }
    }
    return len(stack) == 0
}
```

### Must-know: min stack (design a stack with O(1) `getMin`)

Tests whether you reach for "carry auxiliary state alongside the data" rather than re-scanning — push a `(value, currentMin)` pair so the min is always O(1) to read.

```go
type MinStack struct {
    stack [][2]int // [value, minSoFar]
}

func (s *MinStack) Push(val int) {
    min := val
    if len(s.stack) > 0 && s.stack[len(s.stack)-1][1] < min {
        min = s.stack[len(s.stack)-1][1]
    }
    s.stack = append(s.stack, [2]int{val, min})
}

func (s *MinStack) Pop() { s.stack = s.stack[:len(s.stack)-1] }
func (s *MinStack) Top() int { return s.stack[len(s.stack)-1][0] }
func (s *MinStack) GetMin() int { return s.stack[len(s.stack)-1][1] }
```

### Must-know: implement a queue using two stacks (or vice versa)

Tests whether you understand *why* the naive version is O(n): reverse the order twice. Amortized O(1) per operation because each element is only ever moved once across the two stacks.

```go
type MyQueue struct {
    in, out []int
}

func (q *MyQueue) Push(x int) { q.in = append(q.in, x) }

func (q *MyQueue) Pop() int {
    if len(q.out) == 0 {
        for len(q.in) > 0 {
            n := len(q.in) - 1
            q.out = append(q.out, q.in[n])
            q.in = q.in[:n]
        }
    }
    n := len(q.out) - 1
    v := q.out[n]
    q.out = q.out[:n]
    return v
}
```

---

## Hash tables

**Concept:** An array of buckets indexed by a hash of the key, with collision resolution (chaining — a linked list/slice per bucket — or open addressing/probing). O(1) average because a good hash function spreads keys evenly; O(n) worst case if everything collides into one bucket (why hash-flooding is a real denial-of-service vector, and why languages randomize hash seeds per-process). Go's built-in `map` is a hash table with random iteration order by design — never rely on map iteration order in an interview or in production.

**Big-O:** access/search/insert/delete O(1) average, O(n) worst case; O(1) worst case only for perfect hashing on a known key set.

### Must-know: LRU cache

The single most-asked "design" question in coding screens, and the reason the hash-map-plus-doubly-linked-list combination from the linked-list section exists: the hash map gives O(1) lookup by key, the doubly linked list gives O(1) move-to-front on access and O(1) evict-from-the-back on capacity overflow. Neither structure alone hits O(1) for both operations.

```go
type node struct {
    key, val   int
    prev, next *node
}

type LRUCache struct {
    cap        int
    items      map[int]*node
    head, tail *node // head = most recently used
}

func NewLRUCache(capacity int) *LRUCache {
    head, tail := &node{}, &node{}
    head.next, tail.prev = tail, head
    return &LRUCache{cap: capacity, items: make(map[int]*node), head: head, tail: tail}
}

func (c *LRUCache) remove(n *node) {
    n.prev.next, n.next.prev = n.next, n.prev
}

func (c *LRUCache) insertFront(n *node) {
    n.next, n.prev = c.head.next, c.head
    c.head.next.prev, c.head.next = n, n
}

func (c *LRUCache) Get(key int) int {
    n, ok := c.items[key]
    if !ok {
        return -1
    }
    c.remove(n)
    c.insertFront(n)
    return n.val
}

func (c *LRUCache) Put(key, val int) {
    if n, ok := c.items[key]; ok {
        n.val = val
        c.remove(n)
        c.insertFront(n)
        return
    }
    if len(c.items) == c.cap {
        lru := c.tail.prev
        c.remove(lru)
        delete(c.items, lru.key)
    }
    n := &node{key: key, val: val}
    c.items[key] = n
    c.insertFront(n)
}
```

State the complexity explicitly: O(1) for both `Get` and `Put` — that's the entire point of the design, and interviewers listen for whether you say it unprompted.

---

## Trees

**Concept:** A hierarchical structure — each node has children (binary tree: at most two, `left`/`right`). A binary search tree (BST) adds the invariant `left subtree < node < right subtree`, which is what turns tree operations into O(log n) *if the tree is balanced*. An unbalanced BST (e.g., inserting a sorted sequence into a naive BST) degrades to a linked list — O(n). Self-balancing variants (AVL, red-black trees) maintain the O(log n) guarantee by rebalancing on insert/delete; you're rarely asked to implement one from scratch, but you should know *why* they exist and that Go's standard library doesn't ship a balanced BST (reach for a sorted slice + binary search, or a package, when you need one).

**Big-O (balanced BST):** access/search/insert/delete O(log n). Unbalanced worst case: O(n).

### Must-know: level-order traversal (BFS) and the three DFS orders

BFS uses a queue; the three DFS orders (preorder, inorder, postorder) differ only in when you visit the node relative to its children. Inorder on a BST visits nodes in sorted order — that fact alone answers a surprising number of BST questions.

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var result [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        level := make([]int, 0, len(queue))
        var next []*TreeNode
        for _, n := range queue {
            level = append(level, n.Val)
            if n.Left != nil {
                next = append(next, n.Left)
            }
            if n.Right != nil {
                next = append(next, n.Right)
            }
        }
        result = append(result, level)
        queue = next
    }
    return result
}
```

### Must-know: validate a BST

The trap: checking only `node.Val > node.Left.Val && node.Val < node.Right.Val` locally is wrong — a node deep in the left subtree can still violate the *ancestor's* bound. Pass a valid `(min, max)` range down through the recursion.

```go
func isValidBST(root *TreeNode) bool {
    var validate func(n *TreeNode, min, max *int) bool
    validate = func(n *TreeNode, min, max *int) bool {
        if n == nil {
            return true
        }
        if min != nil && n.Val <= *min {
            return false
        }
        if max != nil && n.Val >= *max {
            return false
        }
        return validate(n.Left, min, &n.Val) && validate(n.Right, &n.Val, max)
    }
    return validate(root, nil, nil)
}
```

### Must-know: lowest common ancestor

For a plain binary tree: recurse both sides; if both sides return non-nil, the current node is the LCA. For a BST specifically, you can do better — O(log n) by walking down once, using the ordering: if both targets are less than the current node, go left; if both are greater, go right; otherwise you've found the split point.

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }
    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)
    if left != nil && right != nil {
        return root
    }
    if left != nil {
        return left
    }
    return right
}
```

Also expect: max depth (one-liner recursion), invert a binary tree (swap left/right recursively — the infamous "half of engineers couldn't do this on a whiteboard" problem), and diameter of a binary tree (longest path between any two nodes — postorder DFS returning height while tracking a running max).

---

## Heaps / priority queues

**Concept:** A binary tree stored as an array, satisfying the heap property (min-heap: parent ≤ children; max-heap: parent ≥ children). Peek at the root is O(1); insert and extract-min/max are O(log n) because you only need to fix the path from a leaf to the root (sift-up) or root to a leaf (sift-down), never the whole tree. This is *why* a heap beats a sorted array for "repeatedly get the smallest/largest, and keep inserting" — a sorted array gives O(1) peek too, but O(n) insert.

**Big-O:** peek O(1), insert/extract O(log n), build-heap from an unsorted array O(n) (not O(n log n) — a fact worth stating if asked, it's a common trip-up).

### Must-know: kth largest element

The go-to demonstration of "heap of bounded size beats full sort": a min-heap capped at size k, where anything larger than the heap's min gets pushed and the min popped, gives O(n log k) instead of sorting the whole array at O(n log n).

```go
import "container/heap"

type minHeap []int

func (h minHeap) Len() int           { return len(h) }
func (h minHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h minHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *minHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *minHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func findKthLargest(nums []int, k int) int {
    h := &minHeap{}
    heap.Init(h)
    for _, n := range nums {
        heap.Push(h, n)
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    return (*h)[0]
}
```

Go's `container/heap` requires implementing `sort.Interface` plus `Push`/`Pop` — knowing this API cold (rather than deriving it live) saves real interview time.

### Must-know: merge k sorted lists

The direct generalization of "merge two sorted lists" from the linked-list section: put the head of each list into a min-heap keyed by value, repeatedly pop the smallest and push its successor. O(n log k) where n is total nodes and k is the number of lists — versus O(nk) for the naive pairwise-merge-everything approach.

Also expect: top-k frequent elements (hash map for counts, then a heap of size k — the same "bounded heap" idea as kth-largest) and task scheduler (max-heap by remaining count, cooldown via a queue).

---

## Graphs

**Concept:** Nodes (vertices) and edges, directed or undirected, weighted or not. Represented as an adjacency list (map/slice of neighbors per node — the default choice, O(V+E) space) or an adjacency matrix (O(V²) space, O(1) edge lookup — worth it only when the graph is dense or you need fast edge queries). Trees are graphs with no cycles and exactly one path between any two nodes; most tree algorithms are graph algorithms specialized to that shape.

**Big-O:** BFS/DFS traversal O(V+E). Adjacency-matrix edge lookup O(1) but O(V²) space; adjacency-list edge lookup O(degree) but O(V+E) space.

### Must-know: number of islands (grid BFS/DFS)

The canonical "graph problem disguised as a 2D grid" — every backend engineer should recognize a grid as an implicit graph where each cell is a node connected to its 4 (or 8) neighbors.

```go
func numIslands(grid [][]byte) int {
    rows, cols := len(grid), len(grid[0])
    var sink func(r, c int)
    sink = func(r, c int) {
        if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1' {
            return
        }
        grid[r][c] = '0' // mark visited by mutating; use a visited set if the grid is read-only
        sink(r-1, c)
        sink(r+1, c)
        sink(r, c-1)
        sink(r, c+1)
    }
    count := 0
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if grid[r][c] == '1' {
                count++
                sink(r, c)
            }
        }
    }
    return count
}
```

### Must-know: detect a cycle in a directed graph / course schedule

This is topological sort's companion problem and comes up constantly in "can these tasks/courses be ordered" framing. DFS with three states (unvisited, in-progress, done) — hitting an in-progress node again means a cycle; Kahn's algorithm (repeatedly remove zero-in-degree nodes) is the BFS alternative and is what production dependency-resolution code usually uses. Full walkthrough with backend framing (build systems, service dependency graphs) is in `02-common-patterns.md`.

Also expect: clone a graph (BFS/DFS with a visited map from old node to new node, so cycles don't cause infinite recursion) and word ladder / shortest path in an unweighted graph (BFS gives shortest path by construction — the moment "shortest" or "fewest steps" appears with unweighted edges, that's the recognition cue for BFS over DFS).

---

## What to do with this file

Read it once fully, then treat it as a reference, not memorization material: reimplement each "must-know" from scratch, cold, without looking, until you can. The patterns file next (`02-common-patterns.md`) builds on top of these structures — two pointers assumes you're comfortable with arrays and linked lists, the heap pattern assumes you know `container/heap`'s API, and the graph patterns assume the adjacency-list traversal above is automatic.
