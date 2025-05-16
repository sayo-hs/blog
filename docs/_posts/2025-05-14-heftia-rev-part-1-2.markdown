---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.2 - Take 2"
author: riyo
date:   2025-05-14 18:00:00 +0900
last_modified_at: 2025-05-16 04:59:48 +0900
categories:
  - heftia
tags:
  - heftia
excerpt: |
    Let's start with performance.
    You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
    Below is a benchmark result of a typical real-world use case involving various effects and IO operations for aggregating file sizes. This benchmark was adapted from one originally used by `effectful`.
---

[**Part 1.1**: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[**Part 1.2**: The performance and type safety of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[**Part 1.3**: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

# Performance

You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
Below is a benchmark result of a typical real-world use case involving various effects and IO operations for aggregating file sizes. This benchmark was adapted from one originally used by `effectful`.

<img src="{{ '/assets/images/heftia-part-1/filesize-deep.svg' | relative_url }}" alt="filesize-deep.svg">

Smaller values indicate faster and better performance.

Performance discussions are complex[^5].

[^5]: The causes for Freer's perceived slowness are multifaceted. One significant reason, identified in the paper "Reflection without Remorse," can be resolved by using a data structure called `FTCQueue`.

I will continue working on improving performance.

Performance always has room for improvement, whereas correctness improvements are often impossible due to foundational interface compatibility.

> Make It Work, Make It Right, Make It Fast. <cite><a href="https://kentbeck.com/">Kent Beck</a></cite>

> 1.3 Make It Fast:
> The final phase revolves around optimizing the performance of the software, making it faster and more efficient. However, the need for speed should not compromise the correctness and maintainability achieved in the previous phases. It is crucial to identify specific areas that require performance improvements and make targeted optimizations, rather than pursuing speed at the expense of code quality. <cite><a href="https://medium.com/@ibk9493/make-it-work-make-it-right-make-it-fast-the-evolution-of-software-development-fbbc1eddd33e">Make It Work, Make It Right, Make It Fast: The Evolution of Software Development</a></cite>

## Type Safety

One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
This problem often arises when choosing among multiple programming languages—one language has feature *A* but lacks feature *B*, while another has *B* but lacks *A*...

Supporting this benefit from below is the categorical theory behind these libraries. `mtl` also relies on categorical theory:

> We show that state, reader, writer, and error monad transformers are instances of one general categorical construction: translation of a monad along an adjunction. <cite><a href="https://arxiv.org/abs/2503.20024">Calculating monad transformers with category theory</a></cite>

As a result, the types and functions defined in these libraries correspond to theoretical entities. This provides correctness guarantees, predictability, explainability, the absence of runtime errors, and well-typedness based on underlying theory.

This ensures that even complex, tricky, or innovative combinations of features—never anticipated by library developers—still work correctly. This is composability.

By consistently translating mathematical concepts into Haskell code using algebraic data types, encoding categorical constructs through types, equational reasoning, and parametricity, we can guarantee from the outset the absence of various classes of bugs.

`heftia` is intended as practical proof of this methodology.

`heftia` directly implements categorical foundations described in academic papers. What I did was simply translate the types written in Agda from the papers into Haskell, confirm various isomorphisms through equational reasoning.

Combining algebraic effects (delimited continuations) and higher-order effects in a type-safe way likely requires defining first-order and higher-order effects separately.
Although not yet proven theoretically, the "Hefty Algebras" paper[^10] strongly suggests this direction.

---

[To be continued in Part 1.3...]({% post_url 2025-05-14-heftia-rev-part-1-4 %})

[^10]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)
