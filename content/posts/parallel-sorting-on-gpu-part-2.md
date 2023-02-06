---
title: "Parallel Sorting on GPU: Part 2"
subtitle: ""
date: 2023-01-17T22:09:38+03:00
draft: false
description: "Implementation of Batcher's Odd-Even Mergesort algorithm for GPU with Vulkan API."

tags: [sorting, gpu, c++, vulkan]
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

This is the closing sequel to the [first part][part-1] and dedicated to
- the derivation of an iterative form of the Batcher's Odd-Even Mergesort algorithm;
- implementing it with a GPU shader;
- a little of benchmarks for fun.

## Theory preparation

### Reminder from the first post

The GPU's computation model drastically differs from the CPU one. In simplest terms, we have magnitutes more workers (cores) organized in multiple disjoint local groups. The best (but pretty utopian) way of utilizing this model is to move the data to all of them and let them process it independently (or as locally as possible), because communication between workers from different groups is quite expensive and is able to offset the entire potential speedup from using GPU compared to CPU. What's even more expensive is transferring data between computer memory (RAM) and GPU memory (DRAM).

We'll discuss how to make the previously discussed sorting network algorithm non-recursive to distribute computations between GPU cores effectively.

### Straightforward recursive implementation

For the sake of consistency with other descriptions of this algorithm, we will use the following naming:
- **sort(a)** sorts **a**
- **merge(a)** sorts **a** assuming that the left and right halves are sorted
<!-- - `sort(a, i, size)` sorts the slice `a[i : i + size]` -->
<!-- - `merge(a, i, size, stride)` sorts `a[i : i+size : stride]` assuming that `a[i: i+size/2 : stride]` and `a[i+size/2 : i+size : stride]` are sorted -->

Let's begin with a simple implementation that directly follows the algorithm as it was described in the first part. I'll use a Python snippet because this language is expressive and fairly easy to read.

```python
def sort(a: list[T]) -> list[T]:
    if len(a) == 1:
        return a
    mid = len(a) // 2
    return merge(sort(a[:mid]) + sort(a[mid:]))


def merge(a: list[T]) -> list[T]:
    if len(a) == 2:
        return sorted(a)
    first, *even = merge(a[::2])  # a[0], a[2], ...
    *odd, last = merge(a[1::2])  # a[1], a[3], ...
    return [first] + flatten(map(sorted, zip(even, odd))) + [last]


def flatten(a: Iterable[Iterable[T]]) -> list[T]:
    return [y for x in a for y in x]
```

This implementation doesn't change the passed array **a**, building a new one instead. As it's memory consuming and can't be efficiently projected into a non-recursive form, let's get rid of explicit memory allocations first.

### Transition to in-place form

To make the rest of this article easier to grasp, we'll rewrite the described above snippet to not create copies of the array. In order to do it, we'll transform and pass the state of the recursion.

First, consider **merge**: at the very top level it has a continuous fragment of the array **a**. At the second level it indexes two slices **[a[0], a[2], ...]** and **[a[1], a[3], ...]**, which can be expressed with a start index (0 or 1) and step 2. Similarly, for the third layer the step is going to be 4 and indexes are 0, 2, 1 and 3. We also need to know the size of the current slice to be able to know where it ends, which decreases by half when going down the recursion.

For **sort**, we always have continuous fragment, thus it's only necessary to know the start position and the size of it.

```python
def sort(a: list[T], start: int, size: int):
    if size > 1:
        mid = size // 2
        sort(a, start, mid)
        sort(a, start + mid, mid)
        merge(a, start, size, 1)


def merge(a: list[T], start: int, size: int, stride: int):
    if size == 2:
        compare_and_swap(a, start, start + stride)
    else:
        mid = size // 2
        double_stride = stride * 2
        merge(a, start, mid, double_stride)
        merge(a, start + stride, mid, double_stride)
        for k in range(1, size - 1, 2):
            left_idx = start + k * stride
            right_idx = left_idx + stride
            compare_and_swap(a, left_idx, right_idx)


def compare_and_swap(a: list[T], i: int, j: int):
    if a[i] > a[j]:
        a[i], a[j] = a[j], a[i]
```

To sort **a**, we need to call **sort(a, 0, len(a))**. Below is an illustration of the call stack, in which the numbers represent the order of execution, combined into independent layers. **cas** is an abbreviation for **compare_and_swap** and the calls for the fragments of size 2 are squashed into the **merge** which called it.

<div align="center">
<img src="/images/parallel-sorting-on-gpu-part-2/algo8.svg" width="60%">
</div>

### Example of an iterative form

Let\'s take a look at the recursion upside down (looking at the latest illustration) and first shape an intuition.

Consider an array of size 8. We can split the whole computation into the following stages:
- At the lowest level we have pairs **(a[0], a[1]), (a[2], a[3]), ..., (a[n-2], a[n-1])**, which are simply being ordered with **compare_and_swap**.
- Next, **merge** is being run for the continuous groups of size 4, which runs **merge** for odd and even subgroups of size 2, after which it orders consecutive pairs starting from the second element.
- Then, likewise, **merge** is being run for the only continuous group of size 8 (the whole array). It calls **merge** for odd and even subgroups of size 4, each of which call **merge** for "odd-odd", "odd-even", "even-odd" and "even-even" subgroups.

Below is a colorful image that illustrates the described process. If we order the connected elements row-by-row, we'll end up with a sorted array.


<div align="center">
<img src="/images/parallel-sorting-on-gpu-part-2/merge8.svg" width="80%">
</div>

### Deriving an iterative form in general case

Now, let's get our hands 👐 dirty.

From the [recursive in-place form](#transition-to-in-place-form) we know, that each top-level call of **merge** function starts from **stride=1** between the consecutive elements and doubles it when descending to the odd and even subgroups.

Since we know the stride, what's left is to find out if a specific index is the left one in some pair. If it's **i**, then the right one would be **i + stride**. Notice that after descending to the odd/even subgroup, we can normalize its indices by bitwise shifting one bit to the right (e.g. **[0, 2, 4, ...]** or **[1, 3, 5, ...]** would become **[0, 1, 2, ...]**). Let's name the normalized index as **i_loc** Then, since we know the size of the groups (let it be **size**), the predicate that determines if **i** is the left index for **size >= 4**  is the following:

<div align="center">
<img src="/images/parallel-sorting-on-gpu-part-2/predicate.svg" height="60%">
</div>

There's left a special case, when **size = 2**, when we only have two elements, which should be compared. An index is the left one if the normalized index is even:

<div align="center">
<img src="/images/parallel-sorting-on-gpu-part-2/predicate2.svg" height="60%">
</div>

Here is a diagram for an array of size 16 that can be meditated on:

<div align="center">
<img src="/images/parallel-sorting-on-gpu-part-2/merge16.svg" width="80%">
</div>


## Implementation

The full working implementation is located at [https://github.com/magnickolas/odd-even-mergesort][repo]. It's written with C++, [Vulkan API][vulkan] and [GLSL][GLSL] for the GPU compute shader.

### Non-recursive implementation

{{<admonition type="note" title="Dealing with restrictions on array size">}}
What if the size of the array is not a power of two? In this case we can pad it to the right with **∞** (infinity) value to the nearest power of two and crop it after the sorting. So we would pass **[a[0], a[1], ..., a[n-1], ∞, ..., ∞]**.

However, we can do without using extra memory. Let our algorithm just behave like there is an infinity value at all positions greater than **n-1** and do nothing in case of an upcoming comparison of values at those positions.
{{</admonition>}}

I've implemented it to run on GPU as a GLSL shader run once for each layer of computations, which is compiled to [SPIR-V][spirv] binary intermediate language and later utilized by Vulkan API.

In the following code the following predefined variables will be used:
- **n** is the size of the whole array
- **stride** is the space between the elements, as described in previous sections
- **stride_trailing_zeros** is **log2(stride)** (defined to replace **i / stride** operation with bitwise shift for optimization purposes)
- **inner_reminder** is **0** for **size=2** and **1** else
- **inner_last_idx** is **size-1** (defined to replace **i % size** operation with bitwise and)


```glsl
#version 460
#include "defs.h"

layout(local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

layout(set = 0, binding = 0) buffer Arr {
  int buf[];
} a;

layout(push_constant) uniform PushConstants {
  uint n;
  uint stride;
  uint stride_trailing_zeros;
  uint inner_reminder;
  uint inner_last_idx;
};

bool is_left_index(uint i) {
  uint inner_i = i >> stride_trailing_zeros;
  return (inner_i & 1) == inner_reminder &&
       (inner_i & inner_last_idx) < inner_last_idx;
}

uint get_right_index(uint i) {
  return i + stride;
}

/* Assumes that i < j */
void compare_and_swap(uint i, uint j) {
  if (j < n && a.buf[i] > a.buf[j]) {
    int t = a.buf[i];
    a.buf[i] = a.buf[j];
    a.buf[j] = t;
  }
}

void main() {
  uint i = gl_GlobalInvocationID.x;
  if (i >= n) {
      return;
  }
  if (is_left_index(i)) {
    uint j = get_right_index(i);
    compare_and_swap(i, j);
  }
}
```

Now let's take a look at the shader orchestration process which is rewritten in a *pseudocodish* manner (the real one in C++ is more intimidating and can be looked up [at the repo](https://github.com/magnickolas/odd-even-mergesort/blob/master/src/batcher_sort.cc)).

```
N := 1
while N < n:
    N *= 2

merge_group_size := 2
WHILE merge_group_size <= N:
    stride := merge_group_size / 2
    inner_rem := 0
    WHILE stride >= 1:
        # Count trailing zeros in binary form of `stride`
        stride_trailing_zeros = log2(stride);
        inner_last_idx =
            (merge_group_size / stride) - 1;

        CONSTANTS = [
            n,
            stride, stride_trailing_zeros,
            inner_rem, inner_last_idx
        ]
        PARALLEL_FOR i := 0..n-1:
            RUN_SHADER(i, CONSTANTS)

        # Starting from the second iteration, the inner index
        # should be odd to be the left one
        inner_rem = 1;
        stride /= 2
    }
    merge_group_size *= 2
```

In reality a combination of **PARALLEL_FOR** and **RUN_SHADER** is a dispatch process, that will send the constants to the GPU and spawn multiple local groups of workers. I've chosen the size of these local groups of 256 since it was empirically measured to produce the fastest computations.

The sorting will be performed in $O(log^2(n))$ steps that will only vary in a set of predefined constants, but run the same shader.
At each step, every index is paired with some other, which's being uniquely calculated.
It's nice that there's no data communication between GPU workers.

## Benchmarks

Since it wouldn't be fair to compare such a parallel algorithm directly to the CPU implementation, let's generate a large array, sort it with [std::sort](https://en.cppreference.com/w/cpp/algorithm/sort) and with our hand-crafted sorting algorithm on GPU and compare the results.

Tech specs:
- CPU: AMD Ryzen 5 2400G
- GPU: NVIDIA GeForce RTX 2070

```shell
$ ./batcher_sort 1000000000
GPU time difference:  9.16
CPU time difference:  102.24
```

The speedup is approximately **11.2x**.

## References
- [Lecture notes from MIT](https://math.mit.edu/~shor/18.310/batcher.pdf)
- [The Art of Computer Programming, Volume 3](https://www.amazon.com/Art-Computer-Programming-Sorting-Searching/dp/0201896850), Algorithm 5.2.2M
- [Repository with implementation][repo]


[part-1]: {{< ref "parallel-sorting-on-gpu-part-1" >}}
[GLSL]: https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language
[vulkan]: https://www.vulkan.org/
[spirv]: https://registry.khronos.org/SPIR-V/
[repo]: https://github.com/magnickolas/odd-even-mergesort
