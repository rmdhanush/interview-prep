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

**Go tip:** `append` on a nil slice works fine — `res[key] = append(res[key], str)` handles both new and existing keys without an if/else check.

**Why it works:** Anagrams have the same characters, just in different order. Sorting normalizes them to the same key. Map groups them automatically.

**Reuse when:** Any problem that needs grouping by some canonical/normalized form — grouping by frequency, grouping by pattern, etc.

---

<!-- Add new problems below this line -->
