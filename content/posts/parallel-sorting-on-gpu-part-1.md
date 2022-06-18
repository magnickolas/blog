---
title: "Parallel Sorting on GPU: Part 1"
subtitle: ""
date: 2021-02-11T04:37:56+03:00
draft: false
description: "Description and explanation of Batcher's sorting algorithm for graphics processing units (GPUs)."

tags: [sorting, gpu]
categories: [algorithms]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: true
lightgallery: false
license: ""
---

This post series is dedicated to the description and implementation of sorting an array with a graphics processing unit (GPU). It'll be split into two posts.

In this part, we'll first try to derive one of the earliest comparably efficient parallel sorting algorithm from the [sorting networks](https://en.wikipedia.org/wiki/Sorting_network) class. What's remarkable for this class of sorting algorithms, is that before performing the algorithm, we already know the whole sequence of pairs of indices to be compared and swapped (if needed).

Then we'll calculate the algorithm's theoretical serial and parallel time complexities.

## Batcher's algorithm's scheme description

For simplicity of visualization let the array be of 8 elements, the idea's able to be to be scaled for any size from $\lbrace 2^k\ |\ k \ge 0 \rbrace$.

Here our array is (the odd elements are colored green, the even ones are gray):

<img src="/images/parallel-sorting-on-gpu-part-1/array.svg" width="100%">

**1. Let's sort the first and the second halves of the array separately.**

Now our array is partially sorted. Arrow means: the source of it is not greater than the destination.

<img src="/images/parallel-sorting-on-gpu-part-1/halves.svg" width="100%">

Now we're facing a problem of sorting <ins>an array that has its halves sorted</ins>.

First, we're going to solve the same problem for all odd (green) and even (gray) elements separately. Two additional problems are now here, but then we say magic words \*\*divide and conquer\*\* and they're already solved!

**2. Sort all the odd and all the even elements separately**

After approaching these two subproblems we've obviously got knowledge of elements being sorted in the odd/even partition:

<img src="/images/parallel-sorting-on-gpu-part-1/halves_sorted.svg" width="100%">

But previously we've sorted the halves of the array, and it may give us some extra information. Let's examine it by writing out some facts:

- Every gray element has some green element that is less or equal to it. Consequently, after sorting both subarrays, every gray element can't be less than the previous green one.

<img src="/images/parallel-sorting-on-gpu-part-1/halves_sorted_pairs.svg" width="100%">

- Every (except $1^{\text{st}}$ and $5^{\text{th}}$) green element has some gray element that is less or equal to it. Consequently, after sorting both subarrays, every green element can't be less than the grey one three positions before it.

<img src="/images/parallel-sorting-on-gpu-part-1/halves_odd_even.svg" width="100%">

If we look now at the picture, it can be determined that the first and the last elements are in their right positions. Between them, there are consequent non-overlapping pairs such that if we group them into "super-elements", the array would be sorted (e.g. $a[2] \le a[4],a[5]$ and $a[3] \le a[4],a[5]$).

Thus, the rest is to compare and probably swap the following pairs: $(2,3), (4,5), (6,7)$.

**3. Sort all consequent non-overlapping pairs starting from the second element and to the last but one.**

<img src="/images/parallel-sorting-on-gpu-part-1/sorted.svg" width="100%">

## Elimination of restriction on array size

What if the size of the array $a$ (let it be $n$) to be sorted is not of $\lbrace 2^k\ |\ k \ge 0 \rbrace$? Then we can convert it to the following form:

$$a^\prime[i]= \begin{dcases} a[i],& 1\le i\le n\\\ \infty,& n < i\le n^\prime \end{dcases},$$
$$\text{where}\ n^\prime = \min \lbrace x\\, |\\, x \ge n,\\,x \in \lbrace 2^k\ |\ k \ge 0 \rbrace \rbrace$$

and sort $a^\prime$. Moreover, it's not necessary to convert it programmatically: we can just check if some of the elements to be compared are out of bound, and if that's the case, do nothing.

## Serial time complexity

Let $T(n)$ denote the time complexity of sorting an array of $n$ elements by the described algorithm. And $T_h(n)$ will denote time complexity of sorting an array that has its halves already sorted.

- Calculate $T_h(n)$:

$$T_h(n) = 2\cdot T_h\left(\frac{n}{2}\right) + O(n)$$
$$T_h(n) = O(n\cdot \log(n))$$

- And $T(n)$:

$$T(n) = 2\cdot T\left(\frac{n}{2}\right) + T_h\left(n\right)$$
$$T(n) = 2\cdot T\left(\frac{n}{2}\right) + O(n\cdot \log(n))$$
$$T(n) = O(n\cdot \log^2(n))$$

The serial time complexity is $O(n\cdot \log^2(n))$.

## Parallel time complexity

Suppose that we have <ins>an infinite number</ins> of compute cores (so that we're able to run an arbitrary amount of them) and want to parallelize our sorting algorithm. Let's notice, that:
- for an array of size $n$ we compute $O(\log^2(n))$ sequentially dependent layers in the recursion tree;
- each layer performs $O(n)$ independent of each other comparisons-and-swaps on the array;
- since we can run $O(n)$ of compute cores for each layer, the time complexity for a single layer is $O(1)$.

Thus, the total time complexity will be $O(\text{number of layers})$ = $O(\log^2(n))$.

## Running the algorithm on GPU

We're gonna look at an efficient implementation in the [second part][part-2]. To achieve $O(1)$ of extra memory, we will transform the algorithm into an iterative form.

[part-2]: {{< ref "parallel-sorting-on-gpu-part-2" >}}
