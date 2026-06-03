---
title: "From Arrays to Dijkstra: A Complete DSA Playbook"
date: 2026-06-03
layout: post
permalink: /dsa-mastery-playbook/
categories: [DSA, Algorithms, Data Structures, Python]
tags: [Arrays, LinkedList, Trees, Graphs, DynamicProgramming, Backtracking, Heap, BinarySearch, Sorting, LeetCode, Python, Interview]
description: "Everything you need to go from scratch to confidently solving complex algorithmic problems — arrays, trees, graphs, DP, and beyond. Covers 10+ data structures, 8 sorting algorithms, 4 graph algorithms, one universal backtracking template, and 6 DP families with full recurrences. All solutions on GitHub: https://github.com/sadishihab/Leetcode"
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG Application Developer*  
*June 03, 2026*

<br>

### **Summary**

Most people learn DSA backwards — they jump to LeetCode problems before they understand **why** a hash map beats a nested loop, or what "O(log n)" actually feels like in practice. This guide builds intuition from the ground up, walking through every major data structure, algorithm, and problem-solving pattern in the order they actually make sense.

It covers **10+ core data structures**, **8 sorting algorithms**, **4 graph shortest-path algorithms**, **one universal backtracking template** that solves 90% of combination/permutation problems, and **6 DP families** with full recurrences — all built from real problem-solving sessions across hundreds of LeetCode problems ranging from easy to hard.

The goal was never to memorize solutions, but to internalize **patterns that generalize**.

🔗 **GitHub Repository:** [sadishihab/Leetcode](https://github.com/sadishihab/Leetcode)

<br>

### Table of Contents

- [1. Linear Structures](#1-linear-structures-arrays-stacks-queues--linked-lists)
- [2. Sorting Algorithms](#2-sorting-algorithms-when-to-use-what)
- [3. Binary Search](#3-binary-search-one-idea-many-forms)
- [4. Trees: Traversal, BST & BFS](#4-trees-traversal-bst-operations--bfs)
- [5. Heaps & Priority Queues](#5-heaps--priority-queues)
- [6. Hash Maps, Tries & Union-Find](#6-hash-maps-tries--union-find)
- [7. Graphs](#7-graphs-dfs-bfs-shortest-paths--mst)
- [8. Sliding Window & Two Pointers](#8-sliding-window--two-pointers)
- [9. Backtracking](#9-backtracking-the-universal-template)
- [10. Dynamic Programming](#10-dynamic-programming-from-brute-force-to-optimal)

<br>

---

## 1. Linear Structures: Arrays, Stacks, Queues & Linked Lists

Before anything else, you need a solid grip on how data is stored and accessed in memory. Every complex data structure is built on these building blocks.

| Structure | Key Operations | Notes |
|---|---|---|
| Static Array | Insert end O(1) · insert middle O(n) | Memory is contiguous |
| Dynamic Array / Stack | push O(1) · pop O(1) | Amortized resize |
| Singly Linked List | Insert at tail O(1) with tail pointer | Remove by index O(n) |
| Doubly Linked List | Insert/remove at any end O(1) | Foundation of LRU Cache |
| Queue | Enqueue at tail · dequeue at head | Use for BFS |
| Deque | O(1) push/pop from both ends | Sliding window maximum |

> **Key insight:** The main reason to choose a doubly linked list over a singly linked list is when you need O(1) removal without traversal. This is exactly why LRU Cache uses one. A singly linked list removal at a known node still requires finding the predecessor.

<br>

### Floyd's Tortoise and Hare

By running a slow pointer (1 step) and a fast pointer (2 steps), you can detect cycles, find midpoints, and locate cycle starts — all in O(n) time and O(1) space.

**Mental model:** Imagine two runners on a circular track. The faster one laps the slower one. The moment they meet, reset one runner to the start. Walk both at 1 step — they meet exactly at the cycle entrance. The math works because the distance from the head to the cycle start equals the distance from the meeting point to the cycle start.

| Problem | Technique |
|---|---|
| Find middle of linked list | slow + fast: when fast ends, slow = middle |
| Detect cycle | slow + fast: if they meet → cycle exists |
| Find start of cycle | meet inside cycle → rewind slow to head → both walk 1 step → meet at cycle start |
| Find duplicate number | treat array as linked list → same cycle detection |

<br>

### LRU Cache (LeetCode 146)

Uses a **doubly linked list + hash map** together:

- Hash map → O(1) key lookup to node
- Doubly linked list → O(1) reorder (move to head on access, remove tail on eviction)
- Head = most recently used, tail = least recently used

```python
# On get(key): move node to head
# On put(key): insert at head; if capacity exceeded, remove tail node and delete from map
```

<br>

---

## 2. Sorting Algorithms: When to Use What

Sorting is not just about getting an ordered list — it's about understanding trade-offs between time, space, and stability.

| Algorithm | Best | Average | Worst | Space | Stable |
|---|---|---|---|---|---|
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ No |
| 3-way Quick Sort | O(n) | O(n log n) | O(n²)* | O(log n) | ❌ No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ No |
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ Yes |

\* 3-way QuickSort with random pivot makes worst case astronomically rare in practice.

<br>

### 3-way QuickSort (Dutch National Flag)

> **Flashcard:** "Pick random pivot → smaller left, equal middle, larger right → recursively sort sides → done."

```
Use three pointers:
  lt → boundary of elements < pivot
  i  → current element
  gt → boundary of elements > pivot

If nums[i] < pivot → swap with lt, move both lt and i forward
If nums[i] > pivot → swap with gt, move gt backward
If nums[i] == pivot → just move i forward
```

Why 3-way over standard QuickSort? It handles duplicate-heavy inputs far better — no O(n²) degradation when all elements are the same.

<br>

### Merge Sort vs Quick Sort: The Practical Comparison

- **Merge Sort** is preferred for linked lists (splitting at midpoint is cheap, merging doesn't need random access) and for guaranteed O(n log n) regardless of input.
- **Quick Sort** wins on arrays in practice because of cache efficiency and in-place sorting, despite the theoretical worst case.

<br>

---

## 3. Binary Search: One Idea, Many Forms

Binary search is not just "find a number in a sorted array." It's a general technique for halving a search space whenever you have a **monotonic condition**.

### The Four Forms

**Form 1 — Classic binary search on array**

```python
left, right = 0, len(arr) - 1
while left <= right:
    mid = (left + right) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        left = mid + 1
    else:
        right = mid - 1
return -1
```

Time: O(log n) · Space: O(1)

<br>

**Form 2 — Binary search on a 2D matrix (LeetCode 74)**

Treat the matrix as a virtual 1D array. Index mapping:

```python
row = mid // n   # n = number of columns
col = mid % n
```

Search range: `left = 0`, `right = m * n - 1`

<br>

**Form 3 — Binary search on a condition (not a value)**

```python
# When you don't know the exact target but have a monotonic condition
while left <= right:
    mid = (left + right) // 2
    result = is_correct(mid)
    if result > 0:   # too big → go left
        right = mid - 1
    elif result < 0: # too small → go right
        left = mid + 1
    else:            # condition satisfied
        return mid
```

> **Pattern to remember:** Any time you see a problem asking for the minimum or maximum value satisfying some condition, and the condition is monotonic (once true, stays true as you increase), binary search on the answer space.

<br>

**Form 4 — Binary search on a BST**

```
Check node:
  target > root.val → go right
  target < root.val → go left
  target == root.val → found
  root is None → not found
```

Time: O(h) where h = height. O(log n) balanced, O(n) worst case.

<br>

---

## 4. Trees: Traversal, BST Operations & BFS

Trees appear in almost every category of interview problem. The key is knowing which traversal order gives you what you need.

| Traversal | Visit Order | Use Case |
|---|---|---|
| Inorder (L→Root→R) | Sorted ascending | BST validation, kth smallest |
| Preorder (Root→L→R) | Root first | Serialization, tree copying |
| Postorder (L→R→Root) | Root last | Deletion, bottom-up aggregation |
| Reverse Inorder (R→Root→L) | Sorted descending | kth largest, descending BST output |
| BFS (Level Order) | Level by level | Shortest path, right side view |

<br>

### BST Deletion — The Three-Case Rule

1. **No child** → remove directly (return None)
2. **One child** → replace node with that child
3. **Two children** → swap value with inorder successor (minimum in right subtree), then delete that successor

```python
def remove(root, val):
    if not root:
        return None
    if val < root.val:
        root.left = remove(root.left, val)
    elif val > root.val:
        root.right = remove(root.right, val)
    else:
        if not root.left:
            return root.right
        elif not root.right:
            return root.left
        # Two children: find inorder successor
        min_node = find_min(root.right)
        root.val = min_node.val
        root.right = remove(root.right, min_node.val)
    return root
```

<br>

### Why Preorder + Inorder Rebuilds a Tree (LeetCode 105)

- **Preorder** always gives the root as its first element.
- **Inorder** splits the tree: everything before the root is left subtree, everything after is right subtree.
- Apply recursively → full reconstruction. This works without any extra information because inorder position uniquely identifies the split.

<br>

### BFS Level Order Traversal

```python
from collections import deque

def bfs(root):
    if not root:
        return
    queue = deque([root])
    level = 0
    while queue:
        for _ in range(len(queue)):  # process entire level
            node = queue.popleft()
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        level += 1
```

Time: O(n) · Space: O(w) where w = max width of tree.

<br>

### DFS with Backtracking on Trees

```
Base case: root is None or root.val == 0 → return False
Include node in path: path.append(root.val)
Leaf check: if no left and no right → valid leaf → return True
Recursive: explore left, then right
Backtrack: if neither returns True → path.pop() → return False
```

<br>

---

## 5. Heaps & Priority Queues

A heap gives you the best element in O(1) and lets you insert or remove it in O(log n). That trade-off powers a surprising range of problems.

| Operation | Min-Heap | Max-Heap (Python) |
|---|---|---|
| Push | `heapq.heappush(h, x)` | `heapq.heappush(h, -x)` |
| Pop | `heapq.heappop(h)` | `-heapq.heappop(h)` |
| Peek | `h[0]` | `-h[0]` |
| Heapify | `heapq.heapify(arr)` | negate values first |
| Push then pop | `heapq.heappushpop(h, x)` | — |

Time: O(log n) per push/pop · O(1) peek · O(n) heapify

<br>

### Two Heaps — Running Median (LeetCode 295)

```
small = max-heap (negated)  → stores smaller half
large = min-heap            → stores larger half

Invariant: len(small) >= len(large), difference <= 1
           max(small) <= min(large)

Median:
  odd  → -small[0]
  even → (-small[0] + large[0]) / 2
```

**Insert algorithm:**
1. Push to `small` (max-heap)
2. If `max(small) > min(large)` → swap tops to restore order
3. If `len(small) > len(large) + 1` → move top of small to large

<br>

### Lazy Deletion — Sliding Window Median (LeetCode 480)

Heap cannot remove arbitrary elements in O(log n). The solution: mark elements as deleted in a dictionary, and physically remove them only when they surface at the heap top (lazy deletion).

```python
self.delayed = {}   # { value: count_to_delete }

def _prune(heap):
    while heap and heap[0] in self.delayed and self.delayed[heap[0]] > 0:
        self.delayed[heapq.heappop(heap)] -= 1
```

<br>

### Heap Problem Recognition

> **Signal words:** "kth largest/smallest", "top k elements", "median from data stream", "sliding window median", "task scheduling with priorities"

The pattern: you need the best element repeatedly, and the set keeps changing.

<br>

---

## 6. Hash Maps, Tries & Union-Find

These three structures share one superpower: near-instant answers to membership and connectivity questions.

<br>

### Hash Map

Average O(1) insert, lookup, delete. Built on a hash function + collision resolution.

**Separate chaining with DELETED marker:**

- Hash function: `index = sum(ord(c) for c in key) % capacity`
- On deletion: replace slot with `DELETED` (not `None`) to avoid breaking the probing chain
- On search: stop at `None` (not found), skip over `DELETED` slots
- Rehash when load factor ≥ 0.5 (double capacity, reinsert)

**When to use:** counting frequencies, two-sum style problems, caching, "have I seen this before?", grouping by key.

<br>

### Trie (Prefix Tree) — LeetCode 208

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.word = False        # marks a complete word endpoint

class Trie:
    def insert(self, word):      # walk + create nodes, set word=True at end
    def search(self, word):      # walk + check word flag at final node
    def startsWith(self, prefix): # walk only — no word flag check needed
```

> **Critical:** A path existing ≠ a word existing. `search("ca")` returns False even if "car" and "cat" are inserted. The `word` flag is what separates a prefix from a complete word.

**Advanced Trie problems:**
- `WordDictionary` (LeetCode 211): add DFS at `.` wildcard nodes
- `Word Search II` (LeetCode 212): Trie pruning + backtracking DFS on grid
- `Prefix & Suffix Search` (LeetCode 745): insert `suffix + "#" + word`, query `suffix + "#" + prefix`

<br>

### Union-Find (Disjoint Set Union)

```python
parent = list(range(n))
rank = [0] * n

def find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])   # path compression
    return parent[x]

def union(x, y):
    rx, ry = find(x), find(y)
    if rx == ry:
        return False                   # already connected → cycle detected
    if rank[rx] < rank[ry]:
        parent[rx] = ry
    else:
        parent[ry] = rx
        if rank[rx] == rank[ry]:
            rank[rx] += 1
    return True
```

Time: O(α(n)) ≈ O(1) per operation with both optimizations.

**When to use:** cycle detection, number of connected components, dynamic connectivity, Kruskal's MST, redundant connections, grid island merging, accounts merge.

<br>

---

## 7. Graphs: DFS, BFS, Shortest Paths & MST

Graph algorithms are the most powerful — and most feared — category in competitive programming. The key is knowing which tool fits each problem type.

| Algorithm | Use Case | Complexity |
|---|---|---|
| DFS (recursive) | Cycle detection, path counting, topological sort | O(V+E) |
| BFS (queue) | Shortest path (unweighted), level order | O(V+E) |
| Dijkstra (min-heap) | Shortest path (weighted, non-negative) | O((V+E) log V) |
| Prim's (min-heap) | Minimum Spanning Tree | O(E log V) |
| Kruskal's (Union-Find) | Minimum Spanning Tree | O(E log E) |
| Topological Sort (DFS) | DAG ordering, course schedule | O(V+E) |

<br>

### Dijkstra — The Core Pattern

```python
dist = {i: float('inf') for i in range(1, n+1)}
dist[src] = 0
minHeap = [(0, src)]

while minHeap:
    currDist, node = heapq.heappop(minHeap)
    if currDist > dist[node]:   # stale entry — skip
        continue
    for nei, weight in adj[node]:
        newDist = currDist + weight
        if newDist < dist[nei]:
            dist[nei] = newDist
            heapq.heappush(minHeap, (newDist, nei))
```

> **Key:** When you pop a node, if its recorded distance is stale (someone already found a better path), skip it. This lazy deletion keeps the implementation clean without a decrease-key operation.

<br>

### Prim's vs Kruskal's

| | Prim's | Kruskal's |
|---|---|---|
| Strategy | Grow one tree outward from seed node | Sort all edges, add cheapest non-cycle edge |
| Data structure | Min-heap (weight, node, parent) | Union-Find |
| Better for | Dense graphs (E ≈ V²) | Sparse graphs |
| Returns | MST edges + total cost | MST total weight |

<br>

### Topological Sort — DFS with 3-State Cycle Detection

```python
state = [0] * n   # 0=unvisited, 1=visiting, 2=finished

def dfs(node):
    if state[node] == 1:  return False   # back edge → cycle
    if state[node] == 2:  return True    # already processed
    state[node] = 1
    for nei in adj[node]:
        if not dfs(nei): return False
    state[node] = 2
    res.append(node)
    return True

for i in range(n):
    if not dfs(i): return []

return res[::-1]   # reverse because nodes appended in postorder
```

Applications: Course Schedule (LC 207), Course Schedule II (LC 210), Sort Items by Groups (LC 1203).

<br>

### Grid BFS — Shortest Path Pattern

```python
from collections import deque

directions = [(1,0),(-1,0),(0,1),(0,-1)]   # 4-directional
# For 8-directional (LeetCode 1091): add diagonals

queue = deque([(0, 0)])
visited = set([(0, 0)])
length = 1

while queue:
    for _ in range(len(queue)):
        r, c = queue.popleft()
        if r == ROWS-1 and c == COLS-1:
            return length
        for dr, dc in directions:
            nr, nc = r+dr, c+dc
            if 0 <= nr < ROWS and 0 <= nc < COLS and (nr,nc) not in visited:
                visited.add((nr,nc))
                queue.append((nr,nc))
    length += 1
return -1
```

BFS guarantees shortest path in unweighted grids because every move has equal cost.

<br>

### DFS on Grid — Count Paths / Islands

```python
def dfs(r, c):
    if r < 0 or c < 0 or r >= ROWS or c >= COLS:  return 0
    if grid[r][c] == '0' or (r,c) in visited:     return 0
    visited.add((r,c))
    return 1 + dfs(r+1,c) + dfs(r-1,c) + dfs(r,c+1) + dfs(r,c-1)
```

<br>

---

## 8. Sliding Window & Two Pointers

These two techniques convert O(n²) brute-force solutions into clean O(n) ones.

<br>

### Fixed-Size Sliding Window

Window size k is constant. Slide right: add new element, remove left element, check condition.

```python
# Pattern: number of subarrays of size k with average >= threshold
window_sum = sum(arr[:k])
target = threshold * k
count = 1 if window_sum >= target else 0

for i in range(k, len(arr)):
    window_sum += arr[i] - arr[i - k]
    if window_sum >= target:
        count += 1
```

Time: O(n) · Space: O(1)

<br>

### Variable-Size Sliding Window

Expand right when condition not met. Shrink left when condition satisfied (to minimize window).

```python
# Pattern: minimum size subarray sum >= target
L, total, min_len = 0, 0, float('inf')
for R in range(len(nums)):
    total += nums[R]
    while total >= target:
        min_len = min(min_len, R - L + 1)
        total -= nums[L]
        L += 1
return min_len if min_len != float('inf') else 0
```

Works because all numbers are positive → expanding R always increases sum, shrinking L always decreases it.

<br>

### Window + Hash Set (Longest Substring Without Repeating Characters)

```python
window = set()
L, max_len = 0, 0
for R in range(len(s)):
    while s[R] in window:
        window.remove(s[L])
        L += 1
    window.add(s[R])
    max_len = max(max_len, R - L + 1)
```

<br>

### Two Pointers on Sorted Array

```python
# Two Sum II — sorted input
L, R = 0, len(numbers) - 1
while L < R:
    s = numbers[L] + numbers[R]
    if s == target:    return [L+1, R+1]
    elif s < target:   L += 1
    else:              R -= 1
```

> **Pattern:** sum too large → move right pointer left. Sum too small → move left pointer right.

<br>

### Prefix Sum

Precompute cumulative sums once → answer any range sum query in O(1).

```python
# 1D prefix sum
prefix = []
total = 0
for x in nums:
    total += x
    prefix.append(total)

def range_sum(left, right):
    return prefix[right] - (prefix[left-1] if left > 0 else 0)
```

**2D prefix sum** (LeetCode 304):

```
prefix[i][j] = sum of rectangle (0,0) → (i,j)

Query (r1,c1) → (r2,c2):
  prefix[r2][c2] - prefix[r1-1][c2] - prefix[r2][c1-1] + prefix[r1-1][c1-1]
```

**Subarray Sum Equals K (LeetCode 560):**

```python
# cum_sum - k seen before → valid subarray ending here
prefix_counts = {0: 1}
cum_sum = count = 0
for num in nums:
    cum_sum += num
    count += prefix_counts.get(cum_sum - k, 0)
    prefix_counts[cum_sum] = prefix_counts.get(cum_sum, 0) + 1
```

Works with negative numbers. Sliding window does not.

<br>

---

## 9. Backtracking: The Universal Template

Backtracking is DFS on a decision tree. Every backtracking problem — subsets, combinations, permutations, combination sum, phone letter combos — uses exactly the same skeleton.

```python
res = []

def backtrack(start, path):
    # 1. Base case — goal reached
    if goal_reached:
        res.append(path.copy())
        return

    # 2. Choice loop
    for i in range(start, len(choices)):
        path.append(choices[i])      # choose
        backtrack(next_start, path)  # explore
        path.pop()                   # un-choose (backtrack)

backtrack(0, [])
```

<br>

### Decision Table

| Problem Type | Key Signal | `next_start` | Extra Logic |
|---|---|---|---|
| Subsets | All combinations of any size | `i + 1` | — |
| Combinations (size k) | Fixed-size selection | `i + 1` | Stop when `len(path) == k` |
| Permutations | Order matters | `0` | `used[]` array to track taken elements |
| Combination Sum | Reuse allowed | `i` (same index) | Stop when sum > target |
| Subsets with duplicates | Duplicates in input | `i + 1` | Sort + skip `nums[i] == nums[i-1]` on exclude |
| Permutations with duplicates | Duplicates in input | `0` | Sort + skip if `used[i-1] == False and nums[i] == nums[i-1]` |
| Phone combinations | Fixed-length, multiple groups | `index + 1` | Map digits to letters |

<br>

### The Mental Model That Never Fails

> Backtracking is tree traversal. Each level of the tree is one decision. Each branch is one choice. DFS explores branches. `path.pop()` goes back up the tree.

```
nums = [1,2,3]

                   []
             /      |      \
           [1]     [2]     [3]
          /   \      \
       [1,2] [1,3]  [2,3]
         |
      [1,2,3]
```

`path.append()` = go DOWN the tree. `path.pop()` = go UP the tree.

<br>

### Swap-Based Permutations

An alternative to the `used[]` approach — fix one position at a time by swapping:

```python
def backtrack(start):
    if start == len(nums):
        res.append(nums[:])
        return
    for i in range(start, len(nums)):
        nums[start], nums[i] = nums[i], nums[start]   # choose
        backtrack(start + 1)                            # explore
        nums[start], nums[i] = nums[i], nums[start]   # undo (CRITICAL)
```

<br>

---

## 10. Dynamic Programming: From Brute Force to Optimal

DP is not a technique you apply — it's a way of thinking about overlapping subproblems. The journey from brute force to optimal follows the same path every time.

### The Three Stages

**Stage 1 — Brute Force (naive recursion)**

Directly translate the recurrence. O(2ⁿ). Recomputes everything. Good for understanding the problem structure, bad for submitting.

**Stage 2 — Memoization (top-down DP)**

Cache computed results in a dictionary. Same recursive structure, but each unique subproblem is solved once. O(n) time, O(n) space.

**Stage 3 — Bottom-up DP (tabulation)**

Build from base cases iteratively. No recursion stack. Often allows space optimization.

```python
# Fibonacci: O(2^n) → O(n) → O(1) space
# Bottom-up with O(1) space:
a, b = 0, 1
for _ in range(2, n+1):
    a, b = b, a + b
return b
```

<br>

### DP Families and Recurrences

| Problem | DP State | Recurrence | Traversal Direction |
|---|---|---|---|
| 0/1 Knapsack | `dp[c]` = max profit with capacity c | `dp[c] = max(dp[c], profit + dp[c - weight])` | **Backward** (each item once) |
| Unbounded Knapsack | `dp[c]` = max profit with capacity c | `dp[c] = max(dp[c], profit + dp[c - weight])` | **Forward** (reuse allowed) |
| Coin Change (min) | `dp[c]` = min coins for amount c | `dp[c] = min(dp[c], 1 + dp[c - coin])` | Forward, init `dp[0]=0`, rest `inf` |
| Coin Change II (count) | `dp[c]` = number of ways for amount c | `dp[c] += dp[c - coin]` | Forward, init `dp[0]=1` |
| LCS | `dp[i][j]` = LCS of prefixes | match: `dp[i-1][j-1]+1` · else: `max(top, left)` | 2D fill |
| Edit Distance | `dp[i][j]` = ops to convert prefixes | match: `dp[i-1][j-1]` · else: `1+min(del,ins,rep)` | 2D fill |
| Longest Palindrome Subseq | `dp[i][j]` = LPS in `s[i..j]` | match: `2+dp[i+1][j-1]` · else: `max(skip L, skip R)` | Bottom-up, short → long |
| Unique Paths | `dp[r][c]` = paths from here | `dp[r][c] = dp[r+1][c] + dp[r][c+1]` | Bottom-right → top-left |

<br>

### The Traversal Direction Rule

> **This single rule separates coin change from subset sum:**
>
> - **Backward** (capacity → 0): each item used **at most once** → 0/1 knapsack
> - **Forward** (0 → capacity): items can be **reused** → unbounded knapsack

<br>

### Common DP Problem Recognitions

| Keywords | Problem Type |
|---|---|
| "partition into two equal subsets" | 0/1 knapsack with `target = total // 2` |
| "minimum coins to make amount" | Unbounded knapsack, minimize |
| "number of ways to make amount" | Unbounded knapsack, count (dp[0]=1) |
| "assign + or - to reach target" | 0/1 knapsack count, `capacity = (total - target) / 2` |
| "longest common subsequence" | 2D LCS table |
| "minimum edit operations" | 2D edit distance table |
| "longest increasing subsequence" | 1D DP O(n²), or segment tree O(n log n) |
| "number of unique paths in grid" | 2D paths DP, optimize to O(cols) space |

<br>

### Segment Tree for Range Queries

When the array is **mutable** and you need both range queries and point updates, prefix sums are insufficient. Use a segment tree.

```
Build:  O(n)
Update: O(log n)   — touch only root → leaf path, fix sums while returning
Query:  O(log n)   — visit only overlapping segments
Space:  O(n)       — store in array of size 4*n
```

Used in: Range Sum Query Mutable (LeetCode 307), Longest Increasing Subsequence II (LeetCode 2407).

<br>

---

## What We Covered

| Category | Count |
|---|---|
| Core data structures | 10+ |
| Sorting algorithms compared | 8 |
| Graph algorithms | 4 (DFS, BFS, Dijkstra, Prim, Kruskal, Topological Sort) |
| Backtracking template variants | 7 |
| DP families with recurrences | 8 |
| LeetCode problems patterns | 50+ |

<br>

---

## Key Takeaways

1. **Pattern over memorization.** Every backtracking problem is the same template with one line changed. Every knapsack variant flips one traversal direction. Learn the pattern, not the solution.

2. **Complexity drives choice.** Hash map O(1) beats nested loop O(n²). BFS guarantees shortest path in unweighted graphs. Dijkstra handles weights. Knowing when to use what is more valuable than knowing how to implement everything.

3. **Traversal direction is the most consequential single decision in DP.** Backward = each item once. Forward = reuse allowed. Get this wrong and your solution is subtly broken on every test case.

4. **Floyd's algorithm is elegance in code.** One slow pointer, one fast pointer, and you can solve cycle detection, midpoint finding, and cycle start location — all O(n) time, O(1) space.

5. **The decision tree mental model solves backtracking.** Stop thinking about what the code does. Draw the tree. Write DFS. `path.append()` = go down. `path.pop()` = go back up.

<br>

---

### References

- [LeetCode Problem Set](https://leetcode.com/problemset/)
- [CLRS — Introduction to Algorithms](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
- [Python heapq documentation](https://docs.python.org/3/library/heapq.html)
- [FAISS — Facebook AI Similarity Search](https://github.com/facebookresearch/faiss)

<br>

### License

This blog post is open-source and available under the MIT License.
