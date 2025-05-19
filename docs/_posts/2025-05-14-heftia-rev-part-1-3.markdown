---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.3"
author: riyo
date:   2025-05-16 20:52:30 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
---

This article is the revised version.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-3 %}]({% post_url 2025-05-14-heftia-part-1-3 %})
{: .notice--info}

This page discusses two contrasting design philosophies, focusing primarily on *type-driven development*, which I am able to explain in more detail. However, please note that the purpose is not to compare them in terms of superiority, but purely to examine the characteristics and trade-offs of each approach.
{: .notice--info}

[Part 1.1: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[Part 1.2: The performance of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[**Part 1.3: Discussion on Type Safety in Haskell's Effect Systems**]({% post_url 2025-05-14-heftia-rev-part-1-3  %})<br>
[Part 1.4: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

This time, it will be a bit longer.

# Composability

One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
This problem often arises when choosing among multiple programming languages—one language has feature *A* but lacks feature *B*, while another has *B* but lacks *A*...

Supporting this benefit from below is the categorical theory behind these libraries. `mtl` also relies on categorical theory:

> We show that state, reader, writer, and error monad transformers are instances of one general categorical construction: translation of a monad along an adjunction. <cite><a href="https://arxiv.org/abs/2503.20024">Calculating monad transformers with category theory</a></cite>

As a result, the types and functions defined in these libraries correspond to theoretical entities. This provides correctness guarantees, predictability, explainability, the absence of runtime errors, and well-typedness based on underlying theory.

This ensures that even complex, tricky, or innovative combinations of features—never anticipated by library developers—still work correctly[^2]. This is composability.

`heftia` directly implements the categorical foundations described in the papers, called Hefty Algebras and generic free monads.
What we did was translate the types written in Agda from the papers into Haskell and confirm isomorphisms via equational reasoning.

# Comparison of Approaches

Here I compare the characteristics of each approach to effect systems in Haskell. I only consider libraries that can realize higher-order effects.

## Monad Transformer Approach

The monad transformer (`mtl`) approach is currently the de facto standard in Haskell.

* It leverages type classes and, due to its categorical background, composes well with any other library, making it excellent in terms of interoperability.

* In terms of performance, it is fast compared to other approaches when the transformer stack is shallow, but as the stack grows deeper, it suffers from slower performance compared to the `ReaderT IO` approach.

* In terms of semantics, its behavior can change depending on the order in which `ExceptT` and `StateT` are stacked, which poses predictability challenges. To the best of my knowledge, this issue is not covered by the underlying categorical theory. In other words, the interaction and operational semantics when composing multiple transformers remains nontrivial.

* From the perspective of algebraic effects, it is unclear how its operational semantics and type system relate to `mtl`.

* In practice, there is the n^2 instances problem:

    > For these common transformers, these instances have all been written. However, every time you add an additional transformer - you need to write all these n instances where n is the number of interfaces you intend to use. Thus the n^2 complexity. <cite><a href="https://felixmulder.com/writing/2020/08/08/Revisiting-application-structure.html#the-n2-issue">Revisiting application structure</a></cite>

## Weaving Approach

This approach is adopted by `fused-effects` and `polysemy`.

* Its characteristics are similar to those of `mtl`. It inherits `mtl`’s performance degradation when deeply stacked and its nontrivial semantics when composing multiple effects.

* The weaving operation does not correspond to categorical theory and is understood to be ad-hoc[^4]. In particular, the complexity of behavior when using non-deterministic effects[^3] is a characteristic that did not exist in `mtl`, highlighting **the importance of correspondence with categorical theory from a semantic standpoint**.

* From the perspective of algebraic effects, there is also the limitation that users cannot treat delimited continuations as first-class values, in addition to subtle semantic mismatches.

Despite these challenges, this approach was the first to attempt to combine algebraic effects with higher-order effects, and as such provided a meaningful foundation for subsequent discussion.
In particular, **the generic free monad used internally by `polysemy` was later analyzed categorically and found to provide a unified framework for the higher-order version of algebraic effects**[^6][^8].
Although the encoding has been changed for performance reasons, `heftia` also uses the generic free monad.

## `ReaderT IO` (Evidence Passing) Approach

This approach is adopted by libraries such as `effectful`, `bluefin`, `cleff`, `bluefin-algae`, and `speff`[^7].

It has several distinctive features compared to other approaches.

* First, it tends to offer the highest performance and is less prone to slowdown even with a deep stack.

* In terms of semantics, when implemented correctly, it exhibits relatively good behavior. In particular, unlike `mtl` or the weaving approach, it does not show changes in semantics based on stack order, which is a benefit in terms of predictability.

* On the other hand, the cost of implementing it without runtime errors or other bugs is relatively high. This is because, whereas the types guide the implementation process in other approaches, the `ReaderT IO` approach offers fewer hints from the type system, meaning that the library, under its own responsibility, must carefully restrict the public interface it exposes.

  This topic will be discussed in more detail in the section on the "two contrasting development processes".

* From the perspective of algebraic effects, there are both libraries that do not support delimited continuations (`effectful`, `bluefin`, `cleff`, etc.) and those that do (`bluefin-algae`, `speff`, etc.), where the former are practical, but the latter are still experimental.

## Elaboration Approach

This approach is adopted for the first time by `heftia`.

* It is the only approach that allows safe use of both higher-order effects and the delimited continuation feature of algebraic effects. Here, type safety includes not only protection against runtime errors but also full semantic correctness, meaning adherence to the operational semantics of algebraic effects.

* Its semantics fully match those of algebraic effects in the first-order effect domain. When combined with higher-order effects, it does not suffer from behavior changes based on stack order, demonstrating good predictability.

* Since delimited continuations can be treated as first-class values, its syntax is closest to that of algebraic effects. In other words, variables representing a delimited continuation `k` can appear in code, be invoked freely, passed around, or stored as state.

* Similar to `mtl` and the weaving approach, it tends to slow down as the stack grows deeper.

---

Ultimately, each user needs to decide which approach to adopt based on the trade-off between performance, type safety, and functionality.

# Theory-Backed Interface

Here, as an illustrative example, I’d like to consider the billion-dollar mistake of `null` and Java’s current situation with `Optional`:

{% include linkpreview.html
    url="https://blogs.oracle.com/javamagazine/post/optional-class-null-pointer-drawbacks"
    title="Nothing is better than the Optional type. Really. Nothing is better."
    description="Optional has numerous problems without countervailing benefits. It does not make your code more correct or robust. There is a real problem that Optional tries to solve, and this article shows a better way to solve it. Therefore, you are better off using a regular and possibly null Java reference rather than Optional."
    image="/assets/images/heftia-part-1/java-optional-class.png"
%}

What happened here is that a type system with holes, which served as the foundational interface, was adopted.
Later, when a problem was discovered and an attempt was made to fix the interface, the ecosystem had already become deeply dependent on the previous interface, making it stuck and unable to address the root cause of the issue.

To avoid this situation, in designing a programming language’s type system, one is generally forced to first design a perfect system (compared to ordinary software products) based on type system theory.
This is because it is difficult to gradually refine it based on feedback from the language’s users.

Effect systems are in the same predicament.
The proliferation and lack of interoperability of Haskell’s effect systems to date can be seen as interface incompatibilities born of trial and error, due to the absence of a solid, guiding theory for effect systems that combine algebraic and higher-order effects.

## Two Contrasting Development Processes

Earlier, in the `ReaderT IO` approach section, I used the phrase “types guide implementation.”
What that means is, for example, in `mtl`, the implementation of `runStateT` for `StateT` is almost self-evident, and you can implement it by filling in slots to eliminate type errors like solving a puzzle.
The development process for `heftia` is similar: essentially, after translating the types from the paper into Haskell, it was built by endlessly filling in type puzzles.

In other words, there is a style of development where, if you first design data types correctly, the implementation follows automatically.
Let’s call this type-driven development.

The advantage of type-driven development is that it can minimize the possibility of runtime errors.
In this development process, causing a runtime error—that is, making a function partial—is not permitted, so you can guarantee that no runtime errors exist overall.
If a runtime error does occur, it is clearly due to something outside the library.
In other words, the interface becomes safe with respect to runtime errors automatically.

On the other hand, in the `ReaderT IO` approach, one isn’t all that hesitant to use partial functions.
Instead, one consider which cases would lead to runtime errors, design the public interface in a suitably constrained way to prevent those cases from occurring, and isolate unsafe functions in non-public modules.

I experienced this process myself when attempting to implement the `ReaderT IO` version of `heftia`.
Functions with signatures that cannot be implemented in the current type-safe version of `heftia` can be implemented in the `ReaderT IO` version.
When you run them, runtime errors occur.
To prevent that, you need to think through when runtime errors will occur and restrict the interface accordingly.

Here, if there is an obscure "hole" in those restrictions, a problem arises where runtime errors occur only when certain functions or libraries are combined in a specific pattern, even though they never occur during normal use.
In other words, the composability mentioned at the outset is compromised.
The tricky part of this problem is that the hole has a latent period: it isn’t discovered until a sufficient number of users have used the system.

In the past, this has manifested in some `ReaderT IO` libraries when publicly exposed functions are combined in specific ways, leading to segmentation faults or even enabling the implementation of `unsafeCoerce`. For example:

1. [`unsafeCoerce` derivable from `Coroutine`+`locally`+`abort`](https://github.com/hasura/eff/issues/13)
2. [StateSource can be used to implement unsafeCoerce](https://github.com/tomjaguarpaw/bluefin/issues/49)
3. [phantom type role on State breaks soundness](https://github.com/tomjaguarpaw/bluefin/issues/50)

In the first case, the cause seems to lie in the design of type safety related to delimited continuations.
Attempts to implement delimited continuation features safely using `IO` are therefore considerably more challenging in design than libraries built on `ReaderT IO` that do not support them.

On the other hand, items 2 and 3 are unrelated to difficulties caused by delimited continuations and stem simply from oversights.
They are much more trivial than the first issue and do not have an impact on the ecosystem as a whole.

---

Of course, type-driven development is not immune to such latent-period issues either.
However, these appear not as type-safety violations but as unpredictable performance.
That is, something that seems efficient at first glance can experience drastic performance degradation under certain usage patterns.
This applies, for instance, to space leaks in `polysemy`: [Possible Space Leak](https://github.com/polysemy-research/polysemy/issues/340)

That said, performance issues have the advantage that even if discovered later,
there is room for improvement without changing the interface[^9],
so the risk that the entire ecosystem will be affected by the fix is low.

In contrast, holes in the interface, like Java's null, carry the risk that once they become widespread and entrenched in the ecosystem, fixing them later becomes difficult.
In practice, the risk that such interface holes will necessitate large-scale refactoring of the effects ecosystem may not be that high.
However, I believe it is valuable to be aware of that risk.
Of course, this issue is not limited to the `ReaderT IO` approach.
The moment you depend on `IO` to implement, for example, concurrency interpreters, you must start confronting the problem of designing a safe interface.

However, there is a difference in locality: in the case of `ReaderT IO`, the source of the hole can occur in the core of the library, whereas elsewhere it may be confined to only parts such as the concurrency interpreters.
It is important to keep the unsafe parts where holes may occur as small and verifiable as possible.

In this sense, avoiding interface holes can be achieved by the following strategy:
* By consistently translating mathematical concepts into Haskell code using algebraic data types, encoding categorical constructs through types, equational reasoning, and parametricity,
    we can guarantee from the outset the absence of various classes of bugs and holes in the interface.

    In particular, this approach is exceptionally robust, ensuring that both the interface and its implementation behave precisely as prescribed by the underlying category-theoretic framework, with clear explanations and full accountability.

    I firmly believe maintaining this method delivers substantial value for the future of Haskell's effect system. Establishing theoretically sound interfaces helps prevent repeated costly migrations.
`heftia` and `data-effects` are intended as practical proof of this methodology.

Another countermeasure is also possible:
* Instead of guaranteeing such perfect rigor, one could still include unsafe parts at the core of the library,
    while keeping the core of safe primitive functions as small as possible to make it easier to verify that there are no holes.
    After verification, once it is fully confirmed that there are no holes, one can continue to provide safe primitives for the interface into the future.

    This is adopted in `data-effects` for the open union part, which cannot be implemented safely in principle due to Haskell’s lack of extensible sum types, and in the `bluefin` effect library.

    In particular, by designing the primitives' interfaces based on theory, one can ensure a reliable correspondence between theory and interface.
    For example, in the design of `bluefin`, so that [Lazy functional state threads](https://www.microsoft.com/en-us/research/publication/lazy-functional-state-threads/) provides the theoretical foundation. 

---

From here on, these are purely my personal speculations.

One personal observation I made during the implementation of `heftia` is that
combining algebraic effects (delimited continuations) and higher-order effects in a type-safe way likely requires defining first-order and higher-order effects separately.
Although not yet proven theoretically, the "Hefty Algebras" paper[^1] strongly suggests this direction.

Currently, the ecosystems of other effect systems have been built without taking this possibility into account.
As a result, when we eventually move to a system that supports both algebraic effects and higher-order effects, it may require a major overhaul,
namely separating first-order and higher-order definitions for every effect.

That said, it’s possible that future research will even eliminate the need for such a separation, so I can’t say anything for certain.
The one thing we can say with confidence is that effect systems remain an evolving field, and nobody yet knows which approach will ultimately prove “correct.”

However, based on past examples in the `mtl` and Weaving approaches, I believe that designing interfaces by treating algebraic effects and generic free monads—grounded in category theory—as “reference theories” will undoubtedly bolster the future robustness of effect systems.

---

[To be continued in Part 1.4...]({% post_url 2025-05-14-heftia-rev-part-1-4 %})

[^1]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)

[^2]: Here, “correctly” means that the computation results predicted by the underlying theory and those produced by the implementation must always match.

[^3]: [Choice of functorial state for runNonDet](https://github.com/polysemy-research/polysemy/issues/246)

[^4]: > Their work also considers the problem of modular composition of handlers, and presents a solution–called “weaving”–based on threading a handler state through unknown operations. This approach is rather ad-hoc; it is not a generalization of the forwarding approach for algebraic effects. <cite><a href="https://arxiv.org/abs/2304.09697">A Calculus for Scoped Effects & Handlers</a></cite>

[^6]: > In this work we propose a generic framework for higher-order effects, generalizing algebraic effects & handlers: a generic free monad with higher-order effect signatures and a corresponding interpreter. <cite><a href="https://dl.acm.org/doi/10.1016/j.scico.2024.103086">A framework for higher-order effects & handlers</a></cite>

[^7]: Strictly speaking, `bluefin` may not be accurately described as using `ReaderT IO` (it simply follows the `IO` style). Still, it adopts the evidence-passing approach. That is, there's a subtle difference in meaning between `ReaderT IO` and evidence passing.

[^8]: > In what follows we present the categorical foundations of our generic free monad. <cite><a href="https://dl.acm.org/doi/10.1016/j.scico.2024.103086">A framework for higher-order effects & handlers</a></cite>

[^9]: However, given that the space-leak issue in `polysemy` has remained open for about four years, this is purely a theoretical matter.
