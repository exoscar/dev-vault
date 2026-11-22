# Complete DSA Roadmap for Java Backend Interviews

*Target: product-based companies, 2 years of experience, 20 LPA+. Built for a daily 5-question habit alongside HLD, LLD and Java prep.*

---

## 1. The goal: what "good enough" looks like

You do not need to be a competitive programmer. By the end of this roadmap you should be able to:

- Solve most **standard mediums in 25–35 minutes** while explaining your thinking out loud.
- Recognize the 15–18 core patterns within the first 2 minutes of reading a problem.
- State time and space complexity, and discuss at least one alternative approach.
- Solve **some hards** (DP on strings/trees, advanced graphs) as a bonus for top-paying companies.

**Volume target:** about 250–300 problems done *properly* (pattern noted, redone after a few days), plus 8–10 weekly contests. This beats 600 problems done once.

---

## 2. How to practice (the system matters more than the list)

### The per-problem routine

1. **Read and restate** the problem. Write 2 examples and 1 edge case before touching code.
2. **Think for 10–15 minutes.** Brute force first, then find the bottleneck.
3. **Code for 15–20 minutes.** Dry run on your examples.
4. **Timebox: 30–40 minutes total.** If stuck, read the hint or solution, understand it fully, close it, and write the code yourself.
5. **Log it** in your tracker: pattern, key insight, mistake made, difficulty, date.
6. **Redo from scratch** if you needed the solution.

### Spaced revision (non-negotiable)

Revisit every problem you struggled with after **day 3, day 7, and day 21**. Re-solve without hints. Anything you fail again goes back into the queue. Use 1 hour of every weekday for this.

### Tracker columns

| Problem | Pattern | Difficulty | Time taken | Needed hint? | Key insight | Mistake | Revision dates |
| --- | --- | --- | --- | --- | --- | --- | --- |

A Notion page, Google Sheet or plain Markdown file all work. Review the "Mistake" column every Sunday.

### Daily volume ramp

- **Weeks 1–4:** 3–4 problems/day, done properly with notes. Quality over quantity while you build the habit.
- **Weeks 5 onward:** 5 problems/day (2 new-topic, 2 mixed/revision, 1 stretch).
- **Weekends:** 3 hours Saturday on the week's weak pattern, plus the Sunday weekly contest and upsolving.

---

## 3. Java toolkit (week 0, 3–4 days)

Before starting, make sure these are fluent so syntax never slows you down:

- `ArrayList`, `LinkedList`, `ArrayDeque` (use it instead of `Stack` and `Queue` implementations), `HashMap`, `HashSet`, `TreeMap`, `TreeSet`, `PriorityQueue`.
- Comparators: `(a, b) -> a[0] - b[0]` (beware of overflow, use `Integer.compare`), `Comparator.comparingInt(...)`.
- `PriorityQueue` for max-heap: `new PriorityQueue<>(Collections.reverseOrder())`.
- `StringBuilder` for string building, `char[]` for in-place manipulation.
- `int` overflow, `long` casts, `Integer.MAX_VALUE` pitfalls, integer division and modulo with negatives.
- `Arrays.fill`, `Arrays.sort` on 2D arrays, `Collections.sort` stability, `List.toArray`.
- Recursion depth: Java's default stack can overflow on \~10⁴–10⁵ deep recursion. Know when to go iterative.
- Complexity basics: Big-O for common operations (HashMap O(1) average, TreeMap O(log n), PriorityQueue offer/poll O(log n)).

---

## 4. The 16-week roadmap

Each week lists the concept to learn first, then problems in suggested order. Problems marked ⭐ are must-dos. The rest are core/stretch. If you fall behind, drop stretch problems, not revision.

### Phase 1: Foundations (Weeks 1–6)

#### Week 1: Arrays and Hashing

**Concepts:** frequency maps, prefix sums, hash sets for membership, sorting for grouping.

- ⭐ Two Sum
- ⭐ Contains Duplicate
- ⭐ Valid Anagram
- ⭐ Group Anagrams
- ⭐ Top K Frequent Elements
- ⭐ Product of Array Except Self
- ⭐ Longest Consecutive Sequence
- ⭐ Subarray Sum Equals K (prefix sum + hashmap, a very common pattern)
- Encode and Decode Strings
- Valid Sudoku
- Majority Element, Rotate Array, Set Matrix Zeroes, Spiral Matrix

#### Week 2: Two Pointers and Sliding Window

**Concepts:** opposite-end pointers on sorted data, fast/slow, fixed-size window, variable-size window (expand right, shrink left while invalid).

- ⭐ Valid Palindrome
- ⭐ Two Sum II
- ⭐ 3Sum
- ⭐ Container With Most Water
- Trapping Rain Water
- ⭐ Best Time to Buy and Sell Stock
- ⭐ Longest Substring Without Repeating Characters
- ⭐ Longest Repeating Character Replacement
- ⭐ Permutation in String
- ⭐ Minimum Window Substring
- Sliding Window Maximum (monotonic deque)
- Max Consecutive Ones III, Fruit Into Baskets

#### Week 3: Stack, Queue and Binary Search

**Concepts:** matching/nesting with stacks, monotonic stack, binary search on index and on the answer space.

- ⭐ Valid Parentheses
- ⭐ Min Stack
- ⭐ Evaluate Reverse Polish Notation
- ⭐ Daily Temperatures
- ⭐ Next Greater Element I/II
- Car Fleet
- ⭐ Largest Rectangle in Histogram
- ⭐ Binary Search
- ⭐ Search in Rotated Sorted Array
- ⭐ Find Minimum in Rotated Sorted Array
- ⭐ Koko Eating Bananas (binary search on answer)
- ⭐ Capacity to Ship Packages Within D Days
- Search a 2D Matrix, Time Based Key-Value Store
- Median of Two Sorted Arrays (hard, stretch)

#### Week 4: Linked Lists

**Concepts:** dummy nodes, fast/slow pointers, in-place reversal, merging.

- ⭐ Reverse Linked List
- ⭐ Merge Two Sorted Lists
- ⭐ Linked List Cycle (and Cycle II)
- ⭐ Middle of the Linked List
- ⭐ Remove Nth Node From End
- ⭐ Reorder List
- ⭐ Add Two Numbers
- ⭐ Copy List with Random Pointer
- ⭐ LRU Cache (doubly linked list + hashmap; also an LLD favorite)
- ⭐ Merge k Sorted Lists
- Reverse Nodes in k-Group (hard, stretch)
- Find the Duplicate Number

#### Week 5: Trees

**Concepts:** DFS (pre/in/post-order), BFS level order, returning values up the recursion, BST properties.

- ⭐ Invert Binary Tree
- ⭐ Maximum Depth of Binary Tree
- ⭐ Diameter of Binary Tree
- ⭐ Balanced Binary Tree
- ⭐ Same Tree / Subtree of Another Tree
- ⭐ Lowest Common Ancestor (BST and Binary Tree)
- ⭐ Binary Tree Level Order Traversal
- ⭐ Binary Tree Right Side View
- ⭐ Count Good Nodes in Binary Tree
- ⭐ Validate Binary Search Tree
- ⭐ Kth Smallest Element in a BST
- ⭐ Construct Binary Tree from Preorder and Inorder Traversal
- ⭐ Binary Tree Maximum Path Sum (hard)
- Serialize and Deserialize Binary Tree (hard)
- Zigzag Level Order, Path Sum II, Flatten Binary Tree to Linked List

#### Week 6: Heaps, Intervals and Tries

**Concepts:** top-K with heaps, two-heap median trick, sort-then-sweep for intervals, prefix trees.

- ⭐ Kth Largest Element in a Stream
- ⭐ Last Stone Weight
- ⭐ K Closest Points to Origin
- ⭐ Kth Largest Element in an Array
- ⭐ Task Scheduler
- ⭐ Find Median from Data Stream (hard)
- ⭐ Merge Intervals
- ⭐ Insert Interval
- ⭐ Non-overlapping Intervals
- ⭐ Meeting Rooms I and II
- ⭐ Implement Trie (Prefix Tree)
- ⭐ Design Add and Search Words Data Structure
- Word Search II (hard, stretch)

**Checkpoint (end of Week 6):** about 90–110 problems done. Redo any problem you couldn't solve in the last 3 weeks. Do one timed mock: 2 mediums in 60 minutes.

---

### Phase 2: Depth (Weeks 7–12)

#### Week 7: Backtracking

**Concepts:** choose / explore / un-choose, pruning, handling duplicates by sorting and skipping.

- ⭐ Subsets, Subsets II
- ⭐ Permutations, Permutations II
- ⭐ Combination Sum, Combination Sum II
- ⭐ Word Search
- ⭐ Palindrome Partitioning
- ⭐ Letter Combinations of a Phone Number
- ⭐ Generate Parentheses
- N-Queens (hard, stretch)

#### Week 8: Graphs I (BFS, DFS, Topological Sort, Union-Find)

**Concepts:** adjacency list construction, grid as a graph, visited sets, multi-source BFS, cycle detection, topological order (Kahn's algorithm), disjoint set union.

- ⭐ Number of Islands
- ⭐ Max Area of Island
- ⭐ Clone Graph
- ⭐ Rotting Oranges (multi-source BFS)
- ⭐ Pacific Atlantic Water Flow
- ⭐ Surrounded Regions
- ⭐ Course Schedule I and II
- ⭐ Number of Connected Components in an Undirected Graph
- ⭐ Graph Valid Tree
- ⭐ Redundant Connection (union-find)
- ⭐ Word Ladder (hard, BFS)
- Accounts Merge, Is Graph Bipartite, Walls and Gates

#### Week 9: Graphs II (Shortest Paths and MST)

**Concepts:** Dijkstra with a priority queue, Bellman-Ford for limited steps, Prim/Kruskal for minimum spanning tree.

- ⭐ Network Delay Time (Dijkstra)
- ⭐ Cheapest Flights Within K Stops
- ⭐ Path With Minimum Effort / Swim in Rising Water
- ⭐ Min Cost to Connect All Points (MST)
- ⭐ Alien Dictionary (topological sort, hard)
- Reconstruct Itinerary (hard)
- Shortest Path in Binary Matrix

#### Weeks 10–11: Dynamic Programming

Two weeks, because DP is where most candidates lose marks. Always follow this process: define the state, write the recurrence, find base cases, start with recursion + memoization, then convert to tabulation, then optimize space.

**Week 10: 1D DP and knapsack-style**

- ⭐ Climbing Stairs
- ⭐ House Robber I and II
- ⭐ Min Cost Climbing Stairs
- ⭐ Longest Palindromic Substring and Palindromic Substrings
- ⭐ Decode Ways
- ⭐ Coin Change
- ⭐ Maximum Product Subarray
- ⭐ Word Break
- ⭐ Longest Increasing Subsequence
- ⭐ Partition Equal Subset Sum (0/1 knapsack)
- Jump Game (also greedy), Maximum Subarray (Kadane)

**Week 11: 2D DP, strings, grids**

- ⭐ Unique Paths I and II, Minimum Path Sum
- ⭐ Longest Common Subsequence
- ⭐ Edit Distance
- ⭐ Coin Change II
- ⭐ Target Sum
- ⭐ Best Time to Buy and Sell Stock with Cooldown (state machine DP)
- ⭐ Interleaving String
- Distinct Subsequences, Longest Increasing Path in a Matrix
- Burst Balloons, Regular Expression Matching (hard, stretch)

#### Week 12: Greedy, Bit Manipulation and Math

**Greedy:**

- ⭐ Maximum Subarray
- ⭐ Jump Game I and II
- ⭐ Gas Station
- ⭐ Hand of Straights
- ⭐ Partition Labels
- Merge Triplets to Form Target Triplet, Valid Parenthesis String

**Bit manipulation and math:**

- ⭐ Single Number
- ⭐ Number of 1 Bits
- ⭐ Counting Bits
- ⭐ Reverse Bits
- ⭐ Missing Number
- Sum of Two Integers, Pow(x, n), Rotate Image

**Checkpoint (end of Week 12):** about 200–220 problems done. Take two timed mocks (2 problems in 45 minutes each). Identify your three weakest patterns and give them a dedicated deep-dive in Phase 3.

---

### Phase 3: Interview mode (Weeks 13–16)

#### Weeks 13–14: Mixed practice and harder problems

- Stop practicing by topic. Do **mixed sets**: pick 3 random mediums and give yourself 75 minutes.
- Add 1 hard a day for stretch, even if you only get partial progress.
- Add **company-tagged problems** for your target list (LeetCode Premium tags, or community lists on GitHub).
- Advanced patterns to cover now: monotonic deque, segment tree / Fenwick tree basics (optional), Kadane variants, DP on trees, bitmask DP basics, Dijkstra variants, sliding window with hashmap of counts.
- Practice the **LLD-adjacent DSA** classics: LRU Cache, LFU Cache, Min Stack, Design Hit Counter, Time Based Key-Value Store, Insert Delete GetRandom O(1), Design Twitter.

#### Weeks 15–16: Mocks, revision and polish

- 2–3 live mock interviews per week (friends, Pramp, Interviewing.io). Record yourself or get written feedback.
- Re-solve every problem in your tracker marked "struggled".
- Do a final pass over your **pattern cheat sheet** (section 6).
- Practice communication: clarify, propose brute force, optimize, code, test, state complexity.
- Keep solving 2–3 problems a day so you stay sharp. Do not cram the last 2 days. Rest, revise notes, sleep.

---

## 5. Weekly rhythm for DSA

| Day | DSA task |
| --- | --- |
| Mon–Fri | 3–5 problems daily (new topic + mixed), 1 hour revision |
| Saturday | 3 hours: deep dive on the week's weakest pattern, redo wrong problems |
| Sunday | LeetCode weekly contest (morning, IST), upsolve the problems you missed, 1 hour revision |
| Every 4th Sunday | Review your tracker, update your weak-pattern list, plan the next month |

---

## 6. Pattern recognition cheat sheet

Use this when you read a problem and aren't sure where to start.

| Clue in the problem | Likely pattern |
| --- | --- |
| Sorted array, find pair/triplet | Two pointers |
| Longest/shortest substring or subarray with a condition | Sliding window |
| "Minimize the maximum" / "smallest X such that..." | Binary search on the answer |
| Next greater/smaller element | Monotonic stack |
| Top K, K-th largest, merge K sorted | Heap |
| Running median | Two heaps |
| Overlapping ranges | Sort + sweep (intervals) |
| All combinations/permutations/subsets | Backtracking |
| Grid/matrix with connected regions | BFS/DFS flood fill |
| Shortest path, unweighted | BFS |
| Shortest path, weighted | Dijkstra |
| Dependencies/ordering | Topological sort |
| Connected components, merging groups | Union-Find |
| Prefix matching, dictionary of words | Trie |
| Count ways / min cost / max profit with overlapping subproblems | Dynamic programming |
| Choose locally best and it provably works | Greedy |
| Subarray sum equals K | Prefix sum + hashmap |
| Cycle in linked list, middle node | Fast/slow pointers |
| Detect duplicates, frequency | HashMap / HashSet |

---

## 7. Interview-day communication framework

Follow the same script every time. It scores marks even when you don't reach the optimal solution.

1. **Clarify:** input size, duplicates, negatives, empty input, sorted or not.
2. **Examples:** walk through one normal case and one edge case.
3. **Brute force:** state it and its complexity, even if briefly.
4. **Optimize:** identify the bottleneck and the pattern. Explain *why* it works.
5. **Code:** clean variable names, small helper functions.
6. **Dry run:** trace your example through the code.
7. **Complexity:** time and space, and mention trade-offs.

---

## 8. Common mistakes to avoid

- **Reading solutions too early.** Struggle for the full timebox first. That struggle is where learning happens.
- **Never revising.** Solving a problem once means little. The 3/7/21 day revision is what makes patterns stick.
- **Switching resources constantly.** Pick one roadmap (NeetCode 150 or Striver's sheet) and finish it.
- **Skipping easy problems for "harder" ones.** Easy problems teach the fundamentals. Do them quickly, but do them.
- **Coding without a plan.** Spend the first few minutes on approach, not syntax.
- **Ignoring edge cases.** Empty input, a single element, duplicates, negative numbers, overflow.
- **Doing 600 problems with no notes.** Volume without reflection plateaus fast.
- **Avoiding DP and graphs.** These are the topics where most candidates are weakest, so they differentiate you.

---

## 9. Recommended resources

- **NeetCode 150 / NeetCode roadmap:** best structured list with video explanations, good for patterns.
- **Striver's A2Z DSA Sheet / SDE Sheet:** comprehensive, with a strong Indian-interview focus.
- **LeetCode explore cards and Discuss section:** for alternative solutions after you've attempted a problem.
- **LeetCode weekly contests:** for speed and pressure practice.
- **Company-tagged lists (LeetCode Premium or GitHub community lists):** use in weeks 13–16.

Pick **one** main sheet (NeetCode 150 or Striver) and use the others only for extra practice on weak topics.

---

## 10. Milestones summary

| Week | Expected total solved | Check |
| --- | --- | --- |
| 2 | \~35 | Comfortable with hashmaps, two pointers, sliding window |
| 4 | \~65 | Binary search and linked lists feel routine |
| 6 | \~100 | Trees and heaps solid; first timed mock done |
| 8 | \~140 | Backtracking and basic graphs comfortable |
| 10 | \~170 | 1D DP patterns recognized |
| 12 | \~210 | All core topics covered; weak-pattern list ready |
| 14 | \~250 | Mixed timed sets at about 30 min per medium |
| 16 | \~280+ | Ready to interview; mocks going well |

These numbers assume the daily ramp in section 2. If you're behind, extend Phase 2 by a week or two rather than rushing. DSA is the interview filter, so solidity matters more than finishing on a fixed date.

---

## 11. Final checklist before you apply

- [ ] Can solve most mediums within 30–35 minutes without hints
- [ ] Confident in arrays/hashing, sliding window, binary search, trees, graphs (BFS/DFS) and basic DP
- [ ] Tracker shows no major pattern with repeated failures
- [ ] Completed at least 5 timed mock interviews
- [ ] Can explain complexity and alternatives for every solution
- [ ] Java collections, comparators and recursion limits are second nature
- [ ] Still solving 2–3 problems daily to stay sharp
