---
title: "LeetCode 1282: Group the People, from Capacity Buckets to Emitting Full Groups"
date: 2026-10-10T10:00:00+08:00
draft: false
categories: ["LeetCode"]
tags: ["array", "hash table", "grouping", "LeetCode 1282"]
description: "Distinguish person IDs from group sizes, collect and split capacity buckets, then build a one-pass Python solution that emits full groups without mutating saved results."
summary: "People who request the same group size belong in the same bucket, but one bucket may contain several groups. Build two connected solutions and learn to validate answers with no fixed order."
keywords: ["LeetCode 1282", "Group the People", "hash table grouping", "capacity buckets", "Python list references"]
---

## Problem Requirements

You are given an integer array `groupSizes`. There are `n` people, numbered from `0` to `n - 1`. The value `groupSizes[i]` is **the exact number of people that person `i` must have in their group**, including themselves.

Return a list of groups containing person IDs, with these requirements:

- Each person appears in exactly one group, with no duplicates or omissions.
- If a group contains `k` people, every member's `groupSizes[i]` must equal `k`.
- Neither the order of people within a group nor the order of groups matters. Any valid grouping is accepted.
- People in the same group do not have to occupy consecutive positions in the input array.

The constraints are `1 <= n <= 500` and `1 <= groupSizes[i] <= n`. **The problem guarantees that at least one valid grouping exists.**

Example 1:

```text
Input: groupSizes = [3,3,3,3,3,1,3]
One valid output: [[0,1,2],[5],[3,4,6]]
```

People `0`, `1`, and `2` each require a three-person group, as do people `3`, `4`, and `6`. Person `5` needs a group of one. The same requested size can produce more than one group.

Example 2:

```text
Input: groupSizes = [2,1,3,3,3,2]
One valid output: [[1],[0,5],[2,3,4]]
```

This tutorial is for readers who know Python loops and list operations and are practicing hash-table grouping. The examples use Python 3.10+, and the estimated reading time is 10 minutes. We will start with what belongs in a group, before reaching for a template.

## Step 1: Groups Contain Person IDs, Not Array Values

Start with a smaller input: `groupSizes = [2,1,2]`.

Following the intuition of "put equal values together" might give us `[[2,2],[1]]`. That answer is wrong: person `2` appears twice, and person `0` never appears. The issue is not sorting. It is confusing **a person's identity** with **the capacity that person requires**.

Make just one distinction in this step: when reading the array, look at both the index and its value.

| Person ID | Required group size | A possible arrangement |
| --- | --- | --- |
| 0 | 2 | Join person 2 in a two-person group |
| 1 | 1 | Form a group alone |
| 2 | 2 | Join person 0 in a two-person group |

So `[[0,2],[1]]` is valid. Check each group: `[0,2]` has length `2`, which both members require; `[1]` has length `1`, which that member requires. Each of the IDs `0`, `1`, and `2` appears exactly once.

This suggests a smaller subproblem: **before arranging every group, collect the IDs of people who require the same group size.** People in the same group must agree on its size. Someone who requires two people and someone who requires three cannot share a group.

We can now read the input and check this small candidate by hand. What we still lack is a way to remember earlier people: person `0` and person `2` are separated by someone else, so where should the program keep the first person until it finds a compatible one?

## Step 2: Store Person IDs by Their Required Group Size

Our current baseline is still manual matching. In `[2,1,2]`, we encounter person `0`, then person `1`, and only then person `2`, who can share a group with `0`. Looking only at adjacent positions gives us no place to keep the earlier person.

Add a dictionary named `buckets`. Each key is a required group size, and its value is a list of IDs encountered so far that require that size. For example, `buckets[2] = [0,2]` means both people need a two-person group. We call this list a "bucket" because it temporarily holds items with the same requirement.

First, write a runnable version that only collects IDs, without arranging the final groups:

```python
def collect_by_size(group_sizes: list[int]) -> dict[int, list[int]]:
    buckets: dict[int, list[int]] = {}
    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)
    return buckets


assert collect_by_size([2, 1, 2]) == {2: [0, 2], 1: [1]}
assert collect_by_size([2, 2, 2, 2]) == {2: [0, 1, 2, 3]}
```

`enumerate` provides both the index, `person`, and the value, `size`. Use `size` to find the bucket, then put `person` inside it. When size `2` appears again, we use its existing list rather than overwrite the IDs already stored there.

The first assertion resolves the nonadjacent-matches problem. The second also reveals a limit: all four people require two-person groups, but collecting them produces a bucket of four. **One bucket means one shared requirement, not necessarily one valid group.**

This version can now collect every compatible person ID. It still cannot return all the buckets directly as the answer. The next question is how to split a bucket when its population exceeds the required group size.

## Step 3: Split Each Bucket into Groups of the Required Size

The current version turns `[2,2,2,2]` into `{2: [0,1,2,3]}`. All four people need two-person groups, so returning that entire list would put every person in an oversized group.

We do not need a new classification rule. We need to enforce capacity. From the size-`2` bucket, take two people at a time: `[0,1]`, then `[2,3]`. There are no additional restrictions on who can be paired, so there is no need to search for a "better" pair.

Add a splitting loop to the previous version: keep the collection code, then replace `return buckets` with capacity-sized slices and return the resulting groups.

```python
def group_people_two_pass(group_sizes: list[int]) -> list[list[int]]:
    buckets: dict[int, list[int]] = {}
    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)

    result: list[list[int]] = []
    for size, people in buckets.items():
        for start in range(0, len(people), size):
            result.append(people[start:start + size])
    return result


assert group_people_two_pass([2, 2, 2, 2]) == [[0, 1], [2, 3]]
```

`start` advances by `size` each time, and `people[start:start + size]` extracts one group. This assertion checks this implementation's particular split. It does not mean the problem accepts only this arrangement.

Could the last slice contain fewer than `size` people? Not for a valid problem input. Suppose `m` people require groups of size `k`. Those people can only join groups of size `k`; groups of another size cannot accept them. Since a valid answer is guaranteed to exist, `m` must be a multiple of `k`. Every slice therefore contains exactly enough people.

For the two problem examples, check capacities and ID coverage instead of comparing against one supposedly unique arrangement:

| Input | This implementation returns | Check |
| --- | --- | --- |
| `[3,3,3,3,3,1,3]` | `[[0,1,2],[3,4,6],[5]]` | Two three-person groups and one singleton; IDs `0..6` appear once each |
| `[2,1,3,3,3,2]` | `[[0,5],[1],[2,3,4]]` | Groups of two, one, and three; IDs `0..5` appear once each |

This is our first complete, correct solution. With expected constant-time hash-table operations, collecting the IDs takes `O(n)` time. Slicing copies lists, but all slices together copy only `n` IDs, so splitting also takes `O(n)` time. The temporary buckets require `O(n)` auxiliary space, and the returned groups take another `O(n)` space.

The solution has no time-complexity problem, but it stores everybody before starting the second pass. For `[2,2,2,2]`, the group `[0,1]` is already full when we reach person `1`. Could we emit that group immediately and leave only unfinished groups in the buckets?

## Step 4: Emit a Full Group, Then Start a New List

Our current baseline takes two passes: collect every ID into a bucket, then split the buckets. It is already correct; the issue is simply that completed groups wait until the scan ends before being emitted.

Consider people `0` and `1`, both requesting groups of two. After appending `1` to `buckets[2]`, the list's length equals its capacity, `2`. We can add it to `result` immediately, then make `buckets[2]` the temporary storage for the next group.

Replace the previous collection loop with the version below. We add only a branch for reaching capacity; the second splitting pass is no longer needed:

```python
def group_people(group_sizes: list[int]) -> list[list[int]]:
    buckets: dict[int, list[int]] = {}
    result: list[list[int]] = []

    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)
        if len(buckets[size]) == size:
            result.append(buckets[size])
            buckets[size] = []

    return result


assert group_people([2, 2, 2, 2]) == [[0, 1], [2, 3]]
assert group_people([3, 3, 3, 3, 3, 1, 3]) == [
    [0, 1, 2], [5], [3, 4, 6]
]
```

The crucial line is `buckets[size] = []`: it **rebinds** that dictionary entry to a new list. Do not clear the existing list instead:

```python
bad_bucket = [0, 1]
bad_result = []
bad_result.append(bad_bucket)
bad_bucket.clear()
assert bad_result == [[]]
```

`bad_result` holds a reference to the same list object. Clearing the temporary list also clears the group we already handed to the result. By contrast, `buckets[size] = []` leaves the old list in `result` and uses a new one for the next group. This change in list ownership is essential when one requested size produces several groups: it keeps completed results from being modified by later work.

Walk through `[2,1,2,2,2,1]` in input order. The bucket column shows its contents after appending the current person, before any full group is emitted and its bucket replaced:

| Person read (required size) | Bucket | Full? | Result |
| --- | --- | --- | --- |
| 0 (2) | `[0]` | No | `[]` |
| 1 (1) | `[1]` | Yes | `[[1]]` |
| 2 (2) | `[0,2]` | Yes | `[[1],[0,2]]` |
| 3 (2) | `[3]` | No | `[[1],[0,2]]` |
| 4 (2) | `[3,4]` | Yes | `[[1],[0,2],[3,4]]` |
| 5 (1) | `[5]` | Yes | `[[1],[0,2],[3,4],[5]]` |

For valid input, the total number of people requesting each size is divisible by that size, so the scan cannot leave any unfinished groups. This version removes an explicit splitting pass, but its total work is still `O(n)`. We have not improved the asymptotic bound; we have moved the completion check to the earliest point where a group is ready.

We now have a correct function that needs no second pass. One final step remains: put the same logic into LeetCode's `Solution.groupThePeople` interface, justify it for the whole input, and check the complete code without depending on output order.

## Step 5: Package the Submission and Prove It Correct

The current `group_people` function already implements the algorithm. LeetCode requires a class method, so this step only packages the interface. The loop, completion condition, and list-rebinding rule stay unchanged:

```python
from typing import List


class Solution:
    def groupThePeople(self, groupSizes: List[int]) -> List[List[int]]:
        buckets: dict[int, list[int]] = {}
        result: list[list[int]] = []

        for person, size in enumerate(groupSizes):
            if size not in buckets:
                buckets[size] = []
            buckets[size].append(person)
            if len(buckets[size]) == size:
                result.append(buckets[size])
                buckets[size] = []

        return result
```

### Correctness: What the Scan Maintains

For each group size `k`, `buckets[k]` contains only people we have encountered who require size `k` and have not yet been emitted into the result. Each new person with that requirement is appended to this bucket. Once its length reaches `k`, it is a valid group of exactly `k` people. We append it to the result and replace its bucket with a fresh empty list.

This invariant gives us three conclusions:

1. **Every emitted group has the right capacity.** We emit only when `len(buckets[k]) == k`, so the group length equals every member's required size.
2. **A person cannot be duplicated or lost between groups.** Each ID is appended once by `enumerate`, to exactly one bucket. After its list is emitted, we never write to that list again; later IDs go into a new list. Every encountered person therefore remains either in one pending bucket or in one completed group.
3. **No incomplete group remains at the end.** A valid answer is guaranteed. People requesting size `k` cannot join groups of another size, so their total count must be a multiple of `k`. Emitting each full batch leaves no partial bucket at the end. Combined with the previous point, this means every person appears in exactly one output group.

### Complexity

Each person is read once, appended once, and included in one emitted group. With expected `O(1)` hash-table operations, the total time is `O(n)`. The buckets and returned groups store person IDs; worst-case auxiliary space and output space are both `O(n)`. A single scan improves how the work is organized, not the asymptotic bound below `O(n)`.

### Validating an Answer with No Fixed Order

Do not compare the result against just one fixed arrangement. Instead, check the requirements directly:

```python
def validate(group_sizes: list[int], groups: list[list[int]]) -> None:
    assert all(groups)
    flat = [person for group in groups for person in group]
    assert sorted(flat) == list(range(len(group_sizes)))
    assert all(
        group_sizes[person] == len(group)
        for group in groups
        for person in group
    )


for sizes in ([3, 3, 3, 3, 3, 1, 3], [2, 1, 3, 3, 3, 2], [1], [2, 2, 2, 2]):
    validate(sizes, Solution().groupThePeople(sizes))
```

The first assertion rejects empty groups. The flattened IDs must then cover every person exactly once, and each member's required size must match their group's length. These checks describe valid output rather than one particular ordering.

Also cover the `n = 500` boundary and valid random inputs made by constructing complete groups and shuffling their required sizes. What matters is that each person appears once and every nonempty group has the right capacity, not the order in which groups appear.

## Summary

The key distinction is not about sorting: an array **index identifies a person**, while the **value is that person's required group capacity**. From that distinction, the code develops through three connected versions:

1. Collect compatible person IDs in buckets keyed by capacity.
2. After collection, split each bucket into groups of that capacity.
3. Emit each full group during the scan and use a new list for the next group.

The last detail also applies beyond this problem. Once a list has been handed to a result collection, do not clear it in place to reuse it. Rebinding to a new list keeps later writes from changing the data already returned to the result.

## Practice and Further Reading

- Replace `buckets` with `collections.defaultdict(list)` and compare how it changes the initialization logic.
- Write an ID-coverage check without sorting. Consider how it could reduce validation costs if the constraints grew.
- Compare [LeetCode 49: Group Anagrams](https://leetcode.com/problems/group-anagrams/). Both problems group items by a key. Here, the key comes directly from the required capacity, and we need an additional completion rule because one bucket can produce several groups.

Reference: [LeetCode 1282: Group the People Given the Group Size They Belong To](https://leetcode.com/problems/group-the-people-given-the-group-size-they-belong-to/).
