---
title: "The k<sup>th</sup> Greatest Element In Two Sorted Arrays"
---

Given two sorted arrays `A` and `B`, find the k<sup>th</sup> greatest element of all the elements.

Take the i<sup>th</sup> element from array `A` and the j<sup>th</sup> element from array `B` so that `i + j == k`, we
can see that if `B[j] <= A[i] <= B[j+1]`, then `A[i]` must be the k<sup>th</sup> element of all elements in `A` and
`B`. So now we have two scenarios to consider:

1. `A[i] == B[j]`, then either `A[i]` or `B[j]` is the k<sup>th</sup> element
2. `A[i] < B[j]`, then the k<sup>th</sup> element must be in `A[i+1..A.length]` or `B[1..j]`
3. `A[i] > B[j]`, this is symmetric to case 2

Case 1 is obvious, for case 2, if `A[i-p]` is the k<sup>th</sup> element, then `A[i-p]` must be greater than or equal
to `j + p` elements in `B`, so `A[i-p] >= B[j+p]`, since `A[i] < B[j]`, then `A[i-p] < B[j] < B[j+p]`, which
contradicts to `A[i-p] >= B[j+p]`, so the k<sup>th</sup> element must be in `A[i+1..A.length]`. Same proof can be used
to show that the k<sup>th</sup> element must be in `B[1..j]`.

```
// assume that k is always less than A.length + B.length
def find_k_th_element(A, B, k)
    if A.length > B.length:
        swap(A, B) // make sure A is always the smaller one
    if A.length == 0:
        return B[k]
    if k == 1:
        return min(A[0], B[0])
    i = min(A.length, k / 2)
    j = k - i
    if A[i] == B[j]:
        return A[i]
    else if A[i] < B[j]:
        return find_k_th_element(A[i+1..A.length], B[1..j], k - i)
    else:
        return find_k_th_element(A[1..i], B[j+1..B.length], k - i)
```
