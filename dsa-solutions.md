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
| Prefix + Suffix Arrays | [Product of Array Except Self](#product-of-array-except-self) |
| Sort + Hash Map Grouping | [Group Anagrams](#group-anagrams) |
| Monotonic Deque | [Sliding Window Maximum](#sliding-window-maximum) |
| Sliding Window + Hash Map | [Longest Substring Without Repeating Characters](#longest-substring-without-repeating-characters) |

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

## Product of Array Except Self

**LeetCode #238 (Medium) | Pattern: Prefix + Suffix Arrays | Time: O(n) Space: O(n)**

**Algorithm:**
1. Build `prefix[]` — prefix[i] = product of all elements **before** i
   - prefix[0] = 1 (nothing before first element)
   - prefix[i] = prefix[i-1] * nums[i-1]
2. Build `suffix[]` — suffix[i] = product of all elements **after** i
   - suffix[last] = 1 (nothing after last element)
   - suffix[i] = suffix[i+1] * nums[i+1]
3. Answer: `result[i] = prefix[i] * suffix[i]`

**Example:**
```
nums:    [1,  2,  3,  4]
prefix:  [1,  1,  2,  6]    ← product of everything to my LEFT
suffix:  [24, 12, 4,  1]    ← product of everything to my RIGHT
answer:  [24, 12, 8,  6]    ← left * right
```

**Why it works:** For each index, the answer is "product of everything except me" = "product of everything to my left" × "product of everything to my right". No division needed.

**Reuse when:** Any problem asking "compute something for each element based on all OTHER elements" — prefix/suffix decomposition. Also: Trapping Rain Water (prefix max + suffix max), running sums, subarray products.

---

## Group Anagrams

**LeetCode #49 (Medium) | Pattern: Sort + Hash Map Grouping | Time: O(n * k log k) Space: O(n * k)**

*(where n = number of strings, k = max string length)*

**Algorithm:**
1. Create a map: `sorted_string → list of original strings`
2. For each string:
   - Sort its characters → this becomes the key (all anagrams produce the same sorted key)
   - Append the original string to that key's list
3. Collect all map values as the result

**Example:**
```
Input:  ["eat", "tea", "tan", "ate", "nat", "bat"]

"eat" → sort → "aet" → map["aet"] = ["eat"]
"tea" → sort → "aet" → map["aet"] = ["eat", "tea"]
"tan" → sort → "ant" → map["ant"] = ["tan"]
"ate" → sort → "aet" → map["aet"] = ["eat", "tea", "ate"]
"nat" → sort → "ant" → map["ant"] = ["tan", "nat"]
"bat" → sort → "abt" → map["abt"] = ["bat"]

Output: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

**Why it works:** Anagrams have the same characters, just in different order. Sorting normalizes them to the same key. Map groups them automatically.

**Reuse when:** Any problem that needs grouping by some canonical/normalized form — grouping by frequency, grouping by pattern, etc.

**Go gotchas:**
- Strings are **immutable** in Go — convert to `[]byte` before sorting, then back to `string` for the key
- `append` on a nil map value works fine — `res[key] = append(res[key], str)` handles both new and existing keys, no if/else needed
- When collecting results, use `out = append(out, val)` (direct assignment), not `append(out[i], val...)` which is a type mismatch

---

## Sliding Window Maximum

**LeetCode #239 (Hard) | Pattern: Monotonic Deque | Time: O(n) Space: O(k)**

**Algorithm (Bouncer analogy):**

Imagine a photo booth window that fits k people. You hire a bouncer with one rule:

1. Maintain a deque storing **indices** in **decreasing order** of their values (tallest person at front)
2. Before a new element enters, the bouncer **kicks out everyone shorter or equal** from the back — they can never be the max because the new element is taller AND stays in the window longer
3. Push new index to back of deque
4. If front index is **outside the window** (`<= i - k`), pop it from front
5. Once window is full (`i >= k-1`), the front of deque = current window max

**Example:**
```
nums = [1, 3, -1, -3, 5, 3, 6, 7],  k = 3

i=0: height=1  → deque: [1]
i=1: height=3  → kick 1 → deque: [3]
i=2: height=-1 → deque: [3, -1]          → max = 3
i=3: height=-3 → deque: [3, -1, -3]      → max = 3
i=4: height=5  → kick -3, -1, 3 → [5]   → max = 5
i=5: height=3  → deque: [5, 3]           → max = 5
i=6: height=6  → kick 3, 5 → [6]        → max = 6
i=7: height=7  → kick 6 → [7]           → max = 7

Result: [3, 3, 5, 5, 6, 7]
```

**Why O(n)?** Each element is pushed and popped **at most once** across the entire run = 2n total operations.

**Why it works:** A shorter element that entered before a taller one is useless — the taller one is both bigger AND will stay in the window longer. By proactively removing losers, the front of the deque is always the current max.

**Reuse when:** Any sliding window problem needing min/max — just flip the comparison for min. Also useful in problems like "next greater element" and monotonic stack/deque patterns.

**Interview build-up path:**
1. **Brute force O(n*k)** — scan all k elements per window → "k-1 elements overlap, re-scanning is wasteful"
2. **Max-Heap O(n log n)** — max floats up, lazy-delete expired → "heap keeps useless elements (shorter + entered earlier)"
3. **Monotonic Deque O(n)** — proactively remove losers → each element pushed/popped once

---

## Longest Substring Without Repeating Characters

**LeetCode #3 (Medium) | Pattern: Sliding Window + Hash Map | Time: O(n) Space: O(n)**

**Algorithm:**
1. Two pointers: `left` and `right` define the current window
2. A map tracks characters currently in the window
3. Move `right` one step at a time:
   - If `s[right]` **already in map** → shrink window from left: delete `s[left]` from map, increment `left`. Repeat until the duplicate is removed.
   - If `s[right]` **not in map** → expand window: add `s[right]` to map, move `right` forward
4. At each step, update `maxLen = max(maxLen, right - left + 1)`

**Example:**
```
s = "abcabcbb"

left=0, right=0: 'a' not in map → add → window "a"       max=1
left=0, right=1: 'b' not in map → add → window "ab"      max=2
left=0, right=2: 'c' not in map → add → window "abc"     max=3
left=0, right=3: 'a' IN map → remove s[0]='a', left=1
                  'a' not in map → add → window "bca"     max=3
left=1, right=4: 'b' IN map → remove s[1]='b', left=2
                  'b' not in map → add → window "cab"     max=3
...and so on
```

**Why it works:** The window always contains unique characters. When a duplicate appears, we shrink from the left until the duplicate is gone. We never move left backward, so each character is visited at most twice → O(n).

**Reuse when:** "Find longest/shortest substring with some condition" → sliding window. The map tracks what's inside the window. Shrink from left when condition breaks, expand right when it holds.

Also works for: Minimum Window Substring, Find All Anagrams, Longest Substring with At Most K Distinct Characters.

---

<!-- Add new problems below this line -->
