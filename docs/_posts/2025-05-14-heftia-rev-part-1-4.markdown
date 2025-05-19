---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.4"
author: riyo
date:   2025-05-19 20:52:31 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    At present, documentation for functions and types in Haddock is lacking, so I plan to fill in those gaps.
    In addition, I intend to provide usage explanations in Part 2 or beyond of this series.
---

This article is the revised version.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-4 %}]({% post_url 2025-05-14-heftia-part-1-4 %})
{: .notice--info}

[Part 1.1: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[Part 1.2: The performance of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[Part 1.3: Discussion on Type Safety in Haskell's Effect Systems]({% post_url 2025-05-14-heftia-rev-part-1-3  %})<br>
[**Part 1.4: Future prospects of `heftia`**]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

# Future Challenges for `heftia`

## Enhancing the Documentation

At present, documentation for functions and types in Haddock is lacking, so I plan to fill in those gaps.
In addition, I intend to provide usage explanations in Part 2 or beyond of this series.

I have not yet decided what exactly to include in the later parts.
If there is anything you would like to see explained, whether about usage or anything else, please feel free to send in a request!
Having requests helps me understand what everyone is expecting, which is greatly appreciated.

## Performance Improvements

We need to investigate precisely when the performance of higher-order effects, such as `Local`, becomes noticeably slow in realistic use cases. This means we need benchmarks involving more practical programs using various effects. Concrete performance improvements should ideally follow this investigation.

## Real-World Use of `heftia`

We also need some practical software that uses `heftia`. This is essential to demonstrate its real-world usability. I plan to start using `heftia` in other software projects going forward. This will help clarify which additional effects need to be supported and what interface improvements may be necessary.

## Expanding the Ecosystem

In terms of ecosystem expansion, several effects are on my radar that I’d particularly like to add—such as type-safe file system operations, shell scripting, networking, and other web-related functionality. As a prototype, I’ve already implemented a type-safe effect for using subprocesses: [Control.Monad.Hefty.Concurrent.Subprocess](https://hackage-content.haskell.org/package/heftia-effects-0.7.0.0/docs/Control-Monad-Hefty-Concurrent-Subprocess.html).

## Relaxing Restrictions When Combining Continuations and Higher-Order Effects

Currently, interpreters that use delimited continuations face restrictions in the order of interpretation when combined with higher-order effects.
Specifically, for example, it’s not possible to run `runState` before `runCatch`.
Instead, `runCatch` must be run first, followed by `runState`.
This constraint is necessary to maintain type safety and ensure sound semantics when delimited continuations interact with higher-order effects.

However, we are currently working on partially relaxing this restriction.
We already know that it is feasible; the remaining task is to adjust the interface and implement it.

{% linkpreview "https://github.com/sayo-hs/heftia/issues/26" %}

## Expanding Interoperability

Additionally, interoperability with other effect libraries is essential.

`heftia` already has basic interoperability established with existing Haskell ecosystem standards such as `mtl` and `unliftio`. `heftia`'s `Eff` monad provides instances for `MonadState`, `MonadRWS`, `MonadError`, `MonadUnliftIO`, and others. It's also possible to use the `Eff` monad similarly to monad transformers, allowing `heftia` to layer on top of existing monads.

Currently, I am particularly interested in collaboration with `effectful` and `polysemy`.
This cannot be accomplished by me alone; collaboration from the community is necessary.
These libraries may require interface adjustments to ensure compatibility.
`polysemy` might be simpler since it is relatively close to `heftia`, potentially achievable solely through adjustments on `heftia`'s side.
However, `effectful` poses greater difficulty due to significant differences in effect encoding compared to `heftia`.

If anyone is willing to assist, please reach out to me. Thank you.

# Preventing Ecosystem Fragmentation

Ideally, multiple approaches should coexist, allowing users to choose freely. **The real problem lies in ecosystem fragmentation.** The core issue is the lock-in within each approach, causing a lack of interoperability.

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

I’m grateful to all the Haskell library authors whose ongoing work has paved the way for `heftia`.
Without their contributions, this achievement would not exist.

---

To be continued in Part 2.1...
