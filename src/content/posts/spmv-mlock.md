---
title: SpMV Performance and Pages
pubDate: 2025-12-11
categories: ["Performance", "SpMV"]
description: "Note taking for analyzing the page mechanism impats on SpMV performance"
slug: spmv-lock
draft: false
---

## Before we begin

I have been researched with the Sparse Matrix-Vector Multiplication (SpMV) for years. Apart from the main research goal, there are some subtle experiments which I really interested in, but somehow they quit a re-research or may not craft to be a major contribution yet.

This post will discuss about some of my experiments on the SpMV performance and its data locality (at the page level) which is constructed as followed:

- SpMV locality problem
- How mlock() would help to improve SpMV performance

> **Disclaimer !!!**
> This post is my note-taking and my opinion. It may not provide the correct information and may have the flaw in certain parts. So, please keep in mind that is just one aspect that I observed on the SpMV performance difference.

---

## SpMV Data Locality

In context of SpMV (y=Ax), data locality refer to the use of elements in the dense vector X during the computation. As the matrix layout (how elements are placed in matrix) affect the access pattern on elements in the dense vector, the difference matrix layout would produce different data locality even they have the same amount of non-zero elemetns in the matrix.

The fundamental idea of this experiment is trying the improve the locality of the dense vector X. Ideally, pinning all the values in the last-level of cache (LLC) would improved data locality, reducing the latency.

## mlock()

> mlock() locks pages in the address range starting at _addr_ and continuing for _size_ bytes. All pages that contain a part of the specified address range are guaranteed to be resident in RAM when the call returns successfully; the pages are guaranteed to stay in RAM until later unlocked. -- [Linux man-pages](https://man7.org/linux/man-pages/man2/mlock.2.html)

Okay! `mlock()` does not pin the data in the LCC, Here is my assumption

> **If the input matrix is large enough and cause the swap mechanism, using mlock() may show the benefit performance than baseline (no mock() used )**

Unfortunately ChatGPT told us the may not show the performance differences because paging may not show significant SpMV performance improvement. However, just try to measure it anyway.

## Result

I measured the performance on five matrices whose working set size (WSS) is larger than the Last-Level Cache (LLC) (hoping they would allocate more pages). As expected, the speedup between the baseline and using `mlock()` is slightly different—less than 1%. However, most of them perform slower than the baseline. In my opinion, this case should be considered as having no performance difference, since these are likely caused by tiny interferences or measurement errors, which are very small. I would expect the performance improvement impact to be up to around 10%.

#### Does `mlock()` help?

To show the effects of `mlock()`, I measured the performance counter metrics as shown in the table. With `mlock()`, we expect that the pages of the dense vector X would be found in RAM, which should reduce the number of cache misses. I collected the number of loads and misses to calculate the miss rate (`#miss/#loads`) for LLC, iTLB, and dTLB.

Again, the LLC miss rate does not show huge differences, except for matrix C, which has a lower LLC miss rate when using `mlock()`. We observe that for this matrix, the difference comes from the variation in total LLC loads. The number of LLC misses is almost similar, but the total number of LLC loads differs significantly. As a result, the miss rate difference appears.

The dTLB and iTLB miss results are harder to correlate with the impact of `mlock()` on performance. The TLB handles the translation between virtual and physical pages, and its behavior might not directly reflect the effects of page locking.

| Matrix | Speedup | LLC Miss rate <br>difference | dTLB Miss rate <br>difference | iTLB Miss rate <br>difference |
| ------ | :-----: | :--------------------------: | ----------------------------- | ----------------------------- |
| A      | 1.01874 |           0.10512            | -0.00015                      | -26.97127178                  |
| B      | 0.99649 |           0.86082            | 0.00044                       | 8.129643415                   |
| C      | 0.99913 |         **14.68748**         | 0.00019                       | -8.197961915                  |
| D      | 0.99805 |           -0.70907           | 0.00093                       | 1782.132078                   |
| E      | 0.99810 |           -1.86659           | -0.00045                      | -7553.93233                   |

Difference = baseline - mlock. Minus values would indicates mlock() implemention is better.
[Result and code](https://github.com/songponssw/spmv-mlock)

## Conclusion and Limitation

The impact of `mlock()` is tiny. OS paging is not the primary bottleneck for SpMV. It is difficult to elaborate on the potential factors affecting performance due to the very small performance differences observed. One scenario that might show clearer benefits from page locking is when using a very large matrix with a WSS that causes swapping. In that case, we might see the impact of `mlock()` more clearly.

I know that we may not be able to explain everything happening during computation, as there are many abstractions and optimizations that interplay with performance. This post simply tries to present and interpret some results that may affect SpMV performance in the context of the paging mechanism.

## What's next?

- Memory allocation: `mmap()` vs `malloc()`?
- Streaming the input matrix A?
