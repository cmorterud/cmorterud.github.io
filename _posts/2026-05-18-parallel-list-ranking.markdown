---
layout: post
title:  "Parallel List Ranking with OpenMP and CUDA"
date:   2026-05-18 09:00:00 -0400
categories: systems
---

# Introduction
List ranking is the problem of assigning each node in a linked list its
distance from the head of the list.

For example, suppose the list is represented by an array of `next` pointers:

```text
head = 2

index:  0   1   2   3   4
next:   4   0   1  NIL  3
```

This represents the list:

```text
2 -> 1 -> 0 -> 4 -> 3
```

The rank of each node is:

```text
rank[2] = 0
rank[1] = 1
rank[0] = 2
rank[4] = 3
rank[3] = 4
```

Sequentially, this is straightforward. Start at the head, walk the list, and
increment a counter.

```c
long current = head;
long r = 0;

while (current != NIL) {
    rank[current] = r++;
    current = next[current];
}
```

The parallel version is more interesting because a linked list is not
naturally parallel. Each node points to the next node, and the next node is
where the work continues.

This post is about the OpenMP and CUDA list ranking project I worked on in a
graduate parallel computing course.

## Wyllie's Algorithm
Wyllie's algorithm uses pointer jumping. Each node starts with a rank of 1,
except the head, which starts with rank 0. In each round, nodes update their
successor pointer to skip ahead in the list.

Conceptually:

```c
rank[next[i]] += rank[i];
next[i] = next[next[i]];
```

After each round, the effective distance jumped by each node doubles. This
means the algorithm finishes in `O(log n)` rounds.

The advantage is that each round can be parallelized across the array:

```c
#pragma omp parallel for
for (size_t i = 0; i < n; ++i) {
    if (next[i] != NIL) {
        // update rank and jump pointer
    }
}
```

The disadvantage is memory traffic. Each round touches large arrays, and the
algorithm needs multiple rounds. For a large list, the cost is not just the
number of operations. It is also the number of times the algorithm streams
through memory.

That was the first useful lesson from the project: a parallel algorithm can
have good asymptotic depth and still be limited by memory movement.

## Helman-JaJa
The Helman-JaJa algorithm takes a different approach. Instead of repeatedly
jumping every pointer, it partitions the list into sublists.

The rough steps are:

1. choose several sublist heads
2. walk each sublist in parallel
3. compute the rank of the sublist heads
4. add the sublist head rank to each node in the sublist

The point is to trade many global pointer-jumping rounds for fewer parallel
walks over sublists.

For example, if the full list is:

```text
2 -> 1 -> 0 -> 4 -> 3 -> 6 -> 5
```

We may choose sublist heads like:

```text
2, 4, 6
```

Then the sublists are:

```text
2 -> 1 -> 0
4 -> 3
6 -> 5
```

Each sublist can be walked independently to compute local ranks. After that,
the algorithm needs to know where each sublist fits in the full list. That
part is smaller and can be handled by ranking the sublist heads.

The final pass adds the offset for each sublist:

```c
rank[node] = local_rank[node] + rank[sublist_head];
```

This algorithm was more involved than Wyllie's algorithm, but it also made
the performance problem clearer. The choice of sublist count matters. Too few
sublists means there is not enough parallel work. Too many sublists means
more overhead and more sequential work to stitch the sublists together.

## OpenMP
The OpenMP implementation was a shared-memory version. The input arrays were
already in host memory, and the parallel loops were over the list nodes or
sublist heads.

The simplest part of OpenMP is also why it is useful:

```c
#pragma omp parallel for
for (size_t i = 0; i < num_sublists; ++i) {
    // walk one sublist
}
```

The harder part is deciding what work belongs in the parallel loop. A loop is
not automatically worth parallelizing. The work must be large enough to
offset scheduling overhead, and the memory access pattern must not dominate
the runtime.

List ranking is a good example because the data structure fights the machine.
Arrays are cache-friendly. Linked lists are not. Even though the list is
stored in arrays, following `next` still creates irregular access.

## CUDA
The CUDA version used the same Helman-JaJa idea, but the cost model changed.
The GPU is good at running many simple operations in parallel, but data has
to be copied to device memory, kernels need to be launched, and irregular
memory access is still expensive.

The CUDA version had separate work for:

* initializing arrays on the device
* computing sublist ranks
* computing the ranks of sublist heads
* applying sublist offsets
* copying the final rank array back

That makes the sublist count even more important. If there are not enough
sublists, the GPU does not have enough work. If there are too many, the
overhead of managing the sublists starts to dominate.

This was the second useful lesson from the project: moving an algorithm to a
GPU is not just a syntax change. The algorithm needs enough parallel work to
justify the transfer and launch overheads.

## Testing
Correctness tests for list ranking are useful because wrong answers can look
close. A single bad sublist offset can make many nodes wrong.

The important test cases are:

* a list of one node
* a small list where the expected ranks can be checked by hand
* a list whose node order is different from array order
* large random lists
* lists where the head is not index 0

The last case matters because it is easy to accidentally write code that
works only when the array order matches the list order.

## Conclusion
List ranking is simple when written serially. The parallel version is a
good example of why parallel computing is not only about adding threads.

Wyllie's algorithm is easy to parallelize but does repeated global work.
Helman-JaJa reduces that global work by splitting the list into sublists, but
then the implementation has to choose a good sublist count and combine the
results correctly.

The useful lesson is that the structure of the data matters as much as the
number of processors. A linked list has dependencies built into it, and the
algorithm has to expose enough independent work for OpenMP or CUDA to help.

Please feel free to message me on [LinkedIn]({{ site.linkedin_url }})
with any questions or concerns.
