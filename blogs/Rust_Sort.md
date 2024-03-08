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
