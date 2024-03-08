---
# Must present
author: Ruoqing He
title: Rust Sort
# Optional
date: 2024-03-08
summary: An illustration of Rust's sort implementation.
---

## Rust Sort

"Sort" and the like in Rust is usually invoked by `slice`s. "Sort" here can be divided into two categories: stable sort (which is the default sort) and unstable sort. Their difference is whether the equal elements are reordered (or swapped) during sorting.

### Stable Sort

`sort()` and the like are implemented based on [*TimSort*](https://en.wikipedia.org/wiki/Timsort), which is basically a combination of *merge sort* and *insertion sort*.

### Unstable Sort

`sort_unstable()` and the like are implemented based on [*pattern-defeating quick sort*](https://github.com/orlp/pdqsort), which has an acceptable fast worst case of *heap sort*.

By calling `sort_unstable()`, it is directed to `sort::quicksort()`.

Inside `quicksort()`:

- The passed in element type is compared against `T::IS_ZST` (is zero sized?). `IS_ZST` is defined by `const IS_ZST: bool = size_of::<Self>() == 0`.
- `limit` is calculated to limit the number of imbalanced partitions to `floor(log2(len)) + 1`, by `usize::BITS - v.len().leading_zeros();`. The code here is calculating bits occupied by the number. If the `limit` is exceeded or zero, it will switch to *heap sort*.
- Start `recurse()` with `limit`, `is_less` function used to determine `lt` and `[T]` slice to sort the slice recursively.

Inside `recurse()`:

1. Slice of length 2-20 is directly sorted by *insertion sort*.
2. `limit` gets decremented every time the last partitioning was imbalanced, which implies a bad pivot was chosen. When `limit` reaches during the recursion (which means too many bad pivot choices were made), the slice will be sorted by *heap sort*.
