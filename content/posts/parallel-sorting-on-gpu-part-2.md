---
title: "Parallel Sorting on GPU: Part 2"
subtitle: ""
date: 2021-01-01T04:37:56+03:00
draft: true
description: ""

tags: []
categories: []

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

This post is dedicated to the description and implementation of sorting array on a graphics processing unit. It'll be split into two parts.

In this part, we'll try to derive one of the first efficient parallel sorting algorithm from the [sorting networks](https://en.wikipedia.org/wiki/Sorting_network) class. What's specific is that before doing some sorting algorithm from this class, we know in advance which fixed positions elements we'll compare-and-swap at. Then we'll calculate the algorithm's serial and parallel time complexities.


## Elimination of restriction on array size

What if the size of the array $a$ (let it be $n$) to be sorted is not of $\lbrace 2^k\ |\ k \ge 0 \rbrace$? Then we can convert it to the following form:

$$a^\prime[i]= \begin{dcases} a[i],& 1\le i\le n\\\ \infty,& n < i\le n^\prime \end{dcases},$$
$$\text{where}\ n^\prime = \min \lbrace x\\, |\\, x \ge n,\\,x \in \lbrace 2^k\ |\ k \ge 0 \rbrace \rbrace$$

and sort $a^\prime$. Moreover, it's not necessary to convert it programmatically: we can just check if some of the elements to be compared are out of bound, and if that's the case, do nothing. 

