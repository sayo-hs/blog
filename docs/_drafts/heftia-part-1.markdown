---
title:  "Heftia: The Final Word in Haskell Effect System Libraries - Part 1"
author: riyo
date:   2025-05-13 17:46:45 +0900
categories:
  - heftia
tags:
  - heftia
---

In this series, I will explain `heftia`. This is the first part.

# TL;DR

`heftia` is the first-ever fully type-safe and performant effect system, not just among Haskell libraries but historically across all effect system implementations and languages, to completely implement both *algebraic effects* and *higher-order effects*.

{% linkpreview "https://github.com/sayo-hs/heftia?tab=readme-ov-file#getting-started" %}

`heftia`, a practical next-generation effect library for Haskell, addresses the following major problems found in current effect libraries in a single unified solution:

* **Problems with the `IO` Monad approach**

  Issues inherent to the `IO` monad (`ReaderT IO`) approach employed by libraries such as `effectful`, `cleff`, and `bluefin`:

  * Potential lack of type safety
  * Fundamental inability to support algebraic effects (delimited continuations) due to reliance on `MonadUnliftIO`

* **Semantic Soundness**

  Unsound semantics that occur when combining higher-order effects with algebraic effects (delimited continuations) in effect libraries predating `effectful`, such as `polysemy`, `fused-effects`, and `freer-simple`

* **Interoperability**

  Fragmentation of the Haskell ecosystem and significant migration costs due to the proliferation of incompatible effect libraries

# Overview

`heftia` is a new effect system library for Haskell that I am currently developing. It uniquely provides practical, fully realized implementations of algebraic and higher-order effects with practical performance suitable for real-world use, unmatched by any other existing effect system or language.

As of the current version 0.7, `heftia` is already suitable for practical use.

Over time, numerous Haskell effect libraries have been released, encountered problems, and been replaced by newer solutions. Libraries such as `fused-effects`, `polysemy`, and more recently `cleff`, `effectful`, and `bluefin`, have all emerged.

Due to incompatibility among these libraries, migrating between them has incurred significant costs. Today, the community seeks a definitive solution that ends the cycle of continuous migration.

Recently, the `IO` monad approach (`ReaderT IO`) exemplified by `effectful` has attracted attention as the closest thing to such a definitive solution. It has been praised for improved performance and practical usability compared to previous approaches (`mtl` or `Freer`-based methods), albeit by sacrificing support for **algebraic effects (delimited continuations)**.

But now, there is no longer any need for such compromises. Recent advancements in research on algebraic effects have continued vigorously.

**Leveraging recent solid theoretical foundations[^10], `heftia` simultaneously provides algebraic effect capabilities and high performance, along with ultimate type safety, practicality, and enduring interoperability with other effect libraries.**

[^10]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)<br>
    [A Framework for Higher-Order Effects & Handlers. Birthe van den Berg & Tom Schrijvers, Sci. Comput. Program. 2024.](https://doi.org/10.1016/j.scico.2024.103086)

In this series, I will explain `heftia`. This is the first part.

# Performance

Let's start with performance.
You can find all performance measurements for version 0.7 [here](https://github.com/sayo-hs/heftia/blob/v0.7.0.0/benchmark/performance.md).
Below is a benchmark result of a typical real-world use case involving various effects and IO operations for aggregating file sizes. This benchmark was adapted from one originally used by `effectful`.

<img src="{{ '/assets/images/heftia-part-1/filesize-deep.svg' | relative_url }}" alt="filesize-deep.svg">

As you can see, `heftia` significantly outperforms libraries such as `mtl`, `polysemy`, and `fused-effects`.
Compared to `effectful`, currently the fastest among practical libraries, `heftia` is only slightly slower.
Given that `heftia` is substantially faster than the current de facto standard `mtl` in typical use cases, performance concerns should not hinder adopting `heftia`.

Other benchmarks show that in some cases (such as using `Throw`/`Catch` effects with `Except`), `heftia` even outperforms `effectful`.

While it is relatively slower in cases using the `Local` effect of the `Reader` monad, even then it remains faster than `polysemy` or `fused-effects`. In typical scenarios, you can choose the fastest approach, which closely matches `effectful`. Furthermore, this particular benchmark involves unrealistically heavy usage of `Local`, meaning performance concerns around `Reader` will rarely be an issue in real-world code.

Some may have heard rumors that Freer monad-based libraries are slow[^1].
However, despite internally using a variant of `Freer`, `heftia` achieves this level of performance.

[^1]: Indeed, attempts to implement algebraic effects inherently result in Freer-like structures emerging. This phenomenon likely stems from underlying category-theoretic principles, meaning that implementations of algebraic effects—including languages like `Koka` and evidence-passing implementations—essentially encode Freer structures in various forms.

**The notion that Freer is inherently slow is a misunderstanding**. Performance greatly depends on how the encoding is implemented, and there remains room for further optimization.
Moreover, even `eff`, known for performance by directly interfacing with GHC internals without Freer structures, suffers from quadratic slowdowns relative to program size in certain effects.

Performance discussions are complex[^5], and outcomes can vary significantly with different GHC versions.

[^5]: The causes for Freer's perceived slowness are multifaceted. One significant reason, identified in the paper "Reflection without Remorse," can be resolved by using a data structure called `FTCQueue`, enabling performance comparable to `effectful`.

In any case, I will continue working on improving performance.

Performance always has room for improvement, whereas correctness improvements are often impossible due to foundational interface compatibility. This asymmetry is crucial and will be discussed later.

> Make It Work, Make It Right, Make It Fast. <cite><a href="https://kentbeck.com/">Kent Beck</a></cite>

> 1.3 Make It Fast:
> The final phase revolves around optimizing the performance of the software, making it faster and more efficient. However, the need for speed should not compromise the correctness and maintainability achieved in the previous phases. It is crucial to identify specific areas that require performance improvements and make targeted optimizations, rather than pursuing speed at the expense of code quality. <cite><a href="https://medium.com/@ibk9493/make-it-work-make-it-right-make-it-fast-the-evolution-of-software-development-fbbc1eddd33e">Make It Work, Make It Right, Make It Fast: The Evolution of Software Development</a></cite>

# Issues with the `IO` Monad Approach and Ecosystem

From here, I will describe the issues associated with the current general `IO` monad approach and the broader Haskell effect ecosystem.

## Type Safety

One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
This problem often arises when choosing among multiple programming languages—one language has feature *A* but lacks feature *B*, while another has *B* but lacks *A*...

Supporting this benefit from below is the categorical theory behind these libraries. `mtl` also relies on categorical theory:

> We show that state, reader, writer, and error monad transformers are instances of one general categorical construction: translation of a monad along an adjunction. <cite><a href="https://arxiv.org/abs/2503.20024">Calculating monad transformers with category theory</a></cite>

As a result, the types and functions defined in these libraries correspond to theoretical entities. This provides correctness guarantees, predictability, explainability, the absence of runtime errors, and well-typedness based on underlying theory.

This ensures that even complex, tricky, or innovative combinations of features—never anticipated by library developers—still work correctly. This is composability.

However, libraries based on the `IO` monad approach uniformly move against these valuable advantages of Haskell. Inspecting their source code compared to `lens` or `mtl`, you see frequent use of `IORef`, `error`, various validation checks, runtime errors, and `unsafe` functions. It gives the impression of lower-layer, C-like code rather than typical Haskell code.

This approach is designed explicitly to maximize performance. To be fair, these libraries consistently demonstrate strong benchmark results and are undeniably practical libraries currently available.

However, it’s essential to recognize the compromises involved. This includes not only fewer features but also reduced composability, predictability, explainability, assurances against runtime errors, weakened type safety.

For example, currently, `effectful` and `bluefin` have issues causing runtime errors if [`MonadUnliftIO`](https://hackage.haskell.org/package/unliftio-core-0.2.1.0/docs/Control-Monad-IO-Unlift.html#t:MonadUnliftIO), a typeclass intended for safe exception handling, is improperly used. `effectful` treats this as an accepted behavior rather than a problem.

{% linkpreview "https://github.com/tomjaguarpaw/bluefin/issues/29" %}

Yet, these errors could be prevented at compile time by correctly designed interfaces, as `heftia` does. Exception safety should be guaranteed by types, not runtime checks.

## A Solid Foundation Based on Theory

The methodology behind the `IO` monad approach involves iterative experimentation—implementing solutions, encountering issues, and then applying ad-hoc patches, similar to typical real-world software like web servers or mobile apps.

This leads to a constant cycle of discovering and patching new issues, resembling a game of whack-a-mole. Such a methodology becomes increasingly risky near the core of an ecosystem—what if a newly discovered fundamental flaw requires interface changes severe enough to completely break ecosystem compatibility?

Consider the billion-dollar mistake of `null` and Java’s current situation with `Optional`:

{% include linkpreview.html
    url="https://blogs.oracle.com/javamagazine/post/optional-class-null-pointer-drawbacks"
    title="Nothing is better than the Optional type. Really. Nothing is better."
    description="Optional has numerous problems without countervailing benefits. It does not make your code more correct or robust. There is a real problem that Optional tries to solve, and this article shows a better way to solve it. Therefore, you are better off using a regular and possibly null Java reference rather than Optional."
    image="/assets/images/heftia-part-1/java-optional-class.png"
%}

The history of Haskell’s effect libraries has followed a similar pattern.

However, "the Haskell way" offers a superior approach. By consistently translating mathematical concepts into Haskell code using algebraic data types, encoding categorical constructs through types, equational reasoning, and parametricity, we can guarantee from the outset the absence of various classes of bugs.

I firmly believe maintaining this method is critical for the future of Haskell's effect system. Establishing theoretically sound interfaces helps prevent repeated costly migrations. `heftia` is intended as practical proof of this methodology.

`IO` monad-based libraries like `effectful`, `cleff`, and `bluefin` continually discover and patch issues, as seen in GitHub issues.[^4]

On the other hand, `heftia` directly implements categorical foundations described in academic papers. My work was merely translating types from Agda, checking equivalences, and ensuring types aligned with expected semantics—which they did from the outset. The smoothness of this process strongly confirmed the effectiveness of this methodology.

On the other hand, `heftia` directly implements categorical foundations described in academic papers. What I did was simply translate the types written in Agda from the papers into Haskell, confirm various isomorphisms through equational reasoning. As soon as the functions were given types, all expected semantics were satisfied from the start. I was genuinely surprised while building `heftia` by how smoothly everything worked, and I felt the strength of this methodology firsthand.

The smoothness of this process strongly confirmed the effectiveness of this methodology.

[^4]: Libraries based on the "Weaving" approach, such as `polysemy` and `fused-effects`, experienced similar issues due to their ad-hoc, non-theoretical foundations.

## The Ecosystem Dead-End

Earlier, I mentioned issues related to Java’s `null` problem and type system. Currently, a similar situation is unfolding in Haskell's effect ecosystem.

Combining algebraic effects (delimited continuations) and higher-order effects in a type-safe way likely requires defining first-order and higher-order effects separately. Although not yet proven theoretically, the "Hefty Algebras" paper[^10] strongly suggests this direction. It is likely that future theoretical work over the next few years will provide more precise justifications and formalization of this.

Current interfaces of effect libraries, excluding `heftia`/`data-effects`, do not account for this eventuality. My prediction is that existing libraries and ecosystems will encounter this issue within a few years, forcing users to either continue giving up on algebraic effects or once again endure a costly migration.

If you are interested, keep a close eye on developments in effect research over the coming years! Many theoretical and practical questions remain unanswered in the field of effects.

# Future Challenges and Prospects for `heftia`

First, we need to investigate precisely when the performance of higher-order effects, such as `Local`, becomes noticeably slow in realistic use cases. This means we need benchmarks involving more practical programs using various effects. Concrete performance improvements should ideally follow this investigation.

Additionally, interoperability with other effect libraries is essential.

`heftia` already has basic interoperability established with existing Haskell ecosystem standards such as `mtl` and `unliftio`. `heftia`'s `Eff` monad provides instances for `MonadState`, `MonadRWS`, `MonadError`, and `MonadUnliftIO`. It's also possible to use the `Eff` monad similarly to monad transformers, allowing `heftia` to layer on top of existing monads.

Currently, I am particularly interested in collaboration with `effectful` and `polysemy`. Unfortunately, this cannot be accomplished by me alone; cooperation from the community is necessary. These libraries may require interface adjustments to ensure compatibility. `polysemy` might be simpler since it is relatively close to `heftia`, potentially achievable solely through adjustments on `heftia`'s side. However, `effectful` poses greater difficulty due to significant differences in effect encoding compared to `heftia`.

If anyone is willing to assist, please reach out to me. Thank you.

## Preventing Ecosystem Fragmentation

Earlier, I listed several disadvantages of the `IO` monad approach, but these simply reflect `heftia`'s design philosophy, and I certainly do not think `IO` monad-based libraries shouldn't exist. On the contrary, when performance is critical, there is no reason to exclude any available approach. Ideally, multiple approaches should coexist, allowing users to choose freely. The real problem lies in ecosystem fragmentation. The core issue is the lock-in within each approach, causing a lack of interoperability.

Finally, I have a request for those interested in implementing effects in Haskell:
Please consider building an ecosystem robust to future theoretical developments, encouraging mutual interoperability, such as the [`data-effects`](https://github.com/sayo-hs/data-effects) approach advocated by `heftia`.

`data-effects` provides a generic foundation for defining effects and interpreters, enabling interoperability (ecosystem connections) between the `IO` monad approach, the `heftia` approach, and other methods. Think of it as analogous to the [`Data.Vector.Generic`](https://hackage.haskell.org/package/vector-0.13.2.0/docs/Data-Vector-Generic.html) module from the `vector` library, but for effect systems.

> * Built on the [`data-effects`](https://github.com/sayo-hs/data-effects) effect framework, `heftia` is designed so that it can integrate smoothly with other effect libraries built upon the same framework.
> * Conversion between different libraries' `Eff` monads.
> * `interpret` functions that work independently of any particular library.
> * Currently, only `heftia` is based on this framework.
> * This represents an initial attempt to resolve issues of incompatibility and lack of interoperability caused by the proliferation of effect libraries in Haskell.
> * In addition to monads, an effect system built upon Applicative and Functor is also available.
>
> <cite><a href="https://github.com/sayo-hs/heftia/tree/master?tab=readme-ov-file#key-features">Approach to Inter-Library Compatibility</a></cite>

**Let us stop repeating the cycle of migration hell.**
Let us work together to build the future of the Haskell effect system ecosystem.
