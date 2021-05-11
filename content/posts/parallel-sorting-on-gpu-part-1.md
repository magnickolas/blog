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

This post series is dedicated to the description and implementation of sorting array on a graphics processing unit. It'll be split into two parts.

In this part, we'll try to derive one of the first efficient parallel sorting algorithm from the [sorting networks](https://en.wikipedia.org/wiki/Sorting_network) class. What's specific is that before performing some sorting algorithm from this class, we already know which fixed positions elements we'll compare-and-swap at.

Then we'll calculate the algorithm's serial and parallel time complexities.

## Batcher's algorithm's scheme description

For simplicity of visualization let the array be of 8 elements, the idea's able to be to be scaled for any size from $\lbrace 2^k\ |\ k \ge 0 \rbrace$.

Here our array is (the odd elements are colored green, the even ones are gray):

![](/images/parallel-sorting-on-gpu/array.png)

**1. Let's sort the first and the second halves of the array separately.**

Now our array is partially sorted. Arrow means: the source of it is not greater than the destination.

![](/images/parallel-sorting-on-gpu/halves.png)

Now we're facing a problem of sorting an array that has its halves sorted.

First, we're going to solve the same problem for all odd (green) and even (gray) elements separately. Two additional problems are now here, but then we say magic words *\*divide and conquer\** and they're already solved!

**2. Sort all the odd and all the even elements separately**

After approaching these two subproblems we've obviously got knowledge of elements being sorted in the odd/even partition:

![](/images/parallel-sorting-on-gpu/halves_sorted.png)

But previously we've sorted the halves of the array, and it may give us some extra information. Let's examine it by writing out some facts:

- Every gray element has some green element that is less or equal to it. Consequently, after sorting both subarrays, every gray element can't be less than the previous green one.

![](/images/parallel-sorting-on-gpu/halves_sorted_pairs.png)

- Every (except $1^{\text{st}}$ and $5^{\text{th}}$) green element has some gray element that is less or equal to it. Consequently, after sorting both subarrays, every green element can't be less than the grey one three positions before it.

![](/images/parallel-sorting-on-gpu/halves_odd_even.png)

If we look now at the picture, it can be determined that the first and the last elements are in their right positions. Between them, there are consequent non-overlapping pairs such that if we group them into "super-elements", the array would be sorted (e.g. $a[2] \le a[4],a[5]$ and $a[3] \le a[4],a[5]$).

Thus, the rest is to compare and probably swap the following pairs: $(2,3), (4,5), (6,7)$.

**3. Sort all consequent non-overlapping pairs starting from the second element and to the last but one.**

![](/images/parallel-sorting-on-gpu/sorted.png)

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

Suppose that we have an infinite number of compute cores (so we can pick arbitrary amount of them to run simultaneously) and want to parallelize our sorting algorithm. Let's notice that for array of size $n$ we compute $O(\log^2(n))$ layers that are dependent in recursion tree. Each layer in this recursion performs many independent comparisons and swaps for the whole array. Thus, since the cost of one layer is $O(1)$, the total time complexity will be $O(\log^2(n))$.

## Running algorithm on GPU

We're gonna look at the implementation in the second part. To achieve constant extra memory, the algorithm will be a non-recursive version of Batcher's sorting algorithm.
