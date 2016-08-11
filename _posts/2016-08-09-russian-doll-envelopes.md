---
title: "Russian Doll Envelopes"
tags:
  - problem solving
---

Matryoshka doll is a set of nesting dolls of decresing size placed one inside another. Now we have a set of envelopes
of different sizes, given as `[[1, 2], [3, 4] ...]`, where each element is an array of two elements representing the
length and width of the enevelope, find out the maxmium number of nesting enevelopes.

For envelope `e1` to be nested in `e2`, they should satisfy `e1.length < e2.length` and `e1.width < e2.width`, which
means that the lengths and widths sould be ordered. Sort the input envelopes by length, now the problem becomes finding
the [longest increasing subsequence](https://en.wikipedia.org/wiki/Longest_increasing_subsequence) from the list of widths.

The LIS of an array can be solved in `O(nlogn)` using binary search. We can build up an array that contains some
information about current LIS while looping through the elements, and we want to be able to get the length of the LIS
from that array in the end. Given an array `A`, we maintain:

1. an array `L`, where `A.length == L.length`
2. a variable `l`, indicating the length of the LIS so far

To be able to update `L` while a longer LIS is found, we need to store in `L[l]` the last element in the LIS of length
`l`, so that when we have `A[i] > L[l]`, we will know that we found a longer LIS of length `l+1` ends with
`A[i]`. However, the LIS of an array is not unique. If we find another element `A[i+k]` such that
`L[l] < A[i+k] < A[i] == L[l+1]`, we would end up with 2 LISes of length `l+1`. In this case, we can either leave
`L[l]` as is or replace `L[l]` with `A[i+k]`, what do we do? Notice that if later on we find another element
`A[i+k] < A[i+k+q] < A[i]`, the length of LIS of `A[1..i+k+q]` will become `l+2`, but if `L[l+1]` is still `A[i]`, we
wouldn't be able to update since `L[l+1] == A[i] > A[i+k+q]`. So when we are at `i+k`, we need to replace `L[l]` with
`A[i+k]`. This leaves us with the following invariant:

    L[k] is the smallest last element among all LISes of length k

we can achieve it by the follwing pseudo code:

```
l = 0
L = new Array[A.length]
for i = 1 to A.length
    k = binary_search(L, 1, i, A[i])
    L[k] = A[i]
    if k > l
        l += 1
```

where `binary_search` takes an array to search, the start index, the to index and the target value, and returns the
index of the target, or the index to insert if the target is not found. To prove that the invariant holds:

Initialization
: both `A` and `lis` are empty, the invariant holds.

Maintenance
: suppose the result of the binary search is `k` and `L[k]` is the smallest last element of all LISes of `k`, so A[i]
will be the last element of an LIS of length `k`. If `k <= l`, `L[k] <= A[i]`, replacing `L[k]` with `A[i]` holds the
invariant. If `k > l`, `A[i]` will be the last element of the LIS of length `k+1`, updating `L[k+1]` to `A[i]` holds
the invariant.

Termination
: At the end of last loop, we will have `L` store all of the smallest last elements of the LISes of some length, at the
index equal to the length.

With the efficient algorithm of solving LIS, we now can use it to find the maxmium number of nesting enevelopes. One
caveat though is that when sorting the envelopes, we need to take into account the envelopes with the same length,
since we can only select one envelope in the group of envelopes with the same length. So in this case, we need to
further sort the group of envelopes by width in descending order to avoid multiple counting the LIS.
