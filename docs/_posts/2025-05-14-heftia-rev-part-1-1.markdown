---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.1"
author: riyo
date:   2025-05-19 20:52:28 +0900
last_modified_at: 2025-05-20 03:58:59 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    `heftia` is the first-ever effect system, not just among Haskell libraries but historically across all effect system implementations and languages, to completely implement both *algebraic effects* and *higher-order effects*.
---

This article is the revised version.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-1 %}]({% post_url 2025-05-14-heftia-part-1-1 %})
{: .notice--info}

[**Part 1.1: Summary of Part 1 and an overview of `heftia`**]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[Part 1.2: The performance of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[Part 1.3: Discussion on Type Safety in Haskell's Effect Systems]({% post_url 2025-05-14-heftia-rev-part-1-3  %})<br>
[Part 1.4: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

In this series, I will explain `heftia`. This is the first part.
# Summary

`heftia` is the first-ever effect system, not just among Haskell libraries but historically across all effect system implementations and languages, to completely implement both *algebraic effects* and *higher-order effects*.

{% linkpreview "https://github.com/sayo-hs/heftia?tab=readme-ov-file#getting-started" %}

`heftia` is a Haskell effect library that aims to address major issues found in existing libraries:

* **Compatibility Problems with the `UnliftIO`**

  Inability to support algebraic effects (delimited continuations) due to tight reliance on `MonadUnliftIO`[^3]

* **Semantic Soundness**

  Controversial behaviors that occur when combining higher-order effects with algebraic effects (delimited continuations) in existing effect libraries[^2]

* **Interoperability**

  Fragmentation of the Haskell ecosystem and significant migration costs due to the proliferation of incompatible effect libraries

  Due to incompatibility among these libraries, migrating between them has incurred significant costs. Today, the community seeks a solution that ends the cycle of migration hell.

# Overview

`heftia` is a new effect system library for Haskell that I am currently developing.
It uniquely provides fully realized implementations of algebraic and higher-order effects, unmatched by any other existing effect system or language.

* **Higher-order effects** are effects that take monadic actions as arguments.
    In terms of monad transformers, examples include `local` in `ReaderT` and `catch` in `ExceptT`.
    In contrast, operations like `put`/`get` in `StateT`, `ask` in `ReaderT`, and `throw` in `ExceptT` are classified as **first-order effects**.

    Without higher-order effects, it becomes difficult to use functions like `local` or `catch` flexibly, which can be quite inconvenient.

* **Algebraic effects** are a programming paradigm that has gained attention in recent years.
    They are a language feature and theoretical framework aimed at improving composability and maintainability of programs.

    They have applications in areas such as coroutines and concurrent programming, offering a unified way to express and use such constructs.
    Algebraic effects extend existing control structures, allowing various kinds of control flows—like lightweight threads, asynchronous I/O, or exception handling—to be modularized safely and switched dynamically in a predictable manner.

    Roughly speaking, they overcome the limitations of monad transformers, offering a more convenient, safer, and more predictable alternative.

{% capture alert_content %}
**Definition of Algebraic Effects**

Algebraic effects are sometimes treated as nothing more than a buzzword these days.
In this series, I consistently use the term “algebraic effects” in the sense of the operational semantics and typing rules as defined in the literature by Gordon D. Plotkin and Matija Pretnar:

* [Gordon D. Plotkin and Matija Pretnar, "Handling Algebraic Effects"](https://www.researchgate.net/publication/259151378_Handling_Algebraic_Effects)
* [Matija Pretnar, "An Introduction to Algebraic Effects and Handlers."](https://www.sciencedirect.com/science/article/pii/S1571066115000705)

Furthermore, when I say "implementing algebraic effects (in Haskell)," I mean making the operational semantics and typing rules of algebraic effects fully embeddable in Haskell by using various language features so as to emulate them exactly.

Libraries that implement algebraic effects in Haskell include `freer-simple` and `heftia`.
On the other hand, libraries such as `polysemy`, `effectful`, and `fused-effects` can currently be said to implement a subset of algebraic effects, in the sense that they lack support for **delimited continuations**.
As for `mtl`, its correspondence with algebraic effects is not clear.
{% endcapture %}

<div class="notice--info"> {{ alert_content | markdownify }} </div>

[Here](https://github.com/sayo-hs/heftia?tab=readme-ov-file#comparison) is a comparison table of `heftia` and other effect system implementations in terms of their features:


| Library or Language | Higher-Order Effects | Delimited Conts in Algebraic Effects |
| ------------------- | -------------------- | ---------------------- |
| `heftia`            | ✅                   | ✅                     |
| `mtl`               | ⚠️                   | ⚠️                     |
| `effectful`         | ✅                   | ❌                     |
| `bluefin`           | ✅                   | ❌                     |
| `polysemy`          | ✅                   | ❌                     |
| `fused-effects`     | ✅                   | ❌                     |
| `eff`               | ⚠️                   |  ✅                    |
| `freer-simple`      | ❌                   | ✅                     |
| `in-other-words`    | ✅                   | ⚠️                     |
| `speff`             | ⚠️                   | ✅                     |
| Koka-lang           | ❌                   | ✅                     |
| Eff-lang            | ❌                   | ✅                     |
| OCaml-lang 5        | ❌                   | ✅                     |

✅ = Fully supported / sound<br>
⚠️ = Partially supported or with semantic issues<br>
❌ = Not supported<br>
{: .notice--info}

Recent advancements in research on algebraic effects have continued vigorously.

**Leveraging recent theoretical foundations[^1], `heftia` simultaneously provides capabilities for algebraic effects and higher-order effects, while ensuring ultimate type safety.**

# Code Example

In addition to Hackage, it is also currently available on Stackage Nightly.
Usage is explained on [Haddock](https://hackage-content.haskell.org/package/heftia-0.7.0.0/docs/Control-Monad-Hefty.html).

## Basic Usage
The following is an example of defining, using, and interpreting the first-order effect `Log` for logging and the higher-order effect `Span` for representing named spans in a program.

```hs
import Control.Monad.Hefty 
import Prelude hiding (log, span)

data Log :: Effect where
    Log :: String -> Log f ()
makeEffectF ''Log

data Span :: Effect where
    Span :: String -> f a -> Span f a
makeEffectH ''Span

runLog :: (Emb IO :> es) => Eff (Log : es) ~> Eff es
runLog = interpret \(Log msg) -> liftIO $ putStrLn $ "[LOG] " <> msg

runSpan :: (Emb IO :> es) => Eff (Span : es) ~> Eff es
runSpan = interpret \(Span name m) -> do
    liftIO $ putStrLn $ "[Start span '" <> name <> "']"
    r <- m
    liftIO $ putStrLn $ "[End span '" <> name <> "']"
    pure r

main :: IO ()
main = runEff . runLog . runSpan $ do
    span "example program" do
        log "foo"

        span "greeting" do
            log "hello"
            log "world"

        log "bar"
```

```
> main
[Start span 'example program']
[LOG] foo
[Start span 'greeting']
[LOG] hello
[LOG] world
[End span 'greeting']
[LOG] bar
[End span 'example program']
```

As you can see, the interface is similar to that of `effectful` or `polysemy`, and is very concise.

## Type Inference

Type inference for effects works.
When using `put`/`get` of `State`, there's no need to explicitly specify types like `@String` or `... :: String`.

`hello.hs`:
```hs
import Control.Monad.Hefty 
import Control.Monad.Hefty.State

main :: IO ()
main = runEff $ evalState "" do
    modify (<> "hello ")
    modify (<> "world")
    liftIO . print =<< get
```

```
> main
"hello world"
```

## Delimited Continuations

You can easily define your own handlers using delimited continuations in algebraic effects.
Here is an example of a handler for the non-deterministic computation effect:

```hs
import Control.Monad.Hefty 
import Control.Monad
import Data.List

data NonDet :: Effect where
    Abort :: NonDet f a
    Choice :: [a] -> NonDet f a
makeEffectF ''NonDet

runNonDet :: FOEs es => Eff (NonDet : es) a -> Eff es [a]
runNonDet =
    interpretBy (pure . singleton) \case
        Abort -> \_ -> pure []
        Choice xs -> \resume -> join <$> mapM resume xs

searchCombination :: NonDet :> es => Eff es (Char, Char)
searchCombination = do
    c1 <- choice ['A', 'B', 'C']
    c2 <- choice ['A', 'B', 'C']
    if c1 == c2 then
        abort
    else
        pure (c1,c2)

combination :: [(Char, Char)]
combination = runPure . runNonDet $ searchCombination

-- >>> combination
-- [('A','B'),('A','C'),('B','A'),('B','C'),('C','A'),('C','B')]
```

---

[To be continued in Part 1.2...]({% post_url 2025-05-14-heftia-rev-part-1-2 %})

[^1]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)<br>
    [A Framework for Higher-Order Effects & Handlers. Birthe van den Berg & Tom Schrijvers, Sci. Comput. Program. 2024.](https://doi.org/10.1016/j.scico.2024.103086)

[^2]: [The effect semantics zoo](https://github.com/lexi-lambda/eff/blob/master/notes/semantics-zoo.md)<br>
    [Incorrect semantics for higher-order effects](https://github.com/hasura/eff/issues/12)

[^3]: [The issues with effect systems](https://discourse.haskell.org/t/the-issues-with-effect-systems/5630/19)
