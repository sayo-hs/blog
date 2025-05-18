---
title:  "Heftia: The Next Generation of Haskell Effects Management - Part 1.3"
author: riyo
date:   2025-05-16 10:00:00 +0900
categories:
  - heftia
tags:
  - heftia
  - live
excerpt: |
    One notable benefit of Haskell is the ability to mutually extend language features compatibly through libraries. This means you never encounter issues such as being unable to use `lens` because you chose `mtl`.
---

This article is currently unfinished and will continue to be updated.
The archive of the pre-revision version is available at: [{% post_url 2025-05-14-heftia-part-1-3 %}]({% post_url 2025-05-14-heftia-part-1-3 %})
{: .notice--warning}

[Part 1.1: Summary of Part 1 and an overview of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-1 %})<br>
[Part 1.2: The performance of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-2  %})<br>
[**Part 1.3: Discussion on Type Safety in Haskell's Effect Systems**]({% post_url 2025-05-14-heftia-rev-part-1-3  %})<br>
[Part 1.4: Future prospects of `heftia`]({% post_url 2025-05-14-heftia-rev-part-1-4  %})

今回は少し長くなります。

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

ここでは、Haskellにおけるエフェクトシステムのアプローチごとの特性の比較を行います。
また、ここでは高階エフェクトを実現できるライブラリのみを対象にしています。

## モナドトランスフォーマー方式

モナドトランスフォーマー (`mtl`) は、現在Haskellにおいてデファクトスタンダードの方式です。

- 型クラスを利用する点と、圏論的背景に起因する合成可能性から、他のどのライブラリとも馴染み、相互互換性の観点では非常に優れています。

- パフォーマンスの観点では、トランスフォーマーのスタックが浅いときは他の方式と比較しても高速であるものの、積み上げるにつれて`ReaderT IO`方式と比較した際の低速さが課題とされています。

- セマンティクスの観点では、`ExceptT`と`StateT`をスタックする順番により挙動が変わるなど、挙動の予測可能性の面で課題があるとされます。
    この部分は、私の知る限り、背後の圏論的理論においてカバーされていないようです。
    すなわち、複数のトランスフォーマを合成した場合の相互作用のしかた・操作的意味論に非自明さが残っています。

- 代数的エフェクトの観点では、その操作的意味論及び型システムと`mtl`がどのように結びついているかは不明です。

## Weaving方式

この方式は`fused-effects`及び`polysemy`で採用されています。

- この方式は特性としては`mtl`に近いです。深くスタックした際の低速化、複数のエフェクトを合成した際の意味論の非自明さなどは、`mtl`のものを引き継いでいます。

- また、Weaving操作は圏論的理論との対応関係を持たず、アドホックなものと理解されています[^4]。
    特に、非決定性エフェクト使用の際の挙動の難解さ[^3]は`mtl`には存在しなかった特性であり、セマンティクスの観点において、圏論的理論との対応の重要さが際立っています。

    これらの課題があるものの、代数的エフェクトと高階エフェクトの両立を最初に試みた点において、その後の議論の基礎を提供する有意義なものでした。
    特に、**`polysemy`の内部で使用されている generic free monad 自体は、その後圏論的に分析され、代数的エフェクトの高階エフェクト版の統一的フレームワークを提供するものと分かりました[^6][^8]**。
    パフォーマンス上の理由からエンコーディングを変更していますが、`heftia`でもgeneric free monadを使用しています。

- 代数的エフェクトの観点では、意味論の微妙な不一致の他、限定継続をファーストクラスな値としてユーザが使用することができないという制限があります。

## `ReaderT IO` （エビデンスパッシング） 方式

この方式は`effectful`, `bluefin`, `cleff`, `speff`などのライブラリで採用されています[^7]。

これは他の方式と比較すると特徴的な点がいくつかあります。

- まず、方式の中では最も速度が出やすく、スタックを深くしても低速化しづらいという特徴があります。

- セマンティクスに関しては、正しく実装した場合、比較的良好な振る舞いを示します。
    特に、`mtl`やweaving方式のような、スタック順によりセマンティクスが変化する点が見られず、挙動が直感に近いというメリットがあります。

- 一方で、ランタイムエラー等の不具合なしに実装することのコストが比較的高いです。
    これは、他の方式の実装プロセスにおいては**型が実装を誘導する**のに対し、`ReaderT IO`方式の実装ではそのような型によるヒントが少ないため、
    ライブラリの責任において、公開するインターフェースをうまく制限する必要があるためです。

    これについて、「2つの対照的な開発プロセス」の節で後ほど少し詳しく説明します。

- 代数的エフェクトの観点では、限定継続をサポートしないライブラリ (`effectful`, `bluefin`, `cleff`等) とするライブラリ(`speff`等)が両方あり、前者は実用的ですが、後者はいずれも実験的な状態です。

## Elaboration方式

この方式は、`heftia`が今回初めて採用した方式です。

- 方式の中で唯一、高階エフェクトと、代数的エフェクトの限定継続機能を同時に、かつ型安全に使用することができます。
    ここで型安全というのは、ランタイムエラー対策は勿論、意味論の完全な正常性、すなわち代数的エフェクトの操作的意味論に従うことを含んでいます。

- セマンティクスは、一階エフェクトの範疇では代数的エフェクトの操作的意味論に完全に一致します。
    高階エフェクトと組み合わせた場合、スタックする順番による挙動の変化の問題は発生せず、良好な予測可能性を示します。

- 限定継続をファーストクラスな値として扱うことができるため、その構文論は代数的エフェクトのそれに最も近いです。
    すなわち、コード中で限定継続`k`ののようにして変数が出現し、自由に呼び出したり、持ち回したり、状態として保存したりできます。

- `mtl`, weaving方式と同様、スタックが深くなるにつれて低速化する傾向にあります。

---

最終的には、ユーザ各自が、パフォーマンス・型安全性・多機能性のトレードオフに応じて、どれを採用するかを決定する必要があります。

# Theory-Backed Interface

Consider the billion-dollar mistake of `null` and Java’s current situation with `Optional`:

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

## 2つの対照的な開発プロセス

先程、アプローチ間の比較の`ReaderT IO`のところで、型が実装を誘導するというフレーズを使いました。
これがどういうことかというと、例えば`mtl`においては、`StateT`に対する`runStateT`の実装はほとんど自明で、型エラーをなくす穴埋めパズルを解くようにして実装することができます。
`heftia`の開発プロセスも同様で、基本的には、型を論文からHaskellに翻訳して持ってきた後、あとはひたすら型パズルを埋める作業のみで作られました。

つまり、まず最初にデータ型を正しく設計すると、実装は後から自動で付いてくるという、このような開発のスタイルがあります。
これを型誘導開発とここでは呼びましょう。

型誘導開発のメリットは、ランタイムエラーの可能性を極限まで排除できることです。
この開発プロセスでは、ランタイムエラーを起こす、すなわち関数を部分関数にすることを許容しないため、全体としてランタイムエラーが存在しないことを保証できます。
仮にランタイムエラーが発生した場合、それはライブラリの外部に原因であることがはっきりとします。
これは言い換えれば、インターフェースが自動でランタイムエラーに関して安全になるということでもあります。

一方で、`ReaderT IO`方式の開発プロセスでは、部分関数の使用をあまり躊躇しません。
その代わりに、どのようなケースがランタイムエラーになるかを考え、そのようなケースが起こらないように公開するインターフェースをうまく制限した形で設計し、非安全な関数は非公開モジュールの中に隔離します。

このような開発プロセスは、私自身、`heftia`の`ReaderT IO`バージョンの実装を試みていた際に体験しました。
`heftia`の現在の型安全なバージョンでは実装することができないようなシグネチャの関数が、`ReaderT IO`の方では実装できてしまいます。
そしてそれを実行すると、ランタイムエラーが発生します。それを防ぐには、頭を使って、どのような場合にランタイムエラーが発生するかを見極め、インターフェースをうまく制限する必要があります。

このとき、制限にわかりずらい「穴」があると、通常の使用法では発生しないが、複数の関数やライブラリを特定のパターンで組み合わせた時だけランタイムエラーになるという問題が発生します。
つまり、最初に述べたcomposabilityが損なわれています。
この問題の厄介な点は、十分な数のユーザに使用されるまでその穴が見つからないという、「潜伏期間」があるということです。

これは過去、各種`ReaderT IO`ライブラリにおいて、公開されている関数を特定の方法で組み合わせると、segmentation faultが発生したり、`unsafeCoerce`関数が実装できてしまうという形で現れてきました:

- [`unsafeCoerce` derivable from `Coroutine`+`locally`+`abort`](https://github.com/hasura/eff/issues/13)
- [StateSource can be used to implement unsafeCoerce](https://github.com/tomjaguarpaw/bluefin/issues/49)
- [phantom type role on State breaks soundness](https://github.com/tomjaguarpaw/bluefin/issues/50)

もちろん、型誘導開発においても、このような「潜伏期間」の問題からは自由ではありません。
ただしそれは型安全性ではなく、パフォーマンスの予測可能性という形で現れます。
つまり、一見効率的であるようだが、特定の使い方をすると極端に性能が落ちる、ということがありえるのです。
これは`polysemy`のスペースリークの例などが当てはまります: [Possible Space Leak](https://github.com/polysemy-research/polysemy/issues/340)

ただし性能問題は、後に発見された場合でも、インターフェースを変更せずに改善する余地があるため[^9]、エコシステム全体が修正の影響を受けるというリスクが少ないという違いがあります。
一方でインターフェースの穴は、Javaのnullのように、一度普及しエコシステムに定着してしまうと後の修正が困難になるというリスクがあります。
現実的には、このようなインターフェースの穴によりエフェクトのエコシステムの大規模な改修が必要になるというリスクはそれほど大きくないのかもしれません。
しかし、そのリスクを把握しておくことには価値があると思います。
もちろん、この問題は`ReaderT IO`方式以外も無縁ではありません。
並行性を利用したインタプリタなどを実装するために`IO`に依存した瞬間、その時点で安全なインターフェースを設計する問題と向き合い始める必要があるからです。

ただし穴の発生源が、`ReaderT IO`の場合はライブラリのコア部分で起こりうるが、一方で他ではその並行性インタプリタのような一部の部分だけに抑えられる、というように、その局所性において差はあります。
穴が起きうるunsafeな部分を可能な限り小さく、検証可能にすることが重要です。

この点において, "the Haskell way" offers a superior approach.
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

[To be continued in Part 1.4...]({% post_url 2025-05-14-heftia-rev-part-1-4 %})

[^1]: [Hefty Algebras: Modular Elaboration of Higher-Order Algebraic Effects. Casper Bach Poulsen & Cas van der Rest, POPL 2023.](https://dl.acm.org/doi/10.1145/3571255)

[^2]: Here, “correctly” means that the computation results predicted by the underlying theory and those produced by the implementation must always match.

[^3]: [Choice of functorial state for runNonDet](https://github.com/polysemy-research/polysemy/issues/246)

[^4]: > Their work also considers the problem of modular composition of handlers, and presents a solution–called “weaving”–based on threading a handler state through unknown operations. This approach is rather ad-hoc; it is not a generalization of the forwarding approach for algebraic effects. <cite><a href="https://arxiv.org/abs/2304.09697">A Calculus for Scoped Effects & Handlers</a></cite>

[^5]: The causes for Freer's perceived slowness are multifaceted. One significant reason, identified in the paper "Reflection without Remorse," can be resolved by using a data structure called `FTCQueue`. In `heftia`, as in `freer-simple`, `FTCQueue` is used to achieve higher performance compared to other Freer implementations.

[^6]: > In this work we propose a generic framework for higher-order effects, generalizing algebraic effects & handlers: a generic free monad with higher-order effect signatures and a corresponding interpreter. <cite><a href="https://dl.acm.org/doi/10.1016/j.scico.2024.103086">A framework for higher-order effects & handlers</a></cite>

[^7]: ただし、`bluefin`に関しては厳密には`ReaderT IO`とは呼ぶことができない（単に`IO`方式）という話があります。依然としてエビデンスパッシング方式ではあります。つまり、`ReaderT IO`とエビデンスパッシングには若干の意味の差があります。

[^8]: > In what follows we present the categorical foundations of our generic free monad. <cite><a href="https://dl.acm.org/doi/10.1016/j.scico.2024.103086">A framework for higher-order effects & handlers</a></cite>

[^9]: ただし、polysemyのスペースリーク問題が4年ほどcloseされていないことを鑑みると、あくまで「原理上」の話であることがわかります。
