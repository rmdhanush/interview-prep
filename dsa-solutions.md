# DSA Solutions — Patterns & Approaches

> Quick-reference for revision. Pattern, approach in plain English, complexity, and when to reuse.

---

## Patterns Index

| Pattern | Problems |
|---------|----------|
| Hash Map Lookup | [Two Sum](#two-sum) |
| Stack (LIFO) | [Valid Parentheses](#valid-parentheses) |
| Sort + Two Pointers | [3Sum](#3sum) |
| Dummy Node + Merge | [Merge Two Sorted Lists](#merge-two-sorted-lists) |

---

## Two Sum

**LeetCode #1 (Easy) | Pattern: Hash Map Lookup | Time: O(n) Space: O(n)**

**Algorithm:**
1. Create a map: `value → index`
2. For each number, compute `complement = target - num`
3. If complement exists in map → return both indices
4. Otherwise, store current number in map

**Why it works:** One pass. For each number, ask "have I already seen the number that completes the pair?" Map gives O(1) lookup.

**Reuse when:** "Find two things that satisfy a condition" → check if the complement exists in a map.

---

## Valid Parentheses

**LeetCode #20 (Easy) | Pattern: Stack (LIFO) | Time: O(n) Space: O(n)**

**Algorithm:**
1. Create a map of closing → opening brackets: `)→(`, `]→[`, `}→{`
2. For each character:
   - Opening bracket → push to stack
   - Closing bracket → pop from stack, must match. If stack empty or mismatch → false
3. At end, stack must be empty (every opener had a closer)

**Why it works:** Most recent opening bracket must match the next closing bracket — that's exactly LIFO (stack) behavior.

**Reuse when:** Matching/nesting problems — parentheses, HTML tags, nested structures. Stack = "remember the most recent thing I need to come back to."

---

## 3Sum

**LeetCode #15 (Medium) | Pattern: Sort + Fix One + Two Pointers | Time: O(n²) Space: O(1)**

**Algorithm:**
1. Sort the array
2. For each `i` (0 to n-2):
   - Skip if `nums[i] == nums[i-1]` (avoid duplicate triplets)
   - Set `target = -nums[i]`, `j = i+1`, `k = last index`
   - Two pointer loop while `j < k`:
     - `sum > target` → k--
     - `sum < target` → j++
     - `sum == target` → record triplet, skip duplicate j and k, move both inward

**Why it works:** Sorting enables two pointers. Fix one number, reduce to 2Sum on the remaining sorted subarray. Duplicate skipping at i, j, k prevents repeated triplets.

**Reuse when:** "Find X numbers that sum to Y" → Sort + fix (X-2) numbers + two pointers for last two. Works for 2Sum II, 3Sum, 4Sum.

---

## Merge Two Sorted Lists

**LeetCode #21 (Easy) | Pattern: Dummy Node + Two Pointer Merge | Time: O(n+m) Space: O(1)**

**Algorithm:**
1. Create a dummy node (fake head) and a `curr` pointer
2. While both lists have nodes:
   - Compare heads, attach the smaller one to `curr.Next`
   - Advance that list's pointer
   - Advance `curr`
3. Attach whichever list still has remaining nodes
4. Return `dummy.Next` (skip the fake head)

**Why it works:** Dummy node avoids all "what if head changes" edge cases. Always pick the smaller node. When one list runs out, the remainder is already sorted — just attach.

**Reuse when:** Any linked list problem where the head might change or you're building a new list. Also: Merge K Sorted Lists, Partition List, Sort List.

---

<!-- Add new problems below this line -->
