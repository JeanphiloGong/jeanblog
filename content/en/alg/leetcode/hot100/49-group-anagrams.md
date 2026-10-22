---
title: "LeetCode 49: Group Anagrams, from Scanning Groups to Building a Grouping Key"
subtitle: "Start by comparing two words, then build an index with sorted strings and character counts"
date: 2026-10-10T10:00:00+08:00
draft: false
summary: "Build a Python solution one step at a time: compare two anagrams, scan group representatives, then derive sorted-string and character-count keys to understand why hash-table grouping works."
categories: ["LeetCode"]
tags: ["Hot100", "hash table", "string", "LeetCode 49"]
keywords: ["Group Anagrams", "grouping key", "character counts", "hash table", "Python"]
readingTime: 10
---

## Problem Requirements

Given an array of strings `strs`, place anagrams in the same group and return all the groups. Two words are anagrams when they contain the same letters with the same number of occurrences of each letter; their character order may differ.

An example from the problem:

```text
Input: strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
One valid output: [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

Neither the order of groups nor the order of words within a group matters. However, every input occurrence must be preserved: `["a", "a"]` must produce `[["a", "a"]]`, not a deduplicated result.

The constraints are `1 <= len(strs) <= 10^4`, each string has length `0..100`, and all characters are lowercase English letters. Empty strings are valid: `[""]` produces `[[""]]`.

This article uses Python 3.10+ and assumes you can write loops. The goal is to understand what makes a valid basis for grouping.

## Step 1: Solve the Smaller Problem of Comparing Two Words

Set the full array aside for a moment and answer a smaller question: **should these two words belong to the same group?**

Direct string comparison is too strict: `"eat" != "tea"`, yet they should be grouped together. Could we compare only which letters appear? This custom counterexample shows why that also fails:

```text
aab: a appears twice, b appears once
abb: a appears once, b appears twice
```

Both character sets are `{"a", "b"}`, but one word cannot be rearranged into the other. Sets discard multiplicity; we only want to ignore order.

Make one small change: sort the characters in each word, then compare the results.

```text
eat -> aet
tea -> aet
aab -> aab
abb -> abb
```

Sorting preserves every character occurrence while putting the characters in a common order. Two words are therefore anagrams if and only if their sorted character lists are equal. Here is the smallest runnable version of that test:

```python
def same_anagram(a: str, b: str) -> bool:
    return sorted(a) == sorted(b)


assert same_anagram("eat", "tea")
assert not same_anagram("aab", "abb")
assert same_anagram("", "")
assert not same_anagram("a", "aa")
assert set("aab") == set("abb")  # Equal sets are not enough to establish anagrams.
```

This version can decide whether two words belong together without losing repeated letters. It cannot yet group an entire array. Next, we need to decide whether each new word joins an existing group or starts a new one.

## Step 2: Use the First Word in Each Group as Its Representative

Our existing `same_anagram` only compares two words. When we reach the third word, `"tan"`, we also need to remember the groups formed so far.

Store those groups in a list named `groups`. Every group is nonempty, so `group[0]` can represent it: if the new word is an anagram of that representative, append it to the group. If none of the representatives match, create a new group containing just the new word.

Why not compare against every word in a group? All existing members have the same sorted characters as the representative. If the new word also matches that sorted sequence, it matches every member of the group.

Place the following function after the previous `same_anagram` definition to get the first complete solution:

```python
def group_scan(strs: list[str]) -> list[list[str]]:
    groups: list[list[str]] = []

    for word in strs:
        for group in groups:
            if same_anagram(word, group[0]):
                group.append(word)
                break
        else:
            groups.append([word])

    return groups


assert group_scan(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert group_scan(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

The `else` belongs to the inner `for`, not the `if`. In Python, `for ... else` runs the `else` block **when the loop finishes without a `break`**. A match appends the word and breaks out of the scan; no match creates a new group. When `groups` is initially empty, the inner loop runs zero times and its `else` still executes, creating the first group.

Trace the first four words slowly:

| New Word | Existing Representatives Checked | Action | `groups` Afterward |
| --- | --- | --- | --- |
| `eat` | None yet | Create a group | `[["eat"]]` |
| `tea` | Matches `eat` | Append to the first group | `[["eat", "tea"]]` |
| `tan` | Does not match `eat` | Create a group | `[["eat", "tea"], ["tan"]]` |
| `ate` | Matches `eat` | Append to the first group | `[["eat", "tea", "ate"], ["tan"]]` |

Each iteration either appends to one existing group or creates one new group. Duplicate words are therefore preserved. The assertions check the order this implementation happens to produce; the problem itself does not require that order.

This version already solves the entire problem correctly. Its remaining weakness is efficiency: if every word belongs to a different group, the `i`-th word checks all `i - 1` earlier representatives, sorting both words again for each comparison.

## Step 3: Turn the Sorted Result into a Dictionary Key

The previous version's bottleneck is not the anagram test itself. To find a word's group, it must try the existing groups one by one. If all `n` words belong to different groups, that means `n(n - 1) / 2` representative comparisons. Since `same_anagram` already converts each word into a stable sorted sequence, could that sequence locate its group directly?

Look at the sorted forms:

```text
eat -> aet
tea -> aet
tan -> ant
```

Equal sorted sequences identify exactly the same anagram class. `sorted(word)` returns a list, which cannot be a dictionary key. Joining it into a string gives us an immutable, hashable key such as `"aet"`.

Replace the `groups` list with a dictionary named `groups_by_key`. Each key is a sorted string; its value is still a list of original words. We do not replace `"eat"` with `"aet"` in the output: the key only identifies the group.

This change replaces the search for a group. We still append the original word to its group and collect all groups at the end:

```python
def group_by_sorted(strs: list[str]) -> list[list[str]]:
    groups_by_key: dict[str, list[str]] = {}

    for word in strs:
        key = "".join(sorted(word))
        if key not in groups_by_key:
            groups_by_key[key] = []
        groups_by_key[key].append(word)

    return list(groups_by_key.values())


assert group_by_sorted(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert group_by_sorted(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

We now locate groups through dictionary operations rather than scanning existing groups. Under the usual average-case hash-table model, lookup avoids a search across all groups; constructing and hashing the string key still requires work on its characters. Two keys are equal if and only if their sorted character sequences match, so the same anagram test still governs membership.

This is already a clear, correct, commonly used solution. It still sorts every word, though. The problem limits the alphabet to 26 lowercase English letters, which gives us one more question to investigate: do we need to arrange the characters in order at all?

## Step 4: Build the Final Key from 26 Character Counts

Sorting makes each word's letters and their multiplicities comparable. Because the input contains only lowercase English letters, we can record the number of occurrences of each letter directly: `counts[0]` represents `a`, `counts[1]` represents `b`, and so on.

For example, these vectors abbreviate the intervening zero counts:

```text
eat -> (1, 0, 0, 0, 1, ..., 1, ..., 0)
tea -> (1, 0, 0, 0, 1, ..., 1, ..., 0)
aab -> (2, 1, 0, ..., 0)
abb -> (1, 2, 0, ..., 0)
```

The first two 26-element vectors match; the last two do not. A list cannot be a dictionary key, but converting this list of integers into an immutable tuple makes it usable as one. Each character is visited once, so constructing the key for a word of length `k` costs `O(k + 26)`, including initialization and tuple creation.

Only the key construction needs to change; the dictionary grouping loop can stay as it is. First, test the new key on its own:

```python
def count_key(word: str) -> tuple[int, ...]:
    counts = [0] * 26
    for char in word:
        counts[ord(char) - ord("a")] += 1
    return tuple(counts)


assert count_key("eat") == count_key("tea")
assert count_key("aab") != count_key("abb")
assert count_key("") == (0,) * 26
```

`ord(char) - ord("a")` maps `a` to index `0`, `b` to `1`, and `z` to `25`. For `"eat"`, it increments indices `4`, `0`, and `19`, leaving every other count at zero. This indexing works because of the problem's lowercase-English constraint.

Now put that key construction into the earlier dictionary loop and use the platform's `class Solution` interface. The class wrapper adds no algorithm. The counting code is written inline below so this final block runs independently, without requiring the demonstration helper above:

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        groups_by_key: dict[tuple[int, ...], list[str]] = {}

        for word in strs:
            counts = [0] * 26
            for char in word:
                counts[ord(char) - ord("a")] += 1
            key = tuple(counts)

            if key not in groups_by_key:
                groups_by_key[key] = []
            groups_by_key[key].append(word)

        return list(groups_by_key.values())


solution = Solution()
assert solution.groupAnagrams(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert solution.groupAnagrams(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

The loop invariant is: after processing each word, every entry in `groups_by_key` contains exactly the original words processed so far whose count vector equals that entry's key. Initially there are no processed words, so an empty dictionary satisfies it. Each iteration computes the current word's vector and appends it exactly once to the matching group, creating that group if needed; all other groups stay unchanged.

Two count vectors are equal if and only if every letter has the same multiplicity in both words, which is precisely the anagram condition. At the end, every word is therefore in the correct group and every input occurrence has been preserved. The solution is complete for the stated problem; the 26-slot key does not apply unchanged to arbitrary alphabets.

### Complexity

Let `S` be the total number of characters, `n` the number of words, and `g` the number of groups. We count each character once. Initializing, converting, and hashing each fixed-size key also takes work proportional to 26, so the expected running time is `O(S + 26n)`, or `O(S + n)` with the alphabet size treated as a constant. This uses average-case hash-table behavior.

We could simplify that to `O(S)` if every word had at least one character. This problem allows empty words, though: processing `n` empty strings still takes `O(n)` time even when `S = 0`. Keeping the `+ n` makes that case explicit.

The dictionary entries and their keys occupy `O(26g)` space. The group lists store `O(n)` references to input strings, and the returned outer list holds `O(g)` group references; the strings themselves are not copied. Including those output lists, space usage is `O(26g + n)`. Excluding output storage, the dictionary and keys account for `O(26g)` auxiliary space, plus the current 26-element counter.

If we do not want to rely on the lowercase-English constraint, Step 3's sorted-string key is more general. Writing the `i`-th word's length as `k_i`, its expected time is `O(n + sum(k_i * log(k_i + 1)))`. The `n` covers per-word overhead, including empty words, and `log(k_i + 1)` also handles lengths zero and one cleanly. Counting is not automatically the better choice: it trades a fixed alphabet assumption for linear character processing.

## Boundaries and Common Mistakes

- The key for an empty string contains 26 zeros. Multiple empty strings belong together; do not skip them.
- Preserve multiplicities. Using only `set(word)` would incorrectly merge `aab` and `abb`.
- Return lists of original words, not the sorted keys. A key locates a group; it is not the output value.
- `ord(char) - ord("a")` depends on the guaranteed lowercase English letters. For mixed case, Unicode, or another alphabet, reconsider the key representation and any normalization rules.
- Output order is unrestricted. The order shown in the assertions follows this implementation's traversal; it is not an additional problem requirement.

## Takeaways

The key to this problem is not memorizing a `defaultdict` template. It is working out which kind of equality means that two items belong in the same group:

1. Compare sorted character sequences to recognize two anagrams.
2. Scan group representatives to turn that test into a first complete solution.
3. Use the sorted sequence as a dictionary key to eliminate the linear search through groups.
4. Use a count tuple under the fixed 26-letter constraint to avoid sorting and obtain the final `Solution`.

As an exercise, keep the slower Step 2 implementation as a reference for small test cases and compare its groups with the final version's results. The optimization changes how we locate groups, not what the groups mean.

## References

- [LeetCode 49: Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- [Doocs problem and solution mirror](https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0049.Group%20Anagrams/README_EN.md)
