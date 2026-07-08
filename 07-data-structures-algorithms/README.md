# Data Structures & Algorithms

Backend interviews consistently test Data Structures & Algorithms (DSA) — not because you'll hand-roll a red-black tree in production, but because DSA is a proxy for how you reason about correctness, performance, and trade-offs. The work of a backend engineer is full of DSA-shaped problems: choosing the right data structure for an index, reasoning about the cost of a join, bounding the latency of a fan-out, designing a queue with ordering guarantees, or avoiding an O(n^2) blow-up in a hot path that serves millions of requests.

## Why DSA matters for backend

- **Performance is a feature.** A factors-of-n difference in a per-request code path is the difference between 5ms and 500ms p99.
- **Data structure choice encodes system design.** LRU cache → hash map + doubly linked list. Job scheduler → priority queue. Dependency resolver → topological sort. Rate limiter → sliding window. These show up in both "coding" and "system design" rounds.
- **It filters for careful thinking.** Interviewers care less about a perfect one-liner and more about: do you understand the problem, do you state assumptions, do you pick the right structure, do you reason about complexity, do you handle edge cases.

## How to use this section

1. Learn the patterns in `01-common-patterns.md`. Most interview problems are a recognizable pattern plus one twist. Pattern recognition gets you 70% of the way.
2. Drill the problem set in `02-interview-questions.md`. Each problem lists the pattern, an approach sketch, and a tiny I/O example. Try the problem first, then read the approach to check your thinking.
3. Practice in timed conditions (30-45 min per medium/hard) with a real code editor. Talk through your reasoning out loud.

## Big-O cheat sheet (common data structure operations)

Average-case unless noted. `n` = number of elements.

| Data structure          | Access      | Search      | Insert           | Delete           | Space   |
|-------------------------|-------------|-------------|------------------|------------------|---------|
| Array (static)          | O(1)        | O(n)        | O(n) (end: O(1)) | O(n)             | O(n)    |
| Dynamic array (slice)   | O(1)        | O(n)        | O(1) amortized   | O(n)             | O(n)    |
| Singly linked list      | O(n)        | O(n)        | O(1) (at head)   | O(1) (at head)   | O(n)    |
| Doubly linked list      | O(n)        | O(n)        | O(1)             | O(1)             | O(n)    |
| Hash map                | O(1) avg    | O(1) avg    | O(1) avg         | O(1) avg         | O(n)    |
| Balanced BST            | O(log n)    | O(log n)    | O(log n)         | O(log n)         | O(n)    |
| Heap (binary)           | O(1) (peek) | O(n)        | O(log n)         | O(log n) (top)   | O(n)    |
| Queue / stack           | O(n)        | O(n)        | O(1)             | O(1)             | O(n)    |
| Priority queue (heap)   | O(1) (peek) | O(n)        | O(log n)         | O(log n)         | O(n)    |
| Trie                    | O(m)        | O(m)        | O(m)             | O(m)             | O(Σ·m)  |
| Union-Find              | O(α(n))     | O(α(n))     | O(α(n))          | n/a              | O(n)    |

Notes:
- Hash map worst case is O(n) (collisions / rehashing); in interviews assume O(1) average and mention the caveat if asked.
- `m` for a trie is the length of the key. `α(n)` for Union-Find is the inverse Ackermann function, effectively <= 5 for all realistic inputs.
- Sorted arrays give O(log n) search via binary search; inserts/deletes are O(n) because of shifting.

## Sorting algorithm cheat sheet

| Algorithm        | Best     | Average   | Worst     | Space    | Stable | When to use                          |
|------------------|----------|-----------|-----------|----------|--------|--------------------------------------|
| Merge sort       | O(n lg n) | O(n lg n) | O(n lg n) | O(n)     | Yes    | Stable external sort, linked lists  |
| Quick sort       | O(n lg n) | O(n lg n) | O(n^2)    | O(lg n)  | No     | In-place, cache-friendly default     |
| Heap sort        | O(n lg n) | O(n lg n) | O(n lg n) | O(1)     | No     | Guaranteed O(n lg n), in-place       |
| Counting sort    | O(n+k)   | O(n+k)    | O(n+k)    | O(n+k)   | Yes    | Small integer key range              |
| Introsort (C++)  | O(n lg n) | O(n lg n) | O(n lg n) | O(lg n)  | No     | Most standard-library sorts          |

Most languages' built-in sort is a hybrid (Timsort for Python/Java, pdqsort for Go 1.19+, introsort for C++). In interviews you almost never implement sort from scratch unless explicitly asked — but you should know quicksort's partitioning (used for quickselect / kth largest) and heap operations.

## Recommended practice resources

- **LeetCode** — the de facto bank. Suggested order:
  - Start with the NeetCode 150 / LeetCode 75 curated lists; they cover every pattern above with canonical problems.
  - Prioritize Medium problems over Easy (interview-realistic) and over Hard (diminishing returns).
  - Do at least 10-15 Array/Hashing, 5-10 Tree/Graph, 5 Heap, 5 DP, 5 Two-Pointer/Sliding-Window.
- **NeetCode** (neetcode.io) — free pattern-organized roadmap with video explanations. Best structured resource for the patterns in `01-common-patterns.md`.
- **"Elements of Programming Interviews"** (Adnan Aziz et al.) — higher signal per problem than LeetCode random; works through patterns deliberately. There is a Go/C++/Java/Python edition.
- **"Cracking the Coding Interview"** (McDowell) — gentler intro, good if you've been away from DSA for years.
- **AlgorithmFridays / algorithms.wtf** — written by a competitive programmer, rigorous on correctness and complexity; good for the harder graph/DP problems.
- **Grokking the Coding Interview: Patterns for Coding Questions** (DesignGurus) — pattern-first organization; useful as a companion to NeetCode for the "recognize the pattern" muscle.
- **Prep practice** — do mock interviews on pramp.com or interview a friend. Talking while coding is a different skill from coding alone; people fail the "think out loud" part more than the coding part.

## Files in this section

| File | Contents |
|------|----------|
| `README.md` | This file — overview, Big-O cheat sheet, resources. |
| `01-common-patterns.md` | The recurring interview patterns, with recognition cues, core idea, complexity, and a canonical problem each. |
| `02-interview-questions.md` | 20+ canonical backend-interview problems grouped by difficulty, with approach sketches and I/O examples. |