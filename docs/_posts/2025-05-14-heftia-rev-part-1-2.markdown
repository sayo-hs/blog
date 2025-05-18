---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.2"
author: riyo
date:   2025-05-16 10:00:00 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
---

This article is currently unfinished and will continue to be updated.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-1 %}]({% post_url 2025-05-14-heftia-part-1-1 %})
{: .notice--warning}

[**Part 1.1**: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[**Part 1.2**: The performance and type safety of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[**Part 1.3**: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

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

# Type Safety

One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
This problem often arises when choosing among multiple programming languages—one language has feature *A* but lacks feature *B*, while another has *B* but lacks *A*...

Supporting this benefit from below is the categorical theory behind these libraries. `mtl` also relies on categorical theory:

> We show that state, reader, writer, and error monad transformers are instances of one general categorical construction: translation of a monad along an adjunction. <cite><a href="https://arxiv.org/abs/2503.20024">Calculating monad transformers with category theory</a></cite>

As a result, the types and functions defined in these libraries correspond to theoretical entities. This provides correctness guarantees, predictability, explainability, the absence of runtime errors, and well-typedness based on underlying theory.

This ensures that even complex, tricky, or innovative combinations of features—never anticipated by library developers—still work correctly[^2]. This is composability.

By consistently translating mathematical concepts into Haskell code using algebraic data types, encoding categorical constructs through types, equational reasoning, and parametricity, we can guarantee from the outset the absence of various classes of bugs.

`heftia` is intended as practical proof of this methodology.

`heftia` directly implements the categorical foundations described in the papers, called Hefty Algebras and Higher-order Freer monads.
What we did was translate the types written in Agda from the papers into Haskell and confirm isomorphisms via equational reasoning.

# Theory-Backed Interface

Consider the billion-dollar mistake of `null` and Java’s current situation with `Optional`:

{% include linkpreview.html
    url="https://blogs.oracle.com/javamagazine/post/optional-class-null-pointer-drawbacks"
    title="Nothing is better than the Optional type. Really. Nothing is better."
    description="Optional has numerous problems without countervailing benefits. It does not make your code more correct or robust. There is a real problem that Optional tries to solve, and this article shows a better way to solve it. Therefore, you are better off using a regular and possibly null Java reference rather than Optional."
    image="/assets/images/heftia-part-1/java-optional-class.png"
%}

What happened here is that a type system with *holes*, meaning the foundational interface, was adopted, and when a problem was later discovered and an attempt was made to fix the interface,
the ecosystem was already deeply dependent on the previous interface, leaving it stuck and unable to address the root cause of the issue.

To avoid this situation, in designing a programming language’s type system, one is generally forced to first design a perfect system (compared to ordinary software products) based on type system theory.
This is because it is difficult to gradually refine it based on feedback from the language’s users.

Effect systems are in the same predicament.
The proliferation of Haskell’s effect systems to date and their lack of mutual compatibility are a perfect illustration of this.
To prevent this, I believe that theoretical support is essential when designing an effect system's interface.

Here, "the Haskell way" offers a superior approach.
By consistently translating mathematical concepts into Haskell code using algebraic data types, encoding categorical constructs through types, equational reasoning, and parametricity, we can guarantee from the outset the absence of various classes of bugs.

I firmly believe maintaining this method is critical for the future of Haskell's effect system. Establishing theoretically sound interfaces helps prevent repeated costly migrations.
`heftia` and `data-effects` are intended as practical proof of this methodology.

From here on, these are purely my personal speculations.

One personal observation I made during the implementation of `heftia` is that
combining algebraic effects (delimited continuations) and higher-order effects in a type-safe way likely requires defining first-order and higher-order effects separately.
Although not yet proven theoretically, the "Hefty Algebras" paper[^1] strongly suggests this direction.

Currently, the ecosystems of other effect systems have been built without taking this possibility into account.
As a result, when we eventually move to a system that supports both algebraic effects and higher-order effects, it may require a major overhaul,
namely separating first-order and higher-order concerns for every effect.

That said, it’s possible that future research will even eliminate the need for such a separation, so I can’t say anything for certain.
The one thing we can say with confidence is that effect systems remain an evolving field, and nobody yet knows which approach will ultimately prove “correct.”


---

[To be continued in Part 1.3...]({% post_url 2025-05-14-heftia-rev-part-1-4 %})

[^1]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)

[^2]: Here, “correctly” means that the computation results predicted by the underlying theory and those produced by the implementation must always match.

[^5]: The causes for Freer's perceived slowness are multifaceted. One significant reason, identified in the paper "Reflection without Remorse," can be resolved by using a data structure called `FTCQueue`. In `heftia`, as in `freer-simple`, `FTCQueue` is used to achieve higher performance compared to other Freer implementations.
