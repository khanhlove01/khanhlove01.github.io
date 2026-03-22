---
title: Arrays
description: A structured walkthrough of 15 classic array problems — summaries, edge cases, and Python solutions from naive to optimal, with follow-ups on space and variants.
author: Tran Minh Khanh
date: 2026-03-22 14:00:00 +0700
categories: [DSA, Arrays]
tags: [leetcode, python, arrays, two-pointers, hashing]
math: true
---

Arrays are the default training ground for interview-style problem solving. In one dimension you repeatedly meet the same **ideas**: sweep with **two pointers**, compress state with **hash maps / sets**, reuse **prefix or suffix aggregates**, **sort or cyclic-sort** to place values where indices mean something, **reverse or swap in place** to avoid extra memory, and **greedy** choices when a local rule implies a global optimum.

**What you can take away from this topic**

- **Time vs space:** Many optimal solutions are $O(n)$ time with $O(n)$ extra space (hashing). The next step is often “can we do **$O(1)$ extra space**?” — usually by mutating the input (with care), using indices as encoding, or a two-pass accumulation into the output array.
- **In-place discipline:** When the problem says “modify in place”, think about **where overwriting is safe** (e.g. merge from the **end** of a buffer).
- **Order of categories in `categories: [DSA, Arrays]`** matches how this note is filed next to other DSA notes.

The problems below follow a fixed pattern: **(1)** plain-language summary and examples, **(2)** edge cases, **(3)** solutions from least efficient to most efficient (with Python), then a **follow-up** question.

---

## [1. Two Sum](https://leetcode.com/problems/two-sum/)

### Step 1 — Problem summary and examples

Given an integer array `nums` and an integer `target`, return **indices** $i \neq j$ such that `nums[i] + nums[j] == target`. Exactly one valid pair exists.

| Example | `nums` | `target` | Output |
|--------|--------|----------|--------|
| 1 | `[2, 7, 11, 15]` | `9` | `[0, 1]` |
| 2 | `[3, 2, 4]` | `6` | `[1, 2]` |
| 3 | `[3, 3]` | `6` | `[0, 1]` |

### Step 2 — Edge cases

- Length $2$ (minimal).
- Duplicate values that still form the answer (example 3).
- Negative numbers and zero are fine as long as the pair sums to `target`.

### Step 3 — Solutions (least → most optimal)

**A. Brute force — $O(n^2)$ time, $O(1)$ space**

Try every pair.

```python
from typing import List


class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        n = len(nums)
        for i in range(n - 1):
            for j in range(i + 1, n):
                if nums[i] + nums[j] == target:
                    return [i, j]
        return []
```

**B. One-pass hash map — $O(n)$ time, $O(n)$ space**

While scanning, store each value’s index; if `target - nums[i]` is already in the map, return both indices.

```python
from typing import Dict, List


class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        seen: Dict[int, int] = {}
        for i, x in enumerate(nums):
            need = target - x
            if need in seen:
                return [seen[need], i]
            seen[x] = i
        return []
```

**Follow-up:** If the array were **sorted**, you could use **two pointers** in $O(n)$ time and $O(1)$ space (this version asks for indices on the **original** unsorted array, so sorting would require tracking indices). If duplicates were allowed for multiple answers, you would adapt the map or the two-pointer rules accordingly.

---

## [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/)

### Step 1 — Problem summary and examples

Given **non-decreasing** `nums`, remove duplicates **in place** so each unique value appears once at the front. Return $k$ = number of unique elements; the judge checks `nums[:k]`.

| Example | `nums` (before) | $k$ | `nums[:k]` |
|--------|-------------------|-----|------------|
| 1 | `[1, 1, 2]` | `2` | `[1, 2]` |
| 2 | `[0, 0, 1, 1, 1, 2, 2, 3, 3]` | `5` | `[0, 1, 2, 3]` |
| 3 | `[5]` | `1` | `[5]` |

### Step 2 — Edge cases

- Length $1$.
- **All identical** → $k = 1$.
- **No duplicates** → $k = n$.

### Step 3 — Solutions

**A. Extra array — $O(n)$ time, $O(n)$ space**

Copy unique values to a new list, then copy back (not required by constraints but easy to reason about).

**B. Two pointers (slow / fast) — $O(n)$ time, $O(1)$ space**

`write` marks the last position of the deduplicated prefix; `read` scans forward.

```python
from typing import List


class Solution:
    def removeDuplicates(self, nums: List[int]) -> int:
        if len(nums) <= 1:
            return len(nums)
        write = 0
        for read in range(1, len(nums)):
            if nums[read] != nums[write]:
                write += 1
                nums[write] = nums[read]
        return write + 1
```

**Follow-up:** **Remove duplicates at most twice** (LeetCode 80) — generalize the invariant to “allow $2$ copies” or use a separate counter.

---

## [41. First Missing Positive](https://leetcode.com/problems/first-missing-positive/)

### Step 1 — Problem summary and examples

Find the **smallest missing positive integer** in an unsorted array. Classic hard constraint: **$O(n)$ time**, **$O(1)$ extra space**.

| Example | `nums` | Answer |
|--------|--------|--------|
| 1 | `[1, 2, 0]` | `3` |
| 2 | `[3, 4, -1, 1]` | `2` |
| 3 | `[7, 8, 9, 11]` | `1` |

### Step 2 — Edge cases

- Answer is **`1`** (no `1` present).
- Answer is **`n + 1`** when `{1..n}` all appear.
- Zeros and negatives are **noise**; only values in `[1, n]` can sit at “correct” indices.

### Step 3 — Solutions

**A. Brute force — $O(n^2)$ or $O(n \cdot \text{range})$ time**

For each candidate $1, 2, \ldots$, scan the array.

**B. Hash set of values — $O(n)$ time, $O(n)$ space**

Store all numbers, then scan $1 \ldots n+1$.

```python
from typing import List, Set


class Solution:
    def firstMissingPositive(self, nums: List[int]) -> int:
        s: Set[int] = set(nums)
        n = len(nums)
        for i in range(1, n + 2):
            if i not in s:
                return i
        return n + 1
```

**C. Index as bucket (cyclic swaps) — $O(n)$ time, $O(1)$ space**

Place value $x$ at index $x-1$ when $1 \le x \le n$. First index where `nums[i] != i+1` gives the answer.

```python
from typing import List


class Solution:
    def firstMissingPositive(self, nums: List[int]) -> int:
        n = len(nums)
        for i in range(n):
            while 1 <= nums[i] <= n and nums[nums[i] - 1] != nums[i]:
                j = nums[i] - 1
                nums[i], nums[j] = nums[j], nums[i]
        for i in range(n):
            if nums[i] != i + 1:
                return i + 1
        return n + 1
```

**Follow-up:** Why must we **swap** instead of only overwriting? You must not drop a value before you know it is “useless” for the index-based placement.

---

## [66. Plus One](https://leetcode.com/problems/plus-one/)

### Step 1 — Problem summary and examples

Digits are given **MSB-first** as a non-negative integer; return digits of **value + 1**.

| Example | `digits` | Output |
|--------|----------|--------|
| 1 | `[1, 2, 3]` | `[1, 2, 4]` |
| 2 | `[9]` | `[1, 0]` |
| 3 | `[9, 9, 9]` | `[1, 0, 0, 0]` |

### Step 2 — Edge cases

- Last digit `< 9` → only last position changes.
- **All `9`s** → new length $n+1$ with leading `1` and trailing zeros.

### Step 3 — Solutions

**A. Convert to int — $O(n)$ time but can overflow in theory** (not acceptable for arbitrary big integers in real systems).

**B. Carry from the right — $O(n)$ time, $O(1)$ extra space** (amortized; output may be length $n+1$).

```python
from typing import List


class Solution:
    def plusOne(self, digits: List[int]) -> List[int]:
        n = len(digits)
        carry = 1
        for i in range(n - 1, -1, -1):
            s = digits[i] + carry
            digits[i] = s % 10
            carry = s // 10
            if carry == 0:
                return digits
        return [1] + [0] * n
```

**Follow-up:** **Add two big digit arrays** (strings or lists) — same carry pattern from the end; **multiply** uses convolution / FFT in advanced settings.

---

## [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/)

### Step 1 — Problem summary and examples

Merge sorted `nums2` (length `n`) into `nums1` (length `m+n`), where the first `m` elements of `nums1` are the first array’s data and the rest is padding `0`.

| `nums1` (initial) | `m` | `nums2` | `n` | Resulting `nums1` |
|-------------------|-----|---------|-----|-------------------|
| `[1,2,3,0,0,0]` | 3 | `[2,5,6]` | 3 | `[1,2,2,3,5,6]` |
| `[1]` | 1 | `[]` | 0 | `[1]` |
| `[0]` | 0 | `[1]` | 1 | `[1]` |

### Step 2 — Edge cases

- `m = 0` or `n = 0`.
- **All of `nums2` smaller** or **all larger** than `nums1`’s real part.

### Step 3 — Solutions

**A. New array then copy — $O(m+n)$ time, $O(m+n)$ space**

Merge like merge-sort, then write back.

**B. Fill from the tail — $O(m+n)$ time, $O(1)$ extra space**

Compare `nums1[i]` vs `nums2[j]` from the end to avoid overwriting unread values in `nums1`.

```python
from typing import List


class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        i, j, k = m - 1, n - 1, m + n - 1
        while i >= 0 and j >= 0:
            if nums1[i] > nums2[j]:
                nums1[k] = nums1[i]
                i -= 1
            else:
                nums1[k] = nums2[j]
                j -= 1
            k -= 1
        while j >= 0:
            nums1[k] = nums2[j]
            j -= 1
            k -= 1
```

**Follow-up:** Merge **$k$** sorted arrays — use a **min-heap** or **divide-and-conquer** merge.

---

## [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

### Step 1 — Problem summary and examples

One buy and one sell; maximize profit. If no profit, return `0`.

| `prices` | Best profit |
|----------|-------------|
| `[7,1,5,3,6,4]` | `5` (buy 1, sell 6) |
| `[7,6,4,3,1]` | `0` |
| `[2,4,1]` | `2` |

### Step 2 — Edge cases

- Length $1$.
- Strictly decreasing → `0`.

### Step 3 — Solutions

**A. Try all pairs — $O(n^2)$ time, $O(1)$ space**

**B. One pass minimum price — $O(n)$ time, $O(1)$ space**

```python
from typing import List


class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        min_price = float("inf")
        best = 0
        for p in prices:
            min_price = min(min_price, p)
            best = max(best, p - min_price)
        return best
```

**Follow-up:** **At most $k$ transactions** (LeetCode 188) — DP over days and transaction count.

---

## [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)

### Step 1 — Problem summary and examples

Unlimited transactions; maximize sum of gains (no overlapping holds).

| `prices` | Profit |
|----------|--------|
| `[7,1,5,3,6,4]` | `7` |
| `[1,2,3,4,5]` | `4` |
| `[7,6,4,3,1]` | `0` |

### Step 2 — Edge cases

- Flat prices → `0`.
- Length $1$ → `0`.

### Step 3 — Solutions

**A. Greedy on positive diffs — $O(n)$ time, $O(1)$ space**

```python
from typing import List


class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        return sum(max(0, prices[i] - prices[i - 1]) for i in range(1, len(prices)))
```

**B. Valley / peak — same asymptotics**, explicit buy-low sell-high segments.

```python
from typing import List


class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        n = len(prices)
        i = 0
        profit = 0
        while i < n - 1:
            while i < n - 1 and prices[i] >= prices[i + 1]:
                i += 1
            valley = prices[i]
            while i < n - 1 and prices[i] <= prices[i + 1]:
                i += 1
            profit += prices[i] - valley
        return profit
```

**Follow-up:** With a **fee per transaction** or **cooldown**, greedy on adjacent diffs stops being enough — use DP.

---

## [169. Majority Element](https://leetcode.com/problems/majority-element/)

### Step 1 — Problem summary and examples

Value appearing **strictly more than $\lfloor n/2 \rfloor$** times; it always exists.

| `nums` | Majority |
|--------|----------|
| `[3,2,3]` | `3` |
| `[2,2,1,1,1,2,2]` | `2` |
| `[1]` | `1` |

### Step 2 — Edge cases

- Length $1$.
- Only two distinct values with heavy skew.

### Step 3 — Solutions

**A. Brute force counts — $O(n^2)$ time**

**B. Hash map counts — $O(n)$ time, $O(n)$ space**

**C. Sort middle element — $O(n \log n)$ time, $O(1)$ or $O(\log n)$ space** depending on sort

**D. Boyer–Moore vote — $O(n)$ time, $O(1)$ space**

```python
from typing import List


class Solution:
    def majorityElement(self, nums: List[int]) -> int:
        candidate = 0
        count = 0
        for x in nums:
            if count == 0:
                candidate = x
            count += 1 if x == candidate else -1
        return candidate
```

**Follow-up:** Elements appearing **> n/3** (LeetCode 229) — Boyer–Moore **extended** to two candidates + verification pass.

---

## [189. Rotate Array](https://leetcode.com/problems/rotate-array/)

### Step 1 — Problem summary and examples

Rotate **right** by `k` steps in place.

| `nums` | `k` | Result |
|--------|-----|--------|
| `[1,2,3,4,5,6,7]` | `3` | `[5,6,7,1,2,3,4]` |
| `[-1,-100,3,99]` | `2` | `[3,99,-1,-100]` |
| `[1]` | `10` | `[1]` |

### Step 2 — Edge cases

- `k` larger than `n` → use `k % n`.
- `k == 0` → unchanged.

### Step 3 — Solutions

**A. Naive shift `k` times — $O(n \cdot k)$ worst**

**B. Extra buffer — $O(n)$ time and space**

**C. Triple reverse — $O(n)$ time, $O(1)$ space**

```python
from typing import List


class Solution:
    def rotate(self, nums: List[int], k: int) -> None:
        n = len(nums)
        if n <= 1:
            return
        k %= n

        def rev(lo: int, hi: int) -> None:
            while lo < hi:
                nums[lo], nums[hi] = nums[hi], nums[lo]
                lo += 1
                hi -= 1

        rev(0, n - 1)
        rev(0, k - 1)
        rev(k, n - 1)
```

**Follow-up:** **Left** rotation by `k` is the same idea with different slice lengths; **rotate matrix** uses layer-wise cycles.

---

## [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate/)

### Step 1 — Problem summary and examples

Return `True` if **any** value appears at least twice.

| `nums` | Result |
|--------|--------|
| `[1,2,3,1]` | `True` |
| `[1,2,3,4]` | `False` |
| `[1,1,1,1]` | `True` |

### Step 2 — Edge cases

- Length $1$ → `False`.
- Large range of values (hashing still $O(n)$).

### Step 3 — Solutions

**A. All pairs — $O(n^2)$ time**

**B. Sort and scan neighbors — $O(n \log n)$ time, $O(1)$ extra if sort in place**

**C. Hash set — $O(n)$ time, $O(n)$ space**

```python
from typing import List


class Solution:
    def containsDuplicate(self, nums: List[int]) -> bool:
        seen = set()
        for x in nums:
            if x in seen:
                return True
            seen.add(x)
        return False
```

**Follow-up:** **Contains duplicate within distance `k`** or **in sorted subarrays of size `t`** — sliding window + ordered structure.

---

## [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/)

### Step 1 — Problem summary and examples

Return array `answer[i]` = product of all elements **except** `nums[i]`. **No division**; prefer $O(1)$ extra space excluding output.

| `nums` | Output |
|--------|--------|
| `[1,2,3,4]` | `[24,12,8,6]` |
| `[-1,1,0,-3,3]` | `[0,0,9,0,0]` |
| `[2,3]` | `[3,2]` |

### Step 2 — Edge cases

- **Zeros:** more than one zero → all products `0`; exactly one zero → only that position non-zero.
- Negative numbers — sign handled by multiplication.

### Step 3 — Solutions

**A. Naive product per index — $O(n^2)$ time**

**B. Prefix and suffix arrays — $O(n)$ time, $O(n)$ space**

**C. Output as left product, then multiply by running right product — $O(n)$ time, $O(1)$ extra**

```python
from typing import List


class Solution:
    def productExceptSelf(self, nums: List[int]) -> List[int]:
        n = len(nums)
        out = [0] * n
        out[0] = 1
        for i in range(1, n):
            out[i] = out[i - 1] * nums[i - 1]
        right = 1
        for i in range(n - 1, -1, -1):
            out[i] *= right
            right *= nums[i]
        return out
```

**Follow-up:** **With division** (if allowed) still fails on zeros unless handled separately — why interviewers ban division.

---

## [283. Move Zeroes](https://leetcode.com/problems/move-zeroes/)

### Step 1 — Problem summary and examples

Move all `0`s to the **end**, preserve relative order of non-zeros — **in place**.

| `nums` (before → after) |
|-------------------------|
| `[0,1,0,3,12]` → `[1,3,12,0,0]` |
| `[0]` → `[0]` |
| `[1,2,3]` → `[1,2,3]` |

### Step 2 — Edge cases

- All zeros.
- No zeros.
- Zeros only at end (already valid).

### Step 3 — Solutions

**A. New array of non-zeros + pad zeros — $O(n)$ time, $O(n)$ space**

**B. Two-pointer write index — $O(n)$ time, $O(1)$ space** (assign non-zeros, fill tail with zeros)

**C. Swap non-zero forward — $O(n)$ time, $O(1)$ space** (fewer writes when dense)

```python
from typing import List


class Solution:
    def moveZeroes(self, nums: List[int]) -> None:
        w = 0
        for i in range(len(nums)):
            if nums[i] != 0:
                if w != i:
                    nums[w], nums[i] = nums[i], nums[w]
                w += 1
```

**Follow-up:** **Move all instances of `val`** to the end — identical pattern with a different predicate.

---

## [334. Increasing Triplet Subsequence](https://leetcode.com/problems/increasing-triplet-subsequence/)

### Step 1 — Problem summary and examples

Return `True` if there exist **indices** $i < j < k$ with `nums[i] < nums[j] < nums[k]` (not necessarily consecutive).

| `nums` | Result |
|--------|--------|
| `[1,2,3,4,5]` | `True` |
| `[5,4,3,2,1]` | `False` |
| `[20,100,10,12,5,13]` | `True` (e.g. 10, 12, 13) |

### Step 2 — Edge cases

- Length $< 3$ → `False`.
- **Equal values** do not count as strictly increasing.

### Step 3 — Solutions

**A. Triple nested loop — $O(n^3)$ time**

**B. Greedy smallest ends — $O(n)$ time, $O(1)$ space**

Maintain the smallest first value and the smallest second value of a valid length-2 increasing pair seen so far.

```python
from typing import List


class Solution:
    def increasingTriplet(self, nums: List[int]) -> bool:
        first = second = float("inf")
        for x in nums:
            if x <= first:
                first = x
            elif x <= second:
                second = x
            else:
                return True
        return False
```

**Follow-up:** **Length-$k$ increasing subsequence** → patience sorting / binary search on tails ($O(n \log n)$) — classic **LIS** structure.

---

## [350. Intersection of Two Arrays II](https://leetcode.com/problems/intersection-of-two-arrays-ii/)

### Step 1 — Problem summary and examples

Multiset intersection: each value appears in the output as many times as it appears in **both** arrays.

| `nums1` | `nums2` | Output (any order) |
|---------|---------|---------------------|
| `[1,2,2,1]` | `[2,2]` | `[2,2]` |
| `[4,9,5]` | `[9,4,9,8,4]` | `[4,9]` or `[9,4]` |
| `[1]` | `[2]` | `[]` |

### Step 2 — Edge cases

- One array empty.
- **Counts matter** (unlike set intersection).

### Step 3 — Solutions

**A. Hash map frequency on smaller array — $O(m+n)$ time, $O(\min(m,n))$ space**

```python
from typing import Dict, List


class Solution:
    def intersect(self, nums1: List[int], nums2: List[int]) -> List[int]:
        if len(nums1) > len(nums2):
            nums1, nums2 = nums2, nums1
        freq: Dict[int, int] = {}
        for x in nums1:
            freq[x] = freq.get(x, 0) + 1
        out: List[int] = []
        for x in nums2:
            if freq.get(x, 0) > 0:
                out.append(x)
                freq[x] -= 1
        return out
```

**B. Sort + two pointers — $O(n \log n + m \log m)$ time, $O(1)$ extra** if sorting in place

**Follow-up:** Arrays **sorted** and **very large on disk** — stream one file, binary search or merge on the other; if `nums2` cannot fit in memory, stream counts or use external merge.

---

## [2348. Number of Zero-Filled Subarrays](https://leetcode.com/problems/number-of-zero-filled-subarrays/)

### Step 1 — Problem summary and examples

Count the number of **contiguous** subarrays that contain **only** zeros.

| `nums` | Subarrays (conceptually) | Count |
|--------|--------------------------|-------|
| `[1,3,0,0,2,0,0,4]` | runs of length 2 and 2 | `6` |
| `[0,0,0]` | lengths 1,2,3 inside run 3 | `6` |
| `[1,2,3]` | none | `0` |

### Step 2 — Edge cases

- Entire array zeros.
- Isolated single `0` → contributes `1` subarray.

### Step 3 — Solutions

**A. Enumerate every subarray — $O(n^2)$ or worse**

**B. For each maximal run of length $L$, add $\frac{L(L+1)}{2}$ — $O(n)$ time, $O(1)$ space**

```python
from typing import List


class Solution:
    def zeroFilledSubarray(self, nums: List[int]) -> int:
        total = 0
        i = 0
        n = len(nums)
        while i < n:
            if nums[i] != 0:
                i += 1
                continue
            j = i
            while j < n and nums[j] == 0:
                j += 1
            L = j - i
            total += L * (L + 1) // 2
            i = j
        return total
```

**Follow-up:** Count subarrays with **at most** $k$ zeros → sliding window / prefix on zero counts.

---

## Closing

These fifteen problems are a compact **curriculum**: hashing, two pointers, in-place transforms, greedy tracking, and combinatorics on runs. When you finish one, ask the usual follow-ups: **sorted input?**, **streaming / disk?**, **$O(1)$ space?**, and **what if duplicates or counts matter?** That habit maps cleanly from arrays to strings, matrices, and linked lists.

If you keep solutions in a repo (for example under `DSA-2025/1.Array`), this post is meant to mirror that folder as readable notes you can publish alongside your code.
