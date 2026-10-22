---
title: "LeetCode 609: Find Duplicate File in System, from Parsing Records to Grouping by Content"
subtitle: "Decode name(content), then replace pairwise comparisons with a map from contents to full paths"
date: 2026-10-10T10:00:00+08:00
draft: false
categories: ["LeetCode"]
tags: ["hash-table", "string", "grouping", "LeetCode 609"]
description: "Solve LeetCode 609 step by step: parse directory descriptions, build a correct pairwise baseline, group by content, and account for path construction and real-filesystem limitations."
summary: "Duplicate files share contents, but the answer contains full paths. Five connected checkpoints build parsing, correct grouping, hash-table optimization, and a submission-ready solution."
keywords: ["LeetCode 609", "Find Duplicate File in System", "duplicate files", "hash grouping", "Python"]
readingTime: 12
---

## Problem: Compare Contents, Return Full Paths

You are given an array of strings, `paths`. Each string describes one directory and its files:

```text
directory_path filename(content) filename(content) ...
root/a 1.txt(abcd) 2.txt(efgh)
```

Group files whose **contents are exactly equal**, and return the full paths of the files in each group. Every returned group must contain at least two files; omit contents that occur in only one file. The order of groups and the order of paths within each group do not matter.

For example:

```text
Input:
[
  "root/a 1.txt(abcd) 2.txt(efgh)",
  "root/c 3.txt(abcd)",
  "root/c/d 4.txt(efgh)",
  "root 4.txt(efgh)"
]

One valid output:
[
  ["root/a/1.txt", "root/c/3.txt"],
  ["root/a/2.txt", "root/c/d/4.txt", "root/4.txt"]
]
```

Here, `1.txt` and `3.txt` have different names but the same contents, `abcd`, so they belong together. Conversely, matching filenames do not imply matching contents. A directory can also be just `root`; do not assume that it contains a subdirectory.

The main constraints are:

- `1 <= paths.length <= 2 * 10^4`.
- Each directory description has at most `3000` characters, and all descriptions together have at most `5 * 10^5` characters.
- Each description contains at least one file. A single space separates the directory and file entries.
- Each description refers to a different directory. Files or directories within the same directory do not share a name.

These strings already follow the problem's encoding. The program does not need to open directories on disk. This tutorial uses Python 3.10+ and assumes familiarity with loops, lists, and dictionaries.

## Step 1: Parse Just One File

Start with a smaller question: **given the directory `root/a` and the entry `1.txt(abcd)`, can we recover the full path and the contents?**

Our current baseline is only the input and output format. Comparing the entire entry `1.txt(abcd)` would also compare filenames, incorrectly separating it from `3.txt(abcd)`. Taking only the filename would not tell us whether the contents match either.

Make one change: split the entry into the full path we must return and the contents we must compare.

```python
directory = "root/a"
entry = "1.txt(abcd)"

name, content_and_end = entry.split("(", 1)
content = content_and_end[:-1]
full_path = directory + "/" + name

assert name == "1.txt"
assert content == "abcd"
assert full_path == "root/a/1.txt"
assert "root" + "/" + name == "root/1.txt"
assert "root/a/b" + "/" + name == "root/a/b/1.txt"
```

`split("(", 1)` splits only at the first opening parenthesis, producing `1.txt` and `abcd)`. The slice `[:-1]` removes the final closing parenthesis that ends the entry. Do not use `strip("()")`: we want to remove delimiters at specific positions, not an arbitrary run of characters at either end.

This parser relies on the supplied `filename(content)` format. It is not a general parser for arbitrary operating-system filenames, escapes, or contents containing spaces.

**Checkpoint:** We can now turn one file entry into `(full_path, content)`. The program still handles only a single hardcoded file; it cannot traverse the complete input yet.

## Step 2: Expand Every Directory into File Records

The current version parses one file, but the first line of the example already contains two. Another file with matching contents may appear in a later directory. Taking only the first file on each line would miss `root/a/2.txt`.

This step does not decide which files are duplicates. Put the existing parser inside two loops: the outer loop reads directory descriptions, and the inner loop reads the files in that directory. Store `(full_path, content)` pairs in `records` so the next step can compare all the files.

```python
def collect_files(paths: list[str]) -> list[tuple[str, str]]:
    records = []

    for line in paths:
        parts = line.split(" ")
        directory = parts[0]

        for entry in parts[1:]:
            name, content_and_end = entry.split("(", 1)
            content = content_and_end[:-1]
            full_path = directory + "/" + name
            records.append((full_path, content))

    return records


example_paths = [
    "root/a 1.txt(abcd) 2.txt(efgh)",
    "root/c 3.txt(abcd)",
    "root/c/d 4.txt(efgh)",
    "root 4.txt(efgh)",
]

assert collect_files(example_paths) == [
    ("root/a/1.txt", "abcd"),
    ("root/a/2.txt", "efgh"),
    ("root/c/3.txt", "abcd"),
    ("root/c/d/4.txt", "efgh"),
    ("root/4.txt", "efgh"),
]
```

`parts[0]` is the directory, not a file, so the inner loop starts at `parts[1:]`. Splitting on spaces still relies on the problem's serialized format; it is not suitable for parsing arbitrary real file contents.

The assertion checks that four directory descriptions expand into exactly five file records. The second file on the first line is retained, and the directory containing only `root` receives the correct `/` separator too.

**Checkpoint:** Every file now has contents we can compare and a path we can return. We still need to decide which records belong to the same duplicate group.

## Step 3: Build a Correct Pairwise Baseline

Our baseline is now `collect_files`, which returns every `(path, content)` pair. The most direct next step is to choose one record, compare its contents with each later record, and collect the matches.

There is a specific problem to resolve first. If three files all contain `abcd`, returning the pairs `(A, B)` and `(A, C)` would split one duplicate group into two overlapping groups. Each record must belong to just one content group, and groups with only one member must stay out of the answer.

Add `assigned` to remember whether a record already belongs to an earlier group. Choose each unassigned record as a `seed`, scan the later records, and append matching records to the current `group`. Mark each appended record as assigned immediately.

```python
def find_duplicate_pairwise(paths: list[str]) -> list[list[str]]:
    records = collect_files(paths)
    assigned = [False] * len(records)
    result = []

    for i, (seed_path, seed_content) in enumerate(records):
        if assigned[i]:
            continue

        assigned[i] = True
        group = [seed_path]

        for j in range(i + 1, len(records)):
            if not assigned[j] and records[j][1] == seed_content:
                assigned[j] = True
                group.append(records[j][0])

        if len(group) > 1:
            result.append(group)

    return result


def normalize(groups: list[list[str]]) -> list[tuple[str, ...]]:
    return sorted(tuple(sorted(group)) for group in groups)


three_equal = [
    "root/a 1.txt(abcd)",
    "root/b 2.txt(abcd)",
    "root/c 3.txt(abcd)",
]
assert normalize(find_duplicate_pairwise(three_equal)) == normalize([
    ["root/a/1.txt", "root/b/2.txt", "root/c/3.txt"],
])
assert find_duplicate_pairwise(["root 1.txt(a) 2.txt(b)"]) == []
assert find_duplicate_pairwise([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []
assert normalize(find_duplicate_pairwise([
    "root 1.txt(a)",
    "other 1.txt(a) 2.txt(b)",
])) == normalize([["root/1.txt", "other/1.txt"]])
assert normalize(find_duplicate_pairwise(example_paths)) == normalize([
    ["root/a/1.txt", "root/c/3.txt"],
    ["root/a/2.txt", "root/c/d/4.txt", "root/4.txt"],
])
```

`normalize` is only a test helper: sorting removes irrelevant ordering differences while preserving repeated paths or repeated groups, so an accidental duplicate cannot disappear from the check. It is not part of the grouping algorithm.

For the three-equal case, record A starts a group and is marked assigned. Its scan adds B and C, marking both assigned too. The outer loop then skips B and C, so it returns `[A, B, C]` exactly once. The condition `len(group) > 1` enforces the requirement that a returned group contain at least two files.

**Checkpoint:** This is a complete, correct baseline. Files with different names but equal contents merge; files with the same name but different contents stay separate; singleton groups are omitted. The remaining weakness is repeated work: when all contents differ, almost every seed scans all the records after it.

## Step 4: Use Contents as the Grouping Key

The previous version is correct, but distinct contents force it to revisit the same set of records. The first seed checks every later record; the second checks nearly all the remaining records again. For `F` files with distinct contents, that is `F * (F - 1) / 2` candidate comparisons before accounting for the cost of comparing strings.

We already know that `content` alone decides which group a file belongs to. Instead of searching for matching records, put each path directly into the bucket for its contents.

Replace only the group-building mechanism. Keep `collect_files` and the singleton filter unchanged:

```python
def find_duplicate_by_content(paths: list[str]) -> list[list[str]]:
    records = collect_files(paths)
    groups_by_content: dict[str, list[str]] = {}

    for full_path, content in records:
        if content not in groups_by_content:
            groups_by_content[content] = []
        groups_by_content[content].append(full_path)

    return [
        group
        for group in groups_by_content.values()
        if len(group) > 1
    ]


expected = normalize([
    ["root/a/1.txt", "root/c/3.txt"],
    ["root/a/2.txt", "root/c/d/4.txt", "root/4.txt"],
])
assert normalize(find_duplicate_by_content(example_paths)) == expected
assert normalize(find_duplicate_by_content(three_equal)) == normalize([
    ["root/a/1.txt", "root/b/2.txt", "root/c/3.txt"],
])
assert find_duplicate_by_content(["root 1.txt(a) 2.txt(b)"]) == []
assert find_duplicate_by_content([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []

# Build inputs and expected groups separately from structured test records.
import random

rng = random.Random(609)
for _ in range(20):
    structured = []
    for index in range(rng.randint(1, 6)):
        directory = f"d{index}"
        for file_index in range(rng.randint(1, 4)):
            content = f"c{rng.randrange(5)}"
            structured.append((directory, f"f{file_index}.txt", content))

    generated_paths = []
    for directory in sorted({row[0] for row in structured}):
        entries = [
            f"{name}({content})"
            for row_directory, name, content in structured
            if row_directory == directory
        ]
        generated_paths.append(" ".join([directory, *entries]))

    oracle = {}
    for directory, name, content in structured:
        oracle.setdefault(content, []).append(f"{directory}/{name}")
    expected_random = normalize([
        file_paths
        for file_paths in oracle.values()
        if len(file_paths) > 1
    ])
    assert normalize(find_duplicate_by_content(generated_paths)) == expected_random
    assert normalize(find_duplicate_pairwise(generated_paths)) == expected_random
```

Trace the first three records from the official example. `root/a/1.txt` creates the `abcd` bucket; `root/a/2.txt` creates the `efgh` bucket; then `root/c/3.txt` goes directly into the existing `abcd` bucket. No scan of earlier file records is needed.

Each record now uses a small, fixed number of dictionary operations and one append. Equal contents lead to the same bucket, while filenames appear only inside the stored paths. The condition `len(group) > 1` still excludes singletons, so the optimization does not change the output rule.

Why does this cover every duplicate group? Records with equal contents use equal dictionary keys, so all of them enter the same bucket. Records with different contents use unequal keys, so they cannot share a bucket. Each bucket is therefore exactly one content-equivalence class. Python dictionaries check key equality as well as hashes; a hash collision does not turn different content strings into the same key.

**Checkpoint:** We have removed the explicit pairwise scan and can group directly by content under the usual hash-table assumptions. The remaining work is to package the same logic as `Solution.findDuplicate` and account for input characters, constructed path characters, and hashing costs.

## Step 5: Deliver the Submission and State Its Costs Precisely

The previous version already groups correctly, but LeetCode expects `Solution.findDuplicate`. Put the parser and content buckets we have already checked inside that method.

There is one small structural change besides the wrapper: inline the parsing loop and append each path to its content bucket immediately, removing the intermediate `records` list. Parsing rules, grouping keys, and the singleton filter remain unchanged; the two former loops now connect directly without storing a separate copy of their handoff records.

```python
class Solution:
    def findDuplicate(self, paths: list[str]) -> list[list[str]]:
        groups_by_content: dict[str, list[str]] = {}

        for line in paths:
            parts = line.split(" ")
            directory = parts[0]

            for entry in parts[1:]:
                name, content_and_end = entry.split("(", 1)
                content = content_and_end[:-1]
                full_path = directory + "/" + name

                if content not in groups_by_content:
                    groups_by_content[content] = []
                groups_by_content[content].append(full_path)

        return [
            group
            for group in groups_by_content.values()
            if len(group) > 1
        ]


def normalize(groups: list[list[str]]) -> list[tuple[str, ...]]:
    return sorted(tuple(sorted(group)) for group in groups)


solution = Solution()
official_paths = [
    "root/a 1.txt(abcd) 2.txt(efgh)",
    "root/c 3.txt(abcd)",
    "root/c/d 4.txt(efgh)",
    "root 4.txt(efgh)",
]
official_expected = normalize([
    ["root/a/1.txt", "root/c/3.txt"],
    ["root/a/2.txt", "root/c/d/4.txt", "root/4.txt"],
])
assert normalize(solution.findDuplicate(official_paths)) == official_expected
assert solution.findDuplicate([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []
assert normalize(solution.findDuplicate([
    "deep/directory 1.txt(x) 2.txt(x)",
])) == normalize([[
    "deep/directory/1.txt",
    "deep/directory/2.txt",
]])
assert normalize(solution.findDuplicate([
    "root/a 1.txt(same)",
    "root/b 2.txt(same)",
    "root/c 3.txt(same)",
])) == normalize([[
    "root/a/1.txt", "root/b/2.txt", "root/c/3.txt",
]])
assert solution.findDuplicate([
    "root/a 1.txt(a)",
    "root/b 2.txt(b)",
    "root/c 3.txt(c)",
]) == []

import random

rng = random.Random(6090)
for _ in range(40):
    structured = []
    for directory_index in range(rng.randint(1, 7)):
        directory = f"random/{directory_index}"
        for file_index in range(rng.randint(1, 5)):
            structured.append(
                (directory, f"file{file_index}.txt", f"content{rng.randrange(8)}")
            )

    generated_paths = []
    directories = sorted({directory for directory, _, _ in structured})
    for directory in directories:
        entries = [
            f"{name}({content})"
            for row_directory, name, content in structured
            if row_directory == directory
        ]
        generated_paths.append(" ".join([directory, *entries]))

    oracle = {}
    for directory, name, content in structured:
        oracle.setdefault(content, []).append(f"{directory}/{name}")
    expected_random = normalize([
        file_paths
        for file_paths in oracle.values()
        if len(file_paths) > 1
    ])
    assert normalize(solution.findDuplicate(generated_paths)) == expected_random
```

This block is self-contained. Its `normalize` function and generated cases are tests, not part of the submitted `Solution` class.

### Correctness: Each Bucket Is One Content-Equivalence Class

After processing any prefix of the file records, each bucket contains exactly the full paths of the processed files whose contents equal its key, each included once. Initially there are no records and no buckets. Processing the next file preserves this invariant by appending its full path once to the bucket selected by its exact contents.

After all records have been processed:

1. Files with equal contents use equal keys, so they appear in the same bucket.
2. Files with different contents use unequal keys, so they cannot appear in the same bucket.
3. Returning only buckets larger than one removes exactly the singleton groups the problem excludes.

Together these establish completeness and separation: no duplicate group is missed, and no group contains unequal contents. Filenames and directories only construct return paths; they do not determine grouping.

### Complexity: Count Input Characters and Every Constructed Path

Let:

- `S` be the total number of characters in the input directory descriptions;
- `F` be the number of file records;
- `P` be the total number of characters in **all constructed full paths**, including paths in singleton buckets that will not be returned.

Parsing the descriptions and copying their filename/content pieces takes `O(S)` time. Hashing the content strings is not free: summed across the files, their lengths are bounded by `S`. Under the usual average-case hash-table assumptions, the dictionary operations and content equality checks take expected `O(S + F)` time in total, accounting for string lengths. Constructing full paths takes `O(P)`, and filtering the buckets takes `O(F)`.

Because each file record contributes characters to the input, `F <= S`, so the solution takes **average `O(S + P)` time**. Its auxiliary storage, including the grouped paths and returned lists, is **`O(S + P)`**: content keys, temporary parsed strings, dictionary/list bookkeeping, and all stored full paths fit within that bound. The returned groups refer to existing path strings; they do not copy every string again.

Do not drop `P`. One long directory prefix occurs only once in its input description but is repeated in every full path built for that directory. Counting only directory descriptions also misses that one description can hold many files. These bounds concern `Solution.findDuplicate`, not the sorting and random-input generation used by the tests.

Hash tables can still degrade in the worst case. The expected bound assumes ordinary hashing behavior; it does not make string hashing constant-time or describe the I/O cost of scanning a real disk.

### What Changes in a Real Filesystem?

The problem has already done directory traversal for us. For a real filesystem, additional decisions include:

- How to traverse directories with DFS or BFS, handle permission errors, and avoid cycles through symbolic links.
- How to group large files by size, then use digests computed in chunks to narrow down candidate duplicates.
- How to compare candidate files byte by byte before declaring them equal. Matching fixed-length digests are evidence of a possible match, not a mathematical guarantee of equal contents.
- How to update a digest incrementally when only 1 KB can be read at a time, instead of loading each complete file into memory.

These belong to a separate filesystem-scanning problem, not to the serialized-string parser developed here.

**Checkpoint:** The platform method is runnable, its grouping invariant and character costs are explicit, and its tests cover duplicates and singleton cases. It solves the supplied-string problem; real directory traversal and file I/O remain outside its scope.

## Summary

The useful progression is not simply "see duplicates, use a hash table":

1. Establish that **complete contents**, not names, define a duplicate.
2. Parse each `filename(content)` entry into contents and a full path.
3. Build a correct pairwise baseline with `assigned`, so each duplicate group is returned once.
4. Use the repeated-scan bottleneck to justify promoting contents to a dictionary key.
5. Keep the singleton filter, and account for path characters, hashing assumptions, and real-filesystem boundaries before calling the solution complete.

When input already describes records and their attributes, first turn it into clear data and then choose the grouping key. As a follow-up exercise, construct one long directory description containing many short filenames and compare its input length with the total length of the full paths. That makes the separate `P` term concrete.

## References

- [LeetCode 609: Find Duplicate File in System](https://leetcode.com/problems/find-duplicate-file-in-system/)
- [Problem statement mirror (Doocs)](https://raw.githubusercontent.com/doocs/leetcode/main/solution/0600-0699/0609.Find%20Duplicate%20File%20in%20System/README_EN.md)
