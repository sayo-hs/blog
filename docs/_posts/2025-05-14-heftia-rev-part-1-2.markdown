---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.2"
author: riyo
date:   2025-05-16 10:00:01 +0900
last_modified_at: 2025-05-19 16:17:24 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
---

This article is currently unfinished and will continue to be updated.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-2 %}]({% post_url 2025-05-14-heftia-part-1-2 %})
{: .notice--warning}

[Part 1.1: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[**Part 1.2: The performance of `heftia`**]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[Part 1.3: Discussion on Type Safety in Haskell's Effect Systems]({% post_url 2025-05-14-heftia-rev-part-1-3  %})<br>
[Part 1.4: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})


# Performance

You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
Below is a benchmark result of a typical real-world use case involving various effects and IO operations for aggregating file sizes. This benchmark was adapted from one originally used by `effectful`.

<img src="{{ '/assets/images/heftia-part-1/filesize-deep.svg' | relative_url }}" alt="filesize-deep.svg">

Smaller values indicate faster and better performance.

Performance discussions are complex[^5].

Compared to other approaches, it demonstrates consistently high speed relative to `mtl`, `fused-effects`, and `polysemy`.
On the other hand, similar to those approaches, `heftia` still faces the challenge of decreasing performance as the effect stack becomes deeper when multiple effects are used simultaneously.
In contrast, the `effectful` library, which uses the `ReaderT IO` approach, generally maintains consistent performance regardless of stack depth.

Performance always has room for improvement, whereas correctness improvements are often impossible due to foundational interface compatibility.
This asymmetry is crucial and will be discussed later.

> Make It Work, Make It Right, Make It Fast. <cite><a href="https://kentbeck.com/">Kent Beck</a></cite>

> 1.3 Make It Fast:
> The final phase revolves around optimizing the performance of the software, making it faster and more efficient. However, the need for speed should not compromise the correctness and maintainability achieved in the previous phases. It is crucial to identify specific areas that require performance improvements and make targeted optimizations, rather than pursuing speed at the expense of code quality. <cite><a href="https://medium.com/@ibk9493/make-it-work-make-it-right-make-it-fast-the-evolution-of-software-development-fbbc1eddd33e">Make It Work, Make It Right, Make It Fast: The Evolution of Software Development</a></cite>

---

[To be continued in Part 1.3...]({% post_url 2025-05-14-heftia-rev-part-1-3 %})

[^1]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)

[^2]: Here, “correctly” means that the computation results predicted by the underlying theory and those produced by the implementation must always match.

[^5]: The causes for Freer's perceived slowness are multifaceted. One significant reason, identified in the paper "Reflection without Remorse," can be resolved by using a data structure called `FTCQueue`. In `heftia`, as in `freer-simple`, `FTCQueue` is used to achieve higher performance compared to other Freer implementations.
